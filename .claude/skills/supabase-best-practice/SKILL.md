---
name: supabase-best-practice
description: Hard rules for working with Supabase in the 抽成制理髮師預約平台 — where Supabase holds REAL multi-tenant application data (barbers / services / bookable_slots / bookings / payouts), not auth-only. Covers migration discipline (always a migration file / apply_migration, never a raw prod UPDATE), RLS-by-default on every tenant table, shop-level bank-info columns (on `profiles`) readable only by shop + admin, the publishable-key-client-side / service-role-server-only split, and the one-shop-many-barbers model (a shop has many flexible payout batches; no-double-pay is on `bookings.payout_id`). Use whenever a student is creating a table, applying a migration, writing an RLS policy, promoting an admin, or asking where the service-role key goes. Apply proactively — stop the student before they break one.
---

# Supabase Best Practice (Barber Booking — REAL DATA, multi-tenant, RLS)

> **Read this first — it changes everything.** Unlike a flight/notifier course where Supabase was **auth-only**, here Supabase is the **real application database**: `profiles`, `platform_settings`, `barbers`, `services`, `bookable_slots`, `booking_slots`, `bookings`, `commission_rates`, `payouts` (+ the `owed_bookings` and `bookings_with_start` views), plus the `profiles.role` stub from M0. It is **multi-tenant** — every barber's data lives in shared tables, and the **only** thing keeping shop A from reading/editing shop B's barber (or a customer from reading a shop's bank account) is **Row Level Security**. So the migration, RLS, and key-hygiene rules below are **load-bearing**, not optional.

When you (Claude Code) guide a student through any Supabase work (M0's `profiles` stub + `platform_settings`, M1.1's barber/schedule schema, M1.2's bookings + booking_slots, M2.1's webhook writes + `commission_rates`, M2.2's flexible per-shop `payouts` batches and the admin promotion), **apply these rules proactively** — stop them before they break one. Each maps to a real failure mode of a multi-tenant data-on-Supabase build.

---

## The canonical table set (9 tables + 2 views)

`profiles` · `platform_settings` · `barbers` · `services` · `bookable_slots` · `booking_slots` · `bookings` · `commission_rates` · `payouts` — **plus** the `owed_bookings` and `bookings_with_start` **views**. **NO `transactions` table** and **NO `payout_records` table** (there is no per-booking fee/ledger row — see below). The load-bearing structural facts:

- **One shop → many barbers** — `barbers.shop_id → profiles.id` is **NOT unique** (index it; Rule 5).
- **A booking spans N consecutive slots** — `N = service.required_slots` (an integer count; *not* `ceil(duration_min/30)`). The N held slots live in the **`booking_slots`** join table; **`UNIQUE(slot_id)`** there is the no-double-book guard, and cancelling a booking **deletes** its `booking_slots` rows (free-on-cancel). Slot availability is the anti-join `not exists (select 1 from booking_slots bs where bs.slot_id = s.id)` — `bookable_slots` itself has **no status column**. `bookings` stores **no `start_slot_id`** — the start time is `MIN(starts_at)` over the join, exposed by the **`bookings_with_start`** view.
- **`bookings.status` is 3 states** — `pending_payment | paid | cancelled`. There is **no `payout_pending`/`payout_transferred`** status. Settlement state is **derived from `bookings.payout_id`** (a FK → `payouts`): NULL = `paid` but not yet paid out (**owed**); set = settled into that payout batch (whose own `pending_transfer`/`transferred` status is the *payout's* business).
- **Money lives in `platform_settings.currency`** — money columns are **integers** with **no `_twd` suffix** (`price`, `gross`, `platform_cut`, `shop_cut`). `platform_settings` (single-row config) holds `currency`, `currency_minor_units`, `slot_minutes`.
- **The money split lives NOWHERE on `bookings`** — there are no `platform_fee`/`barber_amount` columns. The 20/80 split is **computed when the admin builds a payout** (M2.2) from the picked paid bookings × the versioned `commission_rates` row, and snapshotted onto a `payouts` batch row — never recomputed off a per-booking ledger (there isn't one).
- **RLS on every tenant table** (Rule 2); bank fields on `profiles`, shop+admin only (Rule 3).

---

## Execution mode: Cowork vs CLI

| Operation | CLI mode | Cowork mode |
|---|---|---|
| Apply a schema change | `supabase migration new …` → edit → `supabase db push` | Supabase MCP **`apply_migration`** (named migration — leaves a record) |
| Run a one-off read/query | `psql` / dashboard SQL editor | Supabase MCP `execute_sql` |
| Check RLS / security advisors | dashboard → Advisors | Supabase MCP **`get_advisors`** (run after every migration) |
| List tables / inspect schema | dashboard → Table editor | Supabase MCP `list_tables` |
| Get URL / publishable key | dashboard → Settings → API | Supabase MCP `get_project_url` + `get_publishable_keys` |

> **Always prefer `apply_migration` over `execute_sql` for anything that changes schema or production rows.** `execute_sql` is for **reads** and genuine one-offs; structural change and data correction go through a **named migration** so there's a versioned, reviewable record (Rule 1).

---

## Hard rules

### Rule 1 — Every schema change AND every production data fix goes through a migration. Never a raw ad-hoc `UPDATE`.

> **The rule:** Create tables, add columns, write RLS policies, and even **promote an admin** via a **migration file** (`apply_migration` in Cowork, `supabase migration new` + `db push` in CLI). **Never** open the dashboard SQL editor and run a raw `UPDATE public.profiles SET role='admin' …` against production by hand.

**Why:** A migration is **versioned, reviewable, and replayable** — it's in git, it ran the same way on every environment, and you can see *what* changed and *when*. A raw console `UPDATE` is **invisible and unrepeatable**: there's no record that it happened, a typo'd `WHERE` clause silently updates **every** row (promote every user to admin, zero out every price), and a fresh environment has no way to reproduce the state. In a money-handling multi-tenant app, an untracked prod edit is how you get "it works on mine but not in prod" and an un-auditable security change.

**How to apply:**
- Schema (M0 `platform_settings`, M1.1 `barbers`/`services`/`bookable_slots`, M1.2 `bookings`/`booking_slots` + the `bookings_with_start` view, M2.1 `commission_rates`, M2.2 `payouts` + the `owed_bookings` view) → `apply_migration`, one named migration per change.
- **Admin promotion (M2.1 prereq)** is a migration too:
  ```sql
  -- migration: promote_admin
  update public.profiles set role = 'admin' where email = '<your-email>';
  ```
  applied via `mcp__claude_ai_Supabase__apply_migration`. A user can never self-escalate because promotion only ever happens through a migration **you** run. (Local/dev: a `seed.sql` can promote a known dev account.)
- A genuine read ("which user has this email?") → `execute_sql` is fine. A *write* that fixes prod data → still a migration.
- If a student is about to paste a raw `UPDATE`/`ALTER` into the dashboard: 「改 schema 或改正式資料一律走 migration（`apply_migration`），不要在 console 直接下 SQL。」

---

### Rule 2 — RLS ON by default on every tenant table, with policies that scope rows to their shop.

> **The rule:** The moment you create `barbers` / `services` / `bookable_slots` / `booking_slots` / `bookings` / `commission_rates` / `payouts`, **`alter table … enable row level security`** and add explicit policies. A shop may CRUD **only their own** barbers/services/slots; a customer may read public barber/service/slot data and CRUD **only their own** bookings; the admin (via `role='admin'`) reads what the payout page needs.

**Why:** These are **shared, multi-tenant** tables — every shop's rows sit next to every other shop's. With RLS **off**, the publishable key (which ships to every browser) can `select * from barbers` and read **every** shop's data, or `update` someone else's prices. RLS is the *only* boundary between tenants on the client path. And the failure mode is **silent**: a missing/over-broad policy doesn't error — it just leaks or lets a write through. (The inverse also bites: a `.from(a).select('*, b(*)')` join returns **empty with no error** when RLS denies either side — a classic half-day debug.) Turn it on with the table, not "later."

**How to apply:**
```sql
alter table public.barbers enable row level security;

-- public can read barber listings (for /barbers, /barbers/[id]) — and `barbers` has no bank columns
-- at all now (bank is shop-level, on profiles — Rule 3), so this read is clean by construction
create policy "barbers_public_read" on public.barbers for select using (true);
-- only the owning shop can write their barber profile
create policy "shops_owner_write" on public.barbers for all
  using (shop_id = auth.uid()) with check (shop_id = auth.uid());
```
- `services` / `bookable_slots`: public `select`; `insert/update/delete` only where the parent `barber.shop_id = auth.uid()`. (`bookable_slots` is just a time window — no status column; see Rule 6.)
- `bookings`: a customer reads/writes **their own** (`customer_id = auth.uid()`); a shop reads bookings for barbers they own — and because `bookings` has **no `barber_id`**, that read policy **joins through the service**: `exists (select 1 from services sv join barbers b on b.id = sv.barber_id where sv.id = bookings.service_id and b.shop_id = auth.uid())`. The **webhook writes via the service-role key** (Rule 4), which bypasses RLS by design — that's correct, because Stripe isn't a logged-in user.
- **Run `get_advisors` after every migration** — it flags tables with RLS off or policy gaps. Treat any "RLS disabled" advisory as a blocker.

---

### Rule 3 — Bank-info columns live on the SHOP (`profiles`) and are readable by that shop + admin ONLY. Never on `barbers`, never in a public/RLS-readable view.

> **The rule:** The payout target — `profiles.bank_account_name` and `profiles.bank_account_number` — is **shop-level** (one bank account per shop, shared by all that shop's barbers) and must be reachable **only** by the owning shop and by `role='admin'`. `barbers` carries **no** bank columns. The bank fields must **never** be returned by the public barber-listing read, and never embedded in any view a customer can query.

**Why:** Bank details are the most sensitive field in the schema. Because **one SHOP can run MANY barbers**, the payout account belongs to the **shop**, not a barber profile — so it lives on `profiles`, written once via the shop-level "payout settings" form (M1.1). And the public `barbers` listing (`/barbers`, `/barbers/[id]`) is read by **everyone**: if any public read returns bank columns, **every visitor can scrape every shop's bank account** — a serious breach. Keeping bank fields off `barbers` entirely means the wide, public barber row is never even tempting to over-expose; the only thing to lock down is the shop's own `profiles` row.

**How to apply:**
- **Bank fields on `profiles`, shop+admin-only:** the existing `profiles_select_own` / `profiles_update_own` RLS already scopes a user to their **own** row — so a non-shop cannot read another shop's bank fields. Admin read is allowed via an `is_admin()` check. **No public view** (and no `NEXT_PUBLIC_` query) may project `bank_account_*`.
- **`barbers` has no bank columns:** the public barber read is now clean by construction — `barbers` is just `id, shop_id, name, intro, address, created_at`. A column-safe `barbers_public` view (same non-sensitive columns) is fine to keep, but there are **no** bank columns to drop anymore.
- Admin access: a helper like `is_admin()` = `exists(select 1 from profiles where id = auth.uid() and role='admin')`, used in a `profiles` read policy `using (id = auth.uid() or is_admin())` so admin can read any shop's bank fields for the payout page.
- The admin payout page (M2.2) reads the shop's `profiles` bank fields **server-side** (service-role) under the admin gate — never shipped to a customer's browser.
- **Check it:** as a customer (publishable key), `select bank_account_number from profiles where id <> auth.uid()` must return **nothing**, `barbers` must have **no** bank columns at all, and the `/barbers` listing payload must not contain a bank field. If a student puts bank columns on `barbers` or in the public listing: 「銀行資訊是店家層級、放在 `profiles`，只有本人和 admin 能讀，不要放在 `barbers`、也不要進公開的店家列表。」

---

### Rule 4 — Publishable key in the browser; service-role key ONLY in the webhook + admin server routes (Vercel server env, never the browser).

> **The rule:** The front-end uses **only** the Supabase **publishable key** (`sb_publishable_*`, RLS-protected, browser-safe). The **service-role key** — which **bypasses RLS entirely** — lives **only** in **Vercel server env** (`SUPABASE_SERVICE_ROLE_KEY`) and is read **only** by server-side code: the Stripe webhook (M2.1) and admin server routes (M2.2). It must never appear in client code, a Lovable prompt, a commit, a `NEXT_PUBLIC_*` var, or a screenshot.

**Why:** The publishable key can only do what RLS + auth allow — safe to ship. The **service-role key owns the database**: it ignores every RLS policy, so anyone who gets it can read every shop's bank account and rewrite the payout ledger. The webhook *needs* it (Stripe carries no user session, so it must flip a booking to `paid` past RLS) and the admin payout writes need it — but those are **server-only** contexts. The instant it's prefixed `NEXT_PUBLIC_` or pasted into a component, it's in every visitor's network tab. There is no recovering a leaked service-role key except rotation.

**How to apply:**
- Front-end / Lovable: publishable (anon) key only — `NEXT_PUBLIC_SUPABASE_URL` + the publishable key.
- Server routes (`/api/stripe/webhook`, `/api/admin/*`): create the client with `SUPABASE_SERVICE_ROLE_KEY` from **Vercel server env** (no `NEXT_PUBLIC_` prefix). This key is read at request time; it is **not** stored in AWS ([[aws-secrets-best-practice]]).
- If a student pastes a `service_role` key into the front-end, a `NEXT_PUBLIC_*` var, or a commit → **stop them**, explain the blast radius, and rotate it (dashboard → Settings → API → roll).
- **Go-live check:** grep the deployed browser bundle for `service_role` / the service key → must be **absent** (only the publishable key may appear client-side).

---

### Rule 5 — One SHOP can run MANY barbers. The bank/payout target is on the SHOP (`profiles`); a shop has MANY flexible payout batches, and no-double-pay is enforced by `bookings.payout_id`.

> **The rule:** A shop (`profiles.id`, `role = 'shop'`) can run **many** barbers — `barbers.shop_id` → `profiles.id` is **NOT unique** (add an **index** on `shop_id`, not a unique constraint). Money is attributed to a shop by joining `bookings → services → barbers.shop_id` (bookings has no `barber_id`; the barber is reached via `service_id` — see Rule 6). `payouts` is a **flexible per-shop batch**: `payouts.shop_id` is **NOT unique** (a shop has many payouts over time), `status` is `pending_transfer | transferred | cancelled`, and the admin picks **which** of that shop's owed bookings to include. There is **NO `(shop_id, month)` unique key and NO `monthly_payouts` view** — the no-double-pay guarantee is on **`bookings.payout_id`** (a booking is in **≤ 1** payout).

**Why:** The model — "compute each shop's 80% owed, mark transferred, one bank transfer per shop" — needs a clean mapping from **money → one payable person**. That payable person is the **shop**, who may run several barbers; so the bank account lives **once** on the shop (`profiles`), and a payout's owed amount is the **sum across all their barbers' picked bookings**. One payout = one shop, so its single `status` cleanly means "this shop's batch got paid"; the admin makes **one** transfer per batch, not one per barber. The batch is **free-form** (any dates, a partial month, a single late booking) rather than a fixed monthly grouping — so `payouts.shop_id` must **not** be unique. A `unique(shop_id)` on `barbers` would wrongly block a second barber, and a `unique(shop_id, month)` on `payouts` would wrongly block a second batch — don't add either. Double-paying a booking is prevented structurally: `bookings.payout_id` is single-valued, so a booking can belong to at most one payout.

**How to apply:**
- `barbers.shop_id` references `profiles.id`; **NOT unique** — `create index idx_barbers_shop on public.barbers(shop_id)` so one shop can have many barbers.
- A booking attributes revenue to a barber **through its service**: `bookings.service_id → services.barber_id → barbers.shop_id`. There is **no `barber_id` column on `bookings`** (Rule 6) — `service_id` already pins the barber, so don't denormalize it onto the booking.
- M2.2's `owed_bookings` view lists the live owed pool (`status='paid' and payout_id is null`, with the derived split, joining `bookings → services → barbers` for `shop_id`). The admin filters/groups it by shop in the UI and builds a `payouts` batch for **one** shop at a time, stamping `bookings.payout_id` on the picked rows. `payouts.shop_id` is indexed but **NOT unique**; the build RPC must verify every picked booking resolves to the **same** shop. Bank fields for the payout come from the shop's `profiles` row (Rule 3).
- **No double-pay:** rely on `bookings.payout_id` (single-valued FK) + the build guard `... and payout_id is null` — NOT a unique on `payouts`. Cancelling a `pending_transfer` payout nulls its bookings' `payout_id` (they return to owed); a `transferred` payout is immutable.
- If a student asks for **multiple barbers/staff per barber** (separate payable people inside one barber): "That needs per-staff attribution and an in-barber split — out of scope for v1. The course models one SHOP that may run many barbers, and pays the shop; it does not split within a barber."

---

## Things to actively watch out for

1. **A raw `UPDATE … SET role='admin'` in the dashboard SQL editor** → Rule 1 (use a migration; a bad `WHERE` promotes everyone).
2. **A new table with RLS still off** → Rule 2 (`get_advisors` flags it; the publishable key can read/write every tenant's rows).
3. **Bank account numbers showing up in the `/barbers` listing payload, or bank columns added to `barbers`** → Rule 3 (bank fields are shop-level on `profiles`; `barbers` must carry none, and no public read may project them).
4. **A join (`select '*, services(*)'`) returns empty with no error** → Rule 2 (RLS denied one side silently; check the policy, not the query).
5. **`SUPABASE_SERVICE_ROLE_KEY` prefixed `NEXT_PUBLIC_` or in a component** → Rule 4 (it bypasses RLS — rotate immediately; it belongs in Vercel server env only).
6. **The webhook can't flip a booking to `paid` (RLS denies it)** → Rule 4 (the webhook must use the **service-role** client; the publishable key can't write past RLS for a non-user actor).
7. **A `unique(shop_id)` on `barbers` blocking a second barber, or a `unique(shop_id, month)` on `payouts` blocking a second batch** → Rule 5 (a shop runs MANY barbers AND has MANY payout batches; index `shop_id` on both, make neither unique; no-double-pay is on `bookings.payout_id`, not a unique on `payouts`).
8. **Skipping `get_advisors` after a migration** → run it every time; it's the cheapest catch for an RLS gap before it ships.

---

## Out of scope for this course (real prod, not enforced here)

- **Multiple barbers/staff per barber / per-staff attribution + in-barber revenue split** — the course models one SHOP that may run many barbers and pays the shop; it does not split *within* a barber (Rule 5).
- **Column-level encryption of bank fields / Supabase Vault for app data** — the shop's `profiles` bank columns are plain columns gated by RLS (shop + admin); field-level crypto is a prod hardening pass. (Vault's role in *this* course is discussed in [[aws-secrets-best-practice]], and it's not used for app data.)
- **The simultaneous-click slot race** (DB-level locking / `select … for update` on a slot) — deferred until ~1,000 concurrent customers/barber; "first to pay wins" is sufficient. See [[m1.2-buyer-booking]].
- **Read replicas / connection pooling tuning / PITR** — course runs at tens-of-rows scale.

When a student asks "shouldn't we encrypt the bank field / handle the slot race?" → "Yes, for production. The course optimizes for the minimum correct multi-tenant data model with RLS as the boundary; the rest is a hardening pass once the milestones are stable."

---

## Cross-references

- [[m1.1-barber-shop-and-schedule]] — where `barbers`/`services`/`bookable_slots` + their RLS (Rules 2, 3, 5) are introduced, a shop can create MANY barbers, and the shop-level "payout settings" write the bank fields to `profiles`.
- [[m2.2-admin-to-seller-payment]] — the flexible per-shop `payouts` batches + the `owed_bookings` view, no-double-pay on `bookings.payout_id` (Rule 5), and the admin-only read of the shop's `profiles` bank fields (Rule 3).
- [[m2.1-buyer-to-admin-payments-prerequisites]] — the admin promotion done as a migration (Rule 1).
- [[m0-landing-page]] — the `profiles.role` stub these tables gate on.
- [[aws-secrets-best-practice]] — why app-runtime Supabase keys live in **Vercel env** (Rule 4), not AWS, and why Supabase Vault isn't used for secrets here.
- [[stripe-best-practice]] — the webhook that flips a booking to `paid` via the service-role client (Rule 4).
