# Mixpanel Sub/Unsub Tracking Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Capture Mixpanel events chính xác khi shop v2 subscribe hoặc unsubscribe khỏi app B2B Lock.

**Architecture:**
- Subscribe tracking: thêm vào endpoint `POST /upload-content/subscription` — endpoint này đã được v1 gọi sau khi shop subscribe thành công, `ctx.shop` đã có plan mới.
- Unsubscribe tracking: thêm webhook handler `POST /webhooks/app-subscriptions/update` trong v2 để bắt sự kiện Shopify `app_subscriptions/update` khi subscription bị CANCELLED.
- Không sửa `login-api` (v1).

**Tech Stack:** NestJS, Mixpanel Node SDK, TypeScript, Shopify Webhook

---

## File Map

| Action | File |
|--------|------|
| Modify | `login-api-v2/src/shared/mixpanel/mixpanel.service.ts` |
| Modify | `login-api-v2/src/modules/upload/upload.module.ts` |
| Modify | `login-api-v2/src/modules/upload/upload.controller.ts` |
| Modify | `login-api-v2/src/modules/shopify/webhook/webhook.service.ts` |
| Modify | `login-api-v2/src/modules/shopify/webhook/webhook.controller.ts` |
| Modify | `shopify.app.toml` |

---

## Task 1: Thêm method `setTrackAfterSubscribed` và `setTrackAfterUnsubscribed` vào MixpanelService

**Files:**
- Modify: `login-api-v2/src/shared/mixpanel/mixpanel.service.ts`

- [ ] **Step 1: Đọc file hiện tại để nắm context**

File hiện tại chỉ có `setTrackAfterUninstalled`. Cần thêm 2 method mới.

- [ ] **Step 2: Thêm type và method mới vào MixpanelService**

Thay thế nội dung `mixpanel.service.ts` thành:

```typescript
import { mappingAppPlan } from './../../utils/mappingAppPlan';
import { Injectable } from '@nestjs/common';
import { LoggerService } from '@shared/logger/logger.service';
import * as Mixpanel from 'mixpanel';

interface mixPanelPayloadType {
    shopDomain: string;
    email: string;
    plan_code: string;
    current_sub_plan: string;
    interval_subscriptions?: string;
}

@Injectable()
export class MixpanelService {
    private mixpanel: any;

    constructor(private readonly logger: LoggerService) {
        this.mixpanel = Mixpanel.init('85b0b254995dd4e1d7fa52caa9643851');
        this.logger.setContext(MixpanelService.name);
    }

    setTrackAfterUninstalled = function (payload: mixPanelPayloadType) {
        if (payload?.email?.indexOf('bsscommerce.com') === -1) {
            try {
                const domain = payload.shopDomain;
                const now = new Date().toISOString();
                this.mixpanel.people.set(domain, {
                    'Last Uninstalled At': now,
                });
                this.mixpanel.people.set_once(domain, {
                    'Uninstalled times': now,
                });
                this.mixpanel.track('Uninstalled', {
                    distinct_id: domain,
                    'Plan Interval': payload.interval_subscriptions ?? '',
                    'App Plan': mappingAppPlan(payload.plan_code, payload.current_sub_plan)
                });
            } catch (error) {
                this.logger.error(error);
            }
        }
    };

    setTrackAfterSubscribed = function (payload: mixPanelPayloadType) {
        if (payload?.email?.indexOf('bsscommerce.com') === -1) {
            try {
                const domain = payload.shopDomain;
                const now = new Date().toISOString();
                this.mixpanel.people.set(domain, {
                    'Last Subscribed At': now,
                    'App Plan': mappingAppPlan(payload.plan_code, payload.current_sub_plan),
                    'Plan Interval': payload.interval_subscriptions ?? '',
                });
                this.mixpanel.track('Subscribed', {
                    distinct_id: domain,
                    'Plan Interval': payload.interval_subscriptions ?? '',
                    'App Plan': mappingAppPlan(payload.plan_code, payload.current_sub_plan),
                });
            } catch (error) {
                this.logger.error(error);
            }
        }
    };

    setTrackAfterUnsubscribed = function (payload: mixPanelPayloadType) {
        if (payload?.email?.indexOf('bsscommerce.com') === -1) {
            try {
                const domain = payload.shopDomain;
                const now = new Date().toISOString();
                this.mixpanel.people.set(domain, {
                    'Last Unsubscribed At': now,
                });
                this.mixpanel.track('Unsubscribed', {
                    distinct_id: domain,
                    'Plan Interval': payload.interval_subscriptions ?? '',
                    'App Plan': mappingAppPlan(payload.plan_code, payload.current_sub_plan),
                });
            } catch (error) {
                this.logger.error(error);
            }
        }
    };
}
```

