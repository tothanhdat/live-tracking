# Nghiệp vụ & Yêu cầu — Horus Eye

> Nguồn: mã nguồn thực tế `Athena/Projects/horus-eye` (README, `go/`, docker-compose, migrations, docs). Snapshot 27/07/2026. Nếu cần thông tin "hiện tại", đối chiếu lại với repo `live-tracking` thay vì chỉ tin tài liệu này.

## 1. Nguồn gốc & định vị

Horus Eye khởi nguồn là **fork của Umami** (umami.is) nhưng đã viết lại toàn bộ backend bằng **Go**. Bản Route Handler TypeScript gốc chỉ còn lưu trong `archive/legacy-ts-api/` để so sánh parity, không còn phục vụ request thật. Đây là hệ thống thực tế đang chạy (Go + Next.js), không phải kế hoạch port.

Khác biệt cốt lõi so với Umami gốc:

- Backend Go (Umami gốc là ứng dụng JavaScript đơn khối).
- Tách FE/BE: Next.js chạy chế độ `FE_NO_DB`, không chạm database.
- ClickHouse là kho phân tích production (Plan B — xem file 02).
- Tích hợp Cap CAPTCHA (không có ở Umami gốc).
- Workspace RBAC + gói billing Free/Pro/Enterprise + thanh toán crypto (không có ở Umami gốc).
- WebAuthn MFA (passkey) & mời thành viên qua email (không có ở Umami gốc).
- Kafka KHÔNG được dùng (đã gỡ khỏi compose).

## 2. Mục tiêu sản phẩm

Horus Eye theo đuổi định vị gốc của Umami — "a simple, fast, privacy-focused alternative to Google Analytics" — và mở rộng thêm năng lực vận hành doanh nghiệp (phân quyền workspace, billing, CAPTCHA, MFA). Ba từ khóa **Đơn giản · Nhanh · Riêng tư** vẫn là kim chỉ nam đánh giá mọi yêu cầu mới, cộng thêm tiêu chí **Sẵn sàng vận hành** (workspace multi-tenant, RBAC, gói Free/Pro/Enterprise, MFA passkey, CAPTCHA).

## 3. Bài toán

Google Analytics đặt dữ liệu trên hạ tầng bên thứ ba, dùng cookie theo dõi, gây khó tuân thủ quyền riêng tư. Horus Eye là giải pháp tự lưu trữ: toàn bộ dữ liệu ở lại hạ tầng khách hàng, không cookie, không PII, kèm bộ báo cáo phong phú (funnel, retention, attribution, journey, revenue) cộng session replay và heatmap.

Tóm tắt một câu cho BA: Horus Eye là "Google Analytics tự host, mã nguồn mở, không cookie" — cộng session replay, heatmap, CAPTCHA, MFA và mô hình workspace + billing.

## 4. Sản phẩm làm được gì (tổng quan luồng)

- Chủ website nhúng một đoạn JavaScript gọn nhẹ (`script.js`) do frontend Next.js phục vụ.
- Khi có khách truy cập, script gửi pageview, custom event, Web Vitals và (tùy bật) session replay về backend Go qua `/api/send`, `/api/record`.
- Dashboard hiển thị: khách realtime, lượt xem, phiên, nguồn truy cập, thiết bị/trình duyệt/OS, quốc gia/vùng/thành phố, UTM, funnel, retention, journey, doanh thu.
- Tổ chức theo workspace (multi-tenant): chia sẻ báo cáo qua share link, gom thành board, tạo short link & tracking pixel, quản lý gói billing và khóa CAPTCHA.

## 5. Đối tượng người dùng (stakeholders)

| Vai trò | Mô tả | Quyền chính |
|---|---|---|
| Global Admin | Quản trị toàn hệ thống (`role: admin`) | Toàn quyền mọi user/workspace/website + `/api/admin/*` (billing plans, crypto prices, deposits, health) |
| User | Người dùng thường (`role: user`) | Được tạo workspace; mọi thực thể thuộc về một workspace |
| Workspace Owner | Chủ sở hữu trong workspace (`workspace-owner`) | Quản lý website/link/pixel/board + thành viên trong workspace; mua Pro |
| Workspace Viewer | Chỉ đọc trong workspace (`workspace-viewer`) | Không có quyền ghi; UI ẩn nút tạo/sửa/xóa |
| Khách truy cập | Người dùng cuối của website — đối tượng đo lường | Không có tài khoản; được bảo vệ bởi không-cookie, hash IP, tôn trọng DNT |

