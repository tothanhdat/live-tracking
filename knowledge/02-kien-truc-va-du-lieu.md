# Kiến trúc & Dữ liệu — Horus Eye

> Nguồn: mã nguồn thực tế `Athena/Projects/horus-eye`. Snapshot 27/07/2026. Đối chiếu lại repo `live-tracking` nếu cần trạng thái "hiện tại".

## 1. Kiến trúc FE/BE (Plan A / Plan B)

Horus Eye tách frontend Next.js khỏi backend Go. Next.js chỉ render UI + phục vụ script tracker và **không chạm database** (`FE_NO_DB`); Go sở hữu toàn bộ API, thu thập và truy cập dữ liệu.

```
Trình duyệt khách ──HTTP──► Go (:3001) ──SQL──► PostgreSQL  (control plane + Plan A analytics)
     (script.js)                  │
                                  └──────────► ClickHouse   (Plan B: analytics R/W khi có CLICKHOUSE_URL)

Next.js (:3000) ──✗──► Postgres / ClickHouse     (FE_NO_DB — không truy cập DB)
     │  Dashboard (SSR/CSR) + phục vụ script.js / recorder.js
     └─ Gọi API tới Go qua API_URL tuyệt đối (http://localhost:3001/api)

Cap (:3002) ── CAPTCHA API ──► cap-redis            Redis (chung) ── logout denylist + rate limit
GeoIP: MaxMind GeoLite2 (tại chỗ)   ·   Kafka: KHÔNG dùng
```

### Hai kế hoạch lưu trữ analytics (Plan A / Plan B)

| Plan | Đường dữ liệu | Khi nào |
|---|---|---|
| Plan A | Go → PostgreSQL cho cả control plane lẫn analytics | Local/đơn giản, không đặt `CLICKHOUSE_URL` |
| Plan B | Go → ClickHouse đọc/ghi analytics; Postgres vẫn giữ control plane + định danh session | Production & Docker mặc định (đặt `CLICKHOUSE_URL`) |

Cách Go chọn kho lưu trữ: hàm `UseCH()` kiểm tra `CLICKHOUSE_URL`. Có → analytics và `session_replay` ghi ClickHouse (Plan B); không → dùng Postgres (Plan A). PostgreSQL luôn là control plane (auth, user, workspace, website, session identity) ở cả hai plan.

### Các dịch vụ trong Docker Compose

| Dịch vụ | Cổng | Vai trò |
|---|---|---|
| `horus-eye` (Next.js) | 3000 | Dashboard + phục vụ `script.js`/`recorder.js`. Chạy `FE_NO_DB`, không nhận biến kho dữ liệu |
| `horus-eye-go` (Go) | 3001 | API `/api/*`, thu thập, record; chạy goose migrate lúc khởi động |
| `cap` | 3002 | Dịch vụ CAPTCHA (`cap/backend-go`); admin UI là Horus Eye |
| `cap-redis` | nội bộ | Redis dùng chung: backend Cap + logout denylist + rate limit của Go |
| `db` | 5432* | PostgreSQL 15 — control plane + Plan A analytics |
| `clickhouse` | nội bộ* | ClickHouse 24.8 — Plan B analytics (không publish host mặc định) |

\* Postgres/ClickHouse chỉ publish host port ở stack dev/gỡ lỗi. Dịch vụ Next không bao giờ nhận biến môi trường của kho dữ liệu.

### Cấu trúc backend Go (`go/`)

| Thành phần | Vị trí | Trách nhiệm |
|---|---|---|
| Entry point | `go/cmd/server/main.go` | Khởi động server, load config, chạy migrate |
| Router | `go/internal/server` (chi) | Định tuyến `/api/*`, CORS, probes, middleware auth/rate-limit |
| Handlers | `go/internal/handlers` | send, record, auth (+webauthn), analytics, reports, billing, crypto, capkeys, invites, email… |
| Store | `go/internal/store` | Bộ ghi/đọc SQL & ClickHouse |
| Crypto | `go/internal/crypto` | Token/JWT tương thích byte-for-byte với TS |
| DB | `go/internal/db` | pgx pool; goose SQL dưới `go/migrations/` |
| Khác | — | `capclient`, `capsecret`, `billing`, `mail` (SMTP), `umwebauthn`, `ratelimit`, `authdeny`, `detect` (UA), `query`, `response` |

