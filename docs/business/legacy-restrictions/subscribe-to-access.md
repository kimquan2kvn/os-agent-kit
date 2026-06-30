---
title: Subscribe to Access (Legacy)
feature: subscribe-to-access
abbrev: SNTAP
category: legacy-restrictions
page_type: overview
audience: merchant
summary: >-
  Subscribe to Access (SNTAP, v1) gates content behind a newsletter signup; the visitor submits their email, which creates or updates a Shopify customer's marketing consent, then content is revealed and remembered via a cart attribute.
keywords: [subscribe to access, SNTAP, newsletter gate, email gate, subscribe newsletter to access pages, email to unlock, rule_type 5, marketing consent]
related: [force-login, passcode, secret-link, locks]
source_refs:
  repos: [b2b-login-access-management-api]
  paths: ["src/services/module/sntap.service.ts", "public/liquid/bss-ltap-sntap-newsletter.liquid"]
  symbols: [sntap, "bss-newsletter-email", createEmailSubscribe]
---
# Subscribe to Access (Legacy)

> **Legacy (v1) feature.** SNTAP = **Subscribe Newsletter To Access Pages**. In v2 the equivalent is a [Lock](../locks/locks.md) with a *Subscribe email* condition. Requires the Plus tier (region restrictions require Premium).

**Subscribe to Access** hides content behind a newsletter signup. The visitor enters their email to unlock; doing so creates a Shopify customer (or updates an existing customer's email-marketing consent to *subscribed*). Internally it is `rule_type = 5`.

## How the gate works

1. The locked content is replaced by a newsletter subscribe form.
2. The visitor submits their email (optionally first/last name).
3. The app creates/updates the Shopify customer and sets the cart attribute `bss-newsletter-email`.
4. On later page loads, the presence of that cart attribute reveals the content directly.

## Who is restricted (`apply_to_customer_type`)

- `0 (RestrictAll)` — everyone must subscribe.
- `1 (AllowByTags)` — tagged customers skip the gate.
- `2 (RestrictNonLogin)` — only guests must subscribe.
- `3 (RestrictLogged)` — only logged-in customers must subscribe.
- `5 (AllowSpecific)` — specific customers skip the gate.

## Customization

- **Subscribe message** (rich text) and a **thank-you message** after submitting are editable and translatable.
- Merchants can fully override the form by adding a `bss-custom-newsletter.liquid` snippet to the theme.

## Use cases

- Build an email list in exchange for access to a lookbook, catalog, or wholesale section.
- Gate downloadable content behind a subscribe.

## How it's enforced

SNTAP rules compile into `bss-ltap-sntap-rules.liquid` files uploaded to the theme. The form posts the email to the app's public endpoint, which upserts the Shopify customer (handling the "already subscribed" case gracefully). Details in the [internals](../_internals/legacy-restrictions-internals.md).
