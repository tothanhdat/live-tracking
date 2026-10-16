## Epic Summary

**Epic Name:** Heatmaps – Cache ảnh nền trang và siết mặc định Mask level

**Goal:**
Workspace Owner và Workspace Viewer xem được vùng click và scroll của từng page trên một ảnh nền ổn định, không phụ thuộc trạng thái đăng nhập của website đang tích hợp, và website tạo mới được đặt mức che dữ liệu cá nhân cao ngay từ đầu.

**Context:**
Heatmaps hiện dựng ảnh nền bằng cách tải trực tiếp website đang tích hợp tại thời điểm xem, nên khi người xem report đã đăng xuất khỏi website đó thì các page sau đăng nhập hiển thị ra màn hình đăng nhập. Ticket này đổi cách lấy ảnh nền sang dùng ảnh đã lưu theo chu kỳ, và đổi giá trị mặc định của Mask level trong Cấu hình Website.

---

## User Stories

### Story 1: Render heatmaps on a stored page snapshot

**Story:** As a Workspace Viewer, I want xem heatmap Clicks và Scroll trên ảnh nền đã được hệ thống lưu lại của page đó, so that report vẫn đọc được đúng dù website đang tích hợp đã bị đăng xuất hoặc thay đổi trạng thái hiển thị.

**Priority:** Must-have

**Acceptance Criteria:**

**AC.01. Heatmap renders on the stored snapshot instead of loading the live site**
Given page tôi chọn đã có ảnh nền được lưu,
When tôi mở tab Clicks hoặc tab Scroll,
Then hệ thống dựng heatmap trên ảnh nền đã lưu của page đó,
And trong trường hợp này hệ thống không tải website đang tích hợp

**AC.02. Stored snapshot is independent of the viewer's session on the integrated site**
Given page tôi chọn đã có ảnh nền được lưu,
And tôi đã đăng xuất khỏi website đang tích hợp trên chính trình duyệt tôi đang xem report,
When tôi mở lại page đó trên Heatmaps,
Then hệ thống vẫn hiển thị ảnh nền đã lưu,
And nội dung màn hình đăng nhập của website đó không thay thế ảnh nền

**AC.03. Snapshot is stored and displayed per page and per screen width option**
Given một khách truy cập mở page với độ rộng màn hình bất kỳ,
When hệ thống ghi lại ảnh nền cho lượt truy cập đó,
Then ảnh nền được lưu vào mốc screen width phù hợp trong danh sách Screen width có sẵn,
And hệ thống không tạo thêm mốc screen width mới ngoài danh sách đó

---

### Story 2: Capture and refresh the snapshot on a fixed cycle

**Story:** As a Workspace Owner, I want ảnh nền của mỗi page được ghi lại từ lượt truy cập thật và tự làm mới sau mỗi 5 ngày, so that report bám theo layout hiện tại của website mà tôi không phải thao tác gì thêm.

**Priority:** Must-have

**Acceptance Criteria:**

**AC.01. Snapshot is captured only when it is missing or older than the cycle**
Given page chưa có ảnh nền hoặc ảnh nền hiện có đã quá 5 ngày kể từ lần ghi gần nhất,
When có khách truy cập thật mở page đó,
Then hệ thống ghi lại ảnh giao diện của page tại thời điểm đó làm ảnh nền,
And ảnh nền được che theo đúng Mask level đang cấu hình của website tại thời điểm ghi

**AC.02. No capture happens while a valid snapshot exists**
Given page đã có ảnh nền được ghi chưa quá 5 ngày,
When có thêm khách truy cập thật mở page đó,
Then hệ thống không ghi lại ảnh nền mới cho page đó

**AC.03. Previous snapshot stays visible until a new one is captured**
Given ảnh nền của một page đã quá 5 ngày,
And chưa có khách truy cập thật nào mở lại page đó,
When tôi mở page đó trên Heatmaps,
Then hệ thống vẫn hiển thị ảnh nền cũ

---

### Story 3: Default the website Mask level to strict

**Story:** As a Workspace Owner, I want website tôi tạo mới có Mask level mặc định là strict, so that dữ liệu cá nhân hiển thị trên giao diện được che ở mức cao ngay từ đầu mà tôi không phải nhớ đổi thủ công.

**Priority:** Must-have

**Acceptance Criteria:**

**AC.01. New website defaults to strict**
Given tôi tạo một website mới,
When tôi mở Cấu hình Website của website vừa tạo,
Then thuộc tính Mask level hiển thị giá trị strict

**AC.02. Existing websites keep their current value**
Given một website đã được tạo trước khi thay đổi này được phát hành,
When tôi mở Cấu hình Website của website đó,
Then Mask level giữ nguyên giá trị đang được cấu hình

**AC.03. Changing Mask level discards the cached snapshots of that website**
Given website đã có ảnh nền được lưu cho một hoặc nhiều page,
When tôi đổi Mask level của website đó và lưu lại,
Then toàn bộ ảnh nền đã lưu của website đó bị bỏ ngay,
And ảnh nền được ghi lại theo mức che mới ở lượt truy cập thật kế tiếp của từng page

---

### Story 4: Handle errors and edge cases

**Story:** As a Workspace Viewer, I want hệ thống vẫn hiển thị được heatmap khi page chưa có ảnh nền, so that tôi không gặp màn hình trống trong lúc chờ ảnh nền đầu tiên.

**Priority:** Must-have

**Acceptance Criteria:**

**AC.01. Page or screen width without a stored snapshot falls back to the live view**
Given page tôi chọn chưa có ảnh nền ở mốc screen width đang xem,
When tôi mở page đó trên Heatmaps,
Then hệ thống tạm dựng heatmap theo cách hiện tại là tải website đang tích hợp,
And ảnh nền được ghi lại ngay ở lượt truy cập thật kế tiếp, không chờ hết chu kỳ 5 ngày

**Edge Cases:**
- Website đổi layout giữa chu kỳ: report vẫn dùng ảnh nền của lần ghi gần nhất cho tới khi hết 5 ngày — đây là đánh đổi đã chốt, không làm mới sớm.

---

## Out of Scope

- **Đổi Mask level cho website đang tồn tại** — ticket này chỉ đổi giá trị mặc định khi tạo website mới, Owner tự đổi nếu muốn.
- **Xử lý lại dữ liệu click/scroll đã ghi nhận trước đây** — giữ nguyên theo mức che cũ, không backfill; các page đã có dữ liệu cũng phải chờ lượt truy cập thật kế tiếp mới có ảnh nền.
- **Cấu hình chu kỳ làm mới và nút chụp lại thủ công** — chu kỳ 5 ngày cố định, chưa mở ra UI.
- **Cách các tính năng khác dùng chung Mask level áp dụng mức che (VD: Session Replay)** — ticket này chỉ đổi giá trị mặc định và cách ảnh nền Heatmaps áp mức che.
