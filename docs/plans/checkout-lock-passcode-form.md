# Plan: Checkout Lock — Show Passcode Form

## 1. Bối cảnh & Vấn đề

Merchant muốn:
- Customer vẫn browse sản phẩm bình thường
- Khi bấm checkout → **phải nhập passcode** → payment methods mới hiện ra

Hiện tại `checkout-lock` (cart-checkout-validation Function) chỉ **block** checkout với error message, không hỗ trợ passcode như một điều kiện.  
`payment-customization` Function hiện có các condition: `customer_tag`, `cart_total`, `product_specific`, `signed_in`, `customer_phone`, `customer_email`, `product_quantity`, `cart_quantity` — **chưa có `checkout_passcode`**.

---

## 2. Giải pháp tổng thể

Dùng combination **Checkout UI Extension + Payment Customization Function + App Proxy**:

```
Customer bấm checkout
    │
    ▼
Payment Function runs
    └─ Condition type: checkout_passcode
    └─ Check cart.attributes._bss_passcode_verified → absent → ẩn tất cả payment methods
    │
    ▼
Checkout UI Extension renders passcode form
(vì payment methods đang ẩn, chưa checkout được)
    │
Customer nhập passcode
    │
    ▼
UI Extension → POST /app-proxy/checkout-passcode/verify { passcode, shopDomain, ruleId }
    │
    ├─ FAIL → hiện error message, cho nhập lại
    │
    └─ SUCCESS
        │
        ▼
        applyAttributeChange({ key: '_bss_passcode_verified', value: '1' })
        Shopify re-evaluate Payment Function
        └─ Thấy attr → bỏ ẩn payment methods
        UI Extension ẩn form (self-hide khi attr đã set)
        │
        ▼
    Customer hoàn tất checkout bình thường
```

### Tại sao dùng cart attribute thay vì signed token?

Payment Functions là sandboxed WASM — không thể call external API hay verify crypto.  
Chúng chỉ đọc được data từ Shopify (metafields, cart data).  
Cart attribute `_bss_passcode_verified` đơn giản, Function đọc được qua `cart.attributes`.  
**Security note:** Cart attributes có thể bị set thủ công qua Storefront API, nhưng đây là UI-level restriction (không phải payment gateway block), chấp nhận được cho use-case B2B.  
Nếu cần stronger security trong tương lai: store token trong metafield per-session.

---

## 3. Các thành phần cần xây dựng

### 3A. Extension mới: `checkout-passcode-ui`
**Repo:** `b2b-login-access-management-script`  
**Type:** `purchase.checkout.block.render` (Checkout UI Extension)

**File structure:**
```
extensions/checkout-passcode-ui/
├── shopify.extension.toml
├── locales/en.default.json
├── src/
│   ├── index.tsx          ← entry, đọc extension metafield để lấy ruleId
│   └── PasscodeForm.tsx   ← UI component
└── package.json
```

**Logic `index.tsx`:**
```tsx
// Đọc metafield extension config: { ruleId, shopDomain }
// Đọc cart.attributes._bss_passcode_verified
// Nếu đã verified → return null (ẩn form)
// Nếu chưa → render <PasscodeForm />
```

**Logic `PasscodeForm.tsx`:**
```tsx
// Input text for passcode
// Submit → fetch App Proxy endpoint
// Loading / Error state
// On success → applyAttributeChange({ key: '_bss_passcode_verified', value: '1' })
```

**shopify.extension.toml:**
```toml
[[extensions]]
type = "ui_extension"
name = "Checkout Passcode Form"
handle = "checkout-passcode-ui"

[[extensions.targeting]]
module = "./src/index.tsx"
target = "purchase.checkout.payment-method-list.render-before"

[extensions.settings]
  [[extensions.settings.fields]]
  key = "rule_id"
  type = "number_integer"
  
  [[extensions.settings.fields]]
  key = "shop_domain"
  type = "single_line_text_field"
```

**Metafield namespace** để UI Extension đọc config:
`$app:checkout-passcode` key: `config` → `{ ruleId: number, shopDomain: string }`

---

### 3B. Update Extension: `payment-customization`
**Repo:** `b2b-login-access-management-script`

**Thay đổi cần thiết:**

#### 1. `const/common.const.ts` — thêm constant
```ts
export const CONDITIONS = Object.freeze({
  // ... existing ...
  CHECKOUT_PASSCODE: "checkout_passcode",  // NEW
});
```

