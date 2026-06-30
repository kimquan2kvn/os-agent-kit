# AI Agent Workflow — Team Development Guide

> Tài liệu này mô tả quy trình làm việc khi cả team sử dụng AI agent (Claude Code) để phát triển nhiều feature song song. Mục tiêu: đảm bảo chất lượng nhất quán, kiểm soát được scope, và không bị conflict giữa các feature.

---

## Tổng quan

| Bước | Người thực hiện | Mô tả |
|------|----------------|-------|
| 1. Brainstorm | Dev + Agent | Khám phá feature, viết `feature-context.md` |
| 2. Plan approval | **Dev Lead** | Duyệt scope, check conflict |
| 3. Dev | Dev + Agent | Code theo plan đã approve |
| 4. Soạn test scope | Tester + Agent | Draft test cases từ `feature-context.md` |
| 5. Test scope review | **Dev Lead** | Bổ sung context hệ thống, approve test scope |
| 6. Test execution | Tester | Chạy test theo scope đã approve |
| 7. Merge review | **Dev Lead** | Scope · quality · conflict check → merge |
| 8. Docs update | Dev | Cập nhật repo docs, conventions |

**Nguyên tắc cốt lõi:** Dev Lead không cần xem tất cả những gì agent làm — chỉ kiểm soát tại 3 gate: Plan approval, Test scope review, và Merge review.

---

## Bước 1 — Brainstorm

**Người thực hiện:** Dev + Agent  
**Trigger:** Có feature mới cần làm  
**Thời gian:** 30–60 phút

### Dev làm gì
Cùng agent brainstorm feature, sau đó tạo file `feature-context.md` trong branch của mình. File này là "bản hợp đồng" giữa dev và agent — xác định rõ scope trước khi bắt đầu code.

### Output: `feature-context.md`

```markdown
# Feature: [Tên feature]

## Scope
[Mô tả ngắn gọn feature làm gì, giới hạn ở đâu]

## Files sẽ bị ảnh hưởng
- path/to/file1.ts
- path/to/file2.ts

## Acceptance criteria
- [ ] Criterion 1
- [ ] Criterion 2

## Decisions đã đưa ra
- [Quyết định kỹ thuật quan trọng và lý do]

## Out of scope
- [Những gì KHÔNG làm trong feature này]

## Dependencies
- [Feature hoặc module nào liên quan]
```

### Lưu ý quan trọng
- `feature-context.md` phải viết xong **trước** khi ping Dev Lead
- Không cho agent chạy dev khi chưa có Plan approval
- Repo docs chính (CLAUDE.md, architecture) **không được sửa** trong lúc đang dev

---

## Bước 2 — Plan approval

**Người thực hiện:** Dev Lead  
**Trigger:** Dev ping sau khi có `feature-context.md`  
**Thời gian:** ~5 phút/feature

### Dev Lead kiểm tra 3 điểm

**1. Scope đúng không?**
- Feature này có giải quyết đúng vấn đề không?
- Scope có quá rộng hoặc quá hẹp không?

**2. Conflict với feature khác không?**
- File nào đang được người khác đụng vào?
- Shared module, schema, API contract có bị ảnh hưởng không?

**3. Risk cao ở đâu?**
- Task nào cần theo dõi kỹ hơn trong quá trình dev?

### Output
- Comment approve trên MR với note nếu có
- `feature-context.md` được cập nhật nếu cần điều chỉnh scope

### Nếu không approve
Dev cập nhật lại `feature-context.md` và ping lại — không bắt đầu dev khi chưa có approve.

---

## Bước 3 — Dev

**Người thực hiện:** Dev + Agent  
**Trigger:** Plan approval từ Dev Lead  
**Thời gian:** Tùy feature

### Dev làm gì
- Cho agent đọc `feature-context.md` trước khi bắt đầu bất kỳ task nào
- Agent code theo đúng scope đã approve
- Dev review output của agent sau mỗi task, không để agent chạy dài mà không check
- Push branch, tạo MR draft khi xong

### Quy tắc với agent trong bước này

```
Đọc feature-context.md trước khi bắt đầu.
Không tự sửa file ngoài danh sách "Files sẽ bị ảnh hưởng".
Sau mỗi session, cập nhật feature-context.md với decisions mới nếu có.
```

