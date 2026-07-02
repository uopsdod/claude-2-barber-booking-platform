---
name: m2.1-buyer-to-admin-payments
description: 抽成制理髮師預約平台 Milestone 2.1 — wire Stripe Checkout so booking = pay-now. Customer confirms a start slot in the M1.2 pop-up dialog → `POST /api/bookings/checkout` creates a Stripe Checkout Session with a DYNAMIC `price_data` line item (`unit_amount` honors `platform_settings.currency_minor_units` — TWD's default config is ×1, USD would be ×100), metadata + `client_reference_id` carry `{booking_id,customer_id}` (the booking_id is the join key; the barber/slots are derivable from the booking — there is NO stored start_slot_id), redirect to checkout.stripe.com. The `POST /api/stripe/webhook` route (raw-body verify, idempotent via a status guard) flips the BOOKING `pending_payment→paid` on `checkout.session.completed` + `payment_status==='paid'` (first to pay wins; slots have no status to flip) and STAMPS `paid_at` — that's it; NO split is computed and there is NO transactions row (no transactions table). The split is computed later at payout-build time (M2.2) from the picked paid bookings × commission_rates, NOT here. Step 2 creates ONLY the `commission_rates` table (versioned 20% ratio). Use when the student says "啟動 M2.1", "start M2.1", "接 Stripe 金流", "讓預約可以付款", "booking payment", or any variant of "預約時要先付款". Run `m2.1-buyer-to-admin-payments-prerequisites` first (Stripe sandbox auth + promote your admin user).
---

# M2.1 — Stripe 預約金流（預約即付款，付款成功才鎖位）

## What this skill does

Turns the M1.2 booking flow from "create a pending booking" into **pay-now via Stripe Checkout**. The customer confirms a start slot in the pop-up dialog → your server creates a **dynamic** Checkout Session for that exact service price → Stripe collects the money into **your platform's own Stripe account** (no Stripe Connect) → a **webhook** is the single source of truth that flips the **booking** `pending_payment → paid` and stamps `paid_at`. That's the whole job: **no split is computed and no ledger row is written** — "money in" is simply a `paid` booking's `price`. The platform/shop split is computed later, when the admin **builds a payout** (M2.2), from the picked `paid` bookings × `commission_rates`. (Slots have no status; the booking already holds its N slots via `booking_slots`, and the browse anti-join stops offering them the moment the `pending_payment` booking exists.)

By the end the student has:

1. A `POST /api/bookings/checkout` route that takes a `pending_payment` booking and creates a **dynamic `price_data` Checkout Session** — `unit_amount = bookings.price * 10^currency_minor_units` (the `price` snapshot from M1.2, scaled per `platform_settings.currency_minor_units`; with the default TWD config that's ×1), `currency` read from `platform_settings.currency`, `metadata: { booking_id, customer_id }` + `client_reference_id: booking_id` (the `booking_id` is the only join key the webhook needs; the barber/slots are derivable from the booking via `service_id → services.barber_id` and `booking_slots` — there is **no stored `start_slot_id`**), then redirects the browser to `checkout.stripe.com`.
2. A `POST /api/stripe/webhook` route that **reads the raw body**, verifies the Stripe signature, is **idempotent via the `pending_payment` status guard**, and on `checkout.session.completed` + `payment_status==='paid'` flips the **booking** `pending_payment → paid` (**first customer to pay wins**; no slot status to touch) and stamps `paid_at`. **No split, no transactions row** — M2.2 computes the split at payout-build time from the picked paid bookings.
2a. New **`commission_rates`** table (Step 2) — the versioned 20% ratio, seeded `2026-01-01 = 0.20`. (No transactions table; no fee columns on bookings; no `payout_records` table.)
3. **Middleware exempts `/api/stripe/webhook`** (otherwise auth middleware 307-redirects Stripe's events to `/login` and your handler never runs).
4. The M1.2 dialog's **confirm** now launches Checkout instead of stopping at a pending booking; a thin `/bookings/success` page **polls** booking status (UX only — the webhook, not this page, is the source of truth).
5. `STRIPE_SECRET_KEY` + `STRIPE_WEBHOOK_SECRET` in **Vercel env** (Production), with the Stripe dashboard webhook endpoint pointed at the Vercel URL.
6. An **admin account** (`role='admin'`) already promoted — done in the prerequisite, because admin only matters once money exists. M2.2's `/admin/payouts` page consumes it.

**Out of scope for M2.1:** the admin payout builder and the settlement UI (that's M2.2, which picks the `paid` bookings this milestone produces into flexible per-shop payout batches and applies the `commission_rates` split); going live with real cards (`[[stripe-go-live]]`); refund→payout reversal (flagged v2).

## When to load this skill

Trigger phrases:
- "啟動 M2.1" / "start M2.1" / "begin M2.1"
- "接 Stripe 金流" / "讓預約可以付款" / "預約時要先付款"
- "booking payment" / "wire Stripe Checkout for bookings"
- Any prompt mapping to "預約即付款，付款成功才確認 / 鎖位".

**Run `m2.1-buyer-to-admin-payments-prerequisites` first** — it confirms Stripe sandbox auth (`livemode:false`) and promotes your admin user. Don't start this build until that prereq is green.

## Execution mode (Cowork-first)

| Part | CLI mode tool | Cowork mode equivalent |
|---|---|---|
| Edit + push the two API routes + dialog | `git` + your editor | git tool, recall GitHub PAT from Secrets Manager (`barber-project/github`) |
| Stripe sandbox keys / confirm `livemode:false` | Stripe CLI (`stripe ...`) | Stripe MCP (`mcp__claude_ai_Stripe__*`) |
| Apply the `commission_rates` migration | `supabase` CLI | Supabase MCP `mcp__claude_ai_Supabase__apply_migration` |
| Set `STRIPE_*` env vars + create the webhook endpoint | Stripe/Vercel dashboards | **dashboards** — Stripe MCP does NOT manage webhook endpoints; Vercel MCP does NOT manage env vars (both are dashboard steps) |
| Local webhook testing | `stripe listen --forward-to localhost:3000/api/stripe/webhook` | — (Cowork students test against the deployed Vercel URL) |

The two genuinely-manual steps — **create the webhook endpoint in the Stripe dashboard** and **add the two env vars in the Vercel dashboard** — have no MCP in 2026. Everything else the connectors do.

## Architecture

![Barber platform architecture (M2.1) — the customer confirms a start slot in the booking dialog on /barbers/[id]; the browser calls POST /api/bookings/checkout, which reads the pending_payment bookings row (price snapshot) from Supabase and creates a dynamic Stripe Checkout Session (price_data with unit_amount scaled by platform_settings.currency_minor_units, metadata {booking_id,customer_id} + client_reference_id; the barber/slots are derivable from the booking — no stored start_slot_id), then redirects to checkout.stripe.com. Stripe collects 100% into the platform's own account. On payment, Stripe POSTs checkout.session.completed to POST /api/stripe/webhook (middleware EXEMPTS this path); the webhook verifies the raw-body signature, is idempotent via the pending_payment status guard, flips the bookings row pending_payment→paid (slots have no status), and stamps paid_at. No transactions row is written and no split is stored on the booking — M2.2 computes platform 20% / shop 80% at payout-build time from the picked paid bookings × commission_rates. The browser lands on /bookings/success, which only POLLS the bookings row. Env vars STRIPE_SECRET_KEY + STRIPE_WEBHOOK_SECRET live in Vercel; the dashboard webhook endpoint points at the Vercel URL.](assets/architecture-m2.1.png)

How the diagram maps to M2.1:
- **The bottom-left "charge booking" inset = the customer's pay-at-booking loop:** Product Site (`/api/bookings/checkout`) → Stripe Checkout, the `webhook` comes back, and the Product Site then **writes** (`W`) the `booking` row. It's the same round-trip drawn at the top (`payment check` → Stripe `Webhook` → `booking`), just zoomed in — the top view emphasizes that confirmation is **delayed**: you (admin) only know the payment truly landed once Stripe's webhook event arrives, not at redirect time.
- **Dialog confirm → `POST /api/bookings/checkout` → checkout.stripe.com:** the M1.2 dialog's confirm calls the checkout route, which builds a dynamic Session from the booking's `price` snapshot and redirects (Step 4–5).
- **Stripe → `POST /api/stripe/webhook` → `bookings.paid` + `paid_at`:** the only path that marks the booking paid (Step 6). It writes NO transactions row and computes NO split — that's computed at payout-build time in M2.2. **Middleware exempts this path** (Step 7).
- **Browser → `/bookings/success` (poll only):** a UX page that polls the booking row; never mutates (Step 8).
- **Vercel env / Stripe dashboard endpoint:** `STRIPE_SECRET_KEY` + `STRIPE_WEBHOOK_SECRET` in Vercel; the endpoint points at the Vercel URL (custom domain comes in M3 — see [[stripe-go-live]]).

## Conversational flow

You (Claude Code) **implement every step you can yourself, in order — do NOT wait for the student's approval between the steps you can do.** Run straight through the tool-doable work: build each step, then **verify it yourself before moving on**, leveraging every tool you have — the Supabase MCP (`apply_migration` / `execute_sql`), the Stripe MCP or CLI, the GitHub push, `call_aws` / the AWS CLI, and direct reads / `curl` against the deployed app.

**A few steps are unavoidable manual UI actions** — as of 2026 neither the Vercel connector manages env vars nor the Stripe MCP manages webhook endpoints, and only a human can type a card into Stripe's hosted Checkout. For those, **don't pretend to do them — GUIDE the student through the UI, hand them the exact values to paste, then wait for them to confirm and verify the result yourself** (e.g. `curl` the webhook path, re-query the booking). The manual steps are:
> - **Step 8.2 / 8.3** — create the webhook endpoint in the Stripe dashboard and paste its `STRIPE_WEBHOOK_SECRET` (`whsec_…`) into Vercel env + redeploy (the secret doesn't exist until the endpoint does).
> - **Step 9** — enter the test card `4242 4242 4242 4242` in Stripe's hosted Checkout page.
>
> (`STRIPE_SECRET_KEY` is no longer a build step — it's set in the prerequisite alongside the Stripe sandbox connection; Step 3 only *confirms* it's there.)

Everything else — the `commission_rates` migration, both API routes, the dialog rewire, the middleware exemption, the success page, and all verification — you do and check yourself. Report what you did and what you verified as you go.

1. Confirm the prereq is green (Stripe sandbox auth + admin promoted)
2. Create the `commission_rates` table (migration)
3. Confirm `STRIPE_SECRET_KEY` is already in Vercel env (set in the prereq)
4. Build `POST /api/bookings/checkout` (dynamic, `unit_amount` honors `currency_minor_units`)
5. Rewire the M1.2 dialog confirm → launch Checkout
6. Build `POST /api/stripe/webhook` (raw-body verify, idempotent, flip pending_payment → paid + stamp paid_at)
7. Exempt `/api/stripe/webhook` in middleware
8. Add the `/bookings/success` poll page + create the dashboard webhook endpoint + `STRIPE_WEBHOOK_SECRET`
9. End-to-end test with `4242 4242 4242 4242` → run the checklist

---

### Step 1 — Confirm the prerequisite is green

Before writing any code, confirm `m2.1-buyer-to-admin-payments-prerequisites` ran:

> 「先確認兩件事都好了：(1) Stripe sandbox 已連上、`livemode:false`；(2) 你的 admin 帳號已經用一次性 migration 升級成 `role='admin'`。如果還沒，先跟我說『啟動 M2.1 的前置作業』，我帶你做完再回來接金流。」

If either is missing, stop and run the prereq. The admin account isn't used *in* M2.1, but promoting it now (while we're in the payment milestone) is the locked-in course design — M2.2's payout page needs it.

---

### Step 2 — Create the `commission_rates` table (migration)

There is **NO `transactions` table** — "money in" is simply a `paid` booking's `price`. M2.1 adds exactly **one** table: **`commission_rates`**, the versioned 20% ratio. The split is **NOT** stored per booking and **NOT** computed by the webhook — it's derived at payout-build time (M2.2) by summing the admin's picked `paid` bookings × the rate in `commission_rates` (a versioned constant), and snapshotted onto the `payouts` batch row. `bookings` already has its `paid_at` column from M1.2's schema. (`platform_settings` is created back in M1.1 — M2.1 only *reads* it for the currency math, it does not create it here.) Apply as a **migration** (never a raw console edit — [[supabase-best-practice]]) via `mcp__claude_ai_Supabase__apply_migration`:

```sql
-- M2.1: the versioned commission ratio. "The rate for a booking" = the row with the greatest
-- effective_from <= the booking's paid_at. Change the rate by INSERTing a new row with a
-- later effective_from — already-built payouts keep the rate snapshotted onto their batch row.
-- NO transactions table, NO fee columns on bookings — the split is computed at payout-build time (M2.2).
create table if not exists public.commission_rates (
  id            uuid primary key default gen_random_uuid(),
  effective_from date not null unique,
  platform_pct  numeric(5,4) not null check (platform_pct >= 0 and platform_pct < 1),  -- e.g. 0.2000 = 20%
  note          text
);
insert into public.commission_rates (effective_from, platform_pct, note)
  values ('2026-01-01', 0.20, 'default 20% platform / 80% shop')
  on conflict (effective_from) do nothing;

alter table public.commission_rates enable row level security;
-- commission_rates: world-readable (it's just the public ratio); only admin writes it.
create policy "commission_rates_select_public" on public.commission_rates for select using (true);
create policy "commission_rates_write_admin"  on public.commission_rates for all using (public.is_admin());
```

> **Note for Claude Code:** there is **deliberately no per-booking ledger** — `bookings` carries no `platform_fee`/`barber_amount`, and there is **no `transactions` table** to insert into. The webhook (Step 6) only flips `bookings.status` to `paid` and stamps `paid_at`; "money in" = the `paid` booking's `price` snapshot (M1.2). M2.2 computes `platform_cut` / `shop_cut` at payout-build time by summing the admin's picked `paid` bookings × `commission_rates`, and snapshots the rate onto the `payouts` batch so a later rate change never alters a settled batch. After applying, run `get_advisors`.

---

### Step 3 — Confirm `STRIPE_SECRET_KEY` is already in Vercel env

`STRIPE_SECRET_KEY` (`sk_test_…`) is set **in the prerequisite**, right after the Stripe sandbox is connected ([[m2.1-buyer-to-admin-payments-prerequisites]] Part A) — it's a pure "copy the sandbox secret key into Vercel env" action with no dependency on M2.1 code, so it lives with the rest of the Stripe setup. **Here you only confirm it's there:**

> 到 **Vercel → Settings → Environment Variables**，確認 `STRIPE_SECRET_KEY`（`sk_test_…`，Production scope）已經在前置作業裡設好了。如果不在，回 `m2.1-buyer-to-admin-payments-prerequisites` 補上再回來。

`STRIPE_WEBHOOK_SECRET` comes in Step 8 (you can't know it until the endpoint exists). These are **app-runtime keys → Vercel env, NOT AWS Secrets Manager** ([[aws-secrets-best-practice]]; AWS holds only operational/dev secrets like the GitHub PAT). The Stripe secret key is server-only — it never ships in the browser bundle.

> **Note for Claude Code:** Vercel MCP does **not** manage env vars in 2026, so you can't read the var directly — ask the student to confirm it's present (or spot it by the checkout route working once deployed). If it was only *just* added, remember a **redeploy** is required for it to take effect.

---

### Step 4 — Build `POST /api/bookings/checkout` (dynamic, `unit_amount` honors `currency_minor_units`)

The route takes a `pending_payment` booking the dialog created, reads its `price` snapshot server-side, and builds a **dynamic** Checkout Session. The Stripe `unit_amount` is `price * 10^currency_minor_units`, with `currency` + `currency_minor_units` read from `platform_settings` — **not** a hard-coded assumption. With the default TWD config (`currency_minor_units = 0`) that's `price × 1`; a USD config (`= 2`) would be `× 100`. Have Claude Code write it, then push (recall the GitHub PAT from Secrets Manager — don't re-ask):

```ts
// app/api/bookings/checkout/route.ts  (Next.js App Router)
import Stripe from 'stripe'
import { NextResponse } from 'next/server'
import { createServiceClient } from '@/lib/supabase/server' // service-role, server only

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!)

export async function POST(req: Request) {
  const { booking_id } = await req.json()
  const supabase = createServiceClient()

  // Load the pending_payment booking + its price SNAPSHOT (set at creation in M1.2) — never trust a price from the client.
  // bookings has NO barber_id AND NO start_slot_id — the barber is reached through the SERVICE:
  // bookings.service_id → services.barber_id → barbers. (The slots live in booking_slots; we don't need them here.)
  const { data: booking } = await supabase
    .from('bookings')
    .select('id, customer_id, status, price, services(name, barber_id, barbers(id, name))')
    .eq('id', booking_id)
    .single()

  if (!booking || booking.status !== 'pending_payment') {
    return NextResponse.json({ error: 'booking not payable' }, { status: 400 })
  }

  const barberId   = booking.services.barber_id               // derived via the service
  const barberName = booking.services.barbers.name

  // Read currency + minor-units from platform_settings (created in M1.1) — do NOT hard-code TWD.
  const { data: cfg } = await supabase
    .from('platform_settings')
    .select('currency, currency_minor_units')
    .single()
  // unit_amount = price scaled into Stripe's smallest unit. TWD (minor_units=0) → ×1; USD (=2) → ×100.
  const unitAmount = booking.price * 10 ** cfg!.currency_minor_units

  const origin = req.headers.get('origin')!
  const session = await stripe.checkout.sessions.create({
    mode: 'payment',
    line_items: [{
      price_data: {
        currency: cfg!.currency,            // from platform_settings (default 'twd')
        product_data: { name: `${booking.services.name} @ ${barberName}` },
        unit_amount: unitAmount,            // price * 10^currency_minor_units (TWD default config → ×1, NOT ×100)
      },
      quantity: 1,
    }],
    metadata: {
      booking_id: booking.id,             // the ONLY join key the webhook needs
      customer_id: booking.customer_id,
    },
    client_reference_id: booking.id,
    success_url: `${origin}/bookings/success?session_id={CHECKOUT_SESSION_ID}`,
    cancel_url: `${origin}/barbers/${barberId}`,
  })

  return NextResponse.json({ url: session.url })
}
```

> **Note for Claude Code:** the **`unit_amount` scaling** is the #1 foot-gun — drive it off `platform_settings.currency_minor_units`, never a hard-coded ×100. With the **default TWD config (`currency_minor_units = 0`)** a NT$500 cut is `unit_amount: 500`, **NOT** `50000` — TWD is zero-decimal. A USD config (`= 2`) would correctly scale ×100. If a student blindly copied a USD (cents) example, a TWD booking charges 100× too much. Cross-reference the user's existing `stripe-mysite` skill for the same TWD handling. Stash `booking_id` in **BOTH** `metadata` and `client_reference_id` ([[stripe-best-practice]] Rule 6) — the webhook reads `metadata.booking_id`, which your authed server set and the customer cannot forge (Rule 10); the barber/slots are derivable from the booking, so they don't go in the metadata. Never look the booking up by email/customer.

---

### Step 5 — Rewire the M1.2 dialog confirm → launch Checkout

In M1.2 the pop-up dialog's **confirm** created a `pending_payment` booking and returned to `/barbers/[id]` with a toast. Now it should create the pending_payment booking **and then** call the checkout route and redirect to Stripe:

> 「把 `/barbers/[id]` 預約彈窗的『確認』改成：先呼叫 M1.2 的 `create_booking(service_id, start_slot_id)` RPC（一樣在交易裡建立 `pending_payment` booking ＋ N 筆 `booking_slots`、快照 `price`，**沒有 slot 狀態要改**，`booking_slots` 的 `UNIQUE(slot_id)` 保證一個時段只有一筆 live 預約），拿到 `booking_id` 後 `POST /api/bookings/checkout`，再 `window.location = url` 跳到 Stripe Checkout。付款頁是 Stripe 託管的，不是我們自己的頁面。」

At this point the booking is still `pending_payment` — it only becomes `paid` when the **webhook** sees the payment (Step 6). The slot has no status; it's held simply because a live (`pending_payment`) booking references it (the `UNIQUE(slot_id)` allows only one live booking per slot, and the browse anti-join already excludes it). A customer who abandons Checkout leaves a stale `pending_payment` booking; the simple model (first to *pay* wins; a stale `pending_payment` can be cancelled to free the slot) is sufficient — we do not engineer against the simultaneous-click race until ~1,000 concurrent customers/barber (locked-in deferral).

---

### Step 6 — Build `POST /api/stripe/webhook` (raw-body verify, idempotent, flip pending_payment → paid)

This is the **only** route that changes booking state. Its entire job is: flip `pending_payment → paid` and stamp `paid_at`. **No split, no transactions row.** Have Claude Code write it and push:

```ts
// app/api/stripe/webhook/route.ts
import Stripe from 'stripe'
import { NextResponse } from 'next/server'
import { createServiceClient } from '@/lib/supabase/server'

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!)
// NOTE: no PLATFORM_RATE here — the webhook does NOT compute the split. The rate lives
// in the commission_rates table and the split is computed at payout-build time (M2.2).

export async function POST(req: Request) {
  const body = await req.text()                     // RAW body — never req.json() first
  const sig = req.headers.get('stripe-signature')!

  let event: Stripe.Event
  try {
    event = stripe.webhooks.constructEvent(body, sig, process.env.STRIPE_WEBHOOK_SECRET!)
  } catch {
    return NextResponse.json({ error: 'signature verification failed' }, { status: 400 })
  }

  if (event.type !== 'checkout.session.completed') {
    return NextResponse.json({ received: true })    // ack unrelated events with 200
  }

  const session = event.data.object as Stripe.Checkout.Session
  if (session.payment_status !== 'paid') {
    return NextResponse.json({ received: true })     // only act on a real, paid session
  }

  const bookingId = session.metadata?.booking_id     // your server set this — can't be forged
  if (!bookingId) {
    return NextResponse.json({ error: 'missing booking_id metadata' }, { status: 400 }) // = your bug
  }

  const supabase = createServiceClient()

  // FIRST TO PAY WINS: flip the BOOKING pending_payment → paid exactly once, at this transition,
  // and stamp paid_at. That is the WHOLE job — NO split is computed and NO ledger row is written
  // (there is no transactions table); "money in" is simply this paid booking's price snapshot.
  // M2.2 computes the platform/shop split at payout-build time from the picked paid bookings × commission_rates.
  // There is NO slot status to mirror — slots have no status. The booking already holds its
  // N slots via booking_slots (the UNIQUE(slot_id) guarantees one live booking per slot), and
  // the browse anti-join stops offering them the moment the pending_payment booking exists.
  //
  // IDEMPOTENCY: Stripe RETRIES any non-2xx within ~10s, and you'll resend the event in the
  // checklist. The .eq('status','pending_payment') guard is the idempotency source of truth —
  // a re-delivered event matches ZERO rows (the booking is already 'paid') and no-ops.
  await supabase.from('bookings').update({
    status: 'paid',
    paid_at: new Date().toISOString(),
  }).eq('id', bookingId).eq('status', 'pending_payment') // guard: only the pending_payment row flips

  return NextResponse.json({ received: true })
}
```

> **Note for Claude Code:** four rules from [[stripe-best-practice]] are load-bearing here:
> - **Rule 2 raw-body verify:** `await req.text()` BEFORE `constructEvent`. `req.json()` re-serializes and breaks the HMAC → 400. (In the App Router the raw text is available directly; no `bodyParser:false` config needed as in the old Pages API.)
> - **Rule 3 idempotency via the status guard:** the `.eq('status','pending_payment')` guard makes the flip a no-op once the booking is already `paid`. Stripe retries non-2xx, and you'll resend the event in the checklist — a second delivery matches zero rows and must NOT re-stamp `paid_at`. (Minimal version uses the status guard; an optional hard backstop is a `bookings.stripe_payment_intent_id` unique column.)
> - **Rule 1 webhook is the source of truth:** only this route writes `paid`. The success page never mutates.
> - **Rule 9 state change exactly once at the right transition:** flip only on `checkout.session.completed` + `payment_status==='paid'`. The `.eq('status','pending_payment')` guard makes the flip safe under retries. The webhook does NOT compute any split — that's M2.2.

---

### Step 7 — Exempt `/api/stripe/webhook` in middleware

> **The silent-failure trap.** Your auth middleware redirects unauthenticated requests to `/login`. Stripe's webhook POST carries no user session, so middleware **307-redirects it to `/login`** and your handler never runs. Symptom: Stripe shows the event fired but delivery never returns 200; your logs show **ZERO hits**.

Add the webhook path to the public allowlist / matcher exclusion:

```ts
// middleware.ts
export const config = {
  // exclude /api/stripe/webhook (and static assets) from the auth matcher
  matcher: ['/((?!api/stripe/webhook|_next/static|_next/image|favicon.ico).*)'],
}
```

> **Note for Claude Code:** verify with `curl -i -X POST https://<app>.vercel.app/api/stripe/webhook` — you want a **400** (signature missing/invalid, i.e. the handler ran) or **200**, **NOT a 307** redirect to `/login`. A 307 means the exemption didn't take. ([[stripe-best-practice]] Rule 5.)

---

### Step 8 — `/bookings/success` poll page + dashboard webhook endpoint + `STRIPE_WEBHOOK_SECRET`

**8.1 — The success page (UX only):**
> 「做一個 `/bookings/success` 頁面：讀 `session_id`，每 1–2 秒去查這筆 booking 的 `status`，顯示『付款處理中…』直到變成 `paid`，再顯示『預約成功！』。這頁**只查不改**——webhook 才是真相來源。使用者付完款後可能直接關掉瀏覽器，所以絕對不能靠這頁來確認預約。」

**8.2 — Create the webhook endpoint in the Stripe dashboard** (no MCP — dashboard step):
> 到 Stripe dashboard（**sandbox / test mode**）→ Developers → **Webhooks → Add endpoint** → URL 填 `https://<your>.vercel.app/api/stripe/webhook` → 訂閱事件 **`checkout.session.completed`** → 建立。建立後 Stripe 會給你一個 **Signing secret**（`whsec_…`）。

**8.3 — Put the signing secret in Vercel env, then redeploy:**
> 到 **Vercel → Environment Variables**，新增 `STRIPE_WEBHOOK_SECRET = whsec_…`（Production），然後 **redeploy**。

> **Note for Claude Code:** the **dashboard-endpoint** `whsec_…` (stable) is **different** from the `stripe listen` CLI banner secret (which rotates each restart) — local dev uses the CLI one, Vercel prod uses this dashboard one ([[stripe-best-practice]] Rule 4). The endpoint points at the **Vercel URL for now**; when M3 attaches the custom domain, you create a *new* live endpoint on that domain ([[stripe-go-live]]). Stripe MCP does not manage webhook endpoints in 2026 — this stays a dashboard step.

---

### Step 9 — End-to-end test with a test card, then run the checklist

> 在 live Vercel 網站上：以 customer 身分挑一位理髮師 → 開預約彈窗 → 選日期/時段 → 確認 → 跳到 Stripe Checkout → 用測試卡 **`4242 4242 4242 4242`**、任意未來到期日、任意 CVC 付款 → 回到 `/bookings/success`，看到狀態變 `paid`。

Then verify the booking in Supabase and re-test idempotency:
- `select status, paid_at, price, payout_id from bookings where id = '<id>';` — `status='paid'` + `paid_at` set, `price` unchanged (e.g. `500`), `payout_id` still **NULL** (it's now an OWED booking — M2.2 stamps `payout_id` when the admin builds a payout). There is **no `transactions` table** to query and **no fee columns** on the booking — "money in" is just this `paid` booking's `price`; the split is M2.2's job.
- In the Stripe dashboard → that event → **Resend** → confirm the webhook returns 200 and `paid_at` does **not** change (the `pending_payment` status guard no-ops the re-delivery — idempotency).

> 「跑 `m2.1-buyer-to-admin-payments-checklist` 驗收。」

> **Note for Claude Code:** test cards work **only in test/sandbox mode** — never a real card in sandbox, never a test card in live ([[stripe-best-practice]] Rule 7). Live cards + go-live are [[stripe-go-live]].

---

## Things to watch out for (common mistakes)

1. **Hard-coding the `unit_amount` scale (the worst one).** Drive `unit_amount = price * 10^currency_minor_units` off `platform_settings`, never a hard-coded ×100. With the default **TWD** config (`currency_minor_units = 0`) NT$500 → `500`, not `50000` — TWD is zero-decimal. Copying a USD (cents) example charges 100× too much on TWD. ([[stripe-best-practice]] TWD rule; cross-ref `stripe-mysite`.)
2. **`req.json()` before signature verify.** Always `await req.text()` first — re-serialization breaks the HMAC → 400 "signature verification failed". (Rule 2.)
3. **Middleware not exempting `/api/stripe/webhook`.** Symptom: Stripe shows the event fired but your logs show zero hits (307 → `/login`). Add the matcher exclusion + verify you get 400/200, not 307. (Rule 5.)
4. **No idempotency.** Stripe retries non-2xx and you'll resend the event in the checklist — without the `.eq('status','pending_payment')` status guard a re-delivery re-stamps `paid_at`. (Rule 3.)
5. **Re-deriving the price from the live service.** The webhook touches only `status`/`paid_at`; "money in" is the `bookings.price` **snapshot** (M1.2), not `services.price` — a later price edit must not rewrite past bookings. (Rule 8 / dynamic adaptation.)
6. **Trusting the success page.** It polls; it must never write `paid`. The user can close the browser after paying or the redirect can drop — only the webhook is authoritative. (Rule 1.)
7. **Looking the booking up by email/customer.** Use `metadata.booking_id` (server-set, unforgeable), not a customer-email lookup. (Rule 10.)
8. **Wrong webhook secret.** The Vercel env uses the **dashboard endpoint's** `whsec_…`, not the rotating `stripe listen` one. (Rule 4.)
9. **Forgetting to redeploy after adding env vars.** New `STRIPE_*` vars need a redeploy to take effect (Steps 3 + 8).
10. **Stripe keys in AWS Secrets Manager.** App-runtime keys go to **Vercel env**; AWS holds only the GitHub PAT / dev secrets. ([[aws-secrets-best-practice]].)
11. **Inventing a `transactions` table or fee columns.** There is **no `transactions` table** and **no `platform_fee`/`barber_amount`** anywhere — the webhook does NOT insert a ledger row or compute a split. "Money in" = a `paid` booking's `price`; M2.2 sums the admin's picked paid bookings × `commission_rates` at payout-build time. Don't add a per-booking ledger.
12. **Refund ≠ payout reversal (v1 limitation).** Refunding a `paid` booking does NOT auto-reverse what M2.2 settles from (v1 leaves the booking `paid`) — flag as out-of-scope/v2 ([[stripe-go-live]]).
13. **Stamping a settlement status on the booking.** `bookings.status` is **3 states only** (`pending_payment | paid | cancelled`) — there is no `payout_pending`/`payout_transferred`. Settlement state is **derived from `bookings.payout_id`** (NULL = owed; set = in that payout batch). The webhook never touches `payout_id`.
14. **Storing/looking up a `start_slot_id`.** `bookings` has **no `start_slot_id`** — the slots live in `booking_slots` and the barber is reached via `service_id → services.barber_id`. The checkout metadata carries only `{booking_id, customer_id}`; don't reference a stored start slot.

## Expected duration

40–70 minutes — most of it is the two API routes + the dialog rewire and one round-trip through the Stripe dashboard (endpoint + secret) and Vercel (env + redeploy). The prereq (Stripe sandbox auth + admin promotion) is separate.

## Next step

When `m2.1-buyer-to-admin-payments-checklist` is green, tell the student:
「M2.1 完成了！現在客人預約就會跳到 Stripe Checkout 付款，付款成功後 webhook 會把預約從 `pending_payment` 變成 `paid`、並蓋上 `paid_at` 時間戳。它**只做這件事**——不算拆帳、也不寫任何 ledger（沒有 transactions 表）；「實收」就是這筆 `paid` 預約的 `price`，而且 `payout_id` 還是 NULL（代表這筆還「欠撥」）。抽成拆帳會在 M2.2 由 admin 挑選欠撥的 `paid` 預約、組成一筆「撥款批次」時用 `commission_rates` 算出來。你也已經有一個 `role='admin'` 的帳號了。準備好的話跟我說『啟動 M2.2』，我們來做 admin 撥款頁，把每間店該領的 80% 算出來、標記轉帳。」
Then load `m2.2-admin-to-seller-payment`.

## Reference

- Stripe Checkout Sessions: https://stripe.com/docs/api/checkout/sessions/create
- Dynamic `price_data`: https://stripe.com/docs/payments/accept-a-payment?integration=checkout
- Zero-decimal currencies (TWD): https://stripe.com/docs/currencies#zero-decimal
- Webhook signature verification: https://stripe.com/docs/webhooks/signature
- Test cards: https://stripe.com/docs/testing
- Cross-skill: [[stripe-best-practice]] · [[m2.1-buyer-to-admin-payments-prerequisites]] · [[m2.2-admin-to-seller-payment]] · [[supabase-best-practice]] · [[stripe-go-live]]
