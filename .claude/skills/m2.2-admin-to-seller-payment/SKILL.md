---
name: m2.2-admin-to-seller-payment
description: 抽成制理髮師預約平台 Milestone 2.2 — the special unit「抽成撥款 + admin 撥款頁」(commission settlement). Build the REPORTING + admin workflow on top of money that is ALREADY in the platform's Stripe balance — no new Stripe work. Create a real-time `owed_bookings` Supabase VIEW (per-paid-booking owed pool: paid bookings WHERE payout_id IS NULL, each with its derived split + shop attribution via bookings→services→barbers→shop_id), a flexible `payouts` settlement TABLE (a free-form BATCH for ONE shop — a shop has MANY payouts over time, shop_id NOT unique, status pending_transfer→transferred / cancelled), an admin-only `/admin/payouts` builder where the admin FILTERS the owed pool and SELECTS which paid bookings to include in a payout (stamping bookings.payout_id), plus row-by-row "Mark as transferred" / "Cancel", and a read-only `/shop/earnings` mirror where each shop sees their own owed vs in-a-payout bookings + status. Settlement is DERIVED from bookings.payout_id (no payout_pending/payout_transferred booking status). Use when the student says "啟動 M2.2", "start M2.2", "做撥款頁", "admin payout page", "抽成結算", "理髮師撥款", or any variant of "讓 admin 看到每間店家還沒撥的款並撥款給他們".
---

# M2.2 —  抽成撥款 + Admin 撥款頁（店家結算與撥款管理）

## What this skill does

This is the **special unit** of the course: **抽成撥款制度** — the commission-settlement workflow that makes the platform a business. Up to now the money has been flowing **into the platform's own Stripe balance** (M2.1: pay-now at booking, the webhook flips the **booking** `pending_payment → paid` and stamps `paid_at`). **M2.2 does NOT move money.** It is **reporting + an admin workflow**: surface every paid booking that hasn't been paid out yet (the **owed pool**), let the admin build a flexible payout **batch for one shop** out of the bookings they pick, record the manual bank transfer, and mirror that status back to the shop.

There is **NO `transactions` table** — that table was dropped from the model. "Money in" is simply a `paid` booking's `price`. A payout sums the bookings the admin picked; bookings carry **no** money-split columns (`platform_fee`/`barber_amount` do not exist).

**The settlement state is DERIVED, not a booking status.** A booking has only **3 states** (`pending_payment` → `paid`, or `→ cancelled`). Whether a paid booking is owed-or-settled is read from its **`payout_id` FK**, not a status: `paid + payout_id NULL` = **OWED**; `paid + payout_id set` = in that payout (read `payouts.status`). There is **no** `payout_pending` / `payout_transferred` booking status — those are gone.

By the end the student has:

