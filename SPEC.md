# Document Extraction API — Spec kỹ thuật v0.1

> **Trạng thái:** bản nháp để bạn duyệt. Chưa chốt, chưa có code.
> **Ngày:** 2026-08-01
> **Đi kèm:** `C:\Users\ADMIN\.claude\plans\oke-l-n-plan-c-n-tingly-lark.md` (lộ trình 12 tuần).
> Plan trả lời *khi nào làm gì*. Spec này trả lời *cái được xây ra trông như thế nào*.

**Spec này cố ý KHÔNG chứa code Java.** Nó định nghĩa hợp đồng (bảng, endpoint, định dạng,
công thức, tiêu chí đúng/sai) — phần dịch sang Java là việc của bạn, theo nguyên tắc "bạn viết
code, không phải AI" trong plan. Chỗ nào spec cố ý dừng lại để bạn tự quyết đều có dấu 🎓.

Viết tiếng Việt vì đây là tài liệu làm việc của bạn. Khi vào repo, README/javadoc/commit
vẫn viết tiếng Anh theo plan — đừng dịch file này rồi coi như đã luyện viết.

---

## 0. Phạm vi

### Sản phẩm làm gì

Nhận file PDF → trích text → dùng LLM local trích xuất ra JSON **theo schema do client định
nghĩa** → chấm điểm tin cậy từng trường → chạy luật kiểm tra nghiệp vụ → đẩy trường đáng ngờ
vào hàng đợi cho người rà soát. Kèm **bộ đo độ chính xác** chạy được bằng một lệnh.

### Hai thứ tạo khác biệt — không được cắt khi rút gọn phạm vi

1. **Schema do client định nghĩa.** Không hardcode cho hoá đơn. Cùng một binary phải trích được
   hoá đơn, hợp đồng, phiếu xuất kho — chỉ khác schema nạp vào.
2. **Bộ đo cấp trường trên tập có nhãn.** Số thật, không phải "nó chạy tốt".

### v1 KHÔNG làm

OCR · vision model · giao diện web · deploy cloud · định dạng khác PDF · streaming response ·
webhook · chia nhỏ tài liệu dài theo chunk rồi ghép (v1 cắt bớt + đánh dấu) · fine-tuning ·
so khớp line item bằng thuật toán ghép cặp tối ưu (v1 so theo thứ tự).

### Nguyên tắc thiết kế

- **Mỗi phase phải chạy được.** Không có bảng/endpoint nào tồn tại "để dành cho phase sau",
  trừ đúng một ngoại lệ đã cân nhắc: `tenant_id` (xem §2.1).
- **Không xây DSL, không xây plugin system, không xây abstraction cho thứ chỉ có một
  implementation.** Cụ thể: không có ngôn ngữ biểu thức cho luật kiểm tra, không có
  `ExtractorStrategy` interface khi chỉ có PDFBox.
- **Thà thô mà đo được, còn hơn tinh vi mà không kiểm chứng được.** Áp dụng mạnh nhất ở
  phần confidence (§7).

---

## 1. Kiến trúc

Một tiến trình Spring Boot duy nhất. Không microservice, không message broker.

```
HTTP  ──▶ Controller ──▶ Service ──▶ Postgres
                 │
                 ├──▶ đĩa (file PDF gốc)
                 ├──▶ PDFBox   (PDF → text)
                 └──▶ Ollama   (text + schema → JSON)   HTTP localhost:11434
```

Xử lý nặng (trích text, gọi model) chạy bất đồng bộ bằng `@Async` với một thread pool cấu hình
được. Không dùng hàng đợi ngoài — mất việc khi restart là chấp nhận được ở v1, đổi lại
`docker compose up` chỉ cần 2 container (app + postgres) và người lạ chạy được trong 5 phút.

**Hệ quả phải ghi vào README, đừng giấu:** job đang chạy dở mà app restart thì bản ghi kẹt ở
trạng thái `PROCESSING`. Cách xử lý ở v1: một job quét lúc khởi động, đưa mọi bản ghi
`PROCESSING` quá 15 phút về `FAILED` với lý do `INTERRUPTED`. Đây là câu trả lời đúng cho quy mô
này; nói thẳng giới hạn còn hơn giả vờ có exactly-once.

**Stack** (theo plan, xác nhận lại toạ độ ở Phase 0 — đọc docs chính thức, không copy tutorial):
Java 21 · Spring Boot 4.1.0 · Gradle 9.6.1 · PostgreSQL 16 + Flyway · Spring AI 2.0.0 + Ollama ·
Apache PDFBox 3 · Jackson · `networknt/json-schema-validator` · springdoc-openapi · Bucket4j ·
Testcontainers.

Dependency mới duy nhất nằm ngoài plan là **`networknt/json-schema-validator`** — cần để kiểm tra
JSON model trả về có đúng schema client khai báo không. Không có nó thì phải tự viết trình duyệt
JSON Schema, đúng thứ không nên tự viết.

---

## 2. Mô hình dữ liệu

Postgres. Khoá chính `UUID` (`gen_random_uuid()`, pgcrypto có sẵn trong Postgres 13+). Mọi mốc
thời gian là `timestamptz` UTC. Tiền và số lượng dùng `numeric`, **không dùng `double`**.

### 2.1 Một quyết định lệch so với plan: `tenant_id` có từ V1

