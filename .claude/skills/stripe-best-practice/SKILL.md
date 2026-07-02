---
name: stripe-best-practice
description: Hard rules for wiring Stripe Checkout into the 抽成制理髮師預約平台 (commission-based barber booking platform) — dynamic per-booking line items, currency-aware pricing (unit_amount = price × 10^currency_minor_units from platform_settings), a webhook that flips a booking from pending_payment to `paid` (the 20/80 split is computed later at payout-build time, NOT in the webhook), raw-body signature verify, idempotency via a status guard, and the middleware exemption for `/api/stripe/webhook`. Use whenever a student is building M2.1 (`m2.1-buyer-to-admin-payments`), debugging a webhook that "fires but never lands," getting a `400 signature verification failed`, or asking why a NT$500 cut charged NT$50,000. Apply these proactively — stop the student before they break one.
---

# Stripe Best Practice (Barber Booking — pay-now Checkout, deferred settlement)

This course's Stripe usage is **pay-now at booking**: a customer confirms a slot, you create a **Stripe Checkout Session with a dynamic `price_data` line item** (the service's price, in `platform_settings.currency`), the customer pays on `checkout.stripe.com`, and a **webhook** flips the booking from `pending_payment` to `paid` and stamps `paid_at`. **The webhook does NOT compute a split or write any fee row** — the **20% platform / 80% shop** split is computed later, when the admin **builds a payout** (M2.2), from the picked paid bookings × the versioned `commission_rates`. The platform collects **100% into its own Stripe account** — there is **no Stripe Connect**, no payout API; the admin bank-transfers each shop manually. So the rules below are about **getting the one webhook right** and **not corrupting money** — every one maps to a real failure mode.

When you (Claude Code) guide a student through M2.1, **apply these rules proactively** — don't wait for them to ask. If you see them about to break one, stop them and explain why.

> **Adapted from a credits/top-up reference Stripe skill.** That product sold **fixed price tiers** and charged the platform's *own* users; ours is a **dynamic per-booking line item**, with the commission split computed later at payout-build time. The transferable rules (webhook-is-truth, raw-body verify, idempotency, middleware exemption, metadata-not-email) carry over verbatim in spirit; the **immutable-Price rule becomes "snapshot the price onto the booking row"** (Rule 8), and there's a **new currency-aware `unit_amount` rule** (Rule 0) that does not exist in the USD reference. Cross-ref [[stripe-mysite]] for an existing TWD + Next.js + Supabase Checkout implementation you can lift the signature/zero-decimal handling from.

---

## Execution mode: Cowork vs CLI

The hard rules apply identically in both — only the tooling around them differs.

| Operation | CLI mode | Cowork mode |
|---|---|---|
| Confirm you're in **sandbox**, not live | `stripe config --list` / check the key prefix is `sk_test_` | Stripe MCP `mcp__claude_ai_Stripe__*` — **the consent page defaults to LIVE; switch it to sandbox** |
| Receive webhook events locally | `stripe listen --forward-to localhost:3000/api/stripe/webhook` (prints the `whsec_…`) | not applicable — test via a deployed Vercel preview + the dashboard endpoint secret |
| Re-send an event (idempotency test) | `stripe events resend <evt_…>` | Stripe dashboard → Developers → Events → **Resend** |
| Create the webhook endpoint | dashboard → Developers → Webhooks → Add endpoint | **dashboard step** — the Stripe MCP does **not** manage webhook endpoints (2026) |

> **Two MCP caveats (2026), state them up front:** the **Stripe MCP does not create/manage webhook endpoints** (a dashboard step), and the **Vercel MCP does not set env vars** (also a dashboard step). So `STRIPE_SECRET_KEY` / `STRIPE_WEBHOOK_SECRET` land in **Vercel env via the dashboard**, and the webhook endpoint is created **in the Stripe dashboard** — neither is automatable here.

---

## Hard rules

### Rule 0 — `unit_amount` = `price × 10^currency_minor_units` — read the exponent from `platform_settings`, never hard-code ×100

