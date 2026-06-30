# Login to View Price — Architecture & Quick Reference

## 1. Feature Architecture Diagram

```
Rule với target_type = PRICE (8) | PRICE_CART_BTN (10) | CART_BTN (9)
  + condition signed_in / customer_tag / passcode / ...
        │
        ▼
getContentHideElement(rule, configShop, customerAccountInfo)
  → Sinh liquid block theo từng target_type
  → Inject vào bssLockContentHideElement
  → Upload lên snippets/bss-lock-content-element.liquid
        │
        ▼
Shopify theme runtime:
  bss_lock_action == 'bss_hide_price'
    └─► {% render 'bss-lock-condition', scope: 'subject', subject: product,
                                        variable: 'bss_lock_ids',
                                        bss_lock_action: 'bss_hide_price' %}
          → access_denied = true → renders bss-lock-content-element
          → Hiện replacement element (link login / message / passcode form)
```

---

## 2. `target_type` liên quan

| Hằng số | Giá trị | Mô tả |
|---------|---------|-------|
| `OTargetType.PRICE` | `8` | Ẩn giá sản phẩm |
| `OTargetType.PRICE_CART_BTN` | `10` | Ẩn giá + nút Add to Cart |
| `OTargetType.CART_BTN` | `9` | Ẩn nút Add to Cart (có option hiện message) |
| `OTargetType.VARIANT` | `6` | Ẩn giá theo variant cụ thể |

---

## 3. Fields của Rule liên quan

### 3.1 Text messages

| Field | Default | Mô tả |
|-------|---------|-------|
| `hide_price_message` | `Login view prices` | Text hiện trong link/button khi chưa đăng nhập (DESIGN_MODE) |
| `element_access_denied_message` | `Price is locked` | Text hiện khi đã đăng nhập nhưng bị từ chối (sai tag, v.v.) |

### 3.2 Design config (`rule.design.hide_price`)

| Field | Type | Default | Mô tả |
|-------|------|---------|-------|
| `editor_mode` | int | `0` | `0` = DESIGN_MODE (dùng styling), `1` = HTML_MODE (dùng html_content) |
| `hide_price_redirect_type` | int | `0` | `0` = LOGIN (redirect đến trang login), `1` = URL (custom URL) |
| `hide_price_redirect_url` | string | `''` | Custom URL khi `hide_price_redirect_type = 1` |
| `html_content` | string | `<a href="/account/login">Login view price</a>` | HTML tùy chỉnh khi `editor_mode = 1` |
| `font_size` | number | `14` | Cỡ chữ (px) |
| `text_format` | string | `"normal"` | `"normal"` / `"bold"` / `"italic"` / `"underline"` |
| `alignment` | string | `"center"` | `"left"` / `"center"` / `"right"` |
| `border_radius` | number | `12` | Border radius (px) |
| `border_thickness` | number | `1` | Border thickness (px) |
| `background_color` | RGBA | trắng | Màu nền |
| `border_color` | RGBA | xám | Màu border |
| `text_color` | RGBA | đen | Màu chữ |
| `custom_css` | string | `""` | CSS tùy chỉnh, áp dụng cho `.bss-lock-message-element-<ruleId>` |

---

## 4. `editor_mode` — 2 chế độ hiển thị

### `DESIGN_MODE` (editor_mode = 0)

Hiển thị link dạng `<a>` với `hide_price_message` làm text, styling được sinh từ các trường design.

```html
<a class="bss-lock-message-element-{{ lock_id }}"
   href="<hidePriceRedirectUrl>"
   target="_blank">
  Login view prices
</a>
```

CSS được build từ `ltspMessageClassTransform(design.hide_price.custom_css, ruleId)`:
- Replace `.bss-lock-message-element` → `.bss-lock-message-element-<ruleId>`
- Đảm bảo mỗi rule có CSS isolated

### `HTML_MODE` (editor_mode = 1)

Merchant tự viết HTML, được inject trực tiếp vào element:

```html
<!-- html_content từ design, ví dụ: -->
<a href="/account/login" target="_blank">Login view price</a>
```

Translation support: nếu `active_translation = true` → dùng `{{ bss_translation['<ruleId>'].hide_price_html_content }}`

---