#### 2. `cart_payment_methods_transform_run.graphql` — thêm cart.attributes
```graphql
cart {
  # ... existing fields ...
  attributes {      # NEW
    key
    value
  }
}
```

#### 3. `rule.ts` — thêm validator cho checkout_passcode
```ts
const validateCheckoutPasscode = (
  cart: CartPaymentMethodsTransformRunInput['cart']
) => {
  const attr = cart.attributes?.find(a => a.key === '_bss_passcode_verified');
  return attr?.value === '1';
};

// Trong switch của validateRule:
case CONDITIONS.CHECKOUT_PASSCODE:
  return validateCheckoutPasscode(input.cart);
```

**Lưu ý:** `checkout_passcode` condition không có operator hay extra config — chỉ check attribute.  
Khi condition này = false (chưa nhập passcode) → Payment Function ẩn payment methods.  
Khi condition này = true → Payment Function không ẩn.

---

### 3C. Backend: `login-api-v2`

#### 3C-1. Thêm `checkout_passcode` vào PaymentRules entity

**File:** `src/modules/module/payment-rule/entities/payment-rules.entity.ts`

Thêm column:
```ts
@Column({ type: 'varchar', nullable: true })
checkoutPasscode: string | null;
// Lưu passcode raw (hoặc '*bss*' separated cho multiple)
// KHÔNG đưa vào metafield gửi lên Shopify
```

**Migration cần tạo:** `AddCheckoutPasscodeToPaymentRules`

#### 3C-2. Update `PaymentCustomizationDto`

Thêm optional field `checkoutPasscode?: string` vào DTO.

Khi service build metafield config để gửi lên Shopify → **KHÔNG include** `checkoutPasscode` value trong metafield JSON.  
Condition type `checkout_passcode` vào metafield nhưng không có passcode value:
```json
{
  "type": "checkout_passcode",
  "id": "some-id"
}
```

#### 3C-3. App Proxy endpoint mới

**File:** `src/modules/shopify/app_proxy/app_proxy.controller.ts`

```ts
@Public()
@Post('/checkout-passcode/verify')
@HttpCode(HttpStatus.OK)
@UseGuards(VerifyAndValidateAppProxyGuard)
async verifyCheckoutPasscode(
  @ReqContext() ctx: ExRequest,
  @Body() body: { passcode: string; ruleId: number }
) {
  return this.appProxyService.verifyCheckoutPasscode(ctx.shop, body);
}
```

**File:** `src/modules/shopify/app_proxy/app_proxy.service.ts`

```ts
async verifyCheckoutPasscode(shop: ShopDTO, data: { passcode: string; ruleId: number }) {
  const { passcode, ruleId } = data;

  const rule = await this.paymentRulesRepository.findOne({
    where: { id: ruleId, shop: { id: shop.id } },
    relations: ['shop']
  });

  if (!rule || !rule.checkoutPasscode) {
    return { success: false, message: 'Rule not found' };
  }

  const listPasscodes = rule.checkoutPasscode.split('*bss*');
  const isValid = listPasscodes.some(p => p.trim() === passcode.trim());

  return { success: isValid, message: isValid ? 'Verified' : 'Invalid passcode' };
}
```

**Dependency:** Inject `PaymentRulesRepository` vào `AppProxyService` (hoặc tách thành `CheckoutPasscodeService`).

**Route public:** Đã có `@Public()` + `@UseGuards(VerifyAndValidateAppProxyGuard)` pattern (xem `app_proxy.controller.ts`).

#### 3C-4. Update `payment-rule.service.ts`

Khi `createPaymentCustomization` hoặc `updateCustomization`:
- Lưu `body.checkoutPasscode` vào `rule.checkoutPasscode` trong DB
- Build metafield config: conditions có `checkout_passcode` type nhưng **không có giá trị passcode**
- Sau khi lưu rule và push metafield, nếu có `checkout_passcode` condition → update metafield của Checkout UI Extension với `{ ruleId, shopDomain }` để Extension biết gọi API nào

#### 3C-5. Metafield update cho Checkout UI Extension config

Khi merchant lưu payment rule có `checkout_passcode` condition, service cần:
```ts
// Set metafield cho Extension config
await shopifyMetafieldService.setAppOwnedMetafield({
  namespace: 'checkout-passcode',
  key: 'config',
  type: 'json',
  value: JSON.stringify({ ruleId: rule.id, shopDomain: shop.domain })
}, shop);
```

---

### 3D. CMS: `login-cms`

