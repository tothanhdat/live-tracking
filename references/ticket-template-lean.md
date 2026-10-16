# Ticket Template — Lean Style (Live Tracking)

## Mục đích

File này mô tả **cấu trúc chuẩn** của một Lean Ticket cho sản phẩm Live Tracking — chỉ tập trung vào format/structure. Lưu ý nghiệp vụ đặc thù (role, data source, privacy...) khi áp dụng structure này được mô tả trong skill `write-ticket` (`skills/write-ticket/SKILL.md`).

---

## Epic Summary Block

```
## Epic Summary

**Epic Name:** [Module] – [Feature Name]

**Goal:**
[1–2 câu. WHO can do WHAT and WHY. Business language, không technical detail.]

**Context:**
[Business context — tiếp nối ticket nào, thuộc feature/module nào, ticket này tập trung giải quyết vấn đề gì. Max 3 câu. Business language, không technical detail.]
```

### Rules cho Epic Summary
- Epic Summary chỉ gồm **Epic Name, Goal, Context**. Không có mục "Expected UI Fields" — dev tham khảo design để biết field và bố cục màn hình.
- Goal = business language; Context = business language.
- Context mô tả: tiếp nối ticket nào, nền tảng đã có gì, ticket này tập trung vào điểm gì mới. Không mô tả API endpoint, query, hoặc implementation detail.
- Ví dụ Context tốt: "Tiếp nối ticket LT-42. Dashboard Realtime đã có nền tảng hiển thị active users. Ticket này tập trung vào việc thêm filter theo traffic source."
- Nếu feature tương tự feature khác, explicitly state difference ở góc độ business (behavior/scope), không technical.
- Goal max 2 câu; Context max 5 câu.
- Context **không** mô tả nguồn dữ liệu, mô hình dữ liệu, cơ chế kỹ thuật, hay ghi chú privacy — những nội dung này là technical detail, không thuộc ticket.
- Nếu ticket liên quan tính năng nhạy cảm (VD: nạp tiền bằng crypto), không tự suy diễn business decision (fraud/abuse, tỷ giá, xác nhận giao dịch, hoàn tiền, compliance) — hỏi rõ trước khi viết.

---

## User Stories Structure

### Story Order chuẩn

| # | Story Name | Required | Notes |
|---|---|---|---|
| 1 | [Core action story] | ✓ | Main happy path — primary thing user làm được. |
| 2 | [Configure / filter / input] | Nếu có | Cách user cấu hình, filter, hoặc nhập dữ liệu cho feature. |
| 3 | [View / verify result] | Nếu có | Cách user xem kết quả (dashboard, report, real-time update). |
| N | Handle errors and edge cases | ✓ | Luôn ở cuối. Viết theo error thực tế của feature. |

> Ưu tiên ít story hơn, mỗi story chứa AC rõ và testable. Số lượng và nội dung story phụ thuộc vào user flow thực tế của feature.

### Rule ngôn ngữ cho Story

- **Tiêu đề Story** ("Story N: ...") giữ nguyên tiếng Anh (verb phrase), VD: `Story 1: Rotate Cap secret key without breaking active verifications`.
- **Dòng "Story:"** (As a.../I want.../so that...) viết nội dung bằng tiếng Việt, nhưng giữ nguyên các cụm nối `As a`, `I want`, `so that` bằng tiếng Anh, cùng tên role và các thuật ngữ/khái niệm kỹ thuật chung không có bản dịch tự nhiên (VD: workspace, sampleRate, rrweb, API, cap key, replay...). Không dịch ép các thuật ngữ này.
  - VD: `As a Workspace Owner, I want xoay secret key của Cap cho website của mình mà không làm gián đoạn các lượt verify CAPTCHA đang diễn ra, so that tôi bảo vệ được form đăng nhập/đăng ký mà không gây lỗi cho khách truy cập thật.`

### Per-Story Structure

```
### Story N: [Title — English verb phrase]

**Story:** As a [role], I want [hành động — tiếng Việt], so that [kết quả — tiếng Việt].

**Priority:** Must-have | Should-have | Nice-to-have

**Acceptance Criteria:**

**AC.01. [One-line summary in English]**
Given [pre-condition],
And [additional pre-condition if needed],
When [trigger / action],
Then [expected outcome],
And [additional outcome]

**AC.02. [Title]**
Given ...,
When ...,
Then ...

**Edge Cases:**
- [Tình huống bất thường — mô tả từ góc độ user/business, không technical]
- [VD: "Traffic bot chiếm phần lớn session trong khung giờ", "Domain tracking script bị gỡ khỏi site"]

```

### AC Format Rules
- AC ID: `AC.01`, `AC.02`, … sequential per story.
- AC ID + one-line title trên cùng dòng (English). **Title phải bold.**
- Given / When / Then / And — mỗi keyword trên dòng riêng, kết thúc bằng dấu phẩy (`,`). Dòng cuối cùng không có dấu phẩy.
- One action per When — nếu 2 actions, split thành 2 ACs.
- Mỗi AC mô tả one observable behavior.
- Khi AC cover multi-option fields (dropdown, filter), embed table (UI Label | API value | Description) bên trong AC body.
- Chỉ viết AC cho behavior quan trọng, xảy ra thường xuyên, và QC verify được rõ ràng. Bỏ qua case hiếm gặp, khó test, hoặc behavior nhỏ không ảnh hưởng giá trị core.
- Edge Cases mô tả tình huống từ góc business, không quote lại nguyên văn error message.

---

## Out of Scope Section

```
## Out of Scope

- **[Feature / case]** — [reason it is out of scope / which ticket covers it].
- **[Feature / case]** — [reason it is out of scope / which ticket covers it].
```

### Rules cho Scope
- **Không viết "In Scope"** — User Stories đã mô tả đầy đủ những gì ticket cover. Viết In Scope sẽ bị trùng lặp.
- **Chỉ viết "Out of Scope"** — liệt kê rõ những gì KHÔNG nằm trong ticket này.

### Out of Scope Checklist
Luôn xem xét các items sau:
- Tính năng liên quan nhưng thuộc module/ticket khác
- Batch / bulk operations
- Historical data backfill / migration
- Tính năng defer sang phase sau
