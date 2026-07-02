# Bài 5: Refactor và hoàn thiện hệ thống với Clean Code & Design Pattern

---

## 1. Prompt 1

Đóng vai là một Technical Lead và Code Reviewer chuyên nghiệp. Hãy đọc toàn bộ mã nguồn của dự án ekyc-spring-boot trong thư mục hiện tại (đặc biệt chú ý đến AccountServiceImpl và AccountController).

Hãy viết một báo cáo phân tích chỉ ra các vấn đề trong mã nguồn hiện tại bao gồm:

Các 'Code Smell' (Mùi code) hoặc Technical Debt (Nợ kỹ thuật) đang tồn tại.

Các điểm vi phạm nguyên lý thiết kế SOLID (nếu có, ví dụ Service đang ôm đồm quá nhiều việc).

Đánh giá về cơ chế xử lý ngoại lệ (Exception Handling) và Tracking/Logging hiện tại.

Hãy trình bày rõ ràng bằng Markdown.

## 2. Prompt 2

Dựa trên phân tích vừa rồi, hãy tiến hành Refactor dự án trực tiếp vào các file trong không gian làm việc của tôi:

Tạo ErrorResponse DTO: Tạo một class ErrorResponse (gồm các trường: timestamp, status, error, message) trong package com.abcbank.ekyc.dto để chuẩn hóa cấu trúc trả về khi có lỗi.

Global Exception Handler: Cập nhật/Tạo class GlobalExceptionHandler (sử dụng @RestControllerAdvice) để bắt và xử lý tập trung 2 loại ngoại lệ:

MethodArgumentNotValidException (Lỗi validation dữ liệu đầu vào từ DTO).

DuplicateResourceException (Lỗi logic nghiệp vụ đã tạo ở Bài 3).
Tất cả ngoại lệ này phải trả về client dưới định dạng JSON của ErrorResponse kèm HTTP Status Code chuẩn xác (400 Bad Request).

Làm sạch Service: Đảm bảo AccountServiceImpl chỉ tập trung xử lý nghiệp vụ, không chứa các logic validation cơ bản của framework.

Hãy tự động ghi (write/apply) các thay đổi này trực tiếp vào các file tương ứng.

## 3. Prompt 3

Tiếp tục Refactor: Hệ thống ngân hàng cần lưu vết rất kỹ. Hãy bổ sung cơ chế Logging bằng SLF4J (sử dụng annotation @Slf4j của Lombok) vào AccountController và AccountServiceImpl.

Yêu cầu ghi log tại các điểm sau:

[INFO] Bắt đầu nhận request đăng ký mở tài khoản mới (ghi log kèm theo số điện thoại hoặc email che giấu một phần - mask data).

[ERROR] Khi có lỗi văng ra (ví dụ trùng lặp dữ liệu).

[INFO] Khi tạo tài khoản thành công và sinh số tài khoản.

Hãy cập nhật trực tiếp mã nguồn của Controller và Service bằng tính năng ghi file của bạn.

## 4. Prompt 4

Hoàn hảo. Nhiệm vụ cuối cùng, hãy lập một bảng tóm tắt bằng Markdown với tiêu đề: 'Báo cáo cải tiến hệ thống (Improvement Summary)'.

Bảng cần so sánh: Mã nguồn trước Refactor vs Sau Refactor đã cải thiện được điều gì. Hãy chia thành các cột: Tiêu chí (Ví dụ: Exception Handling, Logging, Validation, Code Structure), Trước Refactor (Vấn đề tồn tại), Sau Refactor (Giải pháp đã áp dụng), và Lợi ích mang lại.

Hãy trình bày thật chuyên nghiệp để tôi dùng làm tài liệu báo cáo.
