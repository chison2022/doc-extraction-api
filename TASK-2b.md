# TASK: Phase 2b — endpoint đọc text đã trích

Chạy sau khi Phase 2a xanh. Task này chỉ thêm một endpoint đọc dữ liệu `document_text` đã có
sẵn từ pipeline Phase 2a — không đụng tới logic trích xuất.

## Xong =
`gradle test` exit 0, Docker Desktop đang chạy.

## File được phép tạo/sửa
- `src/main/java/app/document/DocumentController.java` (thêm endpoint)
- `src/main/java/app/document/DocumentDtos.java` (thêm record response)
- `src/main/java/app/document/DocumentService.java` (thêm method lấy text theo id)
- `src/test/java/app/document/DocumentTextApiTest.java`

Mọi file khác CHỈ ĐỌC — kể cả `PdfTextExtractionService.java` và `document_text` schema của 2a.

## Đã chốt — không hỏi lại, không tự đổi
- `GET /api/v1/documents/{id}/text`:
  - Document không tồn tại → **404**.
  - Document tồn tại nhưng `status != TEXT_READY` → **409**, `ProblemDetail` nêu rõ status hiện
    tại (ví dụ "document is PENDING, text not ready yet").
  - `status = TEXT_READY` → **200**, JSON:
    `{ "content": "...", "charCount": 1234, "extractor": "pdfbox", "extractedAt": "..." }`.
  - Trả JSON (không phải `text/plain`) — nhất quán với quy ước "JSON in/out" toàn API, không có
    ngoại lệ cho endpoint này.
- Nếu `status = TEXT_READY` mà `document_text` lại không có bản ghi (không nên xảy ra nếu 2a
  đúng) — không cần xử lý riêng, để lỗi 500 mặc định của Spring nổi lên.
- **Kiểu thời gian của `extractedAt` phải trùng kiểu Phase 1a đã dùng cho `created_at`** trong
  `Document` (`Instant` hay `OffsetDateTime` — mở `Document.java` ra xem, đừng đoán). Trộn hai
  kiểu trong cùng một API là thứ về sau phải đi sửa lại toàn bộ.

## Việc
1. `DocumentDtos`: thêm record `DocumentTextResponse(String content, int charCount,
   String extractor, <kiểu thời gian giống Document.createdAt> extractedAt)`.
2. `DocumentService`: thêm method lấy document + document_text theo id, ném exception phù hợp để
   `ApiExceptionHandler` (đã có từ Phase 1b) convert thành 404/409.
3. `DocumentController`: thêm `GET /documents/{id}/text`.
4. `DocumentTextApiTest` kế thừa `PostgresTestBase`, dùng `MockMvc`, đúng bốn test:
   - upload PDF có text thật qua HTTP, poll tới khi `status = TEXT_READY` (helper poll giống
     Phase 2a, không thêm dependency), gọi endpoint → 200, `content` khớp.
   - gọi endpoint với id ngẫu nhiên không tồn tại → 404.
   - insert thẳng một `document` `status = PENDING` qua repository (không qua HTTP, để không phụ
     thuộc timing async) → gọi endpoint → 409.
   - upload PDF trắng (không có text layer), poll tới khi `status = UNSUPPORTED` → gọi endpoint →
     409.
5. Chạy `gradle test`.

## Không làm
- Không sửa logic trích xuất hay ngưỡng phát hiện scan — đó là 2a, chỉ đọc dữ liệu đã có.
- Không thêm endpoint nào khác ngoài `GET /documents/{id}/text`.
- Không phục vụ file PDF gốc qua HTTP — spec §11 cấm ở v1 (bề mặt path-traversal).

## Bí thì
- Chờ `@Async` xong trong test upload thật: poll `DocumentRepository` bằng vòng lặp
  `Thread.sleep` ngắn có timeout, giống pattern đã dùng ở Phase 2a.
- Muốn test case PENDING/PROCESSING ổn định không phụ thuộc timing → insert thẳng bản ghi qua
  repository thay vì upload thật qua HTTP, như đã chốt ở test thứ ba.
- Lỗi compile / cú pháp: tự sửa, tối đa 3 vòng.
- Sau 3 vòng `gradle test` vẫn fail cùng một lỗi → `ask_claude`, dán NGUYÊN log gradle.
- Muốn đổi thứ trong "Đã chốt" → `ask_claude`, cấm tự quyết.
- Cấm ghi lại một file với nội dung y hệt lần trước.
