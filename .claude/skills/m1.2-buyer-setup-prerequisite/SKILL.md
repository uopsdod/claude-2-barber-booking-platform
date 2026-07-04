---
name: m1.2-buyer-setup-prerequisite
description: One-time carryover check before Milestone 1.2 of the barber-booking course — an INTERACTIVE, agent-driven walkthrough that confirms M1.1's SELLER side is real before the BUYER booking flow is built on top of it. M1.2 has no new accounts/connectors to set up (M0/M1.1 wired AWS + Supabase MCP + GitHub PAT), so this is a data-readiness check of exactly what M1.2 consumes from M1.1: the M1.1 schema (barbers with shop_id NOT unique, services with required_slots, bookable_slots as a plain time window with NO status column, platform_settings.slot_minutes), the `*_write_own` RLS on those tables, at least ONE bookable barber (a service + ≥ required_slots consecutive still-free slots to actually book against), and a clean bookings/booking_slots slate (they don't exist yet — M1.2 creates them). It does NOT require a customer account to pre-exist — on the student's OWN Supabase the only account is the ONE Shop from M1.1 (M0's customer/shop sign-ups were on Lovable's own Supabase / Lovable Cloud and did NOT carry over when M0 swapped auth to the student's project). The customer side is FIRST built in M1.2, so you sign up the test customer AS the first act of the build, not as a prerequisite; this skill only confirms the M0 Customer↔Shop role tab can mint one (`role='customer'`) — a zero-row `where role='customer'` is the EXPECTED state, not a gap. Use when the student starts M1.2 for the first time, says "啟動 M1.2 前置" / "M1.2 prerequisites" / "check M1.2 setup", or when `m1.2-buyer-setup` / `-checklist` detects M1.1 data (barbers/services/slots) or the Supabase MCP is missing.
---

# M1.2 Prerequisites — interactive carryover check (the agent drives)

**You are the Cowork agent running this skill. Drive the student through it one part at a time** — don't dump the whole thing. For each part: say what's about to happen, **check first** (Supabase MCP), tell the student only what actually needs fixing, run the read yourself, confirm it worked, and move on.

