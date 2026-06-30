---
title: Passcode Requests
feature: passcode-requests
category: analytics-and-settings
page_type: overview
audience: merchant
summary: >-
  Passcode Requests lets shoppers ask for a passcode from a locked page; the request is stored and the merchant reviews it in admin, sends a passcode by email, and can track passcode usage history.
keywords: [passcode requests, request passcode, approve passcode, send passcode email, passcode usage history, pending approved rejected, passcode management]
related: [passcode, locks, reports-and-analytics]
source_refs:
  repos: [login-api-v2, b2b-login-access-management-cms]
  paths: ["src/modules/module/passcode/**", "web/client/pages/passcode-requests/**"]
  symbols: [PasscodeRequest, UsageHistoryV2, "/passcode/public/create-request", "/passcode/send-email-to-customer"]
---
# Passcode Requests

**Passcode Requests** is the flow that lets shoppers *ask* for a passcode when they hit a [passcode-protected](passcode.md) page or [Lock](../locks/locks.md), and lets you review and fulfill those requests from the admin.

## The end-to-end flow

1. **Shopper requests** — on a locked page, the shopper clicks "Request passcode" and submits their email, name, and an optional message. (This is public and rate-limited to one request per minute per IP.)
2. **Stored** — the request is saved with status **pending**.
3. **You're notified** (optional) — if notifications are on, you get an email at your configured address. Duplicate pending requests (same shop/rule/email/page) don't re-notify.
4. **You review** — in **Passcode management** (the Passcode Requests page), the **Requests** tab lists requests; you can filter by date, rule, and search.
5. **You send a passcode** — open a request, pick a passcode from the rule's configured codes, and click **Send**. The shopper gets it by email and the request is marked **approved**.
6. **Usage tracked** — when the shopper enters the code on the storefront, it's recorded in **usage history** (success/fail), visible under the **Access** tab.

## Request statuses

- `pending` — submitted, awaiting action.
- `approved` — a passcode was emailed to the shopper.
- `rejected` — you declined the request.
- `invalid` — marked invalid (e.g. duplicate/spam).

## Enabling the feature

Turn on passcode requests for the shop (a global toggle) and configure whether you receive email notifications and at which address. The feature requires a qualifying plan.

## What you see

- **Access card** — count of successful passcode uses.
- **Request card** — pending count and total requests.
- **Rules affected** — unique requester emails and how many rules have requests.

## Email template

The email sent when you approve a request is editable under [Settings → Template Email Passcode](settings-and-translations.md).
