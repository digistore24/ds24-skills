---
name: ds24-products
description: Use when connecting an app to a Digistore24 account for the first time — getting the API key in, creating the products to sell, registering the IPN webhook connection, and requesting marketplace approval. Use it whenever the user mentions a Digistore24 API key, "connect Digistore", creating products or plans, registering an IPN URL, or asks why Digistore24 never calls their webhook.
---

# Connecting the app to Digistore24

Nothing about a Digistore24 integration works until three things exist on the
Digistore24 side: an API key your app can use, a product to sell, and an IPN
connection pointing at your endpoint. **Do this before anything else** — an IPN
handler nobody calls is untestable, and a checkout link for a product that does
not exist is a 404.

## Step 0 — what already exists?

Look before you ask:

- Is there a `DIGISTORE_API_KEY` in the environment or secret store?
- Is there a product registry in the project (a JSON/config file listing plans
  with prices)?
- Is `DIGISTORE_IPN_PASSPHRASE` set?

Then ask the user only what is genuinely missing. If all three are there, go to
Step 4 and check the connection rather than rebuilding it.

## Step 0a — is this copy of the skill pack current?

Fetch `https://raw.githubusercontent.com/digistore24/ds24-skills/main/VERSION`
and compare with the `VERSION` file in this pack. Mention a mismatch in one
sentence, then carry on.

## Step 1 — the API

```
POST https://www.digistore24.com/api/call/<FUNCTION>/format/json
Header: X-DS-API-KEY: <the key>
Body:   application/x-www-form-urlencoded
```

**The key travels in the header, never as a form parameter.** It is a secret:
environment variable or the platform's secret store, never in the code, never in
anything the browser receives.

The user creates it themselves in their Digistore24 account under
*Settings → API keys*. Ask for it, tell them where to put it, and do not try to
scrape it out of a browser session.

⚠️ **Tell them to give it WRITE permission** (Digistore24 calls it *writable*).
A key is scoped when it is created, and a read-only one reads products perfectly
and then fails on the two calls the app cannot do without: creating the products
and creating a checkout URL. Say it while they are on that screen — going back to
widen a key afterwards means creating a new one and replacing it everywhere.

## Step 2 — one price list, in your app

**Keep the plans in one file in your project** and let everything read from it:
the pricing page, the checkout, the entitlement check. One entry per **offer**,
and under it one entry per **way to pay**.

🚨 **Monthly and yearly are two ways to pay for ONE offer — not two offers.**
That is the single most consequential shape decision in this skill. A registry
that makes them two entries makes them two product keys, and then every access
check in the app has to name both: `hasAccess(m, "pro_monthly") ||
hasAccess(m, "pro_yearly")`, in every gate, for ever. One that is written with a
single key silently refuses half the buyers, behind a page that renders. Keep
them under one key and the question stays one question.

At Digistore24 the two axes behave differently, and this is why:

| axis | what it costs |
|---|---|
| **language** | one PRODUCT each — the order form's language is a property of the product (Step 3) |
| **way to pay** | one PAYMENT PLAN each, on that same product |

```
pro:
  name: "Pro"
  paymentOptions:                     # one payment plan each
    monthly: { priceCents: 3900,  billingInterval: 1_month }
    yearly:  { priceCents: 39000, billingInterval: 12_month }
  productIds:                         # one Digistore24 product per language
    de: null
    en: null
  payplanIds:                         # written back by your sync
    de: { monthly: null, yearly: null }
    en: { monthly: null, yearly: null }
```

⚠️ **Whatever CHECKS this file has to read the price off the way to pay, not
off the offer.** An offer that declares `paymentOptions` has no `priceCents` of
its own, so a validator written against the older shape tells every correctly
written price list that it has no price. Measured, on the first real run: a
warning that fires on correct input teaches the reader to skip warnings.

### Where the price lives

**Your file authors it. Digistore24 gets a copy, as payment plans.**

`data[amount]` on the product is deprecated and discarded, so no price is ever
set on the product itself — the API says so in its own warning: *"create a
payment plan instead"*. That is what `createPaymentplan` / `updatePaymentplan`
are for, and your sync writes one per way to pay (Step 3).

