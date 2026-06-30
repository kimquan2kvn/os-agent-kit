---
title: Pricing (Login to See Price)
feature: pricing
abbrev: LTSP
category: pricing
page_type: overview
audience: merchant
summary: >-
  Pricing (Login to See Price) hides product prices and optionally the add-to-cart button until a shopper logs in or meets conditions, replacing the price with a message or login prompt.
keywords: [login to see price, LTSP, hide price, hide add to cart, B2B pricing lock, request price, passcode to view price, hide price on google]
related: [locks, checkout-lock, plans-and-pricing]
source_refs:
  repos: [b2b-login-access-management-api, b2b-login-access-management-script, login-api-v2]
  paths: ["src/services/module/ltsp.service.ts", "src/ltsp-js/**", "extensions/theme-app-extension/assets/bss-ltsp-v2.min.js"]
  symbols: [ltsp, wrapPricePrefix, wrapPriceSuffix, "bsscommerce-login-to-see-price"]
---
# 💲 Pricing (Login to See Price)

**Login to See Price (LTSP)** hides product prices — and optionally the add-to-cart button — from shoppers who aren't logged in or don't meet the conditions you set. Instead of the price they see a message or a prompt to log in. It's the classic wholesale/B2B pattern: browse the catalog, but you must be an approved customer to see prices and buy.

> In v2 you can achieve price hiding with a [Lock](../locks/locks.md) whose target is `PRICE`, `CART_BTN`, or `PRICE_CART_BTN`. The dedicated **LTSP module** described here is the established (legacy-API-backed) pricing engine that injects price wrappers directly into the theme. Both are documented because stores use both depending on `client_version`.

## What it does

- **Hides the price** on product pages, collection/listing cards, search results, and related products.
- **Optionally hides the add-to-cart button** so shoppers can't purchase without qualifying.
- **Shows a replacement** — a message ("Log in to see prices") or a login link — in place of the price.

## Action modes (`login_to_see_price_action`)

- `0` — hide **both** the price and the add-to-cart button.
- `1` — hide the **add-to-cart button only** (price stays visible).

## Who is restricted (`apply_to_customer_type`)

LTSP decides which shoppers get the price hidden:

- `0 (RestrictAll)` — hide for everyone.
- `1 (AllowByTags)` — show only to customers with allowed tags.
- `2 (RestrictNonLogin)` — hide for guests only (logged-in customers see prices).
- `3 (RestrictByTags)` — hide for customers with specific tags.
- `4 (RestrictSpecific)` — hide except for specific customers.
- `5 (AllowSpecific)` — show only to specific customers.
- `6 (RestrictLogged)` — hide for logged-in customers.

## Which products (`apply_to_product_type` / `all_products`)

- `1 (AllowAll)` — apply to all products.
- `2 (ByTags)` — products with selected tags.
- `3 (ByProducts)` — selected products.
- `4 (ByCollections)` — products in selected collections.

## Related pricing modules

The legacy API also ships pricing variants that share this engine:

- **Hide Price (HPC / hidePC)** — hide prices for products/collections.
- **Passcode to View Price (PTVP)** — require a passcode to reveal the price.
- **Hide Price on Google Search (HPOGS)** — keep locked prices out of Google Shopping/Search.
- **Hide Variants (HV)** — hide specific variants.

These are documented at a high level in the [Settings & Values Glossary](../reference/settings-and-values-glossary.md) and the [internals](../_internals/pricing-internals.md).

## How it's applied

Unlike v2 Locks (which use metafields + app embed), LTSP works by **injecting Liquid wrappers into your theme's price templates** via the Shopify Asset API, plus storefront JS that replaces the rendered price. There are **no Shopify metafields** for LTSP — configuration lives in the app's MySQL database and the injected theme code. See [On the storefront](on-the-storefront.md) and the [internals](../_internals/pricing-internals.md).

## Next steps

- [Set up Pricing](set-up-pricing.md)
- [On the storefront](on-the-storefront.md)
