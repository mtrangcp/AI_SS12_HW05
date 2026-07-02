# Báo Cáo Phân Tích & Review Mã Nguồn Dự Án eKYC Backend

Báo cáo này được thực hiện bởi Technical Lead nhằm đánh giá chất lượng mã nguồn, nhận diện các vấn đề về thiết kế (SOLID), nợ kỹ thuật (Technical Debt), cơ chế xử lý lỗi (Exception Handling) và ghi vết hệ thống (Logging) trong dự án Spring Boot eKYC hiện tại.

---

## 1. Các Vấn Đề Code Smell & Nợ Kỹ Thuật (Technical Debt)

### 1.1. Cơ chế sinh số tài khoản ngân hàng bằng Vòng lặp SELECT (Do-While Loop)
* **Vị trí:** Lớp [AccountServiceImpl.java](file:///c:/Users/84363/.gemini/antigravity-ide/scratch/ekyc-backend/src/main/java/com/abcbank/ekyc/service/impl/AccountServiceImpl.java#L86-L104) tại phương thức `generateUniqueAccountNumber`.
* **Vấn đề:** 
  Hệ thống sử dụng vòng lặp `do-while` để sinh ngẫu nhiên số tài khoản và gọi phương thức `existsByAccountNumber` kiểm tra trùng lặp trực tiếp với cơ sở dữ liệu.
  * **Rủi ro hiệu năng:** Khi dữ liệu phình to (ví dụ: hàng triệu khách hàng), xác suất đụng độ (collision) số ngẫu nhiên tăng cao. Vòng lặp có thể chạy nhiều lần, gửi hàng loạt truy vấn `SELECT` vào Database, gây tắc nghẽn tài nguyên (CPU, Connection Pool) và tăng đáng kể độ trễ (latency) của API.
  * **Cạnh tranh luồng (Race Condition):** Hai luồng xử lý đồng thời có thể sinh ra cùng một số tài khoản chưa tồn tại trong DB tại cùng một thời điểm kiểm tra, dẫn đến lỗi trùng lặp khi ghi vào DB.
* **Giải pháp khuyến nghị:**
  * Thay thế cơ chế sinh số ngẫu nhiên bằng việc sử dụng **Database Sequence** hoặc **Redis Atomic Increment** để tăng giá trị tuần tự một cách an toàn.
  * Hoặc áp dụng thuật toán phân tán như **Snowflake ID** để sinh số định danh duy nhất mà không cần kiểm tra lại cơ sở dữ liệu.

### 1.2. Tranh chấp luồng do sử dụng `java.util.Random`
* **Vị trí:** Khai báo `private final Random random = new Random();` tại [AccountServiceImpl.java](file:///c:/Users/84363/.gemini/antigravity-ide/scratch/ekyc-backend/src/main/java/com/abcbank/ekyc/service/impl/AccountServiceImpl.java#L26).
* **Vấn đề:**
  Trong môi trường Spring Boot, các Service mặc định là Singleton (được chia sẻ giữa nhiều luồng yêu cầu đồng thời). Lớp `java.util.Random` tuy là thread-safe nhưng hoạt động dựa trên cơ chế đồng bộ hóa (Synchronization) ở mức hạt nhân để cập nhật seed. Khi lượng request lớn, các luồng sẽ xảy ra hiện tượng tranh chấp tài nguyên (thread contention), làm chậm tiến trình xử lý.
* **Giải pháp khuyến nghị:**
  * Sử dụng `ThreadLocalRandom.current().nextInt(...)` để tránh tranh chấp luồng.
  * Đối với ứng dụng tài chính cần độ bảo mật cao hơn cho số tài khoản ngẫu nhiên, nên cân nhắc sử dụng `SecureRandom`.

### 1.3. Thiếu Mapper độc lập (Manual Mapping)
* **Vị trí:** Các bước chuyển đổi DTO <-> Entity tại phương thức `register` của [AccountServiceImpl.java](file:///c:/Users/84363/.gemini/antigravity-ide/scratch/ekyc-backend/src/main/java/com/abcbank/ekyc/service/impl/AccountServiceImpl.java#L59-L77).
* **Vấn đề:** 
  Việc ánh xạ dữ liệu được thực hiện thủ công bằng Lombok Builder ngay trong lớp Service. Khi số lượng trường thông tin tài khoản tăng lên hoặc cấu trúc phức tạp hơn, mã nguồn Service sẽ bị phình to bởi các đoạn mã mapping boilerplate.
* **Giải pháp khuyến nghị:**
  * Tích hợp các thư viện tự động ánh xạ như **MapStruct** hoặc **ModelMapper** để tách biệt mã mapping khỏi mã xử lý logic nghiệp vụ.

---

## 2. Vi Phạm Nguyên Lý Thiết Kế SOLID

### 2.1. Vi phạm nguyên lý đơn nhiệm (Single Responsibility Principle - SRP)
* **Chi tiết:**
  Lớp `AccountServiceImpl` hiện đang chịu quá nhiều trách nhiệm khác nhau:
  1. **Kiểm soát nghiệp vụ:** Thực hiện kiểm tra tính hợp lệ về logic của tài khoản (CCCD, Email trùng lặp).
  2. **Sinh định danh số tài khoản:** Tự định nghĩa thuật toán sinh số tài khoản.
  3. **Ánh xạ dữ liệu:** Chuyển đổi qua lại giữa `AccountRequestDTO` -> `Account` -> `AccountResponseDTO`.
  Khi một lớp có nhiều lý do để thay đổi (ví dụ: thay đổi định dạng số tài khoản, thay đổi thư viện ánh xạ, thay đổi nghiệp vụ kiểm tra), nó vi phạm nghiêm trọng nguyên lý SRP.
* **Giải pháp khuyến nghị:**
  * Tách logic sinh số tài khoản ra một Interface/Service riêng, ví dụ: `AccountNumberGenerator`.
  * Tách mã ánh xạ ra lớp Mapper chuyên biệt.

### 2.2. Vi phạm nguyên lý đảo ngược phụ thuộc (Dependency Inversion Principle - DIP)
* **Chi tiết:**
  Lớp `AccountServiceImpl` phụ thuộc trực tiếp vào thư viện cụ thể `java.util.Random` thay vì một abstraction để sinh số tài khoản. Điều này làm giảm tính linh hoạt khi cần thay thế giải thuật sinh số tài khoản trong tương lai (ví dụ chuyển sang sinh số bằng API bên thứ ba hoặc sinh từ Redis).
* **Giải pháp khuyến nghị:**
  * Khai báo một interface `AccountNumberGenerator` và inject nó vào `AccountServiceImpl` thông qua Constructor Injection.

---

## 3. Đánh Giá Cơ Chế Xử Lý Ngoại Lệ (Exception Handling) & Ghi Vết (Logging)

### 3.1. Đánh giá Exception Handling
* **Điểm tốt:** 
  * Đã xây dựng bộ xử lý ngoại lệ tập trung `GlobalExceptionHandler` sử dụng `@RestControllerAdvice` và xử lý tốt lỗi trùng lặp `DuplicateResourceException` cũng như lỗi xác thực định dạng dữ liệu đầu vào.
* **Điểm cần cải thiện:**
  * **Cấu trúc mã lỗi nội bộ (Internal Error Code):** Đối với các hệ thống tài chính/ngân hàng lớn, việc trả về thông điệp lỗi dạng chuỗi text thuần túy cho Client là chưa đủ. Nên định nghĩa một hệ thống mã lỗi nội bộ (ví dụ: `ERR-ACC-001` cho trùng CCCD, `ERR-ACC-002` cho trùng Email). Điều này giúp các hệ thống Client (Frontend, Mobile app) dễ dàng đa ngôn ngữ hóa thông báo lỗi và xử lý kịch bản lỗi tự động chính xác hơn.

### 3.2. Đánh giá Tracking & Logging
* **Điểm tốt:**
  * Đã tích hợp thư viện Logback qua `@Slf4j` và có ghi vết các bước xử lý chính trong luồng đăng ký tài khoản.
* **Điểm cần cải thiện:**
  * **Thiếu mã định danh luồng (Trace ID / Correlation ID):** Hiện tại mỗi dòng log được in ra độc lập. Trong môi trường production chạy đa luồng đồng thời, log của các request sẽ bị đan xen nhau. Nếu không có Trace ID (MDC - Mapped Diagnostic Context) đính kèm trên mỗi request, Tester hoặc Operator sẽ rất khó khăn để lọc ra toàn bộ các bước của duy nhất một lượt đăng ký bị lỗi.
  * **Ghi vết thông tin nhạy cảm (PII Leakage):** Đoạn mã log `log.info("Bắt đầu xử lý đăng ký tài khoản eKYC cho khách hàng: {}", request.getFullName());` ghi nhận trực tiếp họ tên khách hàng dạng clear-text. Tương tự, nếu ghi log trực tiếp số điện thoại, CCCD sẽ vi phạm các tiêu chuẩn bảo mật dữ liệu khách hàng ngành tài chính.
* **Giải pháp khuyến nghị:**
  * Tích hợp **Spring Cloud Sleuth / Micrometer Tracing** hoặc cấu hình một Handler Filter để tự động thêm `Trace-ID` vào MDC trên mỗi request đầu vào.
  * Thiết lập cơ chế **Masking Log** để che giấu bớt các thông tin nhạy cảm của khách hàng (ví dụ: `Nguyen * A`, `098***4321`) trước khi ghi vào log file.
