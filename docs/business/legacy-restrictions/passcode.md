---
title: Passcode (Legacy)
feature: passcode
abbrev: PC
category: legacy-restrictions
page_type: overview
audience: merchant
summary: >-
  Passcode (v1) protects pages behind one or more passcodes stored as a cart attribute; merchants control scope (once for all pages, once per page, or each time) and case sensitivity, and can enable a request-a-passcode flow.
keywords: [passcode, page passcode, password protect page, passcode scope, advanced passcode settings, request passcode, rule_type 1, case sensitive]
related: [force-login, secret-link, subscribe-to-access, passcode-requests, locks]
source_refs:
  repos: [b2b-login-access-management-api, b2b-login-access-management-cms]
  paths: ["src/services/module/passcode.service.ts", "src/liquids/module/passcode.liquid.ts"]
  symbols: [passcode, advanced_passcode_settings, "bss-ltap-pc-rules.liquid"]
---
# Passcode (Legacy)

> **Legacy (v1) feature.** In v2 the equivalent is a [Lock](../locks/locks.md) with a *Passcode* condition. The request-a-passcode flow is shared — see [Passcode Requests](../analytics-and-settings/passcode-requests.md).

**Passcode** protects pages behind a passcode form. The visitor enters a code; on success the result is stored as a Shopify **cart attribute** so they aren't re-prompted on every page. Internally it is `rule_type = 1`.

## Multiple passcodes

A rule can hold several passcodes, stored together delimited by `*bss*`. (Free plan: 1 passcode; higher plans: multiple.)

## Passcode scope & persistence (`advanced_passcode_settings`)

- `0` — **enter once to access all pages**. One entry unlocks everywhere the rule fires.
- `1` — **enter once per page**. Per-entity unlock that persists.
- `2` — **enter each time per page**. Per-entity, but cleared on access so the visitor must re-enter next visit.
- `3` — **collection passcode**. Per-collection unlock; product pages use the global key.

(Modes 1–3 require higher plans; mode 0 requires the Pro tier.)

## Case sensitivity (`passcode_case_sensitive`)

- `0` — case-insensitive (input and stored code are lower-cased).
- `1` — case-sensitive.

## Request a passcode

When enabled (and on a qualifying plan), the passcode form shows a **Request passcode** link. Shoppers submit a request; merchants review and send a code from the admin. See [Passcode Requests](../analytics-and-settings/passcode-requests.md).

## The passcode form

The form (snippet `bss-ltap-passcode-form.liquid`) is fully customizable: the message, input label, placeholder, submit button text, and the "incorrect passcode" error — all translatable per locale via [Settings → Translations](../analytics-and-settings/settings-and-translations.md).

## How it's enforced

Passcode rules compile into `bss-ltap-pc-rules.liquid` (byte-batched) uploaded to the theme; the central `bsscommerce_login_require.liquid` includes them. On submit, the entered code is checked in Liquid against the stored value(s); success writes the cart attribute. Details in the [internals](../_internals/legacy-restrictions-internals.md).
