# Ticket — Phase 2 Hạng mục 1: Document logic detect Bot

## Epic Summary

**Epic Name:** Bot Detection – Document logic & áp dụng ngưỡng theo site

**Goal:**
Platform Admin có một tài liệu chính thức mô tả toàn bộ logic detect bot hiện tại; đồng thời Site Owner / Site Member (admin) cấu hình ngưỡng bot cho site của mình và ngưỡng đó thực sự ảnh hưởng đến kết quả phân loại traffic.

**Context:**
Bot detection hiện dùng hai bộ chấm điểm — heuristic-v1 (tín hiệu môi trường) và rrweb-v1 (hành vi chuột/click/scroll) — phân loại human/suspicious/bot theo ngưỡng hardcode 0.70/0.30. Logic này đang chạy nhưng chưa có tài liệu chính thức mô tả lại. Site đã có thể lưu ngưỡng riêng (bot_threshold, suspicious_threshold, exclude_bots_default) qua UI/API cấu hình có sẵn, nhưng scorer chưa đọc giá trị này nên việc chỉnh ngưỡng hiện không có tác dụng thật. Ticket này viết tài liệu chính thức mô tả toàn bộ logic detect bot, đồng thời nối ngưỡng theo site vào scorer. Tài liệu này cũng là đầu vào cho SDK mobile (Hạng mục 3, 4) vì tín hiệu bot trên app khác web.

**Expected UI Fields:**
- Không có field UI mới — tái sử dụng UI cấu hình ngưỡng bot theo site đã có sẵn (bot_threshold, suspicious_threshold, exclude_bots_default). Tài liệu là nội bộ (Markdown/Confluence), không có màn hình riêng trong sản phẩm.

**Definition of Done:** Ticket được coi là hoàn thành khi cả 3 story hoàn tất — bao gồm cả tài liệu (Story 2) đã được kỹ sư phụ trách sign-off (AC.05), không chỉ riêng phần code.

---

## User Stories

### Story 1: Apply per-site bot threshold during scoring

**Story:** As a Site Owner / Site Member (admin), I want ngưỡng bot/suspicious tôi đã cấu hình cho site của mình thực sự được áp dụng khi chấm điểm traffic, so that việc đổi ngưỡng tạo ra thay đổi thật trong kết quả phân loại, thay vì bị bỏ qua.

**Priority:** Must-have

**Acceptance Criteria:**

**AC.01. Site-level threshold overrides default when scoring**
Given site đã lưu bot_threshold và suspicious_threshold riêng,
When một event của site đó được chấm điểm,
Then hệ thống dùng ngưỡng riêng của site (không dùng ngưỡng mặc định 0.70/0.30) để phân loại human/suspicious/bot.

**AC.02. Threshold change applies to new events**
Given bot_threshold hoặc suspicious_threshold của site vừa được cập nhật,
When có event mới phát sinh sau thời điểm cập nhật,
Then event mới được phân loại theo ngưỡng vừa cập nhật.

**Edge Cases:**
- Event đã được chấm điểm trước khi site đổi ngưỡng — không tự động chấm lại (xem Out of Scope).
- Site bật/tắt exclude_bots_default sau khi đã có dữ liệu — chỉ ảnh hưởng cách tính báo cáo tổng hợp từ thời điểm đổi, không hồi tố.

---

### Story 2: Publish bot detection logic document

**Story:** As a Platform Admin, I want có một tài liệu chính thức mô tả cách bot detection chấm điểm, so that tôi có thể giải thích kết quả phân loại cho Site Owner và có một nguồn tham chiếu duy nhất khi troubleshoot.

**Priority:** Must-have

**Dependency:** Bản tài liệu chính thức (final) chỉ publish sau khi Story 1 (áp dụng ngưỡng theo site) đã deploy xong, để nội dung mô tả đúng với hành vi thực tế đang chạy — publish sớm hơn sẽ mô tả sai trạng thái ngưỡng.

**Acceptance Criteria:**

**AC.01. Document covers signal sources**
Given tài liệu được publish,
When đọc phần "Nguồn tín hiệu chấm bot",
Then tài liệu có đủ 2 đề mục: tín hiệu heuristic-v1 (tín hiệu môi trường) và đặc trưng hành vi dùng bởi rrweb-v1 — danh sách field/tín hiệu chi tiết do dev bổ sung theo AC.05.

**AC.02. Document explains scoring formula, score_version, and threshold**
Given tài liệu được publish,
When đọc phần "Công thức tính điểm", "Ý nghĩa score_version" và "Ngưỡng & cách phân loại",
Then tài liệu có đủ 3 đề mục: cách tính của từng score_version (công thức chính xác do dev bổ sung theo AC.05), ý nghĩa score_version, và ngưỡng phân loại human/suspicious/bot đang áp dụng — gồm cả ngưỡng mặc định và ngưỡng theo site.

**AC.03. Document explains manual labeling and exclusion from reports**
Given tài liệu được publish,
When đọc phần "Cách gán nhãn thủ công" và "Cách loại bot khỏi báo cáo",
Then tài liệu có đủ 2 đề mục: cách gán nhãn thủ công một event (human/bot), và cách event bị loại khỏi số liệu báo cáo theo classification/cấu hình exclude_bots_default.

