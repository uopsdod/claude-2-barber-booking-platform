---
name: m2.2-admin-to-seller-payment-prerequisites
description: One-time carryover check before Milestone 2.2 of the barber-booking course — an INTERACTIVE, agent-driven walkthrough that confirms M2.1's money side is real before the commission-settlement (payout) workflow is built on top of it. M2.2 has NO new accounts/connectors/Stripe work to set up (M0–M2.1 wired AWS + Supabase MCP + GitHub PAT + Stripe), so this is a data-readiness check of exactly what M2.2 consumes: an `admin` user already exists (`role='admin'`, promoted once in the M2.1 prereq — M2.2 only GATES on it, never creates it), there are `paid` bookings with `payout_id IS NULL` (the owed pool the payout builder settles from — produced by M2.1's webhook, with NO fee columns since bookings carry no split), `commission_rates` is seeded (2026-01-01 = 0.20 — the VIEW + build_payout read the rate in force), the shop-attribution chain is intact (`bookings.service_id → services.barber_id → barbers.shop_id`, NO `bookings.barber_id`), the shop-level bank fields + `display_name` live on `profiles` (the payout snapshots `shop_name` + the admin transfers to the shop's bank account), and a clean settlement slate (the `payouts` table + the build/mark/cancel RPCs don't exist yet — M2.2 creates them — while `bookings.payout_id` exists from M1.2 and is all-NULL). There is NO `transactions` table and there are NO `platform_fee`/`barber_amount` columns — a zero-row check for those is the EXPECTED state, not a gap. Use when the student starts M2.2 for the first time, says "啟動 M2.2 前置" / "M2.2 prerequisites" / "check M2.2 setup", or when `m2.2-admin-to-seller-payment` / `-checklist` detects no admin, no paid bookings, `commission_rates` missing, or the Supabase MCP is unreachable.
---

# M2.2 Prerequisites — interactive carryover check (the agent drives)

**You are the Cowork agent running this skill. Drive the student through it one part at a time** — don't dump the whole thing. For each part: say what's about to happen, **check first** (Supabase MCP), tell the student only what actually needs fixing, run the read yourself, confirm it worked, and move on.

> **M2.2 introduces nothing new to set up.** M0 wired AWS/Vercel/Supabase and cached the GitHub PAT; M1.1/M1.2 created the seller + buyer data tables; M2.1 connected Stripe (sandbox), wired the pay-now webhook that flips a booking `pending_payment → paid` + stamps `paid_at`, created the `commission_rates` table, and — in its **prerequisite** — promoted one account to `role='admin'`. M2.2 is the **settlement** milestone: it builds `/admin/payouts` and `/shop/earnings` **on top of** that money, and adds only the `owed_bookings` VIEW + the `payouts` table + three admin RPCs. There is **no Stripe work in M2.2 at all** — the money is already in the platform's balance; M2.2 is reporting + an admin workflow. So this prereq is a **data-readiness carryover check**, not an account setup: confirm the admin exists, confirm there's actually **something owed to settle**, confirm the rate + attribution + bank fields are in place, then confirm the settlement tables are a clean slate.

> ⚠️ **The payout builder needs REAL owed money to exercise, not just tables.** A green M2.1 *checklist* proves the webhook + `commission_rates` are correct, but M2.2's happy path also needs **at least one `paid` booking that is not yet in a payout** — i.e. `status='paid' AND payout_id IS NULL` (the **owed pool**). If every paid booking is already paid out (or there are no paid bookings at all), `/admin/payouts` will render but the owed list will be **empty** and there's nothing to build a payout from. Check this here so the student doesn't hit an empty builder mid-build.

> ⚠️ **The admin is NOT created here — it must already exist from the M2.1 prereq.** There is **no public "sign up as admin" UI**; `admin` is reached **only** by the one-off migration the M2.1 prerequisite ran (`UPDATE public.profiles SET role='admin' WHERE email='…'`). M2.2 **consumes** `role='admin'` to gate `/admin/payouts` (middleware + RLS + the admin-guarded RPCs) — it never promotes anyone. So if `select … where role='admin'` returns **zero rows**, that's a **blocker**: send the student back to the M2.1 prerequisite (Part B) to promote their account, then return. Do **not** add a self-promote path here — the whole security model depends on admin being promotion-only.

