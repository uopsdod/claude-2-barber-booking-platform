---
name: m1.1-seller-setup-checklist
description: 抽成制理髮師預約平台 Milestone 1.1 verification — checks the barber side is real and correctly locked down: barber CRUD (a shop can run MANY barbers), service CRUD, slot publish (`bookable_slots` is a plain time window — NO status column), sample-hairstyle-photo upload (the `barber_photos` table + the public `barber-photos` Storage bucket), the role flips to 'shop', AND the security tests that matter most — RLS denies a cross-barber edit, the shop-level bank fields on `profiles` are NOT world-readable (and `barbers` no longer has bank columns), and a barber can't write into another's photo folder. Use when the student says "驗收 M1.1", "check M1.1", "M1.1 done?", or after the `m1.1-seller-setup` skill completes Step 6.
---

# M1.1 — Barber Profile & Schedule Checklist

## What this skill does

Verifies the student actually completed M1.1 — not just *thinks* they did. People (and LLMs) skip steps, and on the barber side the **silent** failures are the dangerous ones: a missing RLS policy means barber A can edit barber B's prices, and a shop-level bank field (on `profiles`) leaking into a public read exposes account numbers. This skill tests every artifact + the two decisive security properties, reports pass/fail per item, then emits a `READY for M1.2` verdict.

**Run this AFTER `m1.1-seller-setup` Step 6, or any time the student claims M1.1 is done.**

## Execution mode: Cowork vs CLI (read this first)

