# Login to View Page — Architecture & Quick Reference

## 1. Feature Architecture Diagram

```
Rule với condition type = 'signed_in'
        │
        ▼
getRuleCondition() → getContentKeysByRule() → getContentConditionsByKey()
  case 'signed_in':
    {% if customer %} → temp_<conditionId> = true  (đã đăng nhập)
    signedInCloseTag → requires_customer = true     (chưa đăng nhập)
        │
        ▼
bss-lock-condition.liquid (runtime trên Shopify)
  access_denied = true → requires_customer = true
        │
        ▼
bss-lock-content.liquid
  └─► lock_content_guest   (via getLockContentGuest)    → hiện thông báo/form cho guest
  └─► lock_content_logged  (via getLockAccessDeniedContent) → hiện thông báo cho logged-in nhưng bị từ chối
```

---

## 2. Fields của Rule liên quan

### 2.1 Điều kiện hiển thị nội dung cho Guest (chưa đăng nhập)

| Field | Type | Mô tả |
|-------|------|-------|
| `login_type` | int | Loại hành động khi guest truy cập (0–4) |
| `non_login_message` | text | Nội dung hiện khi `login_type=0`, chứa `{login_form}` placeholder |
| `register_message` | text | Nội dung hiện khi `login_type=1`, chứa `{registration_form}` placeholder |
| `custom_message` | text | Nội dung hiện khi `login_type=2`, chứa link `/account/login` và `/account/register` |
| `custom_page_url` | text | URL redirect khi `login_type=3` |
| `fl_redirect_type` | int | Kiểu redirect sau login (dùng cho `login_type=2`) |
| `fl_redirect_url` | varchar(255) | URL đích khi `fl_redirect_type=1` hoặc fallback của type 3 |
| `fl_redirect_config` | json | Array `[{customer_tag, url}]` — redirect per-tag khi `fl_redirect_type=3` |

### 2.2 Điều kiện hiển thị cho Logged-in nhưng bị từ chối

| Field | Type | Mô tả |
|-------|------|-------|
| `not_applied_customer_message` | text | Message hiện khi khách đã đăng nhập nhưng không thỏa condition khác (sai tag, v.v.) |

---

## 3. `login_type` — Các loại hành động với Guest

| Giá trị | Hành vi | Field dùng |
|---------|---------|-----------|
| `0` | Hiện form đăng nhập inline | `non_login_message` (chứa `{login_form}`) |
| `1` | Hiện form đăng ký inline | `register_message` (chứa `{registration_form}`) |
| `2` | Hiện custom message với link login/register, xử lý redirect sau login | `custom_message` + `fl_redirect_type` + `fl_redirect_url` |
| `3` | Redirect thẳng đến `custom_page_url` | `custom_page_url` |
| `4` | Redirect đến `routes.account_login_url` (Shopify native) | — |

---

## 4. `fl_redirect_type` — Redirect sau khi đăng nhập (chỉ dùng khi `login_type = 2`)

| Giá trị | Sau khi login redirect về | Ghi chú |
|---------|--------------------------|---------|
| `0` | Trang hiện tại (canonical URL) | `canonical_url \| remove: shop.url` |
| `1` | `fl_redirect_url` (URL cố định) | Merchant tự nhập |
| `2` | `/account` | Trang account của Shopify |
| `3` | URL theo customer tag (per-tag config) | Đọc từ `fl_redirect_config` |

### `fl_redirect_config` (khi `fl_redirect_type = 3`)

```json
[
  { "customer_tag": "wholesale", "url": "/pages/wholesale" },
  { "customer_tag": "vip",       "url": "/pages/vip" }
]
```

> Nếu customer không khớp tag nào → fallback về `fl_redirect_url`.  
> Logic redirect được thực thi tại `/pages/bss-redirect-customer` qua `getScriptRedirectByTag()`.

---

## 5. New vs Legacy Customer Account

Shopify có 2 hệ thống customer account:
- **Legacy**: `settingsNewCustomerAccount = false` và `themeSupportLegacy = true`
- **New**: `settingsNewCustomerAccount = true` hoặc `themeSupportLegacy = false`

`getMessageNonLogin(rule, configShop, customerAccountInfo)` trong `rule.utils.ts` tự động xử lý:

| Hệ thống | Login URL | Register URL |
|----------|-----------|-------------|
| Legacy | `/account/login?checkout_url=<returnUrl>` | `/account/register?checkout_url=<returnUrl>` |
| New | `/customer_authentication/login?return_to=<returnUrl>` | `/customer_authentication/login?return_to=<returnUrl>` |

---

## 6. Liquid Builder — `getRuleCondition()` với `signed_in`

### Bước 1 — Kiểm tra đăng nhập

```liquid
{# Sinh ra bởi case 'signed_in' trong getContentConditionsByKey() #}
{% assign temp_<conditionId> = false %}
{% if customer %}
  {% assign temp_<conditionId> = true %}
{% endif %}
```