Routing lưu ý: ở Docker mặc định, trình duyệt gọi thẳng Go qua `API_URL=http://localhost:3001/api` (Next 16 standalone không proxy `/api/v2` đáng tin cậy). `/auth/*` và `/config/*` có thể ở TS backend theo `APP_ROUTE_PATTERNS`; token dùng chung nên login TS hoạt động trên Go. Chi tiết: `docs/direct-go-docker-routing.md`.

## 2. Mô hình dữ liệu

PostgreSQL là nguồn sự thật cho control plane; schema quản lý bằng goose (29 migration, `go/migrations/`).

### PostgreSQL — các bảng chính

| Bảng | Mục đích | Cột / ghi chú chính |
|---|---|---|
| `user` | Tài khoản | `id`, `username` (unique), `password` (bcrypt), `role`, `displayName`, `email`, `logoUrl`, `deletedAt` |
| `team` · `team_user` | Workspace & thành viên | `team`: name, accessCode. `team_user`: teamId, userId, role (`workspace-owner`/`workspace-viewer`). **Tên bảng giữ "team", ngôn ngữ sản phẩm là "workspace"** |
| `website` | Website được tracking | `id`, `name`, `domain`, `teamId`/`userId`, `resetAt`, recorder/replay config, `deletedAt` |
| `session` | Phiên ẩn danh | `id`, `websiteId`, `browser`, `os`, `device`, `screen`, `language`, `country`/`region`/`city`, `distinctId` |
| `website_event` | Sự kiện | `sessionId`, `visitId`, `urlPath`/`Query`, `referrer*`, `utm*`, `gclid`/`fbclid`…, `eventType`, `eventName`, `tag`, Web Vitals (`lcp`/`inp`/`cls`/`fcp`/`ttfb`) |
| `event_data` · `session_data` | Thuộc tính event / phiên | `dataKey`, string/number/date value, `dataType`; `session_data` kèm `distinctId` (identify) |
| `revenue` | Doanh thu theo sự kiện | `sessionId`, `eventId`, `eventName`, `currency`, `revenue` |
| `report` · `segment` · `board` | Báo cáo/bộ lọc/dashboard tùy biến | `type`, `name`, `parameters` (JSON), gắn website / workspace |
| `link` · `pixel` · `share` | Short link, pixel, chia sẻ | `slug` (unique), `url` (link), `entityId` + `shareType` (share) |
| `session_replay` | Ghi phiên (rrweb) | `visitId`, `chunkIndex`, `events` (Bytes), `eventCount`, `started`/`endedAt` |
| `heatmap_event` | Click/scroll cho heatmap | `urlPath`, `eventType`, `x`/`y`, `pageX`/`Y`, `viewport`, `scrollPct` (ghi ở Go đang hoãn) |
| CAP (migration 00021) | Khóa & thống kê CAPTCHA | cap key theo website, secret mã hóa, số liệu verify |
| WebAuthn (00022) | Passkey/MFA | credential, public key, counter theo user |
| Billing (00024–00028) | Gói & giao dịch | `billing_plan`, `workspace_subscription`, `billing_event`, `billing_crypto_price`, `team_crypto_account`, `crypto_deposit` |
| Invite (00029) | Mời thành viên qua email | token, email, workspace, role, hạn dùng |

### ClickHouse (Plan B — khối lượng lớn)

Khi bật `CLICKHOUSE_URL`, analytics event lưu trên ClickHouse (engine MergeTree). Schema forward do goose quản (`go/migrations-clickhouse/`, baseline 00001); greenfield từ `db/clickhouse/schema.sql` khi CH boot lần đầu. Các bảng/khung nhìn chính:

- `website_event`, `event_data`, `session_data` — dữ liệu thô, ORDER BY theo giờ + website + session + visit.
- `website_event_stats_hourly` + `_mv` — tổng hợp thống kê theo giờ (materialized view).
- `website_revenue` + `_mv`; `event_data_pivot` / `session_data_pivot` + `_mv` — tăng tốc báo cáo.
- `session_replay` — chunk replay (Plan B).

**Quy tắc CH migration**: mỗi câu `CREATE TABLE`/`MATERIALIZED VIEW`/`ADD PROJECTION` phải dùng `IF NOT EXISTS` vì migration chạy trên cả môi trường đã có bảng từ trước. Luôn giữ `schema.sql` đồng bộ với forward migration.

### Payload gửi lên `POST /api/send`

```
type: 'event' | 'identify' | 'performance'
payload:
  website (uuid) | link (uuid) | pixel (uuid)
  url · referrer · hostname · title · tag
  screen · language · browser · os · device
  name (event name) · data{} (thuộc tính tùy biến)
  lcp · inp · cls · fcp · ttfb   (Web Vitals)
Trả về: { cache } — token phiên để request sau tái sử dụng
```

## 3. Danh mục Event & dữ liệu thu thập

### Cách nhúng & gửi sự kiện

Bước 1 — nhúng tracker vào `<head>`:
```html
<script defer
  src="https://analytics.example.com/script.js"
  data-website-id="a1b2c3d4-...-uuid"></script>
```

Bước 2 — sự kiện tùy biến, hai cách:
```html
<!-- Cách A: khai báo ngay trên HTML -->
<button data-horus-event="dang-ky" data-horus-event-goi="pro">Đăng ký</button>
```
```js
// Cách B: gọi bằng JavaScript
horus.track('dang-ky', { goi: 'pro', gia: 199000 });
```

### Cấu hình tracker qua data-attributes

| Thuộc tính | Mặc định | Tác dụng |
|---|---|---|
| `data-website-id` | — (bắt buộc) | ID website nhận dữ liệu |
| `data-host-url` | gốc của script | Máy chủ nhận dữ liệu = backend Go |
| `data-auto-track` | true | Tự bắt sự kiện; false để tự quản |
| `data-do-not-track` | false | Tôn trọng cờ DNT của trình duyệt |
| `data-exclude-search` / `-hash` | false | Bỏ query string / hash khỏi URL |
| `data-domains` | — | Chỉ track trên các domain cho phép |
| `data-tag` | — | Gắn nhãn (A/B, môi trường) cho mọi event |
| `data-performance` | false | Bật thu Web Vitals (LCP/INP/CLS/FCP/TTFB) |

### Các loại event (eventType)

| Event | eventType | Khi nào phát sinh | Dữ liệu chính |
|---|---|---|---|
| Pageview | 1 pageView | Tải trang & đổi route SPA | urlPath/Query, referrer, utm/clid, title, tag |
| Custom event | 2 customEvent | `data-horus-event-*` hoặc `horus.track()` | eventName + event_data (theo dataType) |
| Link event | 3 linkEvent | Truy cập qua short link (`/q/{slug}`) | redirect & ghi nhận |
| Pixel event | 4 pixelEvent | Tải tracking pixel (`/p/{slug}`) | pixel slug |
| Performance | 5 performance | Khi bật `data-performance` | lcp, inp, cls, fcp, ttfb |

### Kiểu dữ liệu, toán tử & entity

Mỗi thuộc tính lưu theo `DATA_TYPE`: 1 string · 2 number · 3 boolean · 4 date · 5 array. Bộ lọc (`OPERATORS`) gồm: eq/neq, set/notSet, contains/doesNotContain, regex/notRegex, true/false, gt/lt/gte/lte, before/after.

Heatmap event: 1 click · 2 scroll. Entity chia sẻ: 1 website · 2 link · 3 pixel · 4 board. Lọc bot dựa vào phát hiện user-agent ở ingest (có thể tắt bằng biến môi trường) — không chấm điểm hành vi.