Plan xếp multi-tenant vào Phase 6. Nhưng thêm `tenant_id NOT NULL` vào 6 bảng đã có dữ liệu ở
tuần 9 là một migration khó chịu và dễ sai. Chi phí đưa cột này vào từ đầu gần như bằng không:
V1 seed sẵn một tenant `default`, mọi thứ tới Phase 5 dùng nó, Phase 6 chỉ thêm phần **xác thực
và lọc**. Đây là ngoại lệ duy nhất được phép của nguyên tắc "không xây trước".

Lợi ích phụ: bộ đo ở Phase 4 chạy dưới tenant riêng tên `eval`, dữ liệu benchmark không lẫn với
dữ liệu bạn nghịch tay — mà không cần thêm cột `is_eval` nào.

### 2.2 Bảng

**`tenant`** — `id`, `name` (unique), `created_at`.

**`api_key`** (Phase 6) — `id`, `tenant_id`, `key_hash` (SHA-256 hex, unique — **không lưu key
thô**), `label`, `created_at`, `last_used_at`, `revoked_at` (null = còn hiệu lực).

**`document`** — file đã upload.

| cột | kiểu | ghi chú |
|---|---|---|
| `id` | uuid PK | |
| `tenant_id` | uuid FK | |
| `filename` | text | tên gốc người dùng gửi, chỉ để hiển thị |
| `size_bytes` | bigint | |
| `content_type` | text | |
| `sha256` | char(64) | băm nội dung |
| `storage_path` | text | tương đối so với thư mục gốc lưu trữ |
| `page_count` | int null | điền sau khi PDFBox mở được |
| `status` | text | xem §4.1 |
| `failure_reason` | text null | mã lỗi ổn định + mô tả |
| `version` | bigint | optimistic locking (§4.3 🎓) |
| `created_at` / `updated_at` | timestamptz | |

`UNIQUE (tenant_id, sha256)` — upload trùng file trả về bản ghi cũ kèm `duplicate: true` thay vì
tạo mới. Rẻ, và tránh làm bẩn số liệu đo ở Phase 4.

**`document_text`** — `document_id` PK/FK, `content` text, `char_count` int, `extractor` text
(`pdfbox`), `extracted_at`.
*Tách bảng riêng* để `GET /documents` không kéo theo hàng megabyte text.

**`extraction_schema`** — schema do client định nghĩa.
`id`, `tenant_id`, `name`, `version` int, `json_schema` jsonb, `rules` jsonb (§8),
`created_at`. `UNIQUE (tenant_id, name, version)`.
Schema **bất biến**: sửa = tạo version mới. Bắt buộc, nếu không thì mọi số đo ở Phase 4 mất ý
nghĩa vì không biết đo trên schema nào.

**`extraction_run`** — **một lần chạy trích xuất**, không phải "kết quả của tài liệu".

| cột | ghi chú |
|---|---|
| `id`, `tenant_id`, `document_id`, `schema_id` | |
| `model` | tên model Ollama, ví dụ `qwen2.5:7b` |
| `prompt_version` | chuỗi, ví dụ `v3` — để so sánh prompt ở Phase 4 |
| `status` | xem §4.2 |
| `result_json` | jsonb — **giá trị hiện hành** (đã áp bản sửa của người dùng) |
| `raw_response` | text — chuỗi model trả về nguyên văn, bất biến, để truy vết |
| `input_char_count`, `truncated` | boolean: có bị cắt bớt vì quá dài không |
| `started_at`, `finished_at`, `duration_ms` | |
| `failure_reason` | |

**Đây là chỗ lệch thứ hai so với plan.** Plan mô tả một cột `status` trên `document` và một bảng
`extraction_result`. Nhưng Phase 4 yêu cầu chạy **cùng một tài liệu qua 3–4 model** để so sánh —
tức một tài liệu phải có nhiều lần chạy cùng tồn tại. Nếu kết quả là quan hệ 1-1 với document thì
Phase 4 sẽ phải đập lại mô hình dữ liệu giữa lộ trình. Tách ra từ đầu rẻ hơn nhiều.

**`extraction_field`** — bản làm phẳng của `result_json`, mỗi trường một dòng.

| cột | ghi chú |
|---|---|
| `id`, `run_id` | |
| `field_path` | JSON Pointer RFC 6901: `/invoice_number`, `/line_items/0/qty` |
| `value_json` | jsonb — giá trị model trả về |
| `confidence` | numeric(3,2) 0.00–1.00 |
| `confidence_reason` | text — mã lý do, xem §7 |
| `corrected_value_json` | jsonb null — bản người sửa |
| `corrected_at`, `corrected_by` | |

`result_json` là nguồn sự thật; `extraction_field` là **bản chiếu (projection)** được dựng lại
mỗi khi `result_json` thay đổi, trong cùng transaction. Không merge lúc đọc, không hai nguồn sự
thật. Đổi lại phải nhớ: sửa `result_json` mà quên dựng lại projection là bug — viết một test cho
đúng chuyện đó.

