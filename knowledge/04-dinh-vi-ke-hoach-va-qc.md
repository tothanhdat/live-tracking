# Định vị, Kế hoạch & QC — Horus Eye

> Nguồn: mã nguồn thực tế `Athena/Projects/horus-eye` + kế hoạch QC (`horus-eye-qc.html`). Snapshot 27/07/2026. Đối chiếu lại repo `live-tracking` nếu cần trạng thái "hiện tại". Chi tiết đầy đủ 103 test case: xem file gốc `Kế hoạch QC / Kiểm thử — Horus Eye.html` (đính kèm gốc, chưa đưa hết vào knowledge để tránh quá dài — mục 4 dưới đây chỉ tóm tắt).

## 1. So sánh cạnh tranh

Định vị so với các giải pháp analytics phổ biến (định tính, mang tính tham khảo — **cột đối thủ là kiến thức chung thị trường, không trích từ mã nguồn**, cần kiểm chứng khi dùng cho quyết định).

| Tiêu chí | Horus Eye | Google Analytics | Plausible | Matomo |
|---|---|---|---|---|
| Self-hosted / Open source | Có | Không | Có | Có |
| Privacy-first (no cookie) | Có | Hạn chế | Có | Tùy chọn |
| Session replay | Có (rrweb) | Không | Không | Có |
| Heatmap | Cấu hình có, ghi hoãn | Không | Không | Có |
| Funnel / Retention / Journey | Có | Có | Cơ bản | Có |
| MFA (passkey) / CAPTCHA | Có (WebAuthn + Cap) | Có (đám mây) | Cơ bản | Plugin |
| Billing/gói tích hợp | Có (Free/Pro/Ent + crypto) | — | Cloud | Cloud |

## 2. Lộ trình phát triển

Diễn giải theo năng lực đã có trong codebase và các hướng mở rộng (tham khảo, dựa trên docs/migration trong repo):

- **Nền tảng analytics (Đã có)**: Tracker + `/api/send`; báo cáo tổng quan & theo chiều; Postgres control plane; migrations goose.
- **Backend Go & FE/BE split (Đã có)**: viết lại API/collect bằng Go; Next.js `FE_NO_DB`; ClickHouse Plan B; token tương thích TS/Go; bộ e2e parity.
- **Bảo mật & vận hành (Đã có)**: Cap CAPTCHA (00021), WebAuthn MFA (00022), Workspace RBAC (00023), Redis denylist + rate limit.
- **Monetization (Đang hoàn thiện)**: Billing Free/Pro/Enterprise (00024), thanh toán crypto (00025–00028), email invites (00029).
- **Hoàn thiện replay/heatmap**: port đường ghi heatmap ở Go (hiện `heatmap_disabled`) và playback replay chunk.
- **Doanh nghiệp nâng cao**: Alerting, scheduled report, audit log, SSO/SAML — chưa có trong core.

## 3. Rủi ro & Giới hạn

| Rủi ro / Giới hạn | Mô tả | Mức |
|---|---|---|
| Heatmap write chưa hoàn thiện | `POST /api/record type=heatmap` trả `heatmap_disabled`; playback replay chunk out-of-scope ở Go | Trung bình |
| Phụ thuộc Redis ở prod | Thiếu `REDIS_URL` → logout 503, bỏ rate limit; Redis lỗi lúc auth → fail closed 503 | Trung bình |
| Lọc bot theo user-agent | Không chống bot giả UA thật; bù lại có Cap CAPTCHA cho form nhạy cảm | Trung bình |
| Realtime bằng polling | Không WebSocket → độ trễ & tải phụ thuộc tần suất poll | Thấp |
| Báo cáo nặng trên Postgres (Plan A) | Funnel/journey/retention trên dataset lớn có thể chậm nếu không bật ClickHouse (Plan B) | Trung bình |
| Dung lượng session replay | `session_replay` lưu Bytes theo chunk → tăng nhanh khi bật rộng | Trung bình |
| Đồng bộ CH `schema.sql` ↔ goose | Phải giữ `db/clickhouse/schema.sql` khớp forward migration; `IF NOT EXISTS` bắt buộc | Thấp |
| Cấu hình secret nhạy cảm | Prod yêu cầu nhiều secret (`APP_SECRET`, `CAP_*`, `POSTGRES_PASSWORD`…); sai/thiếu → boot fatal | Thấp |

## 4. Hướng phát triển tương lai

Các đề xuất nâng cao giá trị (chưa có trong core — cần xác nhận với đội sản phẩm trước khi đưa vào spec):