## 4. Bản đồ API (backend Go)

Route thực tế từ `go/internal/server`, phục vụ dưới `/api/*`. Route ghi/đọc đều kiểm quyền server-side; collect & share công khai.

**Công khai — Thu thập, share, health**

| Nhóm | Endpoint |
|---|---|
| Collect | `POST /api/send` · `/api/batch` · `/api/record` · `GET /p/{slug}` (pixel) · `GET /q/{slug}` (link) · `GET /api/websites/{id}/recorder` |
| Share & Config | `GET /api/share/{slug}` · `/api/config` |
| Health | `GET /live` · `/ready` · `/api/heartbeat` |
| Callback | `POST /api/billing/dummy/callback` · `/api/payment/crypto/callback` |
| Invite | `GET /api/invites/{token}` (preview) |

**Auth · Account · MFA**

| Nhóm | Endpoint |
|---|---|
| Auth | `/api/auth/login` · `/register` · `/logout` · `/verify` |
| WebAuthn | `/api/auth/webauthn/register/(begin\|finish)` · `/assert/(begin\|finish)` |
| Me | `/api/me` · `/me/password` · `/me/mfa/reset` · `/me/teams` · `/me/workspaces` · `/me/websites` |
| Invite | `POST /api/invites/accept` |

**Workspace / Team (alias) · Users · Admin**

| Nhóm | Endpoint |
|---|---|
| Workspaces | `/api/workspaces(/join)` · `/api/workspaces/{id}(/users\|/websites\|/links\|/pixels\|/boards\|/invites)` · invite resend/delete |
| Teams (alias) | Bộ tương đương `/api/teams/*` (giữ tương thích) + `/teams/{id}/billing(/events\|/checkout\|/cancel\|/crypto/*)` |
| Users | `/api/users` · `/api/users/{id}(/teams\|/workspaces\|/websites\|/mfa/reset)` |
| Admin | `/api/admin/(users\|teams\|workspaces\|websites\|health)` |
| Admin billing | `/api/admin/billing/plans(/{code})` · `/crypto-prices(/sync\|/{cur}/{net})` · `/crypto-deposits` · `PATCH /admin/teams/{id}/billing` |

**Websites — Analytics (nhóm lớn nhất, cần quyền site)**, prefix `/api/websites/{id}/`

| Nhóm | Endpoint |
|---|---|
| Core | `stats` · `pageviews` · `metrics` · `metrics/expanded` · `values` · `active` · `daterange` · `export` |
| Events | `events` · `events/series` · `events/stats` · `event-data(/events\|/fields\|/properties\|/stats\|/values\|/{id})` |
| Pivot | `event-data-pivot(/array-series\|date-series\|numeric-series\|numeric-stats\|property-series)` · `session-data(/*)` · `session-data-pivot` |
| Sessions | `sessions` · `sessions/stats` · `sessions/weekly` · `sessions/{id}(/activity\|/properties\|/replays)` |
| Revenue | `revenue/stats` · `/chart` · `/metrics` · `/sessions` |
| Replay | `replays` · `replays/{id}` · `replays/saved(/{id})` · `recorder` |
| Quản lý | `segments(/{id})` · `reports` · `shares` · `reset` · `transfer` · `cap-keys(/{id}/rotate-secret\|/stats\|/geo-stats)` |
| Realtime | `GET /api/realtime/{websiteId}` (polling) |

**Reports · Links · Pixels**

| Nhóm | Endpoint |
|---|---|
| Reports | `/api/reports` · `/reports/{id}` + `POST` attribution · breakdown · funnel · goal · journey · performance · retention · revenue · utm · heatmap |
| Links · Pixels | `/api/links(/{id}/shares)` · `/api/pixels(/{id}/shares)` |