## 5. `hide_price_redirect_type` — Redirect URL

Chỉ dùng khi `editor_mode = 0` (DESIGN_MODE).

| Giá trị | Hằng số | URL redirect |
|---------|---------|-------------|
| `0` | `LOGIN` | `<loginUrl>?checkout_url={{ bss_lock_ltsp_redirect_url }}` (current page) |
| `1` | `URL` | `hide_price_redirect_url` (fixed URL do merchant nhập) |

**`loginUrl`** phụ thuộc customer account type:
- **Legacy**: `{{ routes.account_login_url }}?checkout_url=`
- **New**: `/customer_authentication/login?return_to=`

**`bss_lock_ltsp_redirect_url`** được set trong `getRedirectUrl(rule, redirectUrlForElement: true)`:
```liquid
{% unless bss-lock-ltsp-redirect-url %}
  {% assign bss-lock-ltsp-redirect-url = canonical_url | remove: shop.url %}
{% endunless %}
```

---

## 6. Liquid Builder — `getContentHideElement()`

Hàm build liquid cộng dồn cho tất cả rules, inject vào `bssLockContentHideElement`.  
Trigger khi `bss_lock_action == 'bss_hide_price'` trong theme.

### 6.1 Cấu trúc chung (case PRICE / PRICE_CART_BTN)

```liquid
{% if current_lock_element_id == '<ruleId>' and bss_lock_action == 'bss_hide_price' %}
  {% if bss_lock_access_denied %}
    {% assign lock_id = '<ruleId>' %}

    {# Capture replacement element — dùng lại nhiều lần theo condition_type #}
    {% capture bss_lock_ltsp_<ruleId> %}
      <span class="bss-lock-element">
        {# editor_mode = 1: dùng html_content trực tiếp #}
        {{ bss_translation['<ruleId>'].hide_price_html_content }}

        {# editor_mode = 0: dùng link với hide_price_message #}
        <a class="bss-lock-message-element-{{ lock_id }}"
           href="<hidePriceRedirectUrl>"
           target="_blank">
          Login view prices
        </a>
      </span>
    {% endcapture %}

    {# Render theo condition_type (xem section 6.2) #}
    ...

    {# Analytics script — track condition type + rule id #}
    {%- capture authentication -%}{"domain":"{{ shop.permanent_domain }}","ruleId":<ruleId>,...}{%- endcapture -%}
    <script id="bss-analytic-script" style="display:none!important;">...</script>

  {% endif %}
{% endif %}
```

### 6.2 Phân nhánh theo `condition_type` khi `access_denied = true`

```liquid
{% if bss_lock_requires_customer or condition_type == 'signed_in' %}
  {# Guest chưa đăng nhập → hiện link login #}
  {% assign bss_condtion_type = "signed_in" %}
  {{ bss_lock_ltsp_<ruleId> }}

{% elsif condition_type == 'start_date' %}
  {# Rule chưa đến ngày hiệu lực → countdown hoặc access denied #}
  {% assign bss_condtion_type = "start_date" %}
  <span class="bss-lock-element">
    {# hasCountdown = true → getClockCountdownContent(rule) #}
    {# hasCountdown = false → element_access_denied_message #}
    <a class='bss-lock-price-access-deniend-<ruleId>'>Price is locked</a>
  </span>

{% elsif bss_lock_requires_passcode %}
  {# Cần nhập passcode → hiện passcode form (xem docs passcode) #}
  {% assign bss_condtion_type = "passcode" %}
  {% render 'bss-lock-passcode-form-price', ... %}

{% elsif bss_lock_requires_customer_tag or condition_type == 'customer_tag' %}
  {# Logged-in: sai tag → access denied; Guest → link login #}
  {% assign bss_condtion_type = "customer_tag" %}
  {% if customer %}
    <span class='bss-lock-price-access-deniend-<ruleId>'>Price is locked</span>
  {% else %}
    {{ bss_lock_ltsp_<ruleId> }}
  {% endif %}

{% elsif condition_type == 'customer_specific' or condition_type == 'customer_email' %}
  {# Tương tự customer_tag #}
  {% if customer %}
    <span class='bss-lock-price-access-deniend-<ruleId>'>Price is locked</span>
  {% else %}
    {{ bss_lock_ltsp_<ruleId> }}
  {% endif %}

{% else %}
  {# Các condition khác (region, secret_link...) → luôn hiện access denied #}
  {% assign bss_condtion_type = condition_type %}
  <a class='bss-lock-price-access-deniend-<ruleId>'>Price is locked</a>
{% endif %}
```

