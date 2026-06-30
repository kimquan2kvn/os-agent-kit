# Location Conditions — IP / Regions / Countries Architecture & Quick Reference

## 1. Feature Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Shopify Store (Storefront)                     │
│                                                                          │
│  Theme render → {% render 'bss-lock-ip' %}                               │
│    → bss-lock-ip.liquid tính remain_lock_conditions                      │
│    → JS: fetch /apps/bss-b2b-lock/ping?locking-conditions=<ids>          │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │ App Proxy request
                               │ (query chứa Shopify HMAC signature)
               ┌───────────────▼────────────────┐
               │  AppProxyController             │
               │  @Get('/ping')                  │
               │  Guard: VerifyAndValidateAppProxy│
               │  Lấy IP từ x-forwarded-for      │
               │  Lấy ipGeo từ cf-region-code /  │
               │              cf-ipcountry        │
               └───────────────┬────────────────┘
                               │
               ┌───────────────▼────────────────┐
               │  AppProxyService                │
               │  checkConditionByIp()           │
               │  ├─ specific_ip  → passedIP     │
               │  ├─ regions      → passedRegion │
               │  └─ countries    → passedCountry│ ← MR !523 / LOGIN-378
               └───────────────┬────────────────┘
                               │ Response JSON:
                               │ { ipUnlockedConditions,
                               │   regionUnlockedConditions,
                               │   countriesUnlockedConditions,  ← MR !523
                               │   currentIp, ipGeo }
               ┌───────────────▼────────────────┐
               │  JS (bss-lock-ip.liquid)        │
               │  POST /cart/update.js           │
               │  Lưu cart.attributes:           │
               │  bss-lock-ip-escape-{id}-check  │
               │  bss-lock-region-escape-{id}-check│
               │  bss-lock-countries-escape-{id}-check│ ← MR !523
               └───────────────┬────────────────┘
                               │ window.location.reload()
               ┌───────────────▼────────────────┐
               │  bss-lock-condition.liquid      │
               │  (baked-in tại theme upload)    │
               │  Đọc cart.attributes            │
               │  → check IP/region/countries    │
               │  → set temp_<id> = true/false   │
               │  → access_denied = true/false   │
               └────────────────────────────────┘

CMS (Merchant cấu hình)
┌──────────────────────────────────────────┐
│  POST /geolocation/search                │
│  → GeolocationService.searchByKeyword()  │
│  → GeoNames API → trả về list regions   │
│  → Merchant chọn region/country          │
│  → Lưu vào condition.configs             │
└──────────────────────────────────────────┘
```

---

## 2. Condition Config Structure

Ba loại condition địa lý được lưu trong `keys.conditions` của rule:

### 2.1 `specific_ip` — IP Address cụ thể

```typescript
condition.type = 'specific_ip'
condition.configs = {
    specific_ip: '*bss*192.168.1.1*bss*10.0.0.1*bss*',
    // Nhiều IP cách nhau bằng '*bss*', bao quanh bởi '*bss*' ở đầu và cuối
}
condition.inverse = 0  // 0 = chỉ IP trong danh sách mới được phép
                       // 1 = chặn IP trong danh sách (whitelist ngược lại)
```

> **Format IP string**: Mỗi IP được bao bởi `*bss*` ở cả hai phía để tránh partial match.  
> Ví dụ `*bss*1.2.3.4*bss*` ngăn `1.2.3` match nhầm vào `1.2.3.4`.

### 2.2 `regions` — Region/State theo IP geolocation

```typescript
condition.type = 'regions'
condition.configs = {
    regions: [
        { regionCode: 'HN', countryCode: 'VN' },  // Hà Nội, Việt Nam
        { regionCode: 'CA', countryCode: 'US' },  // California, USA
    ]
}
condition.inverse = 0  // 0 = chỉ region trong danh sách được phép
                       // 1 = chặn region trong danh sách
```

> **Format**: `regionCode/countryCode` — lấy từ GeoNames API (`cf-region-code`/`cf-ipcountry` của Cloudflare).  
> `ipGeo` được ghép thành `HN/VN` từ Cloudflare headers trên mỗi request.

### 2.3 `countries` — Quốc gia (MR !523 / LOGIN-378)

```typescript
condition.type = 'countries'
condition.configs = {
    countries: ['VN', 'US', 'JP'],  // Mảng ISO 3166-1 alpha-2 country codes
}
condition.inverse = 0  // 0 = chỉ quốc gia trong danh sách được phép
                       // 1 = chặn quốc gia trong danh sách