- Hoàn thiện Heatmap/Replay: port đường ghi heatmap ở Go & playback chunk replay đầy đủ.
- Alerting & Anomaly: cảnh báo ngưỡng traffic/lỗi qua email/webhook; phát hiện bất thường.
- Scheduled report: gửi báo cáo định kỳ tự động (hiện chỉ export dữ liệu website).
- SSO doanh nghiệp: SAML/OIDC, audit log (đã có nền MFA WebAuthn).
- Realtime WebSocket: thay polling để giảm trễ & tải.
- Public API & BI: API công khai ổn định cho tích hợp data warehouse / BI.

## 5. Đóng góp, CI/CD & Quy ước

Trích từ `CONTRIBUTING.md`, `Makefile`, `biome.json`, `.github/`.

### Công cụ & lệnh (Makefile — dự án dual-stack)

| Việc | Lệnh |
|---|---|
| Cài đặt | `make install` |
| Dev FE / BE | `make dev` (Next) · `make dev-go` (Go) |
| Build | `make build` (full) · `build-tracker` · `build-recorder` · `build-geo` |
| Test | `make test` (Vitest) · `test-go` · `test-e2e` (Playwright) · `test-e2e-parity` (TS/Go) |
| Lint/format | `make lint` · `format` · `check` (Biome) |
| DB | `make db-migrate` · `ch-migrate` · `db-generate` (Prisma client, types-only) |
| Docker | `make docker-up` · `docker-up-dev` · `docker-down` |

### Quy ước code

- FE/TS: Biome (không ESLint/Prettier); dòng 100 ký tự, 2 space, nháy đơn, trailing comma all; alias `@/*` → `./src/*`; quản lý gói pnpm (workspace).
- Go: package theo miền dưới `go/internal`; router chi; test đặt cạnh code (`*_test.go`).
- Kiểm thử: Vitest dùng `test` (không `it`), render qua `@/test/render`, test id `data-test`; MSW mock API.

### CI/CD & điều kiện merge

- Workflow trong `.github/workflows/` chạy lint + test (Vitest + Go) + build; PR chỉ hợp lệ khi test xanh + build thành công.
- Bộ parity (`test-e2e-parity`) so sánh hành vi TS vs Go để đảm bảo Go không lệch khỏi bản tham chiếu.
- Đóng gói Docker image cho cả Next và Go; migrate goose chạy tự động lúc container khởi động.

## 6. QC / Kiểm thử — tóm tắt

Toàn cảnh đảm bảo chất lượng. Chi tiết đầy đủ 103 case nằm ở file gốc `Kế hoạch QC / Kiểm thử — Horus Eye.html` — dùng mục này để nắm nhanh cấu trúc và các rule quan trọng khi research/viết ticket liên quan QC.

### Các tầng kiểm thử

| Tầng | Công cụ | Phạm vi |
|---|---|---|
| Unit (Go) | `go test` | Handlers, billing/crypto, cap keys, invites, auth/MFA, retention, crypto cross-language (cần Node) |
| Unit/Component (FE) | Vitest + Testing Library | Lib, permissions, component UI; MSW mock |
| End-to-End | Playwright (Chromium) | Luồng UI + API; chạy được trực tiếp trên Go (`PLAYWRIGHT_BASE_URL=:3001`) |
| Parity | `test-e2e-parity` | So khớp hành vi TS ↔ Go |
| Load | k6 | `make loadtest-peak` vào `/api/send` |
| Thủ công | Demo tool + checklist | Trang demo QC (`qc/qc-tool.html`) mô phỏng website khách |

**Cổng chất lượng (Definition of Done tối thiểu)**: lint sạch + Vitest & Go test xanh + build thành công; các thay đổi đụng cả TS lẫn Go nên chạy thêm bộ parity.

### Ma trận 15 khu vực kiểm thử thủ công (103 case, P0 45 · P1 44 · P2 14)

