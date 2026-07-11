---
name: m2.2-admin-to-seller-payment-checklist
description: 抽成制理髮師預約平台 Milestone 2.2 verification — checks the commission-settlement workflow is real and correctly wired: the real-time `owed_bookings` VIEW math (price × rate = platform_cut + shop_cut, sums back) per paid booking, attributed per SHOP via `bookings → services → barbers` (since bookings has no barber_id and never goes through the slot), ONLY role='admin' reaches /admin/payouts (a non-admin is denied — the decisive test), building a payout flips the selected bookings' payout_id + creates a pending_transfer payout with the snapshot, the same-shop guard rejects a mixed-shop selection, no-double-pay (a booking already in a payout can't be re-grabbed), the row-by-row "Mark as transferred" flips a payout to transferred (its bookings unchanged — settled is derived from payout_id), CANCEL nulls the bookings' payout_id + sets the payout cancelled (booking status stays paid), a transferred payout can't be cancelled, the shop sees the status on /shop/earnings, payouts carries the shop_name snapshot, there is NO transactions table and NO fee columns on bookings, and the shop-level bank fields (on profiles) are not readable by a non-shop/non-admin (and barbers carries no bank columns). Use when the student says "驗收 M2.2", "check M2.2", "M2.2 done?", "撥款頁對不對", or after the `m2.2-admin-to-seller-payment` skill completes Step 4.
---

# M2.2 — 抽成撥款 + Admin 撥款頁 Checklist

## What this skill does

Verifies the student actually completed M2.2 — the commission-settlement workflow — not just *thinks* they did. The high-risk items here are **money math** (does price × rate split into `platform_cut` + `shop_cut` that sum back to price, with no double-rounding?), **the flexible payout model** (does building a payout stamp `bookings.payout_id` + create a `pending_transfer` snapshot for ONE shop, does the same-shop guard hold, is double-pay impossible, does cancel revert cleanly?), and **access control** (can a non-admin reach the payout page or read bank details?). This skill tests every artifact and reports pass/fail per item, then emits a `READY for M3` verdict.

Three model facts the checks enforce throughout:
- there is **NO `transactions` table** (the VIEW lists `paid` bookings directly);
- **bookings carry no money-split columns** (no `platform_fee`/`barber_amount` — the split lives in the `owed_bookings` VIEW and is snapshotted onto `payouts`);
- **settlement is DERIVED from `bookings.payout_id`**, not a booking status — bookings have **3 states only** (`pending_payment`/`paid`/`cancelled`), and there is **no** `payout_pending`/`payout_transferred` booking status.

**Run this AFTER `m2.2-admin-to-seller-payment` Step 4, or any time the student claims M2.2 is done.**

## Execution mode: Cowork vs CLI (read this first)

| Section | CLI mode tool | Cowork mode equivalent |
|---|---|---|
| A — owed_bookings VIEW math (price × rate split + rounding) | Supabase SQL editor | **Supabase MCP `execute_sql`** (preferred both modes) |
| B — Builder running total reconciles | Supabase MCP `execute_sql` + the live `/admin/payouts` page | same |
| C — Admin gate (only `role='admin'`) | `curl` + two browser logins | Supabase MCP + browser / Playwright MCP |
| D — Build / mark-transferred / cancel state machine | Supabase MCP `execute_sql` + the live pages | same |
| E — Bank-field RLS + no-transactions / no-fee-columns | Supabase MCP `execute_sql` as different roles | same |

In Cowork mode every Bash/`curl` block below is CLI-only — use the browser/MCP equivalent. Supabase MCP is preferred for the SQL checks in both modes. There is **no Stripe check in this milestone** (M2.2 moves no money). **Verifying a live URL in Cowork ([[supabase-best-practice]] Rule 7):** the sandbox `curl` is **proxy-blocked** — a `000/403` is **NOT** proof the site is down. Use the **URL-fetch MCP** (`web_fetch_vercel_url`) or a browser. The `/admin/payouts` gate check (only `role='admin'` reaches it) is a real per-path test — the fetch tool can't shareable-URL a subpath, so confirm the **route guard + RLS** structurally and do the decisive allow/deny test by **navigating in a browser** as admin vs customer.

