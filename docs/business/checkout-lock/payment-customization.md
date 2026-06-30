---
title: Payment Customization
feature: checkout-lock
category: checkout-lock
page_type: setup
audience: merchant
summary: >-
  Payment customization is a Shopify Function that hides specific payment methods (by method name) for certain customers, products, or cart conditions, scoped to checkout, product page, or cart page.
keywords: [payment customization, hide payment method, hide payment option, payment rules, cart.payment-methods.transform.run, scope checkout]
related: [checkout-lock, purchase-validation, delivery-customization]
metafield_keys: ["$app:payment-customization.configs"]
source_refs:
  repos: [b2b-login-access-management-script, login-api-v2]
  paths: ["extensions/payment-customization/**", "src/modules/module/payment-rule/**"]
  symbols: ["cart.payment-methods.transform.run", "$app:payment-customization", PaymentRules]
---
# Payment Customization

**Payment customization** hides specific payment methods at checkout for shoppers who match your conditions. It runs as the Shopify Function target `cart.payment-methods.transform.run` and hides methods **by their name**.

> Payment customization only **hides** methods — it never blocks checkout. To block, use [Purchase validation](purchase-validation.md).

## What it can do

- **Hide payment methods** (e.g. hide "Cash on Delivery" or a specific gateway) for certain customers or carts.
- Match on conditions such as **customer tag**, **product tag**, **signed in**, or always.

## Scope

A payment rule has a **scope** controlling where it applies:

- `checkout` — at checkout.
- `product_page` — product page context.
- `cart_page` — cart page context.

## How to set it up

1. Open **Checkout Lock → Payment rules**.
2. Create a rule: pick the conditions, pick the **payment method name(s)** to hide, and the scope.
3. Save. The backend stores the rule in the `payment-rules` table and writes the metafield `$app:payment-customization` / `configs`, which the Function reads at checkout. The Shopify payment customization id is tracked via the `SHOPIFY_PAYMENT_CUSTOMIZATION_ID` configuration.

## What the shopper sees

Matching shoppers simply **don't see** the hidden payment method(s) at checkout. Everyone else sees all methods.

## Notes

- Hiding is **by method name**, so the configured name must match the method as Shopify presents it.
- The Function checks subscription/trial status before acting.