| # | Khu vực | Trọng tâm | Case | P0/P1/P2 |
|---|---|---|---|---|
| 1 | Smoke & môi trường | Dịch vụ up, health probe, migrate, đăng nhập | 6 | 4/2/0 |
| 2 | Cấu hình tracker | Nhúng script, data-attributes, CORS | 6 | 2/3/1 |
| 3 | Pageview & điều hướng | Pageview, SPA, UTM, click-id | 6 | 3/3/0 |
| 4 | Custom event | Declarative, JS, đủ 5 kiểu dữ liệu | 7 | 3/3/1 |
| 5 | Identify & session data | identify, session_data, distinctId | 5 | 2/2/1 |
| 6 | Revenue | USD/EUR/JPY, multi-item, attribution | 5 | 2/3/0 |
| 7 | Batch & collect routes | `/api/batch`, `/p`, `/q`, xử lý lỗi từng item | 6 | 3/2/1 |
| 8 | Session replay | `recorder.js`, chunk, masking, sampleRate | 7 | 2/3/2 |
| 9 | Cap (CAPTCHA) | Widget/programmatic, verify, secret an toàn | 7 | 3/3/1 |
| 10 | Auth & MFA | Login/logout, register gate, rate-limit, WebAuthn | 9 | 5/3/1 |
| 11 | Workspace & RBAC | owner/viewer, alias team, last-owner, admin | 8 | 4/3/1 |
| 12 | Email invites | Tạo/gửi lại/thu hồi, accept, hết hạn | 6 | 2/3/1 |
| 13 | Billing & Crypto | Free/Pro limit, checkout, callback, prepaid | 8 | 4/3/1 |
| 14 | Reports/Realtime/Audience/Share | Funnel/retention…, realtime, Cohorts, Compare, Heatmaps, share/board | 10 | 2/6/2 |
| 15 | Privacy & Vận hành | No-cookie, DNT, IP, Plan A/B, fail-closed | 7 | 4/2/1 |

Thứ tự chạy đề xuất: chạy P0 của Khu vực 1 (Smoke) trước; nếu smoke fail thì dừng. Sau đó chạy 2→15, ưu tiên P0 rồi P1. Hồi quy: tối thiểu chạy lại toàn bộ P0 + P1 mỗi lần release; P2 chạy khi có thay đổi liên quan.

### Rule / gotcha quan trọng cho QC (đã biết, không phải bug)

- **Heatmap "disabled"**: `POST /api/record type=heatmap` trả `heatmap_disabled` — đường ghi heatmap ở Go đang hoãn (case 8.7, 14.10).
- **Replay trống**: cần bật `replayEnabled` + `sampleRate=1`; `sampleRate < 1` nghĩa là không phải phiên nào cũng ghi (case 8.6) — không phải bug.
- **Logout 503**: thiếu `REDIS_URL` (case 10.5) — cấu hình Redis chung với Cap, đây là fail-closed theo thiết kế.
- **Dữ liệu không hiện / CORS**: lỗi cấu hình phổ biến nhất là Host URL trỏ nhầm sang frontend (:3000) thay vì API Go (:3001).
- **"skipped" trên demo tool**: phải Apply script trước Pageview/Event/Identify/Revenue; điền slug trước batch; Apply CAP trước Solve/Verify.
- **Dummy billing chỉ chạy dev**: `BILLING_PROVIDER=dummy` khi `APP_ENV≠development` → boot fatal (case 13.4), đúng thiết kế fail-closed.

### Môi trường test

| Dịch vụ | Local (Docker) | Dev chung (a2zflow) | Vai trò |
|---|---|---|---|
| Frontend / Dashboard | `http://localhost:3000` | `https://livetracking-dashboard-dev.a2zflow.ai` | Phục vụ script.js, recorder.js & dashboard → Script URL |
| API thu thập (Go) | `http://localhost:3001` | `https://livetracking-api-dev.a2zflow.ai` | Nhận dữ liệu: `/api/send`, `/api/batch`, `/api/record`, `/p`, `/q` → Host URL |
| Cap (CAPTCHA) | `http://localhost:3002` | `https://livetracking-cap-dev.a2zflow.ai` | Cấp/xác minh CAPTCHA — chỉ cần khi test Khu vực 9 |
| Demo tool QC | `http://localhost:8080` | — | Trang tĩnh mô phỏng website khách + siteverify proxy (`node demo/server.mjs`) |

Công cụ QC: đường thu thập & Cap dùng demo tool (`qc/qc-tool.html`); auth/RBAC/billing/reports thao tác trên dashboard + kiểm DevTools Network/DB.

## 7. Đọc tiếp

- Bối cảnh nghiệp vụ, stakeholders, FR/NFR: `01-nghiep-vu-va-yeu-cau.md`
- Kiến trúc, dữ liệu, API, luồng nghiệp vụ: `02-kien-truc-va-du-lieu.md`
- Tính năng đặc thù Horus Eye (Workspace/RBAC, Billing, Crypto, Cap, WebAuthn, email invite): `03-tinh-nang-dac-thu-horus-eye.md`
