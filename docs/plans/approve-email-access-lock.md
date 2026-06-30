Phân tích từ docs

Theo docs/modules/lock-rules.md, hiện có 2 condition liên quan đến email khác nhau:

Condition

Behavior hiện tại

Specific customers

Merchant nhập tay danh sách email cụ thể

Email subscribers

Khách nhập email → tự động có access ngay

Feature mới nằm ở giữa hai cái này: Khách nhập email → chờ merchant duyệt → mới có access.

Đề xuất tích hợp

Không thêm condition type mới — mở rộng "Email subscribers"

Khi merchant chọn Email trong Grant Access To, thêm sub-option:

┌─────────────────────────────────────────────────────────┐
│  ● Email  📧                                            │
│  ─────────────────────────────────────────────────────  │
│  How should access be granted?                          │
│                                                         │
│  ○ Auto-approve  (current behavior)                     │
│    Customer submits email → instant access              │
│                                                         │
│  ● Require approval  ← MỚI                             │
│    Customer submits email → you review & approve        │
│                                                         │
│  [ ] Notify me via email when new requests arrive       │
│      Send to: [merchant@email.com            ]          │
└─────────────────────────────────────────────────────────┘


Các điểm tích hợp cụ thể theo từng bước wizard

Step 1 — Grant Access To (Email condition)

Thêm 2 radio: Auto-approve / Require approval

Khi chọn Require approval → hiện thêm toggle notify + email field

Step 2 — Access & Message

Lock message form cho phép customize text của email request form trên storefront:

Form title: "Request access to view prices"

Button label: "Send request"

Success message: "Your request has been sent..."

Summary panel (cột phải)

Rule to access:
• Email subscribers (Approval required)  ← thay vì chỉ "Email subscribers"


Tab mới trong navigation chính

Tương tự Passcode Requests đã có, thêm Email Access Requests:

┌────────────────────────────────────────────────────────────────┐
│  Email Access Requests                                         │
│  ─────────────────────────────────────────────────────────     │
│  [All rules ▼]  [All status ▼]  [Date range]    [Search email] │
│                                                                │
│  Email               Rule name      Requested    Status        │
│  john@gmail.com      View prices    2 hrs ago    [Approve][Reject] │
│  mary@company.com    View prices    1 day ago    ✓ Approved    │
│  bob@gmail.com       View catalog   3 days ago   ✗ Rejected    │
│                                                                │
│  [Bulk approve selected]                                       │
└────────────────────────────────────────────────────────────────┘


So sánh với Passcode Requests để thấy độ tương đồng



Passcode Requests (có sẵn)

Email Access Requests (mới)

Customer làm gì

Điền form xin passcode

Điền email xin access

Merchant làm gì

Approve → gửi passcode qua email

Approve → tự động unlock

Customer dùng access

Nhập passcode vào form

Tự động (qua cart attribute + metaobject)

Tab quản lý

Passcode Requests

Email Access Requests

Notify merchant

Có (toggle)

Có (toggle, tương tự)

# Tech flow

1. Bối cảnh & Vấn đề

Hiện tại condition type email trong Rules yêu cầu merchant thêm email thủ công vào từng key mỗi khi có khách hàng mới cần truy cập. Cách này không scale và tốn công quản lý.

Mục tiêu: Xây luồng self-service — khách tự submit email, merchant chỉ cần approve/reject trên dashboard, hệ thống tự động cấp quyền mà không động đến keys.

2. Kiến trúc kỹ thuật tổng thể

Cơ chế lưu trữ:

Cart Attribute (b2b_email_access_id) — lưu unique_id trên cart của customer, dùng để Liquid check real-time

Metaobject (b2b_price_access) — lưu danh sách unique_id đã được approved theo từng rule

localStorage — persist unique_id trên thiết bị, tự restore khi cart bị clear

Tại sao không dùng rule keys? Rule keys sẽ phình to theo số lượng email, khó quản lý. Metaobject tách biệt, update độc lập.

3. unique_id

unique_id = HMAC-SHA256(email, per_shop_salt)


Deterministic: cùng email luôn ra cùng unique_id

Server giữ per_shop_salt → client không thể tự giả mạo unique_id của người khác

Không expose email khi check status

Flow generate:

Customer nhập email
  → POST /email-access/public/generate-id { email, domain }
  → Server compute HMAC → trả về { unique_id }
  → Client lưu vào localStorage + update cart attribute


Không generate phía client để bảo vệ salt.

4. Flow hoàn chỉnh

4a. Lần đầu — Customer chưa có access

[Storefront]
  Liquid render giá bị ẩn
  → JS kiểm tra cart.attributes.b2b_email_access_id
  → Trống → kiểm tra localStorage['b2b_access_id']
  → Trống → hiện form "Nhập email để yêu cầu xem giá"

[Customer nhập email + submit]
  → POST /email-access/public/generate-id { email, domain }
       ← { unique_id }
  → PUT /cart/update.js { attributes: { b2b_email_access_id: unique_id } }
  → localStorage.setItem('b2b_access_id', unique_id)
  → POST /email-access/public/create-request
       { email, unique_id, ruleId, domain, name?, message? }
  → Hiện thông báo: "Yêu cầu đã gửi, chờ phê duyệt"


4b. Merchant approve trên CMS

