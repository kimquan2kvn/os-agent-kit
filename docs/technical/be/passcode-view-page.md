# Passcode Feature — Architecture & Quick Reference

## 1. Feature Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Storefront                                │
│  Khách nhập passcode → JS gọi POST /passcode/save-usage-history  │
│  Khách xin passcode  → JS gọi POST /passcode/public/create-request│
└──────────────┬──────────────────────────────┬───────────────────┘
               │ verify passcode              │ gửi request
    ┌──────────▼──────────┐       ┌───────────▼───────────────┐
    │ save-usage-history  │       │ public/create-request      │
    │ Guard: StoreFront   │       │ @Public() + ThrottlerGuard │
    │ HMAC auth header    │       │ 1 req / 60s / IP           │
    └──────────┬──────────┘       └───────────┬───────────────┘
               │                              │
    ┌──────────▼──────────┐       ┌───────────▼───────────────┐
    │  PasscodeService    │       │  PasscodeService           │
    │  saveUsageHistory() │       │  createRequest()           │
    └──────────┬──────────┘       └───────────┬───────────────┘
               │                              │
    ┌──────────▼──────────┐       ┌───────────▼───────────────┐
    │  DB: UsageHistoryV2 │       │  DB: passcodeRequestsV2    │
    │  status: Success/Fail│      │  status: pending/approved  │
    └─────────────────────┘       │  /rejected/invalid         │
                                  └───────────┬───────────────┘
                                              │ nếu shop bật notification
                                  ┌───────────▼───────────────┐
                                  │  EmailService              │
                                  │  sendCustomerPasswordRequest│
                                  │  Email()                   │
                                  └───────────────────────────┘
```

---

## 2. Passcode Config trong Condition

Passcode được lưu trong `keys.conditions` của rule, với `condition.type = 'passcode'`:

```typescript
condition.configs = {
    passcode: 'abc*bss*xyz*bss*123',   // nhiều passcode cách nhau bằng '*bss*'
    passcode_case_sensitive: 1,         // 1 = phân biệt hoa/thường, 0 = không
    passcode_type: PASSCODE_TYPE.ONE_TO_ALL,
}
```

### 2.1 PASSCODE_TYPE

| Giá trị | Hằng số | Hành vi |
|---------|---------|---------|
| `0` | `ONE_TO_ALL` | Nhập đúng 1 lần → mở khóa tất cả trang trong session |
| `1` | `ONCE_TO_EACH` | Nhập 1 lần cho mỗi page (per-page unlock) |
| `2` | `ONCE_FOR_ONCE` | Phải nhập lại mỗi lần vào page |

### 2.2 Delimiter & Case Sensitivity

```
passcode config: "Hello*bss*world*bss*TEST"
  → split('*bss*') → ['Hello', 'world', 'TEST']

Nếu passcode_case_sensitive = 0 (không phân biệt):
  → convert cả list và input về lowercase rồi so sánh

Nếu passcode_case_sensitive = 1 (phân biệt):
  → so sánh nguyên bản
```

---

## 3. API Endpoints

Controller: `src/modules/module/passcode/passcode.controller.ts` (`@Controller('passcode')`)

### Merchant/Admin (ShopGuard — yêu cầu session-token header)

| Method | Path | Mô tả |
|--------|------|-------|
| `POST` | `/passcode/request` | List/filter passcode requests (có phân trang, lọc ngày) |
| `POST` | `/passcode/save-request` | Tạo mới hoặc cập nhật request (admin tự tạo) |
| `POST` | `/passcode/delete-request` | Xóa 1 request theo `id` |
| `POST` | `/passcode/change-status-requests` | Bật/tắt `status_passcode_requests` của shop |
| `POST` | `/passcode/send-email-to-customer` | Gửi passcode qua email cho khách, cập nhật status → `approved` |
| `GET`  | `/passcode/get-list-name-rule` | Lấy danh sách rule có passcode (đã từng có usage history) |
| `POST` | `/passcode/get-all-usage-history` | Lấy usage history (có filter: date, status, ruleIds, query `?session_id`) |

### Storefront/Public

| Method | Path | Guard | Mô tả |
|--------|------|-------|-------|
| `POST` | `/passcode/public/create-request` | `@Public()` + `ThrottlerGuard` | Khách gửi yêu cầu xin passcode |
| `POST` | `/passcode/save-usage-history` | `VerifyAndValidateStoreFrontGuard` | Ghi lại lịch sử nhập passcode |

---

## 4. Data Flow — Validate Passcode (save-usage-history)

```
POST /passcode/save-usage-history
  Body: { domain, ruleId, passcode, pageUrl, customerId, session_id }
  Header: authentication: Bearer <hash>
    hash = HMAC_SHA256(JSON.stringify({ domain, ruleId }), KEY_HASH_PASSCODE)
        │
        ▼