Realtime dùng polling `GET /api/realtime/{websiteId}` — không WebSocket. Cả hai bộ route `/teams/*` và `/workspaces/*` cùng tồn tại (workspace là tên sản phẩm mới, team giữ để tương thích).

## 5. Luồng nghiệp vụ chính

**Onboarding chủ website**
1. Đăng nhập (tài khoản admin mặc định sau khi cài) → hệ thống tạo sẵn một workspace mặc định cho user.
2. Tạo website (name + domain) → sinh `websiteId`; website thuộc workspace hiện tại.
3. Copy đoạn nhúng tracker chứa `data-website-id` → dán vào `<head>`.
4. Khách truy cập sinh pageview đầu tiên (gửi tới Go) → xuất hiện trên dashboard.
5. Realtime hiển thị khách đang hoạt động; Overview hiển thị số liệu tổng hợp.

**Vòng đời một event (kỹ thuật)**
1. Tracker tạo payload, gửi `POST /api/send` tới Go qua `sendBeacon` (fallback fetch).
2. Go kiểm bot (user-agent), tôn trọng DNT/cấu hình.
3. Tạo/khôi phục session: `hash(websiteId + IP + user-agent + salt)` → sessionId ẩn danh; phân giải GeoIP + UA; trả về token cache.
4. Ghi `website_event` (+ `event_data`) vào PostgreSQL (Plan A) hoặc ClickHouse (Plan B) tùy `CLICKHOUSE_URL`.
5. Dashboard đọc qua `/api/websites/{id}/*`; Realtime cập nhật bằng polling.

**Phân tích chuyển đổi (Funnel/Goal)**
1. Tạo segment (bộ lọc) & định nghĩa các bước bằng URL/event.
2. Chạy Funnel (`POST /api/reports/funnel`) → xem tỷ lệ rơi từng bước.
3. Đặt Goal; xem Attribution để ghi công nguồn.
4. Lưu thành Report, gắn vào Board để theo dõi định kỳ.

**Xem lại phiên (Session Replay)**
1. Bật replay cho website (`replayEnabled`, `sampleRate`).
2. Tracker nạp thêm `recorder.js` (rrweb), ghi tương tác, gửi theo chunk về `POST /api/record` (`type=record`).
3. Go ghi `session_replay` (Postgres hoặc ClickHouse); danh sách replay của phiên cập nhật.
4. Mở Session Replay để xem lại; lưu phiên quan trọng. (Heatmap write đang hoãn.)

**Nâng cấp gói & thanh toán crypto**
1. Owner vào Settings → Team → Billing, chọn Pro → `POST …/billing/checkout`.
2. Với provider crypto: hệ thống cấp địa chỉ nạp (`…/billing/crypto/address`) gắn với `team_crypto_account`.
3. Khách nạp; gateway gọi `POST /api/payment/crypto/callback` (header `X-CLIENT-SECRET`), trạng thái `FINISHED` → mở khóa Pro một chu kỳ (prepaid).
4. Hết chu kỳ, worker billing hạ về Free; nạp lại để gia hạn (chi tiết: `03-tinh-nang-dac-thu-horus-eye.md`).

## 6. Khả năng mở rộng

Backend Go chạy tốt như một dịch vụ đơn giản, với các "công tắc" mở rộng khi khối lượng tăng:

- **ClickHouse (Plan B)**: chuyển analytics R/W sang cột-store cho dữ liệu event rất lớn — bật bằng `CLICKHOUSE_URL`; Postgres vẫn là control plane.
- **Redis (chung với Cap)**: logout denylist + rate limit; cache lookup — bật bằng `REDIS_URL`.
- **Stateless Go**: không giữ trạng thái phiên cục bộ → scale ngang nhiều instance sau load balancer.
- **Index tối ưu**: composite index trên `website_event`/`session`; materialized view trên ClickHouse tăng tốc báo cáo.
- **FE_NO_DB**: tách frontend khỏi DB → scale UI độc lập với tầng dữ liệu.

