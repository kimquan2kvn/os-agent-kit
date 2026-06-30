---
title: Settings & Values Glossary
feature: _glossary
category: reference
page_type: glossary
audience: merchant
summary: >-
  A dictionary of every B2B Lock enum and flag — target types, action types, login types, customer-type restrictions, passcode scope, product/price modes, condition types, request statuses, plan codes, and rule-type module codes — each as value (meaning).
keywords: [glossary, enum, values, target_type, action_type, login_type, apply_to_customer_type, rule_type, condition types, passcode settings, plan codes, dictionary]
related: [locks, conditions-reference, pricing, force-login]
---
# Settings & Values Glossary

Every enum/flag in B2B Lock, written as `value (meaning)`. Use Ctrl-F for an exact code.

## Lock target (`target_type`) — v2
One block, the v2 Lock target discriminator:

- `0 (ALL)` — entire website
- `1 (PRODUCT)` — product page(s)
- `2 (COLLECTION)` — collection page(s)
- `3 (BLOG)` — blog/article pages
- `4 (URL)` — URL-matched pages
- `5 (PAGE)` — Shopify pages
- `6 (VARIANT)` — product variants
- `8 (PRICE)` — hide price
- `9 (CART_BTN)` — hide add-to-cart
- `10 (PRICE_CART_BTN)` — hide price + add-to-cart
- `11 (SECTION_OR_BLOCK)` — hide theme section/block

## Action (`action_type`) — v2
- `0 (LOCK)` — show a message/form in place of content
- `1 (HIDE)` — remove the element

## Blocked-view type (`login_type`)
- `0` — custom login form
- `1` — sign-up / registration form
- `2` — custom message
- `3` — redirect to a page
- `4` — Shopify default
- *(In v1 Force Login the same field is: `0` custom message, `1` Shopify login form, `2` message + login form, `3` message + register form, `4` redirect.)*

## Redirect target (`redirect_type`) — v1 Force Login
- `0` — `/account`
- `1` — custom URL
- `2` — current page (return after login)

## Customer restriction (`apply_to_customer_type`) — v1 modules
- `0 (RestrictAll / RestrictNonLogin)` — restrict all / guests (module-dependent)
- `1 (AllowByTags)` — allow customers with tags
- `2 (RestrictNonLogin)` — restrict non-logged-in
- `3 (RestrictByTags)` — restrict by tags
- `4 (RestrictSpecific)` — restrict except specific customers
- `5 (AllowSpecific)` — allow only specific customers
- `6 (RestrictLogged)` — restrict logged-in

## Pricing: which products (`apply_to_product_type` / `all_products`)
- `1 (AllowAll)` — all products
- `2 (ByTags)` — by tags
- `3 (ByProducts)` — by specific products
- `4 (ByCollections)` — by collections

## Pricing: action (`login_to_see_price_action`)
- `0` — hide price + add-to-cart
- `1` — hide add-to-cart only

## Passcode scope (`advanced_passcode_settings`) — v1
- `0` — enter once → all pages
- `1` — enter once → per page (persists)
- `2` — enter each time → per page (cleared on access)
- `3` — collection passcode (per collection)

## Passcode case sensitivity (`passcode_case_sensitive`)
- `0` — case-insensitive
- `1` — case-sensitive

## Page scope (`restrict_pages_type`) — v1
- `0` — entire store (with exemptions)
- `1` — specific pages only

## Exclude product type (`ExcludeProductType`) — v1
- `1` — none
- `2` — by product tags
- `3` — by collections
- `4` — by specific product ids

## Market type (`apply_to_market_type`) — v1
- `0` — all regions
- `2` — specific regions (Premium)

## v2 condition types (`type`)
`everyone_has_access`, `signed_in`, `customer_tag`, `customer_specific`, `customer_email`, `passcode`, `subscribe_email`, `start_date`, `end_date`, `market_specific`, `regions`, `specific_ip`, `url_parameter`, `secret_link`, `age_verification`, `custom_liquid`, `customer_amount_order`, `customer_total_order`, `customer_avg_order_amount`, `company_name`, `company_location`. Each may carry an `inverse` (NOT) flag. See [Conditions reference](../locks/conditions-reference.md).

## Analytics condition labels (`condition_type`)
`signed_in`, `passcode`, `customer_tag`, `subscribe_email`, `customer_specific`, `age`, `secret_link`, `start_date`, `end_date`, `custom-liquid`, `specific_ip`, `regions`, `customer_email`.

## v1 module discriminator (`rule_type`)
- `0` — Force Login (FL)
- `1` — Passcode (PC)
- `2` — Login to See Price (LTSP)
- `3` — Secret Link (SL)
- `4` — Hide Price (HP / hidePC)
- `5` — Subscribe to Access (SNTAP)
- `6` — Passcode to View Price (PTVP)
- `8` — Hide Variants (HV)
- `9` — Manual Locking (ML)
- *(HPOGS = Hide Price on Google Search; internal module code `8`.)*

## v1 module code ↔ id (`ModuleCode`)
`fl`=1, `pc`=2, `ltsp`=3, `sntap`=4, `hp`=5, `sl`=6, `ptvp`=7, `hpogs`=8, `hv`=9, `ml`=10.

## Passcode request status
- `pending` · `approved` · `rejected` · `invalid`

## Usage history status
- `Success` · `Fail`

## Theme upload target (`upload_target`)
- `live` — publish to live theme
- `test` — write to a test theme (`test_theme_id`)

## Analytics period
- `day` — last 48h · `week` — last 14d · `month` — last 30d

## Plan code (`plan_code`)
- `free` — lock entire website only
- `advanced` — full features (priced by Shopify plan)
- *(legacy `starter` collapsed into advanced)*

## Shopify plan tiers (pricing)
`null` (Free) · `basic` (Basic Shopify) · `professional` (Grow Shopify) · `unlimited` (Advanced Shopify) · `shopify_plus` (Shopify Plus).

## Ease-of-use rating (feature request)
`very_poor` · `poor` · `neutral` · `good` · `excellent`.
