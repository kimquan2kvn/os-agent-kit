---
title: Set up Pricing (Login to See Price)
feature: pricing
abbrev: LTSP
category: pricing
page_type: setup
audience: merchant
summary: >-
  To set up Login to See Price, choose whether to hide price and/or add-to-cart, choose which customers are restricted and which products are affected, customize the replacement message, and save to inject price wrappers into the theme.
keywords: [set up login to see price, configure LTSP, hide price setup, restrict by tag, apply to products, price message]
related: [pricing, on-the-storefront, settings-and-translations]
source_refs:
  repos: [b2b-login-access-management-cms, b2b-login-access-management-api]
  paths: ["web/frontend/pages/price-restrictions/**", "src/services/module/ltsp.service.ts"]
  symbols: [ltsp.service]
---
# Set up Pricing (Login to See Price)

This page covers configuring **Login to See Price (LTSP)**. The exact admin location depends on your store's generation: in v2 use a [Lock](../locks/set-up-locks.md) with a price target; in the legacy admin use **Price restrictions → Login to See Price**.

## 1. Choose what to hide

- **Hide price and add-to-cart** (`login_to_see_price_action = 0`) — shoppers can't see prices or buy.
- **Hide add-to-cart only** (`login_to_see_price_action = 1`) — prices show, but purchase is blocked until they qualify.

## 2. Choose who is restricted

Set `apply_to_customer_type`:

- Restrict everyone, guests only, logged-in only, by tag, or by specific customers (allow or restrict). See the value list in [Pricing](pricing.md#who-is-restricted-apply_to_customer_type).

If you choose a tag-based or customer-specific mode, provide the tag list or customer list.

## 3. Choose which products

Set `apply_to_product_type` / `all_products`:

- All products, by tag, by specific products, or by collection. Provide the corresponding selection.

## 4. Customize the replacement message

Edit the text shown in place of the hidden price (e.g. "Log in to see prices"), and optionally a login link. Per-locale text is managed under [Settings → Translations](../analytics-and-settings/settings-and-translations.md).

## 5. Save

On save, the app **injects Liquid price wrappers into your theme** (price templates, sections, and snippets) via the Shopify Asset API, and stores the configuration in its database. There are no Shopify metafields involved for LTSP.

> Because LTSP edits theme files, switching or republishing a theme can require a re-sync. If prices reappear after a theme change, re-save the rule or re-run the app's theme install.

## 6. Verify on the storefront

Open a product page as a guest (or a non-qualifying customer) and confirm the price/cart button is hidden and the message shows. See [On the storefront](on-the-storefront.md).

## Related pricing options

- **Passcode to View Price** — require a passcode instead of login.
- **Hide Price on Google Search** — prevent locked prices from leaking into Google.
- **Hide Variants** — hide specific variants entirely.

These share the LTSP engine; see [Pricing](pricing.md#related-pricing-modules).