Nếu condition có `inverse = 1` (rule áp dụng cho người **đã** đăng nhập):
```typescript
checkCustomerSignedIn = `temp_${condition.id} != true`
```

### Bước 2 — Close tag: set `requires_customer`

```liquid
{# signedInCloseTag — sinh ra ở getContentKeysByRule() #}
{% else %}
  {% assign current_lock_element_id = temp_lock_element_id %}
  {% assign condition_type = 'signed_in' %}
  {% assign requires_customer = true %}   {# hoặc false nếu inverse #}
{% endif %}
```

### Bước 3 — Cấu trúc đầy đủ của 1 key condition

```liquid
{% unless opened_lock_ids contains ',<ruleId>,' %}
  {# ... targetCondition (product/page/section/...) #}
  {# ... excludePage #}
  {# ... excludeProduct #}
  {% assign lock_ids = lock_ids | append: ',<ruleId>,' %}

  {# redirect URL #}
  {% unless bss-lock-fl-redirect-url %}
    {% assign bss-lock-fl-redirect-url = <targetRedirect> %}
  {% endunless %}

  {# signed_in condition #}
  {% assign temp_<conditionId> = false %}
  {% if customer %}{% assign temp_<conditionId> = true %}{% endif %}

  {% if temp_<conditionId> %}        {# → user đã đăng nhập #}
    {% assign opened_lock_ids = opened_lock_ids | append: ',<ruleId>,' %}
    {# ... các condition khác (customer_tag, passcode, ...) #}
  {% else %}
    {% assign requires_customer = true %}
  {% endif %}

{% endunless %}
```

---

## 7. Liquid Builder — `getLockContentGuest()` → `bssLockContentGuest`

Hàm này build liquid cộng dồn, inject vào `{{ bss_lock_content_guest_rules }}` trong `bss-lock-content.liquid`.

```liquid
{% if lock_id == '<ruleId>' %}
  {% unless lock_content_guest %}
  {% capture lock_content_guest %}

    {# auto-tag khi khách đăng ký (nếu rule có customer_auto_tag) #}
    {%- assign bss_lock_auto_tag = '<tagName>' -%}

    {# Build message từ getMessageNonLogin(rule, configShop, customerAccountInfo) #}
    {% capture bss_lock_check_content %}
      ...nội dung theo login_type...
    {% endcapture %}

    {# Fallback nếu theme dùng new customer account hoặc có Liquid error #}
    {% if routes.account_login_url contains 'https://shopify.com' and bss_lock_check_content contains "{% render 'bss-lock-" %}
      <p class="bss-fl-message" style="text-align: center;">
        Private content, please
        <a href="{{ routes.account_login_url }}">login</a> to show or
        <a href="{{ routes.account_register_url }}">create an account</a>.
      </p>
    {% elsif bss_lock_check_content contains "Liquid error" %}
      {# Render fallback tương tự #}
    {% else %}
      {{ bss_lock_check_content }}
    {% endif %}

  {% endcapture %}
  {% endunless %}
  {% break %}
{% endif %}
```

### Output của `getMessageNonLogin()` theo từng `login_type`

#### `login_type = 0` — Form đăng nhập inline

```liquid
{# non_login_message với {login_form} được thay bằng snippet render #}
<p class="bss-fl-message" ...>Private content, please log in...</p>
<p class="login-form" ...>
  {% render 'bss-lock-login-form' bss_lock_return_to: pageUrlWithHost %}
</p>
```

#### `login_type = 1` — Form đăng ký inline

```liquid
{# register_message với {registration_form} được thay bằng snippet render #}
<p class="bss-fl-message" ...>Private content, please register...</p>
<p class="register-form" ...>
  {% render 'bss-lock-register-form'
    bss_lock_return_to: pageUrlWithHost,
    bss_lock_auto_tag: bss_lock_auto_tag %}
</p>
```

#### `login_type = 2` — Custom message + redirect

```liquid
{# custom_message với link /account/login và /account/register #}
{# Link được modify theo fl_redirect_type: #}

{# fl_redirect_type = 0: return to current page #}
{% assign login_checkout_url = '/account/login?checkout_url=' | append: pageUrlWithHost %}
{{ custom_message | replace: '/account/login', login_checkout_url | replace: '/account/register', register_checkout_url }}

{# fl_redirect_type = 1: return to fl_redirect_url (fixed URL) #}
{{ custom_message | replace: '/account/login', '/account/login?checkout_url=<fl_redirect_url>' }}

{# fl_redirect_type = 2: return to /account #}
{{ custom_message | replace: '/account/login', '/account/login?checkout_url=/account' }}

{# fl_redirect_type = 3: redirect qua /pages/bss-redirect-customer (per-tag logic) #}
{{ custom_message | replace: '/account/login', '/account/login?checkout_url=/pages/bss-redirect-customer' }}
```

> Với New Customer Account thay `/account/login?checkout_url=` bằng `/customer_authentication/login?return_to=`.

#### `login_type = 3` — Redirect thẳng đến `custom_page_url`

