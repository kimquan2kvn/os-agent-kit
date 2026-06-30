# Location Condition Improvement — Dev Plan

**Status:** Planning  
**Scope:** Frontend (login-cms) — Task này chỉ làm phần Countries, chưa làm Regions

---

## 1. Bối cảnh & Vấn đề

### Hiện trạng

Condition type `'regions'` hiện tại:
- **Label:** "Location – Only for selected countries/regions" (gây nhầm lẫn)
- **Component `SpecificRegions`:** bị comment out hoàn toàn — UI trống, không có gì để chọn
- **Data cũ từ Geonames API:** `configs.regions = [{ id, name, countryName, population }]` (API không còn dùng)
- Không có flag hiển thị, không searchable hiệu quả

### Mục tiêu

| Condition cũ | Condition mới |
|---|---|
| `regions` = "Location – Only for selected countries/regions" | `countries` = "Location – Only for selected countries" _(mới)_ |
| | `regions` = "Location – Only for selected regions" _(giữ, đổi label — làm sau)_ |

**Task này chỉ implement condition `countries`.**  
Condition `regions` giữ nguyên key, chỉ đổi label để tránh nhầm lẫn — UI regions sẽ làm sau.

---

## 2. Data Model

### Condition type mới: `'countries'`

```js
// Trong configs của một condition
{
  countries: ["US", "CA", "VN"]  // mảng ISO 3166-1 alpha-2 codes
}
```

### Backward compatibility cho `'regions'`

Khách cũ đã có rule với condition `regions`:
- Giữ nguyên key `regions` trong DB
- Chỉ đổi label trong UI từ "…countries/regions" → "…regions"
- `configs.regions` = array cũ (giữ nguyên, không migrate)

---

## 3. Files cần thay đổi

### 3.1 Constants

#### `login-cms/web/client/constants/rule.constant.js`
```js
// Thêm vào RULE_CONDITION:
COUNTRY: "countries",

// Thêm vào PREVIEW_PAGE, PREVIEW_ELEMENT:
countries: false,

// Thêm vào defaultRule.configs:
countries: [],

// Đổi label regions:
regions: { label: "Location – Only for selected regions" }
```

#### `login-cms/web/client/constants/locks/access-message.const.js`
```js
// Thêm condition mới vào CONDITION_OPTIONS():
countries: {
  label: t('countries'),           // "Location – Only for selected countries"
  value: 'countries',
  icon: LocationIcon,
  plan: 'Advanced',
  isAllow: true
},

// Đổi label regions:
regions: {
  label: t('regions'),             // "Location – Only for selected regions"
  ...
},

// Thêm 'countries' vào location group trong CONDITION_GROUPS():
{ id: 'location', ..., conditions: ['market_specific', 'specific_ip', 'countries', 'regions'] },
```

#### `login-cms/web/client/constants/common.const.js`
```js
// Thêm action mới:
export const SET_COUNTRIES = "SET_COUNTRIES";
```

#### `login-cms/web/client/constants/countries-with-flag.const.js` _(file mới)_
```js
// Array of { name, code, flag } — dùng để render UI
export const COUNTRIES_WITH_FLAG = [
  { name: "Afghanistan", code: "AF", flag: "/images/flags/af.svg" },
  // ... toàn bộ 250+ countries
];

// Map code → country (lookup nhanh)
export const COUNTRY_MAP = Object.fromEntries(
  COUNTRIES_WITH_FLAG.map(c => [c.code, c])
);
```

### 3.2 Reducer — `useCondition.jsx`

```js
// Import thêm:
import { ..., SET_COUNTRIES } from '../../constants/common.const';

// Thêm case vào reducer:
case SET_COUNTRIES:
  updateKeys[keyIndex].conditions[conditionIndex].configs.countries = newData;
  return { ...prev, keys: updateKeys };

// Trong ADD_CONDITION, thêm countries vào defaultConfig:
countries: [],
```

### 3.3 Components

#### `ConditionConfigurator.jsx` — thêm render cho `countries`
```jsx
{condition.type === 'countries' &&
  <SpecificCountries
    handleChangeKeyCondition={handleChangeKeyCondition}
    keyIndex={keyIndex}
    conditionIndex={conditionIndex}
    countries={condition.configs.countries ?? []}
  />
}
```

#### `SpecificCountries/index.jsx` _(component mới)_
Xem phần 4 — UI Design.

### 3.4 i18n Translation

