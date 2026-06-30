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
| D — RLS cross-barber denial | two browser sessions + Supabase MCP | **Supabase MCP** + a second test account in the live app |
| E — Role flips to 'shop' | Supabase MCP | **Supabase MCP** |
| F — Shop-level bank fields (profiles) not world-readable | Supabase MCP + `get_advisors` | **Supabase MCP** + `get_advisors` |
| G — Sample hairstyle photos | Supabase MCP + Storage | **Supabase MCP** + Storage |

The reads below run through the **Supabase MCP** (`mcp__claude_ai_Supabase__execute_sql` / `get_advisors`) in both modes — that's the authoritative path. The cross-barber **edit** test (Section D) needs a second logged-in barber in the live app, because RLS only bites under a real `auth.uid()`.

## How to run

The student invokes this directly (e.g. types `驗收 M1.1`). You (Claude Code) **actively run** each check and report results — don't just describe them.

### Step 1: Collect what you need (one message)

Ask the student for:
1. The live Vercel URL (to exercise the UI / create a second barber).
2. Two barber test accounts (or offer to create a second one): **Barber A** (owns at least one barber, has a service + a slot) and **Barber B** (a different account). Section D needs both.
3. Confirm the Supabase MCP is connected to the **barber-platform** project.

### Step 2: Run the checklist

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
- **D1** **Try to edit another barber's barber and confirm it's blocked.** As **Barber B** (logged into the live app, so `auth.uid()` = B), attempt to update **Barber A's** barber / service / slot:
  ```sql
  -- run AS barber B (e.g. via the app, or an MCP call scoped to B's session):
  update public.barbers set name = 'HACKED' where shop_id <> auth.uid();   -- expect: 0 rows
  update public.services set price = 1 where barber_id =
      (select id from public.barbers where shop_id <> auth.uid() limit 1);     -- expect: 0 rows
  update public.bookable_slots set starts_at = now() where barber_id =
      (select id from public.barbers where shop_id <> auth.uid() limit 1);     -- expect: 0 rows
  ```
  **Pass = 0 rows affected** on all three (RLS filtered them out) **and** Barber A's data is unchanged when you re-read it. If any row changes → the `*_write_own` policies are missing/too loose. *Recovery:* re-apply the RLS migration (build skill Step 3) and re-run `get_advisors`.
- **D2** Conversely, Barber B **can** edit B's own barber (RLS shouldn't over-block). A self-update returns 1 row.

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
  Then, as a **customer / different barber** (publishable key, so `auth.uid()` ≠ the shop), a read of **another** user's `profiles.bank_account_*` must return **nothing** — the existing `profiles_select_own` RLS scopes a user to their own row, and only `role='admin'` may read across rows. Confirm no public view re-exposes the bank columns, and `get_advisors` shows no RLS-disabled / leaking finding:
  ```text
  mcp__claude_ai_Supabase__get_advisors  →  type: "security"
  ```
  Expect RLS enabled on `platform_settings` / `profiles` / `barbers` / `services` / `bookable_slots` and no `rls_disabled_in_public` for them. (A `security_definer_view` note on `barbers_public` is expected — it carries only non-sensitive columns; not a leak.) *Recovery:* build skill Steps 3–4, plus the `profiles` bank-field RLS introduced in m0-landing-page.

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
- **G3** **A barber cannot write into another barber's photo folder** — as **Barber B**, an upload under Barber A's `<barberA_id>/...` prefix is rejected by the Storage policy (the public bucket is read-only to non-shops). Confirm the path convention `<barber_id>/...` is enforced. *Recovery:* the `barber_photos_write_own` Storage policy (Step 3a).

## Reporting

Emit a table:

| Check | Status | Notes |
|---|---|---|
| A1 barber exists + correct shop | ✅ / ❌ | |
| A2 shop can run MANY barbers (shop_id NOT unique + index) | ✅ / ❌ | a 2nd barber per shop is valid |
| B1 service CRUD (category + whole-unit price) | ✅ / ❌ | `price` in platform_settings.currency, NOT ×100 |
| C1 slot publish (time window — NO status column) | ✅ / ❌ | `id, barber_id, starts_at, ends_at, created_at`; availability derived in M1.2 |
| D1 RLS denies cross-barber edit | ✅ / ❌ | **the decisive test — 0 rows changed** |
| D2 shop can edit own rows | ✅ / ❌ | RLS not over-blocking |
| E1 role flips to 'shop' | ✅ / ❌ | never `admin` |
| F1 `barbers` has no bank columns (moved to `profiles`) | ✅ / ❌ | bank is shop-level now |
| F2 `profiles` bank fields shop+admin-only + advisor clean | ✅ / ⚠️ / ❌ | get_advisors security |
| G1 `barber_photos` table + a photo row (path + is_featured) | ✅ / ⚠️ / ❌ | portfolio + M4 input |
| G2 `barber-photos` bucket public-read + shop-write policy | ✅ / ❌ | |
| G3 barber can't write into another's photo folder | ✅ / ❌ | `<barber_id>/` prefix enforced |

**Verdict:**
- All ✅ → 「M1.1 驗收通過 ✅ READY for M1.2。理髮店能開（多位）理髮師、列服務、發布可預約時段（時段就是一段時間窗、沒有 status 欄位，可預約與否在 M1.2 由 live booking 推導），RLS 也擋住了跨理髮師的竄改、店家層級的銀行欄位（在 `profiles`）沒有外洩。跟我說『啟動 M1.2』，我們來做顧客瀏覽與彈出視窗預約。」
- Any ❌ → list the failed items + the recovery step, and tell the student to fix then re-run `驗收 M1.1`. **If D1 or F2 failed, treat it as blocking** — a cross-barber edit getting through or a leaking bank field is a security hole, not a cosmetic miss; fix the RLS migration (build skill Steps 3–4) before proceeding to M1.2.
