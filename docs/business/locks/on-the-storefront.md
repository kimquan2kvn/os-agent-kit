---
title: Locks on the Storefront
feature: locks
abbrev: LTAP
category: locks
page_type: storefront
audience: merchant
summary: >-
  On the storefront, B2B Lock evaluates each rule mostly in server-side Liquid (login, tags, dates, passcode, secret link) and uses theme-app-extension JS for element-level hiding (price, cart button, sections) and live IP/region checks via the app proxy.
keywords: [storefront, what shopper sees, liquid evaluation, app embed, config-header.liquid, window.Login, app proxy, IP region, element hiding, file sharding]
related: [locks, conditions-reference, pricing]
metafield_keys: [login.configs, bss_ltap.hide-accelerated-checkout, bss_ltap.hide-section-configs, bss-login-configs.hide-cart]
source_refs:
  repos: [b2b-login-access-management-script, login-api-v2]
  paths: ["extensions/theme-app-extension/blocks/config-header.liquid", "extensions/theme-app-extension/assets/**", "src/modules/shopify/app_proxy/**"]
  symbols: [config-header.liquid, "window.Login", "bss-lock-condition.liquid", "/app_proxy/ping"]
---
# Locks on the Storefront

This page explains what actually happens for a shopper when a [Lock](locks.md) is in place, and how the app applies it. Locks only fire when the **App Embed** is enabled in the theme — it boots the storefront code that reads your rules.

## The boot sequence

1. The **App Embed** block renders `config-header.liquid` into the page `<head>`. It exposes the app's configuration to the page as `window.Login` and reads app metafields (`login.configs`, `bss_ltap.hide-accelerated-checkout`, `bss_ltap.hide-section-configs`).
2. For page/store and most content locks, condition evaluation happens **server-side in Liquid** (e.g. `bss-lock-condition.liquid`) as the page renders — so blocked content is never sent to the browser.
3. For **element-level** locks (price, add-to-cart, sections, accelerated checkout), the **theme-app-extension JS** runs in the browser to find and hide the elements.
4. For conditions that need the visitor's network location (**specific IP**, **regions**), the JS calls the **app proxy** (`/apps/bss-b2b-lock/app_proxy/ping`) which resolves the check live and returns which conditions are satisfied.

## What the shopper sees by action

- **Lock (page/store)** — the original content is replaced by your configured login form, sign-up form, custom message, or the page redirects, per `login_type`. Conditions are checked in Liquid, so a blocked shopper never receives the protected HTML.
- **Hide (price / cart button)** — the price and/or add-to-cart elements are removed by JS. Useful for "login to see price". (For the dedicated pricing module, see [Pricing](../pricing/pricing.md).)
- **Hide (section / block)** — a theme section or block (matched by native id or CSS selector, from `bss_ltap.hide-section-configs`) is removed for shoppers who don't pass.
- **Hide accelerated checkout** — express/dynamic checkout buttons (Shop Pay, Apple Pay, etc.) are hidden per `bss_ltap.hide-accelerated-checkout`.

## Condition evaluation at runtime

| Condition | Where evaluated |
| --- | --- |
| Signed in, customer tag, customer specific/email, company | Liquid (server) |
| Start/End date | Liquid (server) |
| Passcode | Liquid (server) against a cart attribute |
| Secret link, URL parameter | Liquid (server) against the URL/token |
| Subscribe email | Liquid (server) against a cart attribute |
| Market specific | Liquid (server) |
| **Specific IP**, **Regions** | **App proxy (live server call)** |
| Age verification | JS (gate UI) |
| Custom Liquid | Liquid (server) |

## Passcode & secret-link persistence

When a shopper enters a passcode or arrives via a secret link, the result is stored as a **cart attribute**, so subsequent page loads recognize them without re-prompting. Clearing the cart/session removes this and re-locks the content.

## Why locked elements sometimes flash

Element-level hiding (price, cart button) runs in JS after the DOM exists, so a pre-hide style is injected early to avoid a flash of the protected element. Whole-page locks don't flash because they're resolved in Liquid before the page is sent.

## Performance: file sharding

When a store has many rules, the compiled storefront config is **sharded** (split at ~250 KB) so a single oversized file doesn't bloat the theme. This is automatic.

## Troubleshooting

- **Nothing is locked** — the App Embed is probably disabled, or the rule's status is off. See [App Overview](../getting-started/app-overview.md).
- **Old behavior persists** — config propagates on save; a CDN/theme cache or a disabled embed can delay it. See [FAQs](../use-cases-and-faqs/faqs/).
- **IP/region lock not working** — these need the app proxy; confirm the app is fully installed.
