---
name: m2.1-buyer-to-admin-payments
description: 抽成制理髮師預約平台 Milestone 2.1 — wire Stripe Checkout so booking = pay-now. Customer confirms a start slot in the M1.2 pop-up dialog → `POST /api/bookings/checkout` creates a Stripe Checkout Session with a DYNAMIC `price_data` line item (`unit_amount = price` scaled into Stripe's smallest unit — TWD is 2-decimal, so ×100 — NOT driven off currency_minor_units, which is display-only), metadata + `client_reference_id` carry `{booking_id,customer_id}` (the booking_id is the join key; the barber/slots are derivable from the booking — there is NO stored start_slot_id), redirect to checkout.stripe.com. The `POST /api/stripe/webhook` route (raw-body verify, idempotent via a status guard) flips the BOOKING `pending_payment→paid` on `checkout.session.completed` + `payment_status==='paid'` (first to pay wins; slots have no status to flip) and STAMPS `paid_at` — that's it; NO split is computed and there is NO transactions row (no transactions table). The split is computed later at payout-build time (M2.2) from the picked paid bookings × commission_rates, NOT here. Step 2 creates ONLY the `commission_rates` table (versioned 20% ratio). Works for both Next.js App Router and Lovable's Vite-SPA (Vercel serverless functions) scaffold. Use when the student says "啟動 M2.1", "start M2.1", "接 Stripe 金流", "讓預約可以付款", "booking payment", or any variant of "預約時要先付款". Run `m2.1-buyer-to-admin-payments-prerequisites` first (Stripe sandbox auth + SUPABASE_SECRET_KEY + promote your admin user).
---

# M2.1 — Stripe 預約金流（預約即付款，付款成功才鎖位）

## What this skill does

Turns the M1.2 booking flow from "create a pending booking" into **pay-now via Stripe Checkout**. The customer confirms a start slot in the pop-up dialog → your server creates a **dynamic** Checkout Session for that exact service price → Stripe collects the money into **your platform's own Stripe account** (no Stripe Connect) → a **webhook** is the single source of truth that flips the **booking** `pending_payment → paid` and stamps `paid_at`. That's the whole job: **no split is computed and no ledger row is written** — "money in" is simply a `paid` booking's `price`. The platform/shop split is computed later, when the admin **builds a payout** (M2.2), from the picked `paid` bookings × `commission_rates`. (Slots have no status; the booking already holds its N slots via `booking_slots`, and the browse anti-join stops offering them the moment the `pending_payment` booking exists.)

By the end the student has:

1. A `POST /api/bookings/checkout` route that takes a `pending_payment` booking and creates a **dynamic `price_data` Checkout Session** — `unit_amount = bookings.price × factor` (the `price` snapshot from M1.2, scaled into Stripe's smallest unit; **TWD is 2-decimal in Stripe, so factor = 100** — a NT$300 cut → `30000`; only true zero-decimal currencies like JPY use ×1), `currency` read from `platform_settings.currency`, `metadata: { booking_id, customer_id }` + `client_reference_id: booking_id` (the `booking_id` is the only join key the webhook needs; the barber/slots are derivable from the booking via `service_id → services.barber_id` and `booking_slots` — there is **no stored `start_slot_id`**), then redirects the browser to `checkout.stripe.com`.
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
| Set `STRIPE_*` env vars + create the webhook endpoint | Stripe/Vercel dashboards | **done in the prereq** (up front) — Stripe MCP does NOT manage webhook endpoints; Vercel MCP does NOT manage env vars. The build only *confirms* them + redeploys (Step 8) |
| Local webhook testing | `stripe listen --forward-to localhost:3000/api/stripe/webhook` | — (Cowork students test against the deployed Vercel URL) |

The genuinely-manual steps — **creating the webhook endpoint in the Stripe dashboard** and **adding the two env vars in the Vercel dashboard** — have no MCP in 2026, so they're done **up front in the prereq** ([[m2.1-buyer-to-admin-payments-prerequisites]]); the build just confirms them and redeploys. Everything else the connectors do.

## Architecture

![Barber platform architecture (M2.1) — the customer confirms a start slot in the booking dialog on /barbers/[id]; the browser calls POST /api/bookings/checkout, which reads the pending_payment bookings row (price snapshot) from Supabase and creates a dynamic Stripe Checkout Session (price_data with unit_amount = price scaled into Stripe's smallest unit — TWD is 2-decimal so ×100, NOT driven off currency_minor_units; metadata {booking_id,customer_id} + client_reference_id; the barber/slots are derivable from the booking — no stored start_slot_id), then redirects to checkout.stripe.com. Stripe collects 100% into the platform's own account. On payment, Stripe POSTs checkout.session.completed to POST /api/stripe/webhook (middleware EXEMPTS this path); the webhook verifies the raw-body signature, is idempotent via the pending_payment status guard, flips the bookings row pending_payment→paid (slots have no status), and stamps paid_at. No transactions row is written and no split is stored on the booking — M2.2 computes platform 20% / shop 80% at payout-build time from the picked paid bookings × commission_rates. The browser lands on /bookings/success, which only POLLS the bookings row. Env vars STRIPE_SECRET_KEY + STRIPE_WEBHOOK_SECRET + SUPABASE_SECRET_KEY (the service-role key both functions use to write past RLS) live in Vercel; the dashboard webhook endpoint points at the Vercel URL.](assets/architecture-m2.1.png)

How the diagram maps to M2.1:
- **The bottom-left "charge booking" inset = the customer's pay-at-booking loop:** Product Site (`/api/bookings/checkout`) → Stripe Checkout, the `webhook` comes back, and the Product Site then **writes** (`W`) the `booking` row. It's the same round-trip drawn at the top (`payment check` → Stripe `Webhook` → `booking`), just zoomed in — the top view emphasizes that confirmation is **delayed**: you (admin) only know the payment truly landed once Stripe's webhook event arrives, not at redirect time.
- **Dialog confirm → `POST /api/bookings/checkout` → checkout.stripe.com:** the M1.2 dialog's confirm calls the checkout route, which builds a dynamic Session from the booking's `price` snapshot and redirects (Step 4–5).
- **Stripe → `POST /api/stripe/webhook` → `bookings.paid` + `paid_at`:** the only path that marks the booking paid (Step 6). It writes NO transactions row and computes NO split — that's computed at payout-build time in M2.2. **Middleware exempts this path** (Step 7).
- **Browser → `/bookings/success` (poll only):** a UX page that polls the booking row; never mutates (Step 8).
- **Vercel env / Stripe dashboard endpoint:** `STRIPE_SECRET_KEY` + `STRIPE_WEBHOOK_SECRET` in Vercel; the endpoint points at the Vercel URL (custom domain comes in M3 — see [[stripe-go-live]]).

## Conversational flow

You (Claude Code) **implement every step you can yourself, in order — do NOT wait for the student's approval between the steps you can do.** Run straight through the tool-doable work: build each step, then **verify it yourself before moving on**, leveraging every tool you have — the Supabase MCP (`apply_migration` / `execute_sql`), the Stripe MCP or CLI, the GitHub push, `call_aws` / the AWS CLI, and direct reads / `curl` against the deployed app.

**A few steps are unavoidable manual UI actions** — as of 2026 neither the Vercel connector manages env vars nor the Stripe MCP manages webhook endpoints, and only a human can type a card into Stripe's hosted Checkout. For those, **don't pretend to do them — GUIDE the student through the UI, hand them the exact values to paste, then wait for them to confirm and verify the result yourself** (e.g. `curl` the webhook path, re-query the booking). The manual steps are:
> - **Step 8.2** — confirm the webhook endpoint + `STRIPE_WEBHOOK_SECRET` that the prereq already created, then redeploy so the newly-built route + env vars go live. (Creating them is the prereq's A5, not a build step.)
> - **Step 9** — enter the test card `4242 4242 4242 4242` in Stripe's hosted Checkout page.
>
> (`STRIPE_SECRET_KEY` is no longer a build step — it's set in the prerequisite alongside the Stripe sandbox connection; Step 3 only *confirms* it's there.)

Everything else — the `commission_rates` migration, both API routes, the dialog rewire, the middleware exemption, the success page, and all verification — you do and check yourself. Report what you did and what you verified as you go.

> **FIRST: detect the scaffold — Next.js App Router vs Lovable's Vite-SPA.** The code blocks below are written for **Next.js App Router** (`app/api/**/route.ts`, `middleware.ts`, `await req.text()`). But **Lovable scaffolds a client-only Vite + React SPA** — no server runtime, no `app/`, no `middleware.ts`. Check for `vite.config.*` / an `index.html` entry / `"type": "module"` in `package.json` before writing any route. If it's Vite, translate every route to the **Vite-SPA track** ([[stripe-best-practice]] Rule 2's Vite variant has the full pattern):
> - **Server routes = Vercel serverless functions** in a top-level `/api` dir: `export default function handler(req: VercelRequest, res: VercelResponse)`. Add the `stripe` + `@vercel/node` deps. (So `app/api/bookings/checkout/route.ts` → `api/bookings/checkout.ts`, `app/api/stripe/webhook/route.ts` → `api/stripe/webhook.ts`.)
> - **Webhook raw body:** a Vercel Node function auto-parses the body, so set `export const config = { api: { bodyParser: false } }` **and** buffer the raw stream yourself (helper in Rule 2) — App Router's `await req.text()` does **not** apply.
> - **Step 7's "middleware exemption" becomes the `vercel.json` SPA rewrite.** There's no middleware; the catch-all rewrite that serves `index.html` will otherwise swallow `/api/*`. Exclude it: `"rewrites": [{ "source": "/((?!api/).*)", "destination": "/index.html" }]`.
> - **ESM import gotcha (runtime-only — green `vite build`, 500 in prod):** `"type": "module"` + Vercel transpiling each `/api/*.ts` separately means a relative import needs the **`.js` extension** — `import { x } from '../_supabaseAdmin.js'` — or the function 500s with `ERR_MODULE_NOT_FOUND`.
> - **Opaque Supabase keys:** new-format `sb_secret_…` keys are not JWTs — the server service-role client needs the same `apikey`-header fetch shim the browser client uses, or requests are unauthorized.

1. Confirm the prereq is green (Stripe sandbox auth + `SUPABASE_SECRET_KEY` + admin promoted)
2. Create the `commission_rates` table (migration)
3. Confirm `STRIPE_SECRET_KEY` + `SUPABASE_SECRET_KEY` are already in Vercel env (set in the prereq)
4. Build `POST /api/bookings/checkout` (dynamic, `unit_amount` scaled into Stripe's smallest unit — TWD ×100)
5. Rewire the M1.2 dialog confirm → launch Checkout
6. Build `POST /api/stripe/webhook` (raw-body verify, idempotent, flip pending_payment → paid + stamp paid_at)
7. Exempt `/api/stripe/webhook` in middleware
8. Add the `/bookings/success` poll page + confirm the webhook endpoint & `STRIPE_WEBHOOK_SECRET` (created in the prereq) + redeploy
9. End-to-end test with `4242 4242 4242 4242` → run the checklist

---

### Step 1 — Confirm the prerequisite is green

Before writing any code, confirm `m2.1-buyer-to-admin-payments-prerequisites` ran:

> 「先確認兩件事都好了：(1) Stripe sandbox 已連上、`livemode:false`；(2) 你的 admin 帳號已經用一次性 migration 升級成 `role='admin'`。如果還沒，先跟我說『啟動 M2.1 的前置作業』，我帶你做完再回來接金流。」

If either is missing, stop and run the prereq. The admin account isn't used *in* M2.1, but promoting it now (while we're in the payment milestone) is the locked-in course design — M2.2's payout page needs it.

---

### Step 2 — Create the `commission_rates` table (migration)

There is **NO `transactions` table** — "money in" is simply a `paid` booking's `price`. M2.1 adds exactly **one** table (**`commission_rates`**, the versioned 20% ratio) plus **one column** on `bookings` (`stripe_payment_intent_id`, UNIQUE — the reconciliation pointer + idempotency backstop the webhook stamps). The split is **NOT** stored per booking and **NOT** computed by the webhook — it's derived at payout-build time (M2.2) by summing the admin's picked `paid` bookings × the rate in `commission_rates` (a versioned constant), and snapshotted onto the `payouts` batch row. `bookings` already has its `paid_at` column from M1.2's schema. (`platform_settings` is created back in M1.1 — M2.1 only *reads* it for the currency math, it does not create it here.) Apply as a **migration** (never a raw console edit — [[supabase-best-practice]]) via `mcp__claude_ai_Supabase__apply_migration`:

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

-- bookings.stripe_payment_intent_id: the webhook stamps session.payment_intent on the paid flip.
-- It is (a) a hard idempotency backstop — the UNIQUE index rejects a concurrent double-fire that
-- slips past the status guard — and (b) a refund/reconciliation pointer back to the Stripe payment
-- (read it back via the Stripe MCP fetch_stripe_resources(pi_…) — no read-proxy Lambda needed).
alter table public.bookings add column if not exists stripe_payment_intent_id text;
create unique index if not exists uniq_bookings_pi on public.bookings(stripe_payment_intent_id);
```

> **Note for Claude Code:** there is **deliberately no per-booking ledger** — `bookings` carries no `platform_fee`/`barber_amount`, and there is **no `transactions` table** to insert into. The webhook (Step 6) only flips `bookings.status` to `paid`, stamps `paid_at`, and records `stripe_payment_intent_id`; "money in" = the `paid` booking's `price` snapshot (M1.2). The `stripe_payment_intent_id` is not money data — it's a pointer to the Stripe payment for **reconciliation** (verify the charged amount via the Stripe MCP's PaymentIntent/Charge reads keyed off it) and a **hard idempotency backstop** (the UNIQUE index). M2.2 computes `platform_cut` / `shop_cut` at payout-build time by summing the admin's picked `paid` bookings × `commission_rates`, and snapshots the rate onto the `payouts` batch so a later rate change never alters a settled batch. After applying, run `get_advisors`.

---

### Step 3 — Confirm `STRIPE_SECRET_KEY` and `SUPABASE_SECRET_KEY` are already in Vercel env

`STRIPE_SECRET_KEY` (`sk_test_…`) and `SUPABASE_SECRET_KEY` (the Supabase **service-role / secret key**, `sb_secret_…`) are both set **in the prerequisite** ([[m2.1-buyer-to-admin-payments-prerequisites]] Part A: A4 + A6) — they're pure "copy the key into Vercel env" actions with no dependency on M2.1 code, so they live with the rest of the pre-code setup. **Here you only confirm they're there:**

> 到 **Vercel → Settings → Environment Variables**，確認 `STRIPE_SECRET_KEY`（`sk_test_…`）和 `SUPABASE_SECRET_KEY`（`sb_secret_…`，Supabase service-role/secret key）都在（Production scope），是前置作業裡設好的。少任何一個就回 `m2.1-buyer-to-admin-payments-prerequisites` 補上再回來。

`SUPABASE_SECRET_KEY` is **required by both serverless functions** (checkout + webhook): Stripe is not a logged-in user, so the functions must use the **service-role key to write past RLS** — without it the webhook can't flip a booking to `paid` and checkout can't read the pending booking. **Never `VITE_`-prefix it** — Vite would inline it into the browser bundle (a full-database key leak); it is server-only. `STRIPE_WEBHOOK_SECRET` is **also set in the prereq** (A5, created with the webhook endpoint) — Step 8 just confirms it and redeploys. All three are **app-runtime keys → Vercel env, NOT AWS Secrets Manager** ([[aws-secrets-best-practice]]; AWS holds only operational/dev secrets like the GitHub PAT). None ships in the browser bundle.

> **Note for Claude Code:** Vercel MCP does **not** manage env vars in 2026, so you can't read these directly — ask the student to confirm they're present (or spot them by the checkout route working once deployed). If any was only *just* added, remember a **redeploy** is required for it to take effect.

---

### Step 4 — Build `POST /api/bookings/checkout` (dynamic, `unit_amount` scaled into Stripe's smallest unit — TWD is 2-decimal → ×100)

The route takes a `pending_payment` booking the dialog created, reads its `price` snapshot server-side, and builds a **dynamic** Checkout Session. The Stripe `unit_amount` is the whole-unit `price` scaled into **Stripe's smallest unit for the currency**: **TWD is a 2-decimal currency in Stripe** (NOT on Stripe's zero-decimal list), so `unit_amount = price × 100` — a NT$300 cut → `30000` (= NT$300.00). Only Stripe's *true* zero-decimal currencies (`jpy`, `krw`, …) use `× 1`. **Do NOT drive the factor off `platform_settings.currency_minor_units`** — that column is a *display* concept (`0` = show whole TWD), a different thing from Stripe's per-currency exponent; scale off the zero-decimal set below. Have Claude Code write it, then push (recall the GitHub PAT from Secrets Manager — don't re-ask):

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

  // Read the currency from platform_settings (created in M1.1) — do NOT hard-code TWD.
  const { data: cfg } = await supabase
    .from('platform_settings')
    .select('currency')
    .single()
  // unit_amount = price scaled into STRIPE's smallest unit for the currency.
  // TWD is 2-decimal in Stripe → ×100 (NT$300 → 30000). Only true zero-decimal currencies use ×1.
  // Do NOT use currency_minor_units here — that's a DISPLAY concept, not Stripe's exponent.
  const ZERO_DECIMAL = new Set(['bif','clp','djf','gnf','jpy','kmf','krw','mga','pyg','rwf','vnd','vuv','xaf','xof','xpf'])
  const factor = ZERO_DECIMAL.has(cfg!.currency.toLowerCase()) ? 1 : 100
  const unitAmount = booking.price * factor

  const origin = req.headers.get('origin')!
  const session = await stripe.checkout.sessions.create({
    mode: 'payment',
    line_items: [{
      price_data: {
        currency: cfg!.currency,            // from platform_settings (default 'twd')
        product_data: { name: `${booking.services.name} @ ${barberName}` },
        unit_amount: unitAmount,            // TWD 300 → 30000 (NT$300.00); NOT 300 (that's NT$3.00 → rejected)
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

> **Note for Claude Code:** the **`unit_amount` scaling** is the #1 foot-gun — scale by Stripe's smallest-unit factor for the currency, and **do NOT drive it off `platform_settings.currency_minor_units`** (that's display-only, a *different* thing from Stripe's exponent). **TWD is 2-decimal in Stripe**, so `factor = 100`: a NT$300 cut is `unit_amount: 30000` (= NT$300.00). The wrong version here is the *reverse* of the usual reflex — someone "knows TWD looks like whole dollars" and sends `× 1` → `300` = **NT$3.00 ≈ US$0.10**, which is **below Stripe's ~US$0.50 minimum, so the Checkout Session is rejected and the customer never reaches the payment page** (empirically confirmed: `price 300 → 30000 → succeeded`; `× 1 → 300 → rejected`). Verify by reading the charged `amount` on the PaymentIntent/Charge, not by eyeballing. **Do NOT copy `stripe-mysite`'s TWD handling if it treats TWD as zero-decimal** — that's the same inverted bug. Stash `booking_id` in **BOTH** `metadata` and `client_reference_id` ([[stripe-best-practice]] Rule 6) — the webhook reads `metadata.booking_id`, which your authed server set and the customer cannot forge (Rule 10); the barber/slots are derivable from the booking, so they don't go in the metadata. Never look the booking up by email/customer.

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
    stripe_payment_intent_id: session.payment_intent as string, // reconciliation pointer + idempotency backstop
  }).eq('id', bookingId).eq('status', 'pending_payment') // guard: only the pending_payment row flips

  return NextResponse.json({ received: true })
}
```

> **Note for Claude Code:** four rules from [[stripe-best-practice]] are load-bearing here:
> - **Rule 2 raw-body verify:** `await req.text()` BEFORE `constructEvent`. `req.json()` re-serializes and breaks the HMAC → 400. (In the App Router the raw text is available directly; no `bodyParser:false` config needed as in the old Pages API.) **Vite-SPA track:** this is a Vercel serverless function instead — set `export const config = { api: { bodyParser: false } }` and buffer the raw stream yourself (`for await (const chunk of req) …`); App Router's `req.text()` does not apply. See the scaffold callout above + [[stripe-best-practice]] Rule 2's Vite variant.
> - **Rule 3 idempotency via the status guard:** the `.eq('status','pending_payment')` guard makes the flip a no-op once the booking is already `paid`. Stripe retries non-2xx, and a re-delivery matches zero rows and must NOT re-stamp `paid_at`. The `bookings.stripe_payment_intent_id` UNIQUE index (added in Step 2) is a **hard backstop** on top of the status guard — a concurrent double-fire that raced past the status check fails the UNIQUE write.
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

> **Vite-SPA track:** there is **no `middleware.ts`** — the equivalent trap is the `vercel.json` SPA catch-all rewrite swallowing `/api/*`. Instead of the matcher above, exclude `/api/` from the rewrite: `{ "rewrites": [{ "source": "/((?!api/).*)", "destination": "/index.html" }] }`. Same silent failure (the webhook POST returns the HTML shell / never runs), different mechanism. See the scaffold callout + [[stripe-best-practice]] Rule 5's Vite variant.

> **Note for Claude Code:** verify with `curl -i -X POST https://<app>.vercel.app/api/stripe/webhook` — you want a **400** (signature missing/invalid, i.e. the handler ran) or **200**, **NOT a 307** redirect to `/login` (Next.js) and **not the HTML `index.html` shell** (Vite — means the rewrite swallowed it). ([[stripe-best-practice]] Rule 5.)

---

### Step 8 — `/bookings/success` poll page + confirm the webhook endpoint/secret (set in the prereq) + redeploy

**8.1 — The success page (UX only):**
> 「做一個 `/bookings/success` 頁面：讀 `session_id`，每 1–2 秒去查這筆 booking 的 `status`，顯示『付款處理中…』直到變成 `paid`，再顯示『預約成功！』。這頁**只查不改**——webhook 才是真相來源。使用者付完款後可能直接關掉瀏覽器，所以絕對不能靠這頁來確認預約。」

**8.2 — Confirm the webhook endpoint + `STRIPE_WEBHOOK_SECRET` (created in the prereq), then redeploy:**

The webhook endpoint (`https://<your>.vercel.app/api/stripe/webhook`, event `checkout.session.completed`) and `STRIPE_WEBHOOK_SECRET` were **already created in the prerequisite** ([[m2.1-buyer-to-admin-payments-prerequisites]] A5) — before this route existed. Now that Steps 6–7 have shipped the route, just confirm both are in place and **redeploy** so the new route + env vars go live:

> 確認前置作業已經：(1) 在 Stripe dashboard（sandbox）建好 webhook 端點指向 `https://<your>.vercel.app/api/stripe/webhook`、只訂閱 `checkout.session.completed`；(2) 把 `STRIPE_WEBHOOK_SECRET = whsec_…` 放進 Vercel env（Production）。**都在的話，觸發一次 Vercel redeploy**，讓剛做好的 `/api/stripe/webhook` 路由和環境變數上線。若少了任何一個，回 `m2.1-buyer-to-admin-payments-prerequisites` A5 補上。

> **Note for Claude Code:** don't re-create the endpoint or re-add the secret here — that's the prereq's job (moved up front so the build is pure code). The **dashboard-endpoint** `whsec_…` (stable) is **different** from the `stripe listen` CLI banner secret (which rotates each restart) — local dev uses the CLI one, Vercel prod uses the dashboard one ([[stripe-best-practice]] Rule 4). The endpoint points at the **Vercel URL for now**; when M3 attaches the custom domain, you create a *new* live endpoint on that domain ([[stripe-go-live]]). Until this redeploy, the endpoint's deliveries were failing (404 — no route yet), which was expected; the actual delivery test is Step 9.

---

### Step 9 — End-to-end test with a test card, then run the checklist

> 在 live Vercel 網站上：以 customer 身分挑一位理髮師 → 開預約彈窗 → 選日期/時段 → 確認 → 跳到 Stripe Checkout → 用測試卡 **`4242 4242 4242 4242`**、任意未來到期日、任意 CVC 付款 → 回到 `/bookings/success`，看到狀態變 `paid`。

Then verify the booking in Supabase and confirm the charged amount:
- `select status, paid_at, price, payout_id, stripe_payment_intent_id from bookings where id = '<id>';` — `status='paid'` + `paid_at` set, `price` unchanged (e.g. `300`), `stripe_payment_intent_id` populated (`pi_…`), `payout_id` still **NULL** (it's now an OWED booking — M2.2 stamps `payout_id` when the admin builds a payout). There is **no `transactions` table** to query and **no fee columns** on the booking — "money in" is just this `paid` booking's `price`; the split is M2.2's job.
- **Verify the charged amount via the PaymentIntent/Charge, not Checkout Sessions.** Read the payment back through the Stripe MCP (`fetch_stripe_resources(pi_…)` from the stored `stripe_payment_intent_id`, or a PaymentIntent/Charge read) and confirm `amount = price × 100` for TWD (e.g. `30000` = NT$300.00), `status: succeeded`. (On the restricted Cowork key the Checkout Sessions resource is read-denied, but PaymentIntents/Charges reads work — key off the PI, don't list sessions.)
- **Idempotency** is enforced by the `.eq('status','pending_payment')` status guard (backed by the `stripe_payment_intent_id` UNIQUE index) — a re-delivered event matches zero rows and no-ops. (No manual "resend the event" step is needed; the guard is the source of truth.)

> 「跑 `m2.1-buyer-to-admin-payments-checklist` 驗收。」

> **Note for Claude Code:** test cards work **only in test/sandbox mode** — never a real card in sandbox, never a test card in live ([[stripe-best-practice]] Rule 7). Live cards + go-live are [[stripe-go-live]].

---

## Things to watch out for (common mistakes)

1. **Getting the `unit_amount` scale wrong (the worst one).** **TWD is 2-decimal in Stripe** → `unit_amount = price × 100` (NT$300 → `30000`). Scale off the zero-decimal set, **NOT** off `platform_settings.currency_minor_units` (that's display-only). The inverted bug (`× 1`) bills NT$3.00 — below Stripe's ~50¢ minimum → the Session is **rejected** and the customer never pays. Copying a "TWD is zero-decimal" example (incl. `stripe-mysite` if it does that) re-introduces this. ([[stripe-best-practice]] Rule 0 — TWD is 2-decimal.)
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

When `m2.1-buyer-to-admin-payments-checklist` is green, tell the student (milestone-scoped — celebrate M2.1, don't surface the next milestone):
「M2.1 完成了！現在客人預約就會跳到 Stripe Checkout 付款，付款成功後 webhook 會把預約從 `pending_payment` 變成 `paid`、並蓋上 `paid_at` 時間戳。它**只做這件事**——不算拆帳、也不寫任何 ledger（沒有 transactions 表）；「實收」就是這筆 `paid` 預約的 `price`，而且 `payout_id` 還是 NULL（代表這筆還「欠撥」）。你也已經有一個 `role='admin'` 的帳號了。預約線上付款這一步到這裡就完整了 ✅」

> **Note for Claude Code (author-only — do NOT volunteer to the student):** the next milestone is the admin commission-settlement page — the admin picks owed (`paid`, `payout_id` NULL) bookings per shop, builds a payout batch, and the 20/80 split is computed from the picked bookings × `commission_rates` at that point. When the instructor advances the student (or the student asks "what's next"), load `m2.2-admin-to-seller-payment`. Don't seed the settlement/payout mechanics into the student's mental model at the end of M2.1.

## Reference

- Stripe Checkout Sessions: https://stripe.com/docs/api/checkout/sessions/create
- Dynamic `price_data`: https://stripe.com/docs/payments/accept-a-payment?integration=checkout
- Currencies & the smallest-unit exponent (TWD is **2-decimal** → `price × 100`; the list here is the *zero-decimal* exception set TWD is NOT in): https://stripe.com/docs/currencies#zero-decimal
- Webhook signature verification: https://stripe.com/docs/webhooks/signature
- Test cards: https://stripe.com/docs/testing
- Cross-skill: [[stripe-best-practice]] · [[m2.1-buyer-to-admin-payments-prerequisites]] · [[m2.2-admin-to-seller-payment]] · [[supabase-best-practice]] · [[stripe-go-live]]
