# TASK: Phase 0 — dựng khung chạy được, xác minh toạ độ Spring AI

## Xong =
`gradle test` exit 0.
Chưa xanh thì CHƯA xong — không được báo cáo hoàn thành.

## File được phép tạo/sửa
- `settings.gradle`
- `build.gradle`
- `src/main/java/app/Application.java`
- `src/main/resources/application.yml`
- `src/test/java/app/ApplicationTests.java`

Không tạo file nào khác. `README.md` và `SPEC.md` là CHỈ ĐỌC.

## Đã chốt — không hỏi lại, không tự đổi
- Java 21 toolchain. Gradle **Groovy DSL** (`build.gradle`, không phải `.kts`).
- Spring Boot 3.x qua plugin `org.springframework.boot` + `io.spring.dependency-management`.
- Dependency, đúng ba cái: `spring-boot-starter-web`, `spring-ai-starter-model-ollama`,
  `spring-boot-starter-test`. Spring AI cần import BOM `spring-ai-bom`.
- CHƯA thêm PostgreSQL, Flyway, PDFBox, Testcontainers, Lombok. Đó là phase sau.
- `application.yml` chỉ có `spring.application.name` và
  `spring.ai.ollama.base-url: http://localhost:11434`.
- Test duy nhất: `@SpringBootTest` với một hàm rỗng, kiểm tra context load được.

## Việc
1. Tạo `settings.gradle` (`rootProject.name = 'doc-extraction-api'`) và `build.gradle`.
2. Tạo `Application.java` — chỉ `@SpringBootApplication` + `main`. Không controller, không bean nào.
3. Tạo `application.yml` và `ApplicationTests.java`.
4. Chạy `gradle test`. Fail thì đọc log, sửa, chạy lại.

## Không làm
- Không tạo Controller, Service, Entity, Repository — Phase 0 không có endpoint nào.
- Không tạo Dockerfile hay docker-compose.
- Không thêm dependency ngoài ba cái đã chốt, kể cả khi thấy "có vẻ cần".
- Không sửa `README.md` hay `SPEC.md`.

## Bí thì
- Gradle không resolve được `spring-ai-starter-model-ollama`: **đó chính là việc của task này**.
  Đọc kỹ thông báo lỗi, sửa version BOM hoặc repository, thử lại. Tối đa 3 vòng.
- Lỗi compile / cú pháp: tự sửa, không hỏi Claude.
- Sau 3 vòng `gradle test` vẫn fail → `ask_claude`, dán NGUYÊN log gradle.
- Muốn đổi thứ trong "Đã chốt" → `ask_claude`, cấm tự quyết.
