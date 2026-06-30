---
title: Locks
feature: locks
abbrev: LTAP
category: locks
page_type: overview
audience: merchant
summary: >-
  A Lock is a v2 rule that hides or restricts a target (whole store, product, collection, blog, page, URL, variant, price, add-to-cart, or section) until a shopper satisfies one or more conditions.
keywords: [lock, content lock, login to access pages, LTAP, restrict page, hide product, rule, target type, action type, lock vs hide]
related: [pricing, checkout-lock, reports-and-analytics, plans-and-pricing]
metafield_keys: [login.configs, bss_ltap.hide-accelerated-checkout, bss_ltap.hide-section-configs, bss-login-configs.hide-cart]
source_refs:
  repos: [login-api-v2, b2b-login-access-management-cms, b2b-login-access-management-script]
  paths: ["src/modules/rules/**", "web/client/pages/locks/**", "extensions/theme-app-extension/**"]
  symbols: [Rules, Keys, Conditions, RuleService, config-header.liquid]
---
# 🔒 Locks

A **Lock** is the central object of B2B Lock v2. One Lock is a **rule** that picks a **target** (what to protect), an **action** (lock or hide it), and a set of **conditions** (who is allowed through). Until a shopper meets the conditions, the target is locked or hidden.

Locks replace the older per-module model (Force Login, Passcode, etc. — see [Legacy restrictions](../legacy-restrictions/force-login.md)) with a single, composable rule builder.

## The anatomy of a Lock

Every Lock has three parts:

1. **Target** — *what* is protected. Set by `target_type`: the whole website, a product, a collection, a blog, a page, a URL, a product variant, the **price**, the **add-to-cart button**, both price and cart button, or a theme **section/block**.
2. **Action** — *how* it's protected. `action_type = 0 (LOCK)` shows a lock message/form in place of the content; `action_type = 1 (HIDE)` removes the element entirely.
3. **Conditions** — *who* gets through, organized as **keys**. See [Conditions reference](conditions-reference.md).

## Target types (`target_type`)

The target type decides what the Lock applies to. Decoded values:

- `0 (ALL)` — the entire website
- `1 (PRODUCT)` — specific product page(s)
- `2 (COLLECTION)` — specific collection page(s)
- `3 (BLOG)` — blog / article pages
- `4 (URL)` — pages matched by URL
- `5 (PAGE)` — Shopify pages
- `6 (VARIANT)` — specific product variants
- `8 (PRICE)` — hide the price
- `9 (CART_BTN)` — hide the add-to-cart button
- `10 (PRICE_CART_BTN)` — hide both price and add-to-cart
- `11 (SECTION_OR_BLOCK)` — hide a theme section or block

> On the **free plan**, only `target_type = 0 (ALL)` is available. Every other target requires the Advanced plan or an active trial. See [Plans & Pricing](../analytics-and-settings/plans-and-pricing.md).

## Lock vs. Hide (`action_type`)

- `action_type = 0 (LOCK)` — the shopper sees a **lock message** or a form (login, passcode, subscribe). They can act to unlock.
- `action_type = 1 (HIDE)` — the element simply isn't there. Used for hiding products, prices, cart buttons, sections.

## When a shopper is blocked (`login_type`)

For page/store locks, the merchant chooses what a blocked shopper sees, via `login_type`:

- `0` — a custom login form
- `1` — a sign-up / registration form
- `2` — a custom message
- `3` — a redirect to another page
- `4` — Shopify's default behavior

## How conditions combine

Conditions are grouped into **keys**. Within a key, all conditions must pass (**AND**). Across keys, any one key passing is enough (**OR**). This lets a merchant express "(logged in AND has tag *wholesale*) OR (entered the passcode)". Full detail in [Conditions reference](conditions-reference.md).

## What the shopper experiences

Depending on target and action, the shopper sees a login wall, a passcode form, a missing price, a hidden product, a removed checkout button, or a blocked section. The exact mechanics — server-side Liquid evaluation plus JS for element-level hiding — are in [On the storefront](on-the-storefront.md).

## Next steps

- [Set up a Lock](set-up-locks.md) — the step-by-step admin flow.
- [Conditions reference](conditions-reference.md) — every condition decoded.
- [On the storefront](on-the-storefront.md) — what fires for the shopper.

## Compatibility & interactions

- Multiple Locks can target the same page; the engine evaluates each. Element-level locks (price, cart button, section) strip certain whole-page conditions during evaluation (`normalizeCondition`).
- Locks (v2) and legacy restrictions (v1) are normally not both active on the same store — the store runs one generation. See the [Feature Compatibility Matrix](../reference/feature-compatibility-matrix.md).
