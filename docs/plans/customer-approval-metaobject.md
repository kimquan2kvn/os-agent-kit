# Customer Approval — Metaobject Plan

Luồng self-service: customer đã đăng nhập → submit request → merchant approve/reject → hệ thống tự cấp quyền qua Metaobject.

Khác với `customer-approval-required.md` (dùng Customer Tag), plan này dùng **Metaobject** làm storage — đồng nhất với cơ chế email approval nhưng định danh bằng `customer.id` thay vì `unique_id` HMAC.

---

## 1. Phân tích so sánh với email plan

| Hạng mục | Email plan (`approve-email-access-lock.md`) | Customer plan (file này) |
|---|---|---|
| Identifier | `unique_id = HMAC(email, salt)` — phải generate server-side | `customer.id` — sẵn có từ Shopify session |
| Persist client | localStorage + cart attribute | **Không cần** — `customer.id` luôn có qua Liquid |
| Re-verify flow | Phức tạp (check localStorage → restore cart attr) | **Không cần** — Liquid check trực tiếp |
| Endpoint generate-id | Cần | **Bỏ** |
| Storefront JS | generate-id → check-status → create-request | **create-request only** (status check do Liquid) |
| Security | HMAC với server-side salt | **StoreFrontGuard** — client gửi HMAC-SHA256(`{ domain, ruleId, shopify_customer_id }`) trong header |
| Storefront check | Cart attribute + Metaobject | **Chỉ Metaobject** (không cần cart attribute) |
| Metaobject field | `approved_ids` (HMAC strings) | `approved_customer_ids`, `pending_customer_ids`, `rejected_customer_ids`, `revoked_customer_ids` (Shopify numeric IDs) |

---

## 2. Condition type

| Field | Value |
|---|---|
| `value` | `access_request` |
| Label | `Approval required` |
| Group | User type |
| Yêu cầu | Customer phải đăng nhập Shopify account |

Liquid dùng `condition_type == 'access_request'` để render đúng block.

Nhóm User type sau khi thêm:

```
User type group
├── Everyone              (hiện có)
├── Signed in             (hiện có)
├── Customer tag          (hiện có)
├── Specific customers    (hiện có)
└── Approval required     (MỚI)
```

---

## 3. Kiến trúc lưu trữ

### Metaobject: `b2b_lock_customer_access`

1 entry per rule. Handle: `rule-{ruleId}`.

```
type (runtime):  app--{APP_ID}--b2b_lock_customer_access
handle:          rule-{ruleId}

fields:
  approved_customer_ids: list.single_line_text_field
  pending_customer_ids:  list.single_line_text_field
  rejected_customer_ids: list.single_line_text_field
  revoked_customer_ids:  list.single_line_text_field   ← có trong code, không chỉ 3 fields
```

Type thực tế lưu Shopify có prefix `app--{APP_ID}--` (Shopify namespacing). Code dùng:
```ts
const appType = `app--${this.appId}--${METAOBJECT_TYPE}`
// METAOBJECT_TYPE = 'b2b_lock_customer_access'
```

**Invariant:** Một `customer_id` xuất hiện trong **đúng 1 list** tại bất kỳ thời điểm nào.  
Mỗi lần rebuild sẽ ghi đầy đủ cả 4 list từ DB — không append/remove từng phần tử.

---

### Liquid check — dùng `contains` trên list field

```liquid
{% assign cid = customer.id | append: '' %}
{% assign rule_meta = shop.metaobjects['app--{APP_ID}--b2b_lock_customer_access']['rule-' | append: rule.id] %}
{% assign approved = rule_meta.approved_customer_ids.value %}
{% assign pending  = rule_meta.pending_customer_ids.value %}
{% assign rejected = rule_meta.rejected_customer_ids.value %}

{% if customer == blank %}
  {% render 'b2b-customer-approval-form', state: 'login_prompt', ... %}
{% elsif approved contains cid %}
  {%- comment -%} approved → render nội dung bình thường {%- endcomment -%}
{% elsif pending contains cid %}
  {% render 'b2b-customer-approval-form', state: 'pending', ... %}
{% elsif rejected contains cid %}
  {% render 'b2b-customer-approval-form', state: 'rejected', ... %}
{% else %}
  {% render 'b2b-customer-approval-form', state: 'form', ... %}
{% endif %}
```

Trong `bss-lock-content.liquid`: dùng variable `bss_lock_request_access` và check `condition_type == 'access_request'`.

---

### Backend operations — BullMQ queue + version key (không dùng Redis lock trực tiếp)

**Nguyên tắc:** DB là source of truth. Metaobject là projection được **rebuild toàn bộ** từ DB sau mỗi thay đổi.  
Dùng **BullMQ job deduplication** (cùng `jobId` → replace job cũ) + **version counter** trên Redis để tránh race condition.

```
// Sau mỗi operation (create/update/bulk-update):
aquireEnqueue(shopId, ruleId):
  versionKey = `shop:${shopId}:rule:${ruleId}:version`
  redis.incr(versionKey)                         // đánh dấu "có thay đổi mới"
  bullmq.add(CUSTOMER_APPROVAL_QUEUE, { ruleId, shop: shopId }, {
      jobId: `customer-approval-shop-${shopId}-rule-${ruleId}`,  // dedup: cùng shopId+ruleId → 1 job
      delay: 500ms,
      attempts: 3,
      backoff: exponential 1s,
  })

// CustomerAccessProcessor (concurrency: 50):
process(job):
  MAX_SYNC_ROUNDS = 5
  versionKey = `shop:${job.data.shop}:rule:${job.data.ruleId}:version`
  for round in 0..MAX_SYNC_ROUNDS:
      startVersion = redis.get(versionKey)
      processRequestData(shopId, ruleId)         // full rebuild Metaobject từ DB
      endVersion = redis.get(versionKey)
      if startVersion == endVersion:             // không có thay đổi mới trong lúc rebuild
          redis.delete(versionKey)
          return
  // nếu sau 5 vòng vẫn racing → job kết thúc (job mới sẽ xử lý tiếp)

// processRequestData(shopId, ruleId):
  SELECT shopify_customer_id, status FROM customerAccessRequests
         WHERE ruleId = ruleId AND shopId = shopId
  ensureMetaobjectDefinition()                   // tạo definition nếu chưa có
  handle = `rule-${ruleId}`
  fields = {
      approved_customer_ids: JSON.stringify([...approved IDs]),
      pending_customer_ids:  JSON.stringify([...pending IDs]),
      rejected_customer_ids: JSON.stringify([...rejected IDs]),
      revoked_customer_ids:  JSON.stringify([...revoked IDs]),
  }
  if metaobjectExists:
      metaobjectUpdate(id, fields)
  else:
      metaobjectCreate(type, handle, fields)
```