Chi tiết mô hình RBAC 3 tầng và quy tắc workspace: xem file `03-tinh-nang-dac-thu-horus-eye.md`.

## 6. Năng lực hiện có (nhóm theo miền)

Đối chiếu `go/internal/handlers` + `src/app`:

- **Thu thập (Tracker)**: script vanilla JS — auto-pageview, SPA route tracking, custom event qua `data-horus-event-*` và `horus.track()`, Web Vitals (khi bật `data-performance`), gửi qua `sendBeacon`/`fetch`.
- **Ingest (Go)**: `POST /api/send` & `/api/batch` — kiểm bot (theo user-agent), tạo/khôi phục session bằng token cache, phân giải GeoIP + UA, ghi Postgres (Plan A) hoặc ClickHouse (Plan B).
- **Báo cáo tổng quan**: stats (views/visitors/sessions/bounce/time), pageviews time-series, metrics theo chiều (url, referrer, browser, os, device, screen, country/region/city, language, event, tag, hostname).
- **Báo cáo hành vi**: Funnel, Retention, Journey, Attribution, UTM, Goal, Breakdown, Revenue, Performance, Heatmap — `POST /api/reports/*`.
- **Realtime**: `GET /api/realtime/{websiteId}` — khách đang hoạt động & sự kiện gần đây, cơ chế **polling** (không WebSocket).
- **Session Replay**: ghi phiên bằng rrweb (`recorder.js`), lưu chunk qua `POST /api/record` → bảng `session_replay`; liệt kê/lưu phiên; bật theo website.
- **Heatmap**: cấu hình & báo cáo click/scroll đã có ở UI; riêng đường ghi heatmap ở Go **đang hoãn** — `type=heatmap` trả `heatmap_disabled` (đã biết, không phải bug — xem file 04 mục rủi ro).
- **Auth & MFA**: đăng nhập username+password (bcrypt), JWT (bọc AES-GCM), logout denylist qua Redis, rate-limit; WebAuthn/passkey MFA; đăng ký công khai bật/tắt bằng `ALLOW_REGISTER`.
- **Workspace (multi-tenant)**: workspace + thành viên & role (owner/viewer), mời qua access code hoặc email invite; mọi website/link/pixel/board thuộc một workspace.
- **Billing & Crypto**: gói Free/Pro/Enterprise theo workspace; self-serve checkout; thanh toán crypto trả trước (prepaid) với callback gateway.
- **Cap (CAPTCHA)**: dịch vụ Cap tích hợp — quản lý cặp khóa theo website, thống kê & geo-stats; admin UI chính là Horus Eye.
- **Chia sẻ, Board, Links & Pixels**: share link công khai (slug + share_type); board gom báo cáo; short link (`/q/{slug}`) & tracking pixel (`/p/{slug}`); segments & custom report.

### Các màn hình chính (frontend Next.js)

| Màn hình | Đường dẫn | Người dùng làm gì |
|---|---|---|
| Danh sách Website | `/websites` | Xem website, thêm mới, lấy đoạn nhúng tracker |
| Chi tiết Website | `/websites/[id]` | Trung tâm báo cáo: overview, realtime, events, funnel, retention, replay, cap-keys… |
| Dashboard / Boards | `/dashboard` · `/boards` | Tổng hợp nhiều website; board tùy biến kéo-thả |
| Workspaces | `/workspaces` · `/workspaces/[id]` | Tạo workspace, mời thành viên (email/access code), phân quyền owner/viewer |
| Links · Pixels | `/links` · `/pixels` | Quản lý short link & tracking pixel |
| Settings | `/settings` | Hồ sơ, đổi mật khẩu, passkey, quản lý workspace, nâng cấp gói |
| Admin | `/admin` | Chỉ role admin: quản trị toàn hệ thống + giá gói/crypto |

