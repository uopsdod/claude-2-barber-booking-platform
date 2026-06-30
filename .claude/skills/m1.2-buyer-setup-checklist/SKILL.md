---
name: m1.2-buyer-setup-checklist
description: 抽成制理髮師預約平台 Milestone 1.2 verification — checks the buyer booking flow is real and correctly wired: a customer can browse every barber on /barbers, open /barbers/[id], click Book to open the date-select DIALOG, pick a date + available slot, confirm to close the dialog and return to /barbers/[id], a `pending_payment` booking row is created with a 3-state status and NO `start_slot_id` (its start time is DERIVED as MIN(starts_at) over its `booking_slots` via the `bookings_with_start` view) + N `booking_slots` rows (N = service.required_slots) and NO `barber_id` column on bookings, the slot is no longer offered because the availability anti-join against `booking_slots` excludes it, a second live booking on the same slot is rejected by the `UNIQUE(slot_id)` on `booking_slots`, cancelling the booking deletes its `booking_slots` rows (frees the slot), the customer sees it on /bookings, and RLS denies reading another customer's bookings. The terminal M1.2 state is `pending_payment` with `paid_at` NULL and `payout_id` NULL — do NOT expect `paid`/`paid_at` or any `payout_*` status (settlement is DERIVED from `payout_id`, not a booking status). Use when the student says "驗收 M1.2", "check M1.2", "M1.2 done?", or after the `m1.2-buyer-setup` skill completes Step 6.
---

# M1.2 — Buyer Booking Checklist

## What this skill does

Verifies the student actually built the M1.2 buyer flow — not just *thinks* they did. People (and LLMs) skip steps. This skill tests every artifact (the `bookings` table + the `booking_slots` join table + RLS, the browse/detail pages, the Book pop-up dialog, the availability anti-join, the my-bookings page) and reports pass/fail per item, then emits a `READY for M2.1` verdict.

**Run this AFTER `m1.2-buyer-setup` Step 6, or any time the student claims M1.2 is done.**

Remember the M1.2 model: **no payment yet, and slots have NO status column.** A successful booking inserts a `bookings` row with `status='pending_payment'` (and `paid_at` NULL, `payout_id` NULL) plus **N `booking_slots(booking_id, slot_id)` rows** (N = service.required_slots) — **all N slots, including the first, live in `booking_slots`**; there is **no `start_slot_id`, no `slot_id`, and no `barber_id` column on bookings**. The booking's start time is **DERIVED** as `MIN(starts_at)` over its `booking_slots` rows (read via the `bookings_with_start` view), and the barber is reached through `bookings → services → barbers`. `bookings.status` is a **3-state** machine (`pending_payment | paid | cancelled`) — **settlement state is DERIVED from `payout_id`, NOT a status**, so there are **no `payout_pending` / `payout_transferred` statuses**. The slot itself is unchanged — its availability is **derived**: a slot is bookable ⟺ NOT EXISTS a `booking_slots` row referencing it, so a fresh booking makes the slot drop out of the available list automatically (and cancelling that booking DELETEs its `booking_slots` rows, freeing the slot again). **`pending_payment` is the TERMINAL M1.2 state — `paid_at=NULL` and `payout_id=NULL` are correct and complete; do NOT expect `paid`/`paid_at`.** `paid` does NOT appear in M1.2 (that's M2.1, the webhook). If you see a `paid` booking, a `paid_at` stamp, a `start_slot_id`/`slot_id`/`barber_id` column on `bookings`, a `payout_*` status, or a `status` column on `bookable_slots`, something jumped ahead / the old model leaked in.

## Execution mode: Cowork vs CLI (read this first)

| Section | CLI mode tool | Cowork mode equivalent |
|---|---|---|
| A — Schema + RLS (`bookings`) | Supabase SQL editor / `psql` | Supabase MCP `execute_sql` / `get_advisors` (preferred both modes) |
| B — Buyer pages reachable | `curl` | Vercel MCP, or open URL in browser |
| C — Book dialog + booking write | browser (click Book) + Supabase MCP | browser/Playwright MCP + Supabase MCP |
| D — My-bookings + RLS isolation | browser + Supabase MCP | Supabase MCP (preferred both modes) |

In Cowork mode every Bash block below is CLI-only — use the equivalent. Don't try to install `curl` in Cowork.

## How to run

The student invokes this directly (e.g. types `驗收 M1.2`). You (Claude Code) **actively run** each check and report results — don't just describe them.

