# os-agent-kit — Gemini CLI Project Context

Repo này là bộ agent config được extract từ dự án **B2B Login to Access**, dùng để bootstrap AI agent cho project mới trong team BSS.

---

## Đọc theo thứ tự này trước khi làm bất cứ việc gì

1. File này (GEMINI.md) — entry point cho Gemini CLI
2. `docs/README.md` — index toàn bộ business & technical docs
3. `docs/technical/be/` — technical docs cho backend features
4. `TEAM_STATUS.md` — ai đang làm gì, branch nào

---

## Agents

Agent files cho Gemini CLI nằm trong `.gemini/agents/`. Đọc file tương ứng và dùng nội dung làm system prompt khi cần:

| Agent | File | Khi nào dùng |
|-------|------|-------------|
| `coder` | `.gemini/agents/coder.md` | Viết code, follow coding conventions |
| `planner` | `.gemini/agents/planner.md` | Lập kế hoạch implementation trước khi code |
| `reviewer` | `.gemini/agents/reviewer.md` | Review code hoặc MR |

---

## 1. Overview

B2B Lock giúp merchant kiểm soát truy cập nội dung (sản phẩm, trang, giá, section) theo điều kiện khách hàng (tag, email, passcode, v.v.).  
Stack: NestJS · Liquid · TypeORM · MySQL · Redis · Shopify GraphQL/REST API · Shopify Extensions (Checkout UI, Functions)

---

## 2. Multi-service Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         Shopify Store                                    │
│   Theme Assets (Liquid/JS/CSS) + Checkout Extensions + Functions         │
└────────────────────────┬──────────────────────────────┬──────────────────┘
                         │ inject/upload                │ Function calls
          ┌──────────────▼──────────────┐   ┌───────────▼───────────────┐
          │       login-api-v2          │   │ b2b-login-access-         │
          │  NestJS · port 3351         │   │ management-script         │
          │  Main backend API           │   │ Shopify Extensions        │
          │  - Access rules             │   │ - cart-checkout-validation│
          │  - Theme upload             │   │ - payment-customization   │
          │  - Passcode                 │   │ - delivery-customization  │
          │  - Hide section/block       │   │ - validation-settings     │
          │  - Payment/Shipping rules   │   │ - theme-app-extension     │
          └──────────────┬──────────────┘   └───────────────────────────┘
                         │
          ┌──────────────▼──────────────┐
          │  login-cms (Admin UI)       │
          │  Express + React · port CMS │
          │  Shopify App Bridge         │
          │  Shopify Polaris            │
          │  Merchant-facing dashboard  │
          └──────────────┬──────────────┘
                         │
          ┌──────────────▼──────────────┐
          │  login-api (v1, legacy)     │
          │  Node.js · Giữ nguyên       │
          │  Không thêm logic mới       │
          └─────────────────────────────┘

