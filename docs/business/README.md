---
title: B2B Lock — Documentation
feature: _overview
category: getting-started
page_type: overview
audience: merchant
summary: >-
  B2B Lock is a Shopify content-and-access-control app that hides pages, products, prices, and checkout options behind login, customer tags, passcodes, regions, dates, and other conditions.
keywords: [b2b lock, login to access pages, login to see price, content locking, access control, passcode, B2B Lock docs, LTAP, LTSP]
related: [locks, pricing, checkout-lock, force-login, reports-and-analytics]
---
# 🔒 B2B Lock — Documentation

B2B Lock restricts what a shopper can see and do on a Shopify storefront. Merchants build **rules** that lock pages, products, collections, prices, add-to-cart buttons, sections, and even checkout options behind conditions such as *being logged in*, *having a customer tag*, *entering a passcode*, *coming from a region*, or *arriving inside a date window*.

This documentation covers **two generations** of the app that run side by side:

- **v2 (current)** — the **Locks** rule engine, **Pricing** (Login to See Price), and **Checkout Lock** (Shopify Functions). Served by the `client/` admin and the `login-api-v2` backend.
- **v1 (legacy)** — the original per-module restrictions (**Force Login, Passcode, Secret Link, Subscribe-to-Access, Manual Locking, Hide Price**). Served by the `frontend/` admin and the `b2b-login-access-management-api` backend.

Which one a store sees is decided per-shop by a `client_version` flag — see [App Overview](getting-started/app-overview.md) and [Architecture Overview](_internals/architecture-overview.md).

## Table of contents

### Getting started
- [App Overview](getting-started/app-overview.md)

### Features (v2 — current)
- **Locks**
  - [What Locks are](locks/locks.md)
  - [Set up a Lock](locks/set-up-locks.md)
  - [Conditions reference](locks/conditions-reference.md)
  - [On the storefront](locks/on-the-storefront.md)
- **Pricing (Login to See Price)**
  - [What Pricing locks are](pricing/pricing.md)
  - [Set up Pricing](pricing/set-up-pricing.md)
  - [On the storefront](pricing/on-the-storefront.md)
- **Checkout Lock**
  - [Overview](checkout-lock/checkout-lock.md)
  - [Purchase validation](checkout-lock/purchase-validation.md)
  - [Payment customization](checkout-lock/payment-customization.md)
  - [Delivery customization](checkout-lock/delivery-customization.md)

### Features (v1 — legacy restrictions)
- [Force Login](legacy-restrictions/force-login.md)
- [Passcode](legacy-restrictions/passcode.md)
- [Secret Link](legacy-restrictions/secret-link.md)
- [Subscribe to Access](legacy-restrictions/subscribe-to-access.md)
- [Manual Locking](legacy-restrictions/manual-locking.md)
- [Hide Accelerated Checkout](legacy-restrictions/hide-accelerated-checkout.md)

### Analytics & administration
- [Reports & Analytics](analytics-and-settings/reports-and-analytics.md)
- [Passcode Requests](analytics-and-settings/passcode-requests.md)
- [Settings & Translations](analytics-and-settings/settings-and-translations.md)
- [Onboarding & Agreement](analytics-and-settings/onboarding-and-agreement.md)
- [Plans & Pricing](analytics-and-settings/plans-and-pricing.md)

### Reference
- [Settings & Values Glossary](reference/settings-and-values-glossary.md)
- [Metafield Contracts](reference/metafield-contracts.md)
- [Feature Compatibility Matrix](reference/feature-compatibility-matrix.md)

### Use cases & FAQs
- [FAQs](use-cases-and-faqs/faqs/)
- [Use cases](use-cases-and-faqs/use-cases/)

### Internal companion (engineers)
- [Architecture Overview](_internals/architecture-overview.md)
- [Locks internals](_internals/locks-internals.md)
- [Pricing internals](_internals/pricing-internals.md)
- [Checkout Lock internals](_internals/checkout-lock-internals.md)
- [Legacy restrictions internals](_internals/legacy-restrictions-internals.md)
