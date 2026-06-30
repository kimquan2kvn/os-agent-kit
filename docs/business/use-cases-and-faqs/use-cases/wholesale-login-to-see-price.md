---
title: "Use case: wholesale — hide prices until an approved customer logs in"
feature: pricing
category: use-cases-and-faqs
page_type: use-case
audience: merchant
summary: >-
  Hide all prices and add-to-cart from guests, show them only to logged-in customers tagged wholesale, by using Login to See Price (or a v2 price Lock) restricted by tag.
keywords: [wholesale, hide price until login, b2b pricing, login to see price use case, tag wholesale, approved customer pricing]
related: [pricing, locks, conditions-reference]
---
# Use case: wholesale — hide prices until an approved customer logs in

**Goal:** guests browse the catalog but can't see prices or buy. Only logged-in customers you've approved (tagged `wholesale`) see prices and add-to-cart.

## Steps

1. **Approve customers with a tag.** Tag your approved B2B customers `wholesale` in Shopify (or via B2B Solution's registration flow).
2. **Create the price lock.**
   - **v2:** add a [Lock](../../locks/set-up-locks.md) with target **Price + Add-to-cart**, action **Hide**, and conditions: a key with *Signed in* **AND** *Customer tag = wholesale*.
   - **v1:** use **Login to See Price** with action `0` (hide price + add-to-cart) and `apply_to_customer_type = 1 (AllowByTags)` allowing `wholesale`.
3. **Set the replacement message** — e.g. "Log in with your wholesale account to see prices."
4. **Save** and verify as a guest (price hidden) and as a tagged customer (price shown).

## Result

| Visitor | Sees price? | Can buy? |
| --- | --- | --- |
| Guest | No (message shown) | No |
| Logged-in, no tag | No | No |
| Logged-in, `wholesale` | Yes | Yes |

## Tips

- Pair with [Checkout Lock → Purchase validation](../../checkout-lock/purchase-validation.md) to enforce a wholesale minimum order.
- Use [Hide Accelerated Checkout](../../legacy-restrictions/hide-accelerated-checkout.md) so express buttons don't bypass the flow.