> **⚠️ CRITICAL — the Supabase MCP runs PRIVILEGED, so it CANNOT exercise the admin-guarded RPCs or the RLS denials directly. Read this before running Sections C, D, E.** The Supabase MCP `execute_sql` connects as a **privileged role with `auth.uid()` = NULL** and it **bypasses RLS**. Two consequences that change how you verify:
> - **Admin-guarded RPCs raise on the MCP.** `select public.build_payout(array[...])` / `cancel_payout(...)` / `mark_payout_transferred(...)` all hit `if not public.is_admin() then raise 'admin only'` at line 1 (because `auth.uid()` is NULL → not admin). So through the MCP you **cannot** exercise the happy path *or* reach the same-shop / no-double-pay / immutability guards — every call dies with `admin only` before the logic runs. This is **not** a bug in the build; it's the privileged session.
> - **RLS denials can't be shown on the MCP.** "As a customer, expect 0 rows" (C3/E1) can't be demonstrated — the privileged session sees **every** row regardless of policy.
>
> **So split every behavioral test into two evidence types:**
> 1. **Behavioral (the real proof) → do it in the BROWSER / live app**, logged in as the actual admin (and as a customer for the deny tests). The admin building a real payout and marking it transferred on the live page *is* the D1/D4 pass; a customer being redirected off `/admin/payouts` *is* the C2 pass. This is the **primary** path for C2, C3, D1, D2, D3, D4, D5, D6, E1.
> 2. **Structural (the MCP-side evidence)** — where you can't drive the browser, confirm the guards *exist in the source* instead of executing them: read the function bodies with **`select pg_get_functiondef(oid) from pg_proc where proname in ('build_payout','cancel_payout','mark_payout_transferred','is_admin')`** and eyeball the guards (`if not public.is_admin()`, `count(distinct ... shop_id) <> 1`, `where ... payout_id is null`, `where ... status='pending_transfer'`), and confirm the **policies exist** with **`select policyname, cmd, qual from pg_policies where tablename in ('payouts','bookings')`**. Structural checks prove the guard is *written*; the browser proves it *fires*. A section passes on the browser evidence — the `pg_get_functiondef`/`pg_policies` reads are the fallback when a live login isn't available, not a substitute for it.

## How to run

The student invokes this directly (e.g. types `驗收 M2.2`). You (Claude Code) **actively run** each check and report results — don't just describe them.

### Step 1: Collect (one message)

Ask the student for:
1. The **live Vercel URL** (`https://<app>.vercel.app`).
2. The **admin account** email (promoted in the M2.1 prereq) and a **non-admin** account (a customer or shop) to test the gate.
3. A **shop that has `paid` bookings** (so the owed pool shows real numbers), ideally one running **multiple barbers** so the rollup is visible.

If there is **no admin user**, stop — send the student back to the M2.1 prerequisite to promote one (there is no admin sign-up). Then re-run.

> **Confirm the M2.2 deploy actually shipped (Vercel MCP — 2026):** if the **Vercel MCP is connected, it *does* see the project** — `list_deployments` / `get_deployment` by project + team id and check the latest is `state: READY` (a fresh push is usually `READY` in ~30–40s) before you trust the live pages. The old "the Vercel connector shows no projects, so the build can't be verified" note is **stale** — don't repeat it. (The sandbox `curl` against the live URL is still proxy-blocked; use the Vercel MCP or the URL-fetch MCP.)

### Step 2: Run the checklist

#### Section A — Real-time owed-pool VIEW math (price × rate split + rounding)
- **A1** The `owed_bookings` VIEW exists and returns the live owed pool — one row per **`paid` booking with `payout_id IS NULL`**, attributed to a shop via `bookings → services → barbers(shop_id)` — through `service_id`, **NOT** through the slot, and **NOT** via a non-existent `bookings.barber_id`:
  ```sql
  -- via Supabase MCP execute_sql
  select booking_id, shop_id, shop_name, barber_name, customer_id, price, platform_pct, platform_cut, shop_cut
  from public.owed_bookings
  order by paid_at desc;
  ```
  The VIEW should expose **`shop_name`** and **`barber_name`** directly (resolved inside the view), so the builder needs no second `profiles` fetch and no unreliable PostgREST embed on the view. Confirm the view definition attributes via the service (not the slot, not a non-existent `bookings.barber_id`) and filters the owed pool:
  ```sql
  select pg_get_viewdef('public.owed_bookings'::regclass, true);
  ```
  Expect `join public.services ... on ... = b.service_id` then `join public.barbers ... on ... = s.barber_id` (and a `join public.profiles ... on ... = bar.shop_id` for `shop_name`), a `where b.status = 'paid' and b.payout_id is null`, and a lateral select from `commission_rates` for the rate. *Recovery:* if the view references `bookings.barber_id`, a `transactions` table, joins through `bookable_slots`, or filters on a `payout_pending`/`payout_transferred` booking status, it's on the old model — rewrite it to list `paid` bookings WHERE `payout_id IS NULL`, joined `bookings → services → barbers`, with the derived split (M2.2 Step 1).
