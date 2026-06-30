---
title: "Use case: members-only store (login wall)"
feature: locks
category: use-cases-and-faqs
page_type: use-case
audience: merchant
summary: >-
  Lock the entire store so only logged-in customers can browse, while keeping account, login, register, and policy pages open, using a whole-store Lock with a Signed-in condition (or v1 Force Login).
keywords: [members only store, login wall, lock whole store, require login, force login use case, private store]
related: [locks, force-login, conditions-reference]
---
# Use case: members-only store (login wall)

**Goal:** the whole storefront is private — only logged-in customers can see anything — but shoppers can still reach the login/register pages to get in.

## Steps

1. **Create a whole-store lock.**
   - **v2:** a [Lock](../../locks/set-up-locks.md) with target **Entire website**, action **Lock**, `login_type` = login form, condition: *Signed in*.
   - **v1:** **Force Login** with `restrict_pages_type = 0` (entire store) and `apply_to_customer_type = 0 (RestrictNonLogin)`.
2. **Keep entry pages open.** The app automatically exempts account/login/register/reset/activate pages so shoppers can authenticate. Optionally exempt the homepage.
3. **Set the redirect-after-login** to return the visitor to the page they wanted (`redirect_type = 2`, v1).
4. **Save** and test in an incognito window.

## Result

- Guests hitting any locked page see your login form (or message), then return to the store after logging in.
- Account/login/register pages stay reachable so the wall isn't a dead end.

## Note

This is available on the **free plan** (whole-store locking is the one free target). Locking only *some* pages requires the advanced plan.
