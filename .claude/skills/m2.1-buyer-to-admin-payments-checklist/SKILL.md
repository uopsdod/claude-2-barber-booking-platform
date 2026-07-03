---
name: m2.1-buyer-to-admin-payments-checklist
description: 抽成制理髮師預約平台 Milestone 2.1 verification — checks the Stripe booking-payment flow is real and correctly wired: a DYNAMIC Checkout Session whose charged `amount` is `price × 100` for TWD (TWD is 2-decimal in Stripe — verify via the PaymentIntent/Charge, NOT off currency_minor_units, which is display-only) with `metadata.{booking_id,customer_id}` (no barber_id, no start_slot_id — the barber/slots are derivable from the booking), raw-body signature verify, the webhook exemption (POST /api/stripe/webhook returns 200/400 not 307 for Next.js middleware, or is not swallowed by the Vite `vercel.json` SPA rewrite), the booking flips `pending_payment`→`paid` on paid + `paid_at` stamped with `payout_id` still NULL (there is no slot status to flip — slots have no status), NO transactions row and NO fee columns are written, `commission_rates` is seeded (2026-01-01 = 0.20), and that an admin account (role='admin') exists. Idempotency (the `pending_payment` status guard) is enforced in the build, not manually re-tested here. Use when the student says "驗收 M2.1", "check M2.1", "M2.1 done?", "Stripe 金流好了嗎", or after the `m2.1-buyer-to-admin-payments` skill completes Step 9.
---

# M2.1 — Stripe 預約金流 Checklist

## What this skill does