- **A2** **The split is correct and sums back to price, per row** — `platform_cut + shop_cut = price`, with no lost unit (rounding closes exactly because `shop_cut = price - platform_cut`), and `platform_cut = round(price * platform_pct)`:
  ```sql
  select booking_id, price, platform_pct, platform_cut, shop_cut,
         (platform_cut + shop_cut) as recomputed_price,
         (price - (platform_cut + shop_cut)) as gap
  from public.owed_bookings
  order by paid_at desc;
  ```
  Expect `gap = 0` on **every** row, and `platform_cut = round(price * platform_pct)`. *Recovery:* the VIEW must compute `platform_cut = round(price * rate)` and `shop_cut = price - platform_cut` (M2.2 Step 1). There is no per-booking fee column to sum — bookings carry no split columns; if you find the VIEW reading `platform_fee`/`barber_amount`, it's on the old model.
- **A3** **The VIEW is `security_invoker` AND `bookings` has an admin SELECT policy** — the two facts that make the owed pool visible to the admin *and* isolated per shop. Both are silent when wrong (empty admin builder / cross-shop leak), so verify them structurally:
  ```sql
  -- (1) owed_bookings must be security_invoker=true (else it BYPASSES bookings RLS)
  select c.relname, c.reloptions
  from pg_class c join pg_namespace n on n.oid=c.relnamespace
  where n.nspname='public' and c.relname='owed_bookings';
  -- (2) bookings must have an admin SELECT policy (M2.2 adds it — it did NOT exist before)
  select policyname, cmd from pg_policies
  where schemaname='public' and tablename='bookings' order by policyname;
  ```
  Expect (1) `reloptions` containing **`security_invoker=true`** (or `=on`), and (2) a `bookings` SELECT policy using `public.is_admin()` (e.g. `bookings_select_admin`) alongside the existing `bookings_select_own` / `bookings_select_shop_owner`. Then run **`get_advisors` (security)** and confirm **no `security_definer_view` ERROR** on `owed_bookings` (the WARNINGs about `anon`/`authenticated` executing the RPCs are **expected/safe** — the `is_admin()` guard is the control). *Recovery:* recreate the view `with (security_invoker = true)` and add `create policy "bookings_select_admin" on public.bookings for select using (public.is_admin());` (M2.2 Step 1). Without (1) a shop sees every shop's owed rows; without (2) the admin sees zero.

#### Section B — Builder running total reconciles
- **B1** On `/admin/payouts`, the owed-list **running total** of the checked rows equals the sum of those rows' columns. Spot-check against SQL for a given shop's owed pool:
  ```sql
  select sum(gross)        as total_gross,
         sum(platform_cut) as total_platform,
         sum(shop_cut)     as total_shop_cut
  from (select price as gross, platform_cut, shop_cut
        from public.owed_bookings where shop_id = '<a shop id>') t;
  ```
  Compare these to the builder's running total when all of that shop's owed rows are checked — they must match exactly, and `total_gross = total_platform + total_shop_cut`. *Recovery:* compute the builder total by summing the **checked rows**, not a separate divergent query (M2.2 Step 2).