```html
<script>
  const customPageURL = '<custom_page_url>';
  window.location.href = customPageURL;
</script>
```

#### `login_type = 4` — Redirect đến Shopify native login

```html
<script>
  window.location.href = '{{ routes.account_login_url }}';
</script>
```

---

## 8. Liquid Builder — `getLockAccessDeniedContent()` → `bssLockAccessDeniedContent`

Dùng để hiển thị message khi khách **đã đăng nhập** nhưng bị từ chối (không đủ điều kiện).  
Inject vào `{{ bss_lock_access_denied_content_rules }}` trong `bss-lock-content.liquid`.

```liquid
{% if lock_id == '<ruleId>' %}
  {% unless lock_content_logged %}
  {% capture lock_content_logged %}
    {# not_applied_customer_message — hoặc translation nếu active_translation #}
    <p style="text-align: center;" class="bss-access-denied-message">
      This content is locked, but it doesn't look like you have access...
    </p>
  {% endcapture %}
  {% endunless %}
  {% break %}
{% endif %}
```

---

## 9. Liquid Builder — Upload snippets liên quan

| Snippet | Điều kiện upload | Mô tả |
|---------|-----------------|-------|
| `snippets/bss-lock-login-form.liquid` | `bssCheckUploadType.login = true` | Form đăng nhập inline |
| `snippets/bss-lock-register-form.liquid` | `bssCheckUploadType.register = true` | Form đăng ký inline |
| `snippets/bss-lock-content.liquid` | Luôn upload | Chứa `{{ bss_lock_content_guest_rules }}` và `{{ bss_lock_access_denied_content_rules }}` |

`bssCheckUploadType.login` = `true` khi tồn tại ít nhất 1 rule có condition `signed_in` với `login_type = 0`.  
`bssCheckUploadType.register` = `true` khi tồn tại ít nhất 1 rule có condition `signed_in` với `login_type = 1`.

---

## 10. `fl_redirect_type = 3` — Redirect theo Customer Tag

Đây là luồng phức tạp nhất, dùng `getScriptRedirectByTag(rule)`:

```
Khách chưa đăng nhập, nhấn link login trong custom_message
  └─► Redirect đến /account/login?checkout_url=/pages/bss-redirect-customer?lock_id=<ruleId>
        │
        ▼
Sau khi đăng nhập thành công, Shopify redirect về:
  /pages/bss-redirect-customer?lock_id=<ruleId>
        │
        ▼
Page bss-redirect-customer chạy liquid từ getScriptRedirectByTag():
  1. Tách lock_id từ query string
  2. Đọc customer.tags → so khớp với fl_redirect_config[].customer_tag
  3. Tìm thấy → redirect_url = fl_redirect_config[].url
  4. Không tìm thấy → redirect_url = fl_redirect_url (fallback)
  5. <script>window.location.href = '{{ redirect_url }}'</script>
```

```liquid
{% assign page_url_with_params_lock_id = page_url_with_params | split: "?lock_id=" %}
{% assign list_tag_check = customer.tags | join: ',' | downcase | split: ',' %}
{% if request.path == '/pages/bss-redirect-customer' and page_url_with_params_lock_id[1] == '<ruleId>' %}
  {% assign redirect_url = '<fl_redirect_url>' %}
  {% for customer_tag in list_tag_check %}
    {% assign check_customer_tag = ',' | append: customer_tag | append: ',' %}
    {% if ",wholesale," contains check_customer_tag %}
      {% assign redirect_url = '/pages/wholesale' %}
      {% break %}
    {% endif %}
    {% if ",vip," contains check_customer_tag %}
      {% assign redirect_url = '/pages/vip' %}
      {% break %}
    {% endif %}
  {% endfor %}
  <script>window.location.href = '{{ redirect_url }}'</script>
{% endif %}
```

---

## 11. Quick Decision Tree

```
Rule cần giới hạn nội dung theo login status
        │
        ├─► Chỉ guest thấy thông báo đăng nhập?
        │   └─► signed_in condition (condition.inverse = 0)
        │
        └─► Chỉ logged-in user thấy (ẩn với guest)?
            └─► signed_in condition (condition.inverse = 1)

Merchant muốn chọn cách hiện thông báo cho guest:
        ├─► login_type = 0  → hiện inline login form (non_login_message)
        ├─► login_type = 1  → hiện inline register form (register_message)
        ├─► login_type = 2  → custom message HTML + redirect sau login
        │   ├─► fl_redirect_type = 0 → return to current page
        │   ├─► fl_redirect_type = 1 → return to fl_redirect_url
        │   ├─► fl_redirect_type = 2 → return to /account
        │   └─► fl_redirect_type = 3 → per-tag redirect (fl_redirect_config)
        ├─► login_type = 3  → redirect thẳng đến custom_page_url
        └─► login_type = 4  → redirect đến routes.account_login_url (Shopify native)

Khách đã đăng nhập nhưng không đủ điều kiện → not_applied_customer_message
```
