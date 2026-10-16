# Feature Budget Proposal — Horus Eye iOS SDK

> **Last updated:** 2026-09-30
> **Prepared by:** Ryan (BA) · **Reviewer:** Dev / Tech Lead (cần bổ sung tên)
> **Decision owner:** PO (cần bổ sung tên) · **Decision needed by:** Cần PO xác nhận (Phase 1 dự kiến bắt đầu 2026-10-01)

---

## 1. Executive Summary

| Item | Nội dung |
|---|---|
| **The Ask** | Duyệt triển khai iOS SDK theo 2 Phase: Phase 1 (Core SDK, 1 Dev, ~2 tuần) để đối tác tích hợp và sử dụng; Phase 2 (mở rộng, 2 Dev, ~4 tuần) chỉ duyệt sau khi Phase 1 hoàn thành |
| **Problem** | Horus Eye hiện chưa có SDK cho iOS, đối tác có app iOS chưa thể tracking hành vi người dùng trên mobile |
| **Expected Value** | Mở khả năng tracking cho app iOS của đối tác, đạt parity với web (event, screen view, session). Giá trị định tính, không quy ra số |
| **Cost** | Est man-day theo hạng mục và role: Dev cung cấp (Section 5). Thời lượng Dev ước lượng: Phase 1 ~2 tuần (1 Dev), Phase 2 ~4 tuần (2 Dev) |
| **Timeline** | Phase 1: bắt đầu 01/10/2026, dev dự kiến xong ~14/10/2026. Phase 2: bắt đầu sau khi PO duyệt gate Phase 1 |
| **Recommendation** | Duyệt Phase 1 trước; Phase 2 duyệt sau khi có kết quả Phase 1 và kết quả nghiên cứu Replay / Heatmap |

---

## 2. Problem / Opportunity

**Context:**
Horus Eye là sản phẩm web/app analytics định vị tương tự Google Analytics và Umami.
Hiện chưa có SDK cho iOS, đối tác chưa ghi nhận được event, màn hình và session từ app iOS.

**Evidence:**

| # | Evidence | Data source | Date |
|---|---|---|---|
| 1 | Horus Eye chưa có SDK iOS; tracking hiện chỉ áp dụng cho web | Meeting minutes | 2026-09 |

**Cost of doing nothing:**
Đối tác có app iOS không thể dùng Horus Eye để theo dõi hành vi người dùng trên mobile.

---

## 3. Proposed Solution & Scope

**Solution summary:**
Cung cấp iOS SDK để đối tác tích hợp vào app, ghi nhận event, screen view và session (ẩn danh / đã định danh), đưa dữ liệu về dashboard Horus Eye. Triển khai theo 2 Phase, Phase 2 phụ thuộc kết quả Phase 1.

### Phase 1 — Core SDK

| Hạng mục | Summary |
|---|---|
| Cấu hình SDK | Đối tác bật SDK một lần; SDK tự tạo định danh thiết bị cố định, không cần thao tác thêm ở từng màn hình hoặc sự kiện |
| Track custom event | Ghi nhận hành động của người dùng (ví dụ "thêm vào giỏ hàng"), có thể kèm thông tin bổ sung |
| Track screen views | Ghi nhận màn hình người dùng đã xem, tương đương pageview trên web |
| Detect Session (Anonymous vs. Identified visitors) | Chưa đăng nhập: theo dõi ẩn danh theo thiết bị, không gắn danh tính thật. Đã đăng nhập: dữ liệu từ thời điểm đó được gắn với người dùng đã xác định |

### Phase 2 — Mở rộng

| Hạng mục | Summary |
|---|---|
| Auto Interaction Event | Tự động ghi nhận thao tác chạm của người dùng, tương tự click tracking trên web |
| Replay | Xem lại thao tác của người dùng trên màn hình. Team nghiên cứu khả năng thực hiện. Risk: policy của iOS |
| Heatmap | Hiển thị vùng được chạm nhiều nhất trong app. Team nghiên cứu khả năng thực hiện. Risk: policy của iOS |
| Performance | Đo trải nghiệm thực tế của người dùng, hiện chỉ có "thời gian ở mỗi màn hình" |

