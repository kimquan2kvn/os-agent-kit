---
title: Pricing / LTSP Internals
feature: pricing
category: _internals
page_type: internals
audience: engineer
summary: >-
  LTSP (rule_type 2) and its variants (hidePC 4, PTVP 6, HPOGS, HV 8) live in the legacy API, use no Shopify metafields, and enforce by injecting Liquid price wrappers into theme files via the Asset API plus storefront JS that rewrites price innerHTML; config lives in MySQL.
keywords: [ltsp internals, login to see price, rule_type 2, hidePC, PTVP, HPOGS, hide variants, asset api, price wrapper, bsscommerce-login-to-see-price, no metafields, bss-ltsp-v2]
related: [architecture-overview, pricing, legacy-restrictions-internals]
source_refs:
  repos: [b2b-login-access-management-api, b2b-login-access-management-script]
  paths: ["src/services/module/ltsp.service.ts", "src/services/module/hidePC.service.ts", "extensions/theme-app-extension/assets/bss-ltsp-v2.min.js"]
  symbols: [ltsp, wrapPricePrefix, wrapPriceSuffix, productPriceTemplateFiles, "bss-ltsp-v2.min.js"]
---
# Pricing / LTSP Internals

> **Engineer-facing.** The pricing modules are theme-file injectors, not metafield consumers.

## Modules and rule_type

- **LTSP** (Login to See Price) — `rule_type = 2`.
- **HP / hidePC** (Hide Price) — `rule_type = 4`.
- **PTVP** (Passcode to View Price) — `rule_type = 6`.
- **HV** (Hide Variants) — `rule_type = 8`.
- **HPOGS** (Hide Price on Google Search) — internal module code `8`.

All live in `b2b-login-access-management-api` (legacy Koa).

## No metafields

LTSP writes **nothing** to Shopify metafields. Configuration is in MySQL (`rules` table with `rule_type` discriminator + join tables). Enforcement is in injected theme code.

## Enforcement: theme-file injection

On save, LTSP injects Liquid wrappers (`wrapPricePrefix` / `wrapPriceSuffix`) around price output in the theme's price files — `productPriceTemplateFiles`, `productPriceSectionFiles`, `productPriceSnippetFiles` — via the Shopify Asset API. Each wrap renders the snippet `bsscommerce-login-to-see-price`.

## Storefront JS

`bss-ltsp-v2.min.js` replaces price element innerHTML for dynamically rendered prices. Companions: `bss-ltsp-process-atc.min.js` (ATC handling), `bss-ltsp-custom.min.js` (per-store customizations; `dev-ltsp-custom.js` source). Listing prices use `search.js?...&view=bss.login`.

## Key enums

- `apply_to_customer_type`: `0 RestrictAll`, `1 AllowByTags`, `2 RestrictNonLogin`, `3 RestrictByTags`, `4 RestrictSpecific`, `5 AllowSpecific`, `6 RestrictLogged`.
- `login_to_see_price_action`: `0` hide price + ATC, `1` ATC only.
- `apply_to_product_type` / `all_products`: `1 AllowAll`, `2 ByTags`, `3 ByProducts`, `4 ByCollections`.

## Edge cases

- Theme switch/duplicate drops the wrappers → re-save / re-install required.
- Uninstall leaves wrappers in theme files (manual cleanup).
- Hardcoded per-store customizations exist in `hidePC.service.ts` and `dev-ltsp-custom.js` (domain-keyed branches).