*Lý do có bảng này thay vì truy vấn thẳng jsonb:* hàng đợi rà soát cần
`WHERE confidence < 0.5 AND tenant_id = ?` xuyên nhiều tài liệu, và bộ đo cần đối chiếu theo
đường dẫn trường. Cả hai là SQL một dòng với bảng phẳng, và là truy vấn khó chịu với `jsonb_each`.
Nó cũng chính là bảng vài nghìn dòng cho bài học index ở Phase 6.
*Dùng JSON Pointer, không tự chế cú pháp đường dẫn* — Jackson đã có `JsonPointer` và
`JsonNode.at()`, không phải viết parser.

**`validation_issue`** (Phase 5) — `id`, `run_id`, `rule_id`, `severity` (`ERROR`|`WARN`),
`field_paths` text[], `message`.

### 2.3 Migration theo phase

`V1__init.sql` (Phase 1) tenant, document, api_key ·
`V2__document_text.sql` (Phase 2) ·
`V3__schema_and_run.sql` (Phase 3) ·
`V4__validation.sql` (Phase 5) ·
`V5__indexes.sql` (Phase 6 — **viết sau khi đã tự đo bằng `EXPLAIN ANALYZE`**, không thêm index
đoán mò từ đầu; cả bài học nằm ở chỗ đó).

---

## 3. Định dạng schema trích xuất

Đây là phần tạo khác biệt số 1 — cần chặt chẽ nhất.

### 3.1 Tập con JSON Schema được hỗ trợ

Nhận JSON Schema draft 2020-12, nhưng **chỉ hỗ trợ tập con dưới đây**. Gửi gì ngoài tập này thì
từ chối lúc tạo schema với 400 — thà nói không rõ ràng còn hơn im lặng bỏ qua rồi ra kết quả sai.

- Gốc bắt buộc là `type: "object"`.
- Kiểu trường: `string`, `number`, `integer`, `boolean`, `array` (phần tử là object hoặc string).
- `format` cho string: `date` (ISO-8601), `email`, không hỗ trợ format khác.
- Từ khoá: `properties`, `required`, `description`, `enum`, `items`.
- Lồng nhau: object trong array — **đúng một cấp** (đủ cho `line_items`). Sâu hơn → 400.
- **`description` là bắt buộc cho mọi trường.** Không phải để đẹp: nó đi thẳng vào prompt và là
  thứ ảnh hưởng độ chính xác nhiều nhất. Schema không có description thì model đoán mò.

Không hỗ trợ: `oneOf`/`anyOf`/`allOf`, `$ref`, `patternProperties`, `additionalProperties`
dạng schema, số học ràng buộc (`minimum`…) — những cái này model nhỏ cũng không tôn trọng nổi,
đưa vào chỉ tạo ảo giác an toàn.

### 3.2 Ví dụ

```json
{
  "type": "object",
  "required": ["invoice_number", "invoice_date", "total"],
  "properties": {
    "invoice_number": { "type": "string",  "description": "Invoice number as printed, without any prefix label" },
    "invoice_date":   { "type": "string", "format": "date", "description": "Issue date of the invoice" },
    "vendor_name":    { "type": "string",  "description": "Legal name of the party issuing the invoice" },
    "currency":       { "type": "string", "enum": ["USD","EUR","VND"], "description": "ISO currency code" },
    "total":          { "type": "number",  "description": "Grand total including tax" },
    "line_items": {
      "type": "array",
      "description": "One entry per billed line",
      "items": {
        "type": "object",
        "required": ["description", "amount"],
        "properties": {
          "description": { "type": "string", "description": "Item description" },
          "quantity":    { "type": "number", "description": "Quantity billed" },
          "unit_price":  { "type": "number", "description": "Price per unit" },
          "amount":      { "type": "number", "description": "Line total" }
        }
      }
    }
  }
}
```

### 3.3 Trường không tìm thấy

Model **phải** trả `null` cho trường không có trong tài liệu, **không được bịa**. Ghi rõ trong
prompt (§6.2). Ở tầng schema: trường trong `required` mà `null` → không phải lỗi kỹ thuật, mà là
`validation_issue` mức `ERROR` + confidence 0. Phân biệt này quan trọng — "model không tìm thấy"
là kết quả hợp lệ và hữu ích, "model bịa" mới là hỏng.

---

## 4. Vòng đời và trạng thái

### 4.1 `document.status`

```
PENDING ──▶ PROCESSING ──┬──▶ TEXT_READY
                         ├──▶ UNSUPPORTED   (PDF scan, không có text layer)
                         └──▶ FAILED        (file hỏng, mã hoá, PDFBox ném lỗi)
```

Trạng thái này **chỉ nói về file**, không nói gì về trích xuất. Đọc được text rồi thì document
xong việc của nó.

**Phát hiện PDF scan:** trích được `< 100` ký tự không phải khoảng trắng trên mỗi trang, tính
trung bình → `UNSUPPORTED`, `failure_reason = NO_TEXT_LAYER`. Ngưỡng để trong config chứ đừng
chôn trong code — tài liệu thật sẽ đòi bạn chỉnh, đây là loại số không đoán đúng từ bàn giấy.

### 4.2 `extraction_run.status`

```
QUEUED ──▶ RUNNING ──┬──▶ SUCCEEDED       kết quả hợp lệ theo schema
                     ├──▶ NEEDS_REVIEW    hợp lệ nhưng có trường confidence thấp hoặc luật fail
                     └──▶ FAILED          model trả về không parse/validate được sau khi thử lại
```

