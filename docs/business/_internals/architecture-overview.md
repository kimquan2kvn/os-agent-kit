---
title: Architecture Overview (Internal)
feature: _architecture
category: _internals
page_type: internals
audience: engineer
summary: >-
  B2B Lock spans four repos (cms host, legacy Koa API, login-api-v2 NestJS, script extensions) over one shared MySQL DB; v1/v2 are chosen per shop via client_version (Redis-cached); flows A–E are all present, with metafields for v2 and theme-file injection for v1/LTSP.
keywords: [architecture, repos, client_version, v1 v2 routing, app proxy, webhooks, shopify functions, scopes, two backends, login-api-v2, legacy api, redis version]
related: [locks-internals, pricing-internals, checkout-lock-internals, legacy-restrictions-internals]
metafield_keys: [login.configs, bss_ltap.hide-accelerated-checkout, bss_ltap.hide-section-configs, bss-login-configs.hide-cart, bss_lock_shop_data, "app:cart-checkout-validation.function-configuration", "$app:payment-customization.configs", "$app:delivery-customization.configs"]
source_refs:
  repos: [b2b-login-access-management-cms, b2b-login-access-management-api, login-api-v2, b2b-login-access-management-script]
  paths: ["web/index.js", "web/afterAuth.js", "src/modules/shopify/**", "shopify.app.toml.example"]
  symbols: [getClientVersion, updateRedisVersion, registerWebhookByVersion, VerifyAndValidateAppProxyGuard]
---
# Architecture Overview (Internal)

> **Engineer-facing.** Repo seams, flows, and the v1↔v2 split.

## Repo map

| Repo | Role | Stack |
| --- | --- | --- |
| `b2b-login-access-management-cms` | OAuth host, admin UI (v1 `frontend/` + v2 `client/`), version-routing proxy | Node/Express, React, App Bridge |
| `b2b-login-access-management-api` | Legacy API (v1 shops) | Node/Koa, Sequelize, MySQL |
| `login-api-v2` | Core API (v2 shops + shared infra) | NestJS, TypeORM, MySQL |
| `b2b-login-access-management-script` | Storefront theme-app-extension assets + Shopify Functions | Liquid/JS; TS Functions |

**Shared DB:** MySQL 8.0 `login_to_access_pages`. Both APIs connect to it; `shops` is the join key. There is **no** DB sharding by version.

## Flows in use (A–E)

- **A — OAuth install:** CMS `afterAuth.js` upserts the shop and calls `registerWebhookByVersion` against **both** `VITE_SERVER_URL` (legacy) and `VITE_SERVER_URL_V2` (v2).
- **B — App proxy:** `apps/bss-b2b-lock/*` → `login-api-v2 /app_proxy`. `GET /app_proxy/ping?locking-conditions=…` resolves `specific_ip` + `regions` live (HMAC-guarded by `VerifyAndValidateAppProxyGuard`).
- **C — Theme asset injection:** on rule save / theme publish, snippets/wrappers written via REST Assets API + GraphQL `onlineStoreThemeFilesUpsert`.
- **D — Webhooks:** `THEMES_PUBLISH`, `APP_UNINSTALLED`, `SHOP_UPDATE`, `COLLECTIONS_DELETE` (+ GDPR `customers/redact`, `customers/data_request`, `shop/redact`). v2 handlers gate on `client_version === 'v2'`.
- **E — Shopify Functions:** `cart.validations.generate.run`, `cart.payment-methods.transform.run`, `cart.delivery-options.transform.run` in the script repo.

## Scopes & app config

From `shopify.app.toml.example` + CMS `.env.example`:

- **Scopes:** `write_themes, write_content, read_products, read_customers, write_customers, read_markets, write_markets`.
- **App proxy:** `url = $SERVER_URL_V2/app_proxy`, `subpath = bss-b2b-lock`, `prefix = apps`.
- **API version:** `2025-01` (configurable).
- **GDPR webhooks** declared in toml → legacy API.
- No `metafield_definitions` in toml; definitions managed programmatically via `MetafieldsService`.

## v1 ↔ v2 routing

`client_version VARCHAR DEFAULT 'v1'` on `shops`. CMS `web/index.js` decision:

1. Read Redis key `${shop}:shop_version`.
2. On miss → `POST $VITE_SERVER_URL/shop/update-redis-version` (`updateRedisVersion` reads DB, caches to Redis).
3. `'v2'` → serve `client/` build; else serve `frontend/` build.

v2 shops route admin calls to `login-api-v2`; v1 shops to the legacy Koa API. The legacy `verifyVersion` middleware gates v1-only paths. Legacy Google Search Console route is hard-closed after `2024-11-20`.

## Metafield model

See [Metafield Contracts](../reference/metafield-contracts.md) for the full table. Summary of namespaces: `login`, `bss_ltap`, `bss-login-configs`, `bss_lock_shop_data` (shop/app config) and `app:cart-checkout-validation`, `$app:payment-customization`, `$app:delivery-customization` (Functions). **v1 modules + LTSP use no metafields** — they inject Liquid into theme files.

## External integrations

- **Brevo** — lifecycle event tracking + transactional email (passcode notifications).
- **Google** — OAuth2 + Search Console (HPOGS; deprecated in legacy path).
- **Geonames** (`geolocation`) — region/country autocomplete in admin; runtime IP→country from proxy headers (`ctx.ipGeo`).
- **Mixpanel / Sentry** — analytics + error monitoring. **Mattermost / Shoffi** — internal install notifications / affiliate.

## Cross-repo seams

- Admin→backend: `VITE_SERVER_URL` (v1) / `VITE_SERVER_URL_V2` (v2), JWT (`ShopGuard`).
- App proxy: `apps/bss-b2b-lock` → `/app_proxy`, HMAC.
- Metafield keys in `config-header.liquid` must match `MetafieldsService` writes.
- Function metafield keys must match the writers in `login-api-v2`.
- Webhooks dual-registered; each server gates by `client_version`.

## Data model

Key v2 entities (TypeORM): `Shop`, `Config`, `Rules` (`rules-v2`), `Keys`, `Conditions`, `Analytic`, `Behavior`, `CheckoutRules`, `PaymentRules`, `ShippingCustomization`, `ThemeInstallations`, `DesignV2`, `UsageHistoryV2`, `PasscodeRequest`, `TemplateEmailPasscode`, `Onboarding`, `GoogleAuth`, `Nps`, `Support`. Legacy (Sequelize): `Rule` + `rule*` join tables, `Module`/`ShopModule`, `Plan`/`PlanLog`, billing tables, etc. — same DB.
