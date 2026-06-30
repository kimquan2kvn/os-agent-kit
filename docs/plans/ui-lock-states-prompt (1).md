# Prompt: Update UI for Customer Approval Lock States

## Overview

Feature `access_request` (condition type) có 2 surfaces cần update UI:
- **Lock page** — full-page block, render qua `bss-lock-content.liquid` (method `getContentCustomerApproval`)
- **Lock price** — inline element, render qua `generateContentElementLock()` trong `upload.service.ts`

Storefront state hoàn toàn do **Liquid xác định server-side** — JS chỉ xử lý submit `create-request`, không check state on load.

> **Icon set:** Codebase chưa có Tabler Icons (`ti-*`). Tất cả icon phải dùng **SVG inline**. SVG cho từng state được định nghĩa ở cuối file này.

---

## Liquid class names (phải giữ đúng — dùng cho CSS và JS targeting)

| State | Lock page class | Lock price class |
|---|---|---|
| Not logged in | `bss-lock-page-login-prompt-{rule.id}` | `bss-lock-price-login-prompt-{rule.id}` |
| Pending (form) | `bss-lock-page-request-access-{rule.id}` | `bss-lock-price-request-access-{rule.id}` |
| Submitted | `bss-lock-page-pending-access-{rule.id}` | `bss-lock-price-pending-access-{rule.id}` |
| Rejected | `bss-lock-page-rejected-access-{rule.id}` | `bss-lock-price-rejected-access-{rule.id}` |
| Revoked | `bss-lock-page-revoked-access-{rule.id}` | `bss-lock-price-revoked-access-{rule.id}` |

Màu sắc của từng class được sinh bởi `getStyleRequestAccess()` từ `design.request_access.*_color` — không hardcode inline.

---

## Merchant-configurable text (default values)

Text lấy từ `rule.messages.request_access`. Default values từ `defaultRule` trong `web/client/constants/rule.constant.js`:

| State | Field | Default value |
|---|---|---|
| Not logged in | `login_prompt` | `"Please log in to request access to this content."` |
| Pending (form) | `request_message` | `"Request access to this content."` |
| Submitted | `pending_message` | `"Your request has been submitted. We'll notify you once approved."` |
| Rejected | `rejected_message` | `"Your request was not approved. Contact us for help."` |
| Revoked | `revoked_message` | `"Your access has been revoked. Contact us for help."` |

**Title và status indicator là fixed** — không configurable, không lấy từ `rule.messages`.

---

## Surface 1 — Lock page

### Yêu cầu

Bọc nội dung mỗi state trong một **card** căn giữa trang. Card gồm: SVG icon circle + title (fixed) + subtitle (configurable) + action (nếu có).

---

### State 1 — Not logged in

**HTML hiện tại** (trong `getContentCustomerApproval`):
```liquid
<p class="bss-lock-page-login-prompt-{rule.id}" style="text-align:center;">
  {loginPrompt}
</p>
```

**HTML mới:**
```html
<div class="bss-ca-card">
  <div class="bss-ca-card__icon" style="background:#EEF2FF;">
    <!-- SVG: icon-user-circle, color #3D4FB5 — xem phần SVG bên dưới -->
  </div>
  <p class="bss-ca-card__title">Sign in to continue</p>
  <p class="bss-ca-card__subtitle bss-lock-page-login-prompt-{rule.id}">{loginPrompt}</p>
  <a class="bss-ca-card__btn"
     href="/account/login?return_url={{ request.path | url_encode }}">
    Log in
  </a>
</div>
```

---

### State 2 — Pending / Not yet requested (form)

**HTML hiện tại:**
```html
<div style="text-align:center;">
  <span class='bss-lock-page-request-access-{rule.id}'
    data-request-access='{...}'
    request-access-token-{rule.id}="{{ b2b_ca_token_{rule.id} }}"
    data-customer-id="{{ customer.id }}"
    data-rule-id="{rule.id}"
    data-customer-email="{{ customer.email }}"
    data-customer-name="{{ customer.name }}"
    data-domain="{{ shop.permanent_domain }}"
    data-create-request-url="{SERVER_URL}/customer-approval/public/create-request"
    onclick="document.getElementById('bss-b2b-lock-ca-modal').style.display='flex'"
    style="cursor:pointer;font-size:14px;font-weight:600;">
    {requestMessage}
  </span>
</div>
```

