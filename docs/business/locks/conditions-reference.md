# Conditions Reference

Conditions decide **who is allowed through** a Lock. Each condition is one rule about the visitor.

**AND / OR logic:**
- Conditions in the **same key** all need to pass → **AND**
- If you have **multiple keys**, any one passing is enough → **OR**

**Inverse (NOT):** Any condition can be flipped — "allow if…" becomes "allow everyone *except*…"

---

## Condition groups

In the app, conditions are organized into tabs:

| Group | Conditions in this group |
|---|---|
| **User type** | Everyone, Signed in, Customer tag, Specific customers, Approval required |

| **Passcode** | Passcode |
| **Email** | Email domain, Subscribe email |
| **Date & Time** | Start date, End date |
| **Location** | Market, IP address, Region |
| **Advanced** | URL parameter, Secret link, Age verification, Custom Liquid |
| **Companies** | Company name, Company location |

---

## Everyone has access

No restriction — the content is open to all visitors. Typically used as a fallback "open" path alongside more restrictive keys.

*No settings to configure.*

---

## Signed in

The visitor must be **logged into a Shopify customer account**.

### Settings

**Lock page — what visitors see before logging in**

Choose how the page looks when a visitor isn't logged in:

| Option | What happens |
|---|---|
| Show a login form | A login form appears directly on the lock page |
| Show a register form | A registration form appears on the lock page |
| Custom message | You write your own content — no embedded form |

All three options let you write your own message text with full HTML support.

**After login — where visitors go**

| Option | Behavior |
|---|---|
| Back to this page | Visitor returns to the page they were trying to view |
| A specific URL | Redirect to a URL you choose |
| Different URL per customer tag | e.g. customers tagged `vip` go to `/vip`, tagged `wholesale` go to `/wholesale` |

**Access denied message**

Shown to visitors who are logged in but still don't meet the conditions (e.g. wrong tag). You can replace the default text with your own.

> Default: *"This content is locked, but it doesn't look like you have access. If you feel this is a mistake, please contact the store owner."*

---

## Customer tag

The visitor must be **logged in and have at least one of the tags** you specify on their Shopify customer account.

### Settings

- **Tag list** — add one or more tags. Having *any one* of them is enough to get through.
- **Inverse** — flip to "everyone *except* customers with these tags".

> Tags are set in Shopify Admin → Customers → select a customer → Tags.

*Same lock page, redirect, and denied message options as Signed in.*

---

## Specific customers

Access is limited to a **hand-picked list of customers**.

### Settings

- **Customer list** — search and add customers by name or email.
- **Inverse** — block those specific customers instead of allowing them.

*Same lock page, redirect, and denied message options as Signed in.*

---

## Approval required

The visitor must be **logged into a Shopify customer account and have been approved** by the merchant. Visitors who are not yet approved can submit a request directly on the storefront — the merchant reviews and approves from the dashboard.

### How it works

| Who sees what | What happens |
|---|---|
| Guest (not logged in) | Sees a login prompt with a link to the account login page |
| Logged-in, not yet approved | Sees a "Request access" form — clicks to submit a request |
| Logged-in, request pending | Sees a pending message while waiting for approval |
| Logged-in, request rejected | Sees a rejection message |
| Logged-in, approved | Sees the content normally |

When the merchant approves a request, a tag is added to the customer's Shopify account. When revoked, the tag is removed and access is immediately lost.

### Settings

**Notifications**

Toggle: **Notify me when new requests arrive**

When on, an email is sent to the address you specify each time a new request comes in.

**Request form text** — customize every piece of text the visitor sees:

| What it is | Default text |
|---|---|
| Form title | *"Request access"* |
| Submit button | *"Send request"* |
| Pending message | *"Your request has been submitted. We'll notify you once approved."* |
| Rejected message | *"Your request was not approved. Contact us for help."* |
| Login prompt (shown to guests) | *"Please log in to request access to this content."* |

### Managing requests

Approved and pending requests are managed in the **Customer Access Requests** page in the app dashboard. From there you can approve, reject, revoke, or bulk-approve multiple requests at once.

---

## Email domain

The visitor must be **logged in and their email must match a rule** you define.