VerifyAndValidateStoreFrontGuard — xác thực HMAC
        │
        ▼
PasscodeService.saveUsageHistory()
  1. Validate: passcode.length <= 255
  2. Load rule theo ruleId
     JOIN keys → JOIN conditions (filter type = 'passcode')
  3. Duyệt tất cả passcodeConditions:
     - Split condition.configs.passcode bằng '*bss*' → listPasscode[]
     - Nếu case_insensitive: lowercase cả list và input
     - Kiểm tra listPasscode.includes(formattedPasscode)
  4. isPasscodeValid = true nếu bất kỳ condition nào match
  5. Lưu UsageHistoryV2:
     - status: 'Success' | 'Fail'
     - passcode, domain_id, page_url, customer_id, rule_id, time_use, session_id
```

---

## 5. Data Flow — Tạo Request (public/create-request)

```
POST /passcode/public/create-request
  Body: { domain, ruleId, email, name, page, message }
  @Public() — không cần session-token
  ThrottlerGuard: 1 request / 60s / IP (key từ x-forwarded-for hoặc cf-connecting-ip)
        │
        ▼
PasscodeService.createRequest()
  1. Tìm shop theo domain
  2. Tạo PasscodeRequests { domain_id, rule_id, email, name, page, message, status: 'pending' }
  3. Lưu DB
  4. Nếu shop.status_enable_receive_notification = true:
     - Check tránh gửi trùng:
       Tìm records cùng { domain_id, rule_id, email, page, status: 'pending' }
       Nếu count > 1 → skip (đã gửi rồi)
     - Gửi email qua EmailService.sendCustomerPasswordRequestEmail()
       Đến: shop.email_receive_passcode_notification ?? shop.email
```

---

## 6. Data Flow — Gửi Passcode cho Khách (send-email-to-customer)

```
POST /passcode/send-email-to-customer
  Body: { id: <requestId>, passcode: <string> }
  Guard: ShopGuard
        │
        ▼
PasscodeService.sendEmail()
  1. Tìm PasscodeRequest theo { domain_id, id }
  2. Gọi EmailService.createEmailPasscodeRequest(passcodeData, domain, passcode, shop)
  3. Cập nhật PasscodeRequest.status → 'approved'