**Out of Scope:**
- Bot Detect cho mobile (nhận diện và lọc dữ liệu rác, ví dụ từ bản debug/test của đội phát triển app). Đang chờ PO confirm nhu cầu, xem Section 8.
- Các nền tảng khác ngoài iOS.

**Target users:**
Đối tác có app iOS tích hợp Horus Eye.

---

## 4. Business Value & Success Metrics

| Metric | Baseline (hiện tại) | Target | Đo sau bao lâu | Data source |
|---|---|---|---|---|
| Delivery: SDK Phase 1 cho đối tác tích hợp được | Chưa có SDK | Đủ 4 hạng mục Phase 1 | Khi release Phase 1 | QC report |
| Usage: có event, screen view từ iOS lên dashboard | 0 | Dữ liệu iOS hiển thị trên dashboard | Khi release Phase 1 | Dashboard Horus Eye |

**Value estimate:**
Không quy ra tiền; giá trị đánh giá theo mức độ hoàn thành scope (định tính). Quyết định ngân sách dựa trên Cost & Resource ở Section 5.

---

## 5. Cost & Resource Estimate

> Effort do Dev / Tech Lead estimate. Thời lượng bên dưới là ước lượng từ cuộc họp, chưa phải est man-day chính thức.

**Tổng quan nguồn lực (theo meeting minutes):**

| | Phase 1 — Core SDK | Phase 2 — Mở rộng |
|---|---|---|
| Nhân sự Dev | 1 Dev | 2 Dev |
| Thời lượng Dev | ~2 tuần, bắt đầu 01/10/2026 | ~4 tuần |
| Est man-day chính thức | Dev cung cấp | Dev cung cấp |

> Cột "Mô tả" do Dev điền: liệt kê chi tiết các task con cần làm để PO hình dung được số lượng và độ phức tạp của từng hạng mục.

**Est man-day theo hạng mục — Phase 1:**

| Hạng mục | Effort (man-days) | Người estimate | Mô tả (task con) |
|---|---|---|---|
| Cấu hình SDK | Dev cung cấp | Dev | Dev cung cấp |
| Track custom event | Dev cung cấp | Dev | Dev cung cấp |
| Track screen views | Dev cung cấp | Dev | Dev cung cấp |
| Detect Session (Anonymous vs. Identified) | Dev cung cấp | Dev | Dev cung cấp |
| **Total Phase 1** | **Dev cung cấp** | | |

**Est man-day theo hạng mục — Phase 2:**

| Hạng mục | Effort (man-days) | Người estimate | Mô tả (task con) |
|---|---|---|---|
| Auto Interaction Event | Dev cung cấp | Dev | Dev cung cấp |
| Replay | Dev cung cấp | Dev | Dev cung cấp (gồm nghiên cứu khả năng thực hiện) |
| Heatmap | Dev cung cấp | Dev | Dev cung cấp (gồm nghiên cứu khả năng thực hiện) |
| Performance | Dev cung cấp | Dev | Dev cung cấp |
| **Total Phase 2** | **Dev cung cấp** | | Replay / Heatmap có thể thay đổi sau khi nghiên cứu xong |

**Est man-day theo role — Phase 1** (1 Dev, ~2 tuần, bắt đầu 01/10/2026)

| Role | Effort (man-days) | Người estimate | Ghi chú |
|---|---|---|---|
| BA (spec, ticket) | Cần bổ sung | Ryan | |
| Design (UI/UX) | Dev cung cấp | Dev | Xác nhận có cần hay không |
| BE | Dev cung cấp | Dev | Xác nhận có cần thay đổi phía server hay không |
| Dev iOS | Dev cung cấp | Dev | Ước lượng thời lượng: ~2 tuần, 1 Dev |
| QC | Dev cung cấp | Dev / QC | |
| **Total** | **Dev cung cấp** | | |

**Est man-day theo role — Phase 2** (2 Dev, ~4 tuần)

