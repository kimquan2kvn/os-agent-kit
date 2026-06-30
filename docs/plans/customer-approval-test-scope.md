# Customer Approval — Test Scope

Feature: `access_request` condition type  
MR: !532 (`feat/LOGIN-387` → staging)  
Ref: `customer-approval-metaobject.md`

---

## 1. Storefront — Liquid rendering

### 1.1 Guest (chưa đăng nhập)
- [ ] Truy cập page có rule `access_request` khi chưa login → hiện **login prompt**
- [ ] Nút "Log in" redirect đúng `/account/login?return_url=<current_page>`
- [ ] Không hiện form submit, không có JS call nào được thực thi

### 1.2 Customer logged-in, chưa có request
- [ ] Liquid check metaobject: cid không có trong approved/pending/rejected → hiện **request form**
- [ ] Form có đầy đủ data-attribute: `data-domain`, `data-rule-id`, `data-customer-id`, `data-token`
- [ ] `data-token` là HMAC hash, không phải raw key

### 1.3 Customer logged-in, đang pending
- [ ] `pending_customer_ids contains customer.id` → hiện **pending message** (không hiện form)
- [ ] Không có nút submit, không có JS call
- [ ] Nội dung pending message khớp với config merchant đã set

### 1.4 Customer logged-in, bị rejected
- [ ] `rejected_customer_ids contains customer.id` → hiện **rejected message**
- [ ] Không hiện form submit

### 1.5 Customer đã được approved
- [ ] `approved_customer_ids contains customer.id` → **hiện nội dung bình thường** ngay lập tức
- [ ] Không hiện bất kỳ form nào

### 1.6 Customer bị revoked
- [ ] Sau khi merchant revoke → `revoked_customer_ids` chứa cid → nội dung bị ẩn
- [ ] Customer reload → không thấy nội dung (không còn trong approved list)

---

## 2. Storefront — Submit request (JS)

### 2.1 Submit thành công
- [ ] Click submit → JS POST `/customer-approval/public/create-request`
- [ ] Header `authentication: Bearer <token>` được gửi đúng (lấy từ `data-token`)
- [ ] Body đúng: `{ domain, ruleId, shopify_customer_id, email, name, message? }`
- [ ] Response `{ message: 'Created successfully' }` → form thay bằng **pending message inline** (không reload page)

### 2.2 Submit lại sau khi bị rejected
- [ ] Customer bị rejected → reload page → thấy rejected message (không thấy form)
- [ ] _(Nếu merchant reset/allow resubmit)_ → Customer submit lại → response `{ message: 'Updated successfully' }` → DB record reset về `pending`
- [ ] Sau submit lại → Metaobject cập nhật: cid chuyển sang `pending_customer_ids`

### 2.3 Throttle
- [ ] Submit 2 request liên tiếp từ cùng IP trong 60s → request thứ 2 bị **429 Too Many Requests**
- [ ] Sau 60s → submit được bình thường

### 2.4 StoreFrontGuard — security
- [ ] Gửi request với `shopify_customer_id` bị giả mạo (không khớp token) → **401/403**
- [ ] Gửi request không có header `authentication` → **401/403**
- [ ] Token hợp lệ nhưng `domain` bị thay → **401/403**

---

## 3. API — Public endpoint

### POST `/customer-approval/public/create-request`

- [ ] Request hợp lệ → trả `{ message: 'Created successfully' }`, DB có record `status=pending`
- [ ] Request hợp lệ với record đã tồn tại (bất kể status cũ) → trả `{ message: 'Updated successfully' }`, status reset về `pending`
- [ ] Sau create/update → **BullMQ job được enqueue** với jobId `customer-approval-shop-{shopId}-rule-{ruleId}`
- [ ] Sau create → **email notify merchant** được gửi (kiểm tra SES log hoặc mock)
- [ ] Missing required field → 400 Bad Request
- [ ] Invalid HMAC → 401/403

---

## 4. API — Admin endpoints (ShopGuard)

### GET `/customer-approval/`

- [ ] Trả danh sách requests của đúng shop (không leak data shop khác)
- [ ] Filter `status=pending` → chỉ trả pending records
- [ ] Filter `status=approved` → chỉ trả approved records
- [ ] Filter `ruleId=X` → chỉ trả records của rule X
- [ ] Filter `email=partial` → LIKE search, case-insensitive
- [ ] Filter `name=partial` → LIKE search, case-insensitive
- [ ] Filter `startDate` & `endDate` → đúng range
- [ ] Pagination: `page=1` trả 100 records, `page=2` trả records tiếp theo
- [ ] Response có đủ fields: `{ items, total, page, limit, totalPages }`
- [ ] Request không có session-token → 401

### POST `/customer-approval/update-request`