⚠️ **You may have read the opposite in an older copy of this pack**, which said
to keep every price out of Digistore24 and send it with each `createBuyUrl`
call. The reasoning was: a stored plan is a second place for the price, and it
cannot do trials, upgrades or vouchers. Half of that was right. What it missed
is that **your checkout link is not the only way into the product**:

- the product has an **order form of its own**, which no link of yours touches;
- an **affiliate** can send traffic straight to it;
- the buyer can **change their billing interval** from inside their purchase
  (`switch_pay_interval_url`, on every IPN) — and that switches to another
  *stored* plan, so with none there is nothing to switch to.

All three charge whatever plans hang on the product. A product with no plan of
yours does not have none: **Digistore24 gives it its own default** — about 27 €,
single payment, as seen on a real account in September 2026; look at the
vendor's product rather than trusting that number. On a **subscription** offer
such an order sends exactly one payment event and never a renewal or a
cancellation, so the buyer pays once and keeps the access **for ever**.

Writing the plans is what closes that. It is the second reason for this shape;
the first is that one offer now needs one product instead of two.

Two things follow, and both are easy to miss:

- **Tell the vendor what they will see in their backoffice**: their own prices,
  one plan per way to pay — **as a copy**. Change a price in the app and sync;
  change it over there and the next sync changes it back. A vendor who is not
  told this edits the wrong copy once and concludes the app forgets prices.
- **A purchase made on the product's own order form still carries no
  `tracking[custom]`**, so your IPN handler has to attribute it some other way
  (**`ds24-checkout`** Step 2 says why a missing `custom` alone does not prove
  that is what happened). What has changed is that it is no longer priced
  differently from yours, and no longer a subscription that never renews.

**If your app speaks more than one language, the entry holds one product id per
language** — not one id. The reason is Step 3; get the shape right here, because
changing it after the first sale means new products and new approvals.

**And if the app has more than one environment, keep one product SET per
environment** (dev / prod — staging only if it really exists). Products you
create against a preview or development URL are test articles: give them their
own ids in the registry, mark them visibly in the product name (`"Pro [DEV]"`
— the Digistore24 API has no tag field, the name is what a human sees in the
backoffice), and leave the live products' names clean. One set must never
claim the other's products — see the idempotency note below. A vendor who
only ever syncs against the live domain has one set, and that is fine.

## Step 3 — create the products

Two calls per product, in this order:

1. `createProduct` / `updateProduct` with the name, description and
   **`language`**. Write the returned product id back into your price list so
   the mapping is recorded, not re-derived.
2. `createPaymentplan` / `updatePaymentplan` **once per way to pay**, with
   `product_id`, `first_amount`, `currency`, `first_billing_interval`,
   `other_amounts`, `other_billing_intervals`, `number_of_installments`
   (`0` = open-ended subscription, `1` = single payment) and `position`. Write
   the returned `paymentplan_id` back too.

   Set `is_switching_allowed = Y` where the offer has more than one way to pay:
   that is what makes Digistore24's own `switch_pay_interval_url` lead
   somewhere, and it saves you building an upgrade flow for the commonest change
   a subscriber makes.

⚠️ **Do the two together, not in two passes.** A product that exists without its
plans has Digistore24's ~27 € default and an order form that charges it; the
window in which that is true should be one API call wide.

🚨 **And write an OWNERSHIP MARK while you are there.** `createProduct` and
`updateProduct` both take `data[note]`, a free internal note no buyer sees. Put
one machine-readable line in it — your app's own id and the environment.
Without it there is no honest answer to "did WE create this product?", and
Step 3b needs one. Do not derive that id from the app's name (vendors rename)
or from the internal product name (two apps built the same way collide on it):
generate it once, store it beside the price list, and never regenerate it.

