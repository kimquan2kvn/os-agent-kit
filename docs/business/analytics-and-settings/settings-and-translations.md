---
title: Settings & Translations
feature: settings-and-translations
category: analytics-and-settings
page_type: setup
audience: merchant
summary: >-
  Settings covers theme installation (auto/manual), global app config (enable, theme target, registration form options, add-to-cart selector), per-rule per-locale translations of all lock text, and the passcode email template.
keywords: [settings, installation, auto install, manual install, translations, locale, translate lock message, app config, upload target, test theme, email template]
related: [locks, passcode-requests, onboarding-and-agreement]
source_refs:
  repos: [login-api-v2, b2b-login-access-management-cms]
  paths: ["src/modules/internal/config/**", "web/client/pages/settings/**"]
  symbols: [Config, "settings/translations/[locale]", theme-installations, upload_target]
---
# ⚙️ Settings & Translations

The **Settings** page has three areas: **Installation**, **Translation**, and **Template Email Passcode**.

## Installation

Manage how the app's code is added to your theme.

- **Auto Install** — pick a theme and let the app write the app embed/snippets for you. Install/uninstall timestamps are tracked per theme.
- **Manual Install** — copy-paste instructions for enabling the App Embed block yourself in the theme editor.

> The **App Embed must stay enabled** for v2 locks to fire. If you switch themes, re-run install on the new theme.

### Theme target (`upload_target`)
The app can write to your **live** theme (`live`) or a **test** theme (`test`, using a `test_theme_id`) — useful for staging lock changes before publishing.

## Global app config

Key global settings (the `configs` record):

- **Enable** — master on/off for the app.
- **Theme id / test theme id** — which theme the app targets.
- **Disable registration form** — hide the registration form on lock pages.
- **Hide create account** — hide the "create account" link.
- **Add-to-cart selector** — a custom CSS selector for your theme's add-to-cart button (so price/cart locks find it).
- **Active translation** — whether translations are applied.
- **Registration page message** — custom text on the registration page.

## Translation

Translate every shopper-facing string, **per rule** and **per locale**. Pick a rule, pick a locale, and edit the text for whichever form types that rule uses:

- **Passcode** — message, input label, incorrect-passcode error, button text, placeholder, and every field of the *request passcode* modal.
- **Login / register** — non-login message, register message, custom message, not-applied message.
- **Hide price** — hide-price message and element access-denied message.
- **Subscribe** — subscribe message and thank-you message.
- **Secret link** — secret-link message.

The app admin UI itself is also localized into ~20 languages. The shopper-facing locale follows the storefront locale.

## Template Email Passcode

A visual editor for the email sent to shoppers when you approve a [passcode request](passcode-requests.md).

## Notes

- Translation sections auto-enable based on which conditions the selected rule actually uses.
- If translated text doesn't show, confirm **Active translation** is on and the rule has text for that locale.
