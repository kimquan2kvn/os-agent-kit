---
title: Plans & Pricing
feature: plans-and-pricing
category: analytics-and-settings
page_type: reference
audience: merchant
summary: >-
  B2B Lock has two internal plan codes — free (lock entire website only) and advanced (all features) — with the advanced price set by the store's Shopify plan tier (Basic, Grow, Advanced, Plus); a 14-day trial unlocks advanced features.
keywords: [plans, pricing, free plan, advanced plan, shopify plan, trial, feature gating, basic grow advanced plus, billing, plan_code]
related: [locks, conditions-reference, checkout-lock]
source_refs:
  repos: [login-api-v2, b2b-login-access-management-cms]
  paths: ["src/utils/plan.ts", "web/client/utils/pricingShopifyPlan.js", "web/client/pages/pricing/**"]
  symbols: [plan_code, PlanCheck, getShopifyPlan, mappingAppPlan]
---
# 💳 Plans & Pricing

B2B Lock has **two plan tiers** internally — `free` and `advanced` — and the *price* of the advanced plan scales with your **Shopify plan**.

## The two tiers (`plan_code`)

- `free` — free forever, but limited to locking the **entire website** (target `ALL`). No locking of specific products, collections, pages, prices, or passcode requests.
- `advanced` — the full feature set.

## Advanced pricing by Shopify plan

The advanced plan's monthly price follows your store's Shopify plan tier:

| Shopify plan | Display name | Monthly | Yearly (per mo) |
| --- | --- | --- | --- |
| (none) | Free | $0 | $0 |
| `basic` | Basic Shopify | $9.99 | $7.99 |
| `professional` | Grow Shopify | $24.99 | $19.99 |
| `unlimited` | Advanced Shopify | $49.99 | $39.99 |
| `shopify_plus` | Shopify Plus | $79.99 | $63.99 |

> Prices are as found in the app's pricing model at the time of writing; the live App Store listing is authoritative.

## What each tier unlocks

| Feature | Free | Advanced / trial |
| --- | --- | --- |
| Lock entire website (`ALL`) | ✅ | ✅ |
| Lock product / collection / blog / page / URL / variant | ❌ | ✅ |
| Hide price / add-to-cart / both | ❌ | ✅ |
| Hide section/block | ❌ | ✅ |
| Exclude specific products/pages from a lock | ❌ | ✅ |
| Passcode requests | ❌ | ✅ |
| Checkout locks (validation/payment/delivery) | ✅* | ✅ |
| Analytics | basic | full |

\* Checkout Lock Functions check subscription status and are inert on an unsubscribed store.

## Trial

New stores get a **14-day trial** of advanced features. During the trial, advanced-gated features behave as if on the advanced plan.

## Development stores

Shopify development/partner store types (`partner_test`, `trial`, `affiliate`, `staff`, `plus_partner_sandbox`) are treated as free/dev and bypass billing.

## Notes on plan naming

You may see "Free" and "Advanced" in the UI with a Shopify-plan-based display name (e.g. "Grow Shopify"). The names *Wholesale*, *Shopify Plus tier*, *Platinum*, *Premium* belong to **B2B Solution**, not B2B Lock — B2B Lock uses only `free` / `advanced` internally.