🚨 **`note` keeps 47 characters and drops the rest — silently.** Measured
against a live account on 2026-09-09: a 120-character value came back cut
mid-word, with no error, no warning and nothing about it in the API
documentation. So keep the mark short — `myapp:1:<appId>:<env>` is about
30 characters — and **do not put the product key or the language in it**: they
are already in `name_intern`, which the listing returns beside the note.

A mark one character too long is not a shorter mark. It does not parse, every
product your app created reads as somebody else's, and your cleanup step
reports "nothing to remove" out of a comparison that found nothing because it
could not. Assert the length in a test.

⚠️ **`data[tag]` is not the way out** — it comes back on a product, and
`updateProduct` refuses to write it (HTTP 400, same measurement).

Two rules follow from 47 characters, and both are about whose field this is:

- **Never write over text you did not write.** There is no room to merge, so if
  the note holds something else, leave it and move on. That product then stays
  unmarked and your cleanup step will not touch it — the safe direction.
- **But treat your own PREFIX as yours even when it does not parse.** The cut
  can land inside a mark you wrote. Without this, a mark you once got wrong is
  permanent: it reads as the vendor's text for ever and can never be repaired.
  A marker you cannot fix is worse than one you cannot read.

### One product per offer AND language — this is the one people get wrong

*(And, to say it once more where it is easiest to get backwards: one product per
LANGUAGE, one payment plan per WAY TO PAY. The language axis multiplies
products; the way to pay does not.)*

**A Digistore24 product carries exactly ONE language, and that language is the
language of the ORDER FORM your buyer fills in** — the field labels, the
buttons, the payment-method names, the cancellation terms. It is
`data[language]` on the product.

**`createBuyUrl` has no language parameter.** Its arguments are `product_id`,
`buyer`, `payment_plan`, `tracking`, `valid_until`, `urls`, `placeholders`,
`settings` and `addons` — there is nothing in there to override the product's
language with, and no URL parameter does it either. So you cannot decide the
form's language at checkout time. You decide it by **choosing which product to
send the buyer to**.

An app whose interface speaks German and English therefore needs **two
Digistore24 products per offer**, one with `language=de` and one with
`language=en`, and the checkout picks by the visitor's language. Send everybody
to one product and half your customers are asked for their card details in a
language they did not choose — which is exactly where a purchase is abandoned.

Three consequences worth writing into whatever you build:

- **Set `data[language]` explicitly on every product.** Left out, Digistore24
  falls back to the language of the API session — nobody's deliberate choice,
  and the usual cause of a German shop showing English order forms.
- **Cover every language your app has.** One that is missing should still sell
  (fall back to another product rather than showing a dead button) — but say so
  in your sync's output, because nothing else ever will: the app renders fine,
  the checkout opens, the purchase completes.
- **Each language product is approved separately**, at the marketplace its own
  language belongs to. See the **`ds24-golive`** skill.

Your product *copy* is a separate question. Sending the same name and
description to both products is a perfectly good default — the *form* around it
is what has to follow the buyer.

Make this **idempotent**: run it twice and the second run updates rather than
creating a duplicate. Key it on your own product key **plus the language** —
and, if you keep separate sets per environment, **plus the environment**
(`pro__en__prod`) — each product needs its own stable handle. Never key on the
display name, which is the same for both languages and changes with the copy.

**Deleting a product from your list does not unpublish it by itself.** The
product Digistore24 already knows stays buyable — through its own order form and
through any checkout link that exists — until something removes it. That is
Step 3b.

🚨 **Which is why the moment to ask is BEFORE you create, not after.**
`deleteProduct(product_id)` exists, so this is not permanent in the way an
older copy of this pack said. But it is only clean while the product has **never
sold**: one that has taken money can only be deactivated, because its buyers'
refunds, chargebacks and cancellations still arrive as IPNs naming its id and
your handler still has to receive them. A product that has taken money is a
decision the vendor keeps.

So the first time your sync would create anything: **print what would be
created, by name, say what it costs, and wait for a yes.** Then create. Runs
after that have ids on file and create nothing, so this is one question at one
moment, not a prompt anybody learns to click through. If some entries are drafts
rather than offers, give your list a flag that keeps them out of the sync
instead of asking the user to delete text they still want.

