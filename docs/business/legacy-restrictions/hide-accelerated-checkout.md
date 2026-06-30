---
title: Hide Accelerated Checkout
feature: hide-accelerated-checkout
category: legacy-restrictions
page_type: overview
audience: merchant
summary: >-
  Hide Accelerated Checkout removes express/dynamic checkout buttons (Shop Pay, Apple Pay, Google Pay, PayPal Express, Buy It Now) on product, cart, and drawer pages based on customer-tag or product-tag conditions.
keywords: [hide accelerated checkout, hide buy it now, hide shop pay, hide express checkout, dynamic checkout button, additional checkout buttons, shopify-payment-button]
related: [pricing, checkout-lock, locks]
metafield_keys: ["bss_ltap.hide-accelerated-checkout"]
source_refs:
  repos: [b2b-login-access-management-script]
  paths: ["src/ltsp-js/hide-accelerated-checkout.js", "extensions/theme-app-extension/assets/bss-hide-accelerated-checkout.min.js"]
  symbols: ["window.Login.bssAcceleratedCheckoutConfig", "bss-hide-accelerated-checkout.min.js"]
---
# Hide Accelerated Checkout

**Hide Accelerated Checkout** removes Shopify's express/dynamic checkout buttons — *Buy It Now*, Shop Pay, Apple Pay, Google Pay, PayPal Express — so shoppers go through the standard cart/checkout flow instead. This is often paired with B2B/wholesale setups where express checkout would bypass intended steps.

## What it hides

- **Product page:** the dynamic checkout button (`.shopify-payment-button`).
- **Cart page & cart drawer:** the additional checkout buttons (`.additional-checkout-buttons`).
- **Quick-view modals:** the dynamic checkout button inside quick add/view.

## Conditions

Hiding can be conditional, configured per page context (product page / cart page) with rules that match on:

- **Customer tag** (contains / does not contain)
- **Product tag** (contains / does not contain), or specific product ids
- **Always**

Rules use *any*/*all* logic within a page context.

## How it's configured & applied

The configuration is stored in the app metafield `bss_ltap.hide-accelerated-checkout` and exposed to the storefront as `window.Login.bssAcceleratedCheckoutConfig`. The script `bss-hide-accelerated-checkout.min.js` reads it, gathers page/customer/cart data (from an injected `bss-lock-store-data` script tag and live `/cart.js`), and hides the matching buttons via injected CSS. A cart drawer is watched with a `MutationObserver` so dynamically loaded buttons are hidden too.

## What the shopper sees

Matching shoppers don't see express checkout buttons — only the standard add-to-cart / checkout path. Non-matching shoppers see everything as normal.

## Notes

- A pre-hide style runs early to prevent the buttons flashing before the script evaluates.
- This feature is delivered as a theme-app-extension asset (it doesn't edit theme files like the v1 Liquid modules).