- [ ] **Step 3: Verify TypeScript compiles**

```bash
cd login-api-v2
pnpm build 2>&1 | grep -E "error|Error" | head -20
```

Expected: No errors related to `mixpanel.service.ts`.

- [ ] **Step 4: Commit**

```bash
git add login-api-v2/src/shared/mixpanel/mixpanel.service.ts
git commit -m "feat(mixpanel): add setTrackAfterSubscribed and setTrackAfterUnsubscribed methods"
```

---

## Task 2: Inject MixpanelService vào UploadModule

**Files:**
- Modify: `login-api-v2/src/modules/upload/upload.module.ts`

Hiện tại `UploadModule` không import `MixpanelModule`, cần thêm để controller có thể dùng `MixpanelService`.

- [ ] **Step 1: Thêm MixpanelModule vào imports và MixpanelService vào providers**

Edit `upload.module.ts`, thêm vào `imports` array:
```typescript
import { MixpanelModule } from '@shared/mixpanel/mixpanel.module';
```

Thêm `MixpanelModule` vào `imports`:
```typescript
imports: [
    TypeOrmModule.forFeature([...]),
    forwardRef(() => GoogleModule),
    forwardRef(() => ShopModule),
    forwardRef(() => RuleModule),
    AdminModule,
    MetafieldsModule,
    forwardRef(() => ThemeInstallationModule),
    ConfigModule,
    MixpanelModule,   // <-- thêm vào đây
],
```

Thêm `MixpanelService` vào `providers`:
```typescript
providers: [
    UploadService,
    ConfigService,
    ThemeService,
    HPCService,
    HidePriceOnGoogleService,
    JwtService,
    HideATCService,
    HideProductCollectionNavbarService,
    HideSectionService,
    // MixpanelService đã được export bởi MixpanelModule, không cần thêm trực tiếp
],
```

> Note: `MixpanelModule` đã `exports: [MixpanelService]` (xem `mixpanel.module.ts`), nên chỉ cần import module là đủ.

- [ ] **Step 2: Verify build**

```bash
cd login-api-v2
pnpm build 2>&1 | grep -E "error|Error" | head -20
```

Expected: No new errors.

- [ ] **Step 3: Commit**

```bash
git add login-api-v2/src/modules/upload/upload.module.ts
git commit -m "feat(upload): add MixpanelModule to UploadModule for subscription tracking"
```

---

## Task 3: Thêm Mixpanel tracking vào endpoint `/upload-content/subscription`

**Files:**
- Modify: `login-api-v2/src/modules/upload/upload.controller.ts`

Endpoint này được v1 gọi sau khi shop subscribe thành công. `ctx.shop` lúc này đã có `plan_code` và `interval_subscriptions` mới được cập nhật từ v1.

- [ ] **Step 1: Inject MixpanelService vào UploadController**

Trong constructor của `UploadController`, thêm `MixpanelService`:

```typescript
import { MixpanelService } from '@shared/mixpanel/mixpanel.service';

// Trong constructor:
constructor(
    private readonly uploadService: UploadService,
    private readonly logger: LoggerService,
    private readonly themeService: ThemeService,
    private readonly themeInstallation: ThemeInstallationService,
    private readonly mixpanelService: MixpanelService,
) {
    this.logger.setContext(UploadController.name);
}
```