Biến `ENABLE_REPLAYS_AND_HEATMAPS` (`src/lib/constants.ts`) ẩn/hiện điều hướng Replays & Heatmaps trên UI.

## 7. Năng lực nâng cao (vượt analytics tối giản)

Các module phân biệt Horus Eye với công cụ đếm pageview đơn thuần (`go/internal/handlers/reports.go` + `/api/reports/*`):

| Module | Trả lời câu hỏi nghiệp vụ | Endpoint |
|---|---|---|
| Funnel | Khách rơi rụng ở bước nào trong quy trình? | `POST /api/reports/funnel` |
| Retention | Bao nhiêu % khách quay lại sau 1 ngày / 1 tuần? | `POST /api/reports/retention` |
| Journey | Khách đi theo lộ trình nào giữa các trang? | `POST /api/reports/journey` |
| Attribution | Kênh/chiến dịch nào mang lại chuyển đổi? | `POST /api/reports/attribution` |
| Goal | Có đạt mục tiêu (đăng ký, tải…) đã đặt không? | `POST /api/reports/goal` |
| Revenue | Doanh thu đến từ đâu, theo sự kiện/tiền tệ nào? | `POST /api/reports/revenue` + `/websites/{id}/revenue/*` |
| UTM | Chiến dịch marketing nào hiệu quả? | `POST /api/reports/utm` |
| Breakdown | Số liệu đổi thế nào khi cắt nhiều chiều? | `POST /api/reports/breakdown` |
| Performance | Website tải nhanh không (Core Web Vitals)? | `POST /api/reports/performance` |
| Heatmap | Khách click/cuộn đến đâu? | `POST /api/reports/heatmap` — ghi hoãn |

**Ghi chú quan trọng về bot detection**: lọc bot dựa trên phát hiện user-agent ở bước ingest — **không phải chấm điểm hành vi (behavioral) hay ML**. Chấm điểm hành vi nâng cao không có trong lõi analytics; thay vào đó Horus Eye dùng Cap CAPTCHA để chặn bot ở các form nhạy cảm (đăng nhập, đăng ký). Đây là điểm khác với định hướng "bot detection bằng hành vi" từng đặt ra ở giai đoạn dự án trước — cần lưu ý khi research/viết spec để không giả định nhầm sang cơ chế behavioral chưa tồn tại.

## 8. Yêu cầu chức năng (FR)

**FR-1 · Tài khoản, phiên & MFA**
- Đăng nhập/đăng xuất username + password (bcrypt), phát hành JWT; xác minh qua `POST /api/auth/verify`; logout thu hồi token (denylist `jti` qua Redis).
- Đăng ký công khai bật/tắt bằng `ALLOW_REGISTER` (`POST /api/auth/register`); rate-limit login/register 5 lần/phút/IP.
- WebAuthn/passkey MFA: đăng ký (`register/begin|finish`) và xác thực (`assert/begin|finish`); tự reset MFA (`/me/mfa/reset`) và admin reset cho user.

**FR-2 · Workspace & phân quyền**
- CRUD workspace, mời thành viên bằng access code (`/workspaces/join`) hoặc email invite (`/workspaces/{id}/invites`).
- Hai role workspace: owner (toàn quyền quản lý) và viewer (chỉ đọc); global admin quản trị mọi thứ.
- Mọi website/link/pixel/board/cap-key thuộc một workspace; chuyển quyền sở hữu website (`/transfer`).

**FR-3 · Thu thập & nạp dữ liệu**
- Tracker gửi `type ∈ {event, identify, performance}` đến `/api/send`; hỗ trợ batch (`/api/batch`).
- Ingest Go: kiểm bot, tạo session (hash `websiteId + IP + UA + salt`), phân giải GeoIP/UA, ghi Postgres (Plan A) hoặc ClickHouse (Plan B).
- Short link (`GET /q/{slug}`) & tracking pixel (`GET /p/{slug}`) ghi nhận link/pixel event; session replay qua `/api/record`.

**FR-4 · Báo cáo analytics**