**Thay đổi UI trong Payment Customization condition builder:**

Thêm option mới vào danh sách condition type: **"Checkout Passcode"**

Khi chọn `checkout_passcode`:
- Hiện input field: **"Passcode(s)"** — nhập passcode, có thể multiple cách nhau bằng dấu phẩy hoặc theo format hiện có
- Không cần operator dropdown (condition này luôn là "customer phải nhập đúng passcode")
- Thêm helper text: *"Payment methods will be hidden until customer enters the correct passcode at checkout"*

UI sẽ render Checkout UI Extension config khi save — không cần thêm UI riêng.

---

## 4. Sequence cài đặt (thứ tự thực hiện)

```
1. [Extension] Tạo checkout-passcode-ui extension skeleton
2. [Backend]   Migration: AddCheckoutPasscodeToPaymentRules
3. [Backend]   Update PaymentRules entity + DTO
4. [Backend]   App Proxy: POST /checkout-passcode/verify
5. [Backend]   payment-rule.service.ts: lưu checkoutPasscode + build metafield đúng
6. [Extension] Update payment-customization: thêm CHECKOUT_PASSCODE condition + cart.attributes query
7. [Extension] Implement PasscodeForm UI (gọi App Proxy, applyAttributeChange)
8. [CMS]       Thêm condition type "Checkout Passcode" vào UI
9. [Test]      E2E: checkout với passcode đúng/sai
```

---

## 5. Edge cases & Cần xử lý

| Case | Xử lý |
|------|-------|
| Customer refresh trang sau khi verify | Cart attribute persist qua session — vẫn verified ✅ |
| Customer mở new tab | Cart attribute là per-checkout, mất nếu checkout mới 🔄 cần nhập lại |
| Nhiều payment rules có checkout_passcode | Chỉ 1 rule nên active tại 1 thời điểm (hoặc handle multiple ruleId) |
| Passcode sai 3 lần | Optional: thêm rate limiting ở App Proxy endpoint |
| `_bss_passcode_verified` bị set thủ công | Acceptable — UI restriction, không phải payment block |
| Checkout UI Extension không load | Payment methods vẫn ẩn → customer stuck. Cần fallback message |

---

## 6. Files cần sửa/tạo (checklist)

### `b2b-login-access-management-script`
- [ ] `extensions/checkout-passcode-ui/` — **TẠO MỚI** toàn bộ
- [ ] `extensions/payment-customization/src/const/common.const.ts` — thêm `CHECKOUT_PASSCODE`
- [ ] `extensions/payment-customization/src/cart_payment_methods_transform_run.graphql` — thêm `cart.attributes`
- [ ] `extensions/payment-customization/src/rule.ts` — thêm case `checkout_passcode`
- [ ] `extensions/payment-customization/generated/api.ts` — regenerate sau khi sửa graphql

### `login-api-v2`
- [ ] `src/modules/module/payment-rule/entities/payment-rules.entity.ts` — thêm `checkoutPasscode`
- [ ] `src/modules/module/payment-rule/dto/payment-rule.dto.ts` — thêm `checkoutPasscode?`
- [ ] `src/modules/module/payment-rule/payment-rule.service.ts` — handle checkout_passcode condition
- [ ] `src/modules/shopify/app_proxy/app_proxy.controller.ts` — thêm POST endpoint
- [ ] `src/modules/shopify/app_proxy/app_proxy.service.ts` — thêm verify logic
- [ ] `src/modules/shopify/app_proxy/app_proxy.module.ts` — inject PaymentRules repo
- [ ] `src/common/types/payment-customization.type.ts` — update ConditionType nếu cần
- [ ] `migrations/` — **TẠO** AddCheckoutPasscodeToPaymentRules migration

### `login-cms`
- [ ] Payment Customization condition UI — thêm `checkout_passcode` option + passcode input

---

## 7. Constraints & Shopify Limitations

- **Payment Function không verify crypto** — chỉ đọc cart attribute value, không verify token
- **Cart attribute bị remove** nếu cart bị abandoned và restored (handle tại UI Extension — hiện lại form)
- **Checkout UI Extension** cần `purchase.checkout.payment-method-list.render-before` target để render trước payment methods
- **App Proxy** URL format: `/a/bss-login/app-proxy/checkout-passcode/verify` — phải match với `shopify.app.toml` proxy config
- **Re-render trigger:** `applyAttributeChange` sẽ trigger Shopify để re-evaluate Functions — đây là cơ chế chính của solution này