**HTML mới** — giữ nguyên tất cả data attributes và onclick, chỉ thay wrapper:
```html
<div class="bss-ca-card">
  <div class="bss-ca-card__icon" style="background:#f0f0f0;">
    <!-- SVG: icon-lock, color #888 -->
  </div>
  <p class="bss-ca-card__title">Members only</p>
  <p class="bss-ca-card__subtitle bss-lock-page-request-access-{rule.id}">{requestMessage}</p>
  <button class="bss-ca-card__btn"
    data-request-access='{...}'
    request-access-token-{rule.id}="{{ b2b_ca_token_{rule.id} }}"
    data-customer-id="{{ customer.id }}"
    data-rule-id="{rule.id}"
    data-customer-email="{{ customer.email }}"
    data-customer-name="{{ customer.name }}"
    data-domain="{{ shop.permanent_domain }}"
    data-create-request-url="{SERVER_URL}/customer-approval/public/create-request"
    onclick="document.getElementById('bss-b2b-lock-ca-modal').style.display='flex'">
    Request access
  </button>
</div>
```

> Button text "Request access" là **fixed**. `{requestMessage}` configurable chuyển sang subtitle.
> JS selector cũ targeting `<span>` → cập nhật sang `button.bss-ca-card__btn[data-rule-id="{rule.id}"]`.

---

### State 3 — Submitted (pending approval)

**HTML hiện tại:**
```liquid
<p class="bss-lock-page-pending-access-{rule.id}" style="text-align:center;">{pendingMessage}</p>
```

**HTML mới:**
```html
<div class="bss-ca-card">
  <div class="bss-ca-card__icon" style="background:#FAEEDA;">
    <!-- SVG: icon-clock, color #854F0B -->
  </div>
  <p class="bss-ca-card__title" style="color:#633806;">Request submitted</p>
  <p class="bss-ca-card__subtitle bss-lock-page-pending-access-{rule.id}">{pendingMessage}</p>
  <div class="bss-ca-card__status">
    <span class="bss-ca-card__dot" style="background:#EF9F27;"></span>
    <span style="color:#854F0B;">Awaiting approval</span>
  </div>
</div>
```

---

### State 4 — Rejected

**HTML hiện tại:**
```liquid
<p class="bss-lock-page-rejected-access-{rule.id}" style="text-align:center;">{rejectedMessage}</p>
```

**HTML mới:**
```html
<div class="bss-ca-card">
  <div class="bss-ca-card__icon" style="background:#FCEBEB;">
    <!-- SVG: icon-x-circle, color #A32D2D -->
  </div>
  <p class="bss-ca-card__title" style="color:#791F1F;">Request denied</p>
  <p class="bss-ca-card__subtitle bss-lock-page-rejected-access-{rule.id}">{rejectedMessage}</p>
  <div class="bss-ca-card__status">
    <span class="bss-ca-card__dot" style="background:#E24B4A;"></span>
    <span style="color:#A32D2D;">Not eligible</span>
  </div>
</div>
```

---

### State 5 — Revoked

**HTML hiện tại:**
```liquid
<p class="bss-lock-page-revoked-access-{rule.id}" style="text-align:center;">{revokedMessage}</p>
```

**HTML mới:**
```html
<div class="bss-ca-card">
  <div class="bss-ca-card__icon" style="background:#F1EFE8;">
    <!-- SVG: icon-lock-off, color #5F5E5A -->
  </div>
  <p class="bss-ca-card__title" style="color:#2C2C2A;">Access revoked</p>
  <p class="bss-ca-card__subtitle bss-lock-page-revoked-access-{rule.id}">{revokedMessage}</p>
  <div class="bss-ca-card__status">
    <span class="bss-ca-card__dot" style="background:#9E9C95;"></span>
    <span style="color:#5F5E5A;">No longer active</span>
  </div>
</div>
```

---

### Lock page CSS (thêm vào `bss-lock-settings.css`)

```css
.bss-ca-card {
  background: #ffffff;
  border: 0.5px solid #e0ddd8;
  border-radius: 14px;
  padding: 32px 28px;
  max-width: 360px;
  width: 100%;
  text-align: center;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.06);
  margin: 60px auto;
}
.bss-ca-card__icon {
  width: 56px;
  height: 56px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 0 auto 16px;
}
.bss-ca-card__icon svg {
  width: 24px;
  height: 24px;
}
.bss-ca-card__title {
  font-size: 17px;
  font-weight: 500;
  color: #1a1a1a;
  margin: 0 0 8px;
}
.bss-ca-card__subtitle {
  font-size: 14px;
  line-height: 1.55;
  margin: 0 0 20px;
  /* color overridden by getStyleRequestAccess() */
}
.bss-ca-card__btn {
  display: block;
  width: 100%;
  padding: 11px;
  background: #1a1a1a;
  color: #fff;
  border: none;
  border-radius: 99px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  text-decoration: none;
  text-align: center;
}
.bss-ca-card__status {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  font-size: 13px;
}
.bss-ca-card__dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  flex-shrink: 0;
}
```

---

## Surface 2 — Lock price

### Yêu cầu