- [ ] `{ id, status: 'approved' }` → DB update, email gửi cho **customer**, BullMQ enqueue
- [ ] `{ id, status: 'rejected' }` → DB update, **không gửi email**, BullMQ enqueue
- [ ] `{ id, status: 'revoked' }` → DB update, **không gửi email**, BullMQ enqueue
- [ ] `id` không thuộc shop đang gọi → không update (shopId check)
- [ ] `id` không tồn tại → affected = 0
- [ ] Missing `id` → return sớm (không crash)

### POST `/customer-approval/bulk-update-requests`

- [ ] `{ ids: [1,2,3], status: 'approved' }` → tất cả records update, email gửi cho **từng customer** được approved
- [ ] `{ ids: [1,2,3], status: 'rejected' }` → update, **không gửi email**
- [ ] Chỉ update records thuộc đúng shop (ids của shop khác bị bỏ qua)
- [ ] BullMQ enqueue cho từng `ruleId` duy nhất trong batch (không duplicate job)
- [ ] `ids` rỗng → return sớm, không crash
- [ ] Email gửi fire-and-forget (`Promise.allSettled`) — 1 email fail không làm fail toàn bộ request

---

## 5. BullMQ Processor — Metaobject rebuild

### 5.1 Basic rebuild
- [ ] Sau `create-request` → BullMQ job chạy → Metaobject `rule-{ruleId}` được tạo (nếu chưa có) với đúng 4 fields
- [ ] Sau `update-request` approve → job chạy → cid chuyển từ `pending_customer_ids` sang `approved_customer_ids`
- [ ] Sau `update-request` reject → cid chuyển sang `rejected_customer_ids`
- [ ] Sau `update-request` revoke → cid chuyển sang `revoked_customer_ids`
- [ ] Mỗi cid chỉ xuất hiện trong **đúng 1 list** tại bất kỳ thời điểm nào

### 5.2 Job deduplication
- [ ] Nhiều thay đổi liên tiếp trên cùng shopId+ruleId trong 500ms → chỉ **1 job** được xử lý (cùng `jobId` → replace)
- [ ] Metaobject cuối cùng phản ánh đúng trạng thái DB sau khi tất cả thay đổi hoàn tất

### 5.3 Version counter
- [ ] Nếu có thay đổi mới trong lúc processor đang rebuild → processor lặp lại rebuild (không bỏ sót)
- [ ] Sau khi version stable → Redis version key bị xóa

### 5.4 Metaobject definition auto-create
- [ ] Shop mới (chưa có Metaobject definition) → processor tự tạo definition trước khi create Metaobject
- [ ] Definition đã tồn tại → skip tạo, không báo lỗi
- [ ] Lỗi `TAKEN` khi tạo definition → được bỏ qua (idempotent)

### 5.5 Error handling
- [ ] Shopify API call fail → processor throw error → BullMQ retry (attempts: 3, exponential backoff 1s)
- [ ] Sau 3 lần retry thất bại → job lưu vào failed queue (age: 5 ngày, count: 1000)

---

## 6. Email notifications

### 6.1 Notify merchant khi có request mới
- [ ] Customer submit request → email gửi đến `shop.email_receive_passcode_notification`
- [ ] Subject: `New access request from {name} — {shop_name}`
- [ ] Body chứa: customer email, customer name, rule name, message
- [ ] Body chứa link deep-link vào CMS: `...?tab=email-request&customer-request-id={id}`
- [ ] From: `no-reply-shopify@bsscommerce.com`

### 6.2 Notify customer khi được approved
- [ ] Merchant approve → email gửi đến customer email
- [ ] Subject: `Your access request has been approved — {shop_name}`
- [ ] Body chứa: customer name, store name, store URL
- [ ] From: `no-reply-shopify@bsscommerce.com`
- [ ] Bulk approve → **mỗi** customer nhận 1 email riêng

### 6.3 Không gửi email
- [ ] Reject (single/bulk) → **không** gửi email customer
- [ ] Revoke → **không** gửi email customer

---

## 7. CMS — Config panel

### 7.1 Condition picker
- [ ] Dropdown User type có option "Approval required"
- [ ] Chọn "Approval required" → hiện config panel đúng
- [ ] Summary sidebar hiện `👤 Approval required`

### 7.2 Message config
- [ ] Có đủ 5 text fields: Request message, Pending message, Rejected message, Revoked message, Login prompt
- [ ] Default values đúng (xem spec)
- [ ] Merchant edit và save → `rules-v2.messages.request_access` được cập nhật
- [ ] Liquid render đúng message đã config

### 7.3 Notification toggle
- [ ] Toggle ON → `designs-v2.request_access` lưu email notify ON
- [ ] Toggle OFF → không gửi email merchant khi có request mới
- [ ] Email field cập nhật đúng merchant email