`NEEDS_REVIEW` là trạng thái **thành công**, không phải lỗi — API vẫn trả kết quả đầy đủ.

### 4.3 🎓 Đồng thời — spec cố ý dừng ở đây

Plan để dành câu này cho bạn tự trả lời, spec tôn trọng điều đó:

> Hai request cùng lúc xử lý một document thì cột `status` ra sao?

Cột `version` đã có sẵn trong bảng. **Đừng hỏi AI cách dùng.** Tự bắn hai request đồng thời, tự
xem chuyện gì xảy ra, rồi mới quyết. Spec chỉ chốt phần quan sát được từ bên ngoài: gọi trích
xuất cho một document đang có run ở trạng thái `RUNNING` với **cùng schema và cùng model** phải
trả `409 Conflict`, không tạo run thứ hai. Khác schema hoặc khác model thì được phép chạy song
song — Phase 4 cần đúng điều đó.

---

## 5. API

Prefix `/api/v1`. JSON in/out. Lỗi theo **RFC 9457 Problem Details** — Spring Boot dựng sẵn
(`ProblemDetail`), không cần class lỗi tự chế.

### 5.1 Endpoint

| Method | Path | Phase | Mô tả |
|---|---|---|---|
| POST | `/documents` | 1 | Upload multipart. Query `?schemaId=&model=` (tuỳ chọn) → tự chạy tiếp tới trích xuất. → **202** |
| GET | `/documents` | 1 | Danh sách, phân trang, lọc `status`, `q` (tên file) |
| GET | `/documents/{id}` | 1 | Metadata |
| GET | `/documents/{id}/text` | 2 | Text thô. 409 nếu chưa `TEXT_READY` |
| POST | `/schemas` | 3 | Tạo schema (kiểm tra tập con §3.1) → 201 |
| GET | `/schemas` · `/schemas/{id}` | 3 | |
| POST | `/documents/{id}/extractions` | 3 | Body `{schemaId, model?, promptVersion?}` → **202** + run id |
| GET | `/documents/{id}/extractions` | 3 | Mọi lần chạy của tài liệu (Phase 4 cần) |
| GET | `/extractions/{runId}` | 3 | Kết quả: `result_json` + confidence + issues |
| GET | `/review` | 5 | Hàng đợi: trường cần rà, xuyên tài liệu. Lọc `maxConfidence`, `schemaId` |
| PATCH | `/extractions/{runId}/fields` | 5 | Body `[{path, value}]` → sửa nhiều trường một lần |

Không có `DELETE` ở v1 — chưa ai cần, thêm sau nếu thật sự cần.

### 5.2 Ví dụ response — `GET /extractions/{runId}`

```json
{
  "id": "…", "documentId": "…", "schema": { "name": "invoice", "version": 1 },
  "model": "qwen2.5:7b", "promptVersion": "v3",
  "status": "NEEDS_REVIEW",
  "durationMs": 8421, "truncated": false,
  "result": {
    "invoice_number": "INV-2026-0042",
    "invoice_date": "2026-03-14",
    "total": 1250.00,
    "line_items": [{ "description": "Consulting", "quantity": 10, "unit_price": 125.00, "amount": 1250.00 }]
  },
  "fields": [
    { "path": "/invoice_number", "confidence": 0.90, "reason": "GROUNDED" },
    { "path": "/invoice_date",   "confidence": 0.40, "reason": "NORMALIZED_NOT_GROUNDED" },
    { "path": "/total",          "confidence": 0.70, "reason": "RULE_CONFIRMED" }
  ],
  "issues": [
    { "ruleId": "TOTALS_MATCH", "severity": "WARN", "fieldPaths": ["/total"], "message": "…" }
  ]
}
```

`result` là JSON **đúng hình dạng schema client khai báo** — đó là thứ client tích hợp. Mọi thứ
khác (`fields`, `issues`) là siêu dữ liệu bên cạnh, không trộn vào trong `result`. Trộn vào là
sai lầm kinh điển của API loại này: client phải bóc tách mới dùng được.

### 5.3 Ràng buộc đầu vào — không được đơn giản hoá

Đây là biên tin cậy, chỗ duy nhất trong spec không áp dụng "làm ít nhất có thể":

- Dung lượng tối đa 20MB (config). Vượt → 413.
- Chỉ nhận PDF, và **kiểm tra magic bytes `%PDF-`**, không tin `Content-Type` client gửi.
- Tên file: **không dùng tên client gửi để đặt tên file trên đĩa.** Lưu theo
  `{storageRoot}/{tenantId}/{documentId}.pdf`. Tên gốc chỉ nằm trong DB để hiển thị. Đây là
  đường path-traversal kinh điển.
- Trường `path` trong PATCH phải là JSON Pointer tồn tại trong run đó → không thì 400.
- Kích thước schema: tối đa 200 trường (đếm cả trường lồng) → tránh prompt phình vô hạn.

### 5.4 Phân trang

`?page=0&size=20`, tối đa `size=100`. Response `{ content: [], page, size, totalElements, totalPages }`.
Dùng `Pageable` của Spring Data, không tự viết.

---

## 6. Lõi trích xuất (Phase 3)

### 6.1 ⚠️ Điểm phải sửa so với plan: structured output với schema động