> ⚠️ **There is NO `transactions` table and NO fee columns on bookings — and that is correct.** The old ledger model was dropped. "Money in" is simply a `paid` booking's `price`; the 20/80 split is computed **live** in the `owed_bookings` VIEW and snapshotted onto a `payouts` row at build time — never stored per booking. So a check that finds **no** `transactions` table and **no** `platform_fee`/`barber_amount` columns is the **expected** state, not a gap. If you *do* find them, that's the old model leaking in — flag it, don't celebrate it.

## Architecture (what this check unlocks)

This prereq confirms the M2 **money** side is real before the **settlement** side is built on it. M2.1 already produced, on the shared Supabase Database: `commission_rates` (the versioned 20% ratio, seeded `2026-01-01`) and a stream of **`paid` bookings** (Stripe Checkout → webhook flips `pending_payment → paid` + stamps `paid_at`, money landing in the platform's own Stripe balance), each carrying only its `price` snapshot + `paid_at` + a NULL `payout_id` — **no** per-booking split, **no** `transactions` row. M2.2 will add the real-time `owed_bookings` VIEW (paid bookings WHERE `payout_id IS NULL`, split derived from `commission_rates`, attributed via `bookings → services → barbers → shop_id`), the flexible `payouts` batch table, three admin RPCs (build / mark-transferred / cancel), and the `/admin/payouts` + `/shop/earnings` pages. This skill verifies the admin exists, `commission_rates` is seeded, there are owed paid bookings, the attribution chain + shop bank fields are intact, and the `payouts`/RPC slate is clean — then hands back to the M2.2 build skill.

The five things you confirm here are exactly what M2.2 builds on:
- **An `admin` user** (`role='admin'`) → what gates `/admin/payouts` (middleware + RLS + the admin-guarded RPCs). M2.2 does **not** create it.
- **`paid` bookings with `payout_id IS NULL`** (the owed pool) → what `owed_bookings` surfaces and the builder settles from. Produced by M2.1's webhook.
- **`commission_rates` seeded** → the rate the `owed_bookings` VIEW + `build_payout` apply (`price × platform_pct → platform_cut` / `shop_cut`). The split is derived, never a code constant.
- **The attribution chain** (`bookings.service_id → services.barber_id → barbers.shop_id`, **no `bookings.barber_id`**) → how a booking rolls up to **one shop** so a payout is one shop = one bank transfer.
- **Shop bank fields + `display_name` on `profiles`** → the admin transfers to `profiles.bank_account_*`; `build_payout` snapshots `shop_name = profiles.display_name`. Both RLS-restricted to shop + admin.
- **`bookings.payout_id` exists (from M1.2) + Supabase MCP reachable** → the FK the build/cancel RPCs stamp/null; the `payouts` table + RPCs are a **clean slate** (M2.2 creates them).

## When to load this skill

- The student says "啟動 M2.2 前置" / "M2.2 prerequisites" / "check M2.2 setup".
- The `[[m2.2-admin-to-seller-payment]]` build skill (or its checklist) detects a missing carryover — **no admin user**, no `paid` bookings, `commission_rates` not seeded, or the Supabase MCP connector is unreachable.

**Opening line to the student (say something like):**
> "M2.2 is the *settlement* side — the admin sees every paid booking that hasn't been paid out yet, builds a payout batch per shop, and records the bank transfer. It has nothing new to install: no new account, no Stripe work (the money's already in the platform's balance from M2.1). So I'll just confirm the carryovers are good — your **admin** account exists, there are actually **paid bookings owed** to settle, the commission rate is seeded, each booking rolls up to a shop with bank details, and the payout tables are a clean slate — then we build. Let me check what's already there."

---

## Step 1 — The admin user exists (the decisive gate — not created here)

M2.2's entire `/admin/payouts` page is gated to `role='admin'` (middleware **and** RLS **and** the admin-guarded RPCs). The admin was promoted **once**, via a one-off migration, in the **M2.1 prerequisite** — there is no public admin sign-up. Confirm it exists before building anything. All reads run through the Supabase MCP **`execute_sql`** tool.

> **Tool names:** this course's Supabase MCP namespaces its tools **per session** (e.g. `mcp__<session-id>__execute_sql`), so a hard-coded `mcp__claude_ai_Supabase__…` string may not resolve. Refer to the tools by their **bare names** — `list_tables`, `execute_sql`, `apply_migration`, `get_advisors`, `generate_typescript_types` — and call whichever namespaced variant your session exposes.

> **Note for Claude Code:** a single `execute_sql` with several `select`s returns only the **last** result set — run multi-read checks as **separate calls** or wrap them in one `json_build_object(...)`.

```sql
select id, email, role from public.profiles where role = 'admin';
```
- **≥ 1 row** with `role='admin'` → ✅ the payout page can be gated. Note the admin's email (you'll log in as them to test the build).
- **0 rows** → **BLOCKER.** Send the student back to the **M2.1 prerequisite (Part B)** to promote their account with a one-off migration:
  > "There's no admin account yet — and M2.2's payout page has to be admin-gated, so we can't build it without one. Admin isn't created in M2.2 (there's no 'sign up as admin' button on purpose — that's what keeps a user from making themselves admin). Hop back to the **M2.1 prerequisite**: sign up / pick the account you want, and I'll promote exactly that one with a one-off migration (`UPDATE public.profiles SET role='admin' WHERE email='你的信箱'` via `apply_migration`), then log out and back in. Then come back and we'll do M2.2."