## Step 3b — remove what is no longer offered

An entry taken out of the price list leaves a product behind. Cleaning those up
is worth doing — an account that fills with abandoned products is how a vendor
ends up selling something they forgot about — and it is the one step where
getting it wrong destroys somebody else's work.

**Never act on anything but your own ownership mark** (Step 3). Not on the
product name, not on the internal name, not on the folder: a vendor's account
holds products from before your app existed, products of a second app built the
same way, and products a support agent made by hand. "Probably ours" is not good
enough for a delete.

Then, per product that carries your mark and is no longer in the price list:

1. **Ask whether it has sales** (`listPurchases` with its `product_id`). If you
   cannot ask, treat that as "it has" — the cheap mistake is an inactive product
   too many.
2. **No sales → `deleteProduct`.** If it fails for any reason, fall through.
3. **Sales, or a failed delete → `updateProduct` with `data[is_active] = N`.**
   The product stops being buyable and stays in the account. Say to the vendor
   that its entry should stay in the price list as a parked one, or its id
   leaves your IPN registration and its buyers' refunds stop arriving.

🚨 **And do nothing at all when you could not read the notes.** If the product
listing came back without them, ownership cannot be established, and "zero
products to remove" is then not an answer — it is silence. Say so and stop.
Proving the walk ran is not proving the comparison did.

**Behind an explicit flag, always.** Unlike creating, this is not a first-run
question: a renamed key or a typo in the price list would otherwise delete a
live product on an ordinary sync. List what would go, change nothing, and let
the user ask for it.

⚠️ **And make sure the flag still works when the price list is EMPTY.** The
natural place to put a "nothing to sync" refusal is before anything talks to
Digistore24 — and then whoever removes their LAST entry keeps that product in
the account with no way to remove it. Measured: it is the easiest of these
mistakes to make, because every test has at least one product in it. "Nothing
to sync" and "nothing to clean up" are two questions.

## Step 4 — register the IPN connection

This is the step that gets forgotten, and its symptom is "the purchase worked
but nothing happened in the app".

- `ipnSetup` registers the endpoint. Digistore24 **validates it immediately**
  with a `GET` and insists on HTTP `200` — a redirect (301/302) fails too.
- **The URL must be public `https`.** Digistore24 refuses `http` and refuses
  `localhost` outright.
- Digistore24 either generates the **IPN passphrase** or takes yours. Whichever
  it is, it must end up in the app's environment as
  `DIGISTORE_IPN_PASSPHRASE` — it is the shared secret the signature is
  computed with, and without it every IPN is correctly rejected.

The call takes these parameters, and two of them decide whether events ever
arrive:

| | |
|---|---|
| `ipn_url` | your endpoint, public https |
| `name` | what the connection is called in the backoffice |
| `domain_id` | **the identity of this connection** — see below |
| `product_ids` | which products it covers — comma-separated ids, or `all` |
| `sha_passphrase` | your own, or `random` to have one generated and returned |

### `ipnSetup` is also the update — the `domain_id` decides

There is no separate update function. Digistore24 looks a connection up by
**(merchant, API key, `domain_id`)**: same id → the existing connection is
updated, unknown id → a second connection comes into being. That is what makes
the call idempotent, and it is why the id has to be **written down** (an
environment variable, a settings row) rather than re-derived from something
that changes.

**And it has to be unique.** This is the part that gets skipped, and it fails
invisibly. A generic value — `test-local-1`, `local-app`, `myapp`, `production`
— is not a name, it is a collision with the user's **own** other project: the
two do not get two connections, they take turns overwriting one. The second
setup silently re-points the first app's IPN at its own URL, and from then on
the first app's purchases arrive nowhere. Both runs report success.

So put a random tail on it and store it:

```
test-local-diw2hvnz73
myapp-prod-k7f2m9x1qc
```

The readable part says which app it is; the tail is what makes it unique. Never
reuse one across two apps, and never change it just because the URL changed —
changing it is how you get a second, duplicate connection.

### `product_ids` — which purchases this connection reports