Điểm theo dõi khi scale: kích thước `website_event`/`event_data` · độ trễ báo cáo nặng (funnel/journey/retention) · dung lượng `session_replay` (Bytes theo chunk) · hit-rate Redis · độ trễ merge ClickHouse. Kafka không có trong kiến trúc — ingest ghi trực tiếp.

## 7. Bảo mật & Quyền riêng tư

| Lớp bảo vệ | Mô tả |
|---|---|
| Xác thực | Username + password (bcrypt); phiên JWT ký với `APP_SECRET`, bọc AES-256-GCM; logout thu hồi qua denylist `jti` (Redis) |
| WebAuthn MFA | Passkey đăng ký/xác thực (register & assert begin/finish); tự reset MFA, admin reset cho user. Cấu hình `WEBAUTHN_RP_ID`/`ORIGINS` |
| RBAC Workspace | Global admin + workspace-owner/viewer; enforce ở mỗi route và ẩn thao tác trên UI cho viewer |
| Cap CAPTCHA | Proof-of-work chặn bot ở form nhạy cảm; khóa theo website, secret mã hóa |
| Không cookie tracking | Không đặt cookie trên khách website; nhận diện phiên qua hash ẩn danh — không cần consent banner |
| Ẩn danh khách | `sessionId = hash(websiteId + IP + UA + salt)`, salt xoay theo tháng; không lưu IP thô |
| Do Not Track | Tracker không gửi khi bật `data-do-not-track` và khách bật DNT |
| Fail-closed secrets | Secret mặc định của compose bị từ chối ở boot khi `APP_ENV≠development`; `BILLING_PROVIDER=dummy` chỉ chạy ở development |

### Vì sao "không cookie" mà vẫn nhận diện phiên?

```
sessionId = hash( websiteId + địa_chỉ_IP + user-agent + salt )
   • salt đổi theo tháng  → sang tháng, cùng khách = phiên "mới"
   • IP chỉ dùng để hash, KHÔNG lưu lại dạng thô
   • salt khác nhau giữa các website → không cross-site tracking
```

Hệ quả nghiệp vụ: đây là đánh đổi có chủ đích cho "riêng tư". Horus Eye đo tốt trong khung thời gian ngắn (ngày/tuần) nhưng không theo dõi một cá nhân qua nhiều tháng/website — đổi lại độ chính xác "returning visitor" dài hạn thấp hơn công cụ dùng cookie.

### Redis & rate limit (bắt buộc ở prod)

- `REDIS_URL` (dùng chung với Cap) là bắt buộc ở prod compose: nếu thiếu, logout trả 503 và bỏ qua rate limit.
- Rate limit login/register: 5 lần/phút/IP. Khi Redis lỗi lúc auth, kiểm tra thu hồi fail closed (503) — không cho qua.

So với Umami gốc: Horus Eye đã bổ sung MFA (WebAuthn) và CAPTCHA (Cap) — hai lớp mà Umami core không có.

## 8. Hiệu năng

| Khía cạnh | Mục tiêu / Đặc tính |
|---|---|
| Tracker nhẹ | Bản phân phối `script.js` minify + gzip (Rollup + terser), gửi qua `sendBeacon` không chặn tải trang |
| Backend biên dịch | Go binary khởi động nhanh, ít RAM; stateless, scale ngang |
| Ghi event | Một round-trip `/api/send`; batch qua `/api/batch`; trả token cache để tái dùng |
| Truy vấn báo cáo | Composite index (Plan A); ClickHouse MergeTree + materialized view (Plan B) |
| Realtime | Polling `/api/realtime` — đơn giản, đủ dùng; đánh đổi độ trễ lấy sự đơn giản |
| Load test | `make loadtest-peak` (k6) mô phỏng traffic bursty vào `/api/send` |

Cần kiểm chứng: repo không công bố con số throughput chính thức; hiệu năng thực tế phụ thuộc tài nguyên, plan lưu trữ (Postgres vs ClickHouse) và cấu hình triển khai. Nên chạy `loadtest-peak` trên môi trường đích để lấy số.