Verifies the student actually completed M2.1 — not just *thinks* they did. The Stripe pieces fail **silently** (a 307'd / SPA-swallowed webhook, a `× 1` TWD charge that's rejected below Stripe's minimum), so this skill tests each one against the live deployment + Supabase and reports pass/fail per item, then emits a `READY for M2.2` verdict. The webhook's whole job is to flip `pending_payment → paid` + stamp `paid_at` (leaving `payout_id` NULL = owed) — there is **no `transactions` table** and **no fee columns** to check; "money in" is just the `paid` booking's `price`.

**Run this AFTER `m2.1-buyer-to-admin-payments` Step 9, or any time the student claims M2.1 is done.**

## Execution mode: Cowork vs CLI (read this first)

| Section | CLI mode tool | Cowork mode equivalent |
|---|---|---|
| A — Checkout session shape + charged amount | Stripe CLI / browser | Stripe MCP PaymentIntent/Charge read (Checkout Sessions may be read-denied — read route source + verify amount off the PI) |
| B — Webhook (raw-body verify, exemption) | `curl` + route source | `curl` for the exemption check + read the route source |
| C — Booking flips to paid + commission_rates seeded | Supabase SQL | Supabase MCP (`execute_sql`) — preferred both modes |
| D — Admin account | Supabase SQL | Supabase MCP (`execute_sql`) |

In Cowork mode every Bash block is CLI-only — use the equivalent. Don't try to install `curl`/Stripe CLI in Cowork.

## How to run

The student invokes this directly (e.g. types `驗收 M2.1`). You (Claude Code) **actively run** each check and report results — don't just describe them.

### Step 1: Collect inputs (one message)

Ask the student for:
1. Vercel deploy URL (`https://<app>.vercel.app`)
2. A **booking id** they just paid for with test card `4242 4242 4242 4242` (or have them do one test booking now)
3. Supabase project ref + confirmation the Stripe MCP / CLI is connected in **sandbox**

### Step 2: Run the checklist

#### Section A — Dynamic checkout session (`unit_amount` = TWD `price × 100`)
- **A1** The `POST /api/bookings/checkout` route builds a **dynamic `price_data`** Session (not a fixed Price object), with `metadata.{booking_id,customer_id}` + `client_reference_id` (the barber is NOT stored on the booking — it's derivable via `service_id → services.barber_id`, and the slots live in `booking_slots`; there is **no `start_slot_id`** on the booking, so the metadata carries neither `barber_id` nor `start_slot_id` — just `booking_id`). Confirm `mode: payment`, `currency` matches `platform_settings.currency` (default `twd`), `metadata.booking_id` present (and `client_reference_id` = the same booking_id).
  ```bash
  # Stripe CLI (sandbox): list recent sessions and read one
  stripe checkout sessions list --limit 1
  ```
  *(Cowork/restricted key: the Checkout Sessions resource is often read-denied — confirm this shape by **reading the route source** instead; the charged amount is verified off the PaymentIntent in A2.)*
- **A2** **`unit_amount` uses Stripe's smallest unit for the currency (TWD → `price × 100`).** A NT$300 cut's charged `amount` must be **`30000`** (= NT$300.00), **NOT** `300`. **Verify via the Stripe MCP PaymentIntent/Charge `amount`** — key off the `stripe_payment_intent_id` the webhook stamped (`fetch_stripe_resources(pi_…)` or a PaymentIntent/Charge read), **not** by listing Checkout Sessions (that resource is read-denied on the restricted Cowork key; PaymentIntents/Charges reads work). Confirm `status: succeeded`. Do NOT drive the scale off `currency_minor_units` (that's display-only).
  *Recovery if `× 1`:* it billed NT$3.00 (below Stripe's ~50¢ minimum → the Session is **rejected** and the customer never pays); fix to `price × 100` via Stripe's zero-decimal set (M2.1 Step 4 / [[stripe-best-practice]] Rule 0 — TWD is 2-decimal).

#### Section B — Webhook: raw-body verify, webhook-path exemption
- **B1** **Webhook path reaches the handler — returns 200/400, NOT 307 (Next.js) and NOT the HTML shell (Vite).** A POST to the webhook path must reach your function (signature check), not get redirected to `/login` (Next.js middleware) or served `index.html` (Vite `vercel.json` SPA rewrite swallowing `/api/*`):
  ```bash
  curl -sS -X POST https://<app>.vercel.app/api/stripe/webhook | head -c 200
  # also check the status code:
  curl -sS -o /dev/null -w "%{http_code}\n" -X POST https://<app>.vercel.app/api/stripe/webhook
  ```
  Expect **400** (signature missing → handler ran) or **200**, and a JSON/plain-text body — **not** a **307** redirect (Next.js middleware) and **not** an HTML `<!doctype html>` shell (Vite rewrite swallowed it).
  *Recovery — Next.js:* add `api/stripe/webhook` to the middleware matcher exclusion. *Recovery — Vite SPA:* exclude `/api/` from the `vercel.json` rewrite (`"source": "/((?!api/).*)"`). (M2.1 Step 7 / [[stripe-best-practice]] Rule 5.)
- **B2** **Raw-body signature verify** — the handler reads `await req.text()` before `stripe.webhooks.constructEvent`, NOT `req.json()`. Confirm in the route source, and that the real paid event (from your test booking) was accepted (Stripe dashboard → that event → 200 response).
  *Recovery:* M2.1 Step 6 / [[stripe-best-practice]] Rule 2.
> **Idempotency is enforced in the build, not manually re-tested here.** The `.eq('status','pending_payment')` status guard (optionally backed by a `UNIQUE(stripe_payment_intent_id)`) makes a re-delivered event match zero rows and no-op — confirm it's present in the webhook source (M2.1 Step 6 / [[stripe-best-practice]] Rule 3). No manual "resend the event" step.

#### Section C — Booking flips to paid + commission_rates seeded
- **C1** **Booking flipped to `paid` on payment, `payout_id` still NULL (= owed)** — after the test payment:
  ```sql
  select status, paid_at, price, payout_id from public.bookings where id = '<booking_id>';
  ```
  Expect `status = 'paid'`, a non-null `paid_at`, `price` unchanged (the M1.2 snapshot), and `payout_id` **NULL** — `bookings.status` is **3 states only** (`pending_payment | paid | cancelled`); there is no `payout_pending`/`payout_transferred`. The booking is now in the **owed pool** (`paid` + `payout_id` NULL); M2.2 stamps `payout_id` when the admin builds a payout. There is **no slot status to flip** — slots have no status; the `paid` booking (plus the `uniq_slot_held` unique index on `booking_slots`) is what keeps the slot held, and the availability anti-join excludes any slot with a live (`pending_payment`/`paid`) booking, so first to pay wins.
  *Recovery:* the webhook isn't flipping the row — check B1/B2 first, then M2.1 Step 6. (If the webhook touched `payout_id`, that's a bug — it must only write `status`/`paid_at`.)
- **C2** **NO ledger row, NO fee columns — `commission_rates` is seeded instead.** The webhook writes NO `transactions` row and `bookings` has NO `platform_fee`/`barber_amount` columns; the 20/80 split is M2.2's payout-build job. What M2.1 must have created is the seeded `commission_rates` table:
  ```sql
  -- there is no transactions table to query; verify the rate table instead
  select effective_from, platform_pct from public.commission_rates order by effective_from;
  ```
  Expect at least the seed row `effective_from = 2026-01-01`, `platform_pct = 0.2000`. "Money in" for M2.2 is summed directly from the admin's picked `paid` bookings' `price` × this rate — there is no per-booking ledger.
  *Recovery:* M2.1 Step 2 — apply the `commission_rates` migration (seed `2026-01-01 = 0.20`); do NOT add a `transactions` table or fee columns ([[supabase-best-practice]]).

#### Section D — Admin account exists
- **D1** **An admin account exists** (promoted in the prereq via a one-off migration):
  ```sql
  select id, email, role from public.profiles where role = 'admin';
  ```
  Expect at least one row with `role = 'admin'`. M2.2's `/admin/payouts` gates on this.
  *Recovery:* run `m2.1-buyer-to-admin-payments-prerequisites` Part B (find the account → promote with `apply_migration`). A user can never self-escalate, so this only appears via that migration.

## Reporting

Emit a table:

| Check | Status | Notes |
|---|---|---|
| A1 dynamic checkout session + metadata | ✅ / ❌ | `price_data`, `metadata.{booking_id,customer_id}` (no barber_id, no start_slot_id), `client_reference_id` |
| A2 charged amount = TWD `price × 100` (verify off PaymentIntent/Charge) | ✅ / ❌ | `amount` = 30000 for NT$300, not 300; TWD is 2-decimal — not driven off `currency_minor_units` |
| B1 webhook path reaches handler (200/400, not 307 / not HTML shell) | ✅ / ❌ | the silent-failure trap — Next.js middleware OR Vite `vercel.json` rewrite |
| B2 raw-body signature verify | ✅ / ❌ | `req.text()` (Next.js) / raw stream buffer + `bodyParser:false` (Vite) before `constructEvent` |
| C1 booking → paid on payment, payout_id NULL (no slot status to flip) | ✅ / ❌ | `status='paid'` + `paid_at` set + `payout_id` NULL (owed; 3-state status); slot held via paid booking + unique index |
| C2 NO transactions row / NO fee columns; commission_rates seeded | ✅ / ❌ | `commission_rates` has `2026-01-01 = 0.2000`; no per-booking ledger |
| D1 admin account exists (role='admin') | ✅ / ❌ | from the prereq's one-off migration |

**Verdict:**
- All ✅ → 「M2.1 驗收通過 ✅ READY for M2.2。預約已經是『付款成功才鎖定』——webhook 把 booking 從 `pending_payment` 翻成 `paid`、蓋上 `paid_at`、`payout_id` 仍是 NULL（代表「欠撥」），沒有任何 ledger 或拆帳欄位（拆帳是 M2.2 由 admin 挑選欠撥的 `paid` 預約、組成撥款批次時用 `commission_rates` 算）。`commission_rates` 已 seed、admin 帳號也備好了。跟我說『啟動 M2.2』，我們來做 admin 撥款頁。」
- Any ❌ → list the failed items + the matching recovery step, and tell the student to fix then re-run `驗收 M2.1`. (Most common fails: A2 `× 1` TWD → rejected below Stripe's minimum, B1 307 / SPA-swallowed webhook.)
