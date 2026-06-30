# Shopify webhooks

## Endpoints
Controller: `src/modules/shopify/webhook/webhook.controller.ts` (`@Controller('webhooks')`)

- `POST /webhooks/uninstall` → `WebhookService.handleUninstallApp(shop)`
- `POST /webhooks/themes/publish` → `WebhookService.handleThemesPublish(shop)`
- `POST /webhooks/shop-update` → placeholder
- `POST /webhooks/collections/delete` → placeholder

## Auth / shop context
Global `ShopGuard` có nhánh special-case cho webhook path:
- Nếu `request.path.includes('/webhooks')` → lấy domain từ header `x-shopify-shop-domain`
- Sau đó load shop từ DB và gắn vào `request.shop`

Lưu ý kỹ thuật:
- App boot với `rawBody: true` (`src/main.ts`) để hỗ trợ các luồng cần raw body (thường dùng cho verify signature). Nếu bạn muốn verify HMAC webhook của Shopify, cần implement thêm (hiện trong controller/service chưa thấy verify).

## Uninstall flow
Service: `src/modules/shopify/webhook/webhook.service.ts`

`handleUninstallApp(shop)` (đối với `client_version == 'v2'`):
- Track uninstall (Mixpanel)
- Reset onboarding
- Reset `shop.is_first_save_rule`
- Cleanup các rule/customization liên quan:
  - `CheckoutValidationService.deleteAllRule(shop)`
  - `PaymentRuleService.deletePaymentCustomizationDB(shop)`
  - `ShippingRuleService.deleteAllShippingCustomization(shop)`

## Theme publish flow
`handleThemesPublish(shop)`:
- Lấy theme live id hiện tại (`ThemeService.getThemeId(..., ignore=true)`)
- Đọc config upload target
- Nếu publish làm thay đổi target theme:
  - Update config theme_id/upload_target/upload_timestamp
  - Update theme installation record
  - Trigger reinstall: `UploadService.uploadContent(shop, curThemeLiveId)`
