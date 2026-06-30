---
title: Set up a Lock
feature: locks
abbrev: LTAP
category: locks
page_type: setup
audience: merchant
summary: >-
  To set up a Lock, choose a target (store, product, collection, page, price, cart button, or section), choose Lock or Hide, add condition keys, customize the lock message/design, and save.
keywords: [create lock, set up lock, add lock, lock configuration, target type, condition keys, lock message, design, priority]
related: [locks, conditions-reference, settings-and-translations]
metafield_keys: [login.configs, bss_ltap.hide-section-configs]
source_refs:
  repos: [login-api-v2, b2b-login-access-management-cms]
  paths: ["web/client/pages/locks/**", "src/modules/rules/**"]
  symbols: [RuleService.saveRule, "/api/rules"]
---
# Set up a Lock

This page walks through creating a Lock in the B2B Lock v2 admin (the **Locks** page). Before you start, make sure the app is installed to your theme and the **App Embed** is enabled (see [App Overview](../getting-started/app-overview.md)) — a saved Lock only fires when the embed is on.

## 1. Open the Locks page and add a Lock

In the app, go to **Locks** and click **Add lock**. Each Lock is one rule with its own name, priority, and on/off status.

## 2. Choose what to protect (target)

Pick the **target type**:

| Choose | Protects | Plan |
| --- | --- | --- |
| Entire website | The whole storefront | Free + paid |
| Product | Selected products | Paid |
| Collection | Selected collections | Paid |
| Blog | Blog/article pages | Paid |
| Page | Selected Shopify pages | Paid |
| URL | Pages matching a URL | Paid |
| Variant | Selected variants | Paid |
| Price | Hides the price | Paid |
| Add-to-cart button | Hides the cart button | Paid |
| Price + Add-to-cart | Hides both | Paid |
| Section / block | Hides a theme section or block | Paid |

For specific targets (product/collection/page/variant) you then select the exact items. For **URL** targets you provide URL patterns. For **section/block** targets you supply the section's native id or a CSS selector.

## 3. Choose Lock or Hide

- **Lock** — show a message or form where the content was (use for pages, store).
- **Hide** — remove the element (use for products, prices, cart buttons, sections).

For page/store **Lock** actions, choose what a blocked visitor sees: a **login form**, **sign-up form**, **custom message**, **redirect**, or **Shopify default** (the `login_type` setting).

## 4. Add conditions (keys)

Conditions decide who passes. Add one or more **condition keys**:

- Within a key, **all** conditions must be true (AND).
- Add another key to express an alternative path (OR).

Example: Key 1 = *Signed in* **AND** *Customer tag = wholesale*; Key 2 = *Passcode*. A shopper passes if they're a logged-in wholesaler **or** they enter the passcode.

Each condition has its own configuration (tags, customer list, passcode value, dates, regions, IPs, etc.). Some conditions are gated by plan tier. See the full list in [Conditions reference](conditions-reference.md).

## 5. Customize the lock message & design

For **Lock** actions, edit the message shown to blocked shoppers (rich text). You can also set the design (colors, fonts, borders, alignment, background) for each display slot — login message, access-denied message, passcode form, subscribe form, secret-link message, countdown, age verification, etc. Translations per locale are managed on the [Settings → Translations](../analytics-and-settings/settings-and-translations.md) page.

## 6. Save

Click **Save**. On save the backend:

1. Persists the rule (`POST /api/rules` → `saveRule`).
2. Compiles the matching theme snippets and updates app/shop **metafields** so the storefront can evaluate the rule.

Changes propagate to the storefront after save. If the storefront still shows old behavior, see the staleness FAQ in [FAQs](../use-cases-and-faqs/faqs/).

## 7. (Optional) Set priority and toggle status

Locks have a **priority** and an on/off **status**. Use priority to control which Lock wins when more than one could apply to the same page. Toggle a Lock off to disable it without deleting it.

## Bulk actions

The Locks list supports **bulk edit** and **delete** (`bulkEditRule`, `deleteRule`). Soft-deleted rules are purged after 30 days.

## What the shopper sees next

See [On the storefront](on-the-storefront.md) for the runtime behavior of each target/action combination.