### Step 1: Collect inputs (one message)

Ask the student for:
1. Vercel deploy URL (`https://<app>.vercel.app`)
2. Supabase project ref (so the MCP targets the right project)
3. Two test customer logins (Customer A + Customer B) — needed for the RLS-isolation test in D3. (Customer A also needs at least one barber with a published, still-available slot — a slot with no `booking_slots` row referencing it — from M1.1.)

### Step 2: Run the checklist

#### Section A — `bookings` + `booking_slots` schema + RLS

- **A1** The `bookings` table exists with the expected columns, and **NO `start_slot_id` / NO `slot_id` / NO `barber_id`**:
  ```sql
  -- via Supabase MCP execute_sql:
  select column_name, data_type, is_nullable
  from information_schema.columns
  where table_schema='public' and table_name='bookings'
  order by ordinal_position;
  ```
  Expect EXACTLY: `id`, `customer_id`, `service_id`, `status`, `price` (NOT NULL), `paid_at` (nullable), `payout_id` (nullable), `created_at`, `updated_at` — and **NO `start_slot_id` column** (the booking's slots, *including the first*, live only in `booking_slots`; the start time is derived as `MIN(starts_at)` over them via the `bookings_with_start` view), **NO `slot_id` column**, **NO `barber_id` column** (the barber is reached via `bookings → services → barbers`, not stored on the booking), and **NO `platform_fee` / `barber_amount`** (the split is derived at settlement, M2.2). Also confirm the `status` CHECK is the **3-state** set `('pending_payment','paid','cancelled')` — **no `payout_pending` / `payout_transferred`** (settlement state is derived from `payout_id`, not a status). *Recovery:* re-apply the M1.2 Step 1 migration — if `bookings.start_slot_id`, `bookings.slot_id`, `bookings.barber_id`, the money-split columns, or `payout_*` statuses are present, the old denormalized model leaked in; drop them.

- **A1a** The `bookings_with_start` view exposes the **derived** start/end time (since there is no stored `start_slot_id`):
  ```sql
  select column_name from information_schema.columns
  where table_schema='public' and table_name='bookings_with_start' order by ordinal_position;
  ```
  Expect the booking's own columns **plus** `starts_at` and `ends_at` (the `MIN(starts_at)` / `MAX(ends_at)` over the booking's `booking_slots`). *Recovery:* M1.2 Step 1 — create `bookings_with_start` (the view `/bookings` and any sort-by-start read uses instead of a stored column).

- **A1b** The `booking_slots` join table exists with its **`UNIQUE(slot_id)`** no-double-book guard, and a booking holds **N rows** (N = service.required_slots):
  ```sql
  -- the join table + its columns:
  select column_name from information_schema.columns
  where table_schema='public' and table_name='booking_slots' order by ordinal_position;
  -- the no-double-book UNIQUE index on slot_id (the entire guard — NO partial index on bookings):
  select indexname, indexdef from pg_indexes
  where schemaname='public' and tablename='booking_slots'
    and indexdef ilike '%slot_id%' and indexdef ilike '%unique%';
  -- the most-recent booking holds exactly required_slots booking_slots rows:
  select b.id, b.status, sv.required_slots,
         (select count(*) from public.booking_slots bs where bs.booking_id = b.id) as held_slots
  from public.bookings b
  join public.services sv on sv.id = b.service_id
  order by b.created_at desc limit 1;
  ```
  Expect: `booking_slots(booking_id, slot_id)`, a `create unique index ... on public.booking_slots (slot_id)` row, and `held_slots = required_slots` for the latest booking. *Recovery:* M1.2 Step 1 (create `booking_slots` + the `uniq_slot_held` UNIQUE index; the `create_booking` RPC inserts the N rows). There is **no** partial unique index on `bookings` — the guard lives on `booking_slots`.

- **A2** RLS is ON and the own-rows policies exist (on **both** tables):
  ```sql
  select tablename, policyname, cmd from pg_policies where tablename in ('bookings','booking_slots');
  ```
  Expect, on `bookings`: select/insert/update scoped to `auth.uid() = customer_id`, plus a read-only shop select that **joins through the service** (`exists (select 1 from services sv join barbers b on b.id = sv.barber_id where sv.id = bookings.service_id and b.shop_id = auth.uid())`) — since bookings has no `barber_id`, the shop is reached via the service. On `booking_slots`: a public-select policy (`using (true)`, so the anti-join can read it) + a write policy scoped to the owning booking's customer. *Recovery:* M1.2 Step 1.

