# CLAUDE.md — Live Tracking

## Tổng quan sản phẩm

Live Tracking (tên sản phẩm thật trong mã nguồn: **Horus Eye**) là sản phẩm phân tích web/app analytics, định vị tương tự Google Analytics và Umami (umami.is).

**Thay đổi lớn (27/07/2026): source code đã được thay thế gần như 100%.** Horus Eye là fork của Umami, viết lại toàn bộ backend bằng Go, giữ Next.js cho frontend/dashboard. Các mô tả kiến trúc/tính năng cũ (nếu còn thấy ở đâu đó trong lịch sử trao đổi) không còn đúng — luôn lấy `knowledge/01-04-*.md` làm nguồn tham chiếu chính, không dùng lại thông tin trước ngày 27/07/2026.

Điểm khác biệt cốt lõi so với Umami gốc/GA4 (theo mã nguồn thật, xem `knowledge/01`):

- **Self-hosted, không cookie**: toàn bộ dữ liệu tracking nằm trong hạ tầng khách hàng; nhận diện phiên bằng hash ẩn danh (salt xoay theo tháng), không cookie, không lưu IP thô.
- **Realtime bằng polling**: `GET /api/realtime/{websiteId}` — không WebSocket (đánh đổi độ trễ lấy sự đơn giản, chưa phải "thời gian thực" tức thời).
- **Lọc bot theo user-agent** (không phải behavioral/ML): ingest loại bot dựa trên phát hiện user-agent; việc chặn bot/lạm dụng nâng cao dựa vào Cap CAPTCHA ở các luồng nhạy cảm (đăng nhập, đăng ký), không phải chấm điểm hành vi.
- **Sẵn sàng vận hành doanh nghiệp**: Workspace multi-tenant + RBAC (owner/viewer), Billing Free/Pro/Enterprise (+ thanh toán crypto), WebAuthn MFA (passkey), mời thành viên qua email — đều là phần Horus Eye bổ sung so với Umami gốc.

Đây là dự án mới, không liên quan đến A2Z Product.

## Giai đoạn dự án

- Đang ở giai đoạn đầu, **chưa có end-user thật**.
- Đang phát triển ở môi trường **dev** và **staging** (deployment tham chiếu: `livetracking/a2zflow`), chưa lên production.
- Toàn bộ codebase được xây dựng theo hướng **vibe coding** — chưa có convention/coding rule chính thức được chốt ngoài các quy ước tối thiểu ghi ở `knowledge/04` (Biome cho FE/TS, package theo miền cho Go). Không giả định thêm coding convention hay implementation detail nếu chưa được xác nhận.
- Tech stack: xem mục "Ghi chú thêm" bên dưới và `knowledge/02-kien-truc-va-du-lieu.md`.

## Cấu trúc repo này

Repo này **không chứa mã nguồn** sản phẩm — đây là workspace tài liệu/nghiệp vụ cho PM/BA, không phải codebase để build/lint/test. Mã nguồn thật nằm ở repo riêng `live-tracking` (không nằm trong thư mục này).

- `knowledge/01-04-*.md` — 4 tài liệu tổng quan được sinh ra từ việc phân tích cấu trúc & mã nguồn thực tế của repo `live-tracking` (thư mục thật: `Athena/Projects/horus-eye`), không phải suy đoán. Đây là nguồn tham chiếu chính về nghiệp vụ, kiến trúc, dữ liệu, tính năng đặc thù, định vị cạnh tranh, roadmap và QC. Đọc các file này trước khi research sâu hoặc viết ticket để tránh suy diễn sai lệch so với mã nguồn thật.
  - `01-nghiep-vu-va-yeu-cau.md` — nguồn gốc (fork Umami, BE Go), bài toán, stakeholders, năng lực hiện có/nâng cao, FR/NFR.
  - `02-kien-truc-va-du-lieu.md` — kiến trúc FE (Next.js)/BE (Go), Plan A (Postgres)/Plan B (ClickHouse), mô hình dữ liệu, danh mục event, bản đồ API, luồng nghiệp vụ, bảo mật, hiệu năng, triển khai/env vars.
  - `03-tinh-nang-dac-thu-horus-eye.md` — 5 mảng khác biệt so với Umami gốc: Workspace & RBAC, Billing Free/Pro/Enterprise, Thanh toán Crypto, Cap (CAPTCHA), WebAuthn MFA & mời email.
  - `04-dinh-vi-ke-hoach-va-qc.md` — so sánh cạnh tranh (GA4/Plausible/Matomo), lộ trình, rủi ro, quy ước code/CI-CD, và tóm tắt kế hoạch QC (15 khu vực, 103 case).
  - Lưu ý: các tài liệu này là snapshot tại thời điểm viết (27/07/2026) — nếu cần thông tin "hiện tại" về mã nguồn thật, đối chiếu lại với repo `live-tracking` thay vì chỉ tin snapshot.
