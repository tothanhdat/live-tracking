# Tính năng đặc thù Horus Eye (khác Umami gốc)

> Nguồn: mã nguồn thực tế `Athena/Projects/horus-eye`. Snapshot 27/07/2026. Đây là 5 mảng làm nên khác biệt của Horus Eye so với Umami gốc — không có trong core Umami. Đối chiếu lại repo `live-tracking` nếu cần trạng thái "hiện tại".

## 1. Workspace & RBAC

Mô hình phân quyền workspace — thay thế mô hình team 4 role của Umami. Nguồn: `docs/workspace-rbac-design.md`, migration `00023_workspace_rbac.sql`, `src/lib/constants.ts`.

### Ba tầng quyền

| Tầng | Role | Phạm vi |
|---|---|---|
| Toàn hệ thống | `admin` | Toàn quyền (all) mọi user/workspace/website + admin billing |
| Người dùng | `user` | Được tạo workspace; mọi thực thể thuộc một workspace |
| Trong workspace | `workspace-owner` | Quản lý website/link/pixel/board/cap-key + thành viên; mua Pro |
| Trong workspace | `workspace-viewer` | Chỉ đọc; UI ẩn nút tạo/sửa/xóa |

**Thay đổi so với Umami**: bốn role team cũ (team-owner/manager/member/view-only) được gộp còn hai: owner/manager/member → `workspace-owner`, view-only → `workspace-viewer`. Không còn quyền sở hữu cá nhân — mọi thực thể sống trong workspace. Tên bảng DB vẫn là `team`/`team_user`; ngôn ngữ sản phẩm & API path là "workspace".

### Quy tắc cốt lõi

- Đăng ký tạo workspace: `CreateUser` tạo transaction một workspace + membership `workspace-owner` mặc định.
- Nhiều owner: owner có thể thêm/sửa/gỡ owner khác — nhưng **chặn gỡ owner cuối cùng**.
- Đăng nhập vào workspace dùng gần nhất (`LAST_WORKSPACE_CONFIG`), nếu không thì workspace đầu/mặc định.
- Route: `/api/workspaces/*` là chính; `/api/teams/*` giữ làm alias tương thích. UI `/teams` redirect sang `/workspaces`.

Trạng thái: theo `docs/workspace-rbac-design.md`, thiết kế đã tới Phase 3 (UI/API alias workspace; bỏ chế độ cá nhân). Xem thêm `docs/workspace-rbac-release-notes.md`.

## 2. Billing (Free / Pro / Enterprise)

Billing gắn với workspace (`team` row) trên Go + Postgres. Nguồn: README.md (Workspace billing), migration `00024_workspace_billing.sql`, `go/internal/handlers/billing.go`.

### Các gói

| Gói | Số website | Retention | Checkout |
|---|---|---|---|
| Free | 1 | 30 ngày | — |
| Pro | 20 | 90 ngày | Self-serve |
| Enterprise | Không giới hạn | Không giới hạn | Admin cấp |

Owner nâng cấp tại Settings → Team → Billing. Bảng dữ liệu: `billing_plan`, `workspace_subscription`, `billing_event`.

### Provider (`BILLING_PROVIDER`)

| Giá trị | Hành vi |
|---|---|
| trống / unset | Self-serve checkout tắt (fail closed) |
| `dummy` | Chỉ dev (`APP_ENV=development`): checkout giả tại `/billing/dummy-checkout`, callback `/api/billing/dummy/callback` |
| `crypto` | UI nạp tiền tại `/billing/crypto-deposit`; Pro mở khi gateway trả `FINISHED` (xem mục 3) |

Endpoint billing: `GET …/billing`, `…/billing/events`; `POST …/billing/checkout`, `…/billing/cancel`. Admin: `/api/admin/billing/plans` (sửa hạn mức gói), `PATCH /admin/teams/{id}/billing` (đặt gói cho workspace).

## 3. Thanh toán Crypto

> ⚠️ **Tính năng nhạy cảm** (liên quan tiền thật, giao dịch on-chain). Theo `CLAUDE.md` của dự án: hiện chưa có yêu cầu bảo mật/compliance cụ thể nào được chốt cho tính năng này. Khi research, viết spec, hoặc breakdown ticket cho crypto — cần làm rõ kỹ các business decision quan trọng (fraud/abuse, tỷ giá, xác nhận giao dịch, hoàn tiền, compliance) trước, **không tự suy diễn thay quyết định business**. Mục dưới đây chỉ mô tả cơ chế đã có trong code, không phải quyết định business đã chốt.

Cổng thanh toán crypto ngoài, mô hình **trả trước (prepaid)**. Nguồn: README.md, `docs/payment/crypto.md`, migrations 00025–00028, `go/internal/handlers/crypto_billing.go`.

### Cơ chế prepaid

- Một lần nạp đủ điều kiện = một chu kỳ Pro (`period_days`, mặc định 30). Nạp lại gia hạn chu kỳ.
- Không auto-renew: subscription lưu dạng prepaid (canceling đến hết chu kỳ), sau đó worker billing hạ về Free.
- Giá cố định theo tiền tệ/mạng trong `billing_crypto_price` (sửa tại Admin → Plans → Crypto deposit prices hoặc SQL); có cột `enabled` (00028).
- API địa chỉ nạp cần id user dạng số → cấp một id/workspace trong `team_crypto_account` (00027) và ánh xạ ngược khi callback.