- **A3** No advisor warnings on `bookings` / `booking_slots` — run the Supabase MCP `get_advisors` (security) and confirm no "RLS disabled" / "policy missing" on either table. *Recovery:* fix the policy, re-run.

#### Section B — Buyer pages reachable

- **B1** `/barbers` browse page returns 200 (publicly viewable):
  ```bash
  curl -sS -o /dev/null -w "%{http_code}\n" https://<app>.vercel.app/barbers
  ```
- **B2** A barber detail page `/barbers/<a-real-barber-uuid>` returns 200 (and does NOT collide with `/shop/bookings`):
  ```bash
  curl -sS -o /dev/null -w "%{http_code}\n" https://<app>.vercel.app/barbers/<barber-id>
  curl -sS -o /dev/null -w "%{http_code}\n" https://<app>.vercel.app/shop/bookings
  ```
  Both 200 (or `/shop/bookings` redirects to sign-in). A 404 on the `[id]` route means the dynamic route didn't ship. *Recovery:* M1.2 Step 4.
- **B3** `/bookings` returns 200 or redirects to `/login` (auth-gated):
  ```bash
  curl -sS -o /dev/null -w "%{http_code}\n" https://<app>.vercel.app/bookings
  ```
- **B4** **Layout + photos (browser check):** `/barbers` is a **marketplace-style card grid** where each card shows a rep photo; `/barbers/<barber-id>` leads with a **photo carousel** of the barber's `barber_photos` (main image + prev/next + thumbnail/dot nav, click-to-zoom) — like a marketplace product-detail page. *Recovery:* M1.2 Step 3/4 (the layout + the gallery read from `barber_photos`). If the barber uploaded no photos, a placeholder is fine.

#### Section C — The Book dialog + the booking write (the decisive test)

Do this **in a browser** as **Customer A** on the live site:

