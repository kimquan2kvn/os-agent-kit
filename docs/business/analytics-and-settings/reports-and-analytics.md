---
title: Reports & Analytics
feature: reports-and-analytics
category: analytics-and-settings
page_type: overview
audience: merchant
summary: >-
  Reports & Analytics shows per-session visitor data for your locks — how many sessions hit a lock, how many unlocked, top rules, known vs guest customers, country distribution, and a detailed session table, over the last 30 days.
keywords: [reports, analytics, lock analytics, sessions, unlock rate, visitor analytics, country distribution, known vs guest, behavior tracking, 30 days]
related: [locks, passcode-requests, settings-and-translations]
source_refs:
  repos: [login-api-v2, b2b-login-access-management-cms]
  paths: ["src/modules/internal/analytics/**", "src/modules/internal/behavior/**", "web/client/pages/reports/**"]
  symbols: [Analytic, Behavior, NewAnalyticsPanel, "/analytics/ranged-session"]
---
# 📊 Reports & Analytics

The **Reports** page shows how shoppers interact with your locks: how often locks are hit, how many visitors unlock them, who they are, and where they're from. It helps you tune rules — e.g. spot a lock that nobody passes, or see which countries hit a region lock.

## What's tracked

For each visitor **session** that hits a lock, B2B Lock records:

- The **rule** that fired and the **conditions** evaluated.
- Whether the visitor **unlocked** it, and when.
- **Location** (country/region/city) and **market**.
- **Customer** info (id, name, tags) if logged in.
- The **pages/products** involved and the **lock type** (`page` or `price`).

Data is collected by the storefront sending session events directly to the app (authenticated; bots and theme previews are filtered out).

## The Reports dashboard

The **Lock analytics** panel offers:

- **Filters:** a date range (up to a 30-day window, within the last 30 days), a **rule** filter, and a **country** filter, plus a manual refresh.
- **Sessions / block** — total sessions vs. sessions that were permitted (unlocked).
- **Top applied rules** — sessions broken down by rule.
- **Unique customers** — known (logged-in) vs. guest visitors.
- **Country distribution** — sessions per rule, broken down by country.
- **Sessions details** — a paginated table: session, rule + type, conditions, unlock status, pages, products, city/country, customer info, and timestamps.

## Behavior (aggregate counts)

Separately from per-session analytics, B2B Lock keeps lightweight **aggregate counters** of lock hits per rule/condition/day (and per hour for "today"). These power quick trend views and contain **no PII** — only counts.

## Data retention

Per-session analytics are kept for ~40 days; the full 30-day dataset is cached for performance and can be force-refreshed from the dashboard.

## Notes

- The date window is limited to the last 30 days; older data isn't shown in the main panel.
- Some internal/dev shops are excluded from analytics recording.