Comma-separated Digistore24 product ids: `product_ids=111,222,333`. The default
is `all`, the whole account.

**Prefer naming the actual products.** A vendor's account usually holds more
than the app you are building — an older funnel, a second app, somebody else's
launch — and a connection scoped to its own products is what lets two apps of
the same vendor be connected at the same time.

`all` is acceptable, on one condition that belongs in the endpoint: **a purchase
of a product your app does not know must be ignored, not guessed at.** Record it
if you like, grant nothing for it. An endpoint that maps an unknown product onto
a default plan hands out access for a purchase that was never yours.

**On a hosted AI-builder platform this is the easy part**, and it is worth
saying to the user: the preview/production URL of a Lovable, Replit, v0 or Manus
app is already public https, so the endpoint can be registered directly. On a
laptop it cannot — a local address needs a tunnel first.

## Step 5 — before real money: approval

A product can be **test-purchased** immediately, by the vendor, with the
Digistore24 test-purchase cookie set — or, in a development environment, with
the testpay parameter on the buy URL (**`ds24-checkout`**, Step 4a). That is
how you verify the whole chain without moving money.

Selling to the public through a **reseller** additionally needs **marketplace
approval** (`approval_status=pending`) — request it only once the description
and the app are genuinely finished, because a half-built product gets rejected
and the second attempt is slower.

**A Direct Seller has no approval step at all.** Only siteowners 1 (Germany),
2 (USA), 3 (UK) and 4 (Ireland) are resellers and approve products; a vendor
selling on their own account has nothing to request and nothing to wait for.
Check which you are dealing with before you promise the user an approval step —
or build a reminder they can never satisfy.

Whether it was granted is readable: `listProducts` / `getProduct` items carry
`approval_status_list`, one entry per marketplace. The **`ds24-golive`** skill
(Step 4) has the field, its value set and its pitfalls — and walks the whole
go-live, including the test purchase.

## Step 6 — prove the connection

Do not report success from an API response alone. Check that:

1. `GET <your IPN url>` answers **200** from the public internet.
2. The product appears in the user's Digistore24 account.
3. `DIGISTORE_IPN_PASSPHRASE` is set in the app's environment — not just in a
   local file the deployed app never reads.

Then prove the endpoint itself — the **`ds24-ipn`** skill says what has to hold
and how to check it on this platform.

## Step 7 — `getPurchase`: look an order up yourself

When the user says *"I bought it and nothing happened"*, do not send them into
their Digistore24 backoffice to read a status out to you. Ask the API:

```
POST https://www.digistore24.com/api/call/getPurchase/format/json
Header: X-DS-API-KEY: <the key>
Body:   purchase_id=ABC12345
```

It returns Digistore24's own view of that one order — status, product, buyer,
billing type, next payment, and the management links (invoice, receipt, stop
rebilling, update payment details). It changes nothing, so it is safe to call
while diagnosing. `listPurchases` is the same thing for many, filtered (e.g. by
buyer email).

**Build it into the app as a small admin/CLI helper the first time you need it**
— it turns an argument into a lookup. The answer sorts the complaint into cases
that have nothing to do with each other:

| What `getPurchase` says | What is actually wrong |
|---|---|
| **Unknown id / no data** | there was no purchase, or it was made in a different Digistore24 account than the key you are using. The app is fine |
| **It knows the order, your app does not** | it was paid and no IPN reached you. Look at the connection: is the registered URL still answering, did another project overwrite the `domain_id`, is this product inside the connection's `product_ids`? |
| **Both know it, but access is missing** | the IPN arrived and the event→access mapping is where the fault is → **`ds24-entitlements`** |

A rejected IPN is a fourth case and has its own tool — the signature check in
**`ds24-ipn`**, run against the raw body that arrived.

## Step 8 — what comes next

- **`ds24-ipn`** — the endpoint that receives the events (build it now if it
  does not exist).
- **`ds24-checkout`** — the buy link, selecting the way to pay.
- **`ds24-golive`** — the test purchase that proves the whole chain.

Say which one you are starting and start it.