```

> **Nguồn country**: Hiện tại (MR !523) lấy từ `ipinfo.io` phía client.  
> ⚠️ **Known issue**: Cần chuyển sang lấy server-side từ `cf-ipcountry` Cloudflare header để tránh client spoofing và rate limit.

---

## 3. API Endpoints

### 3.1 App Proxy — Runtime unlock check

Controller: `src/modules/shopify/app_proxy/app_proxy.controller.ts`

| Method | Path | Guard | Mô tả |
|--------|------|-------|-------|
| `GET` | `/app_proxy/ping` | `VerifyAndValidateAppProxyGuard` | Storefront gọi để check IP/region/countries conditions |

**Query Parameters**:

| Param | Bắt buộc | Mô tả |
|-------|----------|-------|
| `locking-conditions` | ✅ | Comma-separated condition IDs cần verify |
| `country` | ✅ (MR !523) | ISO country code (hiện lấy từ ipinfo.io) |
| `signature` | ✅ | HMAC signature do Shopify App Proxy thêm tự động |
| `shop` | ✅ | Shop domain (do Shopify App Proxy thêm) |

**Response**:
```json
{
  "ipUnlockedConditions": [12, 34],
  "currentIp": "1.2.3.4",
  "regionUnlockedConditions": [56],
  "ipGeo": "HN/VN",
  "countriesUnlockedConditions": [78]
}
```

### 3.2 Geolocation — CMS search region/country

Controller: `src/modules/external/geolocation/geolocation.controller.ts`

| Method | Path | Guard | Mô tả |
|--------|------|-------|-------|
| `POST` | `/geolocation/search` | ShopGuard | CMS dùng để tìm kiếm region theo keyword |

**Request Body**:
```json
{ "search": "Hanoi" }
```

**Response**:
```json
{
  "totalCount": 5,
  "geonames": [
    {
      "id": 1581130,
      "name": "Hanoi",
      "regionCode": "HN",
      "countryName": "Vietnam",
      "countryCode": "VN",
      "population": 1431270
    }
  ]
}
```

> **External API**: GeoNames — `GEO_SERVICE_HOST` + `GEO_SERVICE_USERNAME` trong `.env`

---

## 4. Data Flow — Runtime Unlock (Storefront)

```
Storefront page load
  │
  ▼
bss-lock-ip.liquid render
  1. Đọc Liquid variables: bss_locked_conditions_string, bss_unlocked_conditions_string
     (được generate bởi getLockIpAndRegionContent() vào bss-lock-condition.liquid)
  2. Tính remain_lock_conditions = locked - unlocked (chưa được mở từ cart.attributes)
  3. Nếu remaining_lock_size > 0 → inject <script> + spinner vào page
  │
  ▼
JS chạy trong browser
  1. (MR !523) fetch https://ipinfo.io/json → lấy country code
  2. fetch /apps/bss-b2b-lock/ping
       ?locking-conditions=12,34,56,     ← IDs cần check (có trailing comma)
       &country=VN                        ← từ ipinfo.io (MR !523)
       &signature=<shopify_hmac>          ← Shopify tự thêm cho App Proxy
       &shop=mystore.myshopify.com        ← Shopify tự thêm
  │
  ▼
VerifyAndValidateAppProxyGuard
  - Tách signature ra khỏi query params
  - Sắp xếp các key còn lại theo alphabet
  - HMAC-SHA256(sorted_query_string, SHOPIFY_API_SECRET_KEY) == signature?
  - Nếu pass: load shop từ DB và gắn vào request.shop
  │
  ▼
AppProxyService.checkConditionByIp()
  - Load tất cả rules active (status=1) của shop từ DB
    JOIN keys → JOIN conditions
  - Filter conditions theo lockingConditions[] (chỉ check IDs client gửi lên)
  - Check specific_ip:
      checkIp = '*bss*' + currentIp + '*bss*'
      contains = condition.configs.specific_ip.includes(checkIp)
      → cond.inverse ? !contains : contains
  - Check regions:
      temp = regions.map(r => `${r.regionCode}/${r.countryCode}`)
      conditionString = '*bss*' + temp.join('*bss*') + '*bss*'
      ipGeoCheck = '*bss*' + ipGeo + '*bss*'        ← ipGeo từ Cloudflare headers
      contains = conditionString.includes(ipGeoCheck)
  - Check countries (MR !523):
      conditionString = '*bss*' + countries.join('*bss*') + '*bss*'
      contains = conditionString.includes('*bss*' + country + '*bss*')
  │
  ▼