> **The rule:** `unit_amount` is **`price × 10^currency_minor_units`**, where `currency_minor_units` comes from `platform_settings`. For the default **TWD** config (`currency_minor_units = 0`) that means **×1** — a NT$500 cut is `unit_amount: 500`, **not** `50000`. For a **USD** config (`currency_minor_units = 2`) it would be **×100** ($5.00 → `500`). Do **not** hard-code either factor — read the exponent from config.

**Why:** Stripe classifies currencies as decimal (USD, EUR — smallest unit is 1/100) or **zero-decimal (TWD, JPY, KRW — smallest unit is 1 dollar/yen/won)**. For a zero-decimal currency Stripe charges `unit_amount` *as-is*; for a 2-decimal one it expects cents. Every tutorial, every Stripe code sample, and every other `price * 100` you've ever written assumes cents — so the reflex is to write `service.price * 100`, which under TWD **charges the customer 100× the real price** (a NT$500 haircut bills **NT$50,000**) and silently corrupts the monthly split downstream. The fix is to drive the factor off `platform_settings.currency_minor_units` (TWD's `0` → ×1) instead of assuming TWD or cents. This is the single most expensive foot-gun in the whole course, and it's invisible in test mode unless you read the amount.

**How to apply:**
```ts
// ✅ correct — currency-aware: read the exponent from platform_settings
const { currency, currency_minor_units } = await getPlatformSettings();
unit_amount: service.price * 10 ** currency_minor_units,   // TWD (0) → 500 charges NT$500; USD (2) → 50000 charges $500
// ❌ wrong — the USD/cents reflex; under TWD this charges NT$50,000
unit_amount: service.price * 100,
```
- `services.price` is stored as a **whole integer in `platform_settings.currency`** (e.g. `500` TWD) everywhere — the DB column, the `bookings.price` snapshot. The `unit_amount` math then applies the currency exponent. Never a cents value, never a float.
- After your first test payment, **read the amount in the Stripe dashboard** (or the `payment_intent.amount`) and confirm it matches the currency math (TWD `500`, not `50000`). A wrong magnitude here cascades into a wrong monthly settlement in M2.2.
- This is a known foot-gun lifted from [[stripe-mysite]] — that skill has the same zero-decimal TWD handling in a sibling Next.js + Supabase project; reuse its pattern (ours just reads the exponent from config instead of hard-coding it).

---

### Rule 1 — The webhook is the source of truth for a booking becoming `paid`. The success page never mutates.

> **The rule:** **Only** `POST /api/stripe/webhook` flips a booking from `pending_payment` to `paid` (and stamps `paid_at`). The `/bookings/success` redirect page is **UX-only** — it may *poll* the booking's status to show "已付款", but it must never `update` the booking.

