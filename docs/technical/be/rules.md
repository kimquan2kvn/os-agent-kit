# Rules (locks) engine

## Mục tiêu
Rules (locks) định nghĩa điều kiện khóa/mở nội dung (theo sản phẩm, collection, page, passcode, IP, region, …) và được dùng bởi nhiều feature (upload theme code, passcode validation, app proxy, analytics, …).

## API endpoints
Controller: `src/modules/rules/rule.controller.ts` (`@Controller('rules')`)

- `GET /rules/` → lấy toàn bộ rules theo shop (`RuleService.getAllRules(shopId)`)
- `GET /rules/:id` → lấy 1 rule theo id
- `POST /rules/` → tạo/cập nhật rule (nhận `RuleDto`)
- `POST /rules/delete` → xóa rule
- `POST /rules/bulk` → bulk operation (enable/disable/…)
- `POST /rules/shop` → lấy rules theo filter/pagination (name/status/typeCondition)
- `GET /rules/by-condition/:conditionType` → lấy rules theo condition type (ví dụ `passcode`)
- `POST /rules/save-translation` → cập nhật translation cho rule
- `POST /rules/clear-google-cache` → trigger logic “clear google cache” (HPOGS)

Auth: mặc định bị bảo vệ bởi `ShopGuard` (global `APP_GUARD`).

## Data model
- Entity: `src/modules/rules/entities/rule.entity.ts` (table: `rules-v2`)
- Quan hệ chính:
  - `Rules` 1—N `Keys` (`ruleKeys`)
  - `Keys` 1—N `Conditions` (`ruleConditions`)
  - `Rules` 1—1 `Advanced`
  - `Rules` 1—1 `DesignV2`
  - `Rules` 1—N `Translation`

Gợi ý khi debug:
- Keys/Conditions được cascade + `orphanedRowAction: 'delete'` → update rule thường thay đổi sâu cấu trúc keys/conditions.

## Luồng save/update rule
Entry: `RuleController.saveRule()` → `RuleService.saveOrUpdateRule(shop, ruleData)`.

Điểm đáng chú ý trong service (`src/modules/rules/rule.service.ts`):
- Tự set:
  - `ruleData.domain_id = shop.id`
  - `ruleData.theme_id` lấy từ `Config` của shop
- Transform dữ liệu DTO trước khi save: `RuleDto.toTransform()` (`src/modules/rules/dto/rules.dto.ts`)
  - Chuẩn hóa các field sản phẩm/collection/blog/variant/page/url… về “raw” form
  - `advanced` cũng được normalize (`AdvancedDto.toRawAdvanced`)
- Sau khi save:
  - Có logic cache/invalidation cho Hide Price On Google (HPOGS) qua `handleHPOGSCache(...)`.
  - Với shop mới (first rule): cập nhật `shop.is_first_save_rule` và bắn Brevo event `login_firstrule`.

## Convention về condition type
Một số nơi trong code dựa vào string `condition.type`:
- `passcode` (passcode validation + usage history)
- `specific_ip`, `regions` (App Proxy check theo IP/geo)

Khi thêm condition type mới, cần search toàn codebase các chỗ filter theo `cond.type` để cập nhật đồng bộ.