```

---

## 7. Entities

### 7.1 PasscodeRequests (`passcodeRequestsV2`)

| Column | Type | Mô tả |
|--------|------|-------|
| `id` | int | PK |
| `rule_id` | int | Rule liên kết |
| `domain_id` | int | Shop ID |
| `email` | varchar | Email khách |
| `name` | text | Tên khách |
| `page` | text | URL trang khách đang xem |
| `message` | text | Nội dung tin nhắn của khách |
| `status` | enum | `pending` / `approved` / `rejected` / `invalid` |
| `createdAt` | timestamp | — |
| `updatedAt` | timestamp | — |

### 7.2 UsageHistoryV2 (`UsageHistoryV2`)

| Column | Type | Mô tả |
|--------|------|-------|
| `id` | int | PK |
| `passcode` | varchar(255) | Passcode khách đã nhập |
| `domain_id` | varchar(255) | Domain shop (string, không phải int) |
| `page_url` | varchar(500) | Trang khách nhập passcode |
| `customer_id` | varchar(255) | Shopify customer ID |
| `rule_id` | int | Rule liên kết |
| `status` | varchar(255) | `Success` / `Fail` |
| `time_use` | timestamp | Thời điểm nhập |
| `session_id` | varchar(255) | Session ID (dùng để group history) |
| `createdAt` | timestamp | — |
| `updatedAt` | timestamp | — |

> **Lưu ý**: `UsageHistoryV2.domain_id` là **string** (domain), khác với `PasscodeRequests.domain_id` là **int** (shop.id).

---

## 8. Quick Decision Tree

```
Merchant muốn kiểm soát passcode
        │
        ├─► Xem danh sách request từ khách?
        │   └─► POST /passcode/request (filter theo ngày)
        │
        ├─► Gửi passcode cho khách đã xin?
        │   └─► POST /passcode/send-email-to-customer { id, passcode }
        │       → cập nhật status → 'approved'
        │
        ├─► Xem lịch sử khách nhập passcode?
        │   └─► POST /passcode/get-all-usage-history
        │       (filter: date, status, ruleIds, session_id)
        │
        └─► Bật/tắt nhận notification email khi có request?
            └─► POST /passcode/change-status-requests { status }
                → cập nhật shop.status_passcode_requests

Khách muốn xin passcode
        └─► POST /passcode/public/create-request { domain, ruleId, email, name, page, message }
            → throttle 1 req/60s/IP
            → nếu shop bật notification → gửi email cho merchant

Storefront verify passcode sau khi khách nhập
        └─► POST /passcode/save-usage-history { domain, ruleId, passcode, pageUrl, ... }
            Header: authentication: Bearer <HMAC_SHA256>
            → So sánh với condition.configs.passcode (split '*bss*')
            → Ghi UsageHistoryV2 { status: 'Success' | 'Fail' }
```

---

## 9. Email Notifications

### 9.1 Khi khách gửi request (merchant nhận)
- Trigger: `createRequest()` — sau khi lưu `PasscodeRequests`
- Điều kiện: `shop.status_enable_receive_notification = true` và không có request pending trùng
- Service: `EmailService.sendCustomerPasswordRequestEmail()`
- Đến: `shop.email_receive_passcode_notification` (fallback: `shop.email`)

### 9.2 Khi merchant gửi passcode cho khách
- Trigger: `sendEmail()` — merchant action từ CMS
- Service: `EmailService.createEmailPasscodeRequest()`
- Sau khi gửi: cập nhật `PasscodeRequests.status → 'approved'`

---

## 10. Liquid Builder Pipeline — Theme Upload

Đây là flow chung của toàn bộ app khi merchant lưu/cập nhật rule.  
Service: `src/modules/upload/upload.service.ts`

### 10.1 Tổng quan pipeline

```
CMS → POST /rules → RuleService.saveOrUpdate() → DB
  └─► UploadService.generateAndUpload()
        │
        ▼
  getRulesContent(shop, configShop, customerAccountInfo)
    Loop qua tất cả rules active (status = 1):
    ├─► getRuleCondition(rule, shop)              → bssLockCondition        (cộng dồn)
    ├─► getLockContentPasscodeForm(rule, configShop) → bssLockContentPasscode (cộng dồn)
    └─► getPasscodeFormData(rule)                 → bssLockPasscodeFormData (cộng dồn)
                                                   → bssLockFormPasscodeError (cộng dồn)
        │
        ▼
  Inject vào Liquid template files:
  ├─► snippets/bss-lock-condition.liquid
  │     buildLockConditionPayload() → replace '{{ bss-lock-condition-rules }}' bằng bssLockCondition
  ├─► snippets/bss-lock-content.liquid
  │     .replace('{{ bss_lock_content_pc_rules }}', bssLockContentPasscode)
  ├─► snippets/bss-lock-passcode-form.liquid
  │     .replace('{{ passcode_form_data }}', bssLockPasscodeFormData)
  │     .replace('{{ error_passcode_form_data }}', bssLockFormPasscodeError)
  └─► snippets/bss-lock-passcode-form-price.liquid  (cùng placeholder như trên)
        │
        ▼
  ThemeService.putAsset() → Shopify Theme REST API
