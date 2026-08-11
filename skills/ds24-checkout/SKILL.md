---
name: ds24-checkout
description: Use when building the buy button, pricing page or checkout link for a Digistore24 product — creating a signed buy URL with createBuyUrl, attaching the price as a payment plan, carrying the buyer's identity through to the IPN, and the thank-you page. Use it whenever the user mentions a buy link, checkout, pricing page, "how does the customer pay", or a purchase that arrives without anybody being able to tell whose it was.
---

# The checkout link

A Digistore24 checkout is a **signed, short-lived URL** you create through the
API and send the buyer to. It is not a static link with a product id in it.

## Step 0 — is it already there?

Search the project for `createBuyUrl`, `payment_plan` or a pricing page that
already links out to Digistore24. If it exists, do not rebuild it — check it
against Step 3 and Step 4 and fix only what is wrong.

## Step 0a — is this copy of the skill pack current?

Fetch `https://raw.githubusercontent.com/digistore-io/ds24-skills/main/VERSION`
and compare with this pack's `VERSION`. Mention a mismatch in one sentence, then
carry on.

## Step 1 — the call

```
POST https://www.digistore24.com/api/call/createBuyUrl/format/json
Header: X-DS-API-KEY: <the key>
```

Body (form-encoded), the parts that matter:

```
product_id                              = 512345
valid_until                             = 24h
payment_plan[first_amount]              = 47.00
payment_plan[other_amounts]             = 47.00
payment_plan[currency]                  = EUR
payment_plan[number_of_installments]    = 0        # 0 = open-ended subscription, 1 = one-off
payment_plan[first_billing_interval]    = 1_month  # omit entirely for a one-off
payment_plan[other_billing_intervals]   = 1_month
```

**The price is sent here, at purchase time — not stored on the product.**
Digistore24 discards `data[amount]` on the product itself, and a stored payment
plan cannot carry a voucher, a trial, an upgrade or a per-link affiliate
commission. Read the numbers from the one price list in your project (see
**`ds24-products`**).

**`product_id` also decides the LANGUAGE of the order form — pick it by the
buyer's language.** A Digistore24 product carries exactly one language, and
there is no parameter in this call that overrides it (look at the body above:
`buyer`, `payment_plan`, `tracking`, `urls`, `placeholders`, `settings`,
`addons` — no language anywhere). So a multilingual app keeps **one product id
per language** per offer and resolves the visitor's language to one of them
right here, before the call. If you send everybody to the same id, half of them
fill in a form in the wrong language at the moment they are asked to pay. The
**`ds24-products`** skill has the shape of the price list and the rule in full.

The response is a URL. **Cache it per offering** — it is valid for the
`valid_until` window, and creating a fresh one on every page view is a
round-trip to Digistore24 in the path of your pricing page.

⚠️ **Then the cache key has to include the language**, not just the offer key.
One row per key means the German and the English URL evict each other on every
page view and, in between, the cache serves one language's checkout page to the
other language's buyer. `"<offerKey>:<language>"` is enough.

🚨 **And never cache a URL that carries a buyer's identity.** Step 2 puts the
signed-in member's id into `tracking[custom]`, and a cache keyed on the offer has
no member dimension — so the first signed-in buyer's identity is served to
everybody who opens that page afterwards, and every one of *their* payments
arrives attributed to that first member. Nothing fails while it happens: the page
renders, the checkout opens, the money moves.

So a pricing page has **two paths**, and they are not two versions of one:

- **Signed out → the shared cached URL.** No identity in it, safe for everyone,
  no round trip to Digistore24 while the page renders.
- **Signed in → a URL built at click time**, with that member's identity in it,
  used once and **never written to the cache**.

Decide which by the **content** of the tracking field, not by whether it is set:
a marker naming the *package* is shareable, one naming a *person* is not. Asking
merely "is tracking set" turns every card into a live API call on every page
view, which is what the cache existed to prevent.

