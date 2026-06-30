---
title: App Overview
feature: _overview
category: getting-started
page_type: overview
audience: merchant
summary: >-
  B2B Lock locks pages, products, prices, and checkout options behind login, tags, passcodes, regions, and dates; it ships as a two-generation app (v1 legacy modules and v2 Locks engine) chosen per shop.
keywords: [b2b lock overview, what is b2b lock, content locking, access control app, app embed, login to access pages, v1 v2, client_version]
related: [locks, pricing, checkout-lock, plans-and-pricing]
---
# 🔒 App Overview

B2B Lock is a Shopify access-control app. A merchant configures **rules** in the embedded admin; the app then enforces those rules on the live storefront — hiding or gating content for shoppers who don't meet the conditions. It is the BSS B2B suite's "who can see / who can buy" product, complementing B2B Solution (pricing/wholesale) and B2B Quote (RFQ).

## What B2B Lock can do

| Capability | What it locks | Where it runs | Docs |
| --- | --- | --- | --- |
| **Locks** | Whole store, products, collections, blogs, pages, URLs, variants, sections/blocks | Storefront (Liquid + JS) | [Locks](../locks/locks.md) |
| **Pricing** | Price + Add-to-Cart (Login to See Price) | Storefront (Liquid + JS) | [Pricing](../pricing/pricing.md) |
| **Checkout Lock** | Blocks checkout, hides payment methods, hides delivery options | Shopify Functions (checkout) | [Checkout Lock](../checkout-lock/checkout-lock.md) |
| **Legacy restrictions** | Force login, page passcode, secret link, subscribe-to-access, manual locking | Storefront (Liquid injected into theme) | [Legacy restrictions](../legacy-restrictions/force-login.md) |
| **Analytics** | — (reporting) | Admin | [Reports & Analytics](../analytics-and-settings/reports-and-analytics.md) |

## The two-worlds model

Like every Shopify app, B2B Lock is really **two products**:

1. **The admin app** — embedded in Shopify admin (React under App Bridge). The merchant builds rules here. Nothing the merchant changes takes effect until it reaches the storefront.
2. **The storefront** — the shopper's experience on the live theme. The lock actually *fires* here (via injected Liquid and theme-app-extension JS), or inside **Shopify's checkout** (for Checkout Lock Functions).

The seam between them is **theme code and metafields**: the backend writes config, the storefront reads it. This is why "I changed a setting but the storefront still shows the old content" is the most common question — see the [FAQs](../use-cases-and-faqs/faqs/).

## Two generations: v1 (legacy) and v2 (current)

B2B Lock has been rebuilt. Both generations are live, and **which one a store sees is decided per shop**, not by app version:

| | v1 (legacy) | v2 (current) |
| --- | --- | --- |
| Admin UI | `frontend/` React app | `client/` React app |
| Backend | `b2b-login-access-management-api` (Koa) | `login-api-v2` (NestJS) |
| Model | One **module** per restriction type | One unified **rule** engine (Locks) |
| Storefront | Liquid snippets injected into theme files | Theme App Extension (app embed) + metafields |

The flag that decides this is `client_version` (`v1` or `v2`) on the shop record. The admin host reads it (cached in Redis) and serves the matching frontend. New installs are `v2`. See [Architecture Overview](../_internals/architecture-overview.md) for the routing mechanism.

> **Note for documentation readers:** features under **Locks**, **Pricing**, and **Checkout Lock** are v2. Features under **Legacy restrictions** are v1. A given store will normally use one generation, but the underlying database and some shared features (analytics, plan) are common to both.

## Installing & enabling

1. Install B2B Lock from the Shopify App Store.
2. Accept the one-time **Agreement** screen (explains that app code lives in your theme and that pricing follows your Shopify plan) — see [Onboarding & Agreement](../analytics-and-settings/onboarding-and-agreement.md).
3. Complete **onboarding**: enter a contact email, **install the app to your theme** (auto-install writes the app embed), enable the **App Embed** block in the Shopify theme editor, and create your first lock.
4. The **App Embed** must stay enabled — it boots the storefront JS that reads your rules. If it's off, locks do not apply.

## Plans

Pricing follows the store's Shopify plan tier. The free plan can only lock the **entire website**; locking specific products, collections, pages, prices, or using passcode requests requires the paid (**Advanced**) plan or an active trial. See [Plans & Pricing](../analytics-and-settings/plans-and-pricing.md).

## Support

For support, use the in-app support channel / BSS Commerce support. (Specific contact URLs are intentionally omitted here — refer to the live app listing.)
