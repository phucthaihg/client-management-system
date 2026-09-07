# Tài liệu Thiết kế Cơ sở dữ liệu (Database Schema Documentation)

Tài liệu này mô tả chi tiết cấu trúc các bảng dữ liệu trong hệ thống **Multi-tenant CRM + Twilio PBX + AI Call Summary**. Hệ thống sử dụng cơ sở dữ liệu **PostgreSQL** kết hợp mô hình phân tách đa khách hàng thông qua cột `tenant_id` và cơ chế Row-Level Security (RLS).

---

## 1. Bảng `tenants` (Quản lý Khách hàng / Văn phòng)
Lưu trữ thông tin cấu hình của các doanh nghiệp, văn phòng hoặc tổ chức sử dụng hệ thống phần mềm (Multi-tenant).

| Tên trường (Field) | Kiểu dữ liệu | Ràng buộc (Constraints) | Mô tả chi tiết |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PK, Default: `uuid_generate_v4()` | Mã định danh duy nhất của tenant. |
| `name` | VARCHAR(255) | NOT NULL | Tên công ty hoặc tên văn phòng doanh nghiệp. |
| `slug` | VARCHAR(100) | UNIQUE, NOT NULL | Chuỗi định danh ngắn gọn dùng cho URL hoặc phân tách subdomain (ví dụ: `agency-a`). |
| `subscription_status`| VARCHAR(50) | Default: `'active'` | Trạng thái gói cước (`active`: Đang hoạt động, `past_due`: Quá hạn thanh toán, `canceled`: Đã hủy). |
| `stripe_customer_id` | VARCHAR(255) | NULLABLE | Mã khách hàng tương ứng trên cổng thanh toán Stripe để quản lý hóa đơn. |
| `created_at` | TIMESTAMP | Default: `CURRENT_TIMESTAMP` | Thời điểm khởi tạo bản ghi tenant. |

---

## 2. Bảng `users` (Quản lý Nhân viên & Quản trị viên)
Lưu trữ thông tin tài khoản đăng nhập của nhân viên trực tổng đài (Agent) và Quản trị viên văn phòng (Admin).

| Tên trường (Field) | Kiểu dữ liệu | Ràng buộc (Constraints) | Mô tả chi tiết |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PK, Default: `uuid_generate_v4()` | Mã định danh duy nhất của người dùng. |
| `tenant_id` | UUID | NOT NULL, FK (`tenants.id`) | Khóa ngoại liên kết đến tenant mà user này trực thuộc. |
| `email` | VARCHAR(255) | NOT NULL | Thư điện tử dùng để đăng nhập hệ thống. |
| `password_hash` | VARCHAR(255) | NOT NULL | Mật khẩu đã được mã hóa bằng thuật toán băm (Bcrypt/Argon2). |
| `full_name` | VARCHAR(150) | NOT NULL | Họ và tên đầy đủ của nhân viên/quản trị viên. |
| `role` | VARCHAR(50) | Default: `'AGENT'` | Phân quyền tài khoản (`SUPER_ADMIN`: Quản trị hệ thống, `ADMIN`: Quản lý văn phòng, `AGENT`: Nhân viên tổng đài). |
| `created_at` | TIMESTAMP | Default: `CURRENT_TIMESTAMP` | Thời điểm tạo tài khoản. |

*Chỉ mục duy nhất (Unique Index):* Kết hợp `(tenant_id, email)` để đảm bảo một email không bị trùng lặp trong cùng một tenant.

---

## 3. Bảng `customers` (Quản lý Khách hàng của CRM)
Lưu trữ danh sách khách hàng (leads, contacts) mà các văn phòng đang chăm sóc và thực hiện cuộc gọi.

| Tên trường (Field) | Kiểu dữ liệu | Ràng buộc (Constraints) | Mô tả chi tiết |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PK, Default: `uuid_generate_v4()` | Mã định danh duy nhất của khách hàng. |
| `tenant_id` | UUID | NOT NULL, FK (`tenants.id`) | Khóa ngoại phân lập dữ liệu theo tenant. |
| `full_name` | VARCHAR(150) | NOT NULL | Tên hiển thị của khách hàng. |
| `phone_number` | VARCHAR(50) | NOT NULL | Số điện thoại liên lạc chính chuẩn quốc tế (E.164 format, ví dụ: `+18321234567`). |
| `email` | VARCHAR(255) | NULLABLE | Địa chỉ email liên hệ của khách hàng. |
| `custom_fields` | JSONB | Default: `{}` | Dữ liệu mở rộng dạng JSON tùy biến theo nhu cầu đặc thù của từng ngành nghề kinh doanh. |
| `created_at` | TIMESTAMP | Default: `CURRENT_TIMESTAMP` | Thời điểm thêm mới thông tin khách hàng vào hệ thống. |

*Chỉ mục (Index):* Tạo index trên `(tenant_id, phone_number)` để tối ưu tốc độ tra cứu lịch sử cuộc gọi và hiện thị màn hình thông tin (Screen Pop).

---

## 4. Bảng `phone_numbers` (Quản lý Số điện thoại Twilio DID)
Quản lý kho số điện thoại tổng đài Twilio được gán cho từng văn phòng hoặc phân phối cho nhân viên.

| Tên trường (Field) | Kiểu dữ liệu | Ràng buộc (Constraints) | Mô tả chi tiết |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PK, Default: `uuid_generate_v4()` | Mã định danh bản ghi số điện thoại nội bộ. |
| `tenant_id` | UUID | NOT NULL, FK (`tenants.id`) | Khóa ngoại xác định văn phòng sở hữu số điện thoại này. |
| `phone_number` | VARCHAR(50) | UNIQUE, NOT NULL | Số điện thoại DID thực tế trên hệ thống Twilio. |
| `twilio_sid` | VARCHAR(100) | NOT NULL | Mã định danh cấu hình số điện thoại trên hệ thống Twilio API. |
| `assigned_user_id` | UUID | NULLABLE, FK (`users.id`) | Nhân viên cụ thể được chỉ định độc quyền sở hữu số này (nếu có). |

