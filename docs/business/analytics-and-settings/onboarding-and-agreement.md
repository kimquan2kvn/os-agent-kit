---
title: Onboarding & Agreement
feature: onboarding-and-agreement
category: analytics-and-settings
page_type: overview
audience: merchant
summary: >-
  The Agreement screen is a one-time disclosure (app code lives in your theme, pricing follows your Shopify plan) shown on install; onboarding is a 4-step wizard — contact email, theme install, enable app embed, and create your first lock.
keywords: [onboarding, agreement, first install, setup guide, app embed, create first lock, contact email, opt in, NPS, feature request]
related: [app-overview, settings-and-translations, plans-and-pricing]
source_refs:
  repos: [login-api-v2, b2b-login-access-management-cms]
  paths: ["src/modules/internal/onboarding/**", "web/client/pages/agreement/**", "web/client/pages/onboarding/**"]
  symbols: [Onboarding, active_agreement, "/shop/update-agreement"]
---
# Onboarding & Agreement

## The Agreement screen

On first install (or re-activation), B2B Lock shows a one-time **Agreement** screen before you can use the app. It's a merchant disclosure, not a shopper terms gate. It explains:

1. **App code lives in your theme** — uninstalling the app does not automatically remove its theme code; manual cleanup may be needed.
2. **Pricing follows your Shopify plan** — the app suggests pricing based on your store's Shopify plan tier.
3. **Support** — how to get help.

Click **Accept and continue** to proceed (this clears the `active_agreement` flag). You won't see it again unless reactivated.

## Onboarding (4 steps)

After the agreement, a short wizard gets you live:

1. **Email** — enter a contact email and choose marketing opt-in.
2. **Install** — pick a theme and run **Automatic Install** (writes the app embed to the theme).
3. **Embed** — enable the **App Embed** block in the Shopify theme editor.
4. **Create lock** — choose a starting preset: *Login to see prices*, *Passcode to view pages*, or *Login to view products*. Finishing takes you straight into creating that lock with the preset pre-filled.

You can **Skip onboarding** (confirmed via a prompt); the app marks onboarding done without forcing theme install.

After onboarding, an in-app **setup guide** banner tracks remaining setup tasks.

## Feedback: NPS & feature requests

- **NPS** — a one-time rating popup (stars + optional comment) collected after you save a rule design.
- **Feature requests** — a Dashboard form where you can suggest features or vote on interest areas (analytics, passcode, registration, countdown).

## Notes

- Onboarding progress (current step, completion) is stored per shop so the wizard resumes where you left off.
- The contact email and opt-in you provide are saved to your shop record.
