## Epic Summary

**Epic Name:** Reports – Export report ra file CSV và PDF

**Goal:**
Workspace Owner và Workspace Viewer tải được report đang xem xuống máy dưới dạng CSV hoặc PDF, để chia sẻ số liệu với người không có tài khoản trong workspace và lưu trữ theo từng kỳ.

**Context:**
Live Tracking hiện chỉ cho phép xem report trên UI, chưa có cách nào mang số liệu ra ngoài hệ thống. Ticket này tiếp nối bộ template CSV đã chốt cho Events, Sessions và Realtime, tập trung vào việc tải report đang xem xuống máy.

**Template đính kèm:**
- Events – tab Chart (CSV):
- Sessions – tab Activity (CSV):
- Realtime (CSV):

---

## User Stories

### Story 1: Export the report currently displayed

**Story:** As a Workspace Viewer, I want tải report đang xem xuống máy bằng một nút Export ngay trên trang report đó, so that tôi lấy được số liệu mà không phải chụp màn hình hay nhập tay lại.

**Priority:** Must-have

**Acceptance Criteria:**

**AC.01. Export button is available on the five supported reports**
Given tôi đang đăng nhập với vai trò Workspace Owner hoặc Workspace Viewer,
And tôi đang ở một trang report có hỗ trợ export,
When trang hiển thị xong,
Then hệ thống hiển thị nút Export tại các vị trí sau:

| Report | Vị trí nút Export | Định dạng file |
|---|---|---|
| Events – tab Chart | Trên trang | CSV |
| Sessions – tab Activity | Trên trang | CSV |
| Realtime | Trên trang | CSV |
| Funnels | Trên từng funnel | PDF |
| Journeys | Trên trang | PDF |

**AC.02. Export button is hidden on tabs that do not support export yet**
Given tôi đang ở trang Events hoặc Sessions,
When tôi chuyển sang một tab chưa hỗ trợ export,
Then hệ thống không hiển thị nút Export trên tab đó

**AC.03. Export downloads the file in one action**
Given tôi đang xem một report có nút Export,
When tôi bấm Export,
Then hệ thống tạo file và trình duyệt tải file về máy ngay trong cùng thao tác,
And tôi không phải rời khỏi trang hay chờ một thông báo riêng

**AC.04. Repeat clicks while the file is being created are ignored**
Given tôi đã bấm Export và hệ thống đang tạo file,
When tôi bấm Export thêm lần nữa,
Then nút Export không nhận thêm thao tác cho tới khi file được tạo xong,
And chỉ một file duy nhất được tải về cho thao tác đó

**AC.05. Exported file covers the full data set matching the current conditions**
Given tôi đã chọn khoảng thời gian và/hoặc filter trên trang report,
When tôi bấm Export,
Then file chứa toàn bộ dữ liệu khớp với khoảng thời gian và filter đang chọn,
And phạm vi dữ liệu không bị giới hạn theo trang đang hiển thị trên màn hình

**AC.06. Exported file follows the naming convention**
Given hệ thống tạo file export,
When file được tải về máy,
Then tên file theo đúng mẫu `{website}_{report}_{YYYYMMDD-HHmmss}` cộng phần mở rộng tương ứng,
And phần thời gian là thời điểm bấm Export tính đến giây, nên hai lần export cùng một report không bao giờ trùng tên:

| Report | Mẫu tên file | Ví dụ |
|---|---|---|
| Events – tab Chart | `{website}_events_{YYYYMMDD-HHmmss}.csv` | `misty-automative_events_20260909-143210.csv` |
| Sessions – tab Activity | `{website}_sessions_{YYYYMMDD-HHmmss}.csv` | `misty-automative_sessions_20260909-143210.csv` |
| Realtime | `{website}_realtime_{YYYYMMDD-HHmmss}.csv` | `a2zflow_realtime_20260909-135004.csv` |
| Funnels | `{website}_funnel-{tên funnel}_{YYYYMMDD-HHmmss}.pdf` | `a2zflow_funnel-checkout_20260909-143210.pdf` |
| Journeys | `{website}_journeys_{YYYYMMDD-HHmmss}.pdf` | `a2zflow_journeys_20260909-143210.pdf` |

**AC.07. Website and funnel names are normalized in the file name**
Given tên website hoặc tên funnel có dấu tiếng Việt, khoảng trắng, hoặc ký tự đặc biệt,
When hệ thống đặt tên file,
Then các ký tự đó được chuyển thành chữ thường không dấu và nối bằng gạch ngang,
And mỗi phần tên website hoặc tên funnel được cắt tối đa 30 ký tự để tên file không quá dài

---

### Story 2: Export Events, Sessions and Realtime reports as CSV

**Story:** As a Workspace Owner, I want tải report Events, Sessions và Realtime dưới dạng file CSV theo đúng template đã chốt, so that tôi mở được số liệu bằng Excel và đưa thẳng vào báo cáo của mình.

**Priority:** Must-have

**Acceptance Criteria:**

**AC.01. CSV file follows the attached template**
Given tôi bấm Export trên một trong ba report Events – tab Chart, Sessions – tab Activity, hoặc Realtime,
When hệ thống tạo file,
Then file CSV có đúng cấu trúc của template tương ứng đính kèm ticket này,
And file gồm một khối thông tin chung ở đầu, một dòng trống, rồi đến khối dữ liệu