---

### Vòng đời dữ liệu

```
Action               approved_ids    pending_ids    rejected_ids   revoked_ids
──────────────────────────────────────────────────────────────────────────────
Ban đầu              []              []             []             []
create-request "111" []              ["111"]        []             []
approve "111"        ["111"]         []             []             []
create-request "222" ["111"]         ["222"]        []             []
reject "222"         ["111"]         []             ["222"]        []
revoke "111"         []              []             ["222"]        ["111"]
```

---

### Size limit

Metaobject ~128KB. Customer ID tối đa 13 ký tự → ~14 bytes/entry.  
4 list cộng lại → ~7.000 customers / rule. Nếu vượt → shard: handle `rule-{ruleId}-shard-2`.

**Tại sao không dùng Customer Tag?**
- Tag visible trên Shopify Admin → merchant có thể vô tình xóa
- Tag có thể conflict với tag-based conditions khác trong hệ thống
- Metaobject tách biệt, update độc lập, không bị merchant can thiệp ngoài CMS

---

## 4. Liquid check (storefront)

Liquid snippet trong section 3 là bản đầy đủ. Tóm tắt logic:

Không cần cart attribute, không cần localStorage, không cần JS restore.  
Toàn bộ state (pending / rejected / not_requested) được Liquid xử lý — **JS chỉ cần gọi `create-request` khi submit**.

`bss-lock-content.liquid` dùng biến `bss_lock_request_access` (capture từ snippet) và render khi `condition_type == 'access_request'`.

---

## 5. Flow hoàn chỉnh

### 5a. Guest (chưa login)

```
Liquid render → customer == blank
  → Hiện login prompt: "Vui lòng đăng nhập để yêu cầu truy cập"
  → Nút "Log in" → redirect /account/login?return_url=<current_page>
```

### 5b. Lần đầu — Customer đã login, chưa request

```
[Liquid render — server-side]
  → rule_meta = shop.metaobjects['app--{APP_ID}--b2b_lock_customer_access']['rule-{rule.id}']
  → pending/approved/rejected đều không contains customer.id
  → Compute bearer_token = HMAC-SHA256(
        JSON.stringify({ domain: shop.permanent_domain, ruleId: rule.id, shopify_customer_id: customer.id }),
        KEY_HASH_PASSCODE   ← secret chỉ tồn tại server-side, không ra client
    )
  → Render b2b-customer-approval-form với:
       data-domain="{{ shop.permanent_domain }}"
       data-rule-id="{{ rule.id }}"
       data-customer-id="{{ customer.id }}"
       data-token="{{ bearer_token }}"

[Customer submit — JS chỉ đọc data attribute, không compute gì]
  → POST /customer-approval/public/create-request
       Header: authentication: Bearer {{ bearer_token }}
       Body: { domain, ruleId, shopify_customer_id, email, name, message? }
  ← { message: 'Created successfully' }
  → Server: StoreFrontGuard verify hash → INSERT DB (hoặc UPDATE nếu đã tồn tại, reset về pending)
           → gửi email notify MERCHANT (sendCustomerApprovalNotifyMerchant)
           → aquireEnqueue() → BullMQ job → rebuild Metaobject
  → JS: hiện inline pending message (không reload)
```

> **Lưu ý:** Nếu customer đã có record (trước đó bị rejected rồi submit lại), `createRequest` sẽ UPDATE status về `PENDING` thay vì INSERT mới.

### 5c. Customer đã request, đang pending

```
Liquid render
  → pending_customer_ids contains customer.id
  → Render pending message (không cần JS)
```

### 5d. Customer bị rejected

```
Liquid render
  → rejected_customer_ids contains customer.id
  → Render rejected message
```

### 5e. Merchant approve/reject/revoke trên CMS

```
[CMS]
  Merchant vào tab "Customer Access Requests"
  → Xem danh sách (tên, email, rule, ngày gửi, status)
  → Nhấn Approve / Reject / Revoke

[login-api-v2]
  POST /customer-approval/update-request { id: requestId, status: 'approved' | 'rejected' | 'revoked' }
  → UPDATE DB: status = input.status
  → Nếu status == APPROVED → gửi email cho CUSTOMER (sendCustomerApprovalTemplate)
  → Nếu status == REJECTED hoặc REVOKED → KHÔNG gửi email
  → aquireEnqueue() → BullMQ → rebuild Metaobject
```

### 5e-2. Bulk approve/reject

```
[CMS] Merchant chọn nhiều records → Bulk Approve / Bulk Reject

[login-api-v2]
  POST /customer-approval/bulk-update-requests { ids: [id1, id2, ...], status: 'approved' | 'rejected' }
  → UPDATE DB: status = input.status WHERE id IN ids AND shopId = shop.id
  → Nếu status == APPROVED → gửi email cho từng CUSTOMER (Promise.allSettled, fire-and-forget)
  → Nếu status != APPROVED → KHÔNG gửi email
  → aquireEnqueue() cho từng ruleId duy nhất trong batch
```

### 5f. Customer reload sau khi được approve

```
Liquid render
  → approved_customer_ids contains customer.id → TRUE
  → Hiện nội dung ngay, không cần JS
```

