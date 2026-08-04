# TASK: Phase 1a — Postgres, migration V1, entity Document

Phase 0 đã xong (Boot 4.1.0 + Spring AI 2.0.0, `gradle test` xanh). Task này chỉ làm tầng dữ liệu.
Không viết controller, không viết endpoint — đó là Phase 1b.

## Xong =
`gradle test` exit 0. **Docker Desktop phải đang chạy** (test dùng Testcontainers).
Chưa xanh thì CHƯA xong.

## File được phép tạo/sửa
- `build.gradle` (chỉ thêm dependency)
- `src/main/resources/application.yml`
- `src/main/resources/db/migration/V1__init.sql`
- `src/main/java/app/document/Document.java`
- `src/main/java/app/document/DocumentStatus.java`
- `src/main/java/app/document/DocumentRepository.java`
- `src/test/java/app/PostgresTestBase.java`
- `src/test/java/app/ApplicationTests.java` (sửa để kế thừa base)
- `src/test/java/app/document/DocumentRepositoryTest.java`

`README.md`, `SPEC.md`, `settings.gradle` là CHỈ ĐỌC.

## Đã chốt — không hỏi lại, không tự đổi
- Dependency thêm vào, không kèm version (BOM của Boot quản hết):
  `spring-boot-starter-data-jpa`, `org.flywaydb:flyway-core`,
  `org.flywaydb:flyway-database-postgresql`, `runtimeOnly 'org.postgresql:postgresql'`,
  `testImplementation 'org.springframework.boot:spring-boot-testcontainers'`,
  `testImplementation 'org.testcontainers:junit-jupiter'`,
  `testImplementation 'org.testcontainers:postgresql'`.
- `spring.jpa.hibernate.ddl-auto: validate`. **Flyway tạo bảng, Hibernate không được tạo.**
- `V1__init.sql` đúng ba bảng, tên cột snake_case:
  - `tenant(id uuid pk, name text unique not null, created_at timestamptz not null default now())`
  - `api_key(id uuid pk, tenant_id uuid not null references tenant, key_hash char(64) unique not null,
    label text, created_at timestamptz not null default now(), last_used_at timestamptz, revoked_at timestamptz)`
  - `document(id uuid pk, tenant_id uuid not null references tenant, filename text not null,
    size_bytes bigint not null, content_type text, sha256 char(64) not null, storage_path text not null,
    page_count int, status text not null, failure_reason text, version bigint not null default 0,
    created_at timestamptz not null, updated_at timestamptz not null)`
  - `UNIQUE (tenant_id, sha256)` trên `document`.
  - Cuối file: `INSERT` một tenant mặc định, id cố định
    `00000000-0000-0000-0000-000000000001`, name `default`.
- `DocumentStatus` là enum: `PENDING, PROCESSING, TEXT_READY, UNSUPPORTED, FAILED`.
  Ánh xạ bằng `@Enumerated(EnumType.STRING)`, cột kiểu text.
- `version` dùng `@Version` (optimistic locking). `tenant_id` là cột `UUID` thẳng trong
  `Document`, **không tạo entity `Tenant`, không `@ManyToOne`** — Phase 1 chưa đọc bảng tenant.
- `PostgresTestBase`: một `@Container static PostgreSQLContainer<>("postgres:17-alpine")` kèm
  `@ServiceConnection`. Cả `ApplicationTests` và `DocumentRepositoryTest` kế thừa nó, dùng chung
  một container cho cả lượt test.

## Việc
1. Thêm dependency vào `build.gradle`, thêm cấu hình JPA/Flyway vào `application.yml`.
2. Viết `V1__init.sql`.
3. Viết `DocumentStatus`, `Document`, `DocumentRepository` (`extends JpaRepository<Document, UUID>`).
4. Viết `PostgresTestBase`, sửa `ApplicationTests` kế thừa nó.
5. `DocumentRepositoryTest` — đúng hai test:
   - lưu một `Document` rồi đọc lại, `status` và `sha256` khớp;
   - lưu hai document cùng `tenant_id` + cùng `sha256` phải ném exception (ràng buộc UNIQUE).
6. Chạy `gradle test`. Fail thì đọc log, sửa, chạy lại.

## Không làm
- Không viết Controller, Service, DTO, endpoint nào.
- Không thêm Lombok, MapStruct, QueryDSL.
- Không viết `V2__*.sql` — Phase 2 mới cần.
- Không tạo `docker-compose.yml`; Testcontainers tự lo container cho test.

## Bí thì
- `ddl-auto: validate` báo lệch giữa entity và bảng: sửa cho khớp `V1__init.sql`, **không** đổi
  sang `update` hay `create-drop` để né.
- Testcontainers không khởi động được: kiểm tra Docker Desktop đang chạy. Đó là điều kiện đầu vào,
  không phải lỗi để sửa bằng code.
- Lỗi compile / cú pháp: tự sửa, tối đa 3 vòng.
- Sau 3 vòng `gradle test` vẫn fail cùng một lỗi → `ask_claude`, dán NGUYÊN log gradle.
- Muốn đổi thứ trong "Đã chốt" → `ask_claude`, cấm tự quyết.
- Cấm ghi lại một file với nội dung y hệt lần trước.