Infrastructure: MySQL · Redis · Sentry · Mattermost Bot · Brevo Email
```

**Quy tắc giữa services:**
- Logic mới → `login-api-v2`, KHÔNG sửa `login-api`
- CMS chỉ gọi `login-api-v2` qua REST, xác thực qua `session-token` header
- Extensions gọi Shopify Functions, KHÔNG gọi API nội bộ trực tiếp

---

## 3. login-api-v2 Module Map

```
src/modules/
├── rules/             ← CRUD access rules, entity Rules
├── upload/            ← ⚡ Core: build + upload Liquid/JS/CSS lên Shopify theme
│   ├── upload.service.ts        ← Entry point chính
│   ├── hide-content-page/
│   │   ├── hide-section.service.ts
│   │   ├── hide-price.service.ts
│   │   ├── hide-atc.service.ts
│   │   ├── hide-variant.service.ts
│   │   └── hide-product-collection-navbar.service.ts
│   └── lock-element/
│       └── hide-product-collection.service.ts (HPCService)
├── module/
│   ├── sntap/         ← Hide Section/Block (target_type: 11)
│   ├── passcode/      ← Passcode access feature
│   ├── payment-rule/  ← Payment customization rules
│   ├── shipping-rule/ ← Shipping/delivery rules
│   └── templateEmail/ ← Email passcode templates
├── shopify/
│   ├── theme/         ← ThemeService: get/put theme files (Assets REST API)
│   ├── admin/         ← Admin-level Shopify operations
│   ├── customer/      ← Customer data from Shopify
│   ├── metafield/     ← Metafields & Metaobjects
│   ├── app_proxy/     ← App Proxy endpoints (storefront)
│   └── webhook/       ← Shopify Webhooks handler
├── internal/
│   ├── shop/          ← ShopService: getByDomain, lưu shop info
│   ├── config/        ← Config entity (theme_id, settings)
│   ├── design/        ← DesignService: MessageDesign, PasscodeDesign
│   ├── onboarding/    ← Onboarding flow
│   ├── analytics/     ← Internal analytics
│   ├── behavior/      ← Behavior tracking
│   └── theme-installations/ ← Theme install tracking
├── external/
│   ├── google/        ← Google OAuth (SSO login)
│   ├── brevo/         ← Transactional email (Brevo/Sendinblue)
│   ├── email/         ← Email gửi passcode
│   ├── geolocation/   ← IP geolocation
│   └── support/       ← Support tickets
└── auth/
    └── guards/
        └── shop.guard.ts ← Global guard: validate session-token JWT
```

---

## 4. Feature → Flow chính

### 4a. Tạo/Cập nhật Access Rule
```
CMS → POST /rules → ShopGuard (validate JWT) → RuleController
  → RuleService.saveOrUpdate() → DB (rules-v2 table)
  → UploadService.generateAndUpload() → Render Liquid/JS/CSS
  → ThemeService.putAsset() → Shopify Theme REST API
```

### 4b. Hide Section/Block (target_type = 11)
```
CMS → POST /rules (target_type: 11) → RuleService → DB
  → sntap.service / hide-section.service
  → build Liquid snippet / CSS selector / JS
  → ThemeService upload
```
> Đọc `docs/technical/be/hide-section.md` trước khi sửa feature này

### 4c. Passcode Access
```
Storefront → App Proxy → POST /passcode/verify → PasscodeService
  → check code + customer tag → grant/deny access
```

### 4d. Payment/Shipping Customization
```
Shopify Checkout → Extension Function call
  → Reads Metafields set bởi payment-rule / shipping-rule modules
```

### 4e. Auth Flow (mọi API request)
```
Request header: session-token: <Shopify JWT>
  → ShopGuard.canActivate()
  → jwtService.verify(token, SHOPIFY_API_SECRET_KEY)
  → decode dest = shop domain
  → ShopService.getByDomain(domain) → attach shop to request ctx
Webhook route: dùng x-shopify-shop-domain header thay thế
```

---

## 5. Key Entities (database)

| Entity | Table | Mô tả |
|--------|-------|-------|
| `Rules` | `rules-v2` | Access rule, gồm conditions, keys, design |
| `Shop` | `shops` | Info shop Shopify |
| `Config` | `configs` | Cấu hình per-shop (theme_id, settings) |
| `MessageDesign` | - | Design login message |
| `PasscodeDesign` | - | Design passcode form |
| `PasscodeRequests` | - | Lịch sử passcode |
| `PaymentRules` | - | Payment customization rules |
| `Translation` | - | Bản dịch message per rule |

---

## 6. Auth & Request Context

```typescript
// Session-token JWT payload contains:
{ dest: "https://my-shop.myshopify.com" }

// Extracted by ShopGuard → attached to request:
request.shop: ShopDTO  // dùng qua @ReqContext() ctx: ExRequest

// Các path PUBLIC (skip guard):
'/passcode/save-usage-history'
'/behavior'
'/upload-content/banner-trail-expried'
'/upload-content/update-shop-trail'
'/upload-content/subscription'
'/webhooks/**'
```

---

## 7. Environment Variables (login-api-v2)

| Var | Mô tả |
|-----|-------|
| `DB_URL` | MySQL connection string |
| `REDIS_URL` | Redis connection string |
| `SECRET_KEY` | HMAC secret |
| `SHOPIFY_API_SECRET_KEY` | Verify JWT session-token |
| `JWT_SECRET_SERVER_V2` | Internal JWT |
| `SENTRY_DSN` | Error tracking |
| `APP_ID` | Shopify App ID |
| `API_VERSION` | Shopify API version (e.g. 2024-07) |
| `SHOPIFY_CART_CHECKOUT_VALIDATION_ID` | Extension ID |
| `SHOPIFY_PAYMENT_CUSTOMIZATION_ID` | Extension ID |

---

## 8. Dev Commands

```bash
# login-api-v2
cd login-api-v2
pnpm dev                          # Start dev (port 3351)
pnpm build                        # Build
pnpm test                         # Unit tests

# Migration
npm run migration:create --name=<fileName>
npm run migration:generate --name=<fileName>
npm run migration:run
npm run migration:revert

# login-cms
cd login-cms/web
# xem shopify.web.toml để biết lệnh dev

# Extensions
cd b2b-login-access-management-script
pnpm dev                          # Shopify app dev với extensions
```

---

## 9. Hard Rules — KHÔNG ĐƯỢC VI PHẠM

1. **KHÔNG thêm business logic vào `login-api` (v1)**  
   → Tất cả logic mới vào `login-api-v2`

2. **KHÔNG sửa `upload.service.ts` mà không hiểu flow upload**  
   → Service này build toàn bộ Liquid/JS/CSS và upload lên theme, đụng vào sẽ break cho tất cả shops

3. **KHÔNG trực tiếp gọi Shopify REST trong Controller**  
   → Dùng `ThemeService`, `Rest`/`GraphQL` từ `@bss-sbc/shopify-api-fetcher`

4. **Khi sửa feature Hide Section/Block**  
   → Đọc `docs/technical/be/hide-section.md` trước

5. **Mọi entity mới đều cần migration**  
   → Chạy `migration:generate` sau khi sửa entity, commit file migration cùng code

6. **Auth guard là global** — muốn route public thì dùng `@Public()` decorator hoặc thêm vào `skipShopGuard` list trong `shop.guard.ts`

---

## 10. Reference Docs trong Workspace

| File | Dùng khi |
|------|----------|
| `docs/technical/be/hide-section.md` | Sửa hide section/block feature |
| `docs/technical/be/upload-content.md` | Sửa upload/theme flow |
| `docs/technical/be/rules.md` | Sửa access rules |
| `docs/plans/` | Xem feature plans đã viết sẵn |
| `docs/business/` | Hiểu business logic và user-facing features |

---

## 11. Docs Workflow

- Khi AI làm sai feature → tạo `docs/plans/[feature].md` mô tả lại đúng flow
- Trước khi làm feature phức tạp → viết `docs/plans/[feature].md` như bản spec trước khi code
- Sau khi implement xong → update doc nếu flow thay đổi so với spec ban đầu

---

## 12. Multi-dev Protocol

- KHÔNG sử dụng agent dev khi feature-context.md chưa được Dev Lead approve
- LUÔN đọc feature-context.md của feature hiện tại trước khi bắt đầu code
- KHÔNG sửa file ngoài danh sách "Files sẽ bị ảnh hưởng" trong feature-context.md
- Cập nhật TEAM_STATUS.md mỗi khi bắt đầu hoặc kết thúc session làm việc

---

## 13. Gemini CLI Tools

| Mục đích | Tool |
|----------|------|
| Đọc file | `read_file` |
| Tạo file mới | `write_file` |
| Sửa nội dung file | `replace` |
| Chạy shell command | `run_shell_command` |
| Tìm kiếm trong file | `grep_search` |
| Tìm file theo pattern | `glob` |
| Liệt kê thư mục | `list_directory` |
| Track tasks | `write_todos` |
| Lưu memory | `save_memory` |