#### Section C — Admin gate (the decisive test)
- **C1** **As the admin:** logging in **lands you on `/admin/payouts`** (the restored post-login redirect — not `/barbers`) **and** the admin **Payouts nav link** is visible; the page shows the live owed list (one row per owed paid booking, attributable per shop) + the existing-payouts ledger, plus the 「建立撥款 / Build payout」 action. *Recovery:* if the admin lands on `/barbers`, the M2.1-era login redirect stub (`admin → /barbers`) wasn't restored to `admin → /admin/payouts` — fix `redirectByRole` (e.g. `src/pages/Login.tsx`) and/or add the admin nav link (M2.2 Step 2, entry-point).
- **C2** **The decisive test — as a non-admin** (logged-in customer or shop): hitting `/admin/payouts` is **denied** (redirect to `/login` / 403, NOT the payout page):
  ```bash
  # signed-out / non-admin should NOT get the payout page
  curl -sS -o /dev/null -w "%{http_code}\n" https://<app>.vercel.app/admin/payouts
  ```
  A signed-out request must not return the page; for a logged-in non-admin, confirm in the browser they are redirected/403'd. **This is a BROWSER test — the Supabase MCP can't stand in for it** (it's privileged, so it can't be "a non-admin"). *Recovery:* add the route guard (`/admin/*` middleware, or the `RequireAdmin` component in the Vite SPA) AND rely on RLS (`payouts` + bank fields are admin-or-shop only; the admin actions are admin-guarded RPCs) — M2.2 Step 2.