### Luồng & endpoint

| Bước | Endpoint |
|---|---|
| Xem tiền tệ hỗ trợ | `GET /api/teams/{id}/billing/crypto/currencies` |
| Cấp địa chỉ nạp | `GET /api/teams/{id}/billing/crypto/address` |
| Gateway callback | `POST /api/payment/crypto/callback` (header `X-CLIENT-SECRET`) |
| Audit (admin) | `GET /api/admin/billing/crypto-deposits` — liệt kê `crypto_deposit` |
| Giá crypto (admin) | `GET/POST /admin/billing/crypto-prices(/sync)` · `PATCH …/{cur}/{net}` |

Cấu hình bắt buộc khi crypto: `CRYPTO_PAYMENT_BASE_URL`, `CRYPTO_CLIENT_KEY`, `CRYPTO_CLIENT_SECRET`. Mô phỏng cục bộ: `./scripts/simulate-crypto-callback.sh --user-id 1 --status FINISHED`. Unit test tiền: `cd go && go test ./internal/handlers/ ./internal/billing/ -run 'Crypto|Worker'`.

## 4. Cap (CAPTCHA)

Cap là dịch vụ CAPTCHA proof-of-work tích hợp — thành phần Horus Eye bổ sung, **không có trong Umami gốc**. Nguồn: thư mục `cap/`, migration `00021_cap_integration.sql`, `go/internal/handlers/capkeys.go`, `go/internal/capclient`.

### Vai trò

- Sinh thử thách mà trình duyệt khách phải giải → tạo token; server xác minh token (`siteverify`) bằng secret key.
- Mục đích: chặn bot / lạm dụng ở luồng nhạy cảm (đăng nhập, đăng ký, form công khai).
- Chạy như dịch vụ riêng (`cap/backend-go`, cổng 3002) với Redis riêng (`cap-redis`); admin UI chính là Horus Eye.

### Quản lý khóa theo website

| Chức năng | Endpoint (prefix `/api/websites/{id}/`) |
|---|---|
| CRUD cap key | `GET/POST cap-keys` · `GET/POST cap-keys/{capKeyId}` · `DELETE …` |
| Xoay secret | `POST cap-keys/{capKeyId}/rotate-secret` (secret hiện một lần) |
| Thống kê | `GET cap-keys/{capKeyId}/stats` · `/geo-stats` |

Bảo mật khóa: secret key mã hóa bằng `CAP_SECRET_ENCRYPTION_KEY`; site key công khai (ở widget), secret chỉ ở server.

Hai chế độ tích hợp (chi tiết QC ở file `04-dinh-vi-ke-hoach-va-qc.md`): **Widget** — gắn `<cap-widget>` vào `{capUrl}/{siteKey}/`, khách giải → nhận token; **Programmatic** — gọi `new Cap({apiEndpoint}).solve()`, không cần ô hiển thị. Verify luôn qua proxy server-side (`/cap/siteverify`) — secret không bao giờ ra trình duyệt.

## 5. WebAuthn MFA & Mời thành viên qua Email

Hai bổ sung vận hành gần đây. Nguồn: migrations `00022_webauthn_mfa.sql`, `00029_user_email_team_invite.sql`; handlers `webauthn.go`, `invites.go`, `email.go`; package `umwebauthn`, `mail`.

### WebAuthn / Passkey (MFA)

- Đăng ký passkey: `POST /api/auth/webauthn/register/begin` → `/finish`.
- Xác thực bước 2: `POST /api/auth/webauthn/assert/begin` → `/finish`.
- Tự đặt lại MFA: `POST /api/me/mfa/reset`; admin đặt lại cho user: `POST /api/users/{id}/mfa/reset`.
- Cấu hình: `WEBAUTHN_RP_ID`, `WEBAUTHN_ORIGINS`, `WEBAUTHN_RP_DISPLAY_NAME`.

### Mời thành viên qua Email

- Owner tạo lời mời email: `POST /api/workspaces/{id}/invites` (và alias `/teams/{id}/invites`).
- Gửi lại / thu hồi: `POST …/invites/{inviteId}/resend` · `DELETE …/invites/{inviteId}`.
- Người nhận xem trước bằng token công khai: `GET /api/invites/{token}`; chấp nhận: `POST /api/invites/accept`.
- Gửi email qua package `mail` (SMTP) — cấu hình SMTP truyền vào Go qua compose.

**Ý nghĩa nghiệp vụ**: workspace giờ có thể mời cộng tác viên bằng email thật (không chỉ access code), và bảo vệ tài khoản bằng passkey — hai yêu cầu phổ biến của khách hàng doanh nghiệp mà Umami core không đáp ứng sẵn.

## 6. Đọc tiếp

- Bối cảnh nghiệp vụ, stakeholders, FR/NFR: `01-nghiep-vu-va-yeu-cau.md`
- Kiến trúc, dữ liệu, API, luồng nghiệp vụ: `02-kien-truc-va-du-lieu.md`
- Định vị, roadmap, rủi ro, quy ước code, tóm tắt QC (bao gồm case test Cap/Billing/Crypto/WebAuthn/Invite): `04-dinh-vi-ke-hoach-va-qc.md`
