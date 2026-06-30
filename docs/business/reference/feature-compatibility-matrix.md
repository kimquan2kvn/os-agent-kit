---
title: Feature Compatibility Matrix
feature: _matrix
category: reference
page_type: matrix
audience: merchant
summary: >-
  How B2B Lock features interact — v2 Locks, Pricing, and Checkout Lock stack; v1 legacy modules are an alternative generation chosen per shop; element-level locks normalize whole-page conditions; precedence is by rule priority and first-match flags.
keywords: [compatibility, interactions, stacking, conflicts, precedence, v1 v2, priority, first match, normalizeCondition]
related: [locks, pricing, checkout-lock, force-login]
---
# Feature Compatibility Matrix

How B2B Lock features combine, override, or conflict.

## Generations: v1 vs v2

A store runs **one generation** (`client_version`). v1 (legacy modules) and v2 (Locks engine) are **mutually exclusive in practice** — they are alternate admin/backends over the same database, not features you mix on one store. Migration moves a shop from `v1` to `v2`.

| Feature | Generation | Stacks with | Notes |
| --- | --- | --- | --- |
| Locks | v2 | Pricing (v2 price target), Checkout Lock | Core engine |
| Pricing (LTSP) | v1 engine (also v2 price target) | Locks | Injects theme code; no metafields |
| Checkout Lock | v2 | Locks, Pricing | Shopify Functions, independent of theme |
| Force Login | v1 | Passcode, Secret Link, SNTAP | First-match wins per page |
| Passcode | v1 | Force Login, Secret Link | Cart-attribute persistence |
| Secret Link | v1 | Force Login, Passcode | Token via URL |
| Subscribe to Access | v1 | Force Login | Email gate |
| Manual Locking | v1 | (manual placement) | Not auto-included |
| Hide Accelerated Checkout | v2 (metafield + JS) | Locks, Pricing | CSS/JS hide |

## Precedence & first-match (v1)

Within v1, the central orchestrator (`bsscommerce_login_require.liquid`) evaluates SNTAP → Force Login → Passcode → Secret Link. The **first matching rule** sets an "applied" flag (`isForceLoginApplied`, etc.) so subsequent modules don't double-fire on the same page load.

## Precedence (v2)

Each Lock has a **priority**; when more than one Lock could apply to a page, priority decides which wins. Element-level locks (price, cart button, section) run `normalizeCondition`, which strips certain whole-page conditions that don't make sense at element scope.

## Checkout Lock interactions

The three Checkout Lock Functions are **independent**:

- **Purchase validation** can block checkout (errors).
- **Payment** and **Delivery** customization only **hide** options; they never block.

All three short-circuit if the shop isn't subscribed (read `bss_lock_shop_data.unsub_status`).

## Plan gating as a cross-cutting constraint

Many target types and conditions require the **advanced** plan or trial; on **free**, only whole-website locking is available. See [Plans & Pricing](../analytics-and-settings/plans-and-pricing.md). This effectively gates which combinations a store can build.

## Machine-readable graph

The machine-readable edge set is derived to [`_internals/relations.json`](../_internals/relations.json).