Response → JS nhận về
  Nếu có bất kỳ unlockedConditions nào:
    → Tạo cartData.attributes:
        bss-lock-ip-escape-{id}-check       = currentIp
        bss-lock-region-escape-{id}-check   = ipGeo (e.g. "HN/VN")
        bss-lock-countries-escape-{id}-check = country (e.g. "VN")
    → POST /cart/update.js   ← Shopify native endpoint
    → window.location.reload()
  Nếu không có → hide spinner, kết thúc
  │
  ▼
Page reload → bss-lock-condition.liquid thực thi
  Đọc cart.attributes đã lưu → so sánh với condition config
  → set temp_{id} = true/false → tính access_denied
```

---

## 5. Data Flow — CMS (Merchant cấu hình condition)

```
Merchant tạo rule → chọn condition type "IP/Region/Countries"

Trường hợp specific_ip:
  Merchant nhập thủ công danh sách IP
  CMS format: '*bss*' + ips.join('*bss*') + '*bss*'
  Lưu vào: condition.configs.specific_ip

Trường hợp regions:
  Merchant nhập keyword (ví dụ "Hanoi")
  CMS gọi POST /geolocation/search { search: "Hanoi" }
    → GeolocationService.searchByKeyword()
    → GET ${GEO_SERVICE_HOST}?q=Hanoi&maxRows=10&username=<user>&featureCode=ADM1
    → GeoNames API trả về list { regionCode, countryCode, name, ... }
  Merchant chọn từ kết quả
  Lưu vào: condition.configs.regions = [{ regionCode, countryCode }, ...]

Trường hợp countries (MR !523):
  Merchant chọn quốc gia từ danh sách ISO codes
  Lưu vào: condition.configs.countries = ['VN', 'US', ...]

Merchant save rule
  → RuleService.saveOrUpdate() → DB
  → UploadService.generateAndUpload()
  → Build Liquid → ThemeService.putAsset() → Shopify Theme
```

---

## 6. Liquid Template Generation — `upload.service.ts`

Có **hai ngữ cảnh** sinh Liquid cho IP/region/countries conditions:

### 6.1 `getLockIpAndRegionContent()` — dành cho `bss-lock-ip.liquid`

Hàm này build đoạn Liquid được inject vào **`bss-lock-ip.liquid`** (file cố định, không phải baked-in per-rule).  
Mục đích: tính `bss_locked_conditions_string` và `bss_unlocked_conditions_string` để xác định `remain_lock_conditions`.

#### `specific_ip` — Liquid output:

```liquid
{% comment %}Condition #<id>{% endcomment %}
{% assign bss_locked_conditions_string = bss_locked_conditions_string | append: ',<id>,' %}

{% assign bss_lock_ip_<id>_cart = cart.attributes['bss-lock-ip-escape-<id>-check'] %}
{% if bss_lock_ip_<id>_cart %}
  {% assign bss_lock_ip_<id>_check = '*bss*' | append: bss_lock_ip_<id>_cart | append: '*bss*' %}
{% endif %}
{% assign bss_valid_ip_<id>_check = '*bss*1.2.3.4*bss*10.0.0.1*bss*' %}