- **C1** On `/barbers/<barber-id>`, the **available slots** list shows only slots with **no `booking_slots` row** (the availability anti-join `not exists (select 1 from booking_slots bs where bs.slot_id = s.id)` excludes already-held ones), and there is a **Book** button.
- **C2** **Book opens a pop-up DIALOG** — clicking Book opens a modal to pick **service → date → available slot**, the **price (from `service.price`) is shown before Confirm**, and the URL **stays on `/barbers/<barber-id>`** (no full-page navigation).
- **C3** **Confirm closes the dialog and returns to `/barbers/<barber-id>`** with a success toast — still the same page, dialog gone.
- **C4** **A `pending_payment` booking row was created with NO `start_slot_id` (start DERIVED from `booking_slots`; no `slot_id`, no `barber_id`) + N `booking_slots` rows, and the derived start slot is now excluded by the anti-join:**
  ```sql
  -- via Supabase MCP execute_sql (most-recent booking, joined to its DERIVED start via the view):
  select b.id, b.status, b.service_id, b.price, b.paid_at, b.payout_id,
         v.starts_at,                                  -- DERIVED start = MIN(starts_at) over booking_slots (no start_slot_id column)
         sv.required_slots,
         -- the booking holds N booking_slots rows (N = required_slots):
         (select count(*) from public.booking_slots bs where bs.booking_id = b.id) as held_slots,
         -- derive the barber THROUGH the service (bookings has no barber_id):
         sv.barber_id,
         -- the booking's slots are no longer bookable because booking_slots rows reference them:
         not exists (
           select 1 from public.booking_slots bx
           join public.booking_slots own on own.slot_id = bx.slot_id
           where own.booking_id = b.id
         ) as slot_still_bookable
  from public.bookings b
  join public.bookings_with_start v on v.id = b.id
  join public.services sv on sv.id = b.service_id
  order by b.created_at desc limit 1;
  ```
  Expect: booking `status='pending_payment'`, **no `start_slot_id` column** (the start is read from `bookings_with_start.starts_at`, derived as `MIN(starts_at)` over the booking's `booking_slots`), `price` populated (a sensible whole-unit number, NOT ×100), `paid_at` **NULL** + `payout_id` **NULL** (M1.2 is pre-payment), `held_slots = required_slots`, the `barber_id` derived **via the service join** (not a column on bookings), and `slot_still_bookable = false` (the `booking_slots` rows remove the held slots from availability). *Recovery:* M1.2 Step 2 (the `create_booking` RPC — takes `p_start_slot_id` as an argument but does NOT store it; inserts the booking + N `booking_slots` rows, no slot status flip because slots have no status).
- **C4b** **Double-book is rejected by the `UNIQUE(slot_id)` on `booking_slots`** — manually try to insert a SECOND `booking_slots` row on the same `slot_id`; the `uniq_slot_held` index must reject it:
  ```sql
  -- via Supabase MCP execute_sql — reuse a slot_id already held by the C4 booking.
  -- First make a throw-away booking to own the row, then try to grab an already-held slot.
  -- This MUST fail with a unique-violation (duplicate key value violates unique constraint "uniq_slot_held"):
  insert into public.booking_slots (booking_id, slot_id)
  select (select id from public.bookings order by created_at desc limit 1),  -- any existing booking
         bs.slot_id                                                          -- a slot already held
  from public.booking_slots bs
  order by bs.slot_id limit 1;
  ```
  Expect an **error** (unique violation on `uniq_slot_held`), NOT a second row. *Recovery:* M1.2 Step 1 — add the `create unique index uniq_slot_held on public.booking_slots(slot_id)`. (Note: the guard is on `booking_slots`, NOT a partial index on `bookings`.)
- **C5** **Slot no longer offered** — reload `/barbers/<barber-id>`; the just-booked slot is gone from the available list, because the anti-join excludes any slot that has a `booking_slots` row. *Recovery:* the available list must use the `not exists` anti-join against `booking_slots` (M1.2 Step 4).

#### Section D — My-bookings + RLS isolation

- **D1** As **Customer A**, `/bookings` lists the booking just made (store/service/time/price + `status='pending_payment'`).
- **D2** No `paid` rows exist yet, and no `paid_at` is stamped (M1.2 is pre-payment):
  ```sql
  select
    count(*) filter (where status='paid')        as paid_count,
    count(*) filter (where paid_at is not null)  as paid_at_count
  from public.bookings;
  ```
  Expect both `0`. A non-zero count means payment logic leaked in early (that's M2.1 — the webhook is the ONLY thing that sets `paid` / `paid_at`). *Recovery:* remove any premature `paid` writes / `paid_at` stamps; M1.2 bookings stay `pending_payment` with `paid_at=NULL`.
- **D3** **RLS denies cross-customer reads (the key one):** sign in as **Customer B** and open `/bookings` — Customer B does **NOT** see Customer A's booking. Confirm at the DB level that the policy (not just the UI) enforces it:
  ```sql
  -- as Customer B's JWT (via the app), a select returns only B's rows.
  -- Sanity check with service-role that A's row exists but is owned by A:
  select customer_id, status from public.bookings order by created_at desc limit 5;
  ```
  *Recovery:* fix the `bookings_select_own` policy (M1.2 Step 1) — never rely on front-end filtering alone.
- **D4** **Cancelling a booking frees its slots (the free-on-cancel trigger DELETEs its `booking_slots` rows).** First capture the booking's held slot ids **before** cancelling (there is no `start_slot_id` column to read afterwards — and the trigger deletes the `booking_slots` rows), then set it `cancelled`, then confirm those slots become bookable again — the anti-join no longer excludes them because their `booking_slots` rows are gone:
  ```sql
  -- via Supabase MCP execute_sql (or the customer's own update if a cancel UI exists):
  -- 1) capture the held slot ids BEFORE cancelling (they vanish from booking_slots after):
  select array_agg(slot_id) as held_slot_ids
  from public.booking_slots where booking_id = '<the_C4_booking_id>';
  -- 2) cancel — the free-on-cancel trigger DELETEs the booking's booking_slots rows:
  update public.bookings set status='cancelled' where id = '<the_C4_booking_id>';
  -- 3) those slots are bookable again (no booking_slots row references them):
  select s.id,
         not exists (
           select 1 from public.booking_slots bs where bs.slot_id = s.id
         ) as slot_bookable
  from public.bookable_slots s
  where s.id = any('<held_slot_ids from step 1>'::uuid[]);
  ```
  Expect `slot_bookable = true` for every formerly-held slot after the cancel (and they reappear in the `/barbers/<barber-id>` available list). This proves availability is **derived** from `booking_slots` rows, not stored on the slot or the booking. *Recovery:* the free-on-cancel trigger must DELETE the booking's `booking_slots` rows, and the available list must use the `booking_slots` anti-join (M1.2 Step 1/4).

## Reporting

Emit a table:

| Check | Status | Notes |
|---|---|---|
| A1 `bookings` columns EXACTLY id/customer_id/service_id/status(3-state)/price/paid_at/payout_id/created_at/updated_at — NO `start_slot_id`/`slot_id`/`barber_id`/fee cols, NO `payout_*` status | ✅ / ❌ | barber reached via service→barbers; start derived |
| A1a `bookings_with_start` view exposes derived `starts_at`/`ends_at` | ✅ / ❌ | MIN/MAX over booking_slots |
| A1b `booking_slots` join table + `UNIQUE(slot_id)` guard; booking holds N rows | ✅ / ❌ | N = service.required_slots |
| A2 RLS own-rows policies present (shop read joins through service; booking_slots public-select) | ✅ / ❌ | |
| A3 no advisor warnings on `bookings`/`booking_slots` | ✅ / ⚠️ / ❌ | |
| B1 `/barbers` browse 200 | ✅ / ❌ | |
| B2 `/barbers/[id]` 200 + no collision w/ `/shop/bookings` | ✅ / ❌ | uuid/int id route |
| B3 `/bookings` 200 / redirects to login | ✅ / ❌ | |
| B4 grid + detail photo carousel (marketplace layout) | ✅ / ⚠️ / ❌ | `barber_photos`; placeholder ok if none |
| C1 available slots use `booking_slots` anti-join + Book button | ✅ / ❌ | |
| C2 Book opens a DIALOG (service→date→slot, price shown), stays on `/barbers/[id]` | ✅ / ❌ | pop-up, no navigation |
| C3 Confirm closes dialog + returns to page w/ toast | ✅ / ❌ | |
| C4 `pending_payment` booking created w/ NO `start_slot_id` (start derived) + N `booking_slots`; slots no longer offered | ✅ / ❌ | the decisive one; barber via service join; start via `bookings_with_start`; `paid_at`+`payout_id` NULL |
| C4b 2nd hold on same slot rejected by `UNIQUE(slot_id)` on `booking_slots` | ✅ / ❌ | unique violation expected |
| C5 booked slot no longer offered | ✅ / ❌ | |
| D1 customer sees own booking on `/bookings` | ✅ / ❌ | status `pending_payment` |
| D2 no `paid` rows / no `paid_at` yet (pre-payment) | ✅ / ⚠️ / ❌ | M2.1 owns `paid`/`paid_at` |
| D3 RLS denies reading another customer's bookings | ✅ / ❌ | the key one |
| D4 cancelling a booking frees its slots (deletes `booking_slots` rows) | ✅ / ❌ | availability is derived, not stored |

**Verdict:**
- All ✅ → 「M1.2 驗收通過 ✅ READY for M2.1。客人已經能瀏覽、開理髮師詳細頁、用 pop-up dialog 預約，並建立了 `pending_payment` 預約（N 筆 `booking_slots`、bookings 上**沒有 `start_slot_id`／`slot_id`／`barber_id`**，開始時間是從 `booking_slots` 的 `MIN(starts_at)` 推導出來、用 `bookings_with_start` view 讀；`status` 是 `pending_payment | paid | cancelled` 三態、`paid_at` 與 `payout_id` 皆為 NULL——這就是 M1.2 的完整成果，還沒收錢）。時段沒有 status 欄位——可預約與否是用 `booking_slots` 的 anti-join 推導出來的，所以一有預約就自動從可預約清單消失、取消後刪掉 `booking_slots` 又自動釋出；同一個時段的第二筆 hold 會被 `booking_slots` 上的 `UNIQUE(slot_id)`（`uniq_slot_held`）擋掉。RLS 也擋住了別人的預約。跟我說『啟動 M2.1』，我們來接 Stripe：把那顆 Confirm 改成開 Stripe Checkout，付款成功後 webhook 把預約從 `pending_payment` 變 `paid`、蓋上 `paid_at`、誰先付款誰贏得時段；平台 20%／理髮師 80% 拆帳則是月底（M2.2）才從 `paid` 預約推導出來，結算狀態是看 `payout_id`、不是再多一個 booking status。」
- Any ❌ → list the failed items + the recovery step (the Step number in `m1.2-buyer-setup`), and tell the student to fix then re-run `驗收 M1.2`.