Thêm key dịch cho `countries` và cập nhật label `regions` trong các file translation.

---

## 4. UI Design

### 4a. Trigger (inline)

Khi chọn condition type `countries`, hiện button "Select countries":

```
┌─────────────────────────────────────────────┐
│  Location – Only for selected countries     │
│                                             │
│  [🌍 Select countries ▼]                   │  ← click mở Modal
│                                             │
│  Selected (3):                              │
│  [🇺🇸 United States ×] [🇨🇦 Canada ×]       │
│  [🇻🇳 Vietnam ×]                            │
│                                             │
│  [Clear all]                                │
└─────────────────────────────────────────────┘
```

### 4b. Modal — Country Picker

```
┌─────────────────────────────────────────────────────┐
│  Select countries                               [×] │
├─────────────────────────────────────────────────────┤
│  🔍 [Search countries...                        ]   │
├─────────────────────────────────────────────────────┤
│  □  🇦🇫  Afghanistan                               │
│  □  🇦🇱  Albania                                   │
│  □  🇩🇿  Algeria                                   │
│  □  🇦🇸  American Samoa                            │
│  ✅  🇺🇸  United States                            │
│  ✅  🇨🇦  Canada                                   │
│  ✅  🇻🇳  Vietnam                                  │
│  ...                                               │
├─────────────────────────────────────────────────────┤
│  3 countries selected                               │
│                      [Cancel]  [Confirm selection]  │
└─────────────────────────────────────────────────────┘
```

**Behavior:**
- Search filter theo name (case-insensitive)
- Selected items nổi lên đầu danh sách (hoặc đánh dấu ✅)
- Scroll virtualized nếu cần (250+ items)
- Modal width ~600px, max height ~500px với scroll

### 4c. Component Props Interface

```jsx
// SpecificCountries/index.jsx
function SpecificCountries({
  handleChangeKeyCondition,  // (action, keyIndex, conditionIndex, data) => void
  keyIndex,                  // number
  conditionIndex,            // number
  countries,                 // string[] — array of ISO codes e.g. ["US", "CA"]
})
```

---

## 5. Implementation Steps

```
[ ] Bước 1: Tạo file constants/countries-with-flag.const.js
[ ] Bước 2: Cập nhật rule.constant.js — thêm COUNTRY, countries config
[ ] Bước 3: Cập nhật access-message.const.js — thêm condition, cập nhật groups
[ ] Bước 4: Cập nhật common.const.js — thêm SET_COUNTRIES
[ ] Bước 5: Cập nhật useCondition.jsx — thêm case SET_COUNTRIES
[ ] Bước 6: Tạo component SpecificCountries/index.jsx
[ ] Bước 7: Cập nhật ConditionConfigurator.jsx — render SpecificCountries
[ ] Bước 8: Cập nhật i18n translations
[ ] Bước 9: Test end-to-end: tạo rule mới với countries condition
[ ] Bước 10: Verify backward compat: rule cũ có regions vẫn hiển thị đúng
```

---

## 6. Backend Consideration

Backend (`login-api-v2`) cần được verify:

- **Validate:** `condition.type === 'countries'` với `configs.countries: string[]`
- **Geolocation check:** Khi visitor vào store, backend cần check IP → country code → so sánh với `configs.countries[]`
- **Liquid upload:** Upload service cần handle `countries` condition khi generate Liquid/JS

> Phần backend sẽ làm riêng sau khi frontend hoàn thành. Frontend task này chỉ build UI.

---

## 7. Out of Scope (task này)

- Region condition UI (state/province selector)
- Backend implementation cho countries condition
- City-level targeting
- Include/exclude logic (chỉ "only for" = include)
- Migration data từ regions sang countries

---

## 8. Files Summary

| File | Action |
|---|---|
| `constants/countries-with-flag.const.js` | **Tạo mới** |
| `constants/rule.constant.js` | **Sửa** — thêm COUNTRY, countries default, đổi label regions |
| `constants/locks/access-message.const.js` | **Sửa** — thêm countries condition + group |
| `constants/common.const.js` | **Sửa** — thêm SET_COUNTRIES |
| `components/hooks/useCondition.jsx` | **Sửa** — thêm SET_COUNTRIES case |
| `components/ui/ConditionContent/SpecificCountries/index.jsx` | **Tạo mới** |
| `components/ui/ConditionSettings/ConditionConfigurator.jsx` | **Sửa** — render SpecificCountries |
| i18n translation files | **Sửa** — thêm keys |