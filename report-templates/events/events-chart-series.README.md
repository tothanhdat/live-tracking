# Report template — Events › Chart

File: `events-chart-series.csv`
Nguồn UI: menu **Events**, tab **Chart** (đường biểu đồ theo thời gian). Không bao gồm KPI cards và bảng Event/Count phía dưới — 2 phần đó sẽ có template riêng.

## Cấu trúc file

File gồm 2 block, ngăn cách bằng **1 dòng trống**:

1. **Metadata block** — 2 cột `Field,Value`, dạng key-value.
2. **Data block** — 1 header row + N data rows, long format.

Encoding: UTF-8 with BOM (để Excel trên Windows đọc đúng tiếng Việt). Delimiter `,`. Line ending `CRLF`. Giá trị chứa `,` `"` hoặc newline phải được bọc bằng `"` và escape `"` thành `""` theo RFC 4180.

## Metadata block

| Field | Ý nghĩa | Ghi chú |
|---|---|---|
| `report_name` | Tên report hiển thị | Cố định `Events - Chart` |
| `workspace` | Tên workspace đang chọn | |
| `website_name` | Tên website | |
| `website_domain` | Domain của website | |
| `date_range_start` / `date_range_end` | Biên thời gian thực tế của dữ liệu | ISO 8601 kèm offset. Offset cũng là timezone dùng để chia bucket. Đây là nguồn duy nhất cho biết bucket đầu/cuối có bị cắt hay không |
| `exported_by` | Email user thực hiện export | |
| `exported_at` | Thời điểm export | ISO 8601 kèm offset |

File **không** ghi lại filter đang áp dụng trên UI. Hệ quả: một file export khi đang lọc và một file export không lọc trông giống hệt nhau về cấu trúc, chỉ khác số liệu — người đọc không tự phân biệt được.

## Data block

| Cột | Kiểu | Mô tả |
|---|---|---|
| `period` | string | Mốc thời gian của bucket, format theo độ chi tiết của chart (xem bảng dưới) |
| `event_name` | string | Tên event; riêng `Other` là nhóm gộp, không phải event có thật trong hệ thống |
| `event_count` | integer | Số lượng event trong bucket |
| `pct_of_period_total` | decimal(5,2) | Tỷ trọng của event trong tổng của chính bucket đó |

Format của `period`:

| Độ chi tiết | Format | Ví dụ |
|---|---|---|
| Giờ | `YYYY-MM-DD HH:00` | `2026-08-15 14:00` |
| Ngày | `YYYY-MM-DD` | `2026-08-15` |
| Tuần | `YYYY-MM-DD` (ngày đầu tuần) | `2026-08-10` |
| Tháng | `YYYY-MM` | `2026-08` |

Format này được chọn để vừa đọc trực tiếp được, vừa sort đúng thứ tự thời gian khi user sort cột trong Excel — nhãn hiển thị trên trục X của chart (`Aug`, `Sep`…) không dùng làm giá trị cột vì sort theo alphabet sẽ sai. Độ chi tiết do backend suy ra từ date range, giống logic vẽ chart, và nhận biết được qua chính format của giá trị.

## Business rules

1. **Đủ bucket**: mọi bucket trong date range đều xuất hiện, kể cả khi `event_count = 0` — không bỏ dòng rỗng, để chart/BI vẽ đúng trục thời gian.
2. **Sắp xếp**: `period` tăng dần; trong mỗi bucket, các event giữ đúng thứ tự legend của chart (theo tổng toàn kỳ giảm dần), `Other` nằm cuối.
3. **Không có dòng tổng**: file chỉ chứa các series thành phần, không có dòng `Total`. Tổng của một bucket = `SUM(event_count)` của các dòng cùng `period`; tổng toàn kỳ = `SUM(event_count)` toàn file. Nhờ vậy pivot trong Excel không bị đếm trùng.
4. **Bucket bị cắt**: bucket đầu và bucket cuối có thể không trọn vẹn (ví dụ `2026-09` chỉ chứa dữ liệu tới `date_range_end`). File không đánh dấu riêng — người đọc đối chiếu với `date_range_start`/`date_range_end` ở metadata.
5. **Nhóm `Other`**: chart chỉ vẽ top N event theo tổng số lượng trong toàn date range; phần còn lại gộp thành `Other`. N phải khớp với số series UI đang vẽ. Nếu website có ít hơn hoặc bằng N event thì không sinh dòng `Other`.
6. **Tính %**: mẫu số là tổng của bucket, không phải tổng toàn kỳ. Bucket có tổng bằng 0 thì `pct_of_period_total = 0.00`.
7. **Filter**: dữ liệu export phải là dữ liệu *sau khi* áp filter đang chọn trên UI, đúng bằng những gì chart đang hiển thị.
8. **Tên file**: `{website}_events_{YYYYMMDD-HHmmss}.csv` — ví dụ `misty-automative_events_20260909-143210.csv`. Phần thời gian là thời điểm bấm Export tính đến giây nên không bao giờ trùng tên.

## Open questions

- Top N của nhóm `Other` là bao nhiêu và có cấu hình được không? (Screenshot đang là 7 event + `Other`.)
- Nếu một event có tên thật là `Other` thì phân biệt với nhóm gộp bằng cách nào?
