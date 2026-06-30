# Plan: Passcode Expired After X Hours

## Tổng quan vấn đề

Hiện tại, khi khách nhập đúng passcode, unlock state được lưu vĩnh viễn trong **Shopify cart attributes** (`bss-lock-pc-escape-<id>-success`). Không có cơ chế expire. Feature này thêm config `passcode_expire_hours` để tự động lock lại sau X giờ.

---

## Cơ chế hoạt động

**Unlock state hiện tại:**
```
cart.attributes['bss-lock-pc-escape-<id>-success'] = "abc123"   ← passcode đúng
cart.attributes['bss-lock-pc-escape-<id>-check']   = "abc123"   ← để detect lỗi
```

**Cơ chế expire (mới):**
- Khi unlock: lưu thêm timestamp Unix → `cart.attributes['bss-lock-pc-ts-<id>']`
- Khi Liquid render: so sánh `now` với `stored_ts + expire_seconds`
- Nếu hết hạn → không set `temp_<id> = true` → trang vẫn bị lock
- Nếu expire (JS): dọn cart attributes cũ bằng `/cart/update.js`

**Lý do dùng cart attributes cho timestamp** (thay vì localStorage):
- Cart persist across devices với logged-in customer
- Liquid đọc được → kiểm tra server-side không cần API roundtrip
- Consistent với cơ chế hiện tại

---

## Scope — passcode_type nào áp dụng

| passcode_type | Tên | Áp dụng expire? |
|---|---|---|
| `0` ONE_TO_ALL | Unlock 1 lần → session | ✅ Chính |
| `1` ONCE_TO_EACH | Per-page unlock | ✅ |
| `2` ONCE_FOR_ONCE | Clear mỗi lần vào page | ❌ Đã tự expire rồi (skip) |

---

## Các file cần thay đổi

### 1. `login-api-v2/src/common/types/rule.ts` — Thêm field config

```typescript
// ConditionConfigsType — thêm:
passcode_expire_hours?: number;  // 0 = never expire, > 0 = expire after N hours
```

### 2. `login-api-v2/src/modules/upload/upload.service.ts` — Core changes

#### 2a. Trong `getRuleCondition()` — case `'passcode'`

**Thêm tên cart attribute mới** (sau dòng assign `passAttrNameCheck`):

```typescript
// Naming pattern: giống Success/Check, thêm suffix '-ts'
// Type 0: 'bss-lock-pc-ts-<id>'
// Type 1: 'bss-lock-pc-ts-<id>-<scope>'
```

Liquid mới trong `contentConditionsByKey`:
```liquid
{# Assign tên attr timestamp — cùng pattern với success/check #}
{% assign passAttrNameTs-<id> = 'bss-lock-pc-ts-<id>' %}             {# type 0 #}
{# hoặc #}
{% assign passAttrNameTs-<id> = 'bss-lock-pc-ts-<id>-' | append: extra %}  {# type 1 #}

{# Đọc timestamp từ cart #}
{% assign passAttrTs-<id> = cart.attributes[passAttrNameTs-<id>] | plus: 0 %}
```

**Thay thế block kiểm tra unlock** (từ dòng `{% if bss-lock-pc-escape-... contains ... %}`):

```liquid
{# Nếu có expire config (expire_seconds được bake-in lúc build Liquid) #}
{% if <expire_seconds> > 0 %}
  {% assign bss_now_ts = 'now' | date: '%s' | plus: 0 %}
  {% assign bss_expire_at = passAttrTs-<id> | plus: <expire_seconds> %}
  {% if passAttrTs-<id> > 0 and bss_now_ts <= bss_expire_at %}
    {% if bss-lock-pc-escape-<id> contains passAttrSuccess-<id> %}
      {% assign temp_<id> = true %}
    {% endif %}
  {% endif %}
{% else %}
  {# Không có expire — giữ behavior cũ #}
  {% if bss-lock-pc-escape-<id> contains passAttrSuccess-<id> %}
    {% assign temp_<id> = true %}
  {% endif %}
{% endif %}
```