---

## 6. Sơ đồ tổng hợp

```
Page load (Liquid)
  rule_meta = shop.metaobjects['app--{APP_ID}--b2b_lock_customer_access']['rule-{rule.id}']
  cid = customer.id (string)
    │
    ├─ customer == blank
    │       └──▶ state: 'login_prompt'
    │
    ├─ approved_customer_ids contains cid
    │       └──▶ Hiện nội dung bình thường
    │
    ├─ pending_customer_ids contains cid
    │       └──▶ state: 'pending'
    │
    ├─ rejected_customer_ids contains cid
    │       └──▶ state: 'rejected'
    │
    └─ không có trong list nào (not_requested)
            └──▶ Liquid compute bearer_token = HMAC({ domain, ruleId, customer_id }) server-side
                 Render b2b-customer-approval-form (state: 'form')
                   data-token=<bearer_token>  data-domain data-rule-id data-customer-id
                        │
                        └─ [Submit] JS đọc data attribute → POST create-request
                                       Header: authentication: Bearer <token>
                                       Body: { domain, ruleId, shopify_customer_id, email, name, message? }
                                    → StoreFrontGuard verify hash
                                    → Server: INSERT/UPDATE DB → notify merchant email
                                    → BullMQ job → rebuild Metaobject
                                    → inline: chuyển sang state 'pending'
```

---

## 7. Security

| Rủi ro | Xử lý |
|---|---|
| Customer giả mạo customer_id | `StoreFrontGuard` verify HMAC-SHA256(`{ domain, ruleId, shopify_customer_id }`) — client không thể tự tính vì không có `KEY_HASH_PASSCODE` |
| Brute-force create-request | Throttle: **1 req/60s per IP** (`@Throttle limit:1 ttl:60000`) |
| Customer chưa login bypass | Liquid check `customer == blank` trước → không render form → JS không chạy |
| Metaobject bị sửa ngoài CMS | Shopify App scope kiểm soát write access |
| Metaobject size limit | Shard khi > ~7.000 IDs / rule (4 lists × 128KB) |
| Race condition khi concurrent requests | BullMQ jobId dedup + version counter: cùng shopId+ruleId → chỉ 1 job active, processor retry đến khi version stable |

**Cơ chế StoreFront Guard:**
- Guard sử dụng `VerifyAndValidateStoreFrontGuard` — tương tự flow passcode hiện có
- Client compute hash = `HMAC-SHA256(JSON.stringify({ domain, ruleId, shopify_customer_id }), KEY_HASH_PASSCODE)`
- Hash được gửi qua header `authentication: Bearer <hash>`
- Server recompute hash từ body và so sánh — nếu khớp mới chấp nhận request
- `KEY_HASH_PASSCODE` là server-side secret, inject vào Liquid khi upload theme dưới dạng hash đã compute sẵn (không expose raw key)

---

## 8. API Endpoints

| Endpoint | Method | Guard | Mô tả |
|---|---|---|---|
| `/customer-approval/public/create-request` | POST | StoreFrontGuard + Throttle (1/60s/IP) | Tạo/reset request về pending + rebuild Metaobject |
| `/customer-approval/` | GET | ShopGuard | Merchant lấy danh sách requests (filter by status/rule/search/date/page) |
| `/customer-approval/update-request` | POST | ShopGuard | Update status 1 record (approve/reject/revoke) → email customer nếu approved |
| `/customer-approval/bulk-update-requests` | POST | ShopGuard | Bulk update status nhiều records → email customers nếu approved |

> **Không có** các endpoint `/approve`, `/reject`, `/revoke`, `/delete-request` riêng biệt như trong spec gốc. Tất cả thay đổi status đi qua `update-request` (single) hoặc `bulk-update-requests` (batch).

### Request/Response chi tiết

**POST `/customer-approval/public/create-request`**
```
Header: authentication: Bearer <HMAC-SHA256(JSON.stringify({ domain, ruleId, shopify_customer_id }))>
Body: {
  domain:               string   // shop.permanent_domain
  ruleId:               number   // rule.id
  shopify_customer_id:  string   // customer.id
  email:                string   // customer.email (từ form readonly field)
  name:                 string   // customer nhập
  message?:             string   // optional, từ textarea
}
Response: { message: 'Created successfully' } | { message: 'Updated successfully' }
```

**GET `/customer-approval/`**
```
Query params: startDate?, endDate?, name?, email?, shopify_customer_id?, ruleId?, page?, status?
Response: { items, total, page, limit, totalPages }
```

**POST `/customer-approval/update-request`**
```
Body: { id: number, status: 'approved' | 'rejected' | 'revoked' }
Response: { message: 'Status updated successfully', affected: number }
```

**POST `/customer-approval/bulk-update-requests`**
```
Body: { ids: number[], status: 'approved' | 'rejected' | 'revoked' }
Response: { message: 'Bulk status updated successfully', affected: number }
```

---

## 9. Entity: CustomerAccessRequests

```typescript
@Entity('customer-access-requests')
@Index('customer_rule', ['shopify_customer_id', 'ruleId'], { unique: true })
class CustomerAccessRequests {
  id: number
  domain_id: number   // via shop relation
  ruleId: number
  shopify_customer_id: string   // Shopify numeric customer ID (string để tránh bigint issue)
  email: string                 // từ form submission
  name: string                  // từ form submission
  status: 'pending' | 'approved' | 'rejected' | 'revoked'
  message: string | null        // optional note từ customer
  createdAt: Date
  updatedAt: Date
  shop: Shop                    // ManyToOne, eager: false
  rule: Rules                   // ManyToOne, eager: true
}
```

Index unique `(shopify_customer_id, ruleId)` — mỗi customer chỉ có 1 record per rule, `createRequest` sẽ UPDATE nếu đã tồn tại.

