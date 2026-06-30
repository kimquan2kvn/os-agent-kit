---
title: I uninstalled the app but locks/code remain in my theme
feature: _overview
category: use-cases-and-faqs
page_type: faq
audience: merchant
summary: >-
  Uninstalling B2B Lock does not automatically strip the Liquid snippets it injected into your theme (especially v1 modules and LTSP); this is disclosed at install and may require manual cleanup or restoring a theme backup.
keywords: [uninstall leftover code, theme code remains, remove app code, liquid snippets, ltsp wrappers, cleanup, agreement]
related: [onboarding-and-agreement, pricing]
---
# I uninstalled the app but locks/code remain in my theme

This is expected for the parts of B2B Lock that work by **injecting Liquid into your theme** — the v1 legacy modules (Force Login, Passcode, Secret Link, Subscribe to Access, Manual Locking) and **LTSP (Login to See Price)**. Uninstalling the app removes the app, but Shopify does not automatically delete the snippets the app wrote into your theme files. This is disclosed on the [Agreement](../../analytics-and-settings/onboarding-and-agreement.md) screen at install.

To clean up:

1. **Before uninstalling**, where possible disable/delete rules and re-run the app's tools so it removes its injected content.
2. **After uninstalling**, remove leftover `bss-ltap-*` / `bsscommerce*` snippets and the price wrappers from your theme, or **restore a theme backup** from before installation.
3. v2 metafield/app-embed-based features (Locks, Checkout Lock) stop working once the App Embed/app is gone and don't leave theme-file edits the way v1/LTSP do.

If you're unsure which snippets to remove, contact support before editing theme code.
