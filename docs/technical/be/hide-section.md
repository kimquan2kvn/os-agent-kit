# Hide Block & Section Feature - Architecture & Quick Reference

## 1. Feature Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        API Request                           │
│            POST /api/rules (target_type: 11)                 │
└────────────────────────┬────────────────────────────────────┘
                         │
                    ┌────▼─────┐
                    │ Validate  │◄─── SectionBlockRuleValidator
                    │   Input   │
                    └────┬─────┘
                         │
        ┌────────────────▼────────────────┐
        │      Save to Database           │
        │    rules-v2 table with:         │
        │  - section_or_block_type        │
        │  - section_block_native (JSON)  │
        │    [{ path_file, name, type }]  │
        │  - section_block_css_selector   │
        └────────────────┬────────────────┘
                         │
            ┌────────────▼────────────────────┐
            │  buildAndUploadContentFileHide   │
            │         Section()               │
            └────────────┬────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                                 │
        ▼                                 ▼
   NATIVE (0)                       CUSTOM_CSS (1)
   type: block_in_section            CSS selector-based
   → injectInsideBlock()             → hideSectionByCSS()
        │                            + metafield config
   type: section / snippet
   → injectAtFileStart()
   → injectAtFileEnd()
        │
        ▼
   Upload modified theme files directly
   via uploadThemeFiles() (Shopify GraphQL)
```

---

## 2. Data Flow — Native Section Injection

```
Rule with section_block_native:
[{ path_file: 'sections/header.liquid', name: '', type: 'section' },
 { path_file: 'sections/featured-product.liquid', name: 'price', type: 'block_in_section' }]
        │
        ▼
Group blocks by path_file (Map<string, SectionBlock[]>)
        │
        ▼
For each path_file → fetch current theme file content via GraphQL
        │
        ├─► type === 'block_in_section'
        │       subject key = '${blockName}__${pathFile}'
        │       e.g. 'price__sections/featured-product.liquid'
        │       Strategy 1: Find {% when 'blockName' %} → wrap its content
        │       Strategy 2: Find {% if block.type == 'blockName' %} → wrap its content
        │
        └─► type === 'section' | 'snippet' | 'block'
                subject key = pathFile
                e.g. 'sections/header.liquid'
                → injectAtFileStart()  — wrap entire file from top
                → injectAtFileEnd()    — close before {% schema %} / {% javascript %} / {% stylesheet %}
        │
        ▼
uploadThemeFiles() — push all modified files to Shopify theme in one batch
```

---

## 3. Native Section — Liquid Injection Templates

Injection code được **cài trực tiếp vào file theme** (sections/snippets), KHÔNG tạo file mới.

### 3.1 Injection START (đầu file hoặc đầu block)

```liquid
{%comment%}Bss Hide Section{%endcomment%}
{% capture bss_lock_check %}
  {% render 'bss-lock-condition', scope: 'section', subject: '<SUBJECT_KEY>', variable: 'hide_section' %}
{% endcapture %}
{% unless bss_lock_check contains 'true' %}
{%comment%}Bss Hide Section{%endcomment%}
```

### 3.2 Injection CLOSING (cuối file hoặc cuối block)

```liquid
{%comment%}Bss Hide Section{%endcomment%}
{% else %}
  {% render 'bss-lock-condition', scope: 'section', subject: '<SUBJECT_KEY>', variable: 'hide_section', bss_lock_action: 'section_replacement' %}
{% endunless %}
{%comment%}Bss Hide Section{%endcomment%}
```

### 3.3 Giá trị `subject` theo loại rule

| `type` | `subject` key | Ví dụ |
|--------|--------------|-------|
| `section` / `snippet` / `block` | `pathFile` | `sections/header.liquid` |
| `block_in_section` | `${blockName}__${pathFile}` | `price__sections/featured-product.liquid` |

> **Lý do**: Khi hide block in section, cần phân biệt từng block riêng lẻ trong cùng một file.  
> `bss-lock-condition` sẽ dùng `subject` để tra cứu rule tương ứng trong metafield.

---

## 4. Native Section — Injection Strategies

### 4.1 Hide toàn bộ file (type: `section` | `snippet` | `block`)

```
File: sections/header.liquid
─────────────────────────────────────────────
▼ injectAtFileStart()
{%comment%}Bss Hide Section{%endcomment%}
{% capture bss_lock_check %}{% render 'bss-lock-condition', scope: 'section', subject: 'sections/header.liquid', variable: 'hide_section' %}{% endcapture %}
{% unless bss_lock_check contains 'true' %}
{%comment%}Bss Hide Section{%endcomment%}

... original file content ...