**Migration:** `1781079884422-feat-LOGIN-396.ts` → thêm column `messages` (JSON) vào `rules-v2`.  
**Migration:** `1781086762232-feat-LOGIN-385.ts` → thêm column `request_access` (JSON) vào `designs-v2`.

---

## 10. Config lưu trữ

### Messages (trong Rules entity)

Config message lưu trong `rules-v2.messages` (JSON field), **không phải** trong `conditions` JSON:

```typescript
// rules-v2.messages
{
  request_access: {
    request_message: string,  // default: "Request access to this content."
    pending_message: string,  // default: "Your request has been submitted. We'll notify you once approved."
    rejected_message: string, // default: "Your request was not approved. Contact us for help."
    revoked_message: string,  // thêm so với spec gốc
    login_prompt: string,     // thêm so với spec gốc
  }
}
```

Type: `LockMessages.request_access: RequestAccessMessages`

### Design config (trong designs-v2)

Notification toggle + email config lưu trong `designs-v2.request_access` (JSON field).

---

## 11. CMS — Condition picker & Display Settings (Rule wizard)

**Source MR:** !2264 (feat/LOGIN-385, merged 2026-06-16)

### 11.1. Condition picker

File: `web/client/constants/locks/access-message.const.js`

Condition `access_request` được thêm vào group "User type":

```
┌─────────────────────────────────────────────────────────────┐
│  User type  (icon: PersonLockFilledIcon)                     │
│─────────────────────────────────────────────────────────────│
│  ○  Everyone                                                │
│  ○  Signed in                                               │
│  ○  Customer tag                                            │
│  ○  Specific customers                                      │
│  ○  Approval required  (icon: OrganizationIcon)  [Advanced] │
│     Request access – visitor request, you approve           │
└─────────────────────────────────────────────────────────────┘
```

Config trong `FIELD_CONFIG(t)`:
```js
[CONDITION_TYPE.ACCESS_REQUEST]: {
    label: t('requestAccess'),        // "Approval required"
    value: 'access_request',
    icon: OrganizationIcon,
    plan: 'Advanced',
    isAllow: true
}
```

i18n keys:
- Label trong condition picker: `locks.conditionSetting.requestAccess` → "Approval required"
- Description trong rules table: `locks.conditionSetting.keyDescription.requestAccess` → "Request access – visitor request, you approve"
- Tab label trong Display Settings: `locks.edit.modalDisplaySetting.tabs.accessRequest.title` → "Request access"

`access_request` bị loại khỏi danh sách điều kiện hỗ trợ invert (EXCLUDE_INVERT_CONDITIONS) — không có option "NOT approval required".

---

### 11.2. Display Settings modal — tab "Request access"

Merchant vào Rule wizard → chọn condition "Approval required" → nhấn "Customize display" → mở `ModalDisplaySettingsNew`.

Tab "Request access" được thêm vào modal với 2 variant tùy theo target type:

| Target type | Tab ID | Component |
|---|---|---|
| Element lock (PRICE, CART_BTN, PRICE_CART_BTN) | `access-request-element` | `ContentAccessRequestPrice` |
| Page lock (PAGE, PRODUCT, COLLECTION, v.v.) | `access-request-page` | `ContentAccessRequestPage` |

File: `web/client/components/ui/common/ModalDisplaySettingsNew.jsx`

Cả hai variant dùng chung layout `DesignLayout` (scrollable settings bên trái + live preview bên phải).

---

### 11.3. Scrollable settings panel — `ScrollableRequestAccessPrice`

File: `web/client/components/ui/DisplaySettings/RequestAccessPrice/ScrollableRequestAccessPrice.jsx`

Dùng chung cho cả element lock và page lock. Gồm 2 section collapsible (dùng `ScrollableLayout`):

**Section 1: Messages** (mở sẵn theo default)

```
┌─────────────────────────────────────────────────────────────────────┐
│  Messages                                               [expanded]  │
│─────────────────────────────────────────────────────────────────────│
│  Request message                                                    │
│  [ Request access to this content.                             ]    │
│  helptext: Message displayed to users who have not yet              │
│            requested access                                         │
│  Text color: [color picker] #000000ff                               │
│                                                                     │
│  ─────────────────────────────────────────────────────────          │
│                                                                     │
│  Pending message                                                    │
│  [ Your request has been submitted. We'll notify you once      ]    │
│  [ approved.                                                   ]    │
│  helptext: Message displayed to users who have requested access     │
│            but are still pending approval                           │
│  Text color: [color picker] #000000ff                               │
│                                                                     │
│  ─────────────────────────────────────────────────────────          │
│                                                                     │
│  Rejected message                                                   │
│  [ Your request was not approved. Contact us for help.         ]    │
│  helptext: Message displayed to users whose access request has      │
│            been rejected                                            │
│  Text color: [color picker] #000000ff                               │
│                                                                     │
│  ─────────────────────────────────────────────────────────          │
│                                                                     │
│  Revoked message                                                    │
│  [ Your access has been revoked. Contact us for help.          ]    │
│  helptext: Message displayed to users whose access request has      │
│            been revoked                                             │
│  Text color: [color picker] #000000ff                               │
│                                                                     │
│  ─────────────────────────────────────────────────────────          │
│                                                                     │
│  Login prompt                                                       │
│  [ Please log in to request access to this content.            ]    │
│  helptext: Message displayed to users who are not logged in         │
│  Text color: [color picker] #000000ff                               │
└─────────────────────────────────────────────────────────────────────┘

**Section 2: Custom CSS** (collapsed theo default)

┌─────────────────────────────────────────────────────────────────────┐
│  Custom CSS                                             [collapsed] │
│─────────────────────────────────────────────────────────────────────│
│  [ (textarea, multiline=4, resizable)                          ]    │
└─────────────────────────────────────────────────────────────────────┘
```

