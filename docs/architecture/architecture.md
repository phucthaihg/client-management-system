### **2. Phân tích Chi tiết Từng Thành phần trong Kiến trúc**

#### **Tầng 1: Client Layer (Web Frontend)**

* **Công nghệ đề xuất:** Next.js (React) + Tailwind CSS + Shadcn UI.
* **Thành phần chính:**
* **Twilio WebRTC Client (`@twilio/voice-sdk`):** Chạy trực tiếp trên trình duyệt của Agent, trao đổi luồng âm thanh thời gian thực (RTP/SRTP) với máy chủ Twilio Media.
* **Softphone Floating Widget:** Luôn ghim ở góc ứng dụng, quản lý trạng thái thiết bị (`Ready`, `Busy`, `In-call`), bàn phím số DTMF.
* **Screen Pop Component:** Nhận tín hiệu Inbound call từ WebSocket/Twilio để query tức thì hồ sơ khách hàng.



#### **Tầng 2: API Gateway & Security**

* **Cloudflare WAF / Reverse Proxy:** Bảo vệ chống DDoS, SSL Termination, bảo mật Webhook.
* **Tenant Identification Middleware:** Phân tích domain/subdomain hoặc Header (`X-Tenant-ID` / JWT claims) để xác thực văn phòng trước khi chuyển request vào dịch vụ nội bộ.

#### **Tầng 3: Core Application Services (Backend App)**

* **Công nghệ đề xuất:** Node.js (NestJS) hoặc Go / Python (FastAPI).
* **Voice & Telecom Gateway:**
* Cấp token truy cập (`VoiceGrant`) cho từng Agent.
* Sinh mã TwiML động phản hồi cho Twilio khi có cuộc gọi đi (Outbound) hoặc gọi đến (Inbound).
* Kiểm tra số dư ví (Prepaid Wallet) trước khi cho phép bắt đầu cuộc gọi.


* **Billing & Provisioning Engine:**
* Lắng nghe Stripe Webhooks để cập nhật trạng thái thanh toán.
* Kết nối Twilio REST API để tự động sinh **Twilio Subaccount** và gán số điện thoại doanh nghiệp khi văn phòng mua số mới.



#### **Tầng 4: Hàng đợi bất đồng bộ (Async Queue & Background Workers)**

* **Redis + BullMQ (hoặc Celery):** Tách bạch các tác vụ nặng ra khỏi luồng xử lý web chính.
* **AI Processing Pipeline:**
1. Khi cuộc gọi kết thúc, Twilio bắn webhook `recording_status_callback`.
2. Voice Service đẩy job `{ recordingUrl, callId, tenantId }` vào Redis Queue.
3. Worker kéo file âm thanh về, mã hóa tải lên S3/R2.
4. Worker đẩy qua STT Engine (Deepgram / Whisper) $\rightarrow$ trích xuất transcript.
5. Đẩy transcript qua LLM (GPT-4o mini / Claude) với prompt trích xuất: **Summary**, **Sentiment**, **Action Items**.
6. Lưu kết quả vào DB và bắn thông báo (notification) cho Agent.



#### **Tầng 5: Cơ sở dữ liệu & Lưu trữ (Persistence Layer)**

* **PostgreSQL (Multi-tenant with Row-Level Security):**
* Tách biệt dữ liệu an toàn tuyệt đối: Mỗi bảng nghiệp vụ (`customers`, `call_logs`, `appointments`) đều có cột `tenant_id`.
* Cấu hình RLS tự động filter `tenant_id = current_setting('app.current_tenant_id')`.


* **Encrypted Cloud Storage (S3/R2):**
* Lưu trữ các file audio ghi âm cuộc gọi.
* Áp dụng mã hóa phía máy chủ (SSE-S3 hoặc SSE-KMS) để tuân thủ bảo mật dữ liệu nhạy cảm (HIPAA/GLBA).



---

### **3. Luồng Âm thanh & Tín hiệu (Call Signaling & Media Flow)**

Điểm quan trọng nhất trong kiến trúc này là **âm thanh cuộc gọi không chạy qua Backend của bạn**:

1. **Signaling (Tín hiệu điều khiển):** Trình duyệt gửi lệnh gọi đến Backend $\rightarrow$ Backend tạo TwiML hướng dẫn Twilio.
2. **Media Stream (Âm thanh thoại):** Trình duyệt thiết lập kết nối **Peer-to-Media Server trực tiếp với Twilio Cloud** qua WebRTC. Giúp backend của bạn không bị quá tải băng thông và giảm tối đa độ trễ đàm thoại (latency).
3. **Data Recording (Ghi âm):** Twilio tự động lưu file âm thanh cuộc gọi và bắn Webhook thông báo cho Backend khi cuộc đàm thoại kết thúc.