▼ injectAtFileEnd()  ← chèn VÀO TRƯỚC {% schema %} / {% javascript %} / {% stylesheet %}
{%comment%}Bss Hide Section{%endcomment%}
{% else %}{% render 'bss-lock-condition', scope: 'section', subject: 'sections/header.liquid', variable: 'hide_section', bss_lock_action: 'section_replacement' %}{% endunless %}
{%comment%}Bss Hide Section{%endcomment%}

{% schema %}
  ...
{% endschema %}
─────────────────────────────────────────────
```

**Thứ tự ưu tiên chèn closing tag** (`injectAtFileEnd`):
1. Trước `{% stylesheet %}` hoặc `{% javascript %}` (lấy vị trí nhỏ hơn nếu cả hai đều có)  
2. Trước `{% schema %}`
3. Cuối file nếu không có tag nào ở trên

### 4.2 Hide block in section (type: `block_in_section`)

Subject key = `${blockName}__${pathFile}`

#### Strategy 1 — `{% when %}` tag (Liquid case/when)

```liquid
{% case block.type %}
  {% when 'price' %}
    ← START injection chèn vào đây
    {%comment%}Bss Hide Section{%endcomment%}
    {% capture bss_lock_check %}{% render 'bss-lock-condition', scope: 'section', subject: 'price__sections/featured-product.liquid', variable: 'hide_section' %}{% endcapture %}
    {% unless bss_lock_check contains 'true' %}
    {%comment%}Bss Hide Section{%endcomment%}

    ... original block content ...

    {%comment%}Bss Hide Section{%endcomment%}
    {% else %}{% render 'bss-lock-condition', scope: 'section', subject: 'price__sections/featured-product.liquid', variable: 'hide_section', bss_lock_action: 'section_replacement' %}{% endunless %}
    {%comment%}Bss Hide Section{%endcomment%}
    ← END injection kết thúc trước {% when %} tiếp theo hoặc {% endcase %}

  {% when 'title' %}
    ...
{% endcase %}
```

#### Strategy 2 — `{% if block.type == '...' %}` tag (fallback nếu Strategy 1 không match)

```liquid
{% if block.type == 'price' %}
  ← START injection
  {%comment%}Bss Hide Section{%endcomment%}
  {% capture bss_lock_check %}{% render 'bss-lock-condition', scope: 'section', subject: 'price__sections/featured-product.liquid', variable: 'hide_section' %}{% endcapture %}
  {% unless bss_lock_check contains 'true' %}
  {%comment%}Bss Hide Section{%endcomment%}

  ... original block content ...

  {%comment%}Bss Hide Section{%endcomment%}
  {% else %}{% render 'bss-lock-condition', scope: 'section', subject: 'price__sections/featured-product.liquid', variable: 'hide_section', bss_lock_action: 'section_replacement' %}{% endunless %}
  {%comment%}Bss Hide Section{%endcomment%}
  ← END injection kết thúc trước {% endif %}
{% endif %}
```

> Cũng hỗ trợ reversed form: `{% if 'price' == block.type %}`

---

## 5. Idempotency & Migration

### Idempotency (không inject trùng)

- **File-level START**: Kiểm tra nếu file đã bắt đầu bằng `{%comment%}Bss Hide Section{%endcomment%}` → skip
- **File-level END**: Kiểm tra nếu `closingTag` đã tồn tại trong content → skip
- **Block-level**: Kiểm tra regex `{% when 'name' %}[\s\S]*?{%comment%}Bss Hide Section` → skip nếu đã inject

### Old pattern migration

Nếu tìm thấy pattern cũ (không có `subject` param) trong closing tag:

```liquid
{%comment%}Bss Hide Section{%endcomment%}{% else %}{% render 'bss-lock-condition', scope: 'section', variable: 'section_replacement' %}{% endunless %}{%comment%}Bss Hide Section{%endcomment%}
```

→ Tự động xóa pattern cũ và thay bằng pattern mới có `subject`.

---

## 6. Quick Decision Tree

```
User wants to hide a block/section
        │
        ├─► Can you target it with CSS selector?
        │   ├─ YES ──► Use CUSTOM_CSS (type: 1)
        │   │          Input: section_block_css_selector
        │   │          → hideSectionByCSS() inject vào theme.liquid
        │   │          → buildAndPushHideSectionMessageConfig() lưu config vào metafield
        │   │
        │   └─ NO ──► Use NATIVE (type: 0)
        │              section_block_native: [{ path_file, name, type }]
        │              │
        │              └─► type của block?
        │                  ├─ 'section'         ──► inject toàn bộ file (subject = pathFile)
        │                  ├─ 'snippet'         ──► inject toàn bộ file (subject = pathFile)
        │                  ├─ 'block'           ──► inject toàn bộ file (subject = pathFile)
        │                  └─ 'block_in_section'──► inject vào when/if block
        │                                          (subject = blockName__pathFile)
