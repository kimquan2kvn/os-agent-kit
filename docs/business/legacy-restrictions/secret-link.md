---
title: Secret Link (Legacy)
feature: secret-link
abbrev: SL
category: legacy-restrictions
page_type: overview
audience: merchant
summary: >-
  Secret Link (v1) grants access to locked pages when the visitor's URL carries a valid token (?token=value); the token is validated in Liquid and remembered as a cart attribute for the session.
keywords: [secret link, token access, hidden link, url token, ?token=, rule_type 3, premium, share link access]
related: [force-login, passcode, subscribe-to-access, locks]
source_refs:
  repos: [b2b-login-access-management-api]
  paths: ["src/services/module/secretLink.service.ts"]
  symbols: [secretLink, "sl-token", "bss-ltap-sl-rules.liquid"]
---
# Secret Link (Legacy)

> **Legacy (v1) feature.** In v2 the equivalent is a [Lock](../locks/locks.md) with a *Secret link* condition. Requires the Premium tier.

**Secret Link** lets visitors into locked pages via a special URL containing a token (e.g. `?token=abc123`) — no form to fill in. Anyone with the link gets access; anyone without it is blocked. Internally it is `rule_type = 3`.

## How tokens work

- A rule holds one or more tokens (stored delimited by `*bss*`).
- When a visitor's URL contains a valid token, Liquid grants access.
- On first valid match the token is saved as a **cart attribute** (`sl-token-<id>`) so the visitor stays unlocked for the session without keeping the token in the URL.

## Who is restricted (`apply_to_customer_type`)

- `0 (RestrictAll)` — always require the token.
- `1 (AllowByTags)` — customers with allowed tags skip the token.
- `2 (RestrictNonLogin)` — only guests need the token.
- `5 (AllowSpecific)` — specific customers skip the token.

## Use cases

- Share a private collection or page with select customers via a link.
- Give press/VIPs early access without creating accounts.

## How it's enforced

Secret Link rules compile into `bss-ltap-sl-rules.liquid` files uploaded to the theme, included via `bsscommerce_login_require.liquid`. The Liquid checks both the saved cart attribute and the URL's `?token=` value. Details in the [internals](../_internals/legacy-restrictions-internals.md).

## Notes

- Token persistence relies on cart attributes; clearing the cart/session means the visitor needs the link again.
