---
title: Checkout Lock Internals
feature: checkout-lock
category: _internals
page_type: internals
audience: engineer
summary: >-
  Checkout Lock is three Shopify Functions — validation (blocks via validationAdd), payment transform (hides methods by name), delivery transform (hides options by title/handle) — each reading its own metafield and gating on shop subscription (bss_lock_shop_data.unsub_status, plan/trial).
keywords: [checkout lock internals, shopify functions, validationAdd, FactType, payment transform, delivery transform, function-configuration, payment-customization, delivery-customization, unsub_status, customization id]
related: [architecture-overview, checkout-lock, purchase-validation, payment-customization, delivery-customization]
metafield_keys: ["app:cart-checkout-validation.function-configuration", "$app:payment-customization.configs", "$app:delivery-customization.configs"]
source_refs:
  repos: [b2b-login-access-management-script, login-api-v2]
  paths: ["extensions/cart-checkout-validation/src/run.ts", "extensions/payment-customization/**", "extensions/delivery-customization/**", "src/modules/module/payment-rule/**", "src/modules/module/shipping-rule/**"]
  symbols: ["cart.validations.generate.run", "cart.payment-methods.transform.run", "cart.delivery-options.transform.run", FactType]
---
# Checkout Lock Internals

> **Engineer-facing.** Three independent Shopify Functions.

## Functions, targets, metafields

| Function | Target | Metafield (ns / key) | DB table |
| --- | --- | --- | --- |
| Validation | `cart.validations.generate.run` | `app:cart-checkout-validation` / `function-configuration` | `checkout-rules` |
| Payment | `cart.payment-methods.transform.run` | `$app:payment-customization` / `configs` | `payment-rules` |
| Delivery | `cart.delivery-options.transform.run` | `$app:delivery-customization` / `configs` | `shipping-customizations` |

There is also a `validation-settings` **ui_extension** (`admin.settings.validation.render`) that is informational/redirect-only (`app://checkout-lock`).

## Validation (blocks)

`run.ts` reads `input.validation?.metafield?.value`, evaluates facts, and emits `validationAdd` errors to block checkout. **FactType** enum includes `productHasAnyTag`, `customerHasAnyTag`, `totalCartAmount` (active) and dead/unused `customerNumberOfOrders`, `customerAmountSpent`, `countryCode`.

## Payment (hides by name)

Hides payment methods by `method.name`. Rule has `scope` (`checkout` / `product_page` / `cart_page`), conditions, and `hidePaymentMethod` / `hideDynamicCheckoutButton`. Shopify customization id tracked via `SHOPIFY_PAYMENT_CUSTOMIZATION_ID`.

## Delivery (hides by title/handle)

Hides delivery options by `option.title` / handle. Conditions limited to `customer_tag`, `product_tag`, `signed_in`, `always`. Id tracked via `SHOPIFY_SHIPPING_CUSTOMIZATION_ID`.

## Subscription gating

All three check `bss_lock_shop_data.unsub_status` and plan_code/trial before acting — unsubscribed shops' Functions are inert.

## Lifecycle

- **Save:** writer service formats payload and upserts the metafield via GraphQL.
- **Uninstall webhook:** deletes all validation rules, payment customizations, and shipping customizations.
