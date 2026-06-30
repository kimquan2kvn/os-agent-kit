# Shopify App Proxy (/app_proxy)

## Endpoint
Controller: `src/modules/shopify/app_proxy/app_proxy.controller.ts`

- `GET /app_proxy/ping`
  - `@Public()` nhưng bắt buộc qua guard `VerifyAndValidateAppProxyGuard`
  - Query param: `locking-conditions` (chuỗi condition ids, thường dạng `"1,2,3,"` → code sẽ split và bỏ phần tử cuối)

## Signature verification (guard)
Guard: `src/modules/auth/guards/appProxy.guard.ts`

- Lấy `signature` từ query, phần còn lại là payload
- Tạo `stringPayload`:
  - Map `key=value` (array → join bằng `,`)
  - Sort
  - Join thành 1 string
- Verify:
  - `HMAC_SHA256(stringPayload, SHOPIFY_API_SECRET_KEY)` phải == `signature`
- Nếu pass:
  - Load shop theo `payload.shop`
  - Gắn `request.shop`

## Logic check conditions
Service: `src/modules/shopify/app_proxy/app_proxy.service.ts`

`checkConditionByIp(lockingConditions, currentIp, ipGeo, shop)`:
- Load tất cả rules status=1 của shop (join keys/conditions)
- Hiện implement 2 condition types:
  - `specific_ip`:
    - check string `*bss*<ip>*bss*` nằm trong `cond.configs.specific_ip`
    - hỗ trợ `cond.inverse` để đảo điều kiện
  - `regions`:
    - build string `<regionCode>/<countryCode>` và check với `ipGeo`
    - `ipGeo` format lấy từ `@ReqContext()`: `${cf-region-code}/${cf-ipcountry}`
- Response trả về list condition ids đã “unlock” theo ip/region.