| Section | CLI mode tool | Cowork mode equivalent |
|---|---|---|
| A — Barber CRUD | Supabase MCP / SQL editor | **Supabase MCP** (`execute_sql`) — preferred both modes |
| B — Service CRUD | Supabase MCP / SQL editor | **Supabase MCP** |
| C — Slot publish | Supabase MCP / SQL editor | **Supabase MCP** |
| D — RLS cross-barber denial | two browser sessions + Supabase MCP | **second test account in the live app** (MCP can't prove this — see below) |
| E — Role flips to 'shop' | Supabase MCP | **Supabase MCP** |
| F — Shop-level bank fields (profiles) not world-readable | Supabase MCP + `get_advisors` | **Supabase MCP** + `get_advisors` (structural) |
| G — Sample hairstyle photos | Supabase MCP + Storage | **second test account** for the cross-folder write (see below) |

The reads below run through the **Supabase MCP** (`execute_sql` / `get_advisors`) in both modes — that's the authoritative path. Refer to the MCP tools by their **bare names** (`execute_sql`, `get_advisors`, `list_tables`); this course's connector namespaces them **per session** (e.g. `mcp__<session-id>__execute_sql`), so the literal `mcp__claude_ai_Supabase__…` string won't resolve — call whichever namespaced variant your session exposes.

> ⚠️ **The Supabase MCP runs PRIVILEGED and BYPASSES RLS.** It does **not** execute under a barber's `auth.uid()`, so it **cannot** exercise the behavioral RLS tests (**D1** cross-barber edit, **G3** cross-folder upload): an `execute_sql` UPDATE there neither proves the policy blocks a cross-tenant write nor is itself blocked by it. Via MCP you can only verify **policy *definitions*** (the policy text is scoped to `auth.uid()`) + the advisor. The **behavioral** proof requires a **second account signed into the deployed app** (a real `auth.uid()`). **Never run the destructive `update … set name='HACKED'` from the report through the MCP against real data** — because MCP bypasses RLS, it would actually mutate rows instead of being filtered to 0. Run that attempt only as Barber B inside the live app.

## How to run

The student invokes this directly (e.g. types `驗收 M1.1`). You (Claude Code) **actively run** each check and report results — don't just describe them.

### Step 1: Collect what you need (one message)

Ask the student for:
1. The live Vercel URL (to exercise the UI / create a second barber).
2. Two barber test accounts (or offer to create a second one): **Barber A** (owns at least one barber, has a service + a slot) and **Barber B** (a different account). Section D needs both.
3. Confirm the Supabase MCP is connected to the **barber-platform** project.

### Step 2: Run the checklist

> **Note for Claude Code:** a single `execute_sql` call with **several `select`s returns only the LAST result set**. When a check has multiple reads, either run them as **separate `execute_sql` calls**, or wrap them in one `json_build_object(...)` / `to_jsonb(...)` so every value comes back in one row. Don't assume all statements' outputs are returned.

#### Section A — Barber CRUD (one shop can run MANY barbers)
- **A1** Barber A's barber(s) exist with the right shop:
  ```sql
  select id, shop_id, name, address from public.barbers order by created_at desc limit 5;
  ```
- **A2** **A shop can run MANY barbers** — `shop_id` is NOT unique (a second barber for the same shop is allowed), and there's an **index** on `shop_id` instead of a UNIQUE constraint:
  ```sql
  -- (1) no UNIQUE/PK on shop_id:
  select conname, pg_get_constraintdef(oid)
  from pg_constraint where conrelid = 'public.barbers'::regclass and contype in ('u','p');
  -- (2) an index on shop_id exists (e.g. idx_barbers_shop):
  select indexname, indexdef from pg_indexes
  where schemaname='public' and tablename='barbers' and indexdef ilike '%shop_id%';
  ```
  Expect **NO** UNIQUE constraint on `shop_id` (only the primary key on `id`), and an index covering `shop_id`. To double-check the model holds, confirm a single shop can have more than one row:
  ```sql
  select shop_id, count(*) from public.barbers group by shop_id order by count(*) desc limit 5;
  ```
  A count > 1 for a shop is now **valid** (it would have been a violation under the old one-barber model). *Recovery:* re-apply the M1.1 schema migration (build skill Step 2) — it drops the old `unique(shop_id)` and adds `idx_barbers_shop`.

#### Section B — Service CRUD
- **B1** Barber A's services are present with a valid category + a whole-unit price (in `platform_settings.currency`):
  ```sql
  select id, barber_id, name, category, price, required_slots from public.services order by created_at desc limit 10;
  ```
  Expect `category in (cut/color/perm/beard)` and `price` as a **whole integer** in `platform_settings.currency` (default TWD, NOT ×100). *Recovery:* the service editor in build skill Step 6 / the category `check` constraint.

#### Section C — Slot publish (a slot is just a time window — NO status column)
- **C1** A slot is a bare time window — `id, barber_id, starts_at, ends_at, created_at` with **NO `status` column**. Availability is **derived** (a slot is bookable until a live booking references it — M1.2), it is never stamped on the slot:
  ```sql
  select column_name from information_schema.columns
  where table_schema='public' and table_name='bookable_slots'
  order by ordinal_position;
  ```
  Expect exactly `id, barber_id, starts_at, ends_at, created_at` — and **NO** `status` column. Then confirm published slots have `ends_at > starts_at`:
  ```sql
  select id, barber_id, starts_at, ends_at from public.bookable_slots order by starts_at desc limit 10;
  ```
  *Recovery:* re-apply the M1.1 slot migration (build skill Step 6, section B) — a `bookable_slots.status` column means the old model leaked in; drop it. The booking lifecycle lives only on `bookings.status` (M1.2).

#### Section D — RLS denies cross-barber edits (THE DECISIVE TEST)

> ⚠️ **This test cannot be done through the Supabase MCP** — the MCP is privileged and bypasses RLS, so a cross-tenant UPDATE there would *mutate real rows* instead of being blocked. The behavioral proof needs **Barber B signed into the deployed app** (a real `auth.uid()`). **Offer to create a second test account** if the student only has one: sign up a second `shop` via `/login`, give it its own barber, then run the attempt below from B's live session (the app's Supabase client, or an MCP call genuinely scoped to B's JWT — *not* the privileged connector). What MCP *can* do here is **D0** (verify the policy definitions structurally).

- **D0 (structural, via MCP)** Confirm the `*_write_own` policies exist and are scoped to `auth.uid()` (this is all the privileged MCP can prove):
  ```sql
  select tablename, policyname, cmd, qual, with_check
  from pg_policies
  where schemaname = 'public'
    and tablename in ('barbers','services','bookable_slots')
    and policyname like '%own%'
  order by tablename, policyname;
  ```
  Expect each write policy's `qual` / `with_check` to reference `auth.uid()` (directly for `barbers`, or via the `exists (… s.shop_id = auth.uid())` sub-select for `services` / `bookable_slots`). A policy that's `using (true)` for `update`/`delete` is the bug. *Recovery:* re-apply the RLS migration (build skill Step 3).
- **D1 (behavioral, in the live app — the decisive test)** As **Barber B** (logged into the deployed app, so `auth.uid()` = B), attempt to update **Barber A's** barber / service / slot:
  ```sql
  -- run AS barber B from B's live session (NOT the privileged MCP connector):
  update public.barbers set name = 'HACKED' where shop_id <> auth.uid();   -- expect: 0 rows
  update public.services set price = 1 where barber_id =
      (select id from public.barbers where shop_id <> auth.uid() limit 1);     -- expect: 0 rows
  update public.bookable_slots set starts_at = now() where barber_id =
      (select id from public.barbers where shop_id <> auth.uid() limit 1);     -- expect: 0 rows
  ```
  **Pass = 0 rows affected** on all three (RLS filtered them out) **and** Barber A's data is unchanged when you re-read it. If any row changes → the `*_write_own` policies are missing/too loose. *Recovery:* re-apply the RLS migration (build skill Step 3) and re-run `get_advisors`.
- **D2** Conversely, Barber B **can** edit B's own barber (RLS shouldn't over-block). A self-update from B's live session returns 1 row.

