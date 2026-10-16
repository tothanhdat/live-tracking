# Ticket — Session: Replace session avatar with library-based avatar

## Epic Summary

**Epic Name:** Session – Replace session avatar with library-based avatar

**Goal:**
Workspace Owner / Workspace Viewer xem thông tin session sẽ thấy avatar chuyên nghiệp hơn, thay cho avatar minh họa hiện tại đang bị đánh giá là xấu và thiếu chuyên nghiệp.

**Context:**
Avatar hiện tại đại diện cho một session (VD: trong Session Detail) đang dùng hình minh họa nhân vật, không phù hợp với giao diện sản phẩm. Ticket này thay bằng avatar generate từ thư viện avatar có sẵn trong source code — dev tự research và chọn thư viện theo đề xuất của sếp Bill — áp dụng đồng bộ ở mọi vị trí đang hiển thị avatar đại diện cho session.

**Expected UI Fields:**
- Không có field UI mới — chỉ thay hình ảnh avatar tại các vị trí đã hiển thị avatar đại diện cho session.

---

## User Stories

### Story 1: Replace session avatar with library-generated avatar

**Story:** As a Workspace Owner / Workspace Viewer, I want avatar đại diện cho mỗi session được tạo từ thư viện avatar có sẵn trong source thay vì hình minh họa hiện tại, so that giao diện xem session trông chuyên nghiệp và nhất quán hơn.

**Priority:** Should-have

**Acceptance Criteria:**

**AC.01. Avatar generated deterministically and applied consistently**
Given một session có ID xác định,
When avatar của session đó được hiển thị ở bất kỳ vị trí nào trong sản phẩm,
Then avatar được generate dựa trên Session ID theo cùng một cách ở mọi vị trí,
And cùng một session luôn hiển thị đúng một avatar giống nhau ở mọi lần xem.

**AC.02. Fallback avatar when session identifier unavailable**
Given một session không có ID hợp lệ để generate avatar,
When avatar của session đó cần hiển thị,
Then hệ thống hiển thị một avatar mặc định thay vì để trống hoặc lỗi hiển thị.
