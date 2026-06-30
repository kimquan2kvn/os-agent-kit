---
title: Checkout Lock
feature: checkout-lock
category: checkout-lock
page_type: overview
audience: merchant
summary: >-
  Checkout Lock uses Shopify Functions to control checkout — blocking checkout with validation errors, hiding payment methods, and hiding delivery options based on customer tags, product tags, cart amount, and login state.
keywords: [checkout lock, shopify functions, cart validation, block checkout, hide payment method, hide delivery option, payment customization, delivery customization]
related: [locks, pricing, plans-and-pricing]
metafield_keys: ["app:cart-checkout-validation.function-configuration", "$app:payment-customization.configs", "$app:delivery-customization.configs"]
source_refs:
  repos: [b2b-login-access-management-script, login-api-v2]
  paths: ["extensions/cart-checkout-validation/**", "extensions/payment-customization/**", "extensions/delivery-customization/**"]
  symbols: ["cart.validations.generate.run", "cart.payment-methods.transform.run", "cart.delivery-options.transform.run"]
---
# 🛒 Checkout Lock

**Checkout Lock** controls what happens at checkout using **Shopify Functions** — code that runs inside Shopify's own checkout, not on your theme. It has three independent capabilities:

1. **Purchase validation** — *block* checkout with an error when cart conditions aren't met. → [Purchase validation](purchase-validation.md)
2. **Payment customization** — *hide* specific payment methods for certain customers/carts. → [Payment customization](payment-customization.md)
3. **Delivery customization** — *hide* specific delivery (shipping) options. → [Delivery customization](delivery-customization.md)

Because these run as Shopify Functions, they apply at checkout regardless of theme, and they can't be bypassed by client-side tampering.

## When to use Checkout Lock vs. Locks

- Use **[Locks](../locks/locks.md)** to control what's visible on the storefront (pages, products, prices).
- Use **Checkout Lock** to control the **checkout itself** — who can complete an order, which payment methods appear, which shipping options appear.

## How it's configured

Each Function reads a **metafield** that the app backend writes when you save a rule:

| Function | Target | Metafield |
| --- | --- | --- |
| Purchase validation | `cart.validations.generate.run` | `app:cart-checkout-validation` / `function-configuration` |
| Payment customization | `cart.payment-methods.transform.run` | `$app:payment-customization` / `configs` |
| Delivery customization | `cart.delivery-options.transform.run` | `$app:delivery-customization` / `configs` |

All three also check the shop's subscription/trial status (`bss_lock_shop_data.unsub_status`, plan code) before acting, so an unsubscribed store's Functions are inert.

## Important distinction: block vs. hide

- **Purchase validation** can **stop** checkout (the shopper sees an error and cannot proceed).
- **Payment / delivery customization** only **hide** options — they never block checkout outright. If you hide all payment methods, Shopify behavior for "no methods" applies.

## Plan

Checkout Lock requires a paid plan / active trial; the Functions short-circuit when the shop isn't subscribed. See [Plans & Pricing](../analytics-and-settings/plans-and-pricing.md).

## Next steps

- [Purchase validation](purchase-validation.md)
- [Payment customization](payment-customization.md)
- [Delivery customization](delivery-customization.md)