#### Section E — Role flips to 'shop'
- **E1** A barber account carries `role = 'shop'`:
  ```sql
  select id, email, role from public.profiles where role = 'shop' order by created_at desc limit 5;
  ```
  Expect Barber A (and B) here. *Recovery:* the role wiring / "Become a shop" flow (build skill Step 1) — and confirm the upgrade path writes `shop`, **never `admin`**.

#### Section F — Shop-level bank fields (on `profiles`) NOT world-readable
> Bank details are now **shop-level** — they live on `profiles.bank_account_name` / `profiles.bank_account_number` (shared by all that shop's barbers), NOT on `barbers`. So the test is twofold: `barbers` must carry **no** bank columns, and the `profiles` bank fields must be readable only by their shop + admin.
- **F1** **`barbers` has NO bank columns** (they moved to `profiles`):
  ```sql
  select column_name from information_schema.columns
  where table_schema = 'public' and table_name = 'barbers';
  ```
  Expect `id, shop_id, name, intro, address, created_at` — and **NO** `bank_account_name` / `bank_account_number`. The public projection (`barbers_public`, if present) must likewise have no bank columns. *Recovery:* build skill Step 2/3 (the `barbers` schema dropped bank columns; the `barbers_public` view projects only non-sensitive columns).
- **F2** **`profiles` bank fields are shop+admin-only.** First confirm the columns exist on `profiles`:
  ```sql
  select column_name from information_schema.columns
  where table_schema='public' and table_name='profiles'
    and column_name in ('bank_account_name','bank_account_number');
  ```
  Then, as a **customer / different barber** (publishable key, so `auth.uid()` ≠ the shop), a read of **another** user's `profiles.bank_account_*` must return **nothing** — the existing `profiles_select_own` RLS scopes a user to their own row, and only `role='admin'` may read across rows. (Like D1, the *behavioral* cross-row read needs a real non-shop `auth.uid()` in the live app — the privileged MCP bypasses RLS. Via MCP, verify the policy text + advisor.) Confirm no public view re-exposes the bank columns, and the `get_advisors` tool shows no RLS-disabled / leaking finding:
  ```text
  get_advisors  →  type: "security"
  ```
  Expect RLS enabled on `platform_settings` / `profiles` / `barbers` / `services` / `bookable_slots` and no `rls_disabled_in_public` for them. The advisor should be **clean** — in particular **no `security_definer_view` ERROR on `barbers_public`**, because the build skill creates it with `security_invoker = on`. If that ERROR appears, the view was created without `security_invoker` — re-apply the Step 3 view DDL. *Recovery:* build skill Steps 3–4, plus the `profiles` bank-field RLS introduced in m0-landing-page.

#### Section G — Sample hairstyle photos (portfolio)
- **G1** The `barber_photos` table exists with the right shape and Barber A has at least one photo row:
  ```sql
  select id, barber_id, storage_path, caption, is_featured, sort_order
  from public.barber_photos order by created_at desc limit 10;
  ```
  Expect rows with a `storage_path` like `<barber_id>/<uuid>.<ext>` and a working `is_featured` flag. (If Barber A uploaded none, upload one in the onboarding form first — the gallery/M4 depend on this.) *Recovery:* build skill Step 5 (the uploader) + Step 2 (the `barber_photos` table).
- **G2** The `barber-photos` **Storage bucket** exists and is **public-read**, with a shop-only write policy:
  ```sql
  select id, public from storage.buckets where id = 'barber-photos';                -- expect public = true
  select policyname, cmd from pg_policies
  where schemaname = 'storage' and tablename = 'objects'
    and policyname like 'barber_photos%';                                            -- read + write_own
  ```
  *Recovery:* build skill Step 3a (create the bucket + Storage policies).
- **G3** **A barber cannot write into another barber's photo folder.** Like D1, this is **behavioral and cannot be proven via the privileged MCP** (which bypasses Storage RLS) — it needs **Barber B signed into the live app**. As **Barber B**, an upload under Barber A's `<barberA_id>/...` prefix must be **rejected** by the Storage policy (the public bucket is read-only to non-shops). Via MCP you can only confirm the policy *definition* — that `barber_photos_write_own` exists and keys on `split_part(name,'/',1)` against an owned barber:
  ```sql
  select policyname, cmd, qual from pg_policies
  where schemaname='storage' and tablename='objects' and policyname='barber_photos_write_own';
  ```
  *Recovery:* the `barber_photos_write_own` Storage policy (Step 3a). *(Offer to create a second test account if the student doesn't have Barber B yet.)*

## Reporting

Emit a table:

| Check | Status | Notes |
|---|---|---|
| A1 barber exists + correct shop | ✅ / ❌ | |
| A2 shop can run MANY barbers (shop_id NOT unique + index) | ✅ / ❌ | a 2nd barber per shop is valid |
| B1 service CRUD (category + whole-unit price) | ✅ / ❌ | `price` in platform_settings.currency, NOT ×100 |
| C1 slot publish (time window — NO status column) | ✅ / ❌ | `id, barber_id, starts_at, ends_at, created_at`; availability derived later |
| D0 `*_write_own` policies scoped to `auth.uid()` (structural, MCP) | ✅ / ❌ | policy text — all MCP can prove |
| D1 RLS denies cross-barber edit (behavioral, live app) | ✅ / ⚠️ / ❌ | **the decisive test — 0 rows changed; ⚠️ if no 2nd account to run it** |
| D2 shop can edit own rows | ✅ / ❌ | RLS not over-blocking |
| E1 role flips to 'shop' | ✅ / ❌ | never `admin` |
| F1 `barbers` has no bank columns (moved to `profiles`) | ✅ / ❌ | bank is shop-level now |
| F2 `profiles` bank fields shop+admin-only + advisor clean | ✅ / ⚠️ / ❌ | get_advisors security |
| G1 `barber_photos` table + a photo row (path + is_featured) | ✅ / ⚠️ / ❌ | portfolio + M4 input |
| G2 `barber-photos` bucket public-read + shop-write policy | ✅ / ❌ | |
| G3 barber can't write into another's photo folder (behavioral, live app) | ✅ / ⚠️ / ❌ | `<barber_id>/` prefix enforced; ⚠️ if no 2nd account to run it |

**Verdict** (milestone-scoped — celebrate M1.1, don't surface the next milestone to the student):
- All ✅ → 「M1.1 驗收通過 ✅ 理髮店這一側完整了：能開（多位）理髮師、列服務、發布可預約時段（時段就是一段時間窗、沒有 status 欄位，可預約與否由有沒有人預約推導），RLS 也擋住了跨理髮師的竄改、店家層級的銀行欄位（在 `profiles`）沒有外洩。」
  > **Note for Claude Code (author-only — do NOT volunteer to the student):** when the instructor advances the student, the next milestone is **M1.2** (`[[m1.2-buyer-setup]]`, the customer browse + pop-up booking flow). Don't print "啟動 M1.2" or describe M1.2 in the success message.
- Any ❌ → list the failed items + the recovery step, and tell the student to fix then re-run `驗收 M1.1`. **If D1 or F2 failed, treat it as blocking** — a cross-barber edit getting through or a leaking bank field is a security hole, not a cosmetic miss; fix the RLS migration (build skill Steps 3–4) before the milestone is considered done. **A D1/G3 that's only ⚠️ (no second account available to run the behavioral test) is NOT a pass** — D0's structural check plus a clean advisor is *necessary but not sufficient*; offer to create the second account and run the real attempt before declaring those green.
