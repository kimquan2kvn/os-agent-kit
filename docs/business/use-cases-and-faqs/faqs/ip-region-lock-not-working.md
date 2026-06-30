---
title: My IP or region lock isn't working
feature: locks
category: use-cases-and-faqs
page_type: faq
audience: merchant
summary: >-
  IP and region conditions are evaluated live via the app proxy, not in Liquid; if they don't work, the app proxy must be reachable and the app fully installed, and the visitor's detected IP/location is what's matched.
keywords: [ip lock not working, region lock, geolocation, app proxy, specific_ip, regions, country lock]
related: [conditions-reference, on-the-storefront]
---
# My IP or region lock isn't working

Unlike most conditions (evaluated server-side in Liquid), **specific IP** and **region** conditions are resolved **live through the app proxy** (`/apps/bss-b2b-lock/app_proxy/ping`) because they depend on the visitor's network location.

Check:

1. **The app is fully installed** so the app proxy is configured and reachable.
2. **You're testing with the right IP/location.** The match uses the visitor's *detected* IP/country — VPNs, proxies, and your own office IP can differ from what you expect.
3. **The condition is in the right key.** Remember conditions AND within a key and OR across keys — an over-broad OR key can let visitors through. See [Conditions reference](../../locks/conditions-reference.md).
4. **Plan tier.** Region conditions are gated to higher tiers. See [Plans & Pricing](../../analytics-and-settings/plans-and-pricing.md).

If the proxy is unreachable, IP/region checks can't resolve and the lock may not behave as configured.