| Role | Effort (man-days) | Người estimate | Ghi chú |
|---|---|---|---|
| BA (spec, ticket) | Cần bổ sung | Ryan | |
| Design (UI/UX) | Dev cung cấp | Dev | Heatmap / Replay có thể cần màn hình report |
| BE | Dev cung cấp | Dev | |
| Dev iOS | Dev cung cấp | Dev | Ước lượng thời lượng: ~4 tuần, 2 Dev |
| QC | Dev cung cấp | Dev / QC | |
| **Total** | **Dev cung cấp** | | Replay / Heatmap: effort chỉ ước lượng được sau khi nghiên cứu xong |

**ROI / Payback:**
Không quy ra tiền được; đánh giá theo scope đạt được so với effort Dev cung cấp (Section 5).

---

## 6. Timeline & Funding Gates

| Milestone | Deliverable | Target date | Gate (điều kiện để chi tiếp) |
|---|---|---|---|
| M1 — Discovery & Spec | Ticket + AC Phase 1 được approve | Trước 01/10/2026 (cần PO xác nhận) | PO sign-off spec |
| M2 — Phase 1 Release | Core SDK cho đối tác tích hợp | Dev dự kiến xong ~14/10/2026, chưa gồm QC/release | Pass QC, không có Critical bug |
| M3 — Review Phase 1 | Report kết quả Phase 1 (đối tác tích hợp được, dữ liệu lên dashboard) và kết quả nghiên cứu Replay / Heatmap | Cần bổ sung | PO duyệt Phase 2 |
| M4 — Phase 2 | Auto Interaction Event, Performance; Replay / Heatmap nếu khả thi | ~4 tuần sau khi bắt đầu | Pass QC, không có Critical bug |

---

## 7. Risks & Dependencies

**Risks:**

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| 1 | Policy của iOS có thể hạn chế Replay / Heatmap | Cần Dev đánh giá | H | Nghiên cứu khả thi trước; nếu không khả thi thì loại khỏi Phase 2 |
| 2 | Effort chưa có est man-day chính thức | Cần Dev đánh giá | M | Dev cung cấp est trước khi PO duyệt |
| 3 | Không có Bot Detect thì dữ liệu từ bản debug/test của đội phát triển app có thể làm sai số liệu dashboard | Cần PO đánh giá | M | PO quyết định ở Section 8 |

**Dependencies:**
- Dev cung cấp est man-day và xác nhận có cần BE / Design / QC.
- Kết quả nghiên cứu Replay / Heatmap trước khi chốt scope Phase 2.

**Open Questions (cần confirm trước khi PO quyết):**

| # | Question | Owner | Due |
|---|---|---|---|
| 1 | Est man-day từng role cho Phase 1 và Phase 2 | Dev / Tech Lead | Cần bổ sung |
| 2 | Có cần thay đổi phía BE / Design cho iOS SDK không | Dev / Tech Lead | Cần bổ sung |
| 3 | Replay / Heatmap có khả thi với policy iOS không | Dev | Trước gate M3 |

---

## 8. Decision Required

PO vui lòng chọn:

**Quyết định 1 — Ngân sách:**

- [ ] **Approve** Phase 1 (Core SDK), Phase 2 duyệt sau gate M3
- [ ] **Approve có điều kiện:** [điều kiện]
- [ ] **Defer** đến [quý / tháng] — lý do: [..]
- [ ] **Reject** — lý do: [..]

**Quyết định 2 — Bot Detect cho mobile:**

Bot Detect (nhận diện và lọc dữ liệu rác, ví dụ từ bản debug/test của đội phát triển app, để không làm sai số liệu dashboard) hiện nằm ngoài plan. PO có ưu tiên (prioritize) Bot Detect cho mobile không?

- [ ] Không đưa vào plan
- [ ] Đưa vào Phase 1
- [ ] Đưa vào Phase 2
- [ ] Đưa vào Phase khác / sau Phase 2

**PO comment:** [..]
**Signed off:** [Tên] · [YYYY-MM-DD]
