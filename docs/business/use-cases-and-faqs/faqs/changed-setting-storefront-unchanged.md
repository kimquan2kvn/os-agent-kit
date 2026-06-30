---
title: I changed a lock but the storefront still shows the old behavior
feature: locks
category: use-cases-and-faqs
page_type: faq
audience: merchant
summary: >-
  If a saved lock change isn't reflected on the storefront, check that the App Embed is enabled, the rule status is on, your theme is the published one, and clear cache; for LTSP re-save since it writes theme files.
keywords: [changed setting not working, storefront not updating, lock not applying, app embed disabled, cache, stale, propagation]
related: [on-the-storefront, settings-and-translations]
---
# I changed a lock but the storefront still shows the old behavior

Because the admin (where you configure) and the storefront (where the lock fires) are decoupled, a change can take a moment — or be blocked by one of these:

1. **App Embed is disabled.** v2 locks only fire when the App Embed block is enabled in the theme editor. Re-enable it. See [App Overview](../../getting-started/app-overview.md).
2. **The rule is off.** Confirm the Lock's status is enabled.
3. **You edited a different/unpublished theme.** The app targets your published theme (or a test theme if `upload_target = test`). Make sure you're viewing the theme the app wrote to. See [Settings → Installation](../../analytics-and-settings/settings-and-translations.md).
4. **Cache.** Shopify/CDN or browser cache can briefly serve the old page. Hard-refresh / wait a moment.
5. **LTSP (Login to See Price) specifically.** LTSP injects code into theme files. If you switched/duplicated a theme, re-save the rule or re-run theme install. See [Pricing on the storefront](../../pricing/on-the-storefront.md).

If it still doesn't update after the above, re-run **Auto Install** for the current theme.