- [ ] **Step 2: Thêm Mixpanel track call vào `uploadContentAfterSubscription`**

Thêm gọi `setTrackAfterSubscribed` sau khi upload thành công:

```typescript
@Post('/subscription')
@UseGuards(VerifyServerLoginV1Guard)
async uploadContentAfterSubscription(@ReqContext() ctx: ExRequest) {
    const shop = ctx.shop;
    const themeId = await this.themeService.getThemeId(
        shop.domain,
        shop.id,
        shop.token,
        true,
    );
    const doneUpload = await this.uploadService.uploadRulesContent(
        shop,
        themeId,
    );
    if (!doneUpload) {
        throw new BadRequestException('Upload content after subscription', {
            cause: {
                context: UploadController.name,
                name: 'Upload content after subscription',
            },
        });
    }
    this.mixpanelService.setTrackAfterSubscribed({
        shopDomain: shop.domain,
        email: shop.customer_email,
        plan_code: shop.plan_code,
        current_sub_plan: shop.current_sub_plan,
        interval_subscriptions: shop.interval_subscriptions,
    });
    return doneUpload;
}
```

- [ ] **Step 3: Verify build**

```bash
cd login-api-v2
pnpm build 2>&1 | grep -E "error|Error" | head -20
```

Expected: No errors.

- [ ] **Step 4: Commit**

```bash
git add login-api-v2/src/modules/upload/upload.controller.ts
git commit -m "feat(upload): track Mixpanel Subscribed event after subscription upload"
```

---

## Task 4: Thêm webhook handler cho `app_subscriptions/update` trong WebhookService

**Files:**
- Modify: `login-api-v2/src/modules/shopify/webhook/webhook.service.ts`

Shopify gửi webhook `app_subscriptions/update` khi subscription status thay đổi. Payload chứa `status`:
- `"CANCELLED"` → merchant hủy subscription → track Unsubscribed
- `"ACTIVE"` → subscription được kích hoạt (không cần track vì đã track tại subscribe endpoint)

- [ ] **Step 1: Thêm `handleAppSubscriptionsUpdate` method vào WebhookService**

Thêm method sau vào `webhook.service.ts`:

```typescript
async handleAppSubscriptionsUpdate(shop: ShopDTO, body: any) {
    const status = body?.app_subscription?.status ?? body?.status;
    if (status === 'CANCELLED' && shop?.client_version === 'v2') {
        this.mixpanelService.setTrackAfterUnsubscribed({
            shopDomain: shop.domain,
            email: shop.customer_email,
            plan_code: shop.plan_code,
            current_sub_plan: shop.current_sub_plan,
            interval_subscriptions: shop.interval_subscriptions,
        });
    }
}
```

> **Lý do check `status === 'CANCELLED'`:** Chỉ track khi merchant hủy subscription, không track các status khác (EXPIRED, FROZEN, PENDING). `client_version === 'v2'` vì v1 shops dùng Mixpanel riêng trong v1 code.

> **Lý do lấy `plan_code` từ shop DB tại thời điểm webhook đến:** Webhook đến TRƯỚC khi v1 cập nhật DB về `free`. Đây là thời điểm chính xác nhất để lấy plan đang hủy.

- [ ] **Step 2: Verify method signature khớp với ShopDTO**

Kiểm tra `ShopDTO` (trong `src/common/dto/shop.dto.ts`) có các fields:
- `customer_email` ✓ (line 100)
- `plan_code` ✓ (line 64)
- `current_sub_plan` ✓ (line 103)
- `interval_subscriptions` ✓ (line 57)
- `client_version` — cần verify field name

```bash
grep -n "client_version\|customer_email\|plan_code\|current_sub_plan\|interval_subscriptions" \
  login-api-v2/src/common/dto/shop.dto.ts
```

- [ ] **Step 3: Verify build**

```bash
cd login-api-v2
pnpm build 2>&1 | grep -E "error|Error" | head -20
```

