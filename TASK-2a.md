# TASK: Phase 2a — pipeline trích text PDFBox (async), phát hiện PDF scan

Chạy sau khi Phase 1a + 1b xanh. Task này chỉ làm pipeline nền — chưa có endpoint đọc text
(đó là Phase 2b).

## Xong =
`gradle test` exit 0, Docker Desktop đang chạy.

## File được phép tạo/sửa
- `build.gradle` (thêm PDFBox, có version — không có BOM quản)
- `src/main/resources/application.yml` (thêm khoá `extraction`)
- `src/main/java/app/Application.java` (thêm `@EnableAsync`, không tạo class config riêng)
- `src/main/resources/db/migration/V2__document_text.sql`
- `src/main/java/app/document/DocumentText.java`
- `src/main/java/app/document/DocumentTextRepository.java`
- `src/main/java/app/document/PdfTextExtractionService.java`
- `src/main/java/app/document/DocumentStartupRecoveryRunner.java`
- `src/main/java/app/document/DocumentService.java` (sửa: gọi extraction sau khi lưu document mới)
- `src/test/java/app/document/PdfTextExtractionServiceTest.java`
- `src/test/java/app/document/DocumentStartupRecoveryRunnerTest.java`

Mọi file khác CHỈ ĐỌC — kể cả mọi thứ Phase 1a/1b đã tạo.

## Đã chốt — không hỏi lại, không tự đổi
- **PDFBox 3.x**: kiểm tra Maven Central lấy version mới nhất, ghi rõ version vào `build.gradle`
  (`implementation 'org.apache.pdfbox:pdfbox:<version>'`). **Không đoán từ trí nhớ** — API PDFBox 3
  đổi khác PDFBox 2 (`Loader.loadPDF(...)` thay vì `PDDocument.load(...)`).
- **Async bằng cấu hình có sẵn của Boot**, không tự viết `Executor` bean:
  `@EnableAsync` trên `Application.java`, pool chỉnh qua
  `spring.task.execution.pool.core-size` / `max-size` trong `application.yml` (Boot tự tạo
  `ThreadPoolTaskExecutor` cho `@Async`). Đặt core=2, max=4.
- **Trigger**: `DocumentService` gọi `PdfTextExtractionService` (method có `@Async`) ngay sau khi
  lưu document MỚI (không phải bản ghi trùng sha256) và ghi file xong. Không có endpoint riêng để
  kích hoạt trích text.
- **Ranh giới transaction — đọc kỹ, sai chỗ này thì test chạy lúc xanh lúc đỏ:**
  - Method upload trong `DocumentService` **KHÔNG được đánh `@Transactional`**. Chỉ gọi
    `repository.save(...)` (bản thân nó đã chạy trong transaction riêng và commit ngay), ghi file,
    rồi mới gọi method `@Async`. Nếu bọc `@Transactional` quanh cả cụm thì thread async khởi động
    trước lúc commit, query không thấy document → lỗi ngẫu nhiên.
  - Trong `PdfTextExtractionService`: ghi `status = PROCESSING` bằng **một transaction riêng đã
    commit xong** rồi mới mở file PDF. Không bọc `@Transactional` quanh toàn bộ method async.
    Lý do: bọc cả method thì `PROCESSING` chỉ commit lúc method kết thúc, app crash giữa chừng sẽ
    rollback về `PENDING` — mà runner recovery chỉ tìm `PROCESSING`, nên bản ghi kẹt vĩnh viễn.
    Cách đơn giản nhất: tách hai method nhỏ `markProcessing(id)` và `markFinished(...)`, mỗi cái
    `@Transactional` riêng, đặt ở một bean khác để proxy của Spring có hiệu lực (gọi method
    `@Transactional` của chính class mình thì proxy bị bỏ qua — đây là bẫy self-invocation).
- **Vòng đời status** (`document.status`, enum đã có từ Phase 1a):
  `PENDING` → `PROCESSING` (set ngay khi extraction bắt đầu, trước khi mở file) →
  `TEXT_READY` / `UNSUPPORTED` / `FAILED`.
- **Phát hiện PDF scan**: mở được file, trích text từng trang. Trung bình số ký tự
  non-whitespace/trang **< `extraction.scanned-pdf-char-threshold`** (mặc định `100`, config
  được) → `UNSUPPORTED`, `failure_reason = NO_TEXT_LAYER`. Ngược lại → `TEXT_READY`.
- **Lỗi mở/parse file** (corrupt, mã hoá không đọc được) → `FAILED`,
  `failure_reason = EXTRACTION_ERROR`. Log message gốc, **không cần lưu stacktrace vào DB**.
- **`page_count`**: set ngay khi PDFBox mở file thành công — trước khi biết `TEXT_READY` hay
  `UNSUPPORTED` (cả hai đều nghĩa là "mở được").
- **`document_text`**: ghi đúng một dòng khi có text thật (trạng thái `TEXT_READY`). Trạng thái
  `UNSUPPORTED`/`FAILED` → không ghi dòng nào vào `document_text`.
- **Startup recovery**: một `ApplicationRunner`, chạy đúng một lần lúc app khởi động. Update tất
  cả `document` có `status = 'PROCESSING'` và `updated_at` cũ hơn
  `extraction.stuck-processing-minutes` phút (mặc định `15`, config được) →
  `status = 'FAILED'`, `failure_reason = 'INTERRUPTED'`. Một câu `UPDATE`, không đọc từng bản ghi
  ra rồi lưu lại. Câu `UPDATE` này viết trong repository dạng `@Modifying @Query(...)` và
  **bắt buộc phải có `@Transactional`** đi kèm — thiếu là ném `TransactionRequiredException`
  lúc chạy, không phải lỗi compile nên đọc log mới thấy.