**And when the call fails, the page still has to render.** Digistore24 being slow
or a key being wrong must produce a disabled button with a reason ("checkout
unavailable"), never a thrown error on the pricing page and never a dead link.
Return the failure to the caller instead of raising it.

**That URL is not finished yet in a development environment.** Until the
product is marketplace-approved nobody can buy through it at all, and the way
to unlock a test purchase without touching your browser is to append the
testpay parameter — **appended to the return value, after the cache, and only
where a customer can never reach it**. Do not build the checkout and leave this
for later: it is the step that decides whether you can prove any of the rest
works. **Step 4a** is the recipe and the guardrails.

## Step 2 — carry the buyer's identity through

The single most common failure in a Digistore24 integration is a payment that
arrives and cannot be matched to an account. Somebody paid, the app has no idea
who, and support has to do it by hand.

Send an identifier in the tracking field. Digistore24 stores it on the purchase
and hands it back on **every** later event for that order — the renewal a year
on, the refund, the chargeback. It arrives in the IPN as `custom`:

```
tracking[custom] = m:<member id>;t:<a short random token stored on that member>
```

**The field is one opaque string that is entirely yours**, so give it a layout you
can extend: `;`-separated `key:value` pairs, and a reader that **ignores keys it
does not know** rather than failing on them. You will want to carry a second id
through later (which package, which kind of purchase, an intent the buyer
expressed at checkout), and by then there are live purchases holding the old
value. A new id is then a new pair; a second *format* is a migration you cannot
do, because the values already sitting at Digistore24 cannot be rewritten.

**Two things about that token.** It corroborates the member id, so a guessed or
edited id alone never claims somebody else's purchase — and it is **not a
credential**: it never authenticates a session, it only says "this id was not
invented by the person typing the URL". Both halves must be present and
well-formed or the value names nobody: half an identity is not a weaker identity.

At the other end, in the IPN handler, attribute in this order — and the order is
a security rule, not a preference:

1. **The identifier from `custom`, token matching → authenticated.** Your app
   wrote this value, Digistore24 stored it server-side, and the buyer never had
   a copy they could edit.
2. **Otherwise the buyer's e-mail against your accounts → unauthenticated.**
   That address was typed into a Digistore24 form by whoever was paying, and
   **Digistore24 does not verify that they control it**. It is usually right and
   it is never proof.
3. **Otherwise store the order unattributed** and attach it when that address
   first signs in.

Two refusals are what make step 2 safe to have at all:

- 🚨 **An address matching more than one account is refused, not resolved to the
  first row.** Ask for at most two matches and treat "two" as *cannot tell*. The
  query that returns a list and takes `[0]` is the exact shape of this bug, and
  what it does is hand one customer another customer's purchase. Unattributed is
  the correct outcome; guessing is not a fallback.
- **Attribution only ever grants — it never moves and never revokes.** An e-mail
  match may attach an order that belongs to nobody yet. It may not re-point an
  order that is already attributed, and no attribution failure may end access
  that exists. That one-directionality is the whole reason an unauthenticated
  path is tolerable.

And anything that authorises an **unattended** act later — charging a stored
payment method, arming an automatic top-up (**`ds24-tokens`**) — accepts path 1
only. A path-2 match is a good guess about who bought something; it is not
permission to charge a card.

An unattributed order is a support ticket. A wrongly attributed one is a customer
looking at somebody else's purchase, and it is the more expensive of the two.

## Step 3 — a purchase without an account must still work

Let people buy from the public pricing page without signing in first. That is
how most of them arrive, and forcing an account before payment costs sales.
Path 3 above is what makes it safe: the order waits, and the first sign-in from
that address claims it.

## Step 4 — the thank-you page

Digistore24 sends the buyer to a URL of yours after payment, with the order id
in it. Two rules:

- **It is public.** The buyer has no session yet. Do not put anything behind it
  that assumes one.
- **Do not grant access from it.** It is a browser hitting a URL — anybody can
  hit it. Access comes from the IPN, which is signed. The thank-you page says
  "thank you, it is on its way / here is how to sign in", nothing more.

**Digistore24 stores public https URLs only.** A `localhost` thank-you URL is
rejected outright ("Please only use secure URLs with https://"). On a hosted
platform your app URL is already public, so this is a non-issue; on a laptop it
needs a public redirect helper or a tunnel.

## Step 4a — test payments while unapproved (the testpay key)

A product that is not marketplace-approved yet can only be bought as a **test
purchase**. There are two ways to unlock one, and they suit different places:

- **The test-purchase cookie** — set once in the vendor's browser (Digistore24's
  help centre has the link). Per-browser, expires. The right tool on any
  domain a customer could also reach.
- **The testpay parameter** — fetched via the API and appended to the buy URL,
  so the unlock travels with the link instead of living in a browser:

  ```
  POST https://www.digistore24.com/api/call/getTestpayKey/format/json
  Header: X-DS-API-KEY: <the key>
  ```

  Undocumented, but real. The response carries `testpay_key`,
  `get_param_name` and `expires_at`. Append
  `?<get_param_name>=<testpay_key>` to the buy URL (the NAME comes from the
  response — never hardcode it) and the checkout opens in test-payment mode,
  approved or not. Sending `do_recreate=1` rotates the key: a new one is
  issued and every old copy stops working.

Four guardrails, all load-bearing:

- **Development/preview only — never on a URL a customer can reach.** A
  checkout carrying this parameter takes test "payments": whoever clicks it
  gets the product for free. Gate it on your environment with an allowlist
  (anything not clearly development counts as production and refuses), and
  append it at render/click time.
- **Never into a cached or shared buy URL.** If buy URLs are cached (Step 1),
  cache the clean URL and append the parameter after the cache — a decorated
  URL in a shared cache is served to everybody.
- **The key is account-level — treat it like a secret.** It works on EVERY
  checkout URL of this vendor account, live ones included. Keep it out of the
  repo and out of deployed configuration.
- **Rotate before go-live** (`do_recreate=1`) — see **`ds24-golive`**.

## Step 5 — prove it

1. Create a buy URL and open it. The checkout page must show **your** price,
   currency and interval — if it shows something else, the payment plan did not
   travel.
2. Do a **test purchase** — with the Digistore24 test-purchase cookie set, or
   in a development environment with the testpay parameter appended (Step 4a).
3. Check that the IPN arrived and that the order came out **attributed to the
   right account**. Attribution is the part that looks fine until it is not.
   Test purchases arrive with `api_mode=test` in the IPN payload — process
   them like live ones (that identical path is what the test proves).

## Step 6 — what comes next

- **`ds24-ipn`** — the endpoint that receives what this checkout produces.
- **`ds24-entitlements`** — turning a paid order into "may use the product".
- **`ds24-tokens`** — if you sell prepaid credits rather than plans.
- **`ds24-golive`** — the real test purchase, end to end.

Say which one you are starting and start it.