Plan nhắc `BeanOutputConverter`. **Cách đó không dùng được ở đây**: `BeanOutputConverter<T>` cần
một class Java biết trước lúc biên dịch, còn schema của ta do client định nghĩa lúc chạy. Sinh
class động là con đường không nên đi.

Cách đúng: **đưa thẳng JSON Schema cho Ollama qua tham số `format`** của nó (Ollama hỗ trợ
structured output bằng JSON Schema ở tầng API, model bị ràng buộc lúc sinh token chứ không phải
"được yêu cầu lịch sự"), rồi parse chuỗi trả về bằng Jackson và **validate lại bằng
`json-schema-validator`** trước khi tin.

> Cần xác minh ở Phase 0: **tên chính xác của tham số/API trong Spring AI 2.x để gắn JSON Schema
> vào request Ollama** (`OllamaOptions.format`, `useProviderStructuredOutput()`, `validateSchema()`
> — plan có nhắc, tôi không tự xác minh được ở đây). Đọc docs chính thức, đừng tin tutorial.
> Đường lui nếu Spring AI không lộ ra được: gọi thẳng `POST /api/generate` của Ollama bằng
> `RestClient` — mất tiện nghi, nhưng vẫn đúng và không bị chặn tiến độ.

**Chọn model:** dùng model **không có chế độ thinking**. `qwen3` trả kèm đoạn suy nghĩ dạng text
thô và làm hỏng parse — plan đã ghi bẫy này. Model mặc định để trong config, không hardcode; Phase
4 sẽ thay bằng số liệu chứ không thay bằng cảm giác.

### 6.2 Cấu trúc prompt

Có `prompt_version` để so sánh được ở Phase 4. Ba phần:

1. **System** — vai trò + ba luật cứng: (a) chỉ trích thông tin có trong tài liệu; (b) không tìm
   thấy thì trả `null`, **tuyệt đối không đoán**; (c) giữ nguyên văn giá trị như in trong tài
   liệu, trừ ngày (chuẩn hoá ISO) và số (bỏ dấu phân cách nghìn).
2. **Schema** — JSON Schema kèm mọi `description`.
3. **Tài liệu** — text thô, bọc trong delimiter rõ ràng.

**Cắt bớt khi quá dài:** nếu text vượt `extraction.max-input-chars` (mặc định 24000, config được),
cắt và đặt `truncated = true`. Không im lặng cắt. Chunk + ghép là việc "để sau" — nhưng cột
`truncated` phải có từ đầu, nếu không Phase 4 sẽ cho ra số đo sai mà không biết vì sao.

### 6.3 Xử lý sau khi model trả về

Theo thứ tự, dừng ở bước đầu tiên hỏng:

1. Parse JSON. Hỏng → **thử lại đúng 1 lần** với cùng prompt (model nhỏ thi thoảng lỗi ngẫu
   nhiên). Vẫn hỏng → `FAILED`, `failure_reason = UNPARSEABLE_RESPONSE`, lưu `raw_response`.
2. Validate theo JSON Schema. Hỏng → thử lại 1 lần **có kèm thông báo lỗi validate**. Vẫn hỏng →
   `FAILED` + `SCHEMA_VIOLATION`.
3. Chuẩn hoá theo kiểu khai báo: `date` → ISO-8601; `number` → bỏ ký hiệu tiền tệ, dấu phân cách
   nghìn, xử lý dấu phẩy thập phân kiểu châu Âu; `string` → trim, gộp khoảng trắng.
4. Tính confidence (§7) → dựng `extraction_field`.
5. Chạy luật kiểm tra (§8) → `validation_issue`.
6. Đặt trạng thái: có `ERROR` hoặc có trường `confidence < 0.5` → `NEEDS_REVIEW`, ngược lại
   `SUCCEEDED`.

**Thử lại tối đa 1 lần, không dùng backoff hàm mũ.** Đây là gọi localhost, không phải mạng
Internet chập chờn; retry nhiều lần chỉ làm chậm bộ đo ở Phase 4 chứ không cứu được gì.

---

## 7. Điểm tin cậy

### 7.1 Không hỏi model tự chấm điểm

Cách hay gặp nhất là bắt model tự trả `confidence` cho từng trường. **Không làm.** Điểm đó không
được hiệu chỉnh, model nhỏ gần như luôn trả 0.9–0.95 cho cả trường sai. Tệ hơn: Phase 4 sẽ vạch
trần ngay, và một cột confidence vô nghĩa nằm chình ình trong portfolio thì phản tác dụng.

### 7.2 Chấm bằng tín hiệu xác định được

Ba tín hiệu, tính hoàn toàn bằng code, không tốn thêm lần gọi model nào:

- **GROUNDED** — giá trị (sau chuẩn hoá) xuất hiện nguyên văn trong text nguồn. Với số thì so cả
  vài cách hiển thị (`1250.00` / `1,250.00` / `1.250,00`); với ngày thì thử vài định dạng phổ biến.
- **TYPE_OK** — parse đúng kiểu khai báo.
- **RULE_CONFIRMED** — được một luật nghiệp vụ xác nhận (ví dụ `total` khớp tổng các dòng).

Bảng điểm:

| Tình huống | Điểm | `reason` |
|---|---|---|
| null, và trường nằm trong `required` | 0.00 | `MISSING_REQUIRED` |
| null, trường tuỳ chọn | 0.50 | `ABSENT_OPTIONAL` |
| có giá trị, tìm thấy nguyên văn trong nguồn | 0.90 | `GROUNDED` |
| có giá trị, không nguyên văn, nhưng luật xác nhận | 0.70 | `RULE_CONFIRMED` |
| có giá trị, không nguyên văn, không luật nào đụng tới | 0.40 | `NORMALIZED_NOT_GROUNDED` |
| dính vào một luật `ERROR` | 0.20 | `RULE_VIOLATED` |

Không có 0.87. Sáu bậc, mỗi bậc giải thích được bằng một câu — client hỏi "sao trường này 0.4"
thì trả lời được ngay. Ngưỡng vào hàng đợi rà soát: `< 0.5` (config).

`ABSENT_OPTIONAL` để 0.50 chứ không phải 1.0 là có chủ ý: "không có trong tài liệu" và "model bỏ
sót" nhìn từ ngoài giống hệt nhau, và ta thật sự không biết là cái nào.

### 7.3 Chứng minh nó không phải trang trí

Phase 4 xuất thêm **bảng hiệu chỉnh**: với mỗi bậc confidence, độ chính xác thực đo trên tập có
nhãn.

```
confidence 0.90 → đúng 96%  (n=412)
confidence 0.70 → đúng 81%  (n=88)
confidence 0.40 → đúng 54%  (n=143)
```

Nếu bảng này đơn điệu giảm thì confidence có giá trị thật, và đó là một biểu đồ trong README mà
gần như không portfolio đối thủ nào có. Nếu nó **không** đơn điệu, bảng điểm §7.2 sai và cần
chỉnh — đó cũng là nội dung portfolio, đừng giấu.

*Nâng cấp về sau nếu §7.3 cho thấy chưa đủ:* chạy model 3 lần rồi lấy mức đồng thuận làm điểm
(self-consistency). Đắt gấp 3 nên chỉ làm khi số liệu bảo phải làm.

---

## 8. Luật kiểm tra nghiệp vụ (Phase 5)

**Không xây ngôn ngữ biểu thức.** Vài luật viết sẵn, mỗi luật một class nhỏ, bật/tắt và cấu hình
theo schema qua cột `extraction_schema.rules`:

```json
[
  { "id": "TOTALS_MATCH", "severity": "WARN",
    "params": { "total": "/total", "lines": "/line_items", "amount": "amount", "tolerance": 0.01 } },
  { "id": "DATE_NOT_FUTURE", "severity": "ERROR", "params": { "field": "/invoice_date" } },
  { "id": "REQUIRED_PRESENT", "severity": "ERROR", "params": {} }
]
```

Bộ luật v1: `REQUIRED_PRESENT`, `TOTALS_MATCH`, `DATE_NOT_FUTURE`, `ENUM_VALID`,
`NUMBER_NON_NEGATIVE`. Năm luật, mỗi luật vài chục dòng.

Luật tham chiếu trường bằng JSON Pointer, nên chúng **không biết gì về hoá đơn** — vẫn dùng được
cho hợp đồng hay phiếu xuất kho. Đó là điều kiện để giữ lời hứa "schema do client định nghĩa";
một luật hardcode `invoice.total` sẽ âm thầm phá vỡ nó.

`tolerance` bắt buộc phải có: tài liệu thật làm tròn khác nhau, so bằng `==` trên tiền sẽ báo lỗi
giả liên tục.

---

## 9. Hàng đợi rà soát (Phase 5)

`GET /review?maxConfidence=0.5&schemaId=…&page=0` → các trường cần rà, kèm ngữ cảnh đủ để người
sửa mà không phải mở tài liệu: đường dẫn, giá trị model trả, lý do confidence, tên file, và **đoạn
text nguồn quanh vị trí khớp** nếu có (±200 ký tự).

`PATCH /extractions/{runId}/fields` với `[{path, value}]`:

1. Ghi `corrected_value_json`, `corrected_at`, `corrected_by` vào `extraction_field`.
2. Cập nhật `result_json` tại đúng JSON Pointer đó.
3. Dựng lại projection, chạy lại luật kiểm tra, tính lại trạng thái run.
4. `raw_response` **không đổi** — dấu vết model đã trả gì phải giữ nguyên.

**Vòng phản hồi:** bản sửa của người dùng là ground truth chất lượng cao. Một lệnh
`--export-ground-truth` ghi các run đã rà soát xong ra `eval/ground-truth/` (§10.1), nhập vào tập
đo của Phase 4. Đây chính là chỗ Phase 5 nuôi ngược Phase 4 mà plan nhắc tới.

---

## 10. Bộ đo (Phase 4) ⭐

Phần tạo khác biệt số 2. Chạy **ngoài HTTP**: một `ApplicationRunner` bật bằng profile `eval`,
dùng lại đúng service layer thật (không mock, không đường tắt — đo đường code thật hoặc đừng đo).

### 10.1 Bố cục tập đo — file, không phải DB

```
eval/
  documents/         inv-001.pdf …                 (50 file)
  ground-truth/      inv-001.json …                (nhãn tay)
  schemas/           invoice.v1.json
  reports/           2026-09-14-qwen2.5-7b.md
```