> **Source of truth = GitHub `main` + the Supabase project, not a local checkout.** Before anything, have the student **`git pull` on `main`** and confirm the repo is current — a local `src/` can be commits behind while the DB/deploy are ahead, and debugging "my code doesn't match the live app" wastes a session ([[supabase-best-practice]] watch-out #11).

> **M1.2 introduces nothing new to set up.** M0 wired AWS/Vercel/Supabase and cached the GitHub PAT; M1.1 created the seller-side data tables (`barbers` / `services` / `bookable_slots` + `platform_settings` + `barber_photos`) under RLS. M1.2 is the **buyer** milestone — it builds `/barbers`, `/barbers/[id]`, and `/bookings` **on top of** that data, and adds only the `bookings` + `booking_slots` tables. So this prereq is a **data-readiness carryover check**, not an account setup: confirm M1.1's seller data is real and correctly shaped, confirm there's actually **something bookable**, then confirm the booking tables are a clean slate. (The customer you book *as* is signed up during the build — not confirmed here; see the ⚠️ below.)

> ⚠️ **The buyer flow needs REAL M1.1 data to exercise, not just tables.** A green M1.1 *checklist* proves the schema + RLS are correct, but M1.2's happy path also needs **one barber that is actually bookable right now**: a published service with `required_slots = N`, and **N consecutive `bookable_slots` that no booking holds yet**. If the only slots are in the past or there aren't N in a row, `/barbers/[id]` will render but the Book dialog will have nothing to offer. Check this here so the student doesn't hit an empty dialog mid-build.

> ⚠️ **The customer account is NOT a prerequisite — it's the first act of the M1.2 build.** On the student's own Supabase there is **only one account: the Shop from M1.1.** M0 did sign up both Customer and Shop accounts — but against **Lovable's own Supabase (Lovable Cloud)**; M0 then swapped auth onto the student's own project, so **those M0 accounts did not carry over**. On the real project the student only ever created the **one shop** (in M1.1). So **do not gate on a `role='customer'` profile existing** — its absence is the correct, expected state. When you start building, you sign up the test customer via the **Customer** tab on `/login` and book as them; you just won't reuse the shop account to book. All this prereq confirms is that the **mechanism** to mint a customer — the M0 Customer↔Shop role tab that writes `role='customer'` — is in place (i.e. `profiles.role` and its allowlist are correct). The M1.2 *checklist* (D3) later needs **two** customers for the RLS-isolation test, but those are created during the build, not carried over.

## Architecture (what this check unlocks)

![Barber platform architecture (M1.2 — buyer booking) — this prereq confirms the LEFT (shop) half is real before the RIGHT (customer) half is built. M1.1 already produced, on the shared Supabase Database: profile (role), the many barbers (name/intro/address) with style photos + services (price, required slots #) + bookable slots (start time / end time). M1.2 will add the customer-side Product Site + the booking box (customer, service, bookable slot(s), status, price, paid_at, payout) whose greyed "bookable slot(s)" line is the unshown booking_slots join table — none of which exists yet. This skill verifies the M1.1 seller data (barbers/services/bookable_slots + platform_settings), the *_write_own RLS on it, that at least one barber is actually bookable (a service + ≥ required_slots consecutive free slots), and that bookings/booking_slots are a clean slate — it does NOT require a pre-existing customer (that's minted as the first act of the M1.2 build, not a carryover), only that the M0 role tab can create one.](assets/architecture-m1.2.png)

The four things you confirm here are exactly what M1.2 builds on:
- **M1.1 seller schema** (`barbers` / `services` / `bookable_slots` + `platform_settings`) → what `/barbers` (browse) and `/barbers/[id]` (detail) read.
- **`required_slots` on services** → M1.2 books **N = `service.required_slots`** consecutive slots (NOT `ceil(duration/30)`).
- **`bookable_slots` is a plain time window (NO `status` column)** → M1.2's availability is a **NOT EXISTS anti-join against `booking_slots`**, never a status read; a leftover `status` column means the old model leaked in.
- **A bookable barber** (a service + ≥ `required_slots` future free slots) → so the Book dialog actually has slots to offer once you sign up a customer and test.
- **`*_write_own` RLS + Supabase MCP reachable** → how M1.2 applies its `bookings`/`booking_slots` migration and how the new RLS is scoped.

(The customer you book *as* is created **during** the M1.2 build via the M0 Customer tab — not confirmed here — because M1.2 is where the customer side is first built.)

## When to load this skill

- The student says "啟動 M1.2 前置" / "M1.2 prerequisites" / "check M1.2 setup".
- The `[[m1.2-buyer-setup]]` build skill (or its checklist) detects the M1.1 data (barbers/services/slots) or the Supabase MCP connector is missing.

**Opening line to the student (say something like):**
> "M1.2 is the *buyer* side — browse barbers, open one, and book a slot. It has nothing new to install: it builds on the barbers, services, and schedule slots your shop published in M1.1. So I'll just confirm three carryovers are good — your M1.1 data is really there and correctly shaped, at least one barber is actually bookable right now, and the booking tables are a clean slate — then we build. (You'll sign up the *customer* you book as during the build itself — that's M1.2's job, not a prerequisite.) Let me check what's already there."

---

## Step 1 — M1.1 seller data exists and is correctly shaped

M1.2's browse/detail pages read M1.1's tables directly, so confirm they exist with the M1.1 shapes **before** building on them. All reads run through the Supabase MCP **`execute_sql`** tool.

> **Tool names:** this course's Supabase MCP namespaces its tools **per session** (e.g. `mcp__<session-id>__execute_sql`), so a hard-coded `mcp__claude_ai_Supabase__…` string won't resolve. Refer to the tools by their **bare names** — `list_tables`, `execute_sql`, `apply_migration`, `get_advisors`, `generate_typescript_types` — and call whichever namespaced variant your session exposes.

> **Note for Claude Code:** a single `execute_sql` with several `select`s returns only the **last** result set — run multi-read checks as **separate calls** or wrap them in one `json_build_object(...)`.

**1a — the three M1.1 tables + `platform_settings` exist:**
```sql
select to_regclass('public.barbers')          as barbers,
       to_regclass('public.services')         as services,
       to_regclass('public.bookable_slots')   as bookable_slots,
       to_regclass('public.platform_settings') as platform_settings;
```
- All four **non-NULL** → ✅ the M1.1 schema is present.
- Any is **NULL** → M1.1 didn't land (or a partial run). Stop and send the student back to `[[m1.1-seller-setup]]` (and run `[[m1.1-seller-setup-checklist]]` to confirm before returning).

**1b — `services.required_slots` is the slot count M1.2 books (NOT `duration_min`):**
```sql
select id, barber_id, name, category, price, required_slots
from public.services order by created_at desc limit 10;
```
- Rows have a **`required_slots`** integer (≥ 1) and a whole-integer `price` (in `platform_settings.currency`, TWD **not** ×100) → ✅. M1.2 will book **N = required_slots** consecutive slots.
- **No `required_slots`** (or a `duration_min` instead) → the old model leaked in; re-apply the M1.1 service migration. M1.2 uses `required_slots` **directly** — never `ceil(duration/30)`.

**1c — `bookable_slots` is a plain time window with NO `status` column (the decisive shape check):**
```sql
select column_name from information_schema.columns
where table_schema='public' and table_name='bookable_slots'
order by ordinal_position;
```
- Exactly `id, barber_id, starts_at, ends_at, created_at` and **NO `status` column** → ✅. This is what lets M1.2 derive availability via the **`booking_slots` anti-join** instead of a slot status.
- A **`status` column is present** → the old "stamp the slot" model leaked in. Flag it: M1.2 assumes slots have no status (availability is an anti-join against `booking_slots`), so drop it before building. *Recovery:* re-apply the M1.1 slot migration.

**1d — `platform_settings.slot_minutes` is seeded** (the fixed slot length — M1.2 shows N × slot_minutes to the buyer):
```sql
select currency, currency_minor_units, slot_minutes from public.platform_settings limit 1;
```
- One row with a `slot_minutes` (e.g. 30) and a `currency` → ✅.
- Empty → M1.1's single-row config seed didn't run; re-apply the M1.1 `platform_settings` migration.

---

## Step 2 — The `*_write_own` RLS on the M1.1 tables is intact + Supabase MCP reachable

M1.2 applies its own migration through the MCP and layers new RLS on `bookings`/`booking_slots`; it also relies on M1.1's RLS still being correct (e.g. `bookable_slots` public-select so the anonymous availability anti-join can read it).

**2a — MCP reaches the project** (call the **`list_tables`** tool):
```text
list_tables   →  should list public.profiles, barbers, services, bookable_slots, platform_settings
```
- Returns the M1.1 tables → ✅ the connector points at the barber-platform project. `get_advisors` (used in M1.2 after its migration) is the same connector, so it's callable too.
- **Auth error / wrong project / missing M1.1 tables** → the Supabase Connector isn't installed or points at a different org/project. Have the student re-install it (Cowork → Connectors → **Supabase Connector**) and pick the **barber-platform** project, then re-run.

**2b — the M1.1 write policies are scoped to the owner (structural, via MCP):**
```sql
select tablename, policyname, cmd
from pg_policies
where schemaname='public'
  and tablename in ('barbers','services','bookable_slots')
order by tablename, policyname;
```
- Each table has a `*_write_own` policy (scoped to `auth.uid()` directly for `barbers`, via the `exists (… shop_id = auth.uid())` sub-select for `services`/`bookable_slots`), and `bookable_slots` has a **public-select** policy → ✅.
- A `bookable_slots` with **no public select** → the buyer's availability anti-join can't read slots; note it (M1.2's browse page needs public-read slots). *Recovery:* re-apply the M1.1 RLS migration.

> **Note for Claude Code:** the Supabase MCP runs **privileged and BYPASSES RLS**, so it can only verify policy **definitions** here, not behavioral denial. Don't attempt a cross-tenant write from the MCP to "prove" RLS — that's the M1.1 checklist's job (Section D, in the live app). **Apply no migration in this prereq** — M1.2 owns the `bookings`/`booking_slots` schema.

---

## Step 3 — There is actually something bookable (and the role tab can mint a customer)

This is the check the M1.1 checklist doesn't do — M1.2's happy path needs **live, in-the-future, free** capacity to book against. It does **not** need a customer to pre-exist: you'll sign one up during the build (see the note at the end).

**3a — at least one barber has a service AND ≥ `required_slots` consecutive FUTURE slots that no booking holds.** Since `booking_slots` doesn't exist yet (M1.2 creates it), "held" is vacuously false here — so this reduces to "enough future slots exist on a barber that has a service":
```sql
-- barbers that have at least one service and at least required_slots future slots:
select b.id as barber_id, b.name,
       (select count(*) from public.services s where s.barber_id = b.id) as services,
       (select count(*) from public.bookable_slots sl
          where sl.barber_id = b.id and sl.starts_at > now()) as future_slots,
       (select min(s.required_slots) from public.services s where s.barber_id = b.id) as min_required_slots
from public.barbers b
order by future_slots desc
limit 10;
```
- At least one row where **`services >= 1`** and **`future_slots >= min_required_slots`** → ✅ that barber is bookable; note its `barber_id` for the build skill's first manual test.
- **No barber qualifies** (no future slots, or fewer than `required_slots` in a row) → the buyer flow can't be exercised. Send the student to `[[m1.1-seller-setup]]` `/shop/bookings` to publish a **future** slot window (at least `required_slots` consecutive slots for the service they want to test), then re-run. Don't build M1.2 against an empty schedule — you won't be able to smoke-test the Book dialog.

> **Note for Claude Code:** "consecutive" is enforced by M1.2's `create_booking` RPC at book time; here a rough `future_slots >= min_required_slots` count is enough to confirm the schedule isn't empty. If you want to be exact, eyeball the future slots for one barber (`select starts_at, ends_at from public.bookable_slots where barber_id = '<id>' and starts_at > now() order by starts_at`) and confirm N of them are back-to-back (`ends_at` of one = `starts_at` of the next).

**3b — the M0 role tab can mint a customer (a MECHANISM check — a customer row does NOT and should NOT exist yet).** On the student's own Supabase there is **only the one Shop account** from M1.1. The customer accounts M0 tested were signed up against **Lovable's own Supabase (Lovable Cloud)** — M0 then swapped auth onto the student's own project, so those Lovable-Cloud accounts **did not carry over**. So `select … where role='customer'` returning **zero rows is the correct, expected state** — not a gap to fill. All you confirm here is that the M0 Customer↔Shop sign-up tab writes `role='customer'` correctly, so that when the build starts you *can* sign one up. Read the `profiles.role` allowlist (the same constraint M1.1's prereq checked), **not** a customer row:
```sql
select pg_get_constraintdef(c.oid)
from pg_constraint c
join pg_class t on t.oid = c.conrelid
where t.relname = 'profiles' and c.contype = 'c';
```
- Allowlist includes **`customer`** (e.g. `role in ('customer','shop','admin')`) and `customer` is the M0 **default** → ✅ the Customer tab can mint the account you'll book as.
- **Do NOT query for or wait on a `role='customer'` row** — on the student's own Supabase the only account is the M1.1 shop, and that's exactly right. Just tell the student what happens next:
  > "Heads-up (not a blocker): your own Supabase only has the one **shop** account from M1.1 — you've never signed up a customer here (the ones from M0 lived on Lovable's Supabase and didn't carry over, which is expected). At the very start of the M1.2 build you'll sign up the **customer** you book as, via the **Customer** tab on your live site's `/login` (it writes `role='customer'`). We just won't reuse the shop account to book. (The M1.2 checklist later wants a *second* customer for the RLS-isolation test — also created then, not now.)"

---

## Step 4 — One verified read: the booking tables are a clean slate

M1.2 creates `bookings` + `booking_slots` (+ the `bookings_with_start` view). The "verified read" here is the **negative** read that proves they don't exist yet — exactly the clean slate the build skill starts from:
```sql
select to_regclass('public.bookings')           as bookings,
       to_regclass('public.booking_slots')       as booking_slots,
       to_regclass('public.bookings_with_start')  as bookings_with_start;
```
- All three return **NULL** → ✅ clean slate; M1.2 Step 1 creates them fresh.
- One or more **already exists** (a partial earlier run) → read its columns and tell the student. The M1.2 migration uses `create table if not exists`, so it's safe to re-run, but confirm the shape matches the current model (**no `start_slot_id` / `slot_id` / `barber_id` on `bookings`**; a `UNIQUE(slot_id)` on `booking_slots`) before layering the UI, so an old denormalized model doesn't leak forward.

> **Prereq scope note:** this prereq deliberately does **static** checks only — table/column/constraint shape and expected seed data (Steps 1–4). It does **not** call `create_booking` or any function. **Function-call verification** (the live, rolled-back `create_booking` smoke test that catches a broken RPC even when the schema looks right) lives in the **`[[m1.2-buyer-setup-checklist]]`** (check A3a), which runs *after* the build creates that function — that's the right home for behavioral proofs.

---

## Verify (all must pass)

- ✅ **M1.1 seller schema** — `barbers` / `services` / `bookable_slots` / `platform_settings` all exist; `services.required_slots` present; `bookable_slots` has **NO `status` column**; `platform_settings.slot_minutes` seeded (Step 1).
- ✅ **M1.1 RLS intact + MCP reachable** — `list_tables` returns the M1.1 tables; the `*_write_own` policies are scoped to `auth.uid()` and `bookable_slots` is public-select (Step 2).
- ✅ **Something bookable** — ≥ 1 barber has a service and ≥ `required_slots` future free slots; the `profiles.role` allowlist includes `customer` so the M0 tab can mint the account you'll book as during the build (a customer row is **not** required to exist yet) (Step 3).
- ✅ **Clean booking slate** — `bookings` / `booking_slots` / `bookings_with_start` are NULL (don't exist yet), ready for M1.2's migration (Step 4).

## Next step

When all four are ✅, tell the student:
「前置檢查通過 ✅ —— M1.1 的 `barbers`／`services`（含 `required_slots`）／`bookable_slots`（純時段、**沒有 status 欄位**）都在、RLS 正常、Supabase MCP 連得上；而且至少有一位理髮師是**現在就能約的**（有服務、也有足夠的未來連續時段）；`bookings`／`booking_slots` 則是乾淨的空白狀態。**買家帳號不用事先準備**——M1.2 才是第一次做客人端，所以你會在建置一開始，用 `/login` 的 **Customer** tab 註冊一個 `role='customer'` 帳號來預約（M1.1 是用 shop 帳號跑的，不要拿來預約）。接下來我會用 migration 建 `bookings` ＋ `booking_slots` 這兩張表（含 `bookings_with_start` view、`UNIQUE(slot_id)` 防雙訂、與 RLS），再做 `/barbers`、`/barbers/[id]`（含 Book pop-up dialog）和 `/bookings`。跟我說『啟動 M1.2』就開始。」
Then return to the build skill `[[m1.2-buyer-setup]]` (Step 1).

## Reference

- `[[m1.1-seller-setup]]` / `[[m1.1-seller-setup-checklist]]` — the seller side this prereq confirms is real.
- `[[m1.2-buyer-setup]]` — the build skill this prereq gates.
- `[[m1.2-buyer-setup-checklist]]` — the post-build verification (which needs a **second** customer for the RLS-isolation test).
- `[[supabase-best-practice]]` — migration/RLS discipline the M1.2 tables follow.
