---
name: write-ticket
description: Viết ticket (spec) cho sản phẩm Live Tracking theo Lean Ticket template. Dùng khi user yêu cầu viết ticket, viết spec, breakdown feature thành ticket, hoặc soạn Epic Summary/User Stories cho một feature/bugfix của Live Tracking.
---

# Write Ticket — Live Tracking

## Khi nào dùng

Khi user yêu cầu viết ticket, viết spec, hoặc breakdown một feature/bugfix của Live Tracking thành ticket.

## Cách dùng

1. Đọc cấu trúc chuẩn tại `references/ticket-template-lean.md` (Epic Summary Block, User Stories Structure, Out of Scope Section) — đây là structure bắt buộc phải theo, chỉ đúng 3 section, không thêm section khác (kể cả "Open Question").
2. Trước khi viết, hỏi user tối đa 5 câu để làm rõ context còn thiếu hoặc business decision chưa chốt (VD: behavior khi xóa/lỗi, giới hạn dữ liệu, quyền truy cập, cách xử lý edge case...) — mỗi câu kèm 1 đề xuất/khuyến nghị cụ thể để user dễ chọn nhanh. Không tự suy diễn thay quyết định business.
3. Áp dụng cấu trúc đó cùng với các lưu ý nghiệp vụ đặc thù Live Tracking bên dưới.
4. Sau khi viết xong, tự review lại ticket trước khi gửi cho user — xem checklist ở mục "Tự review trước khi gửi".

## Lưu ý đặc thù Live Tracking

Đây là điểm cần nhớ khi điền vào structure của template, không phải section riêng trong ticket.

- **Role:** Story "As a..." dùng đúng 1 trong 5 role đã có: Global Admin (`admin`, toàn hệ thống), User (chưa thuộc workspace nào hoặc đang thao tác ở tầng user), Workspace Owner (`workspace-owner`, toàn quyền trong workspace), Workspace Viewer (`workspace-viewer`, chỉ đọc trong workspace), Khách truy cập (visitor cuối của website được tracking, không có tài khoản). Chi tiết quyền từng role: `knowledge/01-nghiep-vu-va-yeu-cau.md` mục 5 và `knowledge/03-tinh-nang-dac-thu-horus-eye.md` mục 1.
- **Ngôn ngữ Story:** Tiêu đề Story giữ tiếng Anh. Dòng "Story:" viết bằng tiếng Việt nhưng giữ nguyên cụm nối `As a`/`I want`/`so that`, tên role, và thuật ngữ kỹ thuật không có bản dịch tự nhiên (xem chi tiết + ví dụ tại `references/ticket-template-lean.md` mục "Rule ngôn ngữ cho Story").
- **Không ghi data source và privacy note vào ticket:** không mô tả feature lấy dữ liệu Realtime hay Historical, không mô tả cơ chế ẩn danh/IP/DNT — đây là technical detail, dev tự nắm. Kiến thức nền vẫn đọc ở `knowledge/02-kien-truc-va-du-lieu.md` để viết AC cho đúng behavior, nhưng không đưa vào ticket.
- **Tính năng nhạy cảm (VD: nạp tiền bằng crypto):** không tự suy diễn business decision (fraud/abuse, tỷ giá, xác nhận giao dịch, hoàn tiền, compliance) — hỏi rõ trước khi viết. Xem `knowledge/03-tinh-nang-dac-thu-horus-eye.md` mục 3.

## Mức độ chi tiết

- Giai đoạn hiện tại ưu tiên chạy được (MVP), chưa cần đào sâu edge case hiếm gặp hay chi tiết security/compliance — phần đó sẽ rà lại sau khi tính năng ổn định.
- Không mô tả implementation/technical detail trong ticket (API endpoint, query, retry mechanism...) — Business Analyst viết bằng business language, dev tự quyết kỹ thuật.
- Ưu tiên ít story hơn, mỗi story có AC rõ và testable.
- **Không có mục Expected UI Fields:** Epic Summary chỉ gồm Epic Name, Goal, Context — dev tham khảo design để biết field và bố cục màn hình.
- **UI field phải khớp thực tế:** nếu ticket dựa trên tài liệu tham khảo bên ngoài (VD: doc của sản phẩm tương tự), coi field/config trong đó là giả thuyết, không phải sự thật — chỉ nhắc tên field trong AC khi có xác nhận từ screenshot thật hoặc `knowledge/*.md`. Nếu sau đó user gửi screenshot mâu thuẫn với field đã giả định, rà lại toàn bộ ticket (AC, edge case, story khác) để đồng bộ — không chỉ sửa đúng chỗ user chỉ ra.
- **Không viết AC cho error message chung:** không viết AC mô tả error/status message của hệ thống (VD: "lưu thất bại", "mất kết nối", "có lỗi xảy ra"...) — hệ thống đã có cơ chế xử lý lỗi chung sẵn có, không cần lặp lại cho từng feature/ticket, trừ khi user yêu cầu rõ. Chỉ viết AC cho state/behavior đặc thù của feature đó (VD: "chưa có dữ liệu để hiển thị report", "xóa cascade theo entity cha").
- **Không viết nguyên văn error/status message trong AC** (không đặt message cụ thể trong ngoặc kép) — toàn bộ message/copy hệ thống kế thừa từ nền tảng gốc, dev/design tự quyết nội dung khi build. AC chỉ mô tả trạng thái/hành vi bằng business language, không quote literal text.
  - Bad: `Then hệ thống hiển thị trạng thái "chưa có dữ liệu" thay vì biểu đồ trống hoặc lỗi.`
  - Good: `Then hệ thống hiển thị trạng thái chưa có dữ liệu thay vì biểu đồ trống hoặc lỗi.`
- **Edge Cases phải được resolve, không để mơ hồ:** mỗi Edge Case candidate xử lý theo 1 trong 3 hướng trước khi đưa vào ticket:
  1. Là rule rõ ràng, test được → viết thành AC đầy đủ (Given/When/Then) trong story liên quan, không để dạng bullet mô tả suông.
  2. Đã được 1 AC khác trong ticket cover rồi → bỏ, không lặp lại.
  3. Là business decision chưa chốt → hỏi user trước khi viết (theo bước 2 ở "Cách dùng"), không tự suy diễn. Không thêm section "Open Question" riêng trong ticket trừ khi user yêu cầu — ticket giữ đúng 3 section chuẩn.

## Tự review trước khi gửi

Sau khi viết xong draft, tự kiểm tra trước khi gửi cho user:

- Epic Summary chỉ có Epic Name, Goal, Context — không có mục Expected UI Fields.
- Context không chứa mô tả nguồn dữ liệu, cơ chế kỹ thuật, hay ghi chú privacy.
- Không còn Edge Case bullet mơ hồ (xem mục "Edge Cases phải được resolve" ở trên) — mỗi edge case đã thành AC, đã bị bỏ vì trùng lặp, hoặc đã hỏi user.
- Không có AC nào quote nguyên văn error/status message, và không có AC mô tả error message chung không đặc thù feature.
- Role dùng đúng 1 trong 5 role hợp lệ; ngôn ngữ Story đúng rule (tiêu đề tiếng Anh, dòng Story tiếng Việt giữ cụm nối/role/thuật ngữ).
- Ticket chỉ có đúng 3 section: Epic Summary, User Stories, Out of Scope — không thêm section khác.
- Story cuối cùng vẫn là "Handle errors and edge cases", nhưng chỉ chứa AC đặc thù feature (không phải lỗi form chung).