Data bindings trong `ContentSettings`:
- Mỗi message field: `TextField` bind với `lockContent.messages.request_access[key]`
- Mỗi color picker: `CustomColorPicker` bind với `design.request_access[key_color]`
- Thay đổi message gọi `handleChangeLockContent(newLockContent)` (deep clone trước khi mutate)
- Thay đổi màu gọi `handleChangeDesignSettings(DisplaySettingType.REQUEST_ACCESS, { [type]: hex8Color })`
- Color format: hex8 (8 ký tự, bao gồm alpha), ví dụ `#000000ff`
- Default color cho tất cả 5 fields: `#000000ff`

---

### 11.4. Live preview panel

**Element lock:** `PreviewRequestAccessPrice`  
File: `web/client/components/ui/DisplaySettings/RequestAccessPrice/PreviewRequestAccessPrice.jsx`  
Layout: `PreviewPriceLayout` (price hidden, show AddToCart button)

**Page lock:** `PreviewRequestAccessPage`  
File: `web/client/components/ui/DisplaySettings/RequestAccessPage/PreviewRequestAccessPage.jsx`  
Layout: `PreviewPageLayout` (showLockImg=true)

Preview có mode switcher (6 buttons) khi mở từ Display Settings modal (`isPreviewInRule=false`):

```
[ Request message ] [ Pending message ] [ Rejected message ]
[ Revoked access  ] [ Login prompt    ] [ Modal request    ]
```

Khi `isPreviewInRule=true` (xem preview trong rule wizard step, không phải modal) — mode switcher bị ẩn, chỉ hiện `request_message` mặc định.

**Modal request preview:** Khi chọn mode "Modal request", render overlay modal với form đầy đủ:
- Lock icon (SVG)
- Title: "Request access to this content."
- Subtitle: "Fill in the form below and we'll review your request."
- Field Email (readonly, value: "john.doe@example.com", hint: "Sending as your account email.")
- Field Full name (required, placeholder: "Your full name")
- Field Message (optional, textarea, maxLength=500, hint: "0 / 500")
- Button "Send request"

CSS cho màu sắc được sinh inline từ `design.request_access` qua `hex8ToRgba()`:
```
.request_message { color: rgba(...) }
.pending_message { color: rgba(...) }
.rejected_message { color: rgba(...) }
.revoked_message { color: rgba(...) }
.login_prompt { color: rgba(...) }
{custom_css}
```

---

### 11.5. Preview trong Rule wizard (ContentOnLockPage)

File: `web/client/components/ui/ContentOnLockPage/index.jsx`

Khi condition `access_request` được chọn, tab preview xuất hiện trong step Content của rule wizard:

| Target type | Tab label | Component | Tab value |
|---|---|---|---|
| Element lock | "Request Access" | `RequestAccessPrice` | `'request-access'` |
| Page lock | "Request Access" | `RequestAccessPage` | `'request-access-page'` |

File components:
- `web/client/components/ui/ContentOnLockPage/RequestAccessPrice/RequestAccessPrice.jsx`
- `web/client/components/ui/ContentOnLockPage/RequestAccessPage/RequestAccessPage.jsx`

Cả hai chỉ là wrapper — truyền `ruleData.design` vào preview component tương ứng với `isPreviewInRule=true`.

---

### 11.6. State management & default values

File: `web/client/pages/locks/[id].jsx`

Khi load/update rule, hàm `updateRuleStates()` gọi `fillMissingFields(Rule, defaultRule)` trước khi xử lý — đảm bảo `messages.request_access` và `design.request_access` luôn có giá trị mặc định đầy đủ dù rule cũ chưa có fields này.

File helper: `web/client/helper/rule.helper.js` — export `fillMissingFields(ruleData, defaultRule)`: deep merge, không mutate input, chỉ fill các key bị thiếu hoặc undefined, không override giá trị đã có.

Field `messages` được thêm vào danh sách dirty-check fields (cùng với các fields khác như `show_message_hide_section`).

Default values từ `defaultRule` (file `web/client/constants/rule.constant.js`):

```js
// messages
messages: {
    request_access: {
        request_message: "Request access to this content.",
        pending_message: "Your request has been submitted. We'll notify you once approved.",
        rejected_message: "Your request was not approved. Contact us for help.",
        revoked_message: "Your access has been revoked. Contact us for help.",
        login_prompt: "Please log in to request access to this content.",
    }
}

// design
design: {
    request_access: {
        request_message_color: "#000000ff",
        pending_message_color: "#000000ff",
        rejected_message_color: "#000000ff",
        revoked_message_color: "#000000ff",
        login_prompt_color: "#000000ff",
        custom_css: ""
    }
}
```

`DisplaySettingType.REQUEST_ACCESS = 'request_access'` (file `web/client/utils/displaySettings.js`)

---

### 11.7. Summary panel (sidebar phải)

```
RULE TO ACCESS
[OrganizationIcon]  Approval required
```

---

### 11.8. Ghi chú — Những gì KHÔNG có trong MR này

MR !2264 chỉ implement phần CMS Rule wizard (condition picker + display settings). Các phần sau KHÔNG có trong MR này:

- Notification toggle (email cho merchant) trong config panel — docs cũ ghi sai, phần này chưa implement trong MR này
- API call riêng khi save display settings — settings được save cùng rule data qua flow save rule thông thường
- Validation riêng cho message fields — không có giới hạn ký tự hay required validation

---

## 12. CMS — Customer Access Requests page

```
┌──────────────────────────────────────────────────────────────────────┐
│  Customer Access Requests                                            │
│──────────────────────────────────────────────────────────────────────│
│  [All rules ▼]  [All status ▼]  [🔍 Search by email or name       ] │
│──────────────────────────────────────────────────────────────────────│
│  ☐  Customer            Rule name       Requested     Status         │
│──────────────────────────────────────────────────────────────────────│
│  ☐  john@gmail.com      View prices     2 hours ago   [Approve][Reject] │
│  ☐  mary@company.com    View prices     1 day ago     ✓ Approved [Revoke] │
│  ☐  bob@gmail.com       View catalog    3 days ago    ✗ Rejected     │
│──────────────────────────────────────────────────────────────────────│
│  [Bulk approve]  [Bulk reject]              1–100 of 48   ◀  ▶       │
└──────────────────────────────────────────────────────────────────────┘
```