1. A **real-time `owed_bookings` Supabase VIEW** — the LIVE owed pool: every `paid` booking **WHERE `payout_id IS NULL`**, each carrying its derived split (`price × the rate in force → platform_cut` / `shop_cut`) and its shop attribution via `bookings → services → barbers → shop_id` (so a shop's many barbers all surface under one `shop_id`). It is **live** — the split is computed in the VIEW from `commission_rates`, not stored per booking. The `/admin/payouts` builder reads this.
2. A flexible **`payouts` settlement TABLE** — a free-form **BATCH for ONE shop**. A shop has **MANY** payouts over time (June, July, a single late-booking top-up…), so **`shop_id` is NOT unique** — the only unique key is the PK `id`. Each row snapshots `shop_name` (a denormalized copy of `profiles.display_name`), `gross`, `platform_pct`, `platform_cut`, `shop_cut`, `bookings_count`, plus `status` (`pending_transfer` → `transferred`, or `→ cancelled`), `note`, `bank_reference`, `created_by`, `marked_transferred_at`, `transferred_by`. The chosen bookings point back via `bookings.payout_id`. One payout = one shop = one bank transfer = one status ([[m1.1-seller-setup]]).
3. **RLS** so only `role='admin'` reads/writes every `payouts` row + every shop's `profiles` bank fields; each shop reads only their **own** earnings.
4. An **admin-only `/admin/payouts` builder** (`role='admin'` gated by middleware **and** RLS): the live **owed list** (from `owed_bookings`) with **filters** (by shop / customer / paid_at range — UI convenience, just WHERE clauses) and a **multi-select → build payout** action (an atomic RPC that verifies all selected bookings resolve to the **same shop**, inserts a `pending_transfer` payout snapshot, and stamps `bookings.payout_id`), **plus** a list of each shop's existing payouts with their status and per-row **「標記為已轉帳 / Mark as transferred」** and **「取消 / Cancel」** actions.
5. A read-only **`/shop/earnings` page** — each shop sees their **own** paid bookings, split into **owed** (`payout_id` NULL) vs **in a payout** (with that payout's status `pending_transfer`/`transferred` + date), combined across all their barbers. A **cancelled** payout's bookings reappear as owed.

**Out of scope for M2.2:** anything that moves money. There is **no Stripe Connect, no payout API, no new Checkout** — the manual bank transfer happens **outside the app** (the admin's online banking), and the admin **records** it here. The custom domain is M3.

## When to load this skill

Trigger phrases:
- "啟動 M2.2" / "start M2.2" / "begin M2.2"
- "做撥款頁" / "admin payout page" / "抽成結算"
- "理髮師撥款" / "店家還沒撥的款有多少"
- "讓 admin 標記已轉帳" / "shop earnings 頁"

Do NOT load this for M2.1 (that's the Stripe金流 — the webhook that flips a booking to `paid` and stamps `paid_at`, which this skill **settles from**) or M3 (custom domain). If there is **no admin user yet**, stop — admin promotion lives in the M2.1 prerequisite, not here (see Step 0).

## Execution mode (Cowork-first)

| Part | Cowork mode | Pure-CLI mode |
|---|---|---|
| Supabase VIEW + TABLE + RLS + the RPCs | **Supabase MCP `apply_migration`** (preferred both modes) | same MCP call, or `supabase db push` with a migration file |
| Front-end pages (`/admin/payouts`, `/shop/earnings`) | Lovable prompt → push to GitHub (token from Secrets Manager) → Vercel auto-deploys | edit code locally, `git push`, Vercel auto-deploys |
| The "Build payout" / "Mark as transferred" / "Cancel" actions | each calls an **atomic RPC** (admin-only) | same |
| Verification | Supabase MCP `execute_sql` + the live Vercel URL | `curl` + SQL editor |

All Supabase changes go through **`apply_migration`** — never a raw ad-hoc `UPDATE` ([[supabase-best-practice]]). There is **no Stripe MCP work in this milestone at all.**

## Architecture

![Barber platform architecture (M2.2 — commission settlement / payout). The shop side (left) and customer side (right) each drive a Vercel-hosted Product Site over the SAME Supabase Database. M2.1's money flow is drawn at the top-right: Stripe's Webhook does a payment check and writes the booking row (customer, service, bookable slot(s), status, price, paid_at, payout), where the greyed customer/service/bookable-slot(s)/payout lines are FKs/joins, not stored split columns — a paid booking carries only its price + paid_at + a NULL payout, and there is NO transactions row. M2.2 adds the admin side (far right) and the payout batch: a bold down-arrow runs from the booking box to a new payout box (profile/shop, month, gross, platform %, platform cut, seller cut, booking count, status) — this is the payouts table, a per-shop batch snapshotting the derived 20/80 split from commission_rates at build time. The admin builds a payout from owed paid bookings (paid + payout NULL, attributed to a shop via booking→service→barber→shop_id), then makes ONE bank transfer: the payout flows through a BANK icon to the shop (a left-pointing arrow payout → BANK → shop), recorded in-app as MARK TRANSFERRED. Each of shop/customer/admin has a profile (email, name, role, bank acct) — the bank acct lives on profile (shop level), readable only by the owning shop + admin. Bottom-left insets: the M0 Cowork/GitHub/Supabase/AWS setup loop, and the M2.1 charge-booking loop (Product Site ⇄ Stripe Checkout API via webhook, then W writes the booking). Legend: green = Supabase data, purple = Vercel-hosted Product Site, yellow = Stripe, grey rows = FK/derived (not stored) fields.](assets/architecture-m2.2.jpg)

The M2 money flow ends here. **Paid bookings** (via Stripe Checkout in M2.1, money landing in the platform's own Stripe balance) carry only their `price` and a `paid_at` — **no per-booking split, no `transactions` row**. In M2.2 a real-time `owed_bookings` VIEW surfaces every `paid` booking whose `payout_id IS NULL` — the **owed pool** — attributing each via `bookings → services → barbers → shop_id` (so a shop's many barbers roll up under one `shop_id`) and computing its derived split from `commission_rates`. The admin opens the `role='admin'`-gated `/admin/payouts` builder, **filters** the owed pool (by shop / customer / paid_at range — UI convenience) and **selects** the paid bookings to pay: a **build-payout RPC** verifies they all resolve to the **same shop**, inserts one `payouts` row (`status='pending_transfer'`, snapshotting `gross`/`platform_pct`/`platform_cut`/`shop_cut`/`bookings_count` and `shop_name = profiles.display_name`), and stamps `bookings.payout_id` on the chosen rows (they leave the owed pool). The admin then makes ONE bank transfer to that shop's `profiles` bank account OUTSIDE the app and records it with **MARK TRANSFERRED** (`pending_transfer → transferred`). A still-`pending_transfer` payout can be **CANCELLED** — its bookings' `payout_id` is nulled (back to owed) and the row is kept as `cancelled`. Each shop's `/shop/earnings` page reads its own paid bookings (owed vs in-a-payout) as a read-only mirror. RLS keeps the shop-level bank fields (on `profiles`) + `payouts` readable only by the owning shop and admin.

How the pieces map to M2.2:
- **paid bookings → `owed_bookings` VIEW:** the VIEW lists each `paid` booking with `payout_id IS NULL` and applies the rate in force to split its `price`, **live**. Nothing is copied or frozen until a payout is built (which then snapshots the totals).
- **VIEW → `/admin/payouts` builder (admin):** the admin filters/selects owed bookings and sees a running total.
- **BUILD PAYOUT → a `payouts` row + booking stamps:** the selected (same-shop) bookings get `payout_id` set and a `pending_transfer` payout snapshots their totals.
- **Admin → bank transfer (OUTSIDE the app) → MARK TRANSFERRED:** the money moves in the admin's online banking; the app only **records** it (`status='transferred'` + stamps). The bookings need **no** status change — settled is derived from `payout_id`.
- **CANCEL (only while pending_transfer):** nulls the bookings' `payout_id` (back to owed) and marks the payout `cancelled` (kept for audit). A `transferred` payout is **immutable**.
- **paid bookings + `payouts.status` → `/shop/earnings` (shop):** a read-only mirror — the shop sees what's still owed vs in a payout and whether it's been transferred.

## Conversational flow

You (Claude Code) drive the student through **6 steps**, in order. Don't dump them all at once — after each step, **wait for confirmation** before moving on.

0. Pre-flight: confirm an **admin user already exists** (from the M2.1 prerequisite) + that there are `paid` bookings
1. Apply the Supabase migration: `owed_bookings` VIEW + `payouts` TABLE + the RPCs (build / mark-transferred / cancel) + RLS
2. Build the **`/admin/payouts`** builder (live owed list + filters + multi-select → build payout)
3. Wire the **admin actions** (Build payout + row-by-row "Mark as transferred" + "Cancel")
4. Build the **`/shop/earnings`** page (read-only mirror: owed vs in-a-payout + status)
5. Verify end-to-end → run the checklist

---

### Step 0 — Pre-flight: the admin user must already exist

This milestone **consumes** `role='admin'` to gate `/admin/payouts`; it does **not** create the admin. Admin promotion was done **once**, via a one-off migration, in the M2.1 prerequisite ([[m2.1-buyer-to-admin-payments]] prereq) — there is no public "sign up as admin" UI, so a user can never self-escalate.

Confirm it exists before building anything:

```sql
-- via Supabase MCP execute_sql
select id, email, role from public.profiles where role = 'admin';
```

> 如果這個查詢**回傳 0 筆**，先停下來：回到 M2.1 的 prerequisite，把你自己的帳號用一支 one-off migration 升級成 `role='admin'`（`UPDATE public.profiles SET role='admin' WHERE email='你的信箱';` 透過 `apply_migration`），登出再登入，然後才回來做 M2.2。沒有 admin，整個撥款頁就無從 gate。

> **Note for Claude Code:** do NOT add an admin sign-up path or a self-promote button here. If the student asks to "create an admin", point them at the M2.1 prerequisite's one-off migration. The whole security model depends on admin being promotion-only.

Also sanity-check that M2.1 actually produced `paid` bookings (the owed pool is meaningless without them — there are **no** fee columns to check, since bookings carry no split):

```sql
select id, status, price, paid_at, payout_id
from public.bookings
where status = 'paid'
order by paid_at desc
limit 5;
```

> 你應該看到幾筆 `status='paid'` 而且 `paid_at` 有值的 booking，且這些剛付款的 booking `payout_id` 都還是 `NULL`（代表「還沒撥款 / 欠款」）。bookings **沒有** `platform_fee`／`barber_amount` 欄位 — 抽成是在這裡（M2.2）建立 payout 時按 `price` 加總後才算出來的。如果完全沒有 `paid` 的 booking，代表 M2.1 的 webhook 沒把 booking 翻成 `paid` — 先回 M2.1 修好，M2.2 才有數字可以撥。

---

### Step 1 — Apply the migration: the `owed_bookings` VIEW + `payouts` TABLE + the RPCs + RLS

Have Claude Code apply this as **one Supabase migration** via `mcp__claude_ai_Supabase__apply_migration` (never a raw console edit — [[supabase-best-practice]]).

Four design facts to keep straight:
- **There is NO `transactions` table.** "Money in" is a `paid` booking's `price`. The VIEW lists `paid` bookings directly — it does not read any ledger or accumulator table.
- **Settlement is DERIVED from `bookings.payout_id`, not a booking status.** bookings have only 3 states (`pending_payment`/`paid`/`cancelled`). A `paid` booking with `payout_id IS NULL` is **owed**; with `payout_id` set it's **in that payout** (read `payouts.status`). There is **no** `payout_pending`/`payout_transferred` booking status.
- **The split is computed live in the VIEW** (`price × the rate in force`), NOT stored per booking. bookings have **no** `platform_fee`/`barber_amount`. Because `shop_cut = price - platform_cut`, the two always sum back to the exact price — no lost unit. The totals are **snapshotted** onto the `payouts` row at build time.
- **Attribution is `bookings → services → barbers → shop_id`** (`service_id` already pins the barber — we do **NOT** go through the slot, and there is **no** `bookings.barber_id`).

```sql
-- ============================================================
-- M2.2 — commission settlement (the flexible payout model)
-- A real-time owed-pool VIEW + a flexible per-shop payout BATCH table
-- + atomic admin RPCs (build a payout, mark transferred, cancel)
-- ============================================================

-- 1) The settlement table. A FLEXIBLE BATCH for ONE shop. shop_id is NOT unique
--    (a shop has MANY payouts over time); the only unique key is the PK id.
--    SNAPSHOTS the rate + amounts + the shop's display_name so a later
--    commission_rates / rename change never alters a settled batch.
--    NOTE: bookings.payout_id references this table, so create it FIRST.
create table if not exists public.payouts (
  id                    uuid primary key default gen_random_uuid(),
  shop_id               uuid not null references public.profiles(id) on delete cascade,  -- the ONE shop. NOT unique.
  shop_name             text,                                       -- DENORMALIZED snapshot of profiles.display_name
  status                text not null default 'pending_transfer'
                          check (status in ('pending_transfer','transferred','cancelled')),
  gross                 integer not null default 0,                 -- snapshot: sum of included bookings' price
  platform_pct          numeric(5,4) not null,                      -- snapshot: the rate applied to this batch
  platform_cut          integer not null default 0,                 -- snapshot: round(gross * platform_pct)
  shop_cut              integer not null default 0,                 -- snapshot: gross - platform_cut (paid to the shop)
  bookings_count        integer not null default 0,                 -- snapshot: how many bookings in the batch
  note                  text,                                       -- free-form (e.g. "June payout", "late booking #123")
  bank_reference        text,                                       -- the bank transfer ref / memo
  created_by            uuid references public.profiles(id),        -- the admin who built it
  created_at            timestamptz not null default now(),
  marked_transferred_at timestamptz,                                -- when the admin recorded the bank transfer
  transferred_by        uuid references public.profiles(id)
);
create index if not exists idx_payouts_shop   on public.payouts(shop_id);
create index if not exists idx_payouts_status on public.payouts(status);

-- bookings.payout_id FK → payouts (a booking belongs to at most one payout).
-- (If bookings was created before payouts, add the FK now; M1.2/canonical already declares it.)
-- alter table public.bookings
--   add constraint bookings_payout_id_fkey foreign key (payout_id) references public.payouts(id);

-- 2) The real-time owed pool. ONE ROW PER PAID BOOKING that is NOT yet in a payout.
--    Attribute each to a shop via bookings → services → barbers → shop_id (NOT through the
--    slot; bookings has no barber_id), and apply the rate in force for that booking
--    (the latest commission_rates row whose effective_from <= the booking's paid date).
create or replace view public.owed_bookings as
select b.id as booking_id, b.price, b.paid_at,
       b.customer_id,
       bar.shop_id, bar.id as barber_id, bar.name as barber_name,
       r.platform_pct,
       round(b.price * r.platform_pct)::int               as platform_cut,
       (b.price - round(b.price * r.platform_pct))::int   as shop_cut
from public.bookings b
join public.services s   on s.id = b.service_id
join public.barbers bar  on bar.id = s.barber_id
cross join lateral (
  select platform_pct from public.commission_rates
  where effective_from <= coalesce(b.paid_at::date, current_date)
  order by effective_from desc limit 1
) r
where b.status = 'paid' and b.payout_id is null;   -- the OWED pool

-- 3) RLS — the settlement table is admin-or-shop only.
alter table public.payouts enable row level security;

-- helper: is the current user an admin?
create or replace function public.is_admin()
returns boolean language sql stable security definer set search_path = public as $$
  select exists (
    select 1 from public.profiles p
    where p.id = auth.uid() and p.role = 'admin'
  );
$$;

-- admin can read every payout; a shop can read only their OWN rows
create policy "payouts_select_admin_or_owner" on public.payouts
  for select using (
    public.is_admin()
    or shop_id = auth.uid()
  );

-- only an admin may write payouts (all admin actions run as the admin / service-role)
create policy "payouts_insert_admin" on public.payouts
  for insert with check (public.is_admin());
create policy "payouts_update_admin" on public.payouts
  for update using (public.is_admin());

-- ============================================================
-- ADMIN ACTION A) BUILD A PAYOUT — turn a selection of owed bookings (ONE shop) into a batch.
--   0. verify EVERY selected booking is paid, not yet in a payout, AND resolves to the SAME shop
--      (via service → barber → shop). Reject a mixed-shop or already-paid-out selection.
--   1. INSERT a payouts row (status='pending_transfer', snapshots, shop_name = profiles.display_name).
--   2. UPDATE the selected bookings: set payout_id = the new payout
--      (guarded by status='paid' AND payout_id IS NULL → no double-pay).
-- Atomic. Admin-only.
-- ============================================================
create or replace function public.build_payout(p_booking_ids uuid[], p_note text default null)
returns uuid language plpgsql security definer set search_path = public as $$
declare
  v_shop_id uuid;
  v_shop_count integer;
  v_shop_name text;
  v_rate numeric(5,4);
  v_gross integer;
  v_count integer;
  v_platform_cut integer;
  v_payout_id uuid;
begin
  if not public.is_admin() then
    raise exception 'admin only';
  end if;

  -- which distinct shops do the selected (still-owed) bookings resolve to?
  select count(distinct bar.shop_id), min(bar.shop_id), coalesce(sum(b.price),0), count(*)
    into v_shop_count, v_shop_id, v_gross, v_count
  from public.bookings b
  join public.services s  on s.id = b.service_id
  join public.barbers bar on bar.id = s.barber_id
  where b.id = any(p_booking_ids)
    and b.status = 'paid' and b.payout_id is null;   -- only still-owed bookings count

  if v_count = 0 then
    raise exception 'no owed bookings in selection';
  end if;
  if v_shop_count <> 1 then
    raise exception 'all selected bookings must belong to the SAME shop (got % shops)', v_shop_count;
  end if;

  -- the shop's display_name snapshot + the rate in force (use the current rate at build time)
  select display_name into v_shop_name from public.profiles where id = v_shop_id;
  select platform_pct into v_rate
  from public.commission_rates
  where effective_from <= current_date
  order by effective_from desc limit 1;

  v_platform_cut := round(v_gross * v_rate)::int;

  insert into public.payouts
    (shop_id, shop_name, status, gross, platform_pct, platform_cut, shop_cut, bookings_count, note, created_by)
  values
    (v_shop_id, v_shop_name, 'pending_transfer', v_gross, v_rate,
     v_platform_cut, (v_gross - v_platform_cut), v_count, p_note, auth.uid())
  returning id into v_payout_id;

  -- stamp the chosen bookings (re-guard so a concurrent build can't double-grab one)
  update public.bookings
  set payout_id = v_payout_id, updated_at = now()
  where id = any(p_booking_ids)
    and status = 'paid' and payout_id is null;

  return v_payout_id;
end;
$$;

-- ============================================================
-- ADMIN ACTION B) MARK TRANSFERRED — record the bank transfer for one pending payout.
--   * UPDATE the payout: status 'pending_transfer' → 'transferred',
--     marked_transferred_at/transferred_by/bank_reference.
--   * The included bookings need NO status change — "settled" is derived from payout_id.
-- Atomic. Admin-only.
-- ============================================================
create or replace function public.mark_payout_transferred(p_payout_id uuid, p_bank_reference text default null)
returns void language plpgsql security definer set search_path = public as $$
begin
  if not public.is_admin() then
    raise exception 'admin only';
  end if;

  update public.payouts
  set status = 'transferred',
      marked_transferred_at = now(),
      transferred_by = auth.uid(),
      bank_reference = coalesce(p_bank_reference, bank_reference)
  where id = p_payout_id and status = 'pending_transfer';

  if not found then
    raise exception 'payout not found or not pending_transfer (a transferred/cancelled payout is immutable)';
  end if;
end;
$$;

-- ============================================================
-- ADMIN ACTION C) CANCEL a payout — only while 'pending_transfer' (no transfer happened yet).
--   1. UPDATE its bookings: payout_id → NULL (they revert to OWED; their status stays 'paid').
--   2. UPDATE the payout: status → 'cancelled' (KEEP the row for audit — never delete).
-- A 'transferred' payout is IMMUTABLE — cannot be cancelled.
-- Atomic. Admin-only.
-- ============================================================
create or replace function public.cancel_payout(p_payout_id uuid)
returns void language plpgsql security definer set search_path = public as $$
begin
  if not public.is_admin() then
    raise exception 'admin only';
  end if;

  -- null the link FIRST so the bookings fall back into the owed pool
  update public.bookings
  set payout_id = null, updated_at = now()
  where payout_id = p_payout_id;

  update public.payouts
  set status = 'cancelled'
  where id = p_payout_id and status = 'pending_transfer';

  if not found then
    raise exception 'payout not found or not pending_transfer (a transferred payout is immutable)';
  end if;
end;
$$;
```

> **Note for Claude Code:** the `owed_bookings` VIEW lists each owed booking and attributes it to a **shop** (via `bookings → services → barbers → shop_id`), so the builder can group/filter a shop's many barbers under one `shop_id`. The VIEW inherits RLS from its base table `bookings`; make sure `bookings` has shop/admin-scoped RLS from M1.2/M2.1 so a shop querying it sees only **their own** bookings and the admin sees all. The **bank account name/number** live on **`profiles`** (M0/M1.1, shop level) and are RLS-restricted to shop + admin — re-confirm with `get_advisors` after this migration. All three admin actions are **RPCs** (`security definer`, admin-guarded) so each is **atomic** — the payout row and the booking `payout_id` stamps commit together or not at all. The **same-shop guard** in `build_payout` is the integrity backbone: it rejects a mixed-shop selection, and the `payout_id IS NULL` re-guard on the UPDATE is the no-double-pay guarantee.

**Verify the VIEW returns live owed rows + the split sums back:**

```sql
select booking_id, shop_id, price, platform_pct, platform_cut, shop_cut,
       (platform_cut + shop_cut) as recomputed_price
from public.owed_bookings
order by paid_at desc;
```

You should see one row per **owed** `paid` booking (those with `payout_id IS NULL`), each with `platform_cut + shop_cut = price` (the split sums back exactly because `shop_cut = price - platform_cut`). A shop's "what am I owed?" = `sum(shop_cut)` over its `owed_bookings` rows.

---

### Step 2 — Build the `/admin/payouts` builder (live owed list + filters + multi-select → build payout)

Paste this into Lovable (or have Claude Code edit the code directly and push). The page is **gated to `role='admin'` in middleware AND by RLS** — both, so a non-admin can't reach the route and couldn't read the data even if they did.

> Build an **admin-only** payout builder + ledger page at **`/admin/payouts`**.
>
> **Access gate:** this route is for `role='admin'` only. In the app's middleware, redirect any non-admin (signed-out, customer, or shop) away from `/admin/*` to `/login` or a 403. Also rely on Supabase RLS as the real enforcement — the page reads `payouts` and the bank fields, which only an admin can read.
>
> **Layout — two parts:**
>
> **Part 1 — The owed-pool builder (the live owed list):**
> 1. A **filter bar**: by **shop** (a dropdown of shops that have owed bookings), by **customer**, and by a **paid_at date range**. These are UI convenience only — just WHERE clauses on the live owed list, NOT a schema grouping.
> 2. A **table of owed bookings** (read from the `owed_bookings` VIEW), one row per **paid booking that is not yet in any payout**, columns: a select checkbox, shop, barber, customer, paid date, price, platform cut, **shop cut**. Show a **running total of the checked rows** (gross / platform cut / shop cut).
> 3. A **「建立撥款 / Build payout」** button (calls the build action — Step 3) over the checked bookings. The selected bookings must **all be one shop** (the RPC enforces it; the UI should ideally only let you select within one shop, or surface the same-shop error). Optionally a free-form **note**.
> 4. A convenience **「全選此店本月欠款 / Select all owed for this shop this month」** helper that pre-checks the matching rows for the chosen shop+month — the admin can still add/remove any before building. It's a UI shortcut, **not** a separate data path.
>
> **Part 2 — The payouts ledger (existing batches):**
> A **table of existing `payouts`** (per shop), columns: shop name, created date, # bookings, gross, platform cut, **shop cut**, **status** badge — 「待轉帳 / Pending」 (`pending_transfer`) · 「已轉帳 / Transferred」 (+ date) (`transferred`) · 「已取消 / Cancelled」 (`cancelled`) — plus per-row actions:
>    - **「標記為已轉帳 / Mark as transferred」** (enabled only when `status='pending_transfer'`)
>    - **「取消 / Cancel」** (enabled only when `status='pending_transfer'`; a `transferred` payout is immutable — no cancel)
>
> **Data:** read the `owed_bookings` VIEW for Part 1 (joined to `profiles` for shop display name + to `barbers` for the barber name), and the `payouts` table for Part 2 (joined to `profiles` for the shop's bank fields so the admin can do the transfer). A `cancelled` payout's bookings have reappeared in `owed_bookings` automatically (their `payout_id` was nulled).
>
> All amounts are **whole units** in the platform currency (`platform_settings.currency` — zero-decimal here, no cents, no ×100). Show them as e.g. `NT$12,000`.

> **Note for Claude Code:** the builder's running total must be computed from the **checked rows** themselves so it always matches the build. Amounts are whole-integer money in `platform_settings.currency` (no `_twd` suffix anywhere, no cents) — never divide by 100. The owed list is **live** — after a payout is built, its bookings drop off the owed list (their `payout_id` is set); after a payout is cancelled, its bookings reappear (their `payout_id` was nulled). Don't filter the owed list on a booking `status` other than `paid` — settlement is derived from `payout_id IS NULL`, which the VIEW already encodes. Exclude `cancelled` payouts from any "what's still owed / earned" math (their bookings are already back in the owed pool).

**Verify:** log in as the admin → `/admin/payouts` loads, the owed list shows paid bookings with `payout_id IS NULL`, filtering by shop/customer/date narrows it, and the running total matches the checked rows. Log in as a **customer or shop** → `/admin/payouts` redirects/403s.

---

### Step 3 — Wire the admin actions (Build payout + Mark as transferred + Cancel)

There are **three** distinct admin actions on `/admin/payouts`, each backed by an **atomic RPC** from Step 1.

**Action A — 建立撥款 / Build payout** (over the checked owed bookings). This is what turns a selection of owed bookings into a committed `pending_transfer` payout for **one shop**.

> Wire the **「建立撥款 / Build payout」** button to call the **`build_payout(p_booking_ids, p_note)`** RPC with the array of checked booking ids (+ an optional note).
>
> The RPC:
> - verifies every selected booking is `paid`, still `payout_id IS NULL`, and resolves to the **SAME shop** (via service → barber → shop) — it **rejects a mixed-shop selection** and a selection with no owed bookings.
> - INSERTs a `payouts` row (`status='pending_transfer'`, snapshot `gross = sum(price)`, `platform_pct = current rate`, `platform_cut = round(gross×pct)`, `shop_cut = gross − platform_cut`, `bookings_count`, and **`shop_name` = the shop's `profiles.display_name`**, `created_by = admin`).
> - UPDATEs the selected bookings: `payout_id = the new payout` (guarded by `status='paid' AND payout_id IS NULL` → no double-pay).
>
> After it returns, re-query: those bookings drop off the owed list and a new `pending_transfer` payout appears in the ledger with its "Mark as transferred" / "Cancel" buttons enabled.

**Action B — 標記為已轉帳 / Mark as transferred** (per-row button on a `pending_transfer` payout). The admin does the **actual bank transfer in their own online banking** (outside the app) — **one transfer per shop**, to that shop's `profiles.bank_account_*` — then clicks **「標記為已轉帳」** on that payout to record it. This is the **source of truth** for "has this shop been paid".

> Wire the per-row **「標記為已轉帳 / Mark as transferred」** button (enabled only when `status='pending_transfer'`) to call the **`mark_payout_transferred(p_payout_id, p_bank_reference)`** RPC with that payout's id.
>
> The RPC atomically UPDATEs the payout: `status` `pending_transfer → transferred`, `marked_transferred_at = now()`, `transferred_by = auth.uid()`, `bank_reference = <optional>`. **The included bookings need NO status change** — they're already linked via `payout_id`; "settled" is derived. Optionally prompt the admin for a bank-transfer reference / memo (allow blank). After the call, the row's badge flips to 「已轉帳 / Transferred」 with the date and the button disables. Only an admin may call this (the RPC is `security definer` + raises on a non-admin, and its `where ... and status='pending_transfer'` guard makes a re-click a safe no-op — a `transferred` payout is immutable).

**Action C — 取消 / Cancel** (per-row button, only while `pending_transfer`). If a batch was built by mistake (wrong bookings, wrong shop), the admin cancels it **before** transferring — its bookings flow back into the owed pool.

> Wire the per-row **「取消 / Cancel」** button (enabled only when `status='pending_transfer'`) to call the **`cancel_payout(p_payout_id)`** RPC with that payout's id.
>
> The RPC atomically:
> - UPDATEs that payout's bookings: `payout_id → NULL` (they revert to **owed** — `status` stays `paid`, they reappear in `owed_bookings`).
> - UPDATEs the payout: `status → 'cancelled'` (**KEEP** the row for audit — never delete it).
>
> A **`transferred` payout is IMMUTABLE** — the money already left the bank; the RPC's `where ... and status='pending_transfer'` guard rejects cancelling it. After the call, the row's badge flips to 「已取消 / Cancelled」 and its bookings reappear in the owed list, ready to be re-batched.

> **Note for Claude Code:** all three actions are **server-side RPCs** (run with the admin's RLS-authorized session or the service-role on a server route) — never expose a way for a non-admin client to write `payouts` or stamp `payout_id`. `build_payout` **snapshots** `gross`/`platform_pct`/`platform_cut`/`shop_cut`/`bookings_count` + `shop_name` onto the payout row, so a later `commission_rates` change or a shop rename never alters a built batch (the live `owed_bookings` VIEW stays current, but the settled payout is frozen). Settlement state is **derived from `bookings.payout_id`**, so mark-transferred touches **only** the payout (no booking status flip), and cancel **nulls** the bookings' `payout_id` (they revert to owed; their booking status stays `paid` throughout). There is **no** `payout_pending`/`payout_transferred` booking status anywhere.

**Verify:**
```sql
-- after building a payout:
select id, shop_id, shop_name, status, gross, platform_cut, shop_cut, bookings_count, created_by
from public.payouts order by created_at desc;
-- the batched bookings now carry the link (and dropped off owed_bookings):
select count(*) from public.bookings where payout_id is not null and status='paid';
-- after marking one payout transferred (bookings UNCHANGED — still 'paid'):
select id, shop_name, status, marked_transferred_at, transferred_by, bank_reference
from public.payouts where status='transferred' order by marked_transferred_at desc;
-- after cancelling a pending_transfer payout (its bookings' payout_id nulled → back to owed):
select id, status from public.payouts where status='cancelled';
select count(*) from public.bookings b
where b.status='paid' and b.payout_id is null;   -- the cancelled batch's bookings are owed again
```
After building you see a `pending_transfer` payout and its bookings carry `payout_id` (and have left `owed_bookings`). After mark-transferred the payout reads `transferred` (with a timestamp + the admin's id) and **its bookings are unchanged — still `paid`** (settled is derived from `payout_id`). After cancel the payout reads `cancelled` (row kept) and its bookings have `payout_id = NULL` again (back in the owed pool, still `paid`).

---

### Step 4 — Build the `/shop/earnings` page (read-only mirror: owed vs in-a-payout + status)

This is the **shop's** view of the same numbers — read-only, scoped to themselves, combined across all their barbers.

> Build a **`/shop/earnings`** page for a signed-in **shop** (their own earnings — all their barbers combined).
>
> Show the shop's own **paid bookings**, split into two groups:
> - **尚未撥款 / Owed** — their `paid` bookings with `payout_id IS NULL` (read from `owed_bookings` filtered to their `shop_id`): list/total the **shop cut** they're still owed.
> - **已納入撥款 / In a payout** — their `paid` bookings whose `payout_id` is set: group by payout and show that **payout's status** (「待轉帳 / Pending」 `pending_transfer` · 「已轉帳 / Transferred」 + the transfer date `transferred`) and its `shop_cut`. A **cancelled** payout's bookings are **not** shown here — they've reverted to **owed** (their `payout_id` was nulled), so they appear in the 尚未撥款 group again.
>
> A small summary line: 本月 / 累計 預約數、總額、平台抽成、**你的收入 / Your earnings**.
>
> **Data:** read the shop's own paid bookings (filtered to `shop_id = auth.uid()` via service → barber), joined to `payouts` for the in-a-payout group's status + date. This page is **read-only** — a shop can never build a payout or mark their own payout transferred; only the admin can (RLS + the admin-guarded RPCs). Add a small note: 「平台會分批結算並撥款給你 / We settle and pay out your earnings in batches.」 (Optionally also show a per-barber breakdown for transparency, but the payout itself is one combined number per batch.)
>
> Amounts are whole units in the platform currency (zero-decimal here).

> **Note for Claude Code:** this page and `/admin/payouts` read the **same** underlying data — `/shop/earnings` is a strict read-only mirror filtered to `shop_id = auth.uid()` (so it already sums all the shop's barbers). RLS guarantees a shop sees only their own bookings/payouts and **cannot** read another shop's bank fields or earnings. Settlement state on this page is **derived from `payout_id`** + the joined `payouts.status` — there is no `payout_pending`/`payout_transferred` booking status to read. A worth-showing nuance: a booking in a `pending_transfer` payout is built but **not yet transferred** — it's no longer in the live owed pool, but the money hasn't moved; if that payout is later **cancelled**, the booking returns to the owed group automatically.

**Verify:** as a shop one of whose payouts the admin marked transferred in Step 3 → `/shop/earnings` shows that group as 「已轉帳 / Transferred」 + the date. A payout still `pending_transfer` shows 「待轉帳 / Pending」. Bookings not yet in any payout show under 尚未撥款 / Owed. After the admin **cancels** a pending payout, its bookings move back to the owed group. A shop cannot see any other shop's numbers.

---

### Step 5 — Run the checklist

> **ask:** "Run the `m2.2-admin-to-seller-payment-checklist` skill."

It verifies the owed-pool math (`price × rate = platform_cut + shop_cut`, sums back) per booking with per-shop attribution via `service → barber` (not slot, not `barber_id`), that **only `role='admin'` reaches `/admin/payouts`** (the decisive test), that **building a payout** flips the selected bookings' `payout_id` + creates a `pending_transfer` payout with the snapshot, that the **same-shop guard** rejects a mixed-shop selection, that a booking already in a payout **can't be re-grabbed** (no double-pay), that **mark-transferred** flips the payout to `transferred` (bookings unchanged), that **cancel** nulls the bookings' `payout_id` + sets the payout `cancelled` (booking status stays `paid`), that a **`transferred` payout can't be cancelled**, that the shop sees status on `/shop/earnings`, that `payouts` carries the `shop_name` snapshot, that there is **no `transactions` table** and **no fee columns on bookings**, and that bank fields aren't readable by a non-shop / non-admin.

---

## Things to watch out for (common mistakes)

1. **Reaching for a `transactions` table.** There is **none** — it was dropped. "Money in" is a `paid` booking's `price`. The VIEW (and `build_payout`) sum `bookings.price` directly. If you find yourself joining a `transactions` table, you're on the old model.
2. **Adding fee columns to bookings.** bookings have **no** `platform_fee`/`barber_amount`. The split is computed in the `owed_bookings` VIEW (price × rate) and snapshotted onto the `payouts` row at build time — never stored per booking.
3. **Inventing a `payout_pending`/`payout_transferred` booking status.** bookings have **3 states only** (`pending_payment`/`paid`/`cancelled`). Settlement is **derived from `payout_id`**: `paid + payout_id NULL` = owed; `paid + payout_id set` = in that payout (read `payouts.status`). Never flip a booking's status at payout time.
4. **Re-multiplying drift / rounding direction.** The split is `platform_cut = round(price * rate)`, `shop_cut = price - platform_cut` — so the two **always sum back to the exact price**, no lost unit. Money is whole-integer in `platform_settings.currency` (no cents, no `_twd` suffix); never divide or multiply by 100 ([[supabase-best-practice]]).
5. **Letting a payout mix shops.** A payout pays **one shop** (one bank transfer, one status). `build_payout` **must** verify every selected booking resolves to the same shop (via service → barber → shop) and reject a mixed-shop selection. Never split a shop into per-barber payouts, and never go through the slot for attribution.
6. **Double-paying a booking.** A booking belongs to **at most one** payout (the single-valued `payout_id` FK). `build_payout`'s `WHERE status='paid' AND payout_id IS NULL` guard means a booking already in another payout can't be grabbed. Don't rely on a UNIQUE on payouts for this — the guarantee is on `bookings.payout_id`.
7. **Admin gate in the UI only.** Hiding the menu link is not security. The gate is **middleware + RLS** — `payouts` and the bank fields are admin-or-shop-readable at the database, and all admin actions are admin-guarded RPCs. Test by hitting `/admin/payouts` as a logged-in **customer** and confirming a denial.
8. **Bank fields leaking.** The bank account name/number live on **`profiles`** (shop level) and are readable **only by the owning shop + admin** (M0/M1.1 RLS) — `barbers` has no bank columns. A customer or another shop must never see them — re-confirm with `get_advisors`.
9. **Trying to move money.** There is **no Stripe payout / Connect / transfer** in this course. The money is already in the platform's balance; the bank transfer happens in the admin's online banking and is only **recorded** here (MARK TRANSFERRED).
10. **Forgetting the cancel direction.** CANCEL **nulls** the bookings' `payout_id` (they revert to owed; their booking status stays `paid`) **and** sets the payout `cancelled` — **keep** the row for audit, never delete it. Only a `pending_transfer` payout can be cancelled; a `transferred` payout is **immutable** (the money already left). Every owed/earnings read must **exclude** `cancelled` payouts.
11. **Mark-transferred touching the bookings.** Mark-transferred flips **only the payout** (`pending_transfer → transferred` + stamps). The bookings are already linked via `payout_id` and need **no** change — "settled" is derived. (Contrast the old model, which flipped a booking status — that's gone.)
12. **Letting a shop write their own status.** `/shop/earnings` is strictly read-only. Only the admin can build a payout, mark transferred, or cancel (admin-guarded RPCs + RLS) — a shop must never be able to settle or mark themselves transferred.
13. **Treating the filters as a schema grouping.** The owed-list filters (shop / customer / paid_at range) and the "select all owed for this shop this month" helper are **UI convenience only** — just WHERE clauses + a pre-check, NOT a `(shop, month)` grouping baked into the data. The payout is a free-form batch; the admin can pick any owed bookings of one shop (a partial month, a single late booking, several months).

## Expected duration

30–50 minutes. Most of it is the two Lovable page builds (admin builder + shop earnings) and the one migration (VIEW + table + three RPCs); there is **no Stripe work**, so it's lighter than M2.1.

## Next step

When `m2.2-admin-to-seller-payment-checklist` is green, tell the student (milestone-scoped — celebrate M2.2, don't surface the next milestone):
「M2.2 完成了！抽成撥款制度上線了 — admin 在 `/admin/payouts` 從『欠款池』（還沒撥款的 paid bookings）勾選某一間店家的預約，按『建立撥款』就會產生一筆待轉帳（pending_transfer）的 payout，並把這些 booking 的 `payout_id` 蓋上（離開欠款池）；轉帳後按『標記為已轉帳』，payout 變 `transferred`（booking 不用改狀態，撥款與否是看 `payout_id` 推導的）；建錯了還能在轉帳前『取消』，booking 的 `payout_id` 被清空、退回欠款池。店家在 `/shop/earnings` 立刻看到自己哪些還沒撥、哪些已納入撥款及其狀態。整個預約 → 付款 → 撥款的閉環都通了 ✅」

> **Note for Claude Code (author-only — do NOT volunteer to the student):** the next milestone attaches the student's own custom domain to the Vercel deploy, with DNS records created in AWS Route 53 (the AWS access is already connected — no new setup). When the instructor advances the student (or the student asks "what's next"), run `m3-custom-domain-prerequisites` first (a lightweight carryover check that the Vercel deploy is green and `call_aws` can reach Route 53), then load `[[m3-custom-domain]]`. Don't seed the domain/go-live steps into the student's mental model at the end of M2.2.

## Reference

- Supabase views: https://supabase.com/docs/guides/database/tables#views
- Supabase RLS: https://supabase.com/docs/guides/auth/row-level-security
- Supabase migrations (`apply_migration`): https://supabase.com/docs/guides/deployment/database-migrations
- Postgres arrays / `any()`: https://www.postgresql.org/docs/current/functions-array.html
- Cross-refs: [[m2.1-buyer-to-admin-payments]] · [[supabase-best-practice]] · [[m3-custom-domain]]