---

## 8. CMS — Customer Access Requests page

### 8.1 Hiển thị danh sách
- [ ] List hiện đúng records của shop hiện tại
- [ ] Hiện đủ columns: Customer (email), Rule name, Requested (time), Status
- [ ] Page size 100 records mỗi trang
- [ ] Pagination hoạt động đúng

### 8.2 Filter
- [ ] Filter "All rules" / chọn rule cụ thể → đúng
- [ ] Filter "All status" / pending / approved / rejected / revoked → đúng
- [ ] Search by email → partial match, case-insensitive
- [ ] Search by name → partial match, case-insensitive

### 8.3 Single actions
- [ ] Pending record: nút `[Approve]` và `[Reject]` hiện
- [ ] Click Approve → call `update-request` status=approved → row update, customer nhận email
- [ ] Click Reject → call `update-request` status=rejected → row update, không gửi email
- [ ] Approved record: nút `[Revoke]` hiện
- [ ] Click Revoke → call `update-request` status=revoked → row update, không gửi email

### 8.4 Bulk actions
- [ ] Chọn nhiều pending records → `[Bulk approve]` + `[Bulk reject]` active
- [ ] Bulk approve → call `bulk-update-requests` → tất cả records update, customers nhận email
- [ ] Bulk reject → call `bulk-update-requests` → tất cả records update, không gửi email

---

## 9. Metaobject — Shopify Admin verification

- [ ] Sau khi có request → Shopify Admin → Content → Metaobjects → `b2b_lock_customer_access` → handle `rule-{ruleId}` tồn tại
- [ ] Fields `approved_customer_ids`, `pending_customer_ids`, `rejected_customer_ids`, `revoked_customer_ids` có giá trị đúng
- [ ] Sau approve → `approved_customer_ids` chứa cid, `pending_customer_ids` không chứa cid
- [ ] Sau revoke → `revoked_customer_ids` chứa cid, `approved_customer_ids` không chứa cid

---

## 10. End-to-end flows

### Flow 1: Happy path — Guest → Approved
1. [ ] Guest truy cập page → thấy login prompt
2. [ ] Login → reload → thấy request form
3. [ ] Submit form → thấy pending message inline
4. [ ] Merchant vào CMS → thấy request trong danh sách
5. [ ] Merchant approve → Metaobject rebuild
6. [ ] Customer reload page → thấy **nội dung đầy đủ**
7. [ ] Customer nhận email "access approved"
8. [ ] Merchant nhận email notify khi customer submit

### Flow 2: Reject flow
1. [ ] Customer submit request
2. [ ] Merchant reject
3. [ ] Customer reload → thấy **rejected message**
4. [ ] Customer **không** nhận email

### Flow 3: Revoke flow
1. [ ] Customer đã được approved, đang xem nội dung
2. [ ] Merchant revoke
3. [ ] Customer reload → **không thấy nội dung** (bị ẩn lại)

### Flow 4: Resubmit sau reject
1. [ ] Customer bị rejected
2. [ ] Customer reload → thấy rejected message (không thấy form)
3. [ ] _(Nếu feature resubmit được mở)_ Merchant reset → Customer submit lại → status về pending

### Flow 5: Bulk approve
1. [ ] 3 customers submit request
2. [ ] Merchant vào CMS, chọn tất cả → Bulk approve
3. [ ] 3 customers đều nhận email approved
4. [ ] BullMQ chỉ tạo 1 job (cùng shopId+ruleId) nhưng rebuild phản ánh đúng tất cả 3

---

## 11. Edge cases & regression

- [ ] Rule có condition `access_request` nhưng Metaobject chưa tồn tại → Liquid render state `form` (không crash, không lỗi)
- [ ] Rule bị xóa → `customerAccessRequests` records có `ruleId` đó → không crash (ON DELETE NO ACTION)
- [ ] Customer ID rất dài (13 ký tự) → lưu đúng, không bị truncate
- [ ] Nhiều rules cùng shop với `access_request` condition → Metaobject độc lập per rule (handle `rule-1`, `rule-2`)
- [ ] Rule không phải `access_request` → không bị ảnh hưởng bởi feature này (regression)
- [ ] Existing rules (tag, passcode, v.v.) vẫn hoạt động bình thường sau khi deploy MR !532

---

## 12. Môi trường test

| Item | Value |
|---|---|
| Branch | `feat/LOGIN-387` (đã merge vào staging) |
| Endpoint base | staging API |
| Shopify store | staging store có App cài |
| Tool verify email | SES log / Mailtrap / mock |
| Tool verify BullMQ | Bull Board hoặc Redis CLI `LRANGE bull:customer_approval_queue:*` |
| Tool verify Metaobject | Shopify Admin → Content → Metaobjects |