- **`DocumentText` map phẳng**: `@Id private UUID documentId;` — một trường UUID thường, không
  `@OneToOne`, không `@MapsId`, không quan hệ tới `Document`. Phase 2 chưa cần join.

## Việc
1. Thêm PDFBox vào `build.gradle`.
2. `V2__document_text.sql`: bảng `document_text` (`document_id` PK/FK, `content` text,
   `char_count` int, `extractor` text, `extracted_at` timestamptz).
3. Thêm khoá `extraction.scanned-pdf-char-threshold` và `extraction.stuck-processing-minutes`
   vào `application.yml`; thêm `spring.task.execution.pool.core-size`/`max-size`.
4. `@EnableAsync` trên `Application.java`.
5. `PdfTextExtractionService` — method `@Async` nhận `documentId`: set `PROCESSING` → mở file từ
   `storage_path` bằng PDFBox → trích text từng trang → tính trung bình ký tự/trang → quyết
   `TEXT_READY`/`UNSUPPORTED` → ghi `document_text` nếu có → cập nhật `document`
   (`status`, `page_count`, `failure_reason`, `updated_at`). Lỗi mở file → `FAILED`.
6. Sửa `DocumentService`: gọi bước 5 sau khi lưu document mới thành công (bỏ qua nếu là bản ghi
   trùng sha256 trả lại từ trước).
7. `DocumentStartupRecoveryRunner` — `ApplicationRunner` chạy câu `UPDATE` mô tả ở trên.
8. `PdfTextExtractionServiceTest` — **đúng ba test, không thêm test thứ tư**:
   - `extractsTextFromRealPdf_setsStatusTextReady` — tạo PDF có text bằng chính PDFBox trong test
     (`PDDocument` + `PDPageContentStream`, không cần file nhị phân đính kèm repo), chạy service,
     verify `document_text.content` khớp, `status = TEXT_READY`, `page_count` đúng.
   - `scannedPdfBelowThreshold_setsStatusUnsupported` — PDF tạo trong test không vẽ text (trang
     trắng) → `status = UNSUPPORTED`, `failure_reason = NO_TEXT_LAYER`, không có dòng trong
     `document_text`.
   - `corruptFile_setsStatusFailed` — trỏ `storage_path` tới file không phải PDF hợp lệ → `status
     = FAILED`, `failure_reason = EXTRACTION_ERROR`.
9. `DocumentStartupRecoveryRunnerTest` — đúng hai ca:
   - insert thẳng một `document` `status = PROCESSING`, `updated_at = now() - 20 phút` → chạy
     runner → verify `FAILED` + `INTERRUPTED`.
   - insert một `document` `status = PROCESSING`, `updated_at = now() - 5 phút` → chạy runner →
     verify VẪN `PROCESSING` (chưa đủ ngưỡng).
10. Chạy `gradle test`.

## Không làm
- Không viết `GET /documents/{id}/text` — Phase 2b.
- Không đọc PDF từ multipart trong service này — luôn đọc lại từ đĩa qua `storage_path`.
- Không xử lý bố cục bảng biểu / toạ độ — chỉ text tuần tự theo trang.
- Không retry khi `FAILED` — client tự upload lại nếu muốn.
- Không dùng `@Scheduled` định kỳ cho recovery — chỉ chạy một lần lúc khởi động.
- Không tự viết `Executor` bean — dùng cấu hình `spring.task.execution.pool.*` có sẵn của Boot.

## Bí thì
- Chờ kết quả `@Async` trong test: dùng vòng lặp poll đơn giản (`Thread.sleep` ngắn, có timeout
  vài giây, đọc lại `DocumentRepository`) — **không thêm dependency Awaitility**, viết một helper
  method nhỏ trong test là đủ.
- **API PDFBox 3.x khác PDFBox 2 ở đúng mấy chỗ dưới đây. Gần như mọi tutorial trên mạng còn
  viết theo API 2, chép vào là lỗi compile:**
  - Mở file: `Loader.loadPDF(File)` — **không phải** `PDDocument.load(File)` (đã xoá).
  - Font trong test: `new PDType1Font(Standard14Fonts.FontName.HELVETICA)` — **không phải**
    `PDType1Font.HELVETICA` (đã xoá). Import `org.apache.pdfbox.pdmodel.font.Standard14Fonts`.
  - Vẽ text: `beginText()` → `setFont(font, 12)` → `newLineAtOffset(x, y)` → `showText("...")`
    → `endText()`. Thiếu `beginText`/`endText` là lỗi lúc chạy, không phải lúc compile.
  - Đọc text: `PDFTextStripper` giữ nguyên như cũ (`setStartPage`/`setEndPage`/`getText`).
- Còn nghi ngờ chỗ nào khác của PDFBox: đọc Javadoc chính thức trên trang PDFBox, đừng chép
  tutorial.
- Lỗi compile / cú pháp: tự sửa, tối đa 3 vòng.
- Sau 3 vòng `gradle test` vẫn fail cùng một lỗi → `ask_claude`, dán NGUYÊN log gradle.
- Muốn đổi thứ trong "Đã chốt" → `ask_claude`, cấm tự quyết.
- Cấm ghi lại một file với nội dung y hệt lần trước.
