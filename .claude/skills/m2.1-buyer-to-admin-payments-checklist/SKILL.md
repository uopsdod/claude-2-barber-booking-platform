---
name: m2.1-buyer-to-admin-payments-checklist
description: 抽成制理髮師預約平台 Milestone 2.1 verification — checks the Stripe booking-payment flow is real and correctly wired: a DYNAMIC Checkout Session whose `unit_amount` honors `platform_settings.currency_minor_units` (TWD default config ×1, not ×100) with `metadata.{booking_id,customer_id}` (no barber_id, no start_slot_id — the barber/slots are derivable from the booking), raw-body signature verify, idempotency (resend the event from the Stripe dashboard → no double-process via the `pending_payment` status guard), the middleware exemption (POST /api/stripe/webhook returns 200/400 not 307), the booking flips `pending_payment`→`paid` on paid + `paid_at` stamped with `payout_id` still NULL (there is no slot status to flip — slots have no status), NO transactions row and NO fee columns are written, `commission_rates` is seeded (2026-01-01 = 0.20), and that an admin account (role='admin') exists. Use when the student says "驗收 M2.1", "check M2.1", "M2.1 done?", "Stripe 金流好了嗎", or after the `m2.1-buyer-to-admin-payments` skill completes Step 9.
---

# M2.1 — Stripe 預約金流 Checklist

## What this skill does

Verifies the student actually completed M2.1 — not just *thinks* they did. The Stripe pieces fail **silently** (a 307'd webhook, a ×100 TWD charge, a non-idempotent handler), so this skill tests each one against the live deployment + Supabase and reports pass/fail per item, then emits a `READY for M2.2` verdict. The webhook's whole job is to flip `pending_payment → paid` + stamp `paid_at` (leaving `payout_id` NULL = owed) — there is **no `transactions` table** and **no fee columns** to check; "money in" is just the `paid` booking's `price`.

**Run this AFTER `m2.1-buyer-to-admin-payments` Step 9, or any time the student claims M2.1 is done.**

## Execution mode: Cowork vs CLI (read this first)

| Section | CLI mode tool | Cowork mode equivalent |
|---|---|---|
| A — Checkout session shape | Stripe CLI / browser | Stripe MCP (`mcp__claude_ai_Stripe__*`), or inspect in Stripe dashboard |
| B — Webhook (verify, idempotency, exemption) | `curl` + Stripe dashboard **Resend** | Stripe MCP + dashboard; `curl` for the 307 check |
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

#### Section A — Dynamic checkout session (`unit_amount` honors `currency_minor_units`)
- **A1** The `POST /api/bookings/checkout` route builds a **dynamic `price_data`** Session (not a fixed Price object), with `metadata.{booking_id,customer_id}` + `client_reference_id` (the barber is NOT stored on the booking — it's derivable via `service_id → services.barber_id`, and the slots live in `booking_slots`; there is **no `start_slot_id`** on the booking, so the metadata carries neither `barber_id` nor `start_slot_id` — just `booking_id`). Inspect the most recent test Checkout Session:
  ```bash
  # Stripe CLI (sandbox): list recent sessions and read one
  stripe checkout sessions list --limit 1
  ```
  Confirm `mode: payment`, `currency` matches `platform_settings.currency` (default `twd`), `metadata.booking_id` present (and `client_reference_id` = the same booking_id).
- **A2** **`unit_amount` honors `currency_minor_units`** — the charged `amount_total` equals `price * 10^currency_minor_units`. With the default **TWD** config (`currency_minor_units = 0`, zero-decimal) a NT$500 cut's `amount_total` is **`500`**, NOT `50000`. The route must read `platform_settings.currency_minor_units`, not hard-code ×100.
  *Recovery if ×100 on TWD:* the route hard-coded a USD-cents `unit_amount`; fix to `unit_amount: price * 10 ** currency_minor_units` reading `platform_settings` (M2.1 Step 4 / [[stripe-best-practice]] TWD rule).

#### Section B — Webhook: raw-body verify, idempotency, middleware exemption
- **B1** **Middleware exemption — returns 200/400, NOT 307.** A POST to the webhook path must reach the handler (signature check), not get redirected to `/login`:
  ```bash
  curl -sS -o /dev/null -w "%{http_code}\n" -X POST https://<app>.vercel.app/api/stripe/webhook
  ```
  Expect **400** (signature missing → handler ran) or **200**. A **307** = middleware is redirecting the webhook to `/login`; Stripe's events never reach you.
  *Recovery:* add `api/stripe/webhook` to the middleware matcher exclusion (M2.1 Step 7 / [[stripe-best-practice]] Rule 5).
- **B2** **Raw-body signature verify** — the handler reads `await req.text()` before `stripe.webhooks.constructEvent`, NOT `req.json()`. Confirm in the route source, and that the real paid event (from your test booking) was accepted (Stripe dashboard → that event → 200 response).
  *Recovery:* M2.1 Step 6 / [[stripe-best-practice]] Rule 2.
- **B3** **Idempotency — resend the event, no double-process.** In the Stripe dashboard → Developers → Events → the `checkout.session.completed` for your test booking → **Resend**. Then re-read the booking row:
  ```sql
  select status, paid_at from public.bookings where id = '<booking_id>';
  ```
  The row must be **unchanged** (still `paid`, same `paid_at` — the re-delivery must NOT re-stamp it) and the resend must return **200** `{received:true}`.
  *Recovery:* add the `.eq('status','pending_payment')` status guard so a re-delivered event matches zero rows and no-ops (M2.1 Step 6 / [[stripe-best-practice]] Rule 3).

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
| A2 `unit_amount` honors `currency_minor_units` (TWD default ×1, NOT ×100) | ✅ / ❌ | `amount_total` = price × 10^minor_units (TWD config → 500, not 50000) |
| B1 middleware exemption (200/400, not 307) | ✅ / ❌ | the silent-failure trap |
| B2 raw-body signature verify | ✅ / ❌ | `req.text()` before `constructEvent` |
| B3 idempotency (resend → no double-process) | ✅ / ❌ | row unchanged; `paid_at` not re-stamped (status guard) |
| C1 booking → paid on payment, payout_id NULL (no slot status to flip) | ✅ / ❌ | `status='paid'` + `paid_at` set + `payout_id` NULL (owed; 3-state status); slot held via paid booking + unique index |
| C2 NO transactions row / NO fee columns; commission_rates seeded | ✅ / ❌ | `commission_rates` has `2026-01-01 = 0.2000`; no per-booking ledger |
| D1 admin account exists (role='admin') | ✅ / ❌ | from the prereq's one-off migration |

**Verdict:**
- All ✅ → 「M2.1 驗收通過 ✅ READY for M2.2。預約已經是『付款成功才鎖定』——webhook 把 booking 從 `pending_payment` 翻成 `paid`、蓋上 `paid_at`、`payout_id` 仍是 NULL（代表「欠撥」），沒有任何 ledger 或拆帳欄位（拆帳是 M2.2 由 admin 挑選欠撥的 `paid` 預約、組成撥款批次時用 `commission_rates` 算）。`commission_rates` 已 seed、admin 帳號也備好了。跟我說『啟動 M2.2』，我們來做 admin 撥款頁。」
- Any ❌ → list the failed items + the matching recovery step, and tell the student to fix then re-run `驗收 M2.1`. (Most common fails: A2 ×100 TWD, B1 307 webhook, B3 missing idempotency.)
