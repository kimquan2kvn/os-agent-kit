---
title: Manual Locking (Legacy)
feature: manual-locking
abbrev: ML
category: legacy-restrictions
page_type: overview
audience: merchant
summary: >-
  Manual Locking (v1) is a developer-oriented feature where the merchant defines a rule and then manually wraps any Liquid block in a generated snippet, so that block is shown or replaced with a protected message based on the rule's conditions.
keywords: [manual locking, manual lock, custom liquid lock, wrap content, rule_type 9, implement code, conditional content, developer]
related: [force-login, passcode, locks]
source_refs:
  repos: [b2b-login-access-management-api, b2b-login-access-management-cms]
  paths: ["src/services/module/manualLocking.service.ts", "web/frontend/pages/manual-restrictions/**"]
  symbols: [manualLocking, "bss-ltap-ml-conditions.liquid", bss_ml_condition_protected]
---
# Manual Locking (Legacy)

> **Legacy (v1) feature** for developers/themers. Unlike other restrictions, Manual Locking does **not** auto-target pages — you place its snippet by hand. Internally it is `rule_type = 9`.

**Manual Locking** lets you conditionally show or hide **any block of theme content**. You create a rule (with customer/market/product conditions), then paste a generated Liquid snippet around the exact content you want to protect. When the rule's conditions are met, the content shows; otherwise a "protected" message shows.

## When to use it

Use Manual Locking when the built-in targets (page/product/collection/section) can't express where you need the lock — e.g. a single widget, a banner, a custom section deep in a template.

## How it works

1. Create a Manual Locking rule and configure its conditions (customer type, market, product conditions) and a "not applied" message.
2. The admin's **Implement code** screen gives you a snippet like:
   ```liquid
   {% comment %}start BSS ML Rule <id>{% endcomment %}
   {% capture bss_ml_content %}
     <YOUR_CONTENT_HERE>
   {% endcapture %}
   {% render 'bss-ltap-ml-conditions', rule_id: '<id>', content: bss_ml_content %}
   {% comment %}end BSS ML Rule <id>{% endcomment %}
   ```
3. Paste it into any theme template, replacing `<YOUR_CONTENT_HERE>` with your content.
4. At render time, `bss-ltap-ml-conditions.liquid` evaluates the rule: if the visitor qualifies it outputs your content; otherwise it outputs the protected message (`bss_ml_condition_protected`).

## Who qualifies (`apply_to_customer_type`)

Same value set as Force Login (`0`–`5`): restrict guests, by tag, logged-in, specific customers, etc. See [Force Login](force-login.md#who-is-restricted-apply_to_customer_type).

## Notes

- Manual Locking is **not** auto-included by the central orchestrator — it only runs where you place the `{% render %}` tag.
- It has no login form or redirect — it only shows content or the protected message.
- Region (specific) and product-exclusion options may require the Premium tier.
