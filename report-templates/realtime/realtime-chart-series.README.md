# Report template — Realtime › Chart

File: `realtime-chart-series.csv`
Nguồn UI: menu **Realtime**, biểu đồ 2 đường Visitors/Views theo từng phút. 4 KPI card phía trên (Views, Visitors, Events, Countries) sẽ có template riêng.

## Cấu trúc file

File gồm 2 block, ngăn cách bằng **1 dòng trống**:

1. **Metadata block** — 2 cột `Field,Value`, dạng key-value.
2. **Data block** — 1 header row + N data rows, mỗi dòng là 1 phút.

Encoding: UTF-8 with BOM (để Excel trên Windows đọc đúng tiếng Việt). Delimiter `,`. Line ending `CRLF`. Giá trị chứa `,` `"` hoặc newline phải được bọc bằng `"` và escape `"` thành `""` theo RFC 4180.

## Metadata block

| Field | Ý nghĩa | Ghi chú |
|---|---|---|
| `report_name` | Tên report hiển thị | Cố định `Realtime - Chart` |
| `workspace` | Tên workspace đang chọn | |
| `website_name` | Tên website | |
| `website_domain` | Domain của website | |
| `date_range_start` / `date_range_end` | Biên của cửa sổ realtime tại thời điểm export | ISO 8601 kèm offset. Ở report này không phải khoảng thời gian user chọn mà là cửa sổ trượt do hệ thống quy định |
| `exported_by` | Email user thực hiện export | |
| `exported_at` | Thời điểm export | ISO 8601 kèm offset. Luôn sát ngay sau `date_range_end` |

## Data block

| Cột | Kiểu | Mô tả |
|---|---|---|
| `period` | string | Phút của bucket, format `YYYY-MM-DD HH:MM` (ví dụ `2026-09-09 13:42`) |
| `visitors` | integer | Số visitor duy nhất trong phút đó |
| `views` | integer | Số pageview trong phút đó |

Report này dùng dạng wide (mỗi series là một cột) thay vì long như `events-chart-series.csv`, vì số series ở đây cố định đúng 2 và không thay đổi theo dữ liệu.

## Business rules

1. **File là snapshot của cửa sổ trượt**: Realtime là view cập nhật liên tục, nên file chỉ phản ánh trạng thái tại `exported_at`. Export lại sau 1 phút sẽ ra tập dòng khác. Mốc cửa sổ nằm ở `date_range_start`/`date_range_end`.
2. **`visitors` không cộng dồn được**: đây là số **duy nhất** trong từng phút, một người truy cập 3 phút liên tiếp sẽ được đếm ở cả 3 dòng. Do đó `SUM(visitors)` **không** bằng số visitor của cả cửa sổ — con số đó lấy ở KPI card. `views` thì cộng dồn bình thường.
3. **Đủ bucket**: mọi phút trong cửa sổ đều xuất hiện, kể cả khi `visitors = 0` và `views = 0` — không bỏ dòng rỗng, để chart/BI vẽ đúng trục thời gian.
4. **Sắp xếp**: `period` tăng dần.
5. **Phút cuối chưa trọn**: bucket cuối cùng thường bị cắt giữa chừng nên số liệu thấp bất thường — đối chiếu `date_range_end` với `exported_at` để biết.
6. **Không có dòng tổng**: file chỉ chứa dữ liệu theo phút.
7. **Tên file**: `{website}_realtime_{YYYYMMDD-HHmmss}.csv` — ví dụ `a2zflow_realtime_20260909-135004.csv`. Phần thời gian là thời điểm bấm Export tính đến giây nên không bao giờ trùng tên.

## Sample data

Website `a2zflow` trên screenshot chưa có traffic nên toàn bộ giá trị đang là 0. **Số liệu trong file là dữ liệu minh hoạ**, dựng theo đúng cửa sổ thời gian nhìn thấy trên screenshot (13:19 → 13:49, bucket 1 phút, 31 dòng), có chèn vài phút bằng 0 để thể hiện rule số 3.

## Open questions

- Độ dài cửa sổ realtime là cố định 30 phút hay cấu hình được?
- Bucket có luôn là 1 phút không, hay co giãn theo độ dài cửa sổ?
- Realtime chạy bằng polling nên dữ liệu có độ trễ nhất định — có cần ghi mốc lần polling gần nhất vào metadata để người đọc biết số liệu cũ bao nhiêu giây không?