**AC.04. Document states difference vs GA4 approach**
Given tài liệu được publish,
When đọc phần "So sánh với GA4",
Then tài liệu nêu rõ điểm khác biệt giữa cách app phát hiện bot bằng hành vi so với cách GA4 lọc bot theo danh sách tĩnh (Google + IAB), và ghi nhận đề xuất bổ sung tầng "known bots list" như một hướng dùng kèm (không thay thế chấm điểm hành vi).

**AC.05. Document reviewed by engineering owner before publish**
Given tài liệu đã soạn xong bản nháp (bao gồm các phần đánh dấu "Cần dev cung cấp"),
When chuẩn bị publish chính thức,
Then tài liệu phải được ít nhất một kỹ sư phụ trách bot-scoring service review, bổ sung các phần còn thiếu, và xác nhận khớp với code hiện tại trước khi công bố.

**Edge Cases:**
- Thuật toán scoring hoặc ngưỡng thay đổi sau khi publish (score_version mới) — tài liệu cần được cập nhật thủ công, không tự động đồng bộ.
- Phát hiện khác biệt giữa nội dung tài liệu và code thực tế trong lúc review — sửa lại tài liệu theo code (code là nguồn sự thật).

---

### Story 3: Handle invalid or missing threshold configuration

**Story:** As a Site Owner / Site Member (admin), I want hệ thống hoạt động đúng như kỳ vọng khi site của tôi chưa có threshold tùy chỉnh hoặc cấu hình không hợp lệ, so that việc chấm điểm không bao giờ bị lỗi hoặc cho ra kết quả khó hiểu.

**Priority:** Must-have

**Acceptance Criteria:**

**AC.01. Fallback to default when site has no custom config**
Given site chưa từng lưu bot_threshold/suspicious_threshold riêng,
When event của site đó được chấm điểm,
Then hệ thống dùng ngưỡng mặc định 0.70/0.30.

**AC.02. Reject invalid threshold combination**
Given giá trị suspicious_threshold nhập vào lớn hơn hoặc bằng bot_threshold,
When lưu cấu hình,
Then hệ thống báo lỗi và không lưu giá trị không hợp lệ, giữ nguyên ngưỡng đang áp dụng trước đó.
**[Cần dev xác nhận]** UI/API cấu hình bot threshold hiện tại đã validate trường hợp này chưa. Nếu chưa, cần quyết định: bổ sung validate trong ticket này (mở rộng scope) hay tách sang ticket riêng cho bot config API.

**Edge Cases:**
- Site vừa được tạo, chưa phát sinh event nào — chưa có gì để chấm điểm, không áp dụng.
- Cấu hình threshold của site bị xoá sau khi đã set — quay về ngưỡng mặc định.

---

## Nội dung tài liệu (Draft)

> Phần dưới đây là bản nháp nội dung tài liệu bot detection do BA tổng hợp từ knowledge base hiện có. Các mục đánh dấu **[Cần dev cung cấp]** là chi tiết implementation mà BA không có đủ dữ kiện để mô tả chính xác — dev bổ sung trước khi publish (xem Story 2 – AC.05).

### 1. Nguồn tín hiệu chấm bot

- **heuristic-v1** (`WeightedHeuristicScorer`): áp dụng cho mọi event chuẩn (không riêng interaction event). Đánh giá tín hiệu môi trường/hành vi như: trình duyệt chạy ở chế độ webdriver, headless browser, timing bất thường.
  **[Cần dev cung cấp]** Danh sách đầy đủ và ý nghĩa từng field trong `bot_signals{}` mà snippet gửi kèm mỗi event.
- **rrweb-v1** (`RRWebV1Scorer`): chỉ áp dụng cho `interaction_event`. Tổng hợp theo cửa sổ phiên các đặc trưng: mousemove, click, scroll, touchmove, số nguồn sự kiện duy nhất, trung vị/phương sai khoảng cách thời gian giữa các event, tổng quãng đường di chuyển chuột.

### 2. Công thức tính điểm của từng version

- heuristic-v1: tính theo cơ chế "weighted" trên các tín hiệu trong `bot_signals`.
  **[Cần dev cung cấp]** Trọng số cụ thể của từng tín hiệu và công thức tổng hợp ra điểm 0–1.
- rrweb-v1: tổng hợp các đặc trưng hành vi theo cửa sổ phiên thành một điểm số.
  **[Cần dev cung cấp]** Công thức/trọng số kết hợp các đặc trưng thành điểm số cuối.
- Khi một event vừa có tín hiệu heuristic vừa có tín hiệu rrweb (trường hợp `interaction_event`), điểm `bot_score` cuối cùng được tính/chọn như thế nào giữa hai model.
  **[Cần dev cung cấp]**

### 3. Ngưỡng & cách phân loại

