---
name: ds24-checkout
description: Use when building the buy button, pricing page or checkout link for a Digistore24 product — creating a signed buy URL with createBuyUrl, selecting the product's payment plan for the chosen way to pay, carrying the buyer's identity through to the IPN, and the thank-you page. Use it whenever the user mentions a buy link, checkout, pricing page, "how does the customer pay", or a purchase that arrives without anybody being able to tell whose it was.
---

# The checkout link

A Digistore24 checkout is a **signed, short-lived URL** you create through the
API and send the buyer to. It is not a static link with a product id in it.

## Step 0 — is it already there?

Search the project for `createBuyUrl`, `payment_plan` or a pricing page that
already links out to Digistore24. If it exists, do not rebuild it — check it
against Step 3 and Step 4 and fix only what is wrong.

## Step 0a — is this copy of the skill pack current?

Fetch `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
and compare with this pack's `VERSION`. Mention a mismatch in one sentence, then
carry on.

## Step 1 — the call

```
POST https://www.digistore24.com/api/call/createBuyUrl/format/json
Header: X-DS-API-KEY: <the key>
```

Body (form-encoded), and there are **two shapes** — which one you send depends
on whether the offer's price is already a payment plan on the product
(**`ds24-products`** Step 3 writes them, one per way to pay).

**The ordinary case — sell through the stored plan:**

```
product_id                = 512345
valid_until               = 24h
settings[plan]            = 991      # the payment plan for THIS way to pay
settings[hide_plans]      = Y        # the buyer already chose on your page
payment_plan[template]    = 991      # see below — sent to be VALIDATED, not to price
```

No amounts at all. The stored plan prices the sale, which is what makes your
page, the product's own order form and the buyer's own interval switch charge
the same number.

**The exception — price it here.** Three cases, and only these:

| | |
|---|---|
| an **upgrade or downgrade** (`payment_plan[upgrade_order_id]`) | the price exists for this buyer against a purchase they already hold; there is nothing to store it in |
| a **free trial** (`payment_plan[test_interval]`) | same shape |
| a stored plan that **no longer matches your price list** | somebody edited a price and did not sync — send your own number, not a stale one |

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

🚨 **Why `payment_plan[template]` travels with the first shape.** Digistore24
refuses a template that does not belong to the product it was called for, with
`payment_plan_not_found` — so a plan id that survived a deletion over there
fails **loudly**, where `settings[plan]` alone would be ignored in silence and
the buyer would meet whatever plan the product does have. Because no
`first_amount` accompanies it, its values are then dropped again and the stored
plan does the pricing, which is what you want.

⚠️ **And that error takes out the whole offer, not one card**: the plan belongs
to the product, so every buy button of that offer — every language — fails at
once. Catch it, retry once without the plan and its `settings[plan]`, and log
which plan and which product. The sale goes through at your own price; the
vendor gets told to re-run the sync.

🚨 **Do NOT match on `payment_plan_not_found`.** That name is what the error is
called inside Digistore24; it is a message KEY, translated before it leaves the
server. Measured against a real account on 2026-09-09, what arrives is HTTP 404
with German prose:

```
"Ungültige Bezahlplan-ID: 999999999 - Bezahlplan nicht vorhanden
 oder nicht für das gewählte Produkt.", code 4
```

Match on **the plan id you sent appearing in the message** instead. Digistore24
echoes it in every language and it is unique to that call. A detector written
against the marker matches nothing, the retry never fires, and the page you
wrote it to protect goes dark exactly when it is needed — which is the one thing
no test of your own can show you, because your test raises the error itself.

🚨 **Never send `payment_plan[template]` alone hoping it prices the call.**
Without a `first_amount` Digistore24 discards the entire `payment_plan` — the
template's resolved values included. That is right for the first shape and
fatal if you meant the second and forgot a field.

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

⚠️ **Then the cache key has to include the language AND the way to pay**, not
just the offer key. One row per key means the German and the English URL — or
the monthly and the yearly one, which carry different `settings[plan]` values —
evict each other on every page view and, in between, the cache serves one
buyer's checkout page to another. `"<offerKey>:<paymentOption>:<language>"`.
Drop the middle part where the offer has only one way to pay, so nothing about
a single-price offer changes.

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

🚨 **"No `custom`" is not a diagnosis, and reading it as one is the mistake.** It
has at least two causes that look identical in the log: a buyer of yours who was
signed out when they clicked (you had no member id to write), and somebody who
never went through your app at all — the Digistore24 product has an order form of
its own, on a marketplace once approved, and a purchase made there carries
nothing you wrote.

⚠️ **The amount does not tell them apart**, and an older copy of this pack said
it did. Once the product carries your payment plans (**`ds24-products`**,
Step 3), a purchase on its own order form pays the same number as one through
your link — which is the point of writing them, and it costs you this
heuristic. Treat "no `custom`" as *unattributed*, resolve it by e-mail under the
two refusals below, and never by price.

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
