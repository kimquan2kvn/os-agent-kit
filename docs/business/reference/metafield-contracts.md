---
title: Metafield Contracts
feature: _contracts
category: reference
page_type: contract
audience: engineer
summary: >-
  The metafield contracts B2B Lock writes and reads — login.configs, bss_ltap.hide-accelerated-checkout, bss_ltap.hide-section-configs, bss-login-configs.hide-cart, bss_lock_shop_data, and the three Checkout Lock Function metafields — with owner, writer, reader, and purpose.
keywords: [metafield, namespace, key, login.configs, bss_ltap, hide-accelerated-checkout, hide-section-configs, hide-cart, cart-checkout-validation, payment-customization, delivery-customization, contract]
related: [locks, checkout-lock, hide-accelerated-checkout, pricing]
metafield_keys: [login.configs, bss_ltap.hide-accelerated-checkout, bss_ltap.hide-section-configs, bss-login-configs.hide-cart, "app:cart-checkout-validation.function-configuration", "$app:payment-customization.configs", "$app:delivery-customization.configs"]
source_refs:
  repos: [login-api-v2, b2b-login-access-management-api, b2b-login-access-management-script]
  paths: ["src/modules/shopify/metafield/**", "extensions/theme-app-extension/blocks/config-header.liquid"]
  symbols: [MetafieldsService, config-header.liquid]
---
# Metafield Contracts

The metafield is the seam between B2B Lock's backend (writes config) and the storefront/Functions (read and execute). This page is the consolidated contract. **Note:** v1 (legacy) modules and **LTSP do NOT use metafields** — they inject Liquid into theme files instead (see [Legacy internals](../_internals/legacy-restrictions-internals.md) and [Pricing internals](../_internals/pricing-internals.md)). The metafield-driven surfaces are v2 Locks element-level config and the Checkout Lock Functions.

## v2 Locks / storefront config

### Locks JS config — `login.configs`
- **Owner:** App installation
- **Type:** `json`
- **Written by:** `MetafieldsService` (login-api-v2) on rule save
- **Read by:** `config-header.liquid` → exposed as `window.Login`; consumed by the theme-app-extension JS (price hiding selectors, etc.)
- **Purpose:** storefront selector config and the JS-process flag for element-level locks (price/cart).

### Hide accelerated checkout — `bss_ltap.hide-accelerated-checkout`
- **Owner:** App installation
- **Type:** `json`
- **Written by:** `MetafieldsService.setAcceleratedCheckoutConfig` (login-api-v2)
- **Read by:** `config-header.liquid` → `window.Login.bssAcceleratedCheckoutConfig`; consumed by `bss-hide-accelerated-checkout.min.js`
- **Purpose:** rules for hiding express/dynamic checkout buttons on product/cart pages. See [Hide Accelerated Checkout](../legacy-restrictions/hide-accelerated-checkout.md).

### Hide section/block — `bss_ltap.hide-section-configs`
- **Owner:** App installation
- **Type:** `json`
- **Written by:** `MetafieldsService.setHideSectionBlockConfig` (login-api-v2)
- **Read by:** `config-header.liquid`; consumed by the JS that hides sections/blocks
- **Purpose:** which theme sections/blocks (native id or CSS selector) to hide for `target_type = 11`.

### Hide cart — `bss-login-configs.hide-cart`
- **Owner:** Shop
- **Type:** `boolean`
- **Written by:** `MetafieldsService.setRemoveCartConfig` (login-api-v2)
- **Read by:** storefront via `getRemoveCartConfig`
- **Purpose:** global cart-removal toggle.

### Legacy shop data — `bss_lock_shop_data`
- **Owner:** Shop
- **Type:** varies (multiple keys)
- **Written by:** `MetafieldService.setShopMetafields` (legacy API)
- **Read by:** legacy storefront snippets and the Checkout Lock Functions (read `unsub_status` for subscription gating)
- **Purpose:** legacy v1 shop-level config + subscription status consumed by Functions.

## Checkout Lock — Shopify Function metafields

### Purchase validation — `app:cart-checkout-validation` / `function-configuration`
- **Owner:** Validation object
- **Type:** `json`
- **Written by:** `formatMetafieldsPayload` / `createCheckoutValidation` (login-api-v2)
- **Read by:** the Cart Checkout Validation Function (`cart.validations.generate.run`, `run.ts`)
- **Purpose:** the rule set (facts, match logic, error message, product/customer tags, shop subscription data) the Function enforces. See [Purchase validation](../checkout-lock/purchase-validation.md).

### Payment customization — `$app:payment-customization` / `configs`
- **Owner:** PaymentCustomization object
- **Type:** `json`
- **Written by:** `payment-rule.service` (login-api-v2)
- **Read by:** the Payment Customization Function (`cart.payment-methods.transform.run`)
- **Purpose:** which payment methods (by name) to hide, conditions, and scope. See [Payment customization](../checkout-lock/payment-customization.md).

### Delivery customization — `$app:delivery-customization` / `configs`
- **Owner:** DeliveryCustomization object
- **Type:** `json`
- **Written by:** `shipping-rule.service` (login-api-v2)
- **Read by:** the Delivery Customization Function (`cart.delivery-options.transform.run`)
- **Purpose:** which delivery options (by title/handle) to hide, and conditions. See [Delivery customization](../checkout-lock/delivery-customization.md).

## Write triggers

- **Rule save** — the main trigger for `login.configs`, `bss_ltap.*`, and the three Function metafields.
- **App uninstall webhook** — deletes Checkout Lock rules (validation/payment/delivery).
- **Theme publish webhook** — re-injects v2 lock content into the newly published live theme (for `upload_target = live`).

## Staleness

Because write (backend) and read (storefront/Function) are decoupled, the classic symptom is "changed a setting, storefront unchanged". Causes: App Embed disabled, theme cache, a theme switch dropping injected content, or the metafield not yet propagated. See the [FAQs](../use-cases-and-faqs/faqs/).