**Why:** The success redirect is unreliable and forgeable. The customer can **close the tab right after paying** (no redirect fires), the network can blip on the way back, or a malicious customer can hit `/bookings/success?session_id=…` directly to fake a confirmation. The webhook is the *only* signal Stripe guarantees: payment actually settled. If the success page is what marks the booking paid, you get **paid bookings stuck on `pending_payment`** (the slot stays held but the money is never recorded) and/or **unpaid bookings marked `paid`** (you'll owe a shop 80% of money you never collected when the admin builds a payout that includes it). Make the webhook the single writer and the whole class of bugs disappears.

**How to apply:**
- `/api/stripe/webhook`: the **only** place that runs `update bookings set status='paid', paid_at=now() where id=…`. No split, no fee columns — those don't exist on `bookings` (Rule 9).
- `/bookings/success`: read-only. Poll `select status from bookings where id=…` and render "確認中…" until the webhook lands, then "已付款". No writes.
- If a student wires the status change into the success page "to make it feel instant," stop them: 「成功頁只能讀狀態，不能改 — 真正把預約變 paid 的只有 webhook。」

---

### Rule 2 — Read the RAW request body before verifying the signature. Never `request.json()` first.

> **The rule:** In the webhook route, get the **raw text body** (`const body = await request.text()`) and pass that exact string to `stripe.webhooks.constructEvent(body, sig, secret)`. Do **not** call `request.json()` (or any body parser) before verifying.

**Why:** The Stripe signature is an HMAC over the **exact bytes** Stripe sent. `request.json()` parses and **re-serializes** the body — key order, whitespace, and number formatting change — so the bytes you'd hand to `constructEvent` no longer match the signature, and **every** event fails with `400 No signatures found matching the expected signature`. The symptom is maddening: the payload looks perfectly valid, but verification always fails. It's not the secret — it's that the body was already parsed.

**How to apply (Next.js App Router):**
```ts
export async function POST(req: Request) {
  const body = await req.text();                         // RAW — before any parse
  const sig = req.headers.get('stripe-signature')!;
  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(body, sig, process.env.STRIPE_WEBHOOK_SECRET!);
  } catch (err) {
    return new Response(`Webhook signature verification failed`, { status: 400 });
  }
  // ... only now parse event.data.object
}
```
- App Router route handlers do **not** auto-parse the body, so `req.text()` is enough. In the old Pages API you'd also need `export const config = { api: { bodyParser: false } }`. See [[stripe-mysite]] for the exact pattern.

---

### Rule 3 — Make the webhook idempotent with a status guard. Stripe retries any non-2xx.

> **The rule:** Scope the `paid` update to **only** bookings still in `pending_payment` (`… where id = booking_id and status = 'pending_payment'`). A re-delivered event then no-ops (0 rows affected) instead of re-stamping. Return `200 {received:true}` regardless. Optionally back it with a **UNIQUE** `bookings.stripe_payment_intent_id` as a hard backstop.

**Why:** Stripe **retries delivery** (with backoff, for up to ~3 days) on any response that isn't 2xx — and even on success it can occasionally deliver the same event twice. Without idempotency, a retry re-runs your transition and re-stamps `paid_at`. The status guard makes the second delivery a **no-op**: the booking is already past `pending_payment`, so the `where status='pending_payment'` matches nothing. (There's no split to double-count here — the split is computed later at payout-build time, Rule 9 — but `paid_at` should still be stamped exactly once.)

**How to apply:**
- On `checkout.session.completed`, run a single guarded update: `update bookings set status='paid', paid_at=now() where id = booking_id and status = 'pending_payment'`. A re-delivered event affects **0 rows** — already-paid is the no-op.
- **Optional hard backstop:** add a migration with `alter table bookings add column stripe_payment_intent_id text; create unique index … on bookings(stripe_payment_intent_id);` so a concurrent double-fire fails the second write. The minimal version relies on the status guard alone. (Migration, never a raw prod edit — [[supabase-best-practice]].)
- **Return 200 even on a duplicate / no-op** — a non-2xx tells Stripe to retry forever.

---

### Rule 4 — The `stripe listen` secret ≠ the dashboard-endpoint secret, and it rotates every restart.

> **The rule:** Local dev uses the `whsec_…` printed in the `stripe listen` banner. **Production (Vercel) uses the *dashboard endpoint's* signing secret** — a different, stable value. Don't paste the CLI secret into Vercel env.

**Why:** `stripe listen` mints a **fresh ephemeral signing secret each time you start it** — paste it into Vercel and every prod event 400s (and tomorrow your *local* events 400 too, because it rotated). The dashboard endpoint (Developers → Webhooks → your endpoint → **Signing secret**) is stable and is the one prod must use. Mixing them up is the second-most-common webhook failure after Rule 2.

**How to apply:**
- **Local:** `stripe listen --forward-to localhost:3000/api/stripe/webhook` → copy *its* `whsec_…` into your local `.env.local` only.
- **Vercel prod:** dashboard → Developers → Webhooks → the endpoint pointing at `https://<your-app>/api/stripe/webhook` → reveal **Signing secret** → set `STRIPE_WEBHOOK_SECRET` in **Vercel env** (Production scope) → redeploy.
- When you attach the custom domain in M3, you create a **new** endpoint on the new URL → a **new** `whsec_` → update Vercel env. See [[stripe-go-live]].

---

### Rule 5 — Auth middleware MUST exempt `/api/stripe/webhook`.

> **The rule:** Add `/api/stripe/webhook` to your middleware's public allowlist (or exclude it from the matcher). Stripe is not a logged-in user; if auth middleware runs on it, the request never reaches your handler.