> **Note for Claude Code:** do NOT add an admin sign-up path, a self-promote button, or an ad-hoc console `UPDATE` here. Promotion is the M2.1 prereq's job (via `apply_migration`, recorded in the migration history). If the student insists on "just make me admin now", still route it through the M2.1 prereq's one-off migration — never a raw SQL-editor edit ([[supabase-best-practice]]).

---

## Step 2 — There is actually something owed to settle (`paid` bookings, `payout_id IS NULL`) + the rate is seeded

This is the check the M2.1 checklist doesn't frame as a *pool*: M2.2's happy path needs **live owed money** to build a payout against, and the **commission rate** to split it.

**2a — at least one `paid` booking is not yet in a payout (the owed pool):**
```sql
select count(*) filter (where status='paid' and payout_id is null) as owed_paid,
       count(*) filter (where status='paid')                        as total_paid,
       count(*) filter (where status='paid' and payout_id is not null) as already_paid_out
from public.bookings;
```
- **`owed_paid >= 1`** → ✅ there's something to settle; the builder's owed list won't be empty. (One is enough to exercise the flow; a few across two shops is ideal for testing the same-shop guard later — see the note.)
- **`owed_paid = 0` but `total_paid >= 1`** → every paid booking is already in a payout. That can happen if the student pre-ran something; for a clean M2.2 build-and-test you want at least one **owed** booking. Have them make one more test booking + pay it (M2.1 flow) so the owed list has a row to pick.
- **`total_paid = 0`** → M2.1 never produced a `paid` booking (the webhook isn't flipping bookings). **Stop and fix M2.1 first** — run `[[m2.1-buyer-to-admin-payments-checklist]]`; the owed pool is meaningless without paid bookings. Do **not** proceed to build M2.2 against an empty owed pool.

Peek at a couple of owed rows to confirm the shape (each carries a `price` snapshot + `paid_at`, and **no** fee columns exist to select):
```sql
select id, status, price, paid_at, payout_id, service_id
from public.bookings
where status='paid' and payout_id is null
order by paid_at desc
limit 5;
```
- You should see `status='paid'`, a whole-integer `price` (in `platform_settings.currency` — no cents, no ×100), a non-null `paid_at`, and `payout_id = NULL`. There is **no** `platform_fee`/`barber_amount` to select — the split is computed in M2.2's VIEW, not stored here (that absence is correct).

**2b — `commission_rates` is seeded (the rate the split is derived from):**
```sql
select effective_from, platform_pct, note
from public.commission_rates
order by effective_from;
```
- At least the seed row **`effective_from = 2026-01-01`, `platform_pct = 0.2000`** → ✅. M2.2's `owed_bookings` VIEW and `build_payout` read the row with the greatest `effective_from <= the booking's paid date` (or `current_date` at build time). `platform_cut = round(price × platform_pct)`, `shop_cut = price − platform_cut` → they sum back to `price` exactly.
- **Empty / no row** → M2.1 Step 2's `commission_rates` seed didn't land. Send the student back to `[[m2.1-buyer-to-admin-payments]]` Step 2 to apply the `commission_rates` migration (seed `2026-01-01 = 0.20`). Without a rate, every owed split is undefined and `build_payout` can't snapshot `platform_pct`.

> **Note for Claude Code (nice-to-have, not a blocker):** M2.2's checklist tests the **same-shop guard** (`build_payout` must reject a mixed-shop selection). That test is easiest if the owed pool spans **two different shops**. If all owed bookings resolve to one shop, note it — the student can add a second bookable shop + a paid booking during/after the build to exercise the guard, but it's not required to *start* M2.2.

---

## Step 3 — Attribution chain + shop bank fields are intact

M2.2 attributes every booking to **one shop** via `bookings → services → barbers → shop_id` (there is **no** `bookings.barber_id`, and it never goes through the slot), and the payout snapshots the shop's `display_name` + transfers to the shop's bank account on `profiles`. Confirm both are shaped right.

**3a — `bookings` has NO `barber_id` (attribution goes through the service), and HAS `payout_id`:**
```sql
select column_name from information_schema.columns
where table_schema='public' and table_name='bookings'
order by ordinal_position;
```
- Columns include **`service_id`** and **`payout_id`**, and there is **NO `barber_id`** (and no `start_slot_id`) → ✅. This is what lets M2.2 roll a booking up to a shop via `service_id → services.barber_id → barbers.shop_id`, and stamp/null `payout_id` on build/cancel.
- A **`barber_id` on bookings** (or a missing `payout_id`) → the old denormalized model leaked in. Flag it: M2.2's VIEW + RPCs attribute via the **service**, and stamp settlement onto **`payout_id`**. *Recovery:* re-apply the M1.2 `bookings` migration (no `barber_id`; `payout_id` FK present).

**3b — the attribution chain actually resolves for the owed bookings** (a live join, so a broken `service_id`/`barber_id` surfaces now, not mid-build):
```sql
select b.id as booking_id, s.barber_id, bar.shop_id, p.display_name as shop_name
from public.bookings b
join public.services s   on s.id  = b.service_id
join public.barbers  bar on bar.id = s.barber_id
join public.profiles p   on p.id  = bar.shop_id
where b.status='paid' and b.payout_id is null
limit 5;
```
- Each owed booking resolves to a **non-null `shop_id`** and a **non-null `display_name`** (the shop name) → ✅ the VIEW's shop rollup + the payout's `shop_name` snapshot will work and show a real name.
- A row drops out of the join (NULL `shop_id` / no matching barber or profile) → a data-integrity gap in the seller data; fix it in M1.1's onboarding before building the settlement page (a booking with no resolvable shop can never be paid out).
- The join resolves but **`display_name` is NULL/blank** → the shop finished before M1.1 made the shop name required (or the onboarding form didn't enforce it). Not a hard blocker — the bank transfer keys off `bank_account_*`, not the name — but `build_payout` snapshots `shop_name = display_name`, so the payout row and the `/admin/payouts` list would show a **blank shop name**. Set it before building for a clean demo: `update public.profiles set display_name = '<shop name>' where id = '<shop_id>';` (and, in M1.1, the onboarding form should **require** `display_name` — see `[[m1.1-seller-setup]]` Section A / `[[m1.1-seller-setup-checklist]]` F3).

**3c — the shop-level bank fields + `display_name` live on `profiles` (NOT on `barbers`), and are RLS-restricted:**
```sql
-- bank fields + display_name are on profiles (shop level):
select column_name from information_schema.columns
where table_schema='public' and table_name='profiles'
  and (column_name ilike 'bank%' or column_name = 'display_name')
order by column_name;
-- and barbers carries NO bank columns:
select count(*) as barber_bank_cols from information_schema.columns
where table_schema='public' and table_name='barbers' and column_name ilike 'bank%';
```
- `profiles` has the bank field(s) (e.g. `bank_account_name` / `bank_account_number`) **and** `display_name`, and **`barber_bank_cols = 0`** → ✅. The admin transfers to `profiles.bank_account_*`; `build_payout` snapshots `shop_name = profiles.display_name`.
- Bank fields **missing on `profiles`** or **present on `barbers`** → the M1.1 payout-settings model didn't land / leaked to the wrong table. Send the student to `[[m1.1-seller-setup]]` (shop-level payout settings on `profiles`). The bank account is on the **shop**, since one shop = one bank account.

> **Note for Claude Code:** the Supabase MCP runs **privileged and BYPASSES RLS**, so here you verify policy **definitions** and column placement, not behavioral denial. The decisive "a non-admin/non-shop can't read the bank fields" test is behavioral and lives in the **`[[m2.2-admin-to-seller-payment-checklist]]`** (Section E), run in the live app after the build — not here. **Apply no migration in this prereq** — M2.2 owns the `payouts`/VIEW/RPC schema.

---

## Step 4 — One verified read: the settlement tables are a clean slate

M2.2 creates the `payouts` table + the `owed_bookings` VIEW + three RPCs (`build_payout` / `mark_payout_transferred` / `cancel_payout`). The "verified read" here is the **negative** read that proves they don't exist yet — exactly the clean slate the build skill starts from — plus a confirm that the old ledger model is absent (which is correct) and `bookings.payout_id` is all-NULL.

**4a — `payouts` / `owed_bookings` / the RPCs don't exist yet:**
```sql
select to_regclass('public.payouts')                as payouts,
       to_regclass('public.owed_bookings')          as owed_bookings_view,
       to_regproc('public.build_payout')            as build_payout,
       to_regproc('public.mark_payout_transferred') as mark_transferred,
       to_regproc('public.cancel_payout')           as cancel_payout;
```
- All five return **NULL** → ✅ clean slate; M2.2 Step 1 creates them fresh.
- One or more **already exists** (a partial earlier run) → read its shape and tell the student. The M2.2 migration uses `create ... if not exists` / `create or replace`, so it's safe to re-run, but confirm the shape matches the current model (`payouts.shop_id` **NOT** unique — a shop has many payouts; status `pending_transfer`/`transferred`/`cancelled`; the `shop_name` snapshot column) before layering the UI, so an old model doesn't leak forward.

**4b — the old ledger model is absent (correct), and `payout_id` is a clean NULL slate:**
```sql
select to_regclass('public.transactions')    as transactions_table,   -- must be NULL
       to_regclass('public.payout_records')   as payout_records_table,  -- must be NULL
       count(*) filter (where payout_id is not null) as bookings_already_linked
from public.bookings;
```
- `transactions_table` and `payout_records_table` both **NULL**, and (ideally) `bookings_already_linked = 0` → ✅ the old money model is gone and no booking is pre-linked to a (non-existent) payout. This absence is the **expected** state.
- A `transactions` or `payout_records` table **exists** → the old model leaked in. Flag it: M2.2 has **no** ledger table and **no** fee columns; settlement is derived from `payout_id`. Don't build on top of a stale ledger.
- `bookings_already_linked > 0` while `payouts` doesn't exist → a dangling `payout_id` pointing at nothing. Investigate before building (likely a leftover from a partial run).

> **Prereq scope note:** this prereq deliberately does **static** checks only — table/column/constraint/proc existence, seed data, and one live attribution join (Steps 1–4). It does **not** call `build_payout`/`cancel_payout` or exercise the admin gate. **Behavioral verification** — the live "only `role='admin'` reaches `/admin/payouts`" denial, the same-shop guard rejecting a mixed selection, no-double-pay, the mark-transferred/cancel round-trip, and the bank-field RLS denial — lives in the **`[[m2.2-admin-to-seller-payment-checklist]]`**, which runs *after* the build creates those RPCs and pages. That's the right home for behavioral proofs.

---

## Verify (all must pass)

- ✅ **Admin exists** — `select … where role='admin'` returns ≥ 1 row (promoted in the M2.1 prereq; M2.2 only gates on it, never creates it) (Step 1). **Zero rows = blocker → back to the M2.1 prerequisite Part B.**
- ✅ **Something owed to settle + rate seeded** — ≥ 1 `paid` booking with `payout_id IS NULL` (the owed pool), and `commission_rates` has the `2026-01-01 = 0.2000` seed row (Step 2).
- ✅ **Attribution + bank fields intact** — `bookings` has `service_id` + `payout_id` and **no `barber_id`**; every owed booking resolves `service → barber → shop → profiles.display_name`; the bank fields + `display_name` are on `profiles` and `barbers` has **no** bank columns (Step 3).
- ✅ **Clean settlement slate** — `payouts` / `owed_bookings` / the three RPCs are NULL (don't exist yet); no `transactions`/`payout_records` table; `bookings.payout_id` all-NULL — ready for M2.2's migration (Step 4).

## Next step

When all four are ✅, tell the student:
「前置檢查通過 ✅ —— 你的 **admin** 帳號在（M2.1 前置作業升級的，M2.2 只是拿來 gate 撥款頁、不會再建）；而且真的有**還沒撥款的 paid bookings**（`status='paid'` 且 `payout_id IS NULL`，也就是「欠款池」）可以結算、`commission_rates` 的 `2026-01-01 = 0.20` 費率也 seed 好了；每筆 booking 都能經 `service → barber → shop` 歸戶到一間有 `display_name` 與銀行資料的店家（銀行欄位在 `profiles`、`barbers` 沒有）；`payouts` 表、`owed_bookings` VIEW 和三支 RPC（建立撥款／標記已轉帳／取消）都還是乾淨的空白狀態，`bookings.payout_id` 也全是 NULL，沒有 `transactions` 表也沒有抽成欄位（這是對的）。接下來我會用一支 migration 建 `payouts` 表 ＋ `owed_bookings` VIEW ＋ 三支 admin RPC（含 RLS），再做 `/admin/payouts`（欠款池 + 篩選 + 勾選建立撥款 + 標記已轉帳/取消）和 `/shop/earnings`（店家唯讀鏡像）。跟我說『啟動 M2.2』就開始。」
Then return to the build skill `[[m2.2-admin-to-seller-payment]]` (Step 0/1).

## Reference

- `[[m2.1-buyer-to-admin-payments]]` / `[[m2.1-buyer-to-admin-payments-checklist]]` — the money side (webhook → `paid` bookings, `commission_rates`) this prereq confirms is real.
- `[[m2.1-buyer-to-admin-payments-prerequisites]]` — where the `admin` user was promoted (Part B); the home to send the student back to if no admin exists.
- `[[m2.2-admin-to-seller-payment]]` — the settlement build skill this prereq gates.
- `[[m2.2-admin-to-seller-payment-checklist]]` — the post-build verification (which runs the behavioral admin-gate / same-shop / no-double-pay / bank-RLS tests this prereq intentionally defers).
- `[[m1.1-seller-setup]]` — where the shop-level bank fields + `display_name` on `profiles` come from.
- `[[supabase-best-practice]]` — migration/RLS discipline (and why admin promotion is a migration, never a console edit).