### Settings

**Match type** — choose how the email is checked:

| Option | How it works | Example |
|---|---|---|
| Contains | Email includes this text anywhere | `gmail` matches `user@gmail.com` |
| Equals | Email must match exactly | `john@acme.com` |
| Domain | Matches everything after the `@` | `acme.com` matches all `*@acme.com` accounts |

- **Value list** — add one or more values to check against. Matching *any one* of them is enough.

---

## Passcode

The visitor must **enter a correct passcode** to get through.

### Settings

**Passcode list**

Add one or more passcodes. Entering any one of them correctly unlocks access.

**Case sensitivity**

- Off (default): `SECRET` and `secret` are treated the same
- On: `SECRET` and `secret` are different — must match exactly

**Session behavior** — how the unlock is remembered:

| Option | Behavior |
|---|---|
| Unlock once, open all pages | Enter passcode once → all locked pages in this rule are open for the whole session |
| Unlock once per page | Must enter the passcode for each individual page |
| Re-enter every visit | Must enter the passcode again each time the page is loaded |

**Lock page text** — customize every piece of text on the passcode form:

| What it is | Default text |
|---|---|
| Label above the input | *"Enter passcode to see product"* |
| Placeholder inside the input box | *"Enter passcode"* |
| Submit button | *"Submit"* |
| Error message when wrong passcode entered | *"Wrong passcode. Need help? Contact us."* |
| Optional intro message above the form | (blank) |

**Request passcode flow**

Turn on a *"Request a passcode"* button on the lock page. Visitors fill in their name, email, and a note. You receive a notification in the app and can send them the passcode directly.

**Lock page design**

| What you can change | Default |
|---|---|
| Image displayed on the lock page | App default image |
| Form position on the page | Top-left |
| Input field width × height | 335 × 42 px |
| Input border style, roundness, color | Solid line, slightly rounded, black |
| Submit button width × height | 79 × 42 px |
| Button roundness | 5 px |
| Button background color | Black |
| Button text color | White |
| Title text size and style | 14 px, normal |
| Title color | Black |
| Custom CSS | — |

**Passcode on collection pages** *(enable this in Advanced settings)*

Displays a passcode popup directly on the collection page — visitors unlock prices without leaving the page.

| What you can change | Default |
|---|---|
| Text shown on the product card where the price normally is | *"Enter passcode to see price"* |
| Popup title | *"Passcode required"* |
| Popup subtitle | *"Enter your passcode to reveal exclusive price"* |
| Label inside the input | *"Enter passcode"* |
| Submit button | *"Reveal price"* |
| Request passcode button | *"Request a passcode"* |
| Popup background color | White |
| Text color | Black |
| Button color | Black |
| Button text color | White |
| Custom CSS | — |

---

## Subscribe email

The visitor must **submit their email address** to unlock. Their email is saved as a Shopify marketing subscriber.

### Settings

**Form content**

Write your own HTML for what appears on the lock page. Use `{SNTAP_box}` wherever you want the email input field to be placed.

**Thank you message**

Shown after a successful subscription.
> Default: *"Thanks for subscribing"*

**Form design template**

Choose a visual template for the subscription form.

---

## Start date / End date

Content is locked **before a start date** or **after an end date**. Combine both to create a fixed time window (e.g. a flash sale, a product launch).

### Settings

- **Date and time** — pick exactly when this condition activates or expires.

**Countdown clock** *(optional)*

Show a timer on the lock page counting down to when the content becomes available.

| What you can change | Default |
|---|---|
| Show the countdown clock | Off |
| Message for a locked price (use `{timer}` for the clock) | *"The price will unlock in: {timer}"* |
| Message for a locked page (use `{timer}` for the clock) | *"This page will unlock in: {timer}"* |
| Clock width on desktop | 50% |
| Clock width on mobile | 100% |
| Layout style on desktop / mobile | Inline |
| Message text font and color | Inherited from theme |
| Clock digits font and color | Inherited from theme |
| Background color | White |
| Custom CSS | — |

---

## Market specific

Restrict access based on the visitor's **Shopify Market**.

### Settings

- **All regions in a market** — anyone shopping in that market gets access.
- **Specific regions within a market** — narrow it down to selected regions inside the market.