- Page size: **100 records** (cố định, không configurable từ UI)
- Filter: status, ruleId, email, name, shopify_customer_id, startDate, endDate, page
- Approved record: có thêm action **Revoke** (update status → `revoked`, xóa khỏi approved_customer_ids Metaobject)

---

## 13. Storefront — Liquid/JS components

| File | Mô tả |
|---|---|
| `b2b-customer-approval-login-prompt.liquid` | Hiện khi `customer == blank` |
| `b2b-customer-approval-form.liquid` | Hiện khi not_requested — chứa nút submit, JS gắn handler vào |
| `b2b-customer-approval-pending.liquid` | Hiện khi `pending_customer_ids contains customer.id` |
| `b2b-customer-approval-rejected.liquid` | Hiện khi `rejected_customer_ids contains customer.id` |
| `b2b-customer-access.js` | **Chỉ** xử lý submit create-request — không có check-status on load |

`bss-lock-content.liquid` dùng variable `bss_lock_request_access` (capture snippet) và render khi `condition_type == 'access_request'`.

### b2b-customer-access.js logic

```
Page load
  → Không gọi API (Liquid đã render đúng state)
  → Nếu form present → gắn submit handler

Submit handler
  → Đọc từ data attribute của form:
       domain              = form.dataset.domain
       ruleId              = form.dataset.ruleId
       shopify_customer_id = form.dataset.customerId
       token               = form.dataset.token   ← bearer token đã được Liquid compute server-side
  → POST /customer-approval/public/create-request
       Header: authentication: Bearer <token>
       Body: { domain, ruleId, shopify_customer_id, email, name, message? }
  → success → thay form bằng pending message (inline, không reload)
```

**JS không compute hash** — Liquid đã tính sẵn token server-side, JS chỉ đọc và gửi đi.  
**Không có:** check-status call, localStorage, cart attribute update, generate-id call, App Proxy query params.

---

## 14. Upload pipeline integration

**Trạng thái: Implemented trong MR !531 (LOGIN-396 / LOGIN-385)**

Không có snippet Liquid riêng cho feature này. Thay vào đó, `upload.service.ts` sinh toàn bộ Liquid inline (template string) và nhúng thẳng vào `bss-lock-content.liquid`.

---

### 14.1. Call flow trong `upload.service.ts`

```
generateContent(rules, shop, ...) — hàm tổng hợp nội dung
  └─ loop qua từng rule
       └─ getContentCustomerApproval(rule)  ← method mới, trả về Liquid string
            → tích lũy vào biến bssLockRequestAccessContent

uploadContent() / uploadContentNew()
  └─ bssLockContentLiquid.replace(
       '{{ bss_lock_content_request_access }}',
       bssLockRequestAccessContent
     )
  → upload kết quả lên 'snippets/bss-lock-content.liquid'

generateAndUploadDesign() / generateDesignNew()
  └─ getStyleRequestAccess(design.request_access, ruleId)
       → sinh CSS cho các màu (login_prompt, pending, rejected, revoked, request)
  → tích lũy vào styleDesign
  → upload lên 'assets/bss-lock-settings.css'
```

---

### 14.2. Method `getContentCustomerApproval(rule: Rules)`

Điều kiện kích hoạt: rule phải có ít nhất 1 condition có `type === CONDITION_TYPE.ACCESS_REQUEST` (`'access_request'`) trong `rule.keys[].conditions`.

Nếu không có condition → trả về chuỗi rỗng (không inject gì).
Nếu không có `rule.messages?.request_access` → trả về chuỗi rỗng.

Liquid được sinh ra có cấu trúc như sau (per rule):

```liquid
{% if lock_id == '{rule.id}' %}
  {% unless bss_lock_request_access %}
    {% capture bss_lock_request_access %}
      {% assign b2b_cid_{rule.id} = customer.id | append: '' %}
      {% assign b2b_rule_meta_{rule.id} = shop.metaobjects.app--{APP_ID}--b2b_lock_customer_access['rule-{rule.id}'] %}
      {% assign b2b_pending_{rule.id}  = b2b_rule_meta_{rule.id}.pending_customer_ids.value %}
      {% assign b2b_rejected_{rule.id} = b2b_rule_meta_{rule.id}.rejected_customer_ids.value %}
      {% assign b2b_revoked_{rule.id}  = b2b_rule_meta_{rule.id}.revoked_customer_ids.value %}
      {% assign b2b_approved_{rule.id} = b2b_rule_meta_{rule.id}.approved_customer_ids.value %}

      {% unless b2b_approved_{rule.id} contains b2b_cid_{rule.id} %}
        {% if customer == blank %}
          <p class="bss-lock-page-login-prompt-{rule.id}" style="text-align:center;">
            {loginPrompt}
          </p>
        {% elsif b2b_pending_{rule.id} contains b2b_cid_{rule.id} %}
          <p class="bss-lock-page-pending-access-{rule.id}" style="text-align:center;">{pendingMessage}</p>
        {% elsif b2b_rejected_{rule.id} contains b2b_cid_{rule.id} %}
          <p class="bss-lock-page-rejected-access-{rule.id}" style="text-align:center;">{rejectedMessage}</p>
        {% elsif b2b_revoked_{rule.id} contains b2b_cid_{rule.id} %}
          <p class="bss-lock-page-revoked-access-{rule.id}" style="text-align:center;">{revokedMessage}</p>
        {% else %}
          {%- capture b2b_ca_payload_{rule.id} -%}
            {"domain":"{{ shop.permanent_domain }}","ruleId":{rule.id},"shopify_customer_id":"{{ customer.id }}"}
          {%- endcapture -%}
          {%- assign b2b_ca_token_{rule.id} = b2b_ca_payload_{rule.id} | hmac_sha256: '{KEY_HASH_PASSCODE}' -%}
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
        {% endif %}
      {% endunless %}
    {% endcapture %}
  {% endunless %}
  {% break %}
{% endif %}
```

