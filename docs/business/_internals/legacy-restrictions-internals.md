---
title: Legacy Restrictions Internals (v1)
feature: _legacy
category: _internals
page_type: internals
audience: engineer
summary: >-
  v1 modules (FL 0, PC 1, SL 3, SNTAP 5, ML 9) are served by the legacy Koa API and enforced by injecting batched Liquid snippets into the theme orchestrated by bsscommerce_login_require.liquid; re-injection is reactive (rule save / module toggle), with first-match flags and design-mode exclusion.
keywords: [legacy internals, v1 modules, rule_type, bsscommerce_login_require.liquid, bss-ltap-fl-rules, batching, cron, isForceLoginApplied, design_mode, restricted domains, allow free domain]
related: [architecture-overview, force-login, passcode, secret-link, subscribe-to-access, manual-locking]
source_refs:
  repos: [b2b-login-access-management-api, b2b-login-access-management-cms, b2b-login-access-management-script]
  paths: ["src/services/module/**", "public/liquid/**", "src/cron/index.ts"]
  symbols: [bsscommerce_login_require.liquid, uploadFLFilesToTheme, includeFileToTheme, ModuleCode]
---
# Legacy Restrictions Internals (v1)

> **Engineer-facing.** The original per-module model in `b2b-login-access-management-api`.

## Modules (rule_type / ModuleCode)

| Module | rule_type | code | id |
| --- | --- | --- | --- |
| Force Login | 0 | `fl` | 1 |
| Passcode | 1 | `pc` | 2 |
| LTSP | 2 | `ltsp` | 3 |
| Secret Link | 3 | `sl` | 6 |
| Hide Price | 4 | `hp` | 5 |
| SNTAP | 5 | `sntap` | 4 |
| PTVP | 6 | `ptvp` | 7 |
| HPOGS | (8) | `hpogs` | 8 |
| Hide Variants | 8 | `hv` | 9 |
| Manual Locking | 9 | `ml` | 10 |

## Enforcement: theme injection

`ThemeService.includeFileToTheme()` uploads Liquid via Asset API. The central orchestrator `snippets/bsscommerce_login_require.liquid` (from `public/liquid/`) includes, in order: SNTAP → FL → PC (if customize) → SL, all wrapped in `{% unless request.design_mode or request.visual_preview_mode %}`. ML is **not** auto-included — merchant places `{% render 'bss-ltap-ml-conditions', rule_id, content %}` manually.

Per-module snippet files (batched 30/file for FL/SL/SNTAP/ML; byte-batched for PC):

- FL: `bss-ltap-fl-rules.liquid` (+ `-N`)
- PC: `bss-ltap-pc-rules.liquid` (+ `-N`)
- SL: `bss-ltap-sl-rules.liquid` (+ `-N`)
- SNTAP: `bss-ltap-sntap-rules.liquid` (+ `-N`)
- ML: `bss-ltap-ml-rules-N.liquid` + `bss-ltap-ml-conditions.liquid`

## Re-injection (resync)

Reactive only: after every `saveRule` (fire-and-forget `uploadXFilesToTheme`) and on module toggle (`ShopModuleService.uploadContentOfModule`). **No theme-resync cron.** Crons that exist: daily delete of old soft-deleted rules (30d), monthly passcode-log purge, daily Google-clear mails (deprecated path).

## First-match flags

Liquid booleans `isForceLoginApplied` / `isPasscodeApplied` / `isSecretLinkApplied` / `isSNTAPApplied` ensure only the first matching rule fires per page load.

## Persistence

- **Passcode:** cart attribute `bsscommerce-password-escape-<id>[-…]`; scope/case from `advanced_passcode_settings` / `passcode_case_sensitive`. Multiple codes delimited by `*bss*`.
- **Secret Link:** cart attribute `sl-token-<id>`; token matched in `content_for_header` `?token=`.
- **SNTAP:** cart attribute `bss-newsletter-email`; submit posts to `/sntap/create-email-subscriber` (public), upserts Shopify customer, handles "already taken".

## Notable constants / edge cases

- `allow-free-domain.const.ts` — ~250 BSS dev/training stores whitelisted for free features.
- `restricted-email.constant.ts` — competitor/agency domains blocked.
- Watermark for free-plan shops created after 2024-11-26.
- Multiple hardcoded `domain === '…myshopify.com'` per-store fixes scattered in services.
- FL `login_type` 1/2/3 extracts the theme's own login/register template into `bss-custom-form-login/register.liquid` with an injected redirect input.