- **C3** **RLS-level proof:** a non-admin session cannot read other shops' `payouts` at all:
  ```sql
  -- run as a non-admin (anon/customer) session, NOT service-role
  select count(*) from public.payouts;
  ```
  Expect `0` rows (a customer owns none; a shop sees only its own) — even if rows exist. **⚠️ You CANNOT run this through the Supabase MCP** — the MCP bypasses RLS and would see every row, falsely failing the check. Run it from a real **non-admin browser session** (the app's Supabase client with a customer logged in), or fall back to the **structural** proof: `select policyname, cmd, qual from pg_policies where tablename='payouts'` shows the `payouts_select_admin_or_owner` policy (`is_admin() OR shop_id = auth.uid()`) exists. The admin/service-role sees the real count.

#### Section D — Build / mark-transferred / cancel state machine

> **⚠️ Drive every `build_payout` / `mark_payout_transferred` / `cancel_payout` call from the LIVE APP as the logged-in admin — NOT from the Supabase MCP.** On the MCP `auth.uid()` is NULL, so every one of these RPCs raises `admin only` at line 1 and you never reach the happy path or the guards below. The **`select ... from payouts / bookings / owed_bookings`** read-back queries (which just inspect state) are fine on the MCP; only the **RPC-invoking** lines (`select public.build_payout(...)`, etc.) must be run through the app. If you can't drive the browser, verify the guards **structurally** with `pg_get_functiondef` instead (see the CRITICAL note in Execution mode) and record the item as structurally-confirmed. The real behavioral pass came from the user's own live build+mark-transferred on a real shop.

- **D1** **BUILD A PAYOUT stamps `bookings.payout_id` + creates a `pending_transfer` snapshot for ONE shop.** As the admin **in the live app**, select a shop's owed bookings → 「建立撥款 / Build payout」 (calling `build_payout(array[...])` directly only works from an admin session, not the MCP), then confirm with the read-back below:
  ```sql
  select id, shop_id, shop_name, status, gross, platform_pct, platform_cut, shop_cut,
         bookings_count, created_by
  from public.payouts
  order by created_at desc;
  ```
  Expect a new `status='pending_transfer'` row for that shop, with a non-null **`shop_name`** (the `profiles.display_name` snapshot) and snapshotted `gross`/`platform_pct`/`platform_cut`/`shop_cut`/`bookings_count`, where `platform_cut + shop_cut = gross`. Then confirm the chosen bookings now carry the link and have **left** the owed pool:
  ```sql
  select count(*) from public.bookings where payout_id is not null;       -- the batched bookings
  select count(*) from public.owed_bookings where shop_id = '<that shop>'; -- those bookings are gone from owed
  ```
  The batched bookings carry `payout_id`; their booking `status` is still **`paid`** (NOT a payout status). *Recovery:* `build_payout` must INSERT a `pending_transfer` payout (snapshots + `shop_name`) AND UPDATE the selected bookings' `payout_id`, atomically (M2.2 Step 1/3 RPC).
  > **If the Build-payout call errors with `function min(uuid) does not exist`:** the build skill's Step 1 SQL still has the `min(bar.shop_id)` bug — Postgres has no `min(uuid)` aggregate, so `build_payout` fails at run time on the *first* call (the migration DDL compiled, so it looked fine). Fix `build_payout` to pick the shop with **`(array_agg(distinct bar.shop_id))[1]`** instead of `min(bar.shop_id)`, re-apply the migration, and re-run D1 (M2.2 build skill fix — Step 1 RPC).
  > **If D1 shows the owed list was EMPTY for the admin** (nothing to select / build): the admin can't read `bookings`. Confirm `owed_bookings` is `security_invoker = true` **and** a `bookings_select_admin` (`for select using (public.is_admin())`) policy exists — without both, the admin sees zero owed rows (Section C / M2.2 Step 1).
- **D2** **Same-shop guard.** Building a payout from a selection spanning **two different shops** must be **rejected** (not silently split):
  ```sql
  -- pick one owed booking from shop A and one from shop B, then:
  select public.build_payout(array['<bookingA>','<bookingB>']::uuid[]);
  ```
  Expect an **error** ("all selected bookings must belong to the SAME shop") and **no** payout row created. *Recovery:* `build_payout` must `count(distinct shop_id)` over the selection and raise if `<> 1` (M2.2 Step 1).
- **D3** **No double-pay.** A booking already in a payout cannot be grabbed into a second one. Try building a payout that includes an already-batched booking:
  ```sql
  -- include a booking whose payout_id is already set:
  select public.build_payout(array['<already-batched booking>']::uuid[]);
  ```
  Expect either an error ("no owed bookings in selection") or that the booking is **not** re-stamped (its `payout_id` is unchanged — at most one payout per booking). Confirm no booking has two payouts:
  ```sql
  select count(*) from public.bookings where payout_id is not null group by id having count(*) > 1;
  ```
  Expect **0 rows** (the single-valued `payout_id` FK guarantees it). *Recovery:* `build_payout`'s booking UPDATE must guard `WHERE status='paid' AND payout_id IS NULL` (M2.2 Step 1).
- **D4** **MARK TRANSFERRED flips the payout — bookings UNCHANGED.** Click 「標記為已轉帳 / Mark as transferred」 on one `pending_transfer` payout as the admin (or call `mark_payout_transferred`), then confirm it persisted:
  ```sql
  select id, shop_name, status, marked_transferred_at, transferred_by, bank_reference
  from public.payouts
  where status = 'transferred'
  order by marked_transferred_at desc
  limit 5;
  ```
  Expect a row with `status='transferred'`, a `marked_transferred_at` timestamp, and `transferred_by` = the admin's profile id. Then confirm that payout's bookings were **NOT** touched — settled is derived from `payout_id`:
  ```sql
  select status, count(*)
  from public.bookings
  where payout_id = '<the transferred payout id>'
  group by status;
  ```
  Expect them all still at **`paid`** (there is no `payout_transferred` booking status). *Recovery:* `mark_payout_transferred` must update **only** the payout `pending_transfer → transferred` + stamps — it must NOT change any booking status (M2.2 Step 3).
- **D5** **A transferred payout is IMMUTABLE.** Trying to cancel a `transferred` payout must be rejected:
  ```sql
  select public.cancel_payout('<a transferred payout id>');
  ```
  Expect an **error** ("not pending_transfer / immutable") and the payout unchanged (still `transferred`, its bookings still linked). *Recovery:* `cancel_payout`'s `where ... and status='pending_transfer'` guard must reject a `transferred` payout (M2.2 Step 3).
- **D6** **CANCEL nulls the bookings' `payout_id` + sets the payout `cancelled` (booking status stays `paid`).** Build a fresh `pending_transfer` payout, then cancel it (or call `cancel_payout`), and confirm:
  ```sql
  select id, status from public.payouts where status='cancelled';   -- the row is KEPT (not deleted)
  -- its bookings reverted to owed: payout_id NULL, status still 'paid':
  select status, payout_id is null as is_owed, count(*)
  from public.bookings
  where id = any('<the cancelled batch booking ids>'::uuid[])
  group by status, payout_id is null;
  ```
  Expect the payout row **kept** at `cancelled` (NOT deleted), and its bookings now `payout_id IS NULL` with `status='paid'` (back in `owed_bookings`). Confirm they reappear in the owed pool:
  ```sql
  select count(*) from public.owed_bookings where booking_id = any('<the cancelled batch booking ids>'::uuid[]);
  ```
  Expect them present again. *Recovery:* `cancel_payout` must null the bookings' `payout_id` AND set the payout `cancelled` (keep the row), atomically, only while `pending_transfer` (M2.2 Step 3).
- **D7** **The shop sees it** — log in as a shop → `/shop/earnings` shows their bookings split into 尚未撥款 / Owed (payout_id NULL) vs 已納入撥款 / In a payout (with that payout's status 「待轉帳 / Pending」 / 「已轉帳 / Transferred」 + date). A cancelled payout's bookings appear back under Owed. *Recovery:* `/shop/earnings` must read the shop's own paid bookings (filtered to `shop_id = auth.uid()` via service → barber), joining `payouts` for the in-a-payout status, and exclude `cancelled` payouts (M2.2 Step 4).
- **D8** **Snapshot integrity (a rate change can't alter a built batch).** The `payouts` row froze `platform_pct`/`platform_cut`/`shop_cut`/`shop_name` at build time; the live `owed_bookings` VIEW may differ if the rate later changes, but a built payout does not. Spot-check that a `pending_transfer`/`transferred` payout's snapshot equals what was owed at build (not a recomputed live value). *Recovery:* `build_payout` must snapshot all amounts + `shop_name` onto the payout, never join back to the live VIEW for the settled figure (M2.2 Step 1/3).

#### Section E — Bank-field RLS + no-transactions / no-fee-columns
- **E1** Bank fields are **not** readable by a non-shop / non-admin. The bank columns live on **`profiles`** (shop level). As a **customer** session (or a *different* shop), another user's bank fields must not be exposed:
  ```sql
  -- run as a non-shop non-admin session
  select id, bank_account_name, bank_account_number from public.profiles;
  ```
  Expect only the caller's own row (their own bank fields) — never another shop's `bank_account_*`. **⚠️ Run this from a real non-admin browser session, NOT the Supabase MCP** — the MCP bypasses RLS and sees every row, so it can't demonstrate the denial; its result here proves nothing. Structural fallback: `select policyname, qual from pg_policies where tablename='profiles'` shows the shop+admin-only select policy. The admin sees all (via `profiles_select_admin`). Also confirm `barbers` has **no** bank columns (this schema read *is* fine on the MCP):
  ```sql
  select column_name from information_schema.columns
  where table_schema='public' and table_name='barbers';
  ```
  must NOT list `bank_account_name`/`bank_account_number`. *Recovery:* bank fields belong on `profiles` with shop+admin RLS (M0/M1.1 / [[supabase-best-practice]]); confirm with `get_advisors`.
- **E2** **No `transactions` table; no fee columns on bookings; 3-state bookings + `payout_id`.** Confirm the old money model is gone:
  ```sql
  -- transactions table must NOT exist
  select to_regclass('public.transactions') as transactions_table;   -- expect NULL
  -- bookings must NOT carry split columns; must carry payout_id; must be 3-state
  select column_name from information_schema.columns
  where table_schema='public' and table_name='bookings';
  ```
  `transactions_table` must be `NULL`; the bookings columns must NOT include `platform_fee`/`barber_amount`/`start_slot_id`/`payout_record_id` (they should include `price`, `paid_at`, **`payout_id`**, `status`). Confirm the booking status check is 3-state only (no `payout_pending`/`payout_transferred`):
  ```sql
  select pg_get_constraintdef(oid) from pg_constraint
  where conrelid='public.bookings'::regclass and contype='c' and conname like '%status%';
  ```
  Expect a CHECK over `('pending_payment','paid','cancelled')` only. *Recovery:* drop any `transactions` table and any `platform_fee`/`barber_amount` columns; settlement is derived from `payout_id`, not a `payout_*` booking status (M2.2 Step 1).
- **E3** **`bookings.payout_id` has its foreign key to `payouts`.** M1.2 created `payout_id` as a bare `uuid` (no FK — `payouts` didn't exist yet); M2.2 adds `bookings_payout_id_fkey`. Confirm it landed:
  ```sql
  select conname, confrelid::regclass as references_table, confdeltype
  from pg_constraint
  where conrelid = 'public.bookings'::regclass and contype = 'f' and conname = 'bookings_payout_id_fkey';
  ```
  Expect **one row** — `references_table = payouts`, `confdeltype = 'n'` (`on delete set null`). Zero rows means the FK is missing (the settlement link has no referential integrity — a dangling `payout_id`, no auto-null on payout delete). *Recovery:* add it (guarded, idempotent): `alter table public.bookings add constraint bookings_payout_id_fkey foreign key (payout_id) references public.payouts(id) on delete set null;` — M2.2 Step 1.

## Reporting

Emit a table:

| Check | Status | Notes |
|---|---|---|
| A1 `owed_bookings` VIEW returns the live owed pool (paid + payout_id NULL, shop rollup via service) | ✅ / ❌ | no transactions; no slot/barber_id path |
| A2 split sums back to price (gap=0), platform_cut = round(price×rate) | ✅ / ⚠️ / ❌ | shop_cut = price − platform_cut |
| A3 view is `security_invoker` + `bookings` has admin SELECT; no advisor ERROR | ✅ / ❌ | else silent: empty admin builder / cross-shop leak |
| B1 builder running total reconciles with checked rows | ✅ / ❌ | total = sum of checked rows |
| C1 admin login lands on /admin/payouts (redirect restored) + nav link; owed list + ledger + build | ✅ / ❌ | not stranded on /barbers |
| C2 non-admin denied at /admin/payouts | ✅ / ❌ | **the decisive test** |
| C3 non-admin cannot read others' payouts (RLS) | ✅ / ❌ | |
| D1 build payout → pending_transfer snapshot (shop_name) + bookings.payout_id stamped | ✅ / ❌ | bookings stay 'paid' |
| D2 same-shop guard rejects a mixed-shop selection | ✅ / ❌ | count(distinct shop)=1 |
| D3 no double-pay — already-batched booking can't be re-grabbed | ✅ / ❌ | payout_id IS NULL guard |
| D4 mark-transferred → payout transferred; bookings UNCHANGED | ✅ / ❌ | settled derived from payout_id |
| D5 a transferred payout is immutable (cancel rejected) | ✅ / ❌ | |
| D6 cancel → bookings.payout_id NULL (owed again, still paid) + payout cancelled (row kept) | ✅ / ❌ | not deleted |
| D7 shop sees owed vs in-a-payout + status on /shop/earnings | ✅ / ❌ | read-only mirror |
| D8 snapshot integrity — built batch unaffected by a rate change | ✅ / ❌ | frozen on payouts |
| E1 bank fields (on profiles) not readable by others; barbers has none | ✅ / ❌ | |
| E2 NO transactions table; NO fee cols; 3-state bookings + payout_id | ✅ / ❌ | old money model gone |
| E3 `bookings.payout_id` FK → payouts exists (on delete set null) | ✅ / ❌ | M1.2 left it bare; M2.2 adds it |

**Verdict:**
- All ✅ → 「M2.2 驗收通過 ✅ — 抽成撥款制度正確：`owed_bookings` VIEW 列出每筆還沒撥款的 paid booking（`payout_id IS NULL`）並乘上費率（platform_cut + shop_cut 對得起 price）、按店家經 service→barber 歸戶；只有 admin 進得了撥款頁；『建立撥款』把選取的（同一間店家）booking 蓋上 `payout_id` 並開出 pending_transfer 的快照 payout、混店家的選取會被擋下、已撥款的 booking 無法被重複撥；『標記已轉帳』只翻 payout（booking 不動，撥款狀態是看 `payout_id` 推導）；轉帳前可『取消』、把 booking 的 `payout_id` 清空退回欠款池（booking 仍是 paid、payout 列保留為 cancelled）、已轉帳的 payout 不可取消；店家在 `/shop/earnings` 看得到欠款 vs 已納入撥款及狀態；payouts 有 shop_name 快照、沒有 transactions table、booking 也沒有抽成欄位且是 3 狀態、銀行欄位鎖好了。READY for M3。跟我說『啟動 M3』，我們用 AWS Route 53 把自訂網域接上 Vercel。」
- Any ❌ → list the failed items + the recovery step, and tell the student to fix then re-run `驗收 M2.2`. Common failures: the VIEW still reads a `transactions` table or `bookings.barber_id`/the slot path or a `payout_pending` booking status (A1), the split doesn't sum back because it wasn't `price - platform_cut` (A2), build_payout didn't stamp `payout_id` or didn't snapshot (D1), the same-shop guard is missing so a mixed-shop batch slips through (D2), an already-batched booking gets re-grabbed (D3), mark-transferred wrongly flips a booking status (D4), cancel deletes the row instead of keeping it / doesn't null the bookings' `payout_id` (D6), the `/admin/*` gate is UI-only so a non-admin gets in (C2/C3), or a stray `transactions` table / `platform_fee` column / `payout_pending` status survives from the old model (E2).