Thay `<span>` bằng **lock box** — `<div>` flex row, SVG icon trái + 2 dòng text phải.
States 1 & 2 là **clickable** (có `→`, hover). States 3/4/5 là **informational** (không hover, không arrow).

---

### State 1 — Not logged in *(clickable)*

**HTML hiện tại:**
```liquid
<span class='bss-lock-price-login-prompt-{rule.id}'>{loginPrompt}</span>
```

**HTML mới:**
```html
<a class="bss-ca-price-box bss-ca-price-box--clickable bss-lock-price-login-prompt-{rule.id}"
   href="/account/login?return_url={{ request.path | url_encode }}"
   style="background:#EEF2FF;">
  <!-- SVG: icon-lock, color #3D4FB5, size 14px -->
  <div>
    <p class="bss-ca-price-box__title" style="color:#3D4FB5;">Members price</p>
    <p class="bss-ca-price-box__subtitle" style="color:#6B7ACA;">Log in to view</p>
  </div>
</a>
```

---

### State 2 — Pending / Not yet requested *(clickable)*

**HTML hiện tại:**
```liquid
<span class='bss-lock-price-request-access-{rule.id}'
  data-create-request-url="{SERVER_URL}/customer-approval/public/create-request"
  onclick="document.getElementById('bss-b2b-lock-ca-modal').style.display='flex'">
  {requestMessage}
</span>
```

**HTML mới** — giữ nguyên data attribute và onclick:
```html
<div class="bss-ca-price-box bss-ca-price-box--clickable bss-lock-price-request-access-{rule.id}"
     style="background:#f7f7f5;"
     data-create-request-url="{SERVER_URL}/customer-approval/public/create-request"
     onclick="document.getElementById('bss-b2b-lock-ca-modal').style.display='flex'">
  <!-- SVG: icon-lock, color #888, size 14px -->
  <div>
    <p class="bss-ca-price-box__title" style="color:#1a1a1a;">Members only</p>
    <p class="bss-ca-price-box__subtitle" style="color:#999;">Request to unlock price</p>
  </div>
</div>
```

> Subtitle "Request to unlock price" là **fixed text** — không lấy từ `{requestMessage}`.

---

### State 3 — Submitted *(informational)*

**HTML hiện tại:**
```liquid
<span class='bss-lock-price-pending-access-{rule.id}'>{pendingMessage}</span>
```

**HTML mới:**
```html
<div class="bss-ca-price-box bss-lock-price-pending-access-{rule.id}"
     style="background:#FAEEDA;">
  <!-- SVG: icon-clock, color #854F0B, size 14px -->
  <div>
    <p class="bss-ca-price-box__title" style="color:#633806;">Awaiting approval</p>
    <p class="bss-ca-price-box__subtitle" style="color:#854F0B;">We'll notify you by email</p>
  </div>
</div>
```

---

### State 4 — Rejected *(informational)*

**HTML hiện tại:**
```liquid
<span class='bss-lock-price-rejected-access-{rule.id}'>{rejectedMessage}</span>
```

**HTML mới:**
```html
<div class="bss-ca-price-box bss-lock-price-rejected-access-{rule.id}"
     style="background:#FCEBEB;">
  <!-- SVG: icon-x-circle, color #A32D2D, size 14px -->
  <div>
    <p class="bss-ca-price-box__title" style="color:#791F1F;">Request denied</p>
    <p class="bss-ca-price-box__subtitle" style="color:#A32D2D;">Not eligible at this time</p>
  </div>
</div>
```

---

### State 5 — Revoked *(informational)*

**HTML hiện tại:**
```liquid
<span class='bss-lock-price-revoked-access-{rule.id}'>{revokedMessage}</span>
```

**HTML mới:**
```html
<div class="bss-ca-price-box bss-lock-price-revoked-access-{rule.id}"
     style="background:#F1EFE8;">
  <!-- SVG: icon-lock-off, color #5F5E5A, size 14px -->
  <div>
    <p class="bss-ca-price-box__title" style="color:#2C2C2A;">Access revoked</p>
    <p class="bss-ca-price-box__subtitle" style="color:#5F5E5A;">Removed by administrator</p>
  </div>
</div>
```

---

### Lock price CSS (thêm vào `bss-lock-settings.css`)

```css
.bss-ca-price-box {
  display: flex;
  align-items: center;
  gap: 6px;
  border-radius: 6px;
  padding: 6px 8px;
  position: relative;
  text-decoration: none;
}
.bss-ca-price-box svg {
  width: 14px;
  height: 14px;
  flex-shrink: 0;
}
.bss-ca-price-box__title {
  font-size: 13px;
  font-weight: 500;
  margin: 0;
  line-height: 1.3;
}
.bss-ca-price-box__subtitle {
  font-size: 11px;
  margin: 0;
}
/* Clickable modifier — States 1 & 2 only */
.bss-ca-price-box--clickable {
  cursor: pointer;
  transition: opacity 0.15s;
}
.bss-ca-price-box--clickable:hover {
  opacity: 0.75;
}
.bss-ca-price-box--clickable::after {
  content: '→';
  position: absolute;
  right: 8px;
  top: 50%;
  transform: translateY(-50%);
  font-size: 12px;
  opacity: 0.6;
}
```