Ground truth là file trong git, không nằm trong DB. Lý do: xem diff được khi sửa nhãn, sửa bằng
editor thường, người lạ clone repo là thấy ngay bạn đo trên cái gì — tính minh bạch đó chính là
thứ làm bảng số liệu đáng tin.

**Ground truth dùng đúng hình dạng của kết quả trích xuất**, không có định dạng riêng:

```json
{ "schema": "invoice@1",
  "fields": { "invoice_number": "INV-2026-0042", "invoice_date": "2026-03-14", "total": 1250.00,
              "line_items": [ { "description": "Consulting", "quantity": 10, "amount": 1250.00 } ] } }
```

Trường thật sự không có trong tài liệu → ghi `null` tường minh. Đây là ca kiểm thử quan trọng
nhất: nó đo model có bịa không, và bịa mới là lỗi nguy hiểm nhất của sản phẩm loại này.

> **Nhắc lại rủi ro số 1 của plan:** gán nhãn 50 tài liệu là việc chán nhất cả lộ trình và là thứ
> dễ bỏ nhất. Chia 10 tài liệu/buổi × 5 buổi. **Không có mục này thì dự án chỉ là thêm một demo
> extraction nữa trên GitHub.**

### 10.2 So khớp

So sau khi chuẩn hoá theo kiểu khai báo trong schema:

- `string` — trim, gộp khoảng trắng, so **không phân biệt hoa thường**.
- `number` — so với sai số tuyệt đối 0.01.
- `date` — quy về ISO rồi so.
- `boolean`, `enum` — so đúng tuyệt đối.
- `array` — **so theo thứ tự chỉ số** ở v1. Lệch độ dài: phần tử thừa tính FP, thiếu tính FN.
  (Ghép cặp tối ưu là việc để sau; hoá đơn thật hiếm khi đảo thứ tự dòng.)

### 10.3 Định nghĩa chỉ số — viết ra để đừng cãi nhau với chính mình sau này

Với mỗi cặp (tài liệu, đường dẫn trường): `g` = nhãn, `p` = dự đoán.

| | | |
|---|---|---|
| `g≠null`, `p≠null`, bằng nhau | **TP** | |
| `g≠null`, `p≠null`, khác nhau | **FP và FN** | đếm cả hai — vừa sai vừa sót |
| `g≠null`, `p=null` | **FN** | bỏ sót |
| `g=null`, `p≠null` | **FP** | **bịa — theo dõi riêng chỉ số này** |
| `g=null`, `p=null` | **TN** | |

Báo cáo:
- Theo từng trường: precision, recall, F1, số mẫu.
- Tổng: micro-average trên mọi trường; **độ chính xác cấp trường** = (TP+TN)/tổng ← con số headline.
- **Khớp toàn tài liệu**: bao nhiêu tài liệu đúng 100% mọi trường. Con số này khắc nghiệt và
  thuyết phục hơn hẳn.
- **Tỉ lệ bịa**: FP trên nền `g=null`, chia tổng số trường. Client quan tâm số này nhất.
- Bảng hiệu chỉnh confidence (§7.3).
- p50/p95 thời gian mỗi tài liệu.

### 10.4 So sánh model

`--models=qwen2.5:7b,llama3.1:8b,mistral:7b` → chạy toàn tập với từng model, một bảng gộp:

```
| model          | field acc | doc exact | hallucination | p50    |
|----------------|-----------|-----------|---------------|--------|
| qwen2.5:7b     |    94.2%  |     62%   |         1.1%  |  6.2s  |
```

Bảng này là thứ dán thẳng vào README và proposal Upwork. Nó tồn tại được là nhờ `extraction_run`
tách khỏi `document` (§2.2) và `model` là cột chứ không phải config toàn cục.

---

## 11. Multi-tenant và bảo mật (Phase 6)

- Key trong header `X-API-Key`. DB **chỉ lưu SHA-256**; key thô hiện đúng một lần lúc tạo.
- Một filter Spring Security phân giải key → tenant, nạp vào một bean request-scoped.
- **Lọc theo tenant bằng tham số tường minh trong mọi phương thức repository**
  (`findByIdAndTenantId`), không dùng Hibernate `@TenantId`. Lý do: bộ lọc tự động là thứ vô
  hình, quên hay không quên đều nhìn giống nhau; tham số tường minh thì đọc code là thấy. Đổi lại
  phải có test cách ly tenant — đã nằm trong danh sách kiểm chứng của plan.
- Tài nguyên của tenant khác → **404, không phải 403**. 403 xác nhận cho kẻ dò biết id đó tồn tại.
- Rate limit: Bucket4j trong bộ nhớ, một bucket mỗi tenant, mặc định 60 req/phút (config).
  Chỉ đúng khi chạy một instance — ghi rõ trong README, đừng để người đọc tự phát hiện.
- File PDF gốc **không phục vụ qua HTTP** ở v1. Chưa ai cần, mà đây là bề mặt path-traversal.

---

## 12. Cấu hình

Chỉ những khoá thật sự cần chỉnh. Bí mật qua biến môi trường, không vào git.

```yaml
extraction:
  model: qwen2.5:7b            # mặc định, override theo từng run
  prompt-version: v3
  max-input-chars: 24000
  retry-attempts: 1
  review-threshold: 0.5
  scanned-pdf-char-threshold: 100   # ký tự/trang tối thiểu để coi là có text layer
storage:
  root: ./data/documents
  max-file-size: 20MB
ratelimit:
  requests-per-minute: 60
```