**AC.02. Header block carries the context of the export**
Given hệ thống tạo file CSV,
When tôi mở file,
Then khối thông tin chung ghi tên report, workspace, tên và domain website, biên thời gian của dữ liệu, người xuất và thời điểm xuất

**AC.03. Realtime export is a point-in-time snapshot**
Given tôi đang ở trang Realtime,
When tôi bấm Export,
Then file chứa dữ liệu của cửa sổ realtime tại đúng thời điểm bấm Export,
And biên của cửa sổ đó được ghi trong khối thông tin chung của file

**AC.04. Report with no data still produces a file**
Given report không có dữ liệu nào trong khoảng thời gian đang chọn,
When tôi bấm Export,
Then hệ thống vẫn tạo file CSV với khối thông tin chung đầy đủ,
And khối dữ liệu chỉ còn dòng tiêu đề cột hoặc các dòng có giá trị bằng 0 theo đúng template

---

### Story 3: Export a single funnel or the journey chart as PDF

**Story:** As a Workspace Owner, I want tải biểu đồ của một funnel hoặc của Journeys dưới dạng file PDF, so that tôi gửi được hình ảnh kết quả cho người không có quyền truy cập workspace.

**Priority:** Must-have

**Acceptance Criteria:**

**AC.01. Funnel is exported one at a time**
Given trang Funnels đang có nhiều funnel,
When tôi bấm Export ở một funnel,
Then file PDF chỉ chứa biểu đồ của đúng funnel đó,
And các funnel còn lại không xuất hiện trong file

**AC.02. PDF contains the chart image as displayed**
Given tôi bấm Export trên một funnel hoặc trên trang Journeys,
When hệ thống tạo file,
Then file PDF chứa ảnh của biểu đồ đúng như đang hiển thị trên màn hình

**AC.03. PDF carries a context line above the chart**
Given hệ thống tạo file PDF,
When tôi mở file,
Then phía trên ảnh biểu đồ có tên website, tên funnel hoặc tên report Journeys, khoảng thời gian của dữ liệu và thời điểm xuất file

**AC.04. Journeys exports the configuration currently displayed**
Given tôi đã thay đổi cấu hình đang xem trên trang Journeys,
When tôi bấm Export,
Then file PDF chứa biểu đồ ứng với cấu hình đang hiển thị tại thời điểm đó

**AC.05. Chart is fully rendered before the file is created**
Given biểu đồ chưa hiển thị xong,
When tôi bấm Export,
Then hệ thống chỉ tạo file sau khi biểu đồ đã hiển thị đầy đủ,
And ảnh trong file không bị thiếu phần nào của biểu đồ

---

### Story 4: Handle errors and edge cases

**Story:** As a Workspace Viewer, I want hệ thống xử lý rõ ràng các tình huống bất thường khi export, so that tôi biết file mình nhận được có dùng được hay không.

**Priority:** Must-have

**Acceptance Criteria:**

**AC.01. Funnel with no data still produces a PDF**
Given funnel không có dữ liệu trong khoảng thời gian đang chọn,
When tôi bấm Export,
Then hệ thống vẫn tạo file PDF với dòng thông tin đầy đủ,
And phần biểu đồ thể hiện trạng thái chưa có dữ liệu

**AC.02. Realtime export is not mixed with the next refresh**
Given trang Realtime đang tự cập nhật theo chu kỳ,
When tôi bấm Export,
Then file lấy đúng dữ liệu tại thời điểm bấm,
And dữ liệu của lần cập nhật kế tiếp không lọt vào file

**AC.03. Funnel deleted while being viewed cannot be exported**
Given funnel đã bị xóa bởi người khác trong lúc tôi đang xem,
When tôi bấm Export,
Then hệ thống không tạo file,
And hệ thống hiển thị trạng thái funnel không còn tồn tại

---

## Out of Scope

- **Events tab Activity, Events tab Properties, Sessions tab Properties** — chưa có template report, sẽ bổ sung ở ticket sau.
- **4 KPI card của Events và Realtime** — chưa có template, tách khỏi phần chart theo quy ước một file cho một section.
- **Các menu report còn lại** (Overview, Performance, Compare, Breakdown, Goals, Retention, Replays, Heatmaps, Segments, Cohorts, UTM, Revenue, Attribution) — làm sau khi 5 report đầu tiên chạy ổn định.
- **Xuất nhiều funnel trong một lần** — Funnels export theo từng funnel, không có thao tác xuất hàng loạt.
- **Định dạng file khác** (Excel, JSON, hoặc ảnh rời) — ticket này chỉ làm CSV và PDF.
- **Export bằng cơ chế xử lý nền cho tập dữ liệu lớn** — cần hàng đợi, trạng thái tiến độ và nơi lưu file tạm, tách thành ticket riêng.
- **Gửi report qua email, lên lịch export định kỳ, và lưu lịch sử các lần export** — thuộc phase sau.
- **Cho phép người dùng tùy biến cột hoặc bố cục của file xuất ra** — template do hệ thống quy định.