## 9. Triển khai & Cấu hình

### Mô hình triển khai (Docker Compose)

- **Prod**: `make docker-up` → chạy Next + Go + Cap + cap-redis + Postgres + ClickHouse ở Plan B, chế độ Go, `FE_NO_DB` (mặc định). UI :3000, Go API :3001.
- **Dev (hot-reload)**: `cp .env.example .env` rồi `make docker-up-dev` — `APP_ENV=development`, `ALLOW_REGISTER=1`, `BILLING_PROVIDER=dummy`.
- **Dừng**: `make docker-down`. Không dùng Docker: `make install` → `make dev-go` (Go :3001) → `make api-go-direct` + `make dev` (Next :3000).

Migrate lúc khởi động: `docker-entrypoint.sh` của Go chạy goose Postgres (trừ khi `SKIP_DB_MIGRATE=1`) và goose ClickHouse khi có `CLICKHOUSE_URL` (trừ khi `SKIP_CH_MIGRATE=1`). Thủ công: `make db-migrate` / `make ch-migrate` (+ `*-status`).

### Biến môi trường chính (backend Go)

| Biến | Bắt buộc? | Ý nghĩa |
|---|---|---|
| `DATABASE_URL` | Prod: Có | Kết nối PostgreSQL (control plane). Dev có thể chạy tạm không có (cảnh báo) |
| `APP_SECRET` | Prod: Có | Khóa ký JWT (fallback `DATABASE_URL`). Default compose bị từ chối khi không phải development |
| `APP_ENV` | — | `production` (mặc định) hoặc `development`. Quyết định fail-closed & dummy billing |
| `CLICKHOUSE_URL` | Không | Bật Plan B — analytics + `session_replay` ghi ClickHouse |
| `REDIS_URL` | Prod: Có | Logout denylist + rate limit (chung Cap Redis). Thiếu → logout 503 |
| `ALLOW_REGISTER` | Không | Bật `POST /api/auth/register` công khai. Prod bỏ; dev = 1 |
| `WEBAUTHN_RP_ID` · `_ORIGINS` · `_RP_DISPLAY_NAME` | Không | Cấu hình WebAuthn/passkey MFA |
| `BILLING_PROVIDER` | Không | Trống (tắt checkout) · `dummy` (dev) · `crypto` |
| `APP_PUBLIC_URL` | Nếu billing | Gốc FE cho redirect checkout |
| `CRYPTO_PAYMENT_BASE_URL` · `CRYPTO_CLIENT_KEY` · `_SECRET` | Nếu crypto | Gateway crypto: base URL + header outbound/callback |
| `CAP_ADMIN_KEY` · `CAP_SECRET_ENCRYPTION_KEY` · `CAP_JWT_SECRET` · `CAP_REDIS_*` | Prod: Có | Tích hợp Cap; default bị từ chối ngoài development |
| `API_URL` · `COLLECT_API_ENDPOINT` · `GO_API_URL` | Không | Chọn backend cho FE/tracker (baked vào Next image lúc build) |

### Migration schema (goose, không Prisma migrate)

PostgreSQL dùng goose (`go/migrations/`, 29 file 00001→00029). ClickHouse dùng goose riêng (`go/migrations-clickhouse/`). `prisma/schema.prisma` chỉ để sinh types cho FE — **không chạy** `prisma migrate`. Tạo migration mới = thêm file đánh số kế tiếp bắt đầu bằng `-- +goose Up` (không sửa file đã áp dụng).

## 10. Đọc tiếp

- Bối cảnh nghiệp vụ, stakeholders, FR/NFR: `01-nghiep-vu-va-yeu-cau.md`
- Tính năng đặc thù Horus Eye (Workspace/RBAC, Billing, Crypto, Cap, WebAuthn, email invite): `03-tinh-nang-dac-thu-horus-eye.md`
- Định vị, roadmap, rủi ro, quy ước code, tóm tắt QC: `04-dinh-vi-ke-hoach-va-qc.md`