[CMS]
  Merchant vào tab "Email Access Requests"
  → Xem danh sách pending (email, tên, trang, rule, ngày gửi)
  → Nhấn Approve

[login-api-v2]
  POST /email-access/approve { requestId }
  → Cập nhật DB: status = approved
  → Gọi Shopify Metaobject API:
       upsert type "b2b_price_access":
       { rule_id: "123", approved_ids: [..., "new_unique_id"] }
  → Gửi email xác nhận cho customer:
       "Bạn đã được cấp quyền xem giá tại [store]"


4c. Liquid check khi render

{% assign access_id = cart.attributes.b2b_email_access_id %}
{% assign rule_meta = shop.metaobjects.b2b_price_access
                      | where: "rule_id", rule.id | first %}
{% assign approved = rule_meta.approved_ids.value %}

{% if access_id != blank and approved contains access_id %}
  {{ product.price | money }}
{% else %}
  {% render 'b2b-email-request-form', rule: rule %}
{% endif %}


5. Flow Re-verify (Cart bị clear)

Case 1 — Cùng thiết bị

Page load → cart attribute trống
  → JS check localStorage['b2b_access_id'] → có giá trị
  → Silent PUT /cart/update.js restore cart attribute
  → Không cần user thao tác gì


Case 2 — Thiết bị mới / incognito

Page load → cart attribute trống + localStorage trống
  → Hiện form "Nhập email để xem giá"

Customer nhập email
  → POST /email-access/public/generate-id { email, domain }
       ← { unique_id }
  → POST /email-access/public/check-status { unique_id, ruleId, domain }
       ← { status: 'approved' | 'pending' | 'rejected' | 'not_found' }

  Nếu approved:
    → localStorage.setItem + PUT cart/update.js
    → Giá hiển thị (không cần merchant duyệt lại)

  Nếu pending:
    → "Yêu cầu của bạn đang chờ phê duyệt"

  Nếu rejected:
    → "Yêu cầu của bạn đã bị từ chối"

  Nếu not_found:
    → Tạo request mới (flow 4a)


Sơ đồ tổng hợp

Page load
    │
    ├─ cart attribute có? ──YES──▶ Liquid check Metaobject ──▶ Hiện/Ẩn giá
    │
    NO
    │
    ├─ localStorage có? ──YES──▶ Silent restore cart attr ──▶ Done
    │
    NO
    │
    └─ Hiện form nhập email
              │
              ├─ generate-id + check-status
              │
              ├─ approved  ──▶ Restore + hiện giá
              ├─ pending   ──▶ Thông báo chờ
              ├─ rejected  ──▶ Thông báo từ chối
              └─ not_found ──▶ create-request mới


6. Các thành phần cần xây dựng

Backend — login-api-v2 (module mới: email-access/)

Endpoint

Method

Guard

Mô tả

/email-access/public/generate-id

POST

Public + Throttle

Compute HMAC unique_id từ email

/email-access/public/create-request

POST

Public + Throttle

Tạo request mới

/email-access/public/check-status

POST

Public + Throttle

Kiểm tra status theo unique_id

/email-access/request

GET

ShopGuard

Merchant lấy danh sách requests

/email-access/approve

POST

ShopGuard

Approve → update Metaobject

/email-access/reject

POST

ShopGuard

Reject request

/email-access/delete-request

POST

ShopGuard

Xóa request

Entity mới: EmailAccessRequests

@Entity('emailAccessRequests')
class EmailAccessRequests {
  id: number
  domain_id: number
  rule_id: number
  email: string
  unique_id: string       // HMAC result
  name?: string
  message?: string
  status: 'pending' | 'approved' | 'rejected'
  createdAt: Date
  updatedAt: Date
}


Migration: Tạo bảng emailAccessRequests + index trên (unique_id, rule_id) để check-status nhanh.

CMS — login-cms

Thành phần

Mô tả

Tab "Email Access Requests"

Tương tự tab Passcode Requests

Danh sách requests

Filter theo status / rule / ngày

Bulk approve/reject

Xử lý nhiều request cùng lúc

Toggle nhận thông báo

Bật/tắt email notification cho merchant

Storefront — Liquid/JS

Thành phần

Mô tả

b2b-email-request-form.liquid

Snippet form nhập email

b2b-email-access.js

Logic generate-id, check-status, restore cart attr, localStorage

Inject vào hide-price flow

Tích hợp vào hide-price.service.ts upload pipeline

7. Security

Rủi ro

Xử lý

Giả mạo unique_id

unique_id = HMAC với server-side salt → client không tự tính được

Brute-force check-status

Throttle: 1 req/60s per IP (tương tự passcode)

Enumerate email qua check-status

Endpoint chỉ nhận unique_id, không nhận email → không thể reverse

Metaobject size limit (~128KB)

Nếu > 5000 approved IDs / rule → shard thành nhiều metaobject entries

8. Quyết định cần confirm trước khi implement

#

Câu hỏi

Ảnh hưởng

1

Access có thời hạn không (expire)?

Nếu có → cần thêm expired_at + cron cleanup Metaobject

2

Scope request: theo rule hay toàn shop?

Nếu toàn shop → Metaobject structure khác

3

Customer submit lại sau khi rejected?

Cho phép hay block?

4

Merchant có cần xem lịch sử ai đã access không?

Nếu có → cần log thêm

