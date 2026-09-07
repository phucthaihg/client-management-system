# client-management-system

## 1. Cấu trúc thư mục
```
client-management-system/
├── .github/                           # Cấu hình GitHub Workflows & Templates
│   ├── ISSUE_TEMPLATE/
│   │   ├── feature_request.md
│   │   └── bug_report.md
│   └── workflows/
│       ├── ci.yml                     # Test, lint, typecheck khi mở PR
│       └── deploy.yml                 # Build Docker & deploy
│
├── apps/                              # Các ứng dụng frontend & backend chính
│   ├── web/                           # Next.js (Web Portal cho Agent & Admin)
│   │   ├── src/
│   │   │   ├── app/                   # App Router (Dashboard, Customers, Settings, Admin)
│   │   │   ├── components/
│   │   │   │   ├── dialer/            # Floating Softphone Widget, Dial Pad, DTMF
│   │   │   │   ├── crm/               # Customer Data Table, Profile, Timeline
│   │   │   │   └── ui/                # UI kit (Shadcn/Radix components)
│   │   │   ├── hooks/
│   │   │   │   ├── useTwilioVoice.ts  # Hook quản lý lifecycle của @twilio/voice-sdk
│   │   │   │   └── useScreenPop.ts    # Hook nhận incoming call & popup modal
│   │   │   └── lib/                   # API clients, helpers
│   │   ├── package.json
│   │   └── tsconfig.json
│   │
│   ├── api/                           # Core Backend API (NestJS / Express / FastAPI)
│   │   ├── src/
│   │   │   ├── modules/
│   │   │   │   ├── auth/              # JWT, 2FA, RBAC (Admin vs Agent)
│   │   │   │   ├── tenants/           # Multi-tenancy, tenant resolution middleware
│   │   │   │   ├── customers/         # CRUD khách hàng, JSONB custom fields
│   │   │   │   ├── appointments/      # Quản lý lịch hẹn
│   │   │   │   ├── voice/             # Twilio Token, TwiML Outbound/Inbound, CDR
│   │   │   │   ├── billing/           # Stripe Webhooks, Seat subscriptions, Wallet
│   │   │   │   └── provisioning/      # Twilio Subaccount & Phone number buying API
│   │   │   ├── common/
│   │   │   │   ├── guards/            # TenantGuard, RolesGuard
│   │   │   │   └── interceptors/      # Logging, TenantContextInterceptor
│   │   │   └── main.ts
│   │   ├── package.json
│   │   └── tsconfig.json
│   │
│   └── workers/                       # Background Jobs & Event Processors
│       ├── src/
│       │   ├── queues/                # Cấu hình BullMQ / Redis Queue
│       │   ├── jobs/
│       │   │   ├── ai-summary.job.ts  # Tải audio từ Twilio -> Whisper STT -> LLM
│       │   │   └── sms-reminder.job.ts# Quét lịch hẹn -> Gửi SMS qua Twilio
│       │   └── index.ts
│       └── package.json
│
├── packages/                          # Code dùng chung (Shared Packages)
│   ├── database/                      # Quản lý Database & Migrations
│   │   ├── prisma/                    # hoặc Drizzle / TypeORM
│   │   │   ├── schema.prisma          # Multi-tenant schema với khóa tenant_id
│   │   │   └── migrations/
│   │   └── src/index.ts
│   │
│   ├── types/                         # Shared TypeScript interfaces & DTOs
│   │   ├── customer.ts
│   │   ├── call-log.ts
│   │   └── tenant.ts
│   │
│   └── config/                        # Shared ESLint, Prettier, Tailwind configs
│
├── docs/                              # Lưu trữ tài liệu kỹ thuật & Thiết kế
│   ├── architecture/
│   │   ├── architecture.xml           # File draw.io kiến trúc tổng thể
│   │   └── webrtc-flow.md             # Mermaid diagram luồng cuộc gọi
│   └── database/
│       └── schema.dbml                # File DBML của dbdiagram.io
│
├── docker-compose.yml                 # Chạy PostgreSQL, Redis, Mailpit cục bộ
├── .env.example                       # Biến môi trường mẫu
└── package.json                       # Root workspace config (pnpm-workspace / npm / yarn)
```
