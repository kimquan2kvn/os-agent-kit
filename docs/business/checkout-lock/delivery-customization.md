---
title: Delivery Customization
feature: checkout-lock
category: checkout-lock
page_type: setup
audience: merchant
summary: >-
  Delivery customization is a Shopify Function that hides specific delivery (shipping) options by title or handle for shoppers matching conditions such as customer tag, product tag, or signed-in state.
keywords: [delivery customization, hide shipping option, hide delivery method, shipping rules, cart.delivery-options.transform.run, delivery-customization]
related: [checkout-lock, payment-customization, purchase-validation]
metafield_keys: ["$app:delivery-customization.configs"]
source_refs:
  repos: [b2b-login-access-management-script, login-api-v2]
  paths: ["extensions/delivery-customization/**", "src/modules/module/shipping-rule/**"]
  symbols: ["cart.delivery-options.transform.run", "$app:delivery-customization", ShippingCustomization]
---
# Delivery Customization

**Delivery customization** hides specific delivery (shipping) options at checkout for shoppers who match your conditions. It runs as the Shopify Function target `cart.delivery-options.transform.run` and hides options **by title or handle**.

> Like [Payment customization](payment-customization.md), this only **hides** options — it never blocks checkout.

## What it can do

- **Hide delivery/shipping options** (e.g. hide "Express" or a local-pickup option) for certain shoppers or carts.
- Match on a focused set of conditions: **customer tag**, **product tag**, **signed in**, or **always**.

## How to set it up

1. Open **Checkout Lock → Delivery rules**.
2. Create a rule: pick the conditions and the **delivery option title/handle** to hide.
3. Save. The backend stores the rule in the `shipping_customizations` table and writes the metafield `$app:delivery-customization` / `configs`, read by the Function. The Shopify shipping customization id is tracked via `SHOPIFY_SHIPPING_CUSTOMIZATION_ID`.

## What the shopper sees

Matching shoppers don't see the hidden delivery option(s); everyone else sees all options.

## Notes

- Delivery customization supports **fewer conditions** than purchase validation (customer_tag, product_tag, signed_in, always).
- Matching is **by option title/handle** — these must match what Shopify presents.
- The Function checks subscription/trial status before acting.