**Why:** A typical Next.js auth middleware 307-redirects unauthenticated requests to `/login`. Stripe's webhook POST carries **no session cookie**, so the middleware redirects it — and your route logs show **zero hits** while the Stripe dashboard shows the event "delivered." You'll burn an hour staring at a handler that's never called. The fix is one line.

**How to apply (Next.js `middleware.ts`):**
```ts
export const config = {
  matcher: [
    // run on everything EXCEPT the webhook (+ static assets, etc.)
    '/((?!api/stripe/webhook|_next/static|_next/image|favicon.ico).*)',
  ],
};
```
- Or, if the middleware checks a path allowlist, add `/api/stripe/webhook` to it explicitly.
- **Symptom → cause:** Stripe says "delivered" but your logs are empty → it's the middleware redirect, not your handler. Check the matcher first.

---

### Rule 6 — Stash the `booking_id` join key in BOTH `metadata` AND `client_reference_id` at session creation.

> **The rule:** When you create the Checkout Session, set `metadata: { booking_id, customer_id }` **and** `client_reference_id: booking_id`. The webhook reads `session.metadata.booking_id` to find the booking. Validate `booking_id` is present; return `400` **only** if it's structurally missing (that's your own bug, not a forgery). (The barber/slots aren't carried — they're derivable from the booking via `service_id → services.barber_id` and `booking_slots`. There is **no `start_slot_id`** on the booking to carry.)

**Why:** The webhook gets a Stripe event, not your request context — it needs a way back to *which booking* this payment was for. `booking_id` alone is enough: everything else (the barber, the slots, the price) hangs off the booking row in your DB. `metadata` is set by **your authenticated server** at session-creation time and is echoed back on the event, so it's a trustworthy join key. Storing it in **both** places is belt-and-suspenders: `client_reference_id` is a first-class top-level field, `metadata` is the flexible bag. All metadata **values are strings** — `booking_id` comes back as a string; cast if your column is an int/uuid.

**How to apply:**
```ts
metadata: { booking_id, customer_id },   // strings on the way back
client_reference_id: booking_id,
```
- Webhook: `const { booking_id } = event.data.object.metadata;` — if `booking_id` is missing, that's a bug in *your* session-creation code → `400` and log loudly. (A missing metadata key is never the customer's doing.)

---

### Rule 7 — Test cards only in test mode; real cards only in live mode.

> **The rule:** In **sandbox**, pay with Stripe test cards — `4242 4242 4242 4242`, any **future** expiry, any 3-digit CVC, any ZIP. **Never a real card in sandbox; never a test card in live.**

**Why:** A test card in live mode is **declined** (`card_declined`), so it looks like your integration is broken when it's just the wrong mode. A real card in sandbox does nothing useful and risks confusion about whether money moved. Keeping the mode and the card class matched is what makes "did the booking confirm?" a clean signal during M2.1 testing.

**How to apply:**
- M2.1 testing: confirm the key prefix is `sk_test_` / `pk_test_`, then pay with `4242 4242 4242 4242`, `12/34`, `123`.
- Other useful test cards: `4000 0000 0000 9995` (insufficient funds → declined), `4000 0027 6000 3184` (3DS required). Full list: https://stripe.com/docs/testing.
- Going live ([[stripe-go-live]]) swaps to `sk_live_`/`pk_live_`; from then on test cards stop working and you smoke-test with a **real** card + immediate refund.

---

### Rule 8 — Snapshot the price onto the booking row at creation. The service price can change later.

> **The rule:** When the booking is created (and the Checkout Session built), copy the service's current price into **`bookings.price`**, and charge **that** snapshot. The line item's `unit_amount` is `price × 10^currency_minor_units` (Rule 0) of that same snapshot.