---

## 5. Bảng `call_logs` (Nhật ký Cuộc gọi / CDR)
Lưu trữ toàn bộ thông tin chi tiết về các cuộc gọi đến (Inbound) và cuộc gọi đi (Outbound) thực hiện qua cổng Twilio.

| Tên trường (Field) | Kiểu dữ liệu | Ràng buộc (Constraints) | Mô tả chi tiết |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PK, Default: `uuid_generate_v4()` | Mã định danh nhật ký cuộc gọi. |
| `tenant_id` | UUID | NOT NULL, FK (`tenants.id`) | Khóa ngoại phân lập dữ liệu theo tenant. |
| `customer_id` | UUID | NULLABLE, FK (`customers.id`) | Khách hàng thực hiện hoặc nhận cuộc gọi. |
| `user_id` | UUID | NULLABLE, FK (`users.id`) | Nhân viên (Agent) trực tiếp tham gia xử lý cuộc gọi. |
| `call_sid` | VARCHAR(100) | UNIQUE, NOT NULL | Mã nhận diện cuộc gọi duy nhất trả về từ hệ thống Twilio (`CallSid`). |
| `direction` | VARCHAR(20) | Check: `inbound`, `outbound` | Hướng cuộc gọi (`inbound`: Khách gọi vào, `outbound`: Nhân viên gọi ra). |
| `duration_seconds`| INT | Default: `0` | Thời lượng đàm thoại thực tế tính bằng giây. |
| `recording_url` | TEXT | NULLABLE | Đường dẫn bảo mật (Presigned URL) trỏ tới file ghi âm cuộc gọi trên S3/R2. |
| `status` | VARCHAR(50) | Default: `'completed'` | Trạng thái cuộc gọi (`completed`, `missed`, `busy`, `no-answer`). |
| `started_at` | TIMESTAMP | Default: `CURRENT_TIMESTAMP` | Thời điểm bắt đầu thiết lập cuộc gọi. |

---

## 6. Bảng `call_summaries` (Bản tóm tắt Cuộc gọi bằng AI)
Lưu trữ kết quả xử lý tự động từ AI (Whisper / LLM) bao gồm bản ghi văn bản, tóm tắt ý chính và việc cần làm sau cuộc gọi.

| Tên trường (Field) | Kiểu dữ liệu | Ràng buộc (Constraints) | Mô tả chi tiết |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PK, Default: `uuid_generate_v4()` | Mã định danh bản ghi tóm tắt AI. |
| `tenant_id` | UUID | NOT NULL, FK (`tenants.id`) | Khóa ngoại phân lập dữ liệu theo tenant. |
| `call_log_id` | UUID | UNIQUE, NOT NULL, FK | Khóa ngoại liên kết 1-1 với bảng cuộc gọi (`call_logs.id`). |
| `transcript` | TEXT | NULLABLE | Toàn bộ nội dung đàm thoại chuyển đổi từ giọng nói sang văn bản (Speech-to-Text). |
| `summary` | TEXT | NULLABLE | Đoạn văn bản ngắn gọn đúc kết nội dung chính của cuộc trò chuyện do AI sinh ra. |
| `sentiment` | VARCHAR(30) | NULLABLE | Đánh giá sắc thái cuộc gọi (`positive`: Tích cực, `neutral`: Bình thường, `negative`: Tiêu cực). |
| `action_items` | JSONB | Default: `[]` | Danh sách các công việc cần làm tiếp theo (Action items) được trích xuất tự động dưới dạng mảng JSON. |
| `created_at` | TIMESTAMP | Default: `CURRENT_TIMESTAMP` | Thời điểm hoàn thành quá trình phân tích AI. |

---

## 7. Bảng `appointments` (Quản lý Lịch hẹn)
Lưu trữ lịch hẹn gặp mặt, tư vấn hoặc gọi lại được thiết lập giữa nhân viên và khách hàng.

| Tên trường (Field) | Kiểu dữ liệu | Ràng buộc (Constraints) | Mô tả chi tiết |
| :--- | :--- | :--- | :--- |
| `id` | UUID | PK, Default: `uuid_generate_v4()` | Mã định danh duy nhất của lịch hẹn. |
| `tenant_id` | UUID | NOT NULL, FK (`tenants.id`) | Khóa ngoại phân lập dữ liệu theo tenant. |
| `customer_id` | UUID | NOT NULL, FK (`customers.id`) | Khách hàng có lịch hẹn. |
| `assigned_user_id` | UUID | NULLABLE, FK (`users.id`) | Nhân viên chịu trách nhiệm thực hiện lịch hẹn. |
| `start_time` | TIMESTAMP | NOT NULL | Thời gian bắt đầu lịch hẹn. |
| `end_time` | TIMESTAMP | NOT NULL | Thời gian kết thúc dự kiến của lịch hẹn. |
| `status` | VARCHAR(50) | Check: `scheduled`, `completed`, `cancelled` | Trạng thái lịch hẹn (`scheduled`: Đã lên lịch, `completed`: Đã hoàn thành, `cancelled`: Đã hủy). |
| `notes` | TEXT | NULLABLE | Ghi chú chi tiết về nội dung cuộc hẹn. |
| `created_at` | TIMESTAMP | Default: `CURRENT_TIMESTAMP` | Thời điểm tạo lịch hẹn. |