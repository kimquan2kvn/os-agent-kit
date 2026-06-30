---
title: Purchase Validation
feature: checkout-lock
category: checkout-lock
page_type: setup
audience: merchant
summary: >-
  Purchase validation is a Shopify Function that blocks checkout with a custom error when cart conditions are not met, using facts like product tags, customer tags, and total cart amount.
keywords: [purchase validation, block checkout, cart validation, validation error, minimum order, customer tag checkout, cart.validations.generate.run]
related: [checkout-lock, payment-customization, delivery-customization]
metafield_keys: ["app:cart-checkout-validation.function-configuration"]
source_refs:
  repos: [b2b-login-access-management-script, login-api-v2]
  paths: ["extensions/cart-checkout-validation/src/run.ts", "src/helper/checkout-rule.ts"]
  symbols: ["cart.validations.generate.run", validationAdd, function-configuration]
---
# Purchase Validation

**Purchase validation** stops a shopper from completing checkout when the cart doesn't meet your rules, showing a custom error message. It runs as the Shopify Function target `cart.validations.generate.run` and adds checkout errors via `validationAdd`.

## What it can check (facts)

A rule is a set of conditions ("facts") evaluated against the cart and customer. Supported fact types include:

- **Product has any tag** (`productHasAnyTag`) — the cart contains a product with a given tag.
- **Customer has any tag** (`customerHasAnyTag`) — the logged-in customer has a given tag.
- **Total cart amount** (`totalCartAmount`) — the cart subtotal compared to a threshold (e.g. minimum order value).

> Some fact types exist in the code but are not active/used (`customerNumberOfOrders`, `customerAmountSpent`, `countryCode`). Don't rely on them.

## How to set it up

1. In the app, open **Checkout Lock → Validation** (or the equivalent validation rules screen).
2. Create a rule: choose the facts, the match logic, and the **error message** shown when the rule blocks checkout.
3. Save. The backend writes the rule set to the metafield `app:cart-checkout-validation` / `function-configuration`, which the Function reads at checkout.

## What the shopper sees

When the cart violates the rule, the shopper sees your **error message** at checkout and **cannot proceed** until the cart satisfies the rule (e.g. reaches the minimum amount, removes a restricted product, or logs in as a tagged customer).

## Examples

- **Minimum order value**: block checkout below a `totalCartAmount` threshold with "Minimum order is $X for wholesale".
- **Wholesale-only product**: a product tagged `wholesale` can only be checked out by a customer tagged `wholesale`.

## Notes

- The Function checks the shop's subscription/trial status before enforcing; an unsubscribed store's validation is inert.
- Purchase validation **blocks**; to merely hide payment or shipping options, use [Payment customization](payment-customization.md) / [Delivery customization](delivery-customization.md).
