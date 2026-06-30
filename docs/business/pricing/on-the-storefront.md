---
title: Pricing on the Storefront
feature: pricing
abbrev: LTSP
category: pricing
page_type: storefront
audience: merchant
summary: >-
  On the storefront, Login to See Price injects Liquid price wrappers into the theme that render a replacement snippet, and storefront JS (bss-ltsp-v2.min.js) replaces the rendered price innerHTML and removes add-to-cart for non-qualifying shoppers.
keywords: [login to see price storefront, hide price runtime, price wrapper, bss-ltsp-v2, bsscommerce-login-to-see-price, asset api, search.js view bss.login]
related: [pricing, set-up-pricing, locks]
source_refs:
  repos: [b2b-login-access-management-script, b2b-login-access-management-api]
  paths: ["extensions/theme-app-extension/assets/bss-ltsp-v2.min.js", "src/services/module/ltsp.service.ts"]
  symbols: [wrapPricePrefix, wrapPriceSuffix, "bsscommerce-login-to-see-price", "bss-ltsp-v2.min.js"]
---
# Pricing on the Storefront

This page explains how **Login to See Price** actually hides prices for shoppers and what they see in its place.

## How prices get wrapped

When you save an LTSP rule, the app edits your theme directly (Shopify Asset API). It wraps the theme's price output with a prefix/suffix (`wrapPricePrefix` / `wrapPriceSuffix`) so every place a price renders also renders the app's snippet **`bsscommerce-login-to-see-price`**. The wrappers are injected into the theme's price templates, sections, and snippets (the app maintains lists of these files).

## What the shopper sees

For a shopper who does **not** qualify (per `apply_to_customer_type`):

- The **price is replaced** by your message (e.g. "Log in to see prices") or a login link.
- If the action mode hides the cart button, the **add-to-cart button is removed/disabled** so they can't purchase.

For a shopper who **qualifies**, prices and the cart button render normally.

## The role of the storefront JS

The script **`bss-ltsp-v2.min.js`** runs in the browser to handle dynamically rendered prices — it **replaces the price element's innerHTML** with the replacement content where the server-side wrapper isn't sufficient (e.g. AJAX-loaded cards). Companion scripts handle add-to-cart processing (`bss-ltsp-process-atc.min.js`) and any per-store customizations (`bss-ltsp-custom.min.js`).

Collection/search listings fetch a filtered render via `search.js?...&view=bss.login` so listing prices respect the lock too.

## No metafields

Unlike v2 Locks, LTSP stores **nothing in Shopify metafields**. Its configuration lives in the app database, and its enforcement lives in the injected theme code. This is the key operational difference: LTSP changes are tied to your theme files.

## Theme changes & re-sync

Because LTSP writes into theme files:

- Switching the published theme, or duplicating/editing the theme, can drop the wrappers. If prices reappear, re-save the rule or re-run theme install.
- Uninstalling the app does **not** automatically strip injected theme code; manual cleanup may be needed (this is disclosed on the [Agreement](../analytics-and-settings/onboarding-and-agreement.md) screen).

## Hide Price on Google Search

The HPOGS variant keeps locked prices out of Google by managing a verification/feed path; it prevents the real price leaking into Google Shopping/Search results when prices are hidden on-site.
