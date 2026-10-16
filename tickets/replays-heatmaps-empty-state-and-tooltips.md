# Replays & Heatmaps – Empty-state guidance and Behavior capture tooltips

## Epic Summary

**Epic Name:** Replays & Heatmaps – Empty-state guidance and Behavior capture tooltips

**Goal:**
Giúp người dùng ở workspace hiểu ngay vì sao menu Replays hoặc Heatmaps chưa có dữ liệu, và giúp Global Admin hiểu đúng ý nghĩa từng tham số khi cấu hình.

**Context:**
Cấu hình Replays và Heatmaps không còn nằm trong trang cấu hình của từng website nữa, mà đã chuyển về một bộ thiết lập chung ở trang Admin, mục Behavior capture, và chỉ Global Admin chỉnh được. Hệ quả là khi tính năng chưa bật, người dùng ở workspace mở menu báo cáo chỉ thấy trạng thái không có dữ liệu và không biết nguyên nhân. Bên cạnh đó các tham số trong mục Behavior capture chỉ có label, chưa có phần giải thích ý nghĩa. Ticket này tập trung vào hai điều chỉnh giao diện: thông báo ở hai menu báo cáo và tooltip cho các tham số trong mục Behavior capture.

---

## User Stories

### Story 1: Explain the empty Replays and Heatmaps menus when the feature is off

**Story:** As a Workspace Owner, I want biết được menu Replays hoặc Heatmaps đang trống vì tính năng chưa được bật chứ không phải vì lỗi, so that tôi không mất thời gian tự tìm nguyên nhân.

**Priority:** Must-have

**Acceptance Criteria:**

**AC.01. Menu explains that its feature is off**
Given Behavior capture đang để Replays hoặc Heatmaps ở trạng thái tắt,
When tôi mở menu tương ứng của một website bất kỳ,
Then hệ thống hiển thị thêm thông báo ngay bên dưới trạng thái không có dữ liệu sẵn có,
And nội dung thông báo khớp với bảng dưới đây

| Menu | Message |
|---|---|
| Replays | Session replays aren't turned on yet. Only sessions recorded after they are turned on will appear here. |
| Heatmaps | Heatmaps aren't turned on yet. Only data collected after they are turned on will appear here. |

---

### Story 2: Explain each Behavior capture setting with an inline tooltip

**Story:** As a Global Admin, I want xem được giải thích ngắn cho từng tham số trong mục Behavior capture, so that tôi thiết lập đúng ngay từ đầu mà không phải đoán ý nghĩa của sample rate hay thời lượng tối đa.

**Priority:** Must-have

**Acceptance Criteria:**

**AC.01. Every Behavior capture setting explains itself with a tooltip**
Given tôi đang ở mục Behavior capture trong trang Admin,
When tôi tương tác với phần gợi ý giải thích đặt cạnh nhãn của một tham số,
Then hệ thống hiển thị nội dung giải thích của đúng tham số đó,
And nội dung khớp với bảng dưới đây

| Setting | Tooltip content (English) |
|---|---|
| Replays | Record visitor sessions so they can be played back later. Applies to every active website in this installation. |
| Heatmaps | Collect click and scroll data to visualise where visitors interact. Applies to every active website in this installation. |
| Replay sample rate | Share of eligible sessions recorded for replay, from 5% to 100%. Lower it to reduce the amount of data stored. |
| Heatmap sample rate | Share of eligible sessions included in heatmap collection, from 5% to 100%. Lower it to reduce the amount of data stored. |
| Max duration | Longest session length that will be recorded. Recording stops once a session reaches this limit. |

---

## Out of Scope

- **Nút hoặc liên kết điều hướng từ thông báo sang nơi cấu hình** — cấu hình áp dụng cho toàn bộ installation nên đặt lối đi tắt bên trong một website là không phù hợp; thông báo chỉ hiển thị nội dung text.
- **Thông báo khi tính năng đã bật nhưng chưa thu được dữ liệu** — giữ nguyên trạng thái hiển thị hiện có, không đổi nội dung.
- **Các thay đổi khác đi kèm việc chuyển cấu hình về trang Admin** — phân quyền, phạm vi áp dụng toàn installation, bước xác nhận khi lưu, các tham số đã bị gỡ khỏi giao diện: thuộc ticket khác.
- **Bản dịch đa ngôn ngữ cho nội dung thông báo và tooltip** — ticket này chỉ chốt bản tiếng Anh.