- [ ] **Step 4: Commit**

```bash
git add login-api-v2/src/modules/shopify/webhook/webhook.service.ts
git commit -m "feat(webhook): add handleAppSubscriptionsUpdate for Mixpanel Unsubscribed tracking"
```

---

## Task 5: Thêm endpoint `POST /webhooks/app-subscriptions/update` vào WebhookController

**Files:**
- Modify: `login-api-v2/src/modules/shopify/webhook/webhook.controller.ts`

Shopify webhook dùng `x-shopify-shop-domain` header để identify shop (đã được `ShopGuard` handle).

- [ ] **Step 1: Thêm endpoint mới vào WebhookController**

Thêm vào `webhook.controller.ts`:

```typescript
@Post('/app-subscriptions/update')
async handleAppSubscriptionsUpdate(@ReqContext() ctx: any, @Body() body: any) {
    await this.webhookService.handleAppSubscriptionsUpdate(ctx.shop, body);
    return true;
}
```

> **Note:** Import `Body` từ `@nestjs/common` nếu chưa có. Shopify gửi raw JSON body, NestJS tự parse thành object.

Full updated controller:

```typescript
import { BadRequestException, Body, Controller, Post } from '@nestjs/common';
import { ReqContext } from '@common/decorators/request-context.decorator';
import { LoggerService } from '@shared/logger/logger.service';
import { WebhookService } from './webhook.service';

@Controller('webhooks')
export class WebhookController {
    constructor(
        private readonly webhookService: WebhookService,
        private readonly logger: LoggerService,
    ) {
        this.logger.setContext(WebhookController.name);
    }

    @Post('/uninstall')
    async handleUninstallApp(@ReqContext() ctx: any) {
        const uninstall = this.webhookService.handleUninstallApp(ctx.shop);
        if (!uninstall) {
            throw new BadRequestException('Handle uninstall failed', {
                cause: {
                    context: WebhookController.name,
                    name: 'Handle uninstall',
                },
            });
        }
        return uninstall;
    }

    @Post('/themes/publish')
    async handleThemesPublish(@ReqContext() ctx: any) {
        const publish = this.webhookService.handleThemesPublish(ctx.shop);
        if (!publish) {
            throw new BadRequestException('Handle uninstall failed', {
                cause: {
                    context: WebhookController.name,
                    name: 'Handle uninstall',
                },
            });
        }
        return publish;
    }

    @Post('/shop-update')
    async handleShopUpdate() {
        const productList = this.webhookService.handleShopUpdate();
        return productList;
    }

    @Post('/collections/delete')
    async handleCollectionsDelete() {
        return true;
    }

    @Post('/app-subscriptions/update')
    async handleAppSubscriptionsUpdate(@ReqContext() ctx: any, @Body() body: any) {
        await this.webhookService.handleAppSubscriptionsUpdate(ctx.shop, body);
        return true;
    }
}
```

- [ ] **Step 2: Verify build**

```bash
cd login-api-v2
pnpm build 2>&1 | grep -E "error|Error" | head -20
```

Expected: Không có errors mới.

- [ ] **Step 3: Commit**

```bash
git add login-api-v2/src/modules/shopify/webhook/webhook.controller.ts
git commit -m "feat(webhook): add POST /webhooks/app-subscriptions/update endpoint"
```

---

## Task 6: Đăng ký webhook `app_subscriptions/update` trong shopify.app.toml

**Files:**
- Modify: `shopify.app.toml`

Shopify cần biết URL để gửi webhook này. URL phải point đến v2 API server của staging.

- [ ] **Step 1: Thêm webhook subscription mới vào `shopify.app.toml`**

Thêm vào section `[webhooks]` sau các webhooks hiện có:

```toml
  [[webhooks.subscriptions]]
  topics = [ "app_subscriptions/update" ]
  uri = "/webhooks/app-subscriptions/update"
```