| Nhóm | Báo cáo |
|---|---|
| Tổng quan | stats, pageviews time-series, active users, daterange, values |
| Acquisition | Referrers, UTM, Attribution |
| Audience | Devices, browsers, OS, screen, country/region/city, languages, sessions, weekly |
| Behavior | Events, Event-data (pivot), Funnel, Journey, Retention, Goal, Segments |
| Tech Health | Performance (Web Vitals), Heatmap (ghi hoãn) |
| Revenue | Revenue stats/chart/metrics/sessions |

**FR-5 · Billing & thanh toán**
- Xem gói & sự kiện billing của workspace (`/teams|workspaces/{id}/billing`).
- Self-serve checkout Pro (`/billing/checkout`), hủy (`/billing/cancel`); provider `dummy` (dev) hoặc `crypto`.
- Thanh toán crypto: xem tiền tệ hỗ trợ, cấp địa chỉ nạp (`/billing/crypto/currencies|address`); gateway callback `POST /api/payment/crypto/callback`.
- Lưu ý: chi tiết business decision của tính năng crypto (fraud/abuse, tỷ giá, xác nhận giao dịch, hoàn tiền, compliance) chưa chốt — xem `CLAUDE.md` mục Tính năng nhạy cảm trước khi đào sâu spec.

**FR-6 · Quản trị & chia sẻ**
- Admin: quản lý users/workspaces/websites, gói (`/admin/billing/plans`), giá crypto (`/admin/billing/crypto-prices`), audit deposit (`/admin/billing/crypto-deposits`), health.
- Chia sẻ công khai qua share link; board tùy biến; xuất dữ liệu website (`/websites/{id}/export`).
- Quản lý khóa CAPTCHA theo website (`/websites/{id}/cap-keys` + `rotate-secret` + `stats`).

## 9. Yêu cầu phi chức năng (NFR)

| Thuộc tính | Yêu cầu / Hiện trạng | Cơ chế trong code |
|---|---|---|
| Hiệu năng | Tracker nhẹ; truy vấn nhanh trên tập lớn | Index tổng hợp trên `website_event`/`session`; ClickHouse MergeTree + materialized view (Plan B) |
| Mở rộng | Chịu khối lượng lớn khi cần | ClickHouse cho analytics; Redis cache/denylist; backend Go stateless scale ngang. Không dùng Kafka |
| Sẵn sàng | Probe liveness/readiness rõ ràng | `/live`, `/ready` (503 khi thiếu dep bắt buộc), `/api/heartbeat`, `/api/admin/health` |
| Bảo mật | Xác thực JWT, MFA, fail-closed secrets | bcrypt + JWT(AES-GCM); WebAuthn; từ chối secret mặc định khi `APP_ENV≠development`; rate-limit |
| Quyền riêng tư | Ẩn danh, không cookie, tôn trọng DNT | Hash IP+UA với salt xoay; không lưu IP thô; do-not-track |
| Quốc tế hóa | Đa ngôn ngữ giao diện | `next-intl`, `src/i18n`, script kiểm message thiếu |
| Bảo trì | Schema versioned, code phân tầng | Goose migrations (Postgres + ClickHouse); Go internal packages; Biome lint/format |
| Tương thích token | Go & TS dùng chung token | `internal/crypto` byte-for-byte với `src/lib/crypto.ts`; kiểm cross-language qua `compat/compat.js` |
| Kiểm thử | Có unit (Go + FE) + e2e + parity | Go test, Vitest, Playwright, bộ TS/Go parity (`make test-e2e-parity`) |
| Triển khai linh hoạt | Docker-first, nhiều nền tảng | Docker Compose (prod + dev), Podman, Netlify (FE), goose auto-migrate lúc khởi động |

## 10. Đọc tiếp

- Kiến trúc, dữ liệu, API, luồng nghiệp vụ: `02-kien-truc-va-du-lieu.md`
- Các tính năng mới đặc thù Horus Eye (Workspace/RBAC, Billing, Crypto, Cap, WebAuthn, email invite): `03-tinh-nang-dac-thu-horus-eye.md`
- Định vị cạnh tranh, roadmap, rủi ro, quy ước code, tóm tắt QC: `04-dinh-vi-ke-hoach-va-qc.md`