**Lưu ý quan trọng:**
- `APP_ID` được inject qua `this.appId` (inject vào service từ env `APP_ID`).
- Metaobject type: `app--{APP_ID}--b2b_lock_customer_access` — tên constant `METAOBJECT_B2B_LOCK_CUSTOMER_ACCESS = 'b2b_lock_customer_access'`.
- `KEY_HASH_PASSCODE` là constant hiện có, dùng chung với passcode.
- `data-request-access` chứa JSON `{ messages: requestAccessMessages, ruleId: rule.id }` (single-quoted, HTML-entity-escaped).
- Messages mặc định nếu `rule.messages.request_access` thiếu field: `login_prompt = 'You need to login to see the content'`, `request_message = 'Request access to see content'`, `pending_message = 'Your request is being reviewed'`, `rejected_message = 'Your request has been rejected'`, `revoked_message = 'Your request has been revoked'`.

---

### 14.3. Condition check trong condition liquid (per key)

Trong vòng lặp `switch(condition.type)` tại `generateContent()`:

```liquid
{% assign b2b_cid_{condition.id} = customer.id | append: '' %}
{% assign b2b_rule_meta_{condition.id} = shop.metaobjects.app--{APP_ID}--b2b_lock_customer_access['rule-{rule.id}'] %}
{% assign b2b_approved_{condition.id} = b2b_rule_meta_{condition.id}.approved_customer_ids.value %}
{% assign b2b_pending_{condition.id}  = b2b_rule_meta_{condition.id}.pending_customer_ids.value %}
{% assign b2b_rejected_{condition.id} = b2b_rule_meta_{condition.id}.rejected_customer_ids.value %}
{% assign b2b_revoked_{condition.id}  = b2b_rule_meta_{condition.id}.revoked_customer_ids.value %}
{% assign temp_{condition.id} = false %}
{% if b2b_approved_{condition.id} contains b2b_cid_{condition.id} %}
  {% assign temp_{condition.id} = true %}
{% endif %}
```

closeTag của condition block:
```liquid
{% else %} {% assign current_lock_element_id = temp_lock_element_id %} {% assign condition_type = 'access_request' %} {% endif %}
```

---

### 14.4. Thay đổi trong `bss-lock-content.liquid`

Hai vị trí được sửa:

**1. Capture section (trước capture countdown):** Thêm placeholder để inject `bssLockRequestAccessContent`:
```liquid
{% unless bss_lock_request_access %}
  {% for lock_id in bss_lock_ids %}
    {% unless bss_opened_lock_ids contains lock_id %}
      {{ bss_lock_content_request_access }}
    {% endunless %}
  {% endfor %}
{% endunless %}
```
Placeholder `{{ bss_lock_content_request_access }}` được replace bằng output của `getContentCustomerApproval()` cho tất cả rules.

**2. Render section (trong branch customer đã đăng nhập):** Thêm `{% elsif condition_type == 'access_request' %}` vào chuỗi if/elsif:

```liquid
{% if bss_lock_requires_age %}
  {{ bss_lock_age_form }}
{% elsif bss_lock_requires_sntap %}
  {{ bss_lock_sntap_form }}
  {{ bss_lock_support_contact }}
{% elsif condition_type == 'start_date' %}
  {{ bss_lock_content_countdown }}
{% elsif condition_type == 'access_request' %}     ← THÊM MỚI
  {{ bss_lock_request_access }}
{% else %}
  {{ lock_content_logged }}
{% endif %}
```

Tương tự được thêm trong branch customer chưa đăng nhập (phía dưới `{% elsif bss_lock_requires_sntap %}`):
```liquid
{% elsif condition_type == 'access_request' %}
  {{ bss_lock_request_access }}
```

---

### 14.5. CSS generation — `getStyleRequestAccess()`

Hàm mới trong `design.const.ts`:

```typescript
getStyleRequestAccess(settings: RequestAccessSettings | null, ruleId: number): string
```

Sinh CSS cho 5 màu customizable:
| Setting field | CSS class |
|---|---|
| `login_prompt_color` | `.bss-lock-price-login-prompt-{id}`, `.bss-lock-page-login-prompt-{id}` |
| `pending_message_color` | `.bss-lock-price-pending-access-{id}`, `.bss-lock-page-pending-access-{id}` |
| `rejected_message_color` | `.bss-lock-price-rejected-access-{id}`, `.bss-lock-page-rejected-access-{id}` |
| `revoked_message_color` | `.bss-lock-price-revoked-access-{id}`, `.bss-lock-page-revoked-access-{id}` |
| `request_message_color` | `.bss-lock-price-request-access-{id}`, `.bss-lock-page-request-access-{id}` |

Luôn thêm hover style cho request message: `text-decoration: underline; cursor: pointer`.

`custom_css` từ `design.request_access.custom_css` được nối vào `customCSS` string (không qua `getStyleRequestAccess()`).

Output được tích lũy vào `styleDesign` và upload lên `assets/bss-lock-settings.css`.

---

### 14.6. Entity changes (database)

**`rules-v2` table — field `messages` (JSON, nullable):**
```typescript
// Rule entity
messages: LockMessages | null;

// Type
interface LockMessages {
  request_access: RequestAccessMessages;
}
interface RequestAccessMessages {
  request_message: string;
  pending_message: string;
  rejected_message: string;
  revoked_message: string;
  login_prompt: string;
}
```
Migration: `migrations/1781086762232-feat-LOGIN-385.ts`

**`designs-v2` table — field `request_access` (JSON, nullable):**
```typescript
// DesignV2 entity
request_access: RequestAccessSettings | null;

// Type
type RequestAccessSettings = {
  request_message_color: string;
  pending_message_color: string;
  rejected_message_color: string;
  revoked_message_color: string;
  login_prompt_color: string;
  custom_css: string;
}
```
Migration: `migrations/1781079884422-feat-LOGIN-396.ts`