### 6.3 Sự khác biệt giữa PRICE và CART_BTN

| target_type | `bss_lock_action` | Điều kiện hiện message |
|-------------|------------------|----------------------|
| `PRICE` (8) | `bss_hide_price` | Luôn hiện replacement khi `access_denied` |
| `PRICE_CART_BTN` (10) | `bss_hide_atc` + `bss_hide_price` | Ẩn cả giá lẫn nút ATC |
| `CART_BTN` (9) | `bss_hide_atc` | Chỉ hiện message nếu `rule.show_message_atc = true` |

---

## 7. CSS Isolation — `ltspMessageClassTransform()`

Hàm `ltspMessageClassTransform(customCSS, ruleId, editor_mode)` trong `rule.utils.ts`:

```typescript
// custom_css từ design:
".bss-lock-message-element { font-size: 14px; color: red; }"

// Sau transform (editor_mode = 0):
".bss-lock-message-element-123 { font-size: 14px; color: red; }"
```

CSS được inject vào theme cùng với bss-lock-content.liquid, đảm bảo mỗi rule có selector riêng không bị conflict.

> Nếu `editor_mode = 1` (HTML_MODE) → `ltspMessageClassTransform` return nguyên CSS, không transform.

---

## 8. Mapping: Field ↔ Nơi sử dụng trong Liquid

| Field | Dùng khi | Thay thế bằng |
|-------|---------|--------------|
| `hide_price_message` | `editor_mode = 0` | Text trong `<a>` link |
| `element_access_denied_message` | Logged-in nhưng bị từ chối | Text trong `.bss-lock-price-access-deniend-<id>` |
| `design.hide_price.html_content` | `editor_mode = 1` | HTML inject trực tiếp |
| `design.hide_price.hide_price_redirect_type` | `editor_mode = 0` | Quyết định redirect URL |
| `design.hide_price.hide_price_redirect_url` | `redirect_type = 1` | Custom URL |
| `design.hide_price.custom_css` | Luôn (nếu có) | CSS inject qua `ltspMessageClassTransform` |

Translation override (nếu `configShop.active_translation = true`):
- `rule.hide_price_message` → `{{ bss_translation['<id>'].hide_price_message }}`
- `rule.element_access_denied_message` → `{{ bss_translation['<id>'].element_access_denied_message }}`
- `design.hide_price.html_content` → `{{ bss_translation['<id>'].hide_price_html_content }}`

---

## 9. Quick Decision Tree

```
Merchant muốn ẩn giá sản phẩm:
        │
        ├─► Ẩn giá (text replacement)?          → target_type = PRICE (8)
        ├─► Ẩn giá + nút ATC?                   → target_type = PRICE_CART_BTN (10)
        ├─► Ẩn nút ATC?                         → target_type = CART_BTN (9)
        └─► Ẩn giá theo variant cụ thể?         → target_type = VARIANT (6)

Merchant muốn chọn cách hiện thay thế khi bị ẩn:
        ├─► editor_mode = 0 (DESIGN_MODE):
        │   ├─► hide_price_redirect_type = 0 → link login, return to current page
        │   └─► hide_price_redirect_type = 1 → link đến hide_price_redirect_url
        │
        └─► editor_mode = 1 (HTML_MODE):
            └─► html_content → HTML tùy chỉnh hoàn toàn

Khi access_denied, nội dung hiển thị phụ thuộc condition_type:
        ├─► signed_in (chưa đăng nhập)     → link login (bss_lock_ltsp)
        ├─► customer_tag / customer_email  → link login nếu guest, element_access_denied_message nếu logged-in sai tag
        ├─► passcode (chưa nhập đúng)      → passcode form price (bss-lock-passcode-form-price)
        ├─► start_date (chưa đến ngày)     → countdown clock hoặc element_access_denied_message
        └─► other                          → element_access_denied_message
```
