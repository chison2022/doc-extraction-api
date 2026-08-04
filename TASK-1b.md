# TASK: Phase 1b — upload, list, ràng buộc đầu vào

Chạy sau khi Phase 1a xanh. Task này thêm endpoint và **toàn bộ kiểm tra đầu vào §5.3**.
§5.3 là biên tin cậy — chỗ duy nhất trong dự án không áp dụng "làm ít nhất có thể".

## Xong =
`gradle test` exit 0, Docker Desktop đang chạy. Chưa xanh thì CHƯA xong.

## File được phép tạo/sửa
- `src/main/java/app/document/DocumentController.java`
- `src/main/java/app/document/DocumentService.java`
- `src/main/java/app/document/DocumentDtos.java` (các `record` trong một file)
- `src/main/java/app/document/UploadException.java`
- `src/main/java/app/ApiExceptionHandler.java`
- `src/main/resources/application.yml` (thêm khoá `storage`)
- `src/test/java/app/document/DocumentApiTest.java`

Mọi file khác CHỈ ĐỌC — kể cả `V1__init.sql` và `Document.java` của Phase 1a.

## Đã chốt — không hỏi lại, không tự đổi
- Prefix `/api/v1`. Ba endpoint, không hơn:
  - `POST /documents` — multipart field tên `file` → **202** kèm metadata document.
  - `GET /documents` — `?page=0&size=20&status=&q=`, `q` lọc theo `filename` chứa chuỗi.
  - `GET /documents/{id}` — metadata, không có thì **404**.
- Tenant: Phase 6 mới có auth. Tới lúc đó dùng hằng số
  `00000000-0000-0000-0000-000000000001` (bản ghi `default` do `V1__init.sql` chèn).
  Không viết interceptor, không đọc header.
- **Kiểm tra đầu vào — không được rút gọn:**
  - Quá `storage.max-file-size` (mặc định 20MB) → **413**.
  - Đọc 5 byte đầu, không phải `%PDF-` → **400**. **Không tin `Content-Type` client gửi.**
  - Đường dẫn lưu là `{storage.root}/{tenantId}/{documentId}.pdf`. **Tuyệt đối không dùng tên file
    client gửi để đặt tên trên đĩa** — đây là đường path-traversal kinh điển. Tên gốc chỉ vào cột
    `filename` để hiển thị.
  - `size` của phân trang tối đa 100; xin lớn hơn thì kẹp về 100, không báo lỗi.
- Trùng nội dung: tính `sha256` của file. Đã có bản ghi cùng `(tenant_id, sha256)` thì **trả lại
  bản ghi cũ** kèm `duplicate: true`, không tạo document mới, không ghi thêm file xuống đĩa.
- Lỗi trả theo **RFC 9457** bằng `ProblemDetail` của Spring. Không tự chế class lỗi JSON.
- Phân trang trả `{ content, page, size, totalElements, totalPages }`, dựng từ `Page` của
  Spring Data. Không tự viết logic phân trang.
- Upload xong `status = PENDING`. Phase 1 chưa trích text, **không đụng tới PDFBox**.
- `storage.root` mặc định `./data/documents` (đã có trong `.gitignore`).

## Việc
1. Thêm khoá `storage` vào `application.yml`.
2. `DocumentDtos` — record cho response upload, response list, response metadata.
3. `DocumentService` — kiểm tra đầu vào, băm sha256, chống trùng, ghi file, lưu DB.
4. `DocumentController` + `ApiExceptionHandler` (`@RestControllerAdvice` → `ProblemDetail`).
5. `DocumentApiTest` kế thừa `PostgresTestBase`, dùng `MockMvc`, **đúng năm test**:
   - upload PDF hợp lệ → 202, DB có bản ghi, file nằm đúng `{root}/{tenantId}/{id}.pdf`;
   - upload file không bắt đầu bằng `%PDF-` (dù `Content-Type: application/pdf`) → 400;
   - upload quá 20MB → 413;
   - upload lại đúng file đó → trả cùng `id`, `duplicate: true`, DB vẫn một bản ghi;
   - tên file `../../evil.pdf` → vẫn 202, và **không có file nào được tạo ngoài `storage.root`**.
6. Chạy `gradle test`. Fail thì đọc log, sửa, chạy lại.

## Không làm
- Không viết `GET /documents/{id}/text` — Phase 2.
- Không thêm Spring Security, không auth, không API key.
- Không async, không `@Transactional` trên controller, không message queue.
- Không thêm dependency nào ngoài những gì Phase 1a đã có.

## Bí thì
- Test ghi file rác vào repo: dùng `@TempDir` hoặc ghi đè `storage.root` bằng
  `@DynamicPropertySource` trỏ vào thư mục tạm. Không commit `data/`.
- Lỗi compile / cú pháp: tự sửa, tối đa 3 vòng.
- Sau 3 vòng `gradle test` vẫn fail cùng một lỗi → `ask_claude`, dán NGUYÊN log gradle.
- Muốn đổi thứ trong "Đã chốt" → `ask_claude`, cấm tự quyết.
- Cấm ghi lại một file với nội dung y hệt lần trước.