---

### 14.7. Price/ATC element inline Liquid

Với các target type `PRICE`, `CART_BTN`, `PRICE_CART_BTN` (element lock), Liquid `access_request` được sinh trực tiếp inline trong switch-case của `generateContentElementLock()` — không thông qua `getContentCustomerApproval()`:

```liquid
{% elsif condition_type == 'access_request' %}
  {% if customer == blank %}
    <span class='bss-lock-price-login-prompt-{rule.id}'>{loginPrompt}</span>
  {% elsif b2b_pending_{rule.id} contains b2b_cid_{rule.id} %}
    <span class='bss-lock-price-pending-access-{rule.id}'>{pendingMessage}</span>
  {% elsif b2b_rejected_{rule.id} contains b2b_cid_{rule.id} %}
    <span class='bss-lock-price-rejected-access-{rule.id}'>{rejectedMessage}</span>
  {% elsif b2b_revoked_{rule.id} contains b2b_cid_{rule.id} %}
    <span class='bss-lock-price-revoked-access-{rule.id}'>{revokedMessage}</span>
  {% else %}
    {%- assign b2b_ca_token_{rule.id} = b2b_ca_payload_atc_{rule.id} | hmac_sha256: '{KEY_HASH_PASSCODE}' -%}
    <span class='bss-lock-price-request-access-{rule.id}'
      data-create-request-url="{SERVER_URL}/customer-approval/public/create-request"
      onclick="document.getElementById('bss-b2b-lock-ca-modal').style.display='flex'">
      {requestMessage}
    </span>
  {% endif %}
```

Biến `b2b_pending_{rule.id}`, `b2b_rejected_{rule.id}`, v.v. được gán từ metaobject tại condition check (section 14.3) — không re-assign inline ở element lock.

---

### 14.8. File được upload lên theme

Không có snippet Liquid độc lập mới nào. Các file được upload/update:

| File trên theme | Mô tả |
|---|---|
| `snippets/bss-lock-content.liquid` | Chứa capture block + render block cho `access_request` |
| `assets/bss-lock-settings.css` | Chứa CSS màu sắc từ `getStyleRequestAccess()` |
| `snippets/bss-lock-condition.liquid` | Chứa Liquid check `temp_{condition.id}` cho condition `access_request` |

---

## 15. Email notifications

| Event | Trigger | Recipient | Template |
|---|---|---|---|
| New request submitted | `create-request` endpoint | **Merchant** | `customerApprovalNotifyMerchantTemplate` |
| Request approved (single) | `update-request` với status=approved | **Customer** | `customerApprovalApprovedTemplate` |
| Request approved (bulk) | `bulk-update-requests` với status=approved | **Customer** (mỗi customer) | `customerApprovalApprovedTemplate` |
| Request rejected | `update-request` / `bulk-update-requests` với status=rejected | ❌ Không gửi | — |
| Access revoked | `update-request` với status=revoked | ❌ Không gửi | — |

Email gửi qua SES (`/bss-ses/v2/send`), from: `no-reply-shopify@bsscommerce.com`.  
Merchant email: lấy từ `shop.email_receive_passcode_notification`.  
Customer email: lấy từ `request.email` (được submit khi tạo request).

---

## 16. Các thành phần đã implement (MR !532)

```
LOGIN-385 ──────────────────────────────────────┐
                                                 ↓
LOGIN-396 (entity trước) ──▶ LOGIN-395 ──▶ LOGIN-386
                         └──▶ LOGIN-387 (sau 386)
```

| Ticket | Status | Ghi chú |
|---|---|---|
| LOGIN-385 | ✅ Done | Migration `designs-v2.request_access`, constants, rule type `messages` |
| LOGIN-396 | ✅ Done | Migration `rules-v2.messages`, entity CustomerAccessRequests, public endpoint, BullMQ processor |
| LOGIN-386 | ✅ Done | GET list, update-request, bulk-update-requests, email notifications |
| LOGIN-395 | Storefront/upload | Liquid/JS files + upload.service integration |
| LOGIN-399 | ✅ Done (tích hợp vào 386) | Email templates trong `email-template.const.ts` |

---

## 17. Quyết định cần confirm trước khi implement tiếp

| # | Câu hỏi | Ảnh hưởng |
|---|---|---|
| 1 | Access có expire không? | Nếu có → thêm `expired_at` + cron cleanup Metaobject |
| 2 | Customer rejected có thể submit lại không? | Code hiện tại: **CÓ** — `createRequest` reset về `pending` nếu record đã tồn tại |
| 3 | Metaobject shard khi > ~7.000 IDs: tự động hay báo lỗi? | Nếu tự động → logic shard phức tạp hơn |
| 4 | Revoked customer có nhận email thông báo không? | Hiện tại: KHÔNG gửi |

---

## 18. Acceptance criteria

- Condition picker User type có "Approval required" option (condition type = `access_request`).
- Guest → thấy login prompt, click → redirect account/login với return_url.
- Customer logged-in, chưa request → thấy form → submit → pending state (inline, không reload).
- Customer logged-in, đang pending → thấy pending message, không thấy form.
- Customer logged-in, bị rejected → thấy rejected message.
- Submit create-request → merchant nhận email notification.
- Merchant approve → Customer nhận email → Metaobject được update → customer reload → nội dung hiện ngay (Liquid check).
- Merchant revoke → customer_id chuyển sang `revoked_customer_ids` → content bị ẩn lại.
- Merchant reject → không gửi email customer.
- Trang Customer Access Requests: list (page 100), filter, update-request, bulk-update-requests hoạt động đúng.
- `StoreFrontGuard` verify HMAC hash trước khi chấp nhận `shopify_customer_id` từ body.
- BullMQ processor rebuild Metaobject đúng sau mỗi status change.
- Throttle: 1 request/60s per IP cho public endpoint.