- score ≥ 0.70 → `bot` · score ≥ 0.30 → `suspicious` · còn lại → `human`.
- Hiện ngưỡng áp dụng chung cho toàn hệ thống (hardcode), **chưa** theo cấu hình riêng từng site — dù bảng `bot_configs` đã lưu `bot_threshold`/`suspicious_threshold` per site và có API đọc/ghi. Đây chính là gap được Story 1 của ticket này fix.
- Sau khi Story 1 hoàn thành: mỗi site có thể có ngưỡng riêng; site chưa cấu hình dùng mặc định 0.70/0.30.

### 4. Ý nghĩa `score_version`

- Field lưu trong ClickHouse (`analytics.events.score_version`), xác định event được chấm điểm bởi model/phiên bản nào (VD `heuristic-v1`, `rrweb-v1`).
- Dùng để lọc trong Bot Dashboard theo phiên bản chấm điểm, và cho phép thay đổi thuật toán trong tương lai mà không làm mất khả năng phân biệt dữ liệu cũ/mới.
  **[Cần dev cung cấp]** Quy tắc chính xác khi nào score_version thay đổi/tăng, và cách versioning được set trong code.

### 5. Cách gán nhãn thủ công

- Trong Bot Dashboard, chọn một event nghi ngờ → gọi `POST .../bot/events/{event_id}/label` để gán nhãn `human` hoặc `bot`.
- Nhãn lưu vào cột `bot_label` (mặc định `unlabelled`) — dùng khi cần điều chỉnh một trường hợp bị chấm điểm sai.

### 6. Cách loại bot khỏi báo cáo

- Site có cấu hình `exclude_bots_default` (bảng `bot_configs`, mặc định bật).
- Khi bật, các event bị phân loại `bot` được loại khỏi số liệu tổng hợp hiển thị trong báo cáo (event gốc vẫn lưu đầy đủ trong ClickHouse).
  **[Cần dev cung cấp]** Cơ chế loại trừ áp dụng ở bước nào — lúc query báo cáo hay lúc ghi materialized view — vì ảnh hưởng đến việc bật/tắt config có hồi tố dữ liệu cũ hay không.

### 7. So sánh với GA4 (hành vi vs danh sách)

- GA4 loại bot theo danh sách tĩnh "known bots & spiders" (nghiên cứu của Google + IAB International Spiders and Bots List), bật mặc định.
- Điểm yếu của GA4: phản ứng chậm với bot mới (phải chờ cập nhật danh sách), bỏ lọt bot tinh vi giả người, xoay IP, hoặc đi qua Measurement Protocol.
- App phát hiện bot bằng hành vi thực tế (heuristic-v1 + rrweb-v1) thay vì dựa vào danh sách cố định — bắt được bot mới/tinh vi hơn. Đây là điểm khác biệt cạnh tranh cốt lõi so với GA4.

### 8. Đề xuất bổ sung "known bots list" (kiểu IAB)

- Thêm một tầng danh sách bot phổ biến đã biết (kiểu IAB) để bắt nhanh các bot thông dụng (VD Googlebot, Bingbot...) mà không cần chờ chấm điểm hành vi đầy đủ.
- Tầng này **dùng kèm**, không thay thế chấm điểm hành vi hiện có — heuristic-v1/rrweb-v1 vẫn là lớp chính để bắt bot mới/tinh vi.
  **[Cần dev cung cấp/quyết định]** Nguồn danh sách sử dụng (tự xây hay dùng danh sách mở có sẵn) và tích hợp ở bước nào trong pipeline (ingest, event-processor, hay bot-scoring). Đây là đề xuất để cân nhắc, không bắt buộc implement trong ticket này (xem Out of Scope).

### 9. Đầu vào cho SDK mobile (Hạng mục 3, 4)

- Tài liệu này là input cho SDK Android (Hạng mục 3) và iOS (Hạng mục 4), vì tín hiệu bot trên mobile app khác web (không có DOM/rrweb, không có timing trình duyệt tương tự).
  **[Cần dev cung cấp khi triển khai Hạng mục 3/4]** Bộ tín hiệu bot phù hợp cho môi trường mobile, dựa trên nguyên lý chấm điểm hành vi mô tả ở tài liệu này.

---

## Out of Scope

- **Implement tầng "known bots list"** — chỉ ghi nhận như đề xuất trong tài liệu (mục 8), không code trong ticket này.
- **Chấm lại (reclassify) các event lịch sử sau khi đổi ngưỡng** — event đã chấm điểm giữ nguyên classification cũ; batch reclassification để phase sau nếu cần.
- **Bot detection cho SDK mobile (Android/iOS)** — thuộc Hạng mục 3, 4 riêng; tài liệu ở ticket này chỉ làm đầu vào tham chiếu.
- **Thay đổi công thức/thuật toán chấm điểm hiện có (heuristic-v1, rrweb-v1)** — ticket này chỉ document logic hiện tại và fix việc áp dụng ngưỡng theo site, không đổi cách tính điểm.
- **UI mới để xem tài liệu trong app** — tài liệu là nội bộ (Markdown/Confluence), không có màn hình riêng trong sản phẩm.
