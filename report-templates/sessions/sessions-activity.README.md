# Report template — Sessions › Activity

File: `sessions-activity.csv`
Nguồn UI: menu **Sessions**, tab **Activity** (bảng danh sách session). Tab **Properties** sẽ có template riêng.

## Cấu trúc file

File gồm 2 block, ngăn cách bằng **1 dòng trống**:

1. **Metadata block** — 2 cột `Field,Value`, dạng key-value.
2. **Data block** — 1 header row + N data rows, mỗi dòng là 1 session.

Encoding: UTF-8 with BOM (để Excel trên Windows đọc đúng tiếng Việt). Delimiter `,`. Line ending `CRLF`. Giá trị chứa `,` `"` hoặc newline phải được bọc bằng `"` và escape `"` thành `""` theo RFC 4180.

## Metadata block

| Field | Ý nghĩa | Ghi chú |
|---|---|---|
| `report_name` | Tên report hiển thị | Cố định `Sessions - Activity` |
| `workspace` | Tên workspace đang chọn | |
| `website_name` | Tên website | |
| `website_domain` | Domain của website | |
| `date_range_start` / `date_range_end` | Biên thời gian lọc session | ISO 8601 kèm offset. Offset cũng là timezone của cột `last_seen` |
| `exported_by` | Email user thực hiện export | |
| `exported_at` | Thời điểm export | ISO 8601 kèm offset |

File **không** ghi lại filter và từ khoá search đang áp dụng trên UI. Hệ quả: file export khi đang lọc và khi không lọc trông giống hệt nhau về cấu trúc, chỉ khác tập dòng.

## Data block

| Cột | Kiểu | Nguồn trên UI | Mô tả |
|---|---|---|---|
| `session_id` | string (UUID) | cột Session | Định danh ẩn danh của session. UI chỉ vẽ avatar sinh từ ID này, không hiện ID |
| `visits` | integer | Visits | Số lượt truy cập thuộc session |
| `views` | integer | Views | Số pageview |
| `events` | integer | Events | Số event (gồm cả pageview và custom event) |
| `country` | string | Location (phần sau) | Tên quốc gia |
| `city` | string | Location (phần trước) | Tên thành phố |
| `browser` | string | Browser | |
| `os` | string | OS | |
| `device` | string | Device | `Laptop` / `Desktop` / `Mobile` / `Tablet` |
| `last_seen` | ISO 8601 datetime | Last seen | Thời điểm hoạt động cuối cùng, dạng tuyệt đối |

## Business rules

1. **`last_seen` xuất dạng tuyệt đối**: UI hiển thị dạng tương đối (`about 3 hours ago`) nhưng file phải ghi mốc thời gian thật kèm offset, vì chuỗi tương đối sẽ sai ngay khi mở lại file vào hôm sau.
2. **Tách `Location` thành 2 cột**: UI ghép thành `City, Country`; file tách ra để lọc và pivot theo quốc gia mà không phải cắt chuỗi.
3. **Sắp xếp**: mặc định theo `last_seen` giảm dần, giống thứ tự đang hiển thị trên UI. Nếu user đã đổi cột sort trên UI thì file export theo đúng thứ tự đó.
4. **Giá trị không xác định**: khi không nhận diện được geo/browser/os/device, ghi `(unknown)` — không để ô trống, để phân biệt với lỗi mất dữ liệu khi import.
5. **Phạm vi export**: xuất toàn bộ session khớp date range và filter hiện tại, **không** giới hạn theo trang đang xem trên UI.
6. **Không có dòng tổng**: mỗi dòng là một session; tổng tính bằng `SUM` khi cần.
7. **Filter & search**: dữ liệu export phải là dữ liệu *sau khi* áp filter và từ khoá search đang có trên UI, đúng bằng những gì bảng đang hiển thị.
8. **Tên file**: `{website}_sessions_{YYYYMMDD-HHmmss}.csv` — ví dụ `misty-automative_sessions_20260909-143210.csv`. Phần thời gian là thời điểm bấm Export tính đến giây nên không bao giờ trùng tên.

## Sample data

18 dòng đầu lấy đúng theo các dòng nhìn thấy được trên screenshot (Singapore / Chrome / macOS / Laptop). **2 dòng cuối là dữ liệu minh hoạ thêm**, không có trên screenshot — thêm vào để thể hiện trường hợp khác quốc gia/trình duyệt/thiết bị và trường hợp `(unknown)`.

## Open questions

- Số lượng session tối đa cho một lần export là bao nhiêu, và khi vượt ngưỡng thì xử lý thế nào?
- Cột `events` có bao gồm pageview hay chỉ tính custom event? (Trên screenshot có dòng `views = 3, events = 1`, gợi ý 2 chỉ số đếm độc lập — cần xác nhận với dev.)
- Session ở trạng thái đang hoạt động (chưa kết thúc) có được export không?