> The app needs the `read_markets` permission enabled.

---

## IP address

Allow or block visitors based on their **IP address**.

### Settings

- **IP address list** — add one or more IP addresses.
- **Inverse** — choose between:
  - Allowlist: only visitors with these IPs can enter
  - Blocklist: visitors with these IPs are blocked

> The visitor's IP is checked live on every page load.

---

## Region

Restrict access based on the visitor's **country or region**, detected from their IP address.

### Settings

- **Region list** — search and select regions (state/province + country). You can add multiple.
- **Inverse** — flip between "only these regions" and "everyone except these regions".

> The visitor's location is checked live on every page load.

---

## URL parameter

Unlock when the page URL contains a **specific query parameter** you define. Useful for affiliate links, UTM campaigns, or sharing private access links.

### Settings

**Category** — pick a preset group or use Custom:

| Category | Ready-to-use parameters |
|---|---|
| UTM Tracking | `utm_source`, `utm_medium`, `utm_campaign`, `utm_term`, `utm_content` |
| Affiliate / Referral | `ref`, `aff_id`, `sca_ref` |
| Promo Campaign | `promo`, `discount`, `coupon` |
| Custom | You name the parameter yourself |

**How to match the value:**

| Option | Meaning |
|---|---|
| Matches exactly | URL must have the parameter with that exact value |
| Contains | The value just needs to *include* your text — e.g. `KOL` matches `KOL01` |
| Starts with | The value must begin with your text |
| Has any value | The parameter just needs to exist in the URL — any value passes |

> **Example:** Unlock for visitors coming from a TikTok ad campaign:
> Parameter = `utm_source`, Match = *Matches exactly*, Value = `tiktok`
> → `yourstore.com/page?utm_source=tiktok` → access granted ✓

---

## Secret link

Unlock when the visitor arrives via a URL that carries a **valid token**. Once they open the link, their access is remembered for the session — they don't need to keep the token in the URL as they browse.

### Settings

- **Token list** — the app generates tokens for you. Copy the link and share it with your customers.
- **Lock page message** — shown to visitors who arrive without a valid token.
  > Default: *"Private content, please arrive via the secret link"*
- **Custom CSS** for the lock page.

---

## Age verification

Shows an age gate popup. The visitor must confirm they meet the minimum age. If they decline, they are sent to a page you specify.

### Settings

**Gate rules:**

| Setting | Default |
|---|---|
| Minimum age required | 21 |
| Where to send visitors who decline | `/index` (your homepage) |
| Background image for the popup | (none) |

**Popup text:**

| Setting | Default |
|---|---|
| Heading | *"Age Verification"* |
| Body text (use `{{age}}` to insert the number) | *"You must be at least {{age}} years old to enter this site."* |
| Confirm button | *"Enter Site"* |
| Decline button | *"Exit."* |

**Popup colors:**

| Setting | Default |
|---|---|
| Popup background | White |
| Heading text | Black |
| Confirm button background | Black |
| Confirm button text | White |
| Decline button background | Black |
| Decline button text | White |

**Custom CSS** for the popup.

---

## Custom Liquid

Unlock based on a **Liquid condition you write yourself**. For advanced setups where the built-in conditions aren't enough.

### Settings

- **Condition** — write a Liquid expression that returns `true` (allow) or `false` (deny).
- **Prelude** — optional code that runs first, useful for setting up variables before the condition.

> Intended for developers familiar with Shopify Liquid.

---

## Company name / Company location

Restrict access to visitors whose **company matches a list** you set up. Built for B2B stores that manage multiple business accounts.

### Settings

- **Company name** — add company names. Visitors logged in under a matching company get access.
- **Company location** — add company locations. Useful when one company has multiple offices and you want to restrict by branch.

> **Example use case:** Show wholesale pricing only to customers from *Acme Corp* and *BigCo Ltd*, while other business accounts see the standard storefront.

---

## Plan availability

| Conditions | Minimum plan |
|---|---|
| Everyone, Signed in | Free |
| All others | Advanced or above |
| Region | Higher tier (shown in app UI) |
| Order-based conditions | Coming soon |