---

## Display Settings — Live preview (`PreviewRequestAccessPrice` & `PreviewRequestAccessPage`)

Preview render trong modal Display Settings (`isPreviewInRule=false`) cần hiển thị đúng UI mới cho cả 6 modes.

### Mode switcher (6 buttons)

```
[ Request message ] [ Pending message ] [ Rejected message ]
[ Revoked access  ] [ Login prompt    ] [ Modal request    ]
```

Mỗi mode render state tương ứng với HTML/CSS mới đã định nghĩa ở trên.

### CSS màu sắc trong preview

Preview dùng `hex8ToRgba()` sinh inline style từ `design.request_access`:

```css
.request_message  { color: rgba(...) }   /* từ request_message_color */
.pending_message  { color: rgba(...) }   /* từ pending_message_color */
.rejected_message { color: rgba(...) }   /* từ rejected_message_color */
.revoked_message  { color: rgba(...) }   /* từ revoked_message_color */
.login_prompt     { color: rgba(...) }   /* từ login_prompt_color */
{custom_css}
```

Apply class tương ứng lên `bss-ca-card__subtitle` hoặc `bss-ca-price-box__title` trong preview — thay vì dùng `bss-lock-page-*` / `bss-lock-price-*` class (những class đó chỉ dùng trên storefront).

### Scrollable settings panel — default values cập nhật

Cập nhật placeholder text trong `ScrollableRequestAccessPrice` theo default values mới:

| Field | Default value mới |
|---|---|
| Request message | `"Request access to this content."` |
| Pending message | `"Your request has been submitted. We'll notify you once approved."` |
| Rejected message | `"Your request was not approved. Contact us for help."` |
| Revoked message | `"Your access has been revoked. Contact us for help."` |
| Login prompt | `"Please log in to request access to this content."` |

---

## SVG Icons

Tất cả icon inline SVG, `viewBox="0 0 24 24"`, `fill="none"`, `stroke-width="2"`, `stroke-linecap="round"`, `stroke-linejoin="round"`.

### icon-user-circle (Not logged in — lock page)
```svg
<svg viewBox="0 0 24 24" fill="none" stroke="#3D4FB5" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
  <circle cx="12" cy="8" r="4"/>
  <path d="M4 20c0-4 3.6-7 8-7s8 3 8 7"/>
  <circle cx="12" cy="12" r="10"/>
</svg>
```

### icon-lock (Not logged in — lock price / Pending lock page & price)
```svg
<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
  <rect x="3" y="11" width="18" height="11" rx="2" ry="2"/>
  <path d="M7 11V7a5 5 0 0 1 10 0v4"/>
</svg>
```
> Dùng `stroke="currentColor"` — set màu bằng CSS `color` của container.

### icon-clock (Submitted)
```svg
<svg viewBox="0 0 24 24" fill="none" stroke="#854F0B" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
  <circle cx="12" cy="12" r="10"/>
  <polyline points="12 6 12 12 16 14"/>
</svg>
```

### icon-x-circle (Rejected)
```svg
<svg viewBox="0 0 24 24" fill="none" stroke="#A32D2D" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
  <circle cx="12" cy="12" r="10"/>
  <line x1="15" y1="9" x2="9" y2="15"/>
  <line x1="9" y1="9" x2="15" y2="15"/>
</svg>
```

### icon-lock-off (Revoked)
```svg
<svg viewBox="0 0 24 24" fill="none" stroke="#5F5E5A" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
  <rect x="3" y="11" width="18" height="11" rx="2" ry="2"/>
  <path d="M7 11V7a5 5 0 0 1 9.9-1"/>
  <line x1="3" y1="3" x2="21" y2="21"/>
</svg>
```

---

## Notes

- `getStyleRequestAccess()` — không thay đổi logic, chỉ đảm bảo color class apply lên đúng element `bss-ca-card__subtitle` / `bss-ca-price-box__title`.
- `b2b-customer-access.js` — không thay đổi logic. Sau submit thành công, JS replace wrapper cũ bằng HTML state Submitted mới (dùng `bss-ca-card` cho lock page, `bss-ca-price-box` cho lock price).
- `Add to cart` và `Buy it now` — ẩn hoàn toàn khi price bị lock (tất cả 5 states trừ approved).
- State `approved` → render bình thường, không có lock UI.