- `references/ticket-template-lean.md` — cấu trúc chuẩn (format) của một Lean Ticket: Epic Summary Block, User Stories Structure, Out of Scope Section.
- `skills/write-ticket/SKILL.md` — skill viết ticket cho Live Tracking, áp dụng template ở trên cộng các lưu ý nghiệp vụ đặc thù (role hợp lệ, data source Realtime/Historical, privacy, tính năng nhạy cảm). Dùng khi được yêu cầu viết ticket/spec/breakdown feature.

Không có lệnh build/lint/test cho repo này.

## Quy trình làm việc

- Dự án chạy theo **Kanban**, không phải Scrum. Không giả định có sprint, sprint planning, hay velocity khi hỗ trợ breakdown công việc.
- Người dùng chính tương tác với Claude trong dự án này là Product Manager / Business Analyst, tập trung vào research, discovery, viết spec, và breakdown ticket — không phải viết code.

## Tính năng nhạy cảm

- **Nạp tiền bằng crypto**: đây là tính năng nhạy cảm của sản phẩm (liên quan tiền thật, giao dịch on-chain). Khi research, viết spec, hoặc breakdown ticket cho tính năng này, cần đặc biệt cẩn trọng và làm rõ kỹ các business decision quan trọng (fraud/abuse, tỷ giá, xác nhận giao dịch, hoàn tiền, compliance) — không tự suy diễn thay quyết định business.
- Hiện chưa có yêu cầu bảo mật/compliance cụ thể nào được chốt cho tính năng này. Cần hỏi rõ khi bắt đầu đi sâu vào feature này.

## Ghi chú thêm

> Phần này bổ sung dần theo thời gian: tên chính thức của sản phẩm, tech stack, thuật ngữ nội bộ, và các quyết định đã chốt.

```
Tên sản phẩm: tên workspace là "Live Tracking"; tên sản phẩm thật trong mã nguồn là "Horus Eye"
  (fork của Umami, viết lại backend bằng Go). Xem knowledge/01 mục 1.

Tech stack (của sản phẩm/repo live-tracking = Athena/Projects/horus-eye, theo knowledge/02-kien-truc-va-du-lieu.md
  — không phải tech stack của workspace tài liệu này):
  - Backend: Go đơn khối (go/internal/handlers, không tách microservice ingest/event-processor/
    bot-scoring/realtime-gateway riêng như mô tả trước đây — đó là kiến trúc cũ, KHÔNG còn đúng).
  - Frontend: Next.js (dashboard + phục vụ script.js/recorder.js), chạy FE_NO_DB (không chạm DB).
  - Lưu trữ: PostgreSQL (control plane + Plan A analytics) + ClickHouse tùy chọn (Plan B analytics,
    bật bằng CLICKHOUSE_URL) + Redis (denylist/rate-limit, dùng chung với Cap).
  - Kafka: KHÔNG dùng (đã gỡ khỏi kiến trúc) — khác với thông tin từng ghi nhận trước đây.
  - Thành phần bổ sung: Cap (CAPTCHA proof-of-work, dịch vụ riêng cổng 3002), rrweb (session replay
    qua recorder.js), MaxMind GeoLite2 (geo tại chỗ), WebAuthn (passkey MFA), SMTP (email invite).
  - Migration: goose (không dùng Prisma migrate; Prisma chỉ sinh types cho FE).
  Chi tiết: knowledge/02.

Thuật ngữ nội bộ: xem Glossary cũ ở lịch sử hoặc tra trực tiếp trong knowledge/01-04 (Workspace,
  Plan A/Plan B, Cap, rrweb, prepaid, jti denylist, v.v. — mỗi file có định nghĩa tại chỗ khi dùng thuật ngữ).

Quyết định đã chốt / gap đã biết (theo mã nguồn 27/07/2026):
  - Bot detection: lọc theo user-agent ở bước ingest, KHÔNG phải chấm điểm hành vi (behavioral/ML).
    Thông tin "bot threshold hardcode 0.70/0.30" từng ghi nhận trước đây thuộc kiến trúc cũ, không
    còn áp dụng cho Horus Eye — xem knowledge/01 mục 7.
  - Heatmap: UI cấu hình & report đã có, nhưng đường ghi dữ liệu heatmap ở Go đang HOÃN
    (trả heatmap_disabled) — gap đã biết, chưa fix. Xem knowledge/04 mục 3 & 6.
  - Nạp tiền crypto: cơ chế prepaid (1 lần nạp = 1 chu kỳ Pro, không auto-renew) đã có trong code,
    nhưng CHƯA có quyết định business/compliance chính thức nào được chốt (fraud/abuse, tỷ giá,
    xác nhận giao dịch, hoàn tiền, compliance) — không tự suy diễn khi đào sâu spec. Xem knowledge/03
    mục 3 và mục "Tính năng nhạy cảm" bên dưới.
  - Realtime dùng polling (không WebSocket) — quyết định kỹ thuật hiện tại, chưa có kế hoạch đổi.
```