### Output
- Branch `feature/[tên-feature]`
- MR draft với description tóm tắt những gì agent đã làm
- Pipeline green (unit tests pass)

---

## Bước 4 — Tester soạn test scope

**Người thực hiện:** Tester + Agent  
**Trigger:** Dev báo branch đã sẵn sàng để test  
**Thời gian:** 20–30 phút

### Tester làm gì
Đọc `feature-context.md` của feature, dùng agent để draft test cases, tạo file `test-scope.md`.

### Output: `test-scope.md`

```markdown
# Test scope: [Tên feature]

## Happy path
- [ ] Test case 1: [mô tả]
- [ ] Test case 2: [mô tả]

## Edge cases
- [ ] Test case 3: [mô tả]
- [ ] Test case 4: [mô tả]

## Out of scope (không test trong lần này)
- [Ghi rõ để tránh test nhầm]

## Data setup cần thiết
- [Dữ liệu cần chuẩn bị trước khi test]

## Môi trường
- [ ] Staging
- [ ] Production-like data
```

### Lưu ý
- Chưa bắt đầu test khi chưa có Test scope approval từ Dev Lead
- Nếu không chắc về behavior nào đó, ghi vào phần câu hỏi trong `test-scope.md`

---

## Bước 5 — Test scope review

**Người thực hiện:** Dev Lead  
**Trigger:** Tester ping sau khi có `test-scope.md`  
**Thời gian:** ~10 phút/feature

Đây là gate quan trọng để bù đắp knowledge gap của tester về hệ thống. Dev Lead không chỉ approve mà còn **bổ sung context** mà tester không thể tự biết.

### Dev Lead làm 3 việc

**1. Thêm test case còn thiếu dựa trên kiến thức hệ thống**

Tester mới thường bỏ sót các case liên quan đến:
- Background job / cron job
- Concurrent requests / race condition
- Behavior khác nhau giữa các loại user/store
- Integration với module khác trong hệ thống

**2. Đánh dấu rõ out-of-scope**

Ghi rõ những gì tester không nên test lần này để tránh lãng phí effort và raise bug nhầm feature.

**3. Truyền gotcha của hệ thống**

- Môi trường staging có điểm gì khác production không?
- Data setup đặc biệt nào cần thiết?
- Feature này có phụ thuộc vào feature khác đang develop song song không?

### Output
- `test-scope.md` được approve và bổ sung notes
- Comment trên MR với context bổ sung cho tester

---

## Bước 6 — Test execution

**Người thực hiện:** Tester  
**Trigger:** Test scope approval từ Dev Lead  
**Thời gian:** Tùy feature

### Tester làm gì
- Chạy test theo đúng `test-scope.md` đã approve
- Ghi kết quả pass/fail vào từng test case
- Tạo bug report nếu có, link vào MR

### Output: Test report

```markdown
# Test report: [Tên feature]

## Kết quả tổng quan
- Pass: X/Y cases
- Fail: Z cases

## Chi tiết
| Test case | Kết quả | Ghi chú |
|-----------|---------|---------|
| Happy path 1 | ✅ Pass | |
| Edge case 2  | ❌ Fail | Bug #123 |

## Bugs tìm thấy
- Bug #123: [mô tả ngắn] — [link issue]
```

---

## Bước 7 — Merge review

**Người thực hiện:** Dev Lead  
**Trigger:** Test pass, tester confirm sẵn sàng merge  
**Thời gian:** ~10 phút/feature

Merge review với agent-generated code khác với PR review thông thường. Câu hỏi không phải "code này đúng không" mà là **"agent có làm đúng những gì đã approve không"**.

### 3 lớp kiểm tra theo thứ tự

#### Lớp 1 — Scope check (làm trước tiên)

Mở `feature-context.md` và diff cạnh nhau, kiểm tra:
- Agent có làm đúng những gì đã approve ở bước 2 không?
- Agent có tự ý thêm gì ngoài scope không? *(over-scope)*
- Agent có làm thiếu so với plan không? *(under-scope)*

**Nếu fail:** Yêu cầu revert phần ngoài scope — không refactor, không patch. Agent làm lại từ plan.

> **Lưu ý về under-scope:** Agent đôi khi làm thiếu nhưng test vẫn pass vì agent cũng viết test vừa đủ để match implementation. Cách phát hiện: đọc lại acceptance criteria trong `feature-context.md`, không đọc test file.

