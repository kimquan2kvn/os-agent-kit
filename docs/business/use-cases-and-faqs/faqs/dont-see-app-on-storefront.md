---
title: My locks aren't working / I don't see the app on my storefront
feature: locks
category: use-cases-and-faqs
page_type: faq
audience: merchant
summary: >-
  If locks aren't applying at all, the App Embed block is almost certainly disabled or the app isn't installed to the current theme; enable the embed and re-run installation.
keywords: [app not working, no locks, app embed, not installed, theme install, locks not applying, nothing locked]
related: [app-overview, settings-and-translations, on-the-storefront]
---
# My locks aren't working / I don't see the app on my storefront

If **no** locks apply (not just one rule), it's almost always installation:

1. **Enable the App Embed.** In Shopify admin → Online Store → Themes → Customize → App embeds, turn on **B2B Lock**. This boots the storefront code that reads your rules.
2. **Install to the current theme.** Open the app → **Settings → Installation → Auto Install**, pick your published theme, and install. See [Settings & Installation](../../analytics-and-settings/settings-and-translations.md).
3. **Check the app is enabled.** The global **Enable** setting must be on.
4. **Confirm you're on a supported plan/trial** for the lock type you're testing — free plan can only lock the whole website. See [Plans & Pricing](../../analytics-and-settings/plans-and-pricing.md).

After enabling the embed and installing, reload the storefront. For per-rule problems (one lock not firing), see [the staleness FAQ](changed-setting-storefront-unchanged.md).
