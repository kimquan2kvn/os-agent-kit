# Docs Index — B2B Lock

## Cách dùng

| Bạn là | Muốn | Đọc |
|--------|------|-----|
| Dev | Hiểu flow kỹ thuật, code path, API endpoint | `feature/` |
| Dev / BA | Hiểu tính năng từ góc độ nghiệp vụ, use case merchant | `modules/` |
| Dev | Kế hoạch cải tiến, task đang làm | `plans/` |

---

## `feature/` — Tài liệu kỹ thuật

Dùng khi code hoặc debug. Mô tả architecture, flow từ request → DB → upload theme, entity liên quan.

| File | Nội dung |
|------|----------|
| `rules.md` | Rules engine: API endpoints, entity, flow tạo/cập nhật rule |
| `upload-content.md` | Upload Liquid/CSS lên Shopify theme — entry point chính của hệ thống |
| `hide-section.md` | Architecture hide block/section (target_type: 11) |
| `login-view-page.md` | Login to view page — flow render Liquid, condition signed_in |
| `login-view-price.md` | Login to view price — target_type PRICE/PRICE_CART_BTN/CART_BTN |
| `passcode-view-page.md` | Passcode flow — verify, grant access, lưu usage history |
| `app-proxy.md` | App Proxy endpoints, guard, condition ping |
| `webhooks.md` | Webhook handlers (uninstall, theme publish, shop update) |

---

## `modules/` — Tài liệu nghiệp vụ

Dùng khi cần hiểu tính năng dưới góc nhìn merchant hoặc onboarding dev mới. Không chứa code path.

| File | Nội dung |
|------|----------|
| `merchant-app.md` | Tổng quan dashboard merchant — các section chính |
| `lock-rules.md` | Rule Builder — what to lock, who can access, lock experience |
| `checkout-lock.md` | Block checkout theo điều kiện khách hàng |
| `hide-payment-methods.md` | Ẩn phương thức thanh toán tại checkout |

---

## `plans/` — Kế hoạch & Tasks đang thực hiện

Dùng trước khi bắt đầu làm feature phức tạp. Ghi rõ bối cảnh, vấn đề, scope, approach.

| File | Nội dung |
|------|----------|
| `location-condition-improvement.md` | Cải tiến điều kiện location (countries/regions) — đang planning |

---

## Quy ước thêm doc mới

- **Feature kỹ thuật mới** → tạo `feature/[tên-feature].md`
- **Module nghiệp vụ mới** → tạo `modules/[tên-module].md`
- **Task / cải tiến phức tạp** → tạo `plans/[tên-task].md` trước khi code, update sau khi xong
- Sau khi implement xong, update doc nếu flow thay đổi so với spec ban đầu
