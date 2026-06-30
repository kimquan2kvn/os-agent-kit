# Plan: Improve Lock Template

**Status:** Planning  
**Scope:** Frontend (login-cms) — chủ yếu `formPresets.const.js` + `FormPresets/index.jsx` + i18n  
**Không cần backend** trừ template `geo_lock` (cần confirm condition type)

---

## 1. Bối cảnh & Vấn đề

### Hiện trạng

`RULES_TEMPLATE` hiện có **6 template**:

| ID | Tên | Target | Condition |
|---|---|---|---|
| `manually` | Blank (tự setup) | — | — |
| `ltsp` | Require login to view prices | `PRICE` (8) | `signed_in` |
| `hide_product` | Require login to view products | `PRODUCT` (1) | `customer_specific` |
| `passcode` | Require passcodes to view pages | `PAGE` (5) | `passcode` |
| `checkout_lock` | Require login to checkout | Checkout Lock | `customerLoggedIn` |
| `hpogg` | Hide price on Google Search | `PRODUCT` (1) | `hide_price_on_google` |

**Vấn đề:**
- Template `sub_email` đã có đầy đủ data trong `TEMPLATE_DATA` và ảnh tại `public/images/FormPresets/sub_email.png` nhưng **KHÔNG xuất hiện** trong `RULES_TEMPLATE` → dead code
- Nhiều use case phổ biến (B2B wholesale gate, age verify, secret link, geo-lock, hide collections…) chưa có template → khách phải tự setup từ đầu dù là flow rất phổ biến
- Template card chỉ hiển thị ảnh + tên, không có mô tả ngắn → khách không biết template dùng để làm gì
- Tất cả 6 template nằm cùng 1 grid 3 cột không có nhóm → khó tìm khi template nhiều hơn

### Mục tiêu

1. Thêm **8 template mới** (bao gồm fix `sub_email`)
2. Thêm **short description** vào mỗi template card
3. Nhóm templates theo **category** để dễ tìm
4. (Optional) Thêm badge `New` cho template mới ra

---

## 2. Templates mới đề xuất

### 2.1 Fix ngay — `sub_email` (đã có data)

Template này đã có đầy đủ data + ảnh, chỉ cần thêm vào `RULES_TEMPLATE`.

| Trường | Giá trị |
|---|---|
| `id` | `sub_email` |
| `image` | `/images/FormPresets/sub_email.png` |
| `isNew` | `false` |
| Mô tả | "Gate collections — visitors must subscribe to email to view" |

---

### 2.2 Templates mới hoàn toàn

#### T1 — `wholesale_store`  
**B2B / Wholesale store gate**

| Trường | Giá trị |
|---|---|
| Target | `ALL` (0) — Lock Entire Store |
| Condition | `customer_tag` |
| Mô tả | "Restrict your entire store to tagged wholesale customers only" |
| Use case | Shop B2B chỉ cho phép khách có tag `wholesale` / `b2b` vào xem |

```js
// TEMPLATE_DATA['wholesale_store']
{
    name: "Wholesale store for tagged customers",
    priority: 0, status: 1, action_type: 0,
    target_type: 0,  // ALL
    keys: [{
        redirect_url: "",
        conditions: [{
            inverse: 0,
            type: "customer_tag",
            inputValue: "If the customer has a specific tag",
            configs: {
                token: "*bss**bss*", liquid: "", prelude: "",
                regions: [], passcode: "*bss**bss*",
                date_time: new Date().toISOString(),
                specific_ip: "*bss**bss*", customer_ids: [],
                market_region_data: {}, apply_to_market_type: 0,
                passcode_case_sensitive: 1,
                customer_tag: []   // merchant tự điền tag
            }
        }]
    }],
    advanced: {
        customer_auto_tag: false, force_open_lock: false,
        enable_header_footer: false, enable_navigation_bar: false,
        hide_collection_sitemaps: false, manual_locking: false,
        add_noindex: true,           // ← B2B store không cần SEO index
        hide_collection_link: true, hide_product_link: true,
        allow_home_page: false, allow_customer_areas: true,
        allow_policy_pages: true, hide_page_link: false,
        hide_product: true, hide_product_collection_on_google: false,
        hide_collection: false, lock_product_in_collection: true,
        hide_price_on_google: false, allow_specific_pages: false,
        customer_tag: null, allow_pages: []
    }
}
```

---

#### T2 — `age_verify`  
**Age verification gate**

| Trường | Giá trị |
|---|---|
| Target | `ALL` (0) — Lock Entire Store |
| Condition | `age_verification` |
| Mô tả | "Ask visitors to confirm they are above the required age before entering" |
| Use case | Shop bán rượu, thuốc lá, sản phẩm 18+ |

```js
// TEMPLATE_DATA['age_verify']
{
    name: "Age verification gate",
    priority: 0, status: 1, action_type: 0,
    target_type: 0,  // ALL
    keys: [{
        redirect_url: "",
        conditions: [{
            inverse: 0,
            type: "age_verification",
            inputValue: "If the customer confirms their age",
            configs: {
                token: "*bss**bss*", liquid: "", prelude: "",
                regions: [], passcode: "*bss**bss*",
                date_time: new Date().toISOString(),
                specific_ip: "*bss**bss*", customer_ids: [],
                market_region_data: {}, apply_to_market_type: 0,
                passcode_case_sensitive: 1,
                age_verification: 18  // default minimum age
            }
        }]
    }],
    advanced: {
        customer_auto_tag: false, force_open_lock: false,
        enable_header_footer: true,  // ← giữ header/footer cho trust
        enable_navigation_bar: false,
        hide_collection_sitemaps: false, manual_locking: false,
        add_noindex: false, hide_collection_link: false,
        hide_product_link: false, allow_home_page: false,
        allow_customer_areas: true, allow_policy_pages: true,
        hide_page_link: false, hide_product: false,
        hide_product_collection_on_google: false,
        hide_collection: false, lock_product_in_collection: false,
        hide_price_on_google: false, allow_specific_pages: false,
        customer_tag: null, allow_pages: []
    }
}
```

---

#### T3 — `secret_link`  
**Exclusive access via secret link**

| Trường | Giá trị |
|---|---|
| Target | `PRODUCT` (1) — Hide Products |
| Condition | `secret_link` (`token`) |
| Mô tả | "Share a private link — only visitors with the link can see these products" |
| Use case | Flash sale riêng, VIP product launch, member-only drop |

```js
// TEMPLATE_DATA['secret_link']
{
    name: "Exclusive products via secret link",
    priority: 0, status: 1, action_type: 0,
    target_type: 1,  // PRODUCT
    product_type: 0, // ALL products (merchant tự thêm filter)
    keys: [{
        redirect_url: "",
        conditions: [{
            inverse: 0,
            type: "secret_link",
            inputValue: "If the customer accesses via a secret link",
            configs: {
                token: "*bss**bss*", liquid: "", prelude: "",
                regions: [], passcode: "*bss**bss*",
                date_time: new Date().toISOString(),
                specific_ip: "*bss**bss*", customer_ids: [],
                market_region_data: {}, apply_to_market_type: 0,
                passcode_case_sensitive: 1
            }
        }]
    }],
    advanced: {
        customer_auto_tag: false, force_open_lock: false,
        enable_header_footer: false, enable_navigation_bar: false,
        hide_collection_sitemaps: false, manual_locking: false,
        add_noindex: true, hide_collection_link: false,
        hide_product_link: true, allow_home_page: false,
        allow_customer_areas: false, allow_policy_pages: false,
        hide_page_link: false, hide_product: true,
        hide_product_collection_on_google: true,
        hide_collection: false, lock_product_in_collection: true,
        hide_price_on_google: false, allow_specific_pages: false,
        customer_tag: null, allow_pages: []
    }
}
```

---

#### T4 — `geo_lock`  
**Restrict content by location**

| Trường | Giá trị |
|---|---|
| Target | `ALL` (0) — Lock Entire Store |
| Condition | `countries` (condition type mới — xem plan `location-condition-improvement.md`) |
| Fallback | `regions` nếu `countries` chưa ready |
| Mô tả | "Restrict your store to visitors from specific countries or regions" |
| Use case | Chỉ bán cho một số thị trường, comply với luật địa phương |

> ⚠️ **Phụ thuộc**: Template này dùng condition type `countries` đang được plan trong `location-condition-improvement.md`. Nếu chưa implement `countries`, dùng `regions` tạm thời.

```js
// TEMPLATE_DATA['geo_lock']
{
    name: "Restrict access by location",
    priority: 0, status: 1, action_type: 0,
    target_type: 0,
    keys: [{
        redirect_url: "",
        conditions: [{
            inverse: 0,
            type: "countries",  // fallback: "regions"
            inputValue: "If the visitor is from a specific country",
            configs: {
                token: "*bss**bss*", liquid: "", prelude: "",
                regions: [], countries: [],   // merchant chọn countries
                passcode: "*bss**bss*",
                date_time: new Date().toISOString(),
                specific_ip: "*bss**bss*", customer_ids: [],
                market_region_data: {}, apply_to_market_type: 0,
                passcode_case_sensitive: 1
            }
        }]
    }],
    advanced: {
        customer_auto_tag: false, force_open_lock: false,
        enable_header_footer: false, enable_navigation_bar: false,
        hide_collection_sitemaps: false, manual_locking: false,
        add_noindex: false, hide_collection_link: false,
        hide_product_link: false, allow_home_page: false,
        allow_customer_areas: false, allow_policy_pages: false,
        hide_page_link: false, hide_product: false,
        hide_product_collection_on_google: false,
        hide_collection: false, lock_product_in_collection: false,
        hide_price_on_google: false, allow_specific_pages: false,
        customer_tag: null, allow_pages: []
    }
}
```

---

#### T5 — `hide_collection`  
**Require login to view collections**

| Trường | Giá trị |
|---|---|
| Target | `COLLECTION` (2) |
| Condition | `signed_in` |
| Mô tả | "Hide entire collections from guests — show only to logged-in customers" |
| Use case | Danh mục wholesale, members-only catalog |

```js
// TEMPLATE_DATA['hide_collection']
{
    name: "Require login to view collections",
    priority: 0, status: 1, action_type: 0,
    target_type: 2,  // COLLECTION
    collection_type: 0,
    keys: [{
        redirect_url: "",
        conditions: [{
            inverse: 0,
            type: "signed_in",
            inputValue: "If the customer is signed in",
            configs: {
                token: "*bss**bss*", liquid: "", prelude: "",
                regions: [], passcode: "*bss**bss*",
                date_time: new Date().toISOString(),
                specific_ip: "*bss**bss*", customer_ids: [],
                market_region_data: {}, apply_to_market_type: 0,
                passcode_case_sensitive: 1
            }
        }]
    }],
    advanced: {
        customer_auto_tag: false, force_open_lock: false,
        enable_header_footer: false, enable_navigation_bar: false,
        hide_collection_sitemaps: false, manual_locking: false,
        add_noindex: false, hide_collection_link: true,
        hide_product_link: false, allow_home_page: false,
        allow_customer_areas: true, allow_policy_pages: false,
        hide_page_link: false, hide_product: false,
        hide_product_collection_on_google: false,
        hide_collection: true, lock_product_in_collection: true,
        hide_price_on_google: false, allow_specific_pages: false,
        customer_tag: null, allow_pages: []
    }
}
```

---

#### T6 — `wholesale_price`  
**Wholesale pricing for tagged customers**

| Trường | Giá trị |
|---|---|
| Target | `PRICE` (8) — Hide Price & ATC |
| Condition | `customer_tag` |
| Mô tả | "Show prices and Add to Cart only to customers tagged as wholesale" |
| Use case | B2B pricing — guest thấy ảnh sản phẩm nhưng không thấy giá/nút mua |

```js
// TEMPLATE_DATA['wholesale_price']
{
    name: "Wholesale pricing for tagged customers",
    priority: 0, status: 1, action_type: 0,
    target_type: 8,  // PRICE
    product_type: 0,
    keys: [{
        redirect_url: "",
        conditions: [{
            inverse: 0,
            type: "customer_tag",
            inputValue: "If the customer has a specific tag",
            configs: {
                token: "*bss**bss*", liquid: "", prelude: "",
                regions: [], passcode: "*bss**bss*",
                date_time: new Date().toISOString(),
                specific_ip: "*bss**bss*", customer_ids: [],
                market_region_data: {}, apply_to_market_type: 0,
                passcode_case_sensitive: 1,
                customer_tag: []
            }
        }]
    }],
    advanced: {
        customer_auto_tag: false, force_open_lock: false,
        enable_header_footer: false, enable_navigation_bar: false,
        hide_collection_sitemaps: false, manual_locking: false,
        add_noindex: false, hide_collection_link: false,
        hide_product_link: false, allow_home_page: false,
        allow_customer_areas: false, allow_policy_pages: false,
        hide_page_link: false, hide_product: false,
        hide_product_collection_on_google: false,
        hide_collection: false, lock_product_in_collection: true,
        hide_price_on_google: false, allow_specific_pages: false,
        customer_tag: null, allow_pages: []
    }
}
```

---

#### T7 — `hide_variant`  
**Hide wholesale-only product variants**

| Trường | Giá trị |
|---|---|
| Target | `VARIANT` (6) |
| Condition | `customer_tag` |
| Mô tả | "Hide specific product options (sizes, colors) from customers without the required tag" |
| Use case | Ẩn size/màu bulk/wholesale khỏi khách bán lẻ |

```js
// TEMPLATE_DATA['hide_variant']
{
    name: "Hide wholesale-only product variants",
    priority: 0, status: 1, action_type: 0,
    target_type: 6,  // VARIANT
    keys: [{
        redirect_url: "",
        conditions: [{
            inverse: 0,
            type: "customer_tag",
            inputValue: "If the customer has a specific tag",
            configs: {
                token: "*bss**bss*", liquid: "", prelude: "",
                regions: [], passcode: "*bss**bss*",
                date_time: new Date().toISOString(),
                specific_ip: "*bss**bss*", customer_ids: [],
                market_region_data: {}, apply_to_market_type: 0,
                passcode_case_sensitive: 1,
                customer_tag: []
            }
        }]
    }],
    advanced: {
        customer_auto_tag: false, force_open_lock: false,
        enable_header_footer: false, enable_navigation_bar: false,
        hide_collection_sitemaps: false, manual_locking: false,
        add_noindex: false, hide_collection_link: false,
        hide_product_link: false, allow_home_page: false,
        allow_customer_areas: false, allow_policy_pages: false,
        hide_page_link: false, hide_product: false,
        hide_product_collection_on_google: false,
        hide_collection: false, lock_product_in_collection: false,
        hide_price_on_google: false, allow_specific_pages: false,
        customer_tag: null, allow_pages: []
    }
}
```

---

## 3. RULES_TEMPLATE mới (sau khi thêm)

Tổng: **14 template** (6 cũ + 1 fix sub_email + 7 mới).

```js
export const RULES_TEMPLATE = [
    // ── Starter ──────────────────────────────
    { id: 'manually',         category: 'starter',    image: '...', title: 'manually' },
    { id: 'ltsp',             category: 'starter',    image: '...', title: 'ltsp' },
    { id: 'passcode',         category: 'starter',    image: '...', title: 'passcode' },
    { id: 'hpogg',            category: 'starter',    image: '...', title: 'hpogg' },

    // ── B2B / Wholesale ───────────────────────
    { id: 'wholesale_store',  category: 'b2b',        image: '...', title: 'wholesale_store',  isNew: true },
    { id: 'wholesale_price',  category: 'b2b',        image: '...', title: 'wholesale_price',  isNew: true },
    { id: 'hide_product',     category: 'b2b',        image: '...', title: 'hide_product' },
    { id: 'hide_collection',  category: 'b2b',        image: '...', title: 'hide_collection',  isNew: true },
    { id: 'hide_variant',     category: 'b2b',        image: '...', title: 'hide_variant',     isNew: true },

    // ── Content Gating ────────────────────────
    { id: 'sub_email',        category: 'gating',     image: '...', title: 'sub_email' },
    { id: 'age_verify',       category: 'gating',     image: '...', title: 'age_verify',       isNew: true },
    { id: 'secret_link',      category: 'gating',     image: '...', title: 'secret_link',      isNew: true },
    { id: 'geo_lock',         category: 'gating',     image: '...', title: 'geo_lock',         isNew: true },

    // ── Checkout ──────────────────────────────
    { id: 'checkout_lock',    category: 'checkout',   image: '...', title: 'checkout_lock' },
];
```

> **Lưu ý `category`**: đây là field mới thêm vào object template — dùng để group UI theo section.

---

## 4. UI Changes (FormPresets/index.jsx)

### 4.1 Thêm short description vào template card

Hiện tại card chỉ có `title`. Cần thêm `description` vào object template và render dưới tiêu đề.

```
┌─────────────────────────────────┐
│  [thumbnail image]              │
│  ─────────────────────────────  │
│  Wholesale store for tagged     │  ← title (bold)
│  customers                      │
│                                 │
│  Restrict your entire store to  │  ← description (Text variant="bodySm")
│  tagged wholesale customers     │
│  only                           │
│                       [New]     │  ← badge (nếu có)
└─────────────────────────────────┘
```

### 4.2 Grouping theo category

Thay vì 1 grid phẳng → tách thành **4 sections** dọc, mỗi section có tiêu đề:

```
┌──────────────────────────────────────────────────────────┐
│  STARTER                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │ Manually │  │ Login to │  │ Passcode │               │
│  │          │  │ view     │  │ lock     │               │
│  └──────────┘  └──────────┘  └──────────┘               │
│                                                          │
│  B2B / WHOLESALE                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│  │Wholesale │  │Wholesale │  │  Hide    │  │  Hide    │ │
│  │  Store   │  │  Price   │  │  Colls   │  │ Variants │ │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘ │
│                                                          │
│  CONTENT GATING                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│  │  Email   │  │   Age    │  │ Secret   │  │   Geo    │ │
│  │   Sub    │  │  Verify  │  │  Link    │  │  Lock    │ │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘ │
│                                                          │
│  CHECKOUT                                                │
│  ┌──────────┐                                            │
│  │ Checkout │                                            │
│  │  Lock    │                                            │
│  └──────────┘                                            │
│                                               [Select]  │
└──────────────────────────────────────────────────────────┘
```

**Cách implement grouping:**
- Thêm `category` field vào mỗi item trong `RULES_TEMPLATE`
- Tạo `TEMPLATE_CATEGORIES` constant với thứ tự + label hiển thị
- Trong component: `group by category` rồi render từng section với `<Text variant="headingMd">` làm tiêu đề

```js
// Thêm vào formPresets.const.js
export const TEMPLATE_CATEGORIES = [
    { id: 'starter',  labelKey: 'category.starter' },
    { id: 'b2b',      labelKey: 'category.b2b' },
    { id: 'gating',   labelKey: 'category.gating' },
    { id: 'checkout', labelKey: 'category.checkout' },
];
```

### 4.3 Modal size

Với 14 template chia 4 categories, modal cần `size="large"` (hiện tại không set → mặc định medium).

```jsx
<Modal size="large" ... >
```

---

## 5. Ảnh thumbnail cần tạo

| Template | File cần tạo | Ghi chú |
|---|---|---|
| `sub_email` | `/images/FormPresets/sub_email.png` | **Đã có sẵn** ✅ |
| `wholesale_store` | `/images/FormPresets/wholesale_store.png` | Tạo mới |
| `age_verify` | `/images/FormPresets/age_verify.png` | Tạo mới |
| `secret_link` | `/images/FormPresets/secret_link.png` | Tạo mới |
| `geo_lock` | `/images/FormPresets/geo_lock.png` | Tạo mới |
| `hide_collection` | `/images/FormPresets/hide_collection.png` | Tạo mới |
| `wholesale_price` | `/images/FormPresets/wholesale_price.png` | Tạo mới |
| `hide_variant` | `/images/FormPresets/hide_variant.png` | Tạo mới |

> Kích thước chuẩn: khớp với các ảnh hiện có (xem `manually.png`, `passcode.png`)

---

## 6. i18n Keys cần thêm

File: `login-cms/web/client/locales/en/locks.json` (hoặc tương đương)

Namespace: `presets`

```json
{
  "presets": {
    "title": "Choose a template",
    "select": "Use this template",

    "category": {
      "starter":  "Starter",
      "b2b":      "B2B / Wholesale",
      "gating":   "Content Gating",
      "checkout": "Checkout"
    },

    "manually":        "Start from scratch",
    "ltsp":            "Require login to view prices",
    "passcode":        "Require passcode",
    "hpogg":           "Hide price on Google",
    "hide_product":    "Require login to view products",
    "wholesale_store": "Wholesale store gate",
    "wholesale_price": "Wholesale pricing",
    "hide_collection": "Require login to view collections",
    "hide_variant":    "Hide wholesale variants",
    "sub_email":       "Email subscription gate",
    "age_verify":      "Age verification gate",
    "secret_link":     "Secret link access",
    "geo_lock":        "Location-based restriction",
    "checkout_lock":   "Require login to checkout",

    "desc": {
      "manually":        "Build your own rule from scratch",
      "ltsp":            "Visitors must log in to see product prices",
      "passcode":        "Protect pages behind a secret passcode",
      "hpogg":           "Hide prices from Google crawlers",
      "hide_product":    "Hide products from non-approved customers",
      "wholesale_store": "Lock entire store — only tagged customers enter",
      "wholesale_price": "Show prices only to wholesale-tagged customers",
      "hide_collection": "Hide collections from guests, show to logged-in customers",
      "hide_variant":    "Show bulk/wholesale variants only to tagged customers",
      "sub_email":       "Visitors subscribe to email to unlock access",
      "age_verify":      "Ask visitors to confirm their age before entering",
      "secret_link":     "Grant access exclusively via a shareable private link",
      "geo_lock":        "Restrict access to visitors from specific countries",
      "checkout_lock":   "Block checkout for guests or non-qualifying customers"
    }
  }
}
```

---

## 7. Files cần thay đổi

| File | Loại thay đổi |
|---|---|
| `constants/formPresets.const.js` | **Sửa** — thêm 7 template vào `RULES_TEMPLATE`, thêm 7 entry vào `TEMPLATE_DATA`, thêm `TEMPLATE_CATEGORIES` constant, thêm `description` field |
| `components/ui/FormPresets/index.jsx` | **Sửa** — render theo category, thêm description text, modal size large |
| `locales/en/locks.json` (hoặc i18n file tương ứng) | **Sửa** — thêm keys cho title + desc + category |
| `public/images/FormPresets/` | **Thêm** — 7 ảnh thumbnail mới |

---

## 8. Dependencies & Điểm cần confirm

| Vấn đề | Cần confirm |
|---|---|
| `geo_lock` template dùng `countries` condition | Condition type `countries` chưa implement — xem `location-condition-improvement.md`. Nếu chưa ready: dùng `regions` làm placeholder condition, update sau |
| Thứ tự ưu tiên template | Starter templates nên hiện trước hay B2B? Confirm với PM/Design |
| Ảnh thumbnail | Design team cần cung cấp 7 ảnh mới. Placeholder có thể dùng ảnh hiện có tạm |
| `handleTemplateNavigation` | Logic hiện tại đủ — chỉ cần đảm bảo template IDs mới không trùng với special cases (`checkout_lock`) |

---

## 9. Thứ tự implementation

```
[x] Bước 0: Confirm với Design xem có cần ảnh thumbnail trước khi code không
[ ] Bước 1: Thêm 7 template mới vào TEMPLATE_DATA (formPresets.const.js)
[ ] Bước 2: Fix sub_email — thêm vào RULES_TEMPLATE
[ ] Bước 3: Thêm category + description field vào tất cả items RULES_TEMPLATE
[ ] Bước 4: Thêm TEMPLATE_CATEGORIES constant
[ ] Bước 5: Cập nhật FormPresets/index.jsx — grouping + description + modal size
[ ] Bước 6: Thêm i18n keys
[ ] Bước 7: Đặt ảnh thumbnail (có thể dùng placeholder trước)
[ ] Bước 8: Test — chọn từng template mới → verify data pre-fill đúng trong wizard
[ ] Bước 9: Test backward compat — template cũ vẫn hoạt động bình thường
```

---

## 10. Out of Scope

- Backend thay đổi (không cần trừ `geo_lock` → phụ thuộc `location-condition-improvement`)
- Validate template data khi submit — giữ nguyên validation hiện tại
- Template search/filter (để sau nếu số lượng tiếp tục tăng)
- "Preview" từng template trước khi chọn
- Analytics per-template (ngoài Mixpanel event hiện có)
