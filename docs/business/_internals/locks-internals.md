---
title: Locks Internals (admin → backend → metafield → JS → DOM)
feature: locks
category: _internals
page_type: internals
audience: engineer
summary: >-
  v2 Locks store rules in rules-v2 with ruleKeys (OR) and ruleConditions (AND); saveRule via /api/rules compiles theme snippets and writes app metafields; storefront evaluates conditions mostly in Liquid (config-header.liquid → window.Login, bss-lock-condition.liquid) with JS for element hiding and app-proxy for IP/region.
keywords: [locks internals, rules-v2, ruleKeys, ruleConditions, saveRule, /api/rules, config-header.liquid, window.Login, normalizeCondition, file sharding, bss-lock-condition.liquid]
related: [architecture-overview, locks, conditions-reference]
metafield_keys: [login.configs, bss_ltap.hide-section-configs, bss_ltap.hide-accelerated-checkout, bss-login-configs.hide-cart]
source_refs:
  repos: [login-api-v2, b2b-login-access-management-script]
  paths: ["src/modules/rules/**", "extensions/theme-app-extension/**"]
  symbols: [Rules, Keys, Conditions, RuleService, "/api/rules", config-header.liquid, normalizeCondition]
---
# Locks Internals

> **Engineer-facing** trace of a v2 Lock from admin to DOM.

## Data model

- `rules-v2` (`Rules`) — one row per Lock. Columns include `domain_id`, `priority`, `status`, `action_type` (0 LOCK / 1 HIDE), `target_type` (0/1/2/3/4/5/6/8/9/10/11), target id arrays (`product_ids`, `collection_ids`, `page_ids`, `variant_ids`, `variant_options`), `login_type`, `fl_redirect_url`, `fl_redirect_type`, `section_block_native`, `section_block_css_selector`, `show_countdown_clock`.
- `ruleKeys` (`Keys`) — OR groups, FK → `rules-v2`.
- `ruleConditions` (`Conditions`) — `type`, `inverse`, `configs` (JSON), FK → `ruleKeys`. **AND within a key, OR across keys.**
- `ruleAdvanced`, `ruleTranslation`, `DesignV2` — advanced settings, per-locale text, design.

Routes under `/api/rules`: `getAllRules`, `getRulesByShop`, `saveRule`, `deleteRule`, `bulkEditRule`, `save-translation`, `clear-google-cache`.

## Write path (saveRule)

`POST /api/rules` → `RuleService.saveRule`:
1. Persist rule + keys + conditions.
2. Compile theme snippets via `UploadService.uploadContentNew`.
3. Write app/shop metafields: `login.configs` (selectors / JS-process), `bss_ltap.hide-section-configs` (sections), `bss_ltap.hide-accelerated-checkout`, `bss-login-configs.hide-cart`.

## Storefront read/execute path

1. App Embed renders `config-header.liquid` into `<head>` → exposes `window.Login` and reads the app metafields.
2. Condition evaluation is **server-side Liquid** (`bss-lock-condition.liquid`) for login/tags/identity/dates/passcode(cart attr)/secret-link/URL-param/subscribe/market.
3. **Element-level** locks (price/cart/section) use theme-app-extension JS to hide DOM; price element hiding fetches `search.js?...&view=bss.login` for listing pages.
4. **IP/region** conditions call `GET /app_proxy/ping?locking-conditions=…` (login-api-v2) which resolves against `ctx.ip` / `ctx.ipGeo` and returns satisfied conditions.

## normalizeCondition

For element-level locks the engine strips whole-page-only conditions that don't make sense at element scope before compiling.

## Performance: file sharding

Compiled storefront config is sharded at ~250 KB to avoid a single oversized theme asset.

## Metaobjects

Translations and design are stored in metaobjects `bss_lock__metaobject` (translations) and `bss_ltap__message_login_styleh24` (design) in addition to DB entities.