#### Lớp 2 — Quality check

Đọc code theo pattern, không cần đọc từng dòng:
- Test có cover đủ case đã approve trong `test-scope.md` không?
- Có hardcode magic value (number, string, URL) không?
- Có logic nào bất thường mà agent tự thêm vào không?

**Nếu fail:** Liệt kê cụ thể danh sách fix, dev cho agent chạy lại với list đó.

#### Lớp 3 — Conflict check

Nhìn diff và kiểm tra:
- File này có ai khác trong team đang đụng vào không?
- Có shared utility, database migration, API schema nào bị thay đổi không?
- Merge order có ảnh hưởng đến feature khác không?

**Nếu conflict:** Họp 2 dev liên quan 15 phút để quyết định ai merge trước, không tự merge một cái rồi yêu cầu người kia rebase.

### Output
- MR approved và merge vào main
- Decision log ghi lại nếu có revert hoặc điều chỉnh

---

## Bước 8 — Docs update

**Người thực hiện:** Dev  
**Trigger:** Feature đã merge  
**Thời gian:** 10–15 phút

Bước này hay bị bỏ qua nhưng là thứ làm cho toàn bộ hệ thống tự cải thiện theo thời gian. Agent của các feature sau sẽ đọc docs này — nếu không update thì agent ngày càng đưa ra quyết định lệch với thực tế codebase.

### Cập nhật 3 chỗ

**1. Architecture docs**
Nếu feature thêm table mới, API mới, hay đổi flow — cập nhật để agent sau không bị conflict.

**2. Conventions**
Nếu trong quá trình dev agent đưa ra quyết định kỹ thuật mới (ví dụ: chọn library mới, pattern mới) — ghi vào conventions để tất cả agent sau làm consistent.

**3. `TEAM_STATUS.md`**
Cập nhật trạng thái feature sang "Done" và ghi tóm tắt ngắn về những gì đã làm.

---

## Cách team theo dõi trạng thái hằng ngày

Mỗi dev cập nhật `TEAM_STATUS.md` ở root repo mỗi sáng, format ngắn gọn:

```markdown
# Team status — [ngày]

Dev A — feature/lock-ip       — Dev      — agent đang viết IP matcher
Dev B — feature/analytics     — Test     — chờ Test scope review
Dev C — feature/reserve       — Brainstorm — chờ Plan approval
Dev D — feature/seo           — Plan     — agent đang tạo tasks
```

Dev Lead mở file này mỗi sáng là biết ngay toàn cảnh.

**Trigger để ping Dev Lead:**
- Sau bước 1 → ping để xin Plan approval
- Sau bước 4 → ping để xin Test scope review
- Sau bước 6 → ping để xin Merge review

---

## Shared context — quy tắc không thay đổi

Repo docs chính (CLAUDE.md, architecture, conventions) là **shared context** mà tất cả agent của mọi người đều đọc. Để đảm bảo tính nhất quán:

- Không ai được sửa shared context khi đang trong workflow của một feature
- Chỉ cập nhật shared context ở bước 8 — sau khi feature đã merge
- Nếu cần thay đổi convention khẩn cấp, báo Dev Lead để thông báo toàn team trước

---

## Tóm tắt vai trò

### Dev
- Viết `feature-context.md` đầy đủ trước khi ping Dev Lead
- Không cho agent chạy dev khi chưa có Plan approval
- Review output của agent sau mỗi task, không để agent chạy mà không check
- Cập nhật `TEAM_STATUS.md` mỗi sáng
- Cập nhật docs sau khi merge

### Tester
- Đọc `feature-context.md` trước khi soạn test scope
- Không bắt đầu test khi chưa có Test scope approval
- Ghi rõ pass/fail và bug report trong test report
- Hỏi khi không chắc về behavior — đừng tự đoán

### Dev Lead
- Review và approve tại 3 gate: Plan, Test scope, Merge
- Tổng thời gian: ~25 phút/feature/ngày
- Bổ sung context hệ thống cho tester tại Test scope review
- Quyết định conflict resolution khi 2 feature đụng nhau
- Cập nhật shared context sau khi sprint kết thúc

---

*Workflow này được thiết kế cho team 4 người dùng Claude Code. Điều chỉnh gate và template theo thực tế của team khi cần.*