{# inverse = false (normal — chỉ IP trong list được phép) #}
{% if bss_valid_ip_<id>_check contains bss_lock_ip_<id>_check %}
  {% assign bss_unlocked_conditions_string = bss_unlocked_conditions_string | append: ',<id>,' %}
{% endif %}

{# inverse = true (NOT — chặn IP trong list) #}
{% if bss_lock_ip_<id>_cart %}
  {% unless bss_valid_ip_<id>_check contains bss_lock_ip_<id>_check %}
    {% assign bss_unlocked_conditions_string = bss_unlocked_conditions_string | append: ',<id>,' %}
  {% endunless %}
{% endif %}
```

#### `regions` — Liquid output:

```liquid
{% assign bss_locked_conditions_string = bss_locked_conditions_string | append: ',<id>,' %}

{% assign bss_lock_region_<id>_cart = cart.attributes['bss-lock-region-escape-<id>-check'] %}
{% if bss_lock_region_<id>_cart %}
  {% assign bss_lock_region_<id>_check = '*bss*' | append: bss_lock_region_<id>_cart | append: '*bss*' %}
{% endif %}
{% assign bss_valid_region_<id>_check = '*bss*HN/VN*bss*CA/US*bss*' %}

{% if bss_valid_region_<id>_check contains bss_lock_region_<id>_check %}
  {% assign bss_unlocked_conditions_string = bss_unlocked_conditions_string | append: ',<id>,' %}
{% endif %}
```

#### `countries` — Liquid output (MR !523):

```liquid
{% assign bss_locked_conditions_string = bss_locked_conditions_string | append: ',<id>,' %}

{% assign bss_lock_countries_<id>_cart = cart.attributes['bss-lock-countries-escape-<id>-check'] %}
{% if bss_lock_countries_<id>_cart %}
  {% assign bss_lock_countries_<id>_check = '*bss*' | append: bss_lock_countries_<id>_cart | append: '*bss*' %}
{% endif %}
{% assign bss_valid_countries_<id>_check = '*bss*VN*bss*US*bss*JP*bss*' %}

{% if bss_valid_countries_<id>_check contains bss_lock_countries_<id>_check %}
  {% assign bss_unlocked_conditions_string = bss_unlocked_conditions_string | append: ',<id>,' %}
{% endif %}
```

---

### 6.2 `getRuleCondition()` — dành cho `bss-lock-condition.liquid`

Hàm này sinh Liquid baked-in cho **toàn bộ rule condition check** (inject vào `bss-lock-condition.liquid`).  
Kết quả là `temp_<id> = true/false` sau đó được group qua `specificIPCondition`, `regionsCondition`, `countriesCondition`.

#### `specific_ip` — case trong switch:

```liquid
{% assign temp_<id> = false %}
{% assign bss_lock_ip_<id> = '*bss*1.2.3.4*bss*10.0.0.1*bss*' %}
{% assign bss_lock_ip_<id>_cart = cart.attributes['bss-lock-ip-escape-<id>-check'] %}
{% if bss_lock_ip_<id>_cart %}
  {% assign bss_lock_ip_<id>_check = '*bss*' | append: bss_lock_ip_<id>_cart | append: '*bss*' %}
{% endif %}

{# inverse = false #}
{% if bss_lock_ip_<id> contains bss_lock_ip_<id>_check %}
  {% assign temp_<id> = true %}
{% endif %}

{# inverse = true #}
{% if bss_lock_ip_<id>_cart %}
  {% unless bss_lock_ip_<id> contains bss_lock_ip_<id>_check %}
    {% assign temp_<id> = true %}
  {% endunless %}
{% endif %}
```

#### `regions` — case trong switch:

```liquid
{% assign temp_<id> = false %}
{% assign bss_lock_region_<id> = '*bss*HN/VN*bss*CA/US*bss*' %}
{% assign bss_lock_region_<id>_cart = cart.attributes['bss-lock-region-escape-<id>-check'] %}
{% if bss_lock_region_<id>_cart %}
  {% assign bss_lock_region_<id>_check = '*bss*' | append: bss_lock_region_<id>_cart | append: '*bss*' %}
{% endif %}

{% if bss_lock_region_<id> contains bss_lock_region_<id>_check %}
  {% assign temp_<id> = true %}
{% endif %}
```

#### `countries` — case trong switch (MR !523):

```liquid
{% assign temp_<id> = false %}
{% assign bss_lock_countries_<id> = '*bss*VN*bss*US*bss*JP*bss*' %}
{% assign bss_lock_countries_<id>_cart = cart.attributes['bss-lock-countries-escape-<id>-check'] %}
{% if bss_lock_countries_<id>_cart %}
  {% assign bss_lock_countries_<id>_check = '*bss*' | append: bss_lock_countries_<id>_cart | append: '*bss*' %}
{% endif %}

{% if bss_lock_countries_<id> contains bss_lock_countries_<id>_check %}
  {% assign temp_<id> = true %}
{% endif %}
```

#### Nhóm conditions thành openTag / closeTag:

```typescript
// Sau khi tất cả conditions của key được process:
if (hasSpecificIPCondition) {
    specificIPCondition.openTag  = `{% if ${conditions.join(' and ')} %}`;
    specificIPCondition.closeTag = `{% else %} {% assign invalid_ip = true %} {% endif %}`;
}
if (hasRegionCondition) {
    regionsCondition.openTag  = `{% if ${conditions.join(' and ')} %}`;
    regionsCondition.closeTag = `{% else %} {% assign current_lock_element_id = temp_lock_element_id %} {% assign invalid_ip = true %} {% endif %}`;
}
if (hasCountriesCondition) {
    countriesCondition.openTag  = `{% if ${conditions.join(' and ')} %}`;
    countriesCondition.closeTag = `{% else %} {% assign current_lock_element_id = temp_lock_element_id %} {% assign invalid_ip = true %} {% endif %}`;
}
```

> Priority trong conditions array: IP = 17, Regions = 17, Countries = 18 (theo `priority` field trong conditions list của `getRuleCondition()`).

---

## 7. Cart Attributes Pattern

Mỗi condition type dùng 1 cart attribute key duy nhất để tránh conflict:

| Condition Type | Cart Attribute Key | Giá trị lưu | Ví dụ |
|---------------|--------------------|-------------|-------|
| `specific_ip` | `bss-lock-ip-escape-{id}-check` | IP address | `1.2.3.4` |
| `regions` | `bss-lock-region-escape-{id}-check` | `regionCode/countryCode` | `HN/VN` |
| `countries` | `bss-lock-countries-escape-{id}-check` | ISO country code | `VN` |

**Cơ chế unlock qua cart attribute**:
```
1. JS lưu vào cart.attributes sau khi server xác nhận unlock
2. Shopify lưu cart attributes persist theo session
3. Mỗi lần page load, Liquid đọc cart.attributes
4. Nếu giá trị trong cart attribute khớp với condition config → access granted
5. Không cần server roundtrip mỗi lần render → performance tốt
```

**Wrap `*bss*`**: Giá trị được wrap `*bss*...*bss*` khi check `contains` để tránh partial match:
```
Cart attr: "1.2.3"
Valid IPs: "*bss*1.2.3.4*bss*"
→ "1.2.3" nằm trong "*bss*1.2.3.4*bss*" (partial match) → FALSE POSITIVE ❌

Với wrap:
Cart check: "*bss*1.2.3*bss*"
→ "*bss*1.2.3.4*bss*" contains "*bss*1.2.3*bss*" → false ✅
```

---

## 8. Guard — `VerifyAndValidateAppProxyGuard`

File: `src/modules/auth/guards/appProxy.guard.ts`

```
Request đến /app_proxy/ping
  → Query params chứa: locking-conditions, shop, country, signature, ...
  │
  ▼
Guard.canActivate()
  1. Tách signature ra khỏi các params còn lại
  2. Sort các params còn lại theo alphabet
  3. Format: "key1=val1key2=val2..." (ghép không có dấu &)
  4. HMAC-SHA256(formatted_string, SHOPIFY_API_SECRET_KEY)
  5. So sánh với signature → không khớp → throw UnauthorizedException
  6. Load shop theo payload.shop domain từ DB
  7. Gắn shopObj vào request.shop
```

> **Khác với ShopGuard (session-token JWT)**:
> - `ShopGuard` dùng cho CMS requests — verify JWT `session-token` header
> - `VerifyAndValidateAppProxyGuard` dùng cho App Proxy requests từ Storefront — verify Shopify HMAC query signature

---

## 9. Quick Decision Tree

```
Merchant muốn giới hạn truy cập theo địa lý?
        │
        ├─► Theo IP cụ thể (tầng thấp nhất, chính xác nhất)?
        │   └─► condition.type = 'specific_ip'
        │       configs.specific_ip = '*bss*<ip1>*bss*<ip2>*bss*'
        │       Cart attr: bss-lock-ip-escape-{id}-check
        │
        ├─► Theo vùng/tỉnh (state/province/city)?
        │   └─► condition.type = 'regions'
        │       configs.regions = [{ regionCode, countryCode }, ...]
        │       Merchant dùng /geolocation/search để tìm region
        │       Cart attr: bss-lock-region-escape-{id}-check
        │       ipGeo nguồn: Cloudflare cf-region-code / cf-ipcountry headers
        │
        └─► Theo quốc gia (đơn giản nhất)?   ← MR !523 / LOGIN-378
            └─► condition.type = 'countries'
                configs.countries = ['VN', 'US', 'JP']  (ISO 3166-1 alpha-2)
                Cart attr: bss-lock-countries-escape-{id}-check

Muốn chặn ngược (chặn thay vì cho phép)?
        └─► Bật condition.inverse = 1 trên bất kỳ loại nào
            Logic sẽ chuyển từ IF-contains sang UNLESS-contains

Lỗi: spinner không tắt sau khi IP/region/countries check?
        ├─► Kiểm tra cart.attributes đã được set chưa (DevTools → /cart.js)
        ├─► Kiểm tra /ping endpoint trả về đúng conditionIds chưa
        └─► Kiểm tra bss_locked_conditions_string có đúng condition IDs không
            (thường do upload bị lỗi hoặc theme cũ)

Lỗi: condition luôn unlock (không chặn) dù chưa verify?
        └─► Kiểm tra bss_valid_*_check có đúng giá trị không
            → Trigger upload lại rule để regenerate Liquid
```

---

## 10. Mapping đầy đủ: variable ↔ hàm sinh ↔ file inject

| Liquid variable | Sinh bởi | Inject vào file |
|----------------|----------|-----------------|
| `bss_locked_conditions_string` | `getLockIpAndRegionContent()` | `bss-lock-ip.liquid` qua `{{ ip_conditions_value }}` |
| `bss_unlocked_conditions_string` | `getLockIpAndRegionContent()` | idem |
| `bss_lock_ip_<id>_cart` | `getLockIpAndRegionContent()` + `getRuleCondition()` | cả hai file |
| `bss_valid_ip_<id>_check` | `getLockIpAndRegionContent()` | `bss-lock-ip.liquid` |
| `bss_lock_ip_<id>` | `getRuleCondition()` case `specific_ip` | `bss-lock-condition.liquid` |
| `bss_lock_region_<id>_cart` | cả hai | cả hai file |
| `bss_valid_region_<id>_check` | `getLockIpAndRegionContent()` | `bss-lock-ip.liquid` |
| `bss_lock_region_<id>` | `getRuleCondition()` case `regions` | `bss-lock-condition.liquid` |
| `bss_lock_countries_<id>_cart` | cả hai | cả hai file |
| `bss_valid_countries_<id>_check` | `getLockIpAndRegionContent()` | `bss-lock-ip.liquid` |
| `bss_lock_countries_<id>` | `getRuleCondition()` case `countries` | `bss-lock-condition.liquid` |
| `temp_<id>` | `getRuleCondition()` | `bss-lock-condition.liquid` |

---

## 11. ⚠️ Known Issues & Notes

### 11.1 Countries — Client-side ipinfo.io (MR !523)

Hiện tại MR !523 lấy country từ `https://ipinfo.io/json` phía browser. Có 3 vấn đề:

1. **Security**: Giá trị `country` đến từ client → user có thể giả mạo
2. **Rate limit**: ipinfo.io free tier 50k requests/tháng — nhiều shops cùng dùng sẽ hết quota
3. **Performance**: Thêm 1 serial fetch tăng latency page load

**Fix đúng**: Lấy country từ `cf-ipcountry` Cloudflare header phía server (như `ipGeo` đang làm):
```typescript
// request-context.decorator.ts
ctx.ipCountry = request.header('cf-ipcountry') ?? '';
```

### 11.2 `inverse` condition với countries

Khi `inverse = 1` (chặn quốc gia), cần có cart attribute được set trước mới check được:
```liquid
{% if bss_lock_countries_<id>_cart %}   ← Phải có cart attr
  {% unless ... %}
    {% assign bss_unlocked_conditions_string = ... %}
  {% endunless %}
{% endif %}
```
→ Nếu cart attribute chưa set (lần đầu load) và `inverse = 1`, user sẽ bị chặn cho đến khi JS ping chạy xong.

### 11.3 `invalid_ip` variable dùng cho cả regions lẫn countries

`regionsCondition.closeTag` và `countriesCondition.closeTag` cùng dùng biến `invalid_ip = true`.  
Tên biến gây nhầm lẫn nhưng không gây bug vì cả hai đều dùng cùng 1 flag.
````
This is the code block that represents the suggested code change:
```markdown
## 1. Feature Architecture Diagram
   - Storefront → Backend → DB flow (ASCII diagram)

## 2. Condition Config Structure
   - condition.type = '<type>'
   - condition.configs = { ... }  (các field thực tế trong DB)
   - condition.inverse = 0 | 1  (nếu áp dụng)

## 3. API Endpoints (nếu có)
   - Controller path, method, guard
   - Request/Response format

## 4. Data Flow
   - Runtime: Liquid/JS gọi gì, validate ở đâu
   - Save rule: UploadService sinh Liquid snippet thế nào

## 5. Entities (nếu có bảng riêng)

## 6. Liquid Snippet Generated
   - getRuleCondition() sinh ra gì cho condition type này
   - Cart attribute names, assign statements

## 7. Known Issues / Notes
```
<userPrompt>
Provide the fully rewritten file, incorporating the suggested code change. You must produce the complete file.
</userPrompt>
