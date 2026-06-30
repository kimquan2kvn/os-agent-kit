# Git Hooks

Hai hooks trong thư mục này giúp enforce workflow của team tự động.

## Hooks có sẵn

### `prepare-commit-msg`
Tự động thêm prefix `[feature-name]` vào commit message dựa theo tên branch.  
Ví dụ: branch `feature/payment-rule` → commit message thành `[payment-rule] your message`.

### `pre-push`
Chạy trước mỗi lần push:
1. **BLOCK** nếu đang push thẳng lên `main` hoặc `master` — bắt buộc tạo Pull Request
2. **WARNING** (không block) nếu `TEAM_STATUS.md` chưa được cập nhật trong 24 giờ

---

## Cách cài đặt

### Cách 1 — Khuyến nghị: dùng `core.hooksPath`

```bash
git config core.hooksPath hooks
```

Git sẽ đọc hooks trực tiếp từ thư mục `hooks/` trong repo. Không cần copy tay, tự động áp dụng khi clone mới.

### Cách 2 — Copy thủ công vào `.git/hooks/`

```bash
cp hooks/prepare-commit-msg .git/hooks/prepare-commit-msg
cp hooks/pre-push .git/hooks/pre-push
chmod +x .git/hooks/prepare-commit-msg .git/hooks/pre-push
```

---

> **Lưu ý**: Cách 1 (`core.hooksPath`) là lựa chọn tốt hơn vì hooks được version control cùng repo, áp dụng ngay khi clone mà không cần bước thủ công nào thêm.