```

---

## 7. File Modification Map

Khi rule NATIVE được upload, service **sửa trực tiếp các file theme** sau:

### For `block_in_section`
```
sections/featured-product.liquid   ◄── inject bao quanh {% when 'price' %} block
sections/main-product.liquid       ◄── inject bao quanh {% if block.type == 'badge' %}
```

### For `section` / `snippet` / `block`
```
sections/header.liquid             ◄── inject bao quanh toàn bộ content, trước {% schema %}
snippets/product-card.liquid       ◄── inject bao quanh toàn bộ content (append cuối file)
```

### For `CUSTOM_CSS` (type: 1)
```
layout/theme.liquid                ◄── hideSectionByCSS() inject CSS snippet vào trước </head>
```
Config (selectors + message HTML) được lưu vào Shopify metafield qua `setHideSectionBlockConfig()`.

---

## 8. CUSTOM_CSS — Injection Detail

### 8.1 Snippet được inject vào `theme.liquid`

```liquid
{%comment%}Bss Hide Section{%endcomment%}
{% capture bss_lock_check %}
  {% render 'bss-lock-condition', scope: 'section', variable: 'hide_section_css' %}
{% endcapture %}
{{ bss_lock_check }}
{%comment%}Bss Hide Section{%endcomment%}
```

> **Lưu ý**: CSS snippet này **không có `subject`** — `bss-lock-condition` xử lý tất cả rules CSS theo `variable: 'hide_section_css'` và tự tra cứu selector theo metafield config.

### 8.2 Vị trí chèn trong `theme.liquid` (fallback order)

| Priority | Điều kiện | Hành động |
|----------|-----------|-----------|
| 1 | Tìm thấy `</head>` | Chèn **trước** `</head>` |
| 2 | Tìm thấy `<main>` | Chèn **trước** `<main>` |
| 3 | Tìm thấy `<body>` | Chèn **trước** `<body>` |
| 4 | Không tìm thấy gì | Append cuối file |

### 8.3 Idempotency cho CUSTOM_CSS

Kiểm tra nếu `contentThemeLiquid` đã chứa `{%comment%}Bss Hide Section{%endcomment%}` → **skip**, không inject lại.

### 8.4 Config metafield schema

`buildAndPushHideSectionMessageConfig()` lấy toàn bộ rule active (`status: 1, target_type: 11, section_or_block_type: CUSTOM_CSS`) và ghi vào metafield qua `setHideSectionBlockConfig()`:

```typescript
// Mỗi rule tạo ra 1 config entry:
{
  ruleId: rule.id,
  selectors: rule.section_block_css_selector,  // CSS selector string
  messageHtml: string  // '' nếu show_message_hide_section = false
}

// messageHtml được build từ rule.design.hide_section_block:
// <p class="bss-hide-section-message"
//    data-bss-lock-rule-id="{rule.id}"
//    style="font-size:{fontSize}px; font-style:{italic|normal}; font-weight:{bold|normal};
//           text-decoration:{underline|none}; color:{textColor}; text-align: center;">
//   {message}
// </p>
```

---

## 9. Uninstall / Cleanup

### 9.1 Remove native injection (`removeInjection`)

Xóa tất cả injection blocks ra khỏi nội dung file theme. Pattern regex:

```
{%comment%}Bss Hide Section{%endcomment%}
... bất kỳ nội dung gì ...
{%comment%}Bss Hide Section{%endcomment%}
```

→ Xóa toàn bộ đoạn giữa 2 comment markers (bao gồm cả markers).

**Dùng khi**: Rule bị xóa hoặc `target_type` thay đổi, service cần restore file theme về trạng thái gốc.

### 9.2 Remove CSS injection (`removeHideSectionCSS`)

Xóa snippet CUSTOM_CSS ra khỏi `layout/theme.liquid` bằng cách replace đúng chuỗi constant `BSS_HIDE_SECTION_CSS`.

---

## 10. UploadResult Tracking

Service nhận tham số `uploadResult?: UploadResult` để track kết quả install từng file:

```typescript
uploadResult.installHideSection.push({
  fileName: pathFile,           // e.g. 'sections/header.liquid'
  blockInstall?: {
    blockName: string,          // '' nếu là file-level injection
    type: 'block' | 'file',
  },
  success: boolean,
  reason: '' | 'file-not-found' | 'processing-error'
});
```

---