**Why:** The reference course used Stripe **Price objects**, which are **immutable** — a captured price can never silently change. We use **dynamic `price_data`** (the service's live price at booking time), which has the *opposite* property: if a barber later edits `services.price`, any code that re-reads the service would retroactively change what a **past** booking appears to have cost — corrupting the monthly settlement M2.2 sums. The fix is to make the booking carry its own immutable price: **snapshot it once, at creation**, and never read back through the service for a historical booking.

**How to apply:**
```ts
// at booking creation:
const price = service.price;                        // read once (integer in platform_settings.currency)
await db.bookings.insert({ ..., price, status: 'pending_payment' });
// build the session from the SAME snapshot:
line_items: [{ price_data: { currency,              // from platform_settings (e.g. 'twd')
  product_data: { name: `${service.name} @ ${barber.name}` },
  unit_amount: price * 10 ** currency_minor_units }, quantity: 1 }],   // Rule 0: currency-aware
```
- M2.2's settlement sums **`bookings.price`** for paid bookings — never `services.price`. The service price is "today's price"; the booking row is "what this customer actually paid."

---

### Rule 9 — The webhook ONLY records the booking as paid. The 20/80 split is computed later at payout-build time — never in the webhook.

> **The rule:** On `checkout.session.completed` **with `payment_status === 'paid'`** the webhook does exactly one thing: flip `pending_payment → paid` and stamp `paid_at`. It does **NOT** compute a split, write a `platform_fee`/`barber_amount` (those columns don't exist), touch `payout_id`, or insert any `transactions`/fee row (there is no `transactions` table). The split is computed later, when the admin **builds a payout** (M2.2), from the picked `paid` bookings × the versioned `commission_rates` row, and snapshotted onto a `payouts` batch row.

**Why:** Separating "money came in" (a `paid` booking) from "what we owe the shop" (a payout batch) keeps the commission **versioned and re-runnable**: the rate can change over time (`commission_rates.effective_from`), and the build applies whichever rate was effective. If the webhook baked a fixed split onto each booking, a later rate change or a re-build couldn't be reconciled. So the booking row records only the **price paid** (the snapshot, Rule 8) and `paid_at` (an audit + a filter the admin uses when batching), leaving `payout_id` NULL (= owed). The math lives in M2.2's `owed_bookings` view + the `payouts` snapshot — computed when the admin picks owed bookings into a per-shop batch, deriving `shop_cut = gross - round(gross × platform_pct)` so platform + shop reconcile to the cent.

**How to apply:**
```ts
if (event.type === 'checkout.session.completed' && session.payment_status === 'paid') {
  // the ONLY mutation — no split, no fee row, no transactions insert, no payout_id:
  await db.bookings.update({ status: 'paid', paid_at: new Date() })
    .eq('id', booking_id).eq('status', 'pending_payment');   // status guard = idempotent (Rule 3)
}
```
- Guard on `payment_status === 'paid'` — a `checkout.session.completed` can arrive for an async/unpaid method; only `paid` flips the booking.
- `bookings.status` is **3 states** (`pending_payment | paid | cancelled`) — there's no `payout_*` status; "settled" is **derived from `bookings.payout_id`** (NULL = owed), which M2.2 stamps, not the webhook.
- The commission rate is **not** a code constant here — it lives in the `commission_rates` table (default 20%, effective `2026-01-01`) and is read when the M2.2 payout is built, not by the webhook. See [[m2.2-admin-to-seller-payment]].

---

### Rule 10 — Identify the booking via `metadata`, never by email or a customer lookup.

> **The rule:** The webhook resolves *which booking* from **`session.metadata.booking_id`** — never by matching `customer_email`, looking up a Stripe Customer, or trusting anything the customer could set.

**Why:** Email and customer-object fields are **customer-influenced and ambiguous** — two customers share an email, a guest checkout has none, someone edits the email on the Stripe page. `metadata.booking_id` was written by **your authenticated server** at session creation (Rule 6) and **cannot be forged by the customer**, so it's the only safe join key. Matching on email is how the wrong booking gets confirmed (or someone confirms a booking that isn't theirs).

**How to apply:**
- Webhook: `const bookingId = event.data.object.metadata.booking_id;` → `select … where id = bookingId`. Done.
- Never `where customer_email = session.customer_details.email`. If `metadata.booking_id` is absent, that's a Rule 6 bug → `400`, not a fallback to email.

---

## Dynamic Checkout Session shape (the M2.1 reference)

```ts
const { currency, currency_minor_units } = await getPlatformSettings();   // Rule 0
const session = await stripe.checkout.sessions.create({
  mode: 'payment',
  line_items: [{
    price_data: {
      currency,                              // from platform_settings (e.g. 'twd')
      product_data: { name: `${service.name} @ ${barber.name}` },
      unit_amount: booking.price * 10 ** currency_minor_units,  // Rule 0: currency-aware ; Rule 8: snapshot
    },
    quantity: 1,
  }],
  metadata: { booking_id, customer_id },  // Rule 6 + Rule 10 (barber/slots derivable from the booking — no start_slot_id)
  client_reference_id: booking_id,
  success_url: `${origin}/bookings/success?session_id={CHECKOUT_SESSION_ID}`,  // Rule 1: read-only
  cancel_url: `${origin}/barbers/${barber_id}`,
});
```

---

## Things to actively watch out for

1. **A NT$500 cut charges NT$50,000** → Rule 0 (under TWD `currency_minor_units = 0`, so `unit_amount = price × 1`; drop the hard-coded `* 100`).
2. **`400 No signatures found matching the expected signature`** → Rule 2 (you called `request.json()` before verifying) — *or* Rule 4 (wrong/rotated secret in Vercel env).
3. **Stripe dashboard says "delivered" but your route logs show zero hits** → Rule 5 (middleware redirected the webhook to `/login`).
4. **A retried event re-stamps `paid_at`** → Rule 3 (no status guard on the update).
5. **Paid booking stuck on `pending_payment` (customer closed the tab)** → Rule 1 (you flipped it in the success page, not the webhook).
6. **Webhook can't find the booking** → Rule 6/10 (missing `metadata.booking_id`, or you matched on email).
7. **A barber edits a service price and old bookings' amounts shift** → Rule 8 (you re-read `services.price` instead of the `bookings.price` snapshot).
8. **The webhook tries to compute/write a split, a fee/`transactions` row, or a `payout_id`** → Rule 9 (the webhook only records `paid` + `paid_at`; the 20/80 split is computed later at payout-build time from `commission_rates` — there are no `platform_fee`/`barber_amount` columns, no `transactions` table, and the webhook never touches `payout_id`).
9. **Test card declined in production** → Rule 7 (you're in live mode — use a real card, see [[stripe-go-live]]).
10. **Stripe MCP charged a real card / created a live object** → the MCP consent page defaulted to **LIVE**; switch it to **sandbox** before M2.1 testing.

---

## Out of scope for this course (real prod, not enforced here)

- **Stripe Connect / automated payouts** — we collect 100% into the platform account and pay shops by manual bank transfer (M2.2). No `transfer`/`payout` API.
- **Refund → settlement reversal.** A refund in v1 does **not** auto-reverse a paid booking out of M2.2's `owed_bookings` / `payouts`. Flagged as v2 — see [[stripe-go-live]].
- **The simultaneous-click slot race** (two customers both creating a live `pending_payment` booking on one slot at once) — deferred until ~1,000 concurrent customers/​barber; "first to pay wins" + a cancellable stale `pending_payment` booking (which frees the slots) is sufficient. See [[m1.2-buyer-setup]].
- **3DS / SCA edge flows, alternative payment methods, saved cards** — single-card pay-now only.

When a student asks "shouldn't we use Connect / auto-refund the settlement?" → "Yes, for production. The course optimizes for the minimum money-correct flow end-to-end; Connect + refund reconciliation is a separate v2 pass."

---

## Cross-references

- [[m2.1-buyer-to-admin-payments]] — the build that applies Rules 0–10 (checkout route + webhook + the `commission_rates` table).
- [[m2.1-buyer-to-admin-payments-prerequisites]] — Stripe sandbox auth (`livemode:false`) + the admin promotion.
- [[m2.2-admin-to-seller-payment]] — the payout-build flow that computes the 20/80 split from the `paid` bookings this webhook records (Rule 9).
- [[stripe-go-live]] — sandbox→live keys, the new live webhook endpoint on the custom domain, and the refund-doesn't-reverse-the-settlement flag.
- [[stripe-mysite]] — an existing **TWD + Next.js + Supabase** Checkout implementation (zero-decimal handling, raw-body signature verify, `stripe listen` local testing) you can lift directly.
- [[supabase-best-practice]] — why the idempotency constraint (Rule 3) and the `paid` write go through a migration, and where the service-role key the webhook uses lives.
