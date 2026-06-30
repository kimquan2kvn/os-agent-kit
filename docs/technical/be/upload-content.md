# Upload content to Shopify theme (Liquid/CSS)

## Mục tiêu
Module upload chịu trách nhiệm “install/refresh” code vào Shopify theme:
- Upload snippets Liquid (từ `src/liquid/`)
- Chèn include/link vào `layout/theme.liquid`
- Upload CSS settings (`assets/bss-lock-settings.css`)
- Mutate theme files qua Shopify REST assets API và Shopify GraphQL `themeFilesUpsert` (batch)

## API endpoints
Controller: `src/modules/upload/upload.controller.ts` (`@Controller('upload-content')`)

- `POST /upload-content/all`
  - Lấy `themeId` qua `ThemeService.getThemeId(..., ignore=false)`
  - Gọi `UploadService.uploadContent(shop, themeId)`
  - Update theme installation record (`ThemeInstallationService.createOrUpdateThemeInstallation`)

- `POST /upload-content/force-upload-content` → `UploadService.forceUploadContent(shop)`
- `POST /upload-content/upload-to-theme` (body: `{ themeId }`) → upload sang theme id cụ thể
- `POST /upload-content/rules` → chỉ upload phần liên quan rules (`UploadService.uploadRulesContent`)
- `POST /upload-content/lock-element` → inject code “lock element” (themeId thường lấy với `ignore=true`)
- `POST /upload-content/upload-metaobject` → đồng bộ metaobject/design meta
- `POST /upload-content/upload-display-settings` → build + upload CSS/settings
- `POST /upload-content/restore` → restore code theo `themeIds[]`
- `POST /upload-content/banner-trail-expried` → storefront call (guard riêng)
- `POST /upload-content/subscription` → guarded bởi `VerifyServerLoginV1Guard`
- `POST /upload-content/update-shop-trail` → `@Public()` (có cutoff date trong code)

## Kiến trúc & file chính
- Orchestrator: `src/modules/upload/upload.service.ts`
- Shopify theme operations (REST assets): `src/modules/shopify/theme/theme.service.ts`
- Shopify theme operations (GraphQL batch upsert):
  - `src/graphql/theme/themeFilesUpsert.ts` (`uploadThemeFiles`, chunk size = 50)
  - `src/graphql/theme/getThemeFilesContent.ts` (fetch content)
- Liquid templates nguồn: `src/liquid/*.liquid`

## Luồng upload display settings (ví dụ quan trọng)
Trong `UploadService.uploadDisplaySettings(...)`:
- Tổng hợp style cho từng rule/design → build CSS lớn
- Upload thành asset `assets/bss-lock-settings.css`
- Đảm bảo `layout/theme.liquid` include stylesheet tag:
  - `{{ 'bss-lock-settings.css' | asset_url | stylesheet_tag }}`

## Convention: batch upload theme files
- Khi sửa nhiều file theme, ưu tiên dùng `uploadThemeFiles(domain, token, themeId, files)`
  - Hàm này tự chunk theo `CHUNK_SIZE = 50`
  - Tránh gọi upsert từng file 1 để giảm rate-limit và tăng tốc

## Hide Block/Section (liên quan upload)
Feature này sửa trực tiếp nội dung file theme và upload lại:
- Doc chi tiết: `docs/features/hide-section.md`
- Code chính:
  - `src/modules/upload/hide-content-page/hide-section.service.ts`
  - `src/graphql/theme/themeFilesUpsert.ts`

Quan trọng: logic injection dựa vào marker comment `{%comment%}Bss Hide Section{%endcomment%}` để đảm bảo idempotency (không inject trùng).