Bốn ngưỡng (`max-input-chars`, `review-threshold`, `scanned-pdf-char-threshold`, `tolerance` của
luật) là **núm chỉnh, không phải hằng số**. Tài liệu thật sẽ đòi chỉnh chúng, và không con số nào
trong đó đoán đúng được từ bàn giấy.

---

## 13. Kiểm thử

| Loại | Phạm vi |
|---|---|
| Unit | chuẩn hoá giá trị, bảng điểm confidence, từng luật kiểm tra, làm phẳng/dựng lại JSON Pointer, phát hiện PDF scan |
| Integration (Testcontainers) | upload → trích text → trạng thái; cách ly tenant; migration Flyway chạy sạch |
| Contract | schema ngoài tập con §3.1 bị từ chối; magic bytes sai bị từ chối |
| Có Ollama thật | **gắn tag, không chạy mặc định** — cần model đã tải, chậm. Test thường mock `ChatModel`. |
| Bộ đo | không phải test; chạy tay ở Phase 4 |

Ba test đáng viết trước, vì hỏng âm thầm và hỏng đau:
1. **`result_json` sửa xong thì projection `extraction_field` khớp lại** (§2.2).
2. **Tenant A không đọc được tài liệu tenant B** (§11).
3. **Ground truth `null` mà model trả giá trị thì bị tính là bịa**, không phải bỏ qua (§10.3).

---

## 14. Spec ↔ phase

| Phase | Mục spec |
|---|---|
| 0 | §1 stack, xác minh toạ độ Spring AI |
| 1 | §2.2 tenant/document, §5.1 upload+list, §5.3 kiểm tra đầu vào |
| 2 | §2.2 document_text, §4.1, §6 phát hiện PDF scan |
| 3 | §3 schema, §6 lõi trích xuất, §2.2 run + field |
| 4 | §10 toàn bộ, §7.3 bảng hiệu chỉnh |
| 5 | §8 luật, §9 rà soát |
| 6 | §11 tenant/auth, §2.3 `V5__indexes.sql`, §4.3 🎓 đồng thời |
| 7 | §10.4 bảng số liệu vào README |

---

## 15. Quyết định cần bạn chốt

Tôi đã chọn mặc định cho tất cả, nêu ra để bạn bác nếu thấy sai:

1. **`extraction_run` tách khỏi `document`** (§2.2) — lệch so với plan. Cần cho việc so sánh model
   ở Phase 4. → **giữ tách.**
2. **`tenant_id` có từ V1** (§2.1) — lệch so với plan. Tránh migration đau ở tuần 9. → **giữ.**
3. **Bảng `extraction_field` làm projection** thay vì chỉ truy vấn jsonb (§2.2). Đổi lại phải nhớ
   đồng bộ. Phương án khác: chỉ dùng jsonb, hàng đợi rà soát viết SQL khó hơn. → **giữ bảng phẳng.**
4. **Confidence bằng luật xác định** thay vì model tự chấm (§7). → **giữ luật.**
5. **Ground truth là file trong git**, không phải bảng DB (§10.1). → **giữ file.**
6. **Chỉ hoá đơn** cho tập 50 tài liệu Phase 4, không trộn hợp đồng. Trộn thì mỗi loại chỉ còn 25
   mẫu, sai số thống kê quá lớn để bảng số liệu có nghĩa. Tính đa schema vẫn được chứng minh bằng
   một schema thứ hai không cần bộ đo. → **chỉ hoá đơn.**

Điểm còn hở duy nhất tôi không tự lấp được: **§6.1 — API chính xác của Spring AI 2.x để đưa JSON
Schema động vào request Ollama.** Phải xác minh ở Phase 0 bằng docs chính thức. Có đường lui
(gọi thẳng Ollama bằng `RestClient`) nên nó không chặn tiến độ, nhưng đừng để tới tuần 4 mới phát
hiện — kiểm tra bằng một lệnh `curl` tới Ollama ngay trong Phase 0.

---

## 16. Rủi ro kỹ thuật cụ thể

| Rủi ro | Dấu hiệu sớm | Xử lý |
|---|---|---|
| Model nhỏ không giữ nổi schema nhiều trường | JSON hợp lệ nhưng thiếu trường ở cuối | Giảm số trường mỗi lần gọi; tách schema lớn thành 2 lần gọi (chỉ làm khi có số đo) |
| Text PDFBox mất bố cục bảng → line items sai | `line_items` đúng số dòng nhưng cột lệch | Thử chế độ sắp xếp theo vị trí của PDFBox; đo lại bằng bộ đo |
| Tài liệu dài vượt context | `truncated = true` nhiều | Cột đã có từ đầu; chunk là việc để sau |
| Thời gian mỗi tài liệu quá lâu, bộ đo chạy hàng giờ | p50 > 15s | Đo trước trên 5 tài liệu rồi mới chạy đủ 50; giảm số model so sánh |
| Gán nhãn 50 tài liệu bị bỏ dở | hết tuần 6 chưa xong 20 | Rủi ro số 1 của cả dự án. 10 tài liệu/buổi, 5 buổi, không dồn |
```