> **Note:** Shopify cho phép dùng relative URI (không cần full URL) khi app đã có `application_url` được config. Shopify sẽ prepend `application_url` của API v2 server.

> **Lưu ý production:** File `shopify.app.toml` này là staging config. Production cần cập nhật riêng trong `shopify.app.toml` của production environment hoặc qua Shopify Partners Dashboard.

Kết quả `[webhooks]` section sau khi thêm:

```toml
[webhooks]
api_version = "2024-04"

  [[webhooks.subscriptions]]
  uri = "https://login-staging-api.nomad-test.test-bsscommerce.com/gdpr/customers_data_request"
  compliance_topics = [ "customers/data_request" ]

  [[webhooks.subscriptions]]
  uri = "https://login-staging-api.nomad-test.test-bsscommerce.com/gdpr/customers_redact"
  compliance_topics = [ "customers/redact" ]

  [[webhooks.subscriptions]]
  uri = "https://login-staging-api.nomad-test.test-bsscommerce.com/gdpr/shop_redact"
  compliance_topics = [ "shop/redact" ]

  [[webhooks.subscriptions]]
  topics = [ "app_subscriptions/update" ]
  uri = "https://login-staging-api-v2.nomad-test.test-bsscommerce.com/webhooks/app-subscriptions/update"
```

- [ ] **Step 2: Deploy webhook config lên Shopify**

```bash
cd /path/to/project-root   # thư mục chứa shopify.app.toml
shopify app deploy
```

Hoặc nếu dùng Shopify CLI:
```bash
shopify app config push
```

> **Note:** Cần confirm với team về cách deploy shopify.app.toml lên production vs staging riêng biệt.

- [ ] **Step 3: Commit**

```bash
git add shopify.app.toml
git commit -m "feat(shopify): register app_subscriptions/update webhook for Mixpanel tracking"
```

---

## Task 7: Verify integration end-to-end

- [ ] **Step 1: Start dev server**

```bash
cd login-api-v2
pnpm dev
```

- [ ] **Step 2: Test subscribe event**

Dùng một test shop, đi qua flow subscribe trên staging:
1. CMS → Billing → Chọn plan → Confirm
2. Shopify redirect về `/plan/update-charge-id` trên v1
3. v1 calls `POST /upload-content/subscription` trên v2
4. Check Mixpanel dashboard: tìm `distinct_id = <shop_domain>` và event `Subscribed`

```bash
# Hoặc test thủ công bằng cách call trực tiếp endpoint (với JWT hợp lệ):
curl -X POST https://login-staging-api-v2.nomad-test.test-bsscommerce.com/upload-content/subscription \
  -H "Authorization: Bearer <JWT_FROM_V1>" \
  -H "Content-Type: application/json"
```

Expected: Mixpanel nhận được event `Subscribed` với `distinct_id = shop.domain`.

- [ ] **Step 3: Test unsubscribe event**

1. Từ CMS → Billing → Cancel subscription
2. Shopify sends `app_subscriptions/update` webhook với `status: "CANCELLED"`
3. Check Mixpanel dashboard: event `Unsubscribed`

- [ ] **Step 4: Verify BSS shops bị filter**

Kiểm tra rằng shops có email `*@bsscommerce.com` KHÔNG tạo Mixpanel events (filter đã có trong service).

---

## Checklist Self-Review

### Spec coverage
- [x] Subscribe event → Task 3 (upload-content/subscription endpoint)
- [x] Unsubscribe event → Tasks 4-6 (webhook app_subscriptions/update)
- [x] Filter BSS internal shops → đã có trong `indexOf('bsscommerce.com') === -1`

### Placeholder scan
- Không có TBD/TODO trong các code blocks
- Tất cả method names nhất quán: `setTrackAfterSubscribed`, `setTrackAfterUnsubscribed`

### Type consistency
- `mixPanelPayloadType` dùng thống nhất trong cả 3 methods
- `ShopDTO` fields (`customer_email`, `plan_code`, `current_sub_plan`, `interval_subscriptions`) khớp giữa Task 3 và Task 4