Trong đó `<expire_seconds>` = `condition.configs.passcode_expire_hours * 3600` được bake-in
lúc build Liquid (số cứng, không cần Liquid tính toán).

#### 2b. Trong `getPasscodeFormData()` — Lưu timestamp khi submit

Thêm vào `formDataString` (trong vòng loop `for condition_id in all_passcode_condition_id`):
```liquid
{# Append timestamp cart attribute khi unlock #}
{% assign current_index_ts = passcode_condition_ids | find_index: condition_id %}
formData.append('attributes[{{cart_attributes_ts_names[current_index_ts]}}]', Math.floor(Date.now() / 1000).toString());
```

Cần bổ sung thêm biến `cart_attributes_ts_names` tương tự `cart_attributes_success_names`.

#### 2c. Cleanup khi expire — JS on page load (optional)

Inject JS vào cuối snippet (nếu `passcode_expire_hours > 0`):
```liquid
<script>
  (function() {
    var now = Math.floor(Date.now() / 1000);
    var expireSeconds = <expire_seconds>;
    var storedTs = parseInt("{{ passAttrTs-<id> }}", 10) || 0;
    if (storedTs > 0 && (now - storedTs) > expireSeconds) {
      var data = { attributes: {} };
      data.attributes['{{ passAttrNameSuccess-<id> }}'] = '';
      data.attributes['{{ passAttrNameCheck-<id> }}']   = '';
      data.attributes['{{ passAttrNameTs-<id> }}']      = '';
      fetch('/cart/update.js', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data)
      });
    }
  })();
</script>
```

> Cleanup JS là optional/nice-to-have. Liquid check đã đủ để deny access.
> Cleanup chỉ nhằm tránh cart attribute tích lũy và để form hiện ra ngay sau khi expire.

---

## Migration — Không cần

`passcode_expire_hours` lưu trong `condition.configs` (JSON column của entity `Conditions`),
không cần DB migration mới.

---

## Edge cases & gotchas

| Case | Xử lý |
|---|---|
| `passcode_expire_hours = 0` hoặc không set | Không có expire check → behavior cũ |
| `passAttrTs = 0` (chưa nhập passcode sau khi deploy) | Treat như "không có timestamp" → không bị expire đột ngột |
| `passcode_type = 2` (ONCE_FOR_ONCE) | Skip expire — đã có relock mechanism riêng |
| Clock skew giữa client JS và Shopify server | Chênh lệch vài giây, acceptable với granularity X hours |
| Khách đổi thiết bị | Cart attributes sync theo customer → timestamp cũng sync |
| Merchant thay đổi expire_hours | Liquid re-generated khi save rule → có hiệu lực ngay lần load sau |

---

## Thứ tự implementation

1. **`rule.ts`** — thêm `passcode_expire_hours` vào `ConditionConfigsType` (5 phút)
2. **`upload.service.ts` `getRuleCondition()`** — thêm `passAttrNameTs` + logic expire check trong Liquid (2–3 giờ, phần phức tạp nhất)
3. **`upload.service.ts` `getPasscodeFormData()`** — thêm timestamp vào formData append (30 phút)
4. **Cleanup JS** (optional, 1 giờ)
5. **Test end-to-end** với shop test: regenerate + upload liquid, nhập passcode, đợi hết hạn, reload

---

## Điểm cần confirm trước khi code

1. **CMS side**: Merchant cấu hình `passcode_expire_hours` ở đâu trong form tạo rule? (Input number trong block Passcode condition)
2. **Behavior khi expire**: Chỉ lock lại (hiện form lại) hay còn làm gì thêm (redirect, clear session)?
3. **Đơn vị**: Hours hay minutes? (Plan dùng hours — đổi thành minutes chỉ cần sửa `* 3600` → `* 60`)
4. **Granularity type 1**: Per-page expire (mỗi page có timestamp riêng) hay global expire (1 timestamp chung cho cả rule)?
