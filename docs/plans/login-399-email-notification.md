# LOGIN-399 — [API] Email Notification

Phụ thuộc: LOGIN-396 (entity + public endpoint) + LOGIN-386 (approve/reject endpoints).

---

## 1. Tóm tắt kiến trúc

Email gửi qua **BSS SES** (`bss-ses/v2/send`) — không phải Brevo (Brevo chỉ dùng cho analytics event tracking).

Mở rộng `EmailService` hiện có tại `login-api-v2/src/modules/external/email/email.service.ts`.  
Template constants thêm vào `login-api-v2/src/common/constants/email.const.ts`.

---

## 2. Quyết định bulk email — Option B

**Single approve**: gửi 1 email ngay lập tức.  
**Bulk approve**: group requests theo `email` → gửi **1 email duy nhất** mỗi customer, liệt kê tất cả rules được approve trong batch.

Không dùng queue (Bull/BullMQ) — over-engineering cho quy mô hiện tại.  
Tất cả email đều **fire-and-forget**: DB + Metaobject update xong → trả response → gửi email async.

```typescript
// Bulk approve — gom theo email trước khi gửi
const groupedByEmail = requests.reduce((acc, req) => {
  if (!acc[req.email]) acc[req.email] = { request: req, rules: [] };
  acc[req.email].rules.push(req.rule);
  return acc;
}, {} as Record<string, { request: CustomerAccessRequests; rules: Rules[] }>);

// Fire-and-forget sau khi toàn bộ DB + Metaobject xong
Promise.allSettled(
  Object.values(groupedByEmail).map(({ request, rules }) =>
    emailService.sendApprovalApprovedToCustomer(request, rules, shop)
  )
).then(results => {
  results.forEach((r, i) => {
    if (r.status === 'rejected')
      this.logger.warn(`Approval email failed for ${Object.keys(groupedByEmail)[i]}: ${r.reason}`);
  });
});
```

Reject thường là hành động đơn, không cần group.

---

## 3. 3 triggers gửi email

| Trigger | Gửi đến | Method |
|---------|---------|--------|
| `create-request` | Merchant (nếu `notify_email != null`) | `sendApprovalNotifyMerchant(...)` |
| `approve` (single/bulk) | Customer | `sendApprovalApprovedToCustomer(request, rules[], shop)` |
| `reject` (single/bulk) | Customer | `sendApprovalRejectedToCustomer(request, rule, shop)` |

---

## 4. Email Templates

### 4a. Merchant notification — `customerApprovalNotifyMerchantTemplate`

**Placeholders:**

| Placeholder | Giá trị |
|-------------|---------|
| `{{customer_name}}` | `request.name` |
| `{{customer_email}}` | `request.email` |
| `{{rule_name}}` | `rule.name` |
| `{{message_content}}` | `request.message ?? '(No message provided)'` |
| `{{app_url}}` | Deep link CMS tab Customer Access Requests |
| `{{store_name}}` | `shop.shop_name` |
| `{{store_url}}` | `shop.domain` |

---

### 4b. Customer approved — `customerApprovalApprovedTemplate`

**Placeholders:**

| Placeholder | Giá trị |
|-------------|---------|
| `{{customer_name}}` | `request.name` |
| `{{store_name}}` | `shop.shop_name` |
| `{{store_url}}` | `shop.domain` |
| `{{rules_html}}` | HTML generated từ danh sách rules |

**Service generate `rules_html`:**

```typescript
const rulesHtml = rules
  .map(r => `<p style="font-size:14px;color:#065f46;margin:0 0 8px;font-weight:500;line-height:1.5;">
    <span style="color:#059669;margin-right:8px;">&#10003;</span>${r.name}
  </p>`)
  .join('');
```

---

### 4c. Customer rejected — `customerApprovalRejectedTemplate`

**Placeholders:**

| Placeholder | Giá trị |
|-------------|---------|
| `{{customer_name}}` | `request.name` |
| `{{store_name}}` | `shop.shop_name` |
| `{{store_url}}` | `shop.domain` |
| `{{rule_name}}` | `rule.name` |
| `{{rejected_message}}` | `condition.rejected_message` từ rule config |

---

## 5. Error handling

| Tình huống | Xử lý |
|------------|-------|
| `SES_HOST` không set | Skip — guard `if (process.env.SES_HOST)` như pattern hiện có |
| SES trả lỗi | `logger.warn(domain + requestId)` — không throw, không rollback DB |
| `notify_email` null | Guard `if (notifyEmail)` trước khi gọi |
| Bulk: 1 email fail | `Promise.allSettled` — log warn, không ảnh hưởng các email khác |

---

## 6. Files cần sửa

| File | Action |
|------|--------|
| `src/common/constants/email.const.ts` | Thêm 3 template constants |
| `src/modules/external/email/email.service.ts` | Thêm 3 method mới |
| `src/modules/internal/customer-approval/customer-access-requests.module.ts` | Import `EmailModule` |
| `src/modules/internal/customer-approval/customer-access-requests.service.ts` | Inject `EmailService`, wire vào create-request, approve, reject |

---

## 7. Open questions

1. BSS SES có rate limit không? Bulk 100+ requests → 100 HTTP calls đồng thời.
2. `appHandle` để tạo deep link CMS lấy từ đâu trong `CustomerAccessRequestsService`?
3. Customer bị reject rồi re-submit — có gửi lại email merchant không? (Hiện tại sẽ gửi lại vì `createOrUpdateRequest` không phân biệt create vs update.)
4. Email approved/rejected có dùng `condition.rejected_message` làm nội dung body không?