```

---

### 10.2 `getRuleCondition()` — Build condition cho `bssLockCondition`

Đây là hàm core sinh ra liquid condition cho **mỗi rule**, sau đó cộng dồn vào `bssLockCondition` để inject vào `bss-lock-condition.liquid`.

Với **passcode condition** (`condition.type === 'passcode'`), liquid được sinh ra như sau:

#### Bước 1 — Khai báo biến và danh sách condition

```liquid
{% assign temp_<conditionId> = false %}
{% assign passcode_condition_ids = passcode_condition_ids | append: '<conditionId>,' %}

{# Lưu toàn bộ passcode hợp lệ (đã lowercase nếu case_insensitive) #}
{% assign bss-lock-pc-<conditionId> = 'hello*bss*world*bss*test' %}
{% assign bss-lock-pc-escape-<conditionId> = bss-lock-pc-<conditionId> | escape %}
```

> `bss-lock-pc-escape-<conditionId>` là chuỗi chứa TẤT CẢ passcode hợp lệ, dùng delimiter `*bss*`, đã HTML-escape.

#### Bước 2 — Tên cart attribute (phụ thuộc `passcode_type`)

| `passcode_type` | `passAttrNameSuccess` | `passAttrNameCheck` |
|----------------|----------------------|---------------------|
| `0` ONE_TO_ALL | `bss-lock-pc-escape-<id>-success` | `bss-lock-pc-escape-<id>-check` |
| `1` ONCE_TO_EACH | `bss-lock-pc-escape-<id>-success-<scope>-<pageId>` | `bss-lock-pc-escape-<id>-check-<scope>-<pageId>` |
| `2` ONCE_FOR_ONCE | (giống type 1) | (giống type 1) |

Với type 1/2, suffix là:
- collection page → `c-<collection.id>`
- product page → `pr-<product.id>`
- page → `pg-<page.id>`
- blog → `b-<blog.id>`
- variant → `<variant_select_id>`

```liquid
{# Type 0 — global per session #}
{% assign passAttrNameSuccess-<id> = 'bss-lock-pc-escape-<id>-success' %}
{% assign passAttrNameCheck-<id>   = 'bss-lock-pc-escape-<id>-check' %}

{# Type 1/2 — per page #}
{% case current_scope %}
  {% when 'collection' %}{% assign extra = 'c-' | append: collection.id %}
  {% when 'product'    %}{% assign extra = 'pr-' | append: product.id %}
  ...
{% endcase %}
{% assign passAttrNameSuccess-<id> = 'bss-lock-pc-escape-<id>-success-' | append: extra %}
{% assign passAttrNameCheck-<id>   = 'bss-lock-pc-escape-<id>-check-'   | append: extra %}
```

#### Bước 3 — Đọc cart attribute và kiểm tra unlock

```liquid
{% assign cart_attributes_check_names   = cart_attributes_check_names   | append: passAttrNameCheck-<id>   | append: ',' %}
{% assign cart_attributes_success_names = cart_attributes_success_names | append: passAttrNameSuccess-<id> | append: ',' %}

{# Đọc giá trị đã lưu trong cart attributes #}
{% assign passAttrSuccess-<id> = cart.attributes[passAttrNameSuccess-<id>] | escape %}
{% assign passAttrCheck-<id>   = cart.attributes[passAttrNameCheck-<id>]   | escape %}

{# Wrap với *bss* để tránh partial match khi dùng contains #}
{% if passAttrSuccess-<id> %}
  {% assign passAttrSuccess-<id> = '*bss*' | append: passAttrSuccess-<id> | append: '*bss*' %}
{% endif %}

{# Kiểm tra: nếu passcode đã lưu nằm trong danh sách hợp lệ → mở khóa #}
{% if bss-lock-pc-escape-<id> contains passAttrSuccess-<id> %}
  {% assign temp_<id> = true %}
{% endif %}

{# Nếu chưa mở khóa → thêm conditionId vào danh sách cần hiện form #}
{% unless temp_<id> %}
  {% assign passcode_condition_id = passcode_condition_id | append: ',<conditionId>,' %}
{% endunless %}
```

#### Bước 4 — `requires_passcode` được set ở close tag của key

Sau khi check tất cả conditions trong key, nếu passcode condition chưa pass:

```liquid
{# passcodeCloseTag — generated ở getRuleCondition() #}
{% else %}
  {% assign current_lock_element_id = temp_lock_element_id %}
  {% assign requires_passcode = true %}
{% endif %}
```

---

### 10.3 `getLockContentPasscodeForm(rule, configShop)` → `bssLockContentPasscode`

Build liquid dạng **cộng dồn** — mỗi rule có passcode condition đóng góp 1 block.  
Toàn bộ được inject vào `{{ bss_lock_content_pc_rules }}` trong `bss-lock-content.liquid`.

**Khi nào được gọi**: `bss-lock-content.liquid` loop qua `lock_ids` → với mỗi `lock_id`, check xem rule đó cần hiện form gì.

```liquid
{% if lock_id == '<ruleId>' %}
  {# Guard: chỉ capture 1 lần, tránh duplicate khi render nhiều lần trong cùng page #}
  {% unless lock_content_passcode %}
  {% capture lock_content_passcode %}

    {# getMessagePasscode(rule, configShop) → assign các biến text/label theo translation #}
    {% capture passcode-message-<ruleId> %}...{% endcapture %}
    {% capture passcode-input-label-<ruleId> %}...{% endcapture %}
    {% capture message_passcode_incorrect-<ruleId> %}...{% endcapture %}
    {% capture button_text_form_submit-<ruleId> %}...{% endcapture %}
    {% capture input_placeholder_form_submit-<ruleId> %}...{% endcapture %}
    {% capture passcode_image_url-<ruleId> %}<imageUrl>{% endcapture %}

    {% capture form_style_type-<ruleId> %}0{% endcapture %}
    {% assign ruleId = "<ruleId>" %}

    {# Render form snippet với đầy đủ API URLs và config #}
    {% include 'bss-lock-passcode-form',
      rule_id:                     ruleId,
      origin_rule_id:              <ruleId>,
      passcode_message:            passcode-message-<ruleId>,
      message_passcode_incorrect:  message_passcode_incorrect-<ruleId>,
      button_text_form_submit:     button_text_form_submit-<ruleId>,
      input_placeholder_form_submit: input_placeholder_form_submit-<ruleId>,
      passcode_input_label:        passcode-input-label-<ruleId>,
      save_usage_history:   "https://<SERVER_URL>/passcode/save-usage-history",
      create_request_api:   "https://<SERVER_URL>/passcode/public/create-request",
      behavior_api:         "https://<SERVER_URL>/behavior",
      save_action_api:      "https://<SERVER_URL>/passcode/save-action-v2",
      is_call_api:          "false",
      is_pc_request:        "<true|false>",  {# từ design.passcode.show_pc_request #}
      form_style_type:      form_style_type-<ruleId>,
      passcode_case_sensitive: 1,
      bss_is_passcode_page: true,
      passcode_image_url:   passcode_image_url-<ruleId>,
      remove_cart_attr:     true
    %}

  {% endcapture %}
  {% endunless %}
  {% break %}   {# Dừng loop lock_ids khi đã match rule #}
{% endif %}
```

**Lưu ý quan trọng**:
- `{% unless lock_content_passcode %}` ngăn render trùng khi cùng rule xuất hiện nhiều block trên 1 page.
- `{% break %}` dừng vòng lặp loop qua lock_ids ngay sau khi match → không check tiếp các rule khác.
- `is_pc_request` → bật/tắt nút "Gửi yêu cầu xin passcode" trên form (link tới `create-request` API).

---

### 10.4 `getPasscodeFormData(rule)` → `bssLockPasscodeFormData` + `bssLockFormPasscodeError`

Hàm này build **2 chuỗi liquid** inject vào cả `bss-lock-passcode-form.liquid` và `bss-lock-passcode-form-price.liquid`.

#### `formDataString` (`{{ passcode_form_data }}`) — Submit cart attribute khi khách nhập passcode

Được chèn vào JS trong form snippet. Khi khách nhấn Submit:

```liquid
{% if rule_id == '<ruleId>' %}
  {# List tất cả condition IDs của rule này #}
  {% assign all_passcode_condition_id = '<id1>,<id2>' | split: ',' %}
  {# Chỉ những condition có case_sensitive = YES (1) #}
  {% assign list_passcode_case_sensitive_id = '<id1>' | split: ',' %}

  {# Loop từng condition, append cart attribute:
     - case_sensitive → giữ nguyên giá trị người dùng nhập
     - case_insensitive → toLowerCase() trước khi lưu #}
  {% for condition_id in all_passcode_condition_id %}
    {% assign current_index = passcode_condition_ids | find_index: condition_id %}

    {% if list_passcode_case_sensitive_id contains condition_id %}
      {# SUCCESS attribute — lưu khi passcode đúng #}
      formData.append('attributes[{{ cart_attributes_success_names[current_index] }}]', passcodeElement.value);
      {# CHECK attribute — lưu để detect lỗi lần sau #}
      formData.append('attributes[{{ cart_attributes_check_names[current_index] }}]', passcodeElement.value);
    {% else %}
      formData.append('attributes[{{ cart_attributes_success_names[current_index] }}]', passcodeElement.value.toLocaleLowerCase());
      formData.append('attributes[{{ cart_attributes_check_names[current_index] }}]', passcodeElement.value.toLocaleLowerCase());
    {% endif %}
  {% endfor %}
{% endif %}
```

> **Cơ chế unlock qua cart attribute**: Passcode được lưu vào Shopify cart attributes. Mỗi lần page load, `bss-lock-condition` đọc cart attributes để kiểm tra → không cần server roundtrip.

#### `formErrorString` (`{{ error_passcode_form_data }}`) — Hiển thị lỗi khi sai passcode

```liquid
{% if lock_id == '<ruleId>' %}
  {# passAttrCheck là giá trị khách vừa submit (lưu trong cart) #}
  {% if passAttrCheck-<conditionId> %}

    {# Wrap *bss* để đảm bảo exact match, tránh partial match #}
    {% assign passAttrCheck-<conditionId> = "*bss*" | append: passAttrCheck-<conditionId> | append: "*bss*" %}

    {# So sánh với escaped passcode list:
       - nếu KHÔNG chứa → passcode sai → hiện lỗi
       - nếu chứa → passcode đúng → không hiện lỗi (dù access_denied do condition khác) #}
    {% unless bss-lock-pc-escape-<conditionId> contains passAttrCheck-<conditionId> %}
      <div class="form-message form-message--error"
           style="display: flex; justify-content: flex-start; margin-top: 4px; color: red;">
        {% unless snippet-error-icon contains "Liquid error" %}
          <span style="width: 18px; margin-right: 6px;">{% render 'icon-error' %}</span>
        {% endunless %}
        {{ message_passcode_incorrect }}
      </div>
    {% endunless %}

  {% endif %}
{% endif %}
```

> **Tại sao cần `bssLockFormPasscodeError` riêng?** Vì `bss-lock-condition` chỉ check tại render time. Khi khách nhập sai và page reload, cart attribute `passAttrCheck` vẫn còn → cần đọc lại để hiện thông báo lỗi mà không cần thêm server request.

---

### 10.5 `bss-lock-condition.liquid` — Runtime Entry Point

File `src/liquid/bss-lock-condition.liquid` là snippet được render trong theme Shopify.  
Placeholder `{{ bss-lock-condition-rules }}` được thay bằng `bssLockCondition` (toàn bộ conditions đã build).

```
Storefront render page → {% render 'bss-lock-condition', scope, subject, variable %}
  │
  ▼
1. Init: is_authorized=false, access_denied=false, requires_passcode=false,
         passcode_condition_ids='', cart_attributes_success_names='', ...
2. Set scope/section_name dựa trên params truyền vào
3. {{ bss-lock-condition-rules }}  ← thực thi tất cả rule conditions đã baked-in
   Với mỗi rule:
   ├─ targetCondition  (product/collection/page/section/...)
   ├─ keyCondition     (customer tag, passcode, signed_in, ...)
   │   └─ Passcode: đọc cart.attributes → so với bss-lock-pc-escape → set temp_<id>
   └─ excludeCondition (allowed pages, products, ...)
   → Cộng dồn lock_ids, opened_lock_ids, requires_passcode, passcode_condition_id
4. lock_ids vs opened_lock_ids → access_denied = true/false
5. Nếu access_denied:
   ├─ bss_lock_action == 'section_replacement'
   │   → render 'bss-lock-content-element'
   │       → bss_lock_requires_passcode == true → render 'bss-lock-passcode-form'
   └─ bss_lock_action == 'hide-variant-option' → output 'true'
6. Return theo variable:
   'hide_section'     → {{ hide_section }}
   'hide_section_css' → <style>...CSS...</style> + JSON selectors
   'bss_lock_ids'     → lock_ids array JSON
```

---

### 10.6 Mapping đầy đủ: variable ↔ hàm sinh ra ↔ file inject

| Liquid variable | Sinh bởi | Inject vào file |
|----------------|----------|-----------------|
| `bss-lock-pc-<id>` | `getRuleCondition()` case `passcode` | `bss-lock-condition.liquid` qua `bssLockCondition` |
| `bss-lock-pc-escape-<id>` | `getRuleCondition()` | idem |
| `passAttrNameSuccess-<id>` | `getRuleCondition()` (tên phụ thuộc `passcode_type`) | idem |
| `passAttrNameCheck-<id>` | `getRuleCondition()` | idem |
| `cart_attributes_success_names` | `getRuleCondition()` — append mỗi condition | idem |
| `cart_attributes_check_names` | `getRuleCondition()` — append mỗi condition | idem |
| `passcode_condition_ids` | `getRuleCondition()` — append mỗi condition | idem |
| `passcode_condition_id` | `getRuleCondition()` — chỉ append khi chưa unlock | idem |
| `requires_passcode` | `getRuleCondition()` — passcode close tag | idem |
| `lock_content_passcode` | `getLockContentPasscodeForm()` | `bss-lock-content.liquid` qua `{{ bss_lock_content_pc_rules }}` |
| `{{ passcode_form_data }}` | `getPasscodeFormData().formDataString` | `bss-lock-passcode-form.liquid` + `*-price.liquid` |
| `{{ error_passcode_form_data }}` | `getPasscodeFormData().formErrorString` | `bss-lock-passcode-form.liquid` + `*-price.liquid` |

---

## 11. Security — Auth Header cho save-usage-history

```
Header: authentication: Bearer <hash>
hash = HMAC_SHA256(JSON.stringify({ domain, ruleId }), KEY_HASH_PASSCODE)

Guard: VerifyAndValidateStoreFrontGuard
  → Verify HMAC từ header
  → Attach shop vào request context
```

> Route `save-usage-history` KHÔNG dùng `@Public()` — phải có HMAC header hợp lệ.  
> Route `public/create-request` dùng `@Public()` — không cần auth, chỉ bị throttle.
