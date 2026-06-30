---
title: Force Login (Legacy)
feature: force-login
abbrev: FL
category: legacy-restrictions
page_type: overview
audience: merchant
summary: >-
  Force Login (v1) blocks selected pages or the whole store for guests or non-qualifying customers, showing a login form, sign-up form, custom message, or redirect until they sign in.
keywords: [force login, login to view pages, login wall, restrict store, require login, legacy, rule_type 0, exempt pages, redirect after login]
related: [passcode, secret-link, subscribe-to-access, manual-locking, locks]
source_refs:
  repos: [b2b-login-access-management-api, b2b-login-access-management-cms]
  paths: ["src/services/module/forceLogin.service.ts", "web/frontend/pages/page-restrictions/**"]
  symbols: [forceLogin, "bss-ltap-fl-rules.liquid", uploadFLFilesToTheme]
---
# Force Login (Legacy)

> **Legacy (v1) feature.** Force Login is part of the original per-module restriction model served by the legacy admin (`frontend/`) and the legacy API. In v2 the equivalent is a [Lock](../locks/locks.md) with a *Signed in* condition.

**Force Login** ("Login to View Pages") blocks access to selected pages — or the entire store — until a visitor logs in, registers, or is redirected. It's the classic "members only store" wall. Internally it is `rule_type = 0`.

## What a blocked visitor sees (`login_type`)

- `0` — a **custom rich-text message** (with optional login/register links).
- `1` — the **Shopify default login form** embedded in place.
- `2` — a **custom message with the login form** embedded.
- `3` — a **custom message with the sign-up form** embedded.
- `4` — a **redirect** to a custom page URL.

## After login, where they land (`redirect_type`)

- `0` — the default account page (`/account`).
- `1` — a custom URL.
- `2` — back to the page they were trying to view (current page).

## Who is restricted (`apply_to_customer_type`)

- `0 (RestrictNonLogin)` — guests only.
- `1 (AllowByTags)` — block unless the customer has an allowed tag.
- `2 (RestrictLogged)` — logged-in customers only.
- `3 (RestrictByTags)` — block customers with specific tags.
- `4 (RestrictSpecific)` — block unless the customer is in an allowed list.
- `5 (AllowSpecific)` — allow only specific customers.

## Which pages (`restrict_pages_type`)

- `0` — **entire store**, with automatic exemptions.
- `1` — **specific pages** only.

### Pages always exempt when restricting the whole store

So shoppers can actually log in, these are auto-excluded: account/login/register/reset-password/activate pages, addresses, orders, the wholesaler/B2B registration page. Optionally the homepage. With redirect mode, 404 pages are also excluded.

## How it's enforced

Force Login injects a Liquid snippet (`bsscommerce_login_require.liquid`) that captures the page's content and, when a rule matches, replaces it with the lock message/form (or runs a redirect). Rules compile into `bss-ltap-fl-rules.liquid` files (batched 30 per file) uploaded to the theme. Re-injection happens on every rule save. Details in the [internals](../_internals/legacy-restrictions-internals.md).

## Notes

- A `isForceLoginApplied` flag ensures only the first matching rule fires per page load.
- Lock logic is skipped in the theme editor / preview (`request.design_mode`).
