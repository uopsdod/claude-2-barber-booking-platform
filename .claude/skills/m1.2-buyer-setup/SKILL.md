---
name: m1.2-buyer-setup
description: 抽成制理髮師預約平台 Milestone 1.2 — build the BUYER (customer) booking flow on top of M1.1's barbers/services/30-min slots. Adds a `bookings` table (3-state status `pending_payment`→`paid`→`cancelled`, with a `price` snapshot — NO `start_slot_id`: the booking's start time is DERIVED as MIN(starts_at) over its slots via the `bookings_with_start` view) plus a `booking_slots(booking_id, slot_id)` join table via a Supabase migration. A booking spans N consecutive slots (N = service.required_slots); bookings has NO money split columns (the split is settled at month-end, M2.2) and settlement is DERIVED from `bookings.payout_id` (NULL in M1.2), NOT a booking status. The booking lifecycle lives ONLY on `bookings.status` — slots have NO status; a slot's availability is DERIVED via a NOT EXISTS anti-join against `booking_slots` (free unless a booking_slots row references it), and a `UNIQUE(slot_id)` on booking_slots is the no-double-book guard; cancelling a booking DELETEs its booking_slots rows (freeing the slots). In M1.2 a booking ends at `status='pending_payment'` (NO payment — Stripe is M2.1) and that is a COMPLETE booking. Builds the buyer pages `/barbers` (browse all barbers + search/filter), `/barbers/[id]` (detail — services + available slots + a Book pop-up dialog), and `/bookings` (the customer's my-bookings). NO payment yet (Stripe is M2.1). Use when the student says "啟動 M1.2", "start M1.2", "做買家預約功能", "讓客人可以預約", "build the booking flow", or any variant of "客人瀏覽理髮師並預約一個時段".
---

# M1.2 — 買家預約功能（讓客人瀏覽理髮師並預約一個時段，還不收錢）

## What this skill does

Builds the **customer-facing booking flow** on top of M1.1's barbers / services / **30-min** schedule slots. A customer browses every barber, opens one barber's detail page, clicks **Book**, picks a service + date + start slot inside a **pop-up dialog**, and a `pending_payment` booking spanning **N consecutive slots** (N = service.required_slots) is created — one booking, N rows in a `booking_slots` join table. A slot has **no status** — its availability is **derived**: a slot simply disappears from the available list because a `booking_slots` row now references it (a NOT EXISTS anti-join against `booking_slots`). The customer then sees their booking on `/bookings`. **No money changes hands yet** — payment (Stripe Checkout) is wired into the very same dialog's confirm action in M2.1. In M1.2 a booking ends at `status='pending_payment'` (with `paid_at` NULL) and **that is a complete, valid booking** — the slot HOLD is payment-independent (it keys off the `booking_slots` rows existing, not off any paid status).

By the end the student has:

1. A **`bookings` table** (created via a Supabase migration — never a raw console edit, per [[supabase-best-practice]]): `customer_id`, `service_id`, `status` (a **3-state** machine: `pending_payment → paid`, or `→ cancelled`; **in M1.2 `pending_payment` is the TERMINAL state**), a `price` **snapshot** taken at booking time, a `paid_at` column (**NULL in M1.2** — stamped by the M2.1 webhook), and a `payout_id` FK → `payouts` (**NULL in M1.2** — set at month-end, M2.2). **NO `start_slot_id` column** — the booking's slots (including the first) live **only** in `booking_slots`; the start time is **DERIVED** as `MIN(starts_at)` over those slots, exposed via the `bookings_with_start` view. **NO money-split columns** (`platform_fee`/`barber_amount` are gone — the split is derived at settlement, M2.2) and **NO `barber_id` column** — the barber is reached transitively via `bookings → services(barber_id) → barbers(shop_id)`. **Settlement state is DERIVED from `payout_id`** (NULL = paid-but-not-yet-paid-out), **not a booking status** — there are no `payout_*` statuses on the booking. The booking lifecycle lives **only** on `bookings.status`; creating a booking does **not** flip any slot status (slots have none).
2. A **`booking_slots(booking_id, slot_id)` join table** — a booking spans **N consecutive slots** (N = service.required_slots), one `booking_slots` row per slot (**all N, including the first** — there is no `start_slot_id` on the booking; the start is `MIN(starts_at)` over these rows). The **booking lifecycle** lives on `bookings.status` (3 states): `pending_payment → paid` (the `paid` transition arrives in M2.1 via the Stripe webhook — **whoever pays first wins the slots**) or `pending_payment → cancelled`. A SLOT has **no status**: its availability is **derived** — a slot is bookable ⟺ NOT EXISTS a `booking_slots` row referencing it. A **`UNIQUE(slot_id)` on `booking_slots`** is the DB-level no-double-book guard (a slot can be in at most one booking's hold), and **cancelling a booking DELETEs its `booking_slots` rows** so the slots free up — which is what makes the plain `UNIQUE` mean "one *live* booking per slot". The hold is **payment-independent**: a `pending_payment` booking holds its slots exactly like a `paid` one, because both the anti-join and the `UNIQUE(slot_id)` key off the `booking_slots` row EXISTING, not off `status`.
3. **RLS** so a customer reads/writes **only their own** bookings (and the `booking_slots` rows of their own bookings), and a **shop** reads bookings for its barbers' services via the join through `services → barbers`. `bookable_slots` and `booking_slots` stay **public-select**, so the buyer's available-slots anti-join can read them.
4. A **`/barbers` browse page** — a **marketplace-style responsive card grid** (navbar + a wall of product-style cards) where each card shows a rep photo from `barber_photos`, with search + a category filter (cut / color / perm / beard).
5. A **`/barbers/[id]` detail page** — laid out like a **marketplace product-detail page**: a top **hairstyle photo carousel** (main image + prev/next + thumbnail/dot nav + click-to-zoom), then the barber profile info + services + **available** slots + a **Book** button. (`[id]` is a uuid/int barber id, so this public detail route coexists fine with the barber-management routes `/shop/bookings` and `/shop/earnings`.)
6. The **booking pop-up dialog UX** — Book opens a MODAL (**service first → date → slot**; price shown before confirm), the customer **stays on `/barbers/[id]`**, and on confirm it creates the `pending_payment` booking, closes the dialog, and **returns to `/barbers/[id]` with a toast**. No full-page navigation.
7. A **`/bookings` page** — the customer's personal "my bookings" list with status.

**Out of scope for M1.2:** Stripe Checkout and any real payment; the `paid` transition (and `paid_at` being stamped); the month-end split (M2.2); the success page `/bookings/success`; the admin payout page. Those are M2.1 and M2.2. In M1.2 a booking ends at `pending_payment` with `paid_at=NULL` and `payout_id=NULL` — and that is complete.

## When to load this skill

Trigger phrases:
- "啟動 M1.2" / "start M1.2" / "begin M1.2"
- "做買家預約功能" / "讓客人可以預約" / "客人預約一個時段"
- "build the buyer booking flow" / "build the booking flow"
- Any prompt mapping to "客人瀏覽理髮師、開詳細頁、按 Book、選日期時段、建立一筆 pending 預約"

Run **`m1.2-buyer-setup-prerequisite` first** — it confirms M1.1's seller data is real and correctly shaped (`barbers` / `services` with `required_slots` / `bookable_slots` with **no `status` column** / `platform_settings`), that at least one barber is actually bookable (a service + ≥ `required_slots` future free slots), and that `bookings`/`booking_slots` are a clean slate. It does **not** require a customer account to pre-exist — you sign up the `role='customer'` account you book as as the **first act of this build** (M1.2 is where the customer side is first built), so that's a build step here, not a prerequisite.

Do NOT load this for the **barber** side (publishing barbers/services/slots) — that's M1.1 ([[m1.1-seller-setup]]). Do NOT load this to wire **payment** — that's M2.1 ([[m2.1-buyer-to-admin-payments]]).

## Execution mode (Cowork-first)

This milestone runs in **Cowork on Desktop**, same as M0/M1.1. The two backend touch-points use the **Supabase Connector / MCP**:

| Task | Cowork mode | Pure-CLI mode |
|---|---|---|
| Apply the `bookings` migration + RLS | Supabase MCP `apply_migration` | same MCP call, or `supabase db push` against your migration file |
| Verify a booking row / slot availability | Supabase MCP `execute_sql` | `psql` / Supabase SQL editor |
| Build the buyer pages + dialog | Claude Code edits the repo, pushes to `main` (recall the GitHub PAT from Secrets Manager `barber-project/github`) | `git push` with a locally-authed `gh` |

All Supabase changes go through a **migration file / `apply_migration`** — never a raw ad-hoc `UPDATE`/`INSERT` against prod ([[supabase-best-practice]]).

## Architecture

![Barber platform architecture (M1.2 — buyer booking) — a shop (left) and a customer (right) each drive a Vercel-hosted Product Site over the SAME Supabase Database. M1.1's shop side (left) owns profile (email/name/role/bank acct), the many barbers (profile/shop, name, intro, address) with their style photos + services (price, required slots #) + bookable slots (start time / end time). M1.2 adds the customer side (right): /barbers browses every barber, /barbers/[id] shows one barber's services + available slots, and a Book pop-up dialog writes a booking. The new booking box lists customer, service, bookable slot(s), status, price, paid_at, payout — where the greyed "bookable slot(s)" line is NOT a column on bookings: it is the UNSHOWN booking_slots(booking_id, slot_id) JOIN TABLE (a row per held slot, FK→booking id + FK→bookable slot id), which is why the bold arrow runs from booking straight down to the shop side's bookable slots. So a booking spans N consecutive slots (N = required_slots) as N booking_slots rows; the booking's start time is DERIVED as MIN(starts_at) over those slots via the bookings_with_start view, not a stored start_slot_id (bookings has NO start_slot_id, NO barber_id, NO money-split columns). Slots have no status: availability is derived — a slot is offered unless a booking_slots row references it (a NOT EXISTS anti-join against booking_slots), and a UNIQUE(slot_id) on booking_slots is the no-double-book guard; cancelling a booking deletes its booking_slots rows so the slots free up. A booking is attributed to a barber/shop via bookings→service→services(barber_id)→barbers(shop_id); the greyed payout line = settlement state, derived from payout_id (NULL in M1.2), not a booking status. RLS scopes bookings to their own customer (and the owning shop reads via the service join). In M1.2 a booking ends at pending_payment (paid_at NULL, payout NULL) — a complete booking; the paid transition + Stripe Checkout arrive in M2.1.](assets/architecture-m1.2.png)

How the diagram maps to M1.2:
- **Customer → /barbers (browse) → /barbers/[id] (detail):** the customer-side **Product Site** (right) reads M1.1's `barbers` / `services` / `bookable slots` (left) — a slot is offered when **no `booking_slots` row** references it (the NOT EXISTS anti-join).
- **The greyed `bookable slot(s)` line inside the `booking` box IS the hidden join table.** The diagram draws `bookable slot(s)` as a field on `booking`, but there is **no such column** — physically it's a separate **`booking_slots(booking_id, slot_id)` join table** with an FK to the booking id and an FK to the bookable-slot id (one row per held slot). That's exactly the **bold arrow from `booking` down to `bookable slots`**: a booking references its slots *through* `booking_slots`, never directly. A booking spans **N = `required_slots`** consecutive slots ⇒ **N `booking_slots` rows** (all N, including the first).
- **Book dialog → `bookings` + `booking_slots` (Supabase):** the pop-up's confirm creates a `pending_payment` booking (`service` + `customer` + a `price` **snapshot** — **no `start_slot_id`**) **and N `booking_slots` rows** for the N consecutive slots. The chosen start is passed to the RPC as an argument but **not stored** on the booking — the start time is `MIN(starts_at)` over the `booking_slots` rows (the `bookings_with_start` view). **No slot status to flip** — the `UNIQUE(slot_id)` on `booking_slots` rejects any slot already held by another live booking, and the anti-join now excludes those slots from the available list.
- **`bookings` → `/bookings` (my-bookings):** the customer reads back **only their own** rows (RLS).
- **Attribution:** a booking is tied to a barber/shop via `bookings → service_id → services(barber_id) → barbers(shop_id)` — there is no `barber_id` on `bookings` (the service already pins the barber).
- **Greyed-out (M2.1):** the dialog's confirm will instead launch **Stripe Checkout**, and the webhook will flip the **booking** `pending_payment → paid` and **stamp `paid_at`** (the 20/80 split is derived later, at month-end, M2.2 — not on the booking, and there is no `transactions` table).

## The booking state machine (the canonical model — read this before building)

**A SLOT has no status.** A `bookable_slots` row is just a time window (`id, barber_id, starts_at, ends_at`); its length is the platform-wide `platform_settings.slot_minutes` (currently 30). Its availability is **DERIVED**: a slot is offered **unless** a `booking_slots` row references it. The whole lifecycle lives **only** on `bookings.status`, and the slots a booking holds live in `booking_slots`:

```
pending_payment ──(Stripe webhook, M2.1)──▶ paid
   │
   └──(customer/shop cancels → DELETE its booking_slots rows)──▶ cancelled
```

- **`pending_payment`** — a customer clicked Book and a `pending_payment` booking row + its **N `booking_slots` rows** (N = service.required_slots consecutive slots) were created. Those slots **disappear from the available list** immediately, because the availability anti-join excludes any slot that has a `booking_slots` row. Nobody has paid yet (`paid_at` is NULL). **In M1.2 `pending_payment` is the TERMINAL state** — a booking that ends here is a complete M1.2 booking. M2.1 is what turns "pending_payment" into "awaiting Stripe Checkout".
- **`paid`** — arrives **only in M2.1**, when the Stripe webhook sees a paid Checkout session, flips the **booking** to `paid`, and stamps `paid_at`. **Whoever pays first wins the slots.** The `booking_slots` rows stay (they're only deleted on cancel), so a paid booking keeps holding its slots. (Beyond M2.1, a paid booking is **settled** at month-end, M2.2 — but settlement is **derived from `bookings.payout_id`** being set, NOT a new booking status: the booking stays `paid`. There are no `payout_*` statuses — none of that exists in M1.2, where `payout_id` is always NULL.)
- **`cancelled`** — a `pending_payment` (or `paid`) booking that is cancelled. Cancelling **DELETEs the booking's `booking_slots` rows**, so those slots **automatically free** — the anti-join offers them again. No slot field to reset.

**Availability is an anti-join against `booking_slots`, not a status read.** A slot is bookable ⟺ NOT EXISTS a `booking_slots` row referencing it:

```sql
select s.* from bookable_slots s
where s.barber_id = :id
  and not exists (
    select 1 from booking_slots bs where bs.slot_id = s.id
  );
```

> **Concurrency-deferral note (explicit — do NOT engineer this now).** We deliberately do **not** build against the simultaneous-click race (two customers both grabbing the same slot in the same instant). The cheap, correct-enough guard is the **`UNIQUE(slot_id)` on `booking_slots`** — a plain DB constraint (not a lock/retry dance) that lets a slot belong to **at most one booking's hold**, so the first booking's insert wins and a second one is rejected. Because the whole booking write is one atomic RPC (1 booking + N `booking_slots` rows), if **any** of the N slots is already held the entire insert is rolled back — no partial booking. Beyond that constraint, **ignore this race until a single barber page averages ~1,000 concurrent customers**; engineering a row-lock / `SELECT … FOR UPDATE` / optimistic-retry dance before then is premature. If you find yourself adding retry loops in M1.2, stop — that's gold-plating.

> **Note for Claude Code:** in M1.2, a `pending_payment` booking is a real but **soft** hold. Don't write logic that auto-expires it (no cron, no TTL). There is **no slot status to flip and nothing to keep in sync** — the booking's `booking_slots` rows alone make its slots vanish from the anti-join, and the `UNIQUE(slot_id)` stops a second booking from grabbing a held slot. The `paid` path is intentionally **absent** here — leave the seam, build it in M2.1. Do **not** gate any M1.2 logic on `status='paid'` or `paid_at` — a `pending_payment` booking with `paid_at=NULL` is a complete M1.2 booking, and the hold is payment-independent.

## Conversational flow

You (Claude Code) drive the student through **6 steps**, in order. Don't dump them all at once — after each step, **wait for confirmation** before moving on.

1. Apply the `bookings` migration + indexes + RLS
2. Wire the booking write (INSERT a `pending` booking — no slot flip)
3. Build `/barbers` (browse all barbers + search/filter)
4. Build `/barbers/[id]` (detail — services + available slots + Book)
5. Build the Book **pop-up dialog** (date + slot + service → confirm → stay on the page)
6. Build `/bookings` (my-bookings) → push → run the checklist

---

### Step 1 — Apply the `bookings` migration (+ indexes + RLS)

Have Claude Code apply this as a **Supabase migration** via the MCP `apply_migration` (never a raw console edit — [[supabase-best-practice]]). It assumes M1.1 already created `barbers`, `services`, and `bookable_slots` (and that `bookable_slots` is a plain time window with **no status column**).

```sql
-- M1.2: customer bookings. One row per booking. A booking spans N CONSECUTIVE
-- slots (N = service.required_slots); the actual slots — INCLUDING the first — are in
-- the booking_slots join table below. There is NO start_slot_id on the booking: the
-- start time is DERIVED as MIN(starts_at) over the booking's slots (exposed via the
-- bookings_with_start view). bookings is PURELY buyer-side — no money split columns (the
-- platform/shop split is derived at month-end from paid bookings × commission_rates,
-- see M2.2). price is the SNAPSHOT of what the buyer is charged (in platform_settings.currency).
-- STATUS is a 3-state machine — 'pending_payment','paid','cancelled' — and in M1.2 a booking
-- only ever reaches 'pending_payment' (TERMINAL); 'paid' is M2.1. SETTLEMENT state is DERIVED
-- from payout_id (NULL = paid-but-not-yet-paid-out), NOT a status: there are no 'payout_*' statuses.
create table if not exists public.bookings (
  id            uuid primary key default gen_random_uuid(),
  customer_id   uuid not null references public.profiles(id) on delete cascade,
  service_id    uuid not null references public.services(id) on delete restrict,
  status        text not null default 'pending_payment'
                  check (status in ('pending_payment','paid','cancelled')),  -- 3 states only
  price         integer not null,        -- snapshot of services.price (in platform_settings.currency) — the agreed gross
  paid_at       timestamptz,             -- stamped by the M2.1 webhook when status → 'paid'. NULL in M1.2.
  payout_id     uuid,                    -- which payout BATCH settled this booking (set at month-end, M2.2). NULL in M1.2 (and = paid-but-not-yet-paid-out / OWED once paid). The FK → public.payouts(id) is ADDED in M2.2 once the payouts table exists (it doesn't yet).
  created_at    timestamptz not null default now(),
  updated_at    timestamptz not null default now()  -- auto-bumped by trg_bookings_updated_at (below)
);

-- Keep updated_at honest: bump it on every row change.
create or replace function public.set_updated_at()
returns trigger language plpgsql as $$
begin new.updated_at := now(); return new; end; $$;
drop trigger if exists trg_bookings_updated_at on public.bookings;
create trigger trg_bookings_updated_at
  before update on public.bookings
  for each row execute function public.set_updated_at();
create index if not exists idx_bookings_customer on public.bookings(customer_id);
create index if not exists idx_bookings_status   on public.bookings(status);
-- NO idx_bookings_start index: there is no start_slot_id column; the start time is
-- derived from booking_slots (MIN(starts_at)) via the bookings_with_start view.

-- ── booking_slots: the N slots a booking consumes (one row per slot). ──
create table if not exists public.booking_slots (
  booking_id uuid not null references public.bookings(id) on delete cascade,
  slot_id    uuid not null references public.bookable_slots(id) on delete restrict,
  primary key (booking_id, slot_id)
);
-- NO-DOUBLE-BOOK GUARD: a slot may belong to at most ONE booking's hold. Because the
-- cancel path DELETEs a booking's booking_slots rows, this UNIQUE means "one LIVE
-- booking per slot" with no status column needed on the join table.
create unique index if not exists uniq_slot_held on public.booking_slots(slot_id);
create index if not exists idx_booking_slots_booking on public.booking_slots(booking_id);

-- ── bookings_with_start: the booking + its DERIVED start/end time. ──
-- There is no start_slot_id column on bookings; /bookings and any "sort/show by start
-- time" read uses this view (MIN(starts_at)/MAX(ends_at) over the booking's slots).
-- RLS is inherited from the underlying bookings table.
create or replace view public.bookings_with_start as
select b.*,
       (select min(s.starts_at) from public.booking_slots bs
          join public.bookable_slots s on s.id = bs.slot_id
        where bs.booking_id = b.id) as starts_at,
       (select max(s.ends_at)  from public.booking_slots bs
          join public.bookable_slots s on s.id = bs.slot_id
        where bs.booking_id = b.id) as ends_at
from public.bookings b;

alter table public.bookings      enable row level security;
alter table public.booking_slots enable row level security;

-- A customer reads/writes ONLY their own bookings.
create policy "bookings_select_own" on public.bookings
  for select using (auth.uid() = customer_id);
create policy "bookings_insert_own" on public.bookings
  for insert with check (auth.uid() = customer_id);
create policy "bookings_update_own" on public.bookings
  for update using (auth.uid() = customer_id);
-- The SHOP who owns the barber reads bookings for its services — read-only.
-- There is NO bookings.barber_id; the booking is ATTRIBUTED to a shop by joining
-- service_id → services(barber_id) → barbers(shop_id). (The service already pins
-- the barber.) A shop can run MANY barbers, so payout rollups (M2.2) sum across all
-- barbers where barbers.shop_id = the shop.
create policy "bookings_select_shop_owner" on public.bookings
  for select using (
    exists (
      select 1
      from public.services sv
      join public.barbers b on b.id = sv.barber_id
      where sv.id = bookings.service_id and b.shop_id = auth.uid()
    )
  );

-- booking_slots: PUBLIC select (the availability anti-join is read by anonymous
-- browsers; only (booking_id, slot_id) is exposed — no PII). A customer may
-- write join rows only for their own booking.
create policy "booking_slots_select_public" on public.booking_slots
  for select using (true);
create policy "booking_slots_write_own" on public.booking_slots
  for all using (
    exists (select 1 from public.bookings b where b.id = booking_slots.booking_id and b.customer_id = auth.uid())
  ) with check (
    exists (select 1 from public.bookings b where b.id = booking_slots.booking_id and b.customer_id = auth.uid())
  );
```

> **Note for Claude Code:** `price` is a **whole integer in `platform_settings.currency`** (default TWD, which is Stripe **zero-decimal**, so there is no ×100 here). Snapshotting it onto the booking row (rather than re-reading `services.price` later) is the rule that survives a price change: the booking — and M2.1's Stripe charge — bills what the customer saw. **There are no money-split columns on `bookings`** — when the booking is paid, M2.1's webhook stamps `paid_at`, and the platform/shop split is **derived** at month-end (M2.2) from paid bookings × the month's `commission_rates` (there is **no `transactions` table**). (See [[supabase-best-practice]] and [[m2.1-buyer-to-admin-payments]].)

> **Shop attribution (a shop can run MANY barbers).** A `bookings` row carries **no `barber_id`** — and that's intentional. The model is **one SHOP → many barbers**, so to attribute a booking to a barber/shop you **join `bookings.service_id → services.barber_id → barbers.shop_id`** (the service already pins the barber). The **shop-level rollups happen in M2.2**. Don't add a `barber_id` or `shop_id` column to `bookings`.

**A booking spans N consecutive slots.** Slots are fixed-length units (`platform_settings.slot_minutes`, currently 30, platform-wide). A service needs **N = service.required_slots** adjacent slots on the same barber — taken from `required_slots` directly (NOT `ceil(duration/30)`; services are decoupled from the slot-minute constant). The actual slots are rows in **`booking_slots`** — **all N, including the first**. There is **no `start_slot_id` column on `bookings`**: the booking's start time is **derived** as `MIN(starts_at)` over its `booking_slots` rows (read via the `bookings_with_start` view) for display/ordering. The **`UNIQUE(slot_id)` on `booking_slots`** is the no-double-book guard — a slot can be in at most one booking's hold. Cancelling a booking **deletes its `booking_slots` rows**, which frees those slots (so the UNIQUE means "one *live* booking per slot" with no status column on the join table).

**Verify (Supabase MCP `execute_sql`):**
```sql
-- bookings has 3-state status, price (NOT price_twd), paid_at, payout_id, and NO start_slot_id / slot_id / money-split / barber_id columns:
select column_name from information_schema.columns where table_schema='public' and table_name='bookings';
-- the booking_slots join table + its no-double-book UNIQUE exist:
select column_name from information_schema.columns where table_schema='public' and table_name='booking_slots';
select indexname from pg_indexes where tablename='booking_slots' and indexname='uniq_slot_held';
-- the bookings_with_start view exposes the derived starts_at / ends_at:
select column_name from information_schema.columns where table_schema='public' and table_name='bookings_with_start';
-- RLS + policies on both:
select tablename, policyname from pg_policies where tablename in ('bookings','booking_slots');
```
Also run `get_advisors` and confirm no "RLS disabled" / "policy missing" warnings on `bookings` or `booking_slots`.

**Step 1c — the transactional booking RPC + the free-on-cancel trigger.** The booking write inserts **1 booking + N join rows** and must be atomic + check availability — so do it in a **Postgres function**, not a client multi-insert. Apply as part of the same migration:

```sql
-- Create a booking spanning N consecutive slots, atomically. N = service.required_slots directly.
create or replace function public.create_booking(p_service_id uuid, p_start_slot_id uuid)
returns uuid language plpgsql security definer set search_path = public as $$
declare
  v_barber_id uuid; v_n int; v_price int; v_start timestamptz;
  v_slot_ids uuid[]; v_booking_id uuid;
begin
  -- service price snapshot + required_slots, and the barber the start slot belongs to
  select s.price, s.required_slots, sl.barber_id, sl.starts_at
    into v_price, v_n, v_barber_id, v_start
    from public.services s
    join public.bookable_slots sl on sl.id = p_start_slot_id
    where s.id = p_service_id;
  if v_barber_id is null then raise exception 'service or start slot not found'; end if;
  -- the service must belong to the SAME barber as the slot
  if not exists (select 1 from public.services s where s.id = p_service_id and s.barber_id = v_barber_id) then
    raise exception 'service and slot belong to different barbers';
  end if;

  -- N := service.required_slots (NOT ceil(duration/30) — services use required_slots directly).

  -- Grab the next N slots for this barber starting AT the chosen start slot, in time order.
  declare v_slots record; v_prev_end timestamptz; v_count int := 0;
  begin
    v_slot_ids := array[]::uuid[];
    for v_slots in
      select id, starts_at, ends_at from public.bookable_slots
      where barber_id = v_barber_id and starts_at >= v_start
      order by starts_at
      limit v_n
    loop
      -- CONTIGUITY: every slot after the first must start exactly where the previous ended
      -- (no gap in the published schedule). This is what guarantees a multi-slot service
      -- is fulfilled by N back-to-back windows, not slots scattered across the day.
      if v_count > 0 and v_slots.starts_at <> v_prev_end then
        raise exception 'this service needs % back-to-back slots from that start time, but the barber has a gap', v_n;
      end if;
      v_slot_ids := v_slot_ids || v_slots.id;
      v_prev_end := v_slots.ends_at;
      v_count := v_count + 1;
    end loop;
  end;
  if v_count <> v_n then
    raise exception 'not enough consecutive slots from that start time for this service (needs %)', v_n;
  end if;

  -- insert the booking (status defaults to 'pending_payment'), then the N join rows.
  -- p_start_slot_id is the customer's chosen start — used to derive the N slots above,
  -- but it is NOT stored on bookings (no start_slot_id column). ALL N slots, INCLUDING
  -- the start, go into booking_slots; the start is recoverable as MIN(starts_at) over them.
  -- The UNIQUE(slot_id) on booking_slots aborts the whole transaction if ANY of the
  -- N slots is already held → atomic, no partial booking.
  insert into public.bookings(customer_id, service_id, price)
    values (auth.uid(), p_service_id, v_price)
    returning id into v_booking_id;
  insert into public.booking_slots(booking_id, slot_id)
    select v_booking_id, unnest(v_slot_ids);

  return v_booking_id;
exception when unique_violation then
  raise exception 'one or more of those time slots were just taken — pick another start time';
end; $$;

-- Free the slots when a booking is cancelled: delete its booking_slots rows.
create or replace function public.free_slots_on_cancel()
returns trigger language plpgsql security definer set search_path = public as $$
begin
  if new.status = 'cancelled' and old.status <> 'cancelled' then
    delete from public.booking_slots where booking_id = new.id;
  end if;
  return new;
end; $$;
drop trigger if exists trg_free_slots_on_cancel on public.bookings;
create trigger trg_free_slots_on_cancel
  after update of status on public.bookings
  for each row execute function public.free_slots_on_cancel();
```

> **Note for Claude Code:** the **whole booking write is the RPC `create_booking`** — one transaction, so a half-held booking can never exist. The booking is created at `status='pending_payment'` (the default) — **that is M1.2's terminal state; don't set it to `paid`**. **N = `service.required_slots`** (read it straight off the service — NOT `ceil(duration/30)`; services carry no minutes). The `UNIQUE(slot_id)` on `booking_slots` is the entire no-double-book mechanism: if any of the N slots is already held, the insert raises `unique_violation` and the booking is rolled back; surface that as "those times were just taken". Don't engineer row-locks/retries (deferred until ~1,000 concurrent customers/barber). The **free-on-cancel trigger** is what makes a cancelled booking release its slots, so the plain `UNIQUE` correctly means "one *live* booking per slot".

---

### Step 2 — Wire the booking write (call the `create_booking` RPC)

The booking write is the heart of M1.2 — and because a booking now spans **N consecutive slots**, it must insert **1 booking + N `booking_slots` rows atomically**. So the write is **the `create_booking(service_id, start_slot_id)` RPC** from Step 1c, not a client-side INSERT.

> 幫我加上「建立預約」的後端寫入。當客人在某位理髮師的 `/barbers/[id]` 確認預約時：
>
> 1. 前端拿到客人選的 **`service_id`** 和 **起始時段 `start_slot_id`**，呼叫 Supabase RPC **`create_booking(service_id, start_slot_id)`**（不要在前端自己拼多筆 insert）。
> 2. 該 RPC 會：讀出服務的 `price`（**快照**，整數，幣別由 `platform_settings.currency` 決定、台幣不 ×100）與 `required_slots`，算出 **N = required_slots**（直接用，不是 `ceil(duration/30)`），找出從 `p_start_slot_id` 開始、同一位理髮師、連續的 N 個時段，然後在一個交易裡 INSERT 一筆 `bookings`（`status='pending_payment'`、`price` 快照、`paid_at` 為 NULL、**沒有 `start_slot_id`、沒有 `barber_id`、沒有金額拆帳欄位**）＋ N 筆 `booking_slots`（**含起始時段在內的全部 N 個**）。`p_start_slot_id` 只是用來推算這 N 個時段的「開始時間」參數，**不會存進 `bookings`**；之後要顯示/排序開始時間時，用 `booking_slots` 的 `MIN(starts_at)`（即 `bookings_with_start` view）。
> 3. **不需要改任何時段狀態** —— 時段沒有 status 欄位。一旦 `booking_slots` 寫入，那 N 個時段就會因為 anti-join（排除「已經有 `booking_slots` 指到」的時段）從可預約清單消失。`booking_slots` 上的 `UNIQUE(slot_id)` 會擋掉「同一個時段被第二筆預約佔用」，整個交易會被 rollback —— 把這個錯誤呈現成「這些時段剛剛被搶走了，請換一個開始時間」。
> 4. 成功後回傳 `booking_id`，讓前端關掉 dialog、跳一個 toast、留在 `/barbers/[id]`。
>
> 還不要接 Stripe — 這筆預約停在 `pending_payment` 就是 M1.2 完成的狀態。M2.1 才會在「確認」這一步改成：先 `create_booking` 拿到 `booking_id`，再開 Stripe Checkout（付款成功後 webhook 才把 booking 變 `paid` 並蓋 `paid_at`）。

**CLI equivalent (the shape of the server write):**
```bash
# Call the Postgres RPC (one transaction = 1 booking + N booking_slots), NOT a client multi-insert:
#   supabase.rpc('create_booking', { p_service_id, p_start_slot_id })
# If any of the N slots is already held, the UNIQUE(slot_id) on booking_slots raises a
# unique-violation and the WHOLE booking rolls back — surface "those times were just taken".
```

> **Note for Claude Code:** the booking write is the **`create_booking` RPC** (Step 1c) — it reads **N = service.required_slots** (NOT ceil(duration/30)), grabs the N consecutive slots, and inserts the booking + N join rows in one transaction. The price snapshot and the N-slot logic run on the trusted DB side. The booking lands at `status='pending_payment'` (terminal for M1.2) — don't touch `paid`/`paid_at`. The double-book guard is the **`UNIQUE(slot_id)` on `booking_slots`**; if any slot is taken the whole RPC rolls back — return a clean "those times were just taken" error, don't retry. **No slot-status UPDATE** (slots have no status). Do **not** add Stripe here — M2.1 reuses this exact RPC, then opens Checkout with the returned `booking_id`. Leave that seam clean.

---

### Step 3 — Build `/barbers` (browse all barbers + search/filter)

> **版型參考 / Layout reference:** 做成**電商 marketplace 列表頁**的版型 — 上方一條 navbar（左側選單 icon、中間 logo、右側 profile / 訊息 icon），下面是一個**商品卡片格狀牆（responsive product-card grid）**。我們把「商品」換成「理髮師」。
>
> 做一個 `/barbers` 頁面，讓**客人**瀏覽**所有**理髮師：
>
> - 用一個**卡片格狀排版（card grid，參考上面 marketplace 的卡片牆）** 列出所有 `barbers`，每張卡顯示：名稱、簡介、地址、以及**一張代表作品照**（取該店 `barber_photos` 中 `is_featured=true` 的第一張，沒有就退而取最新一張，再沒有就用 placeholder）。
> - 上方放一個**搜尋框**（依名稱/地址關鍵字過濾）+ 一排**分類 filter chips**：All / Cut / Color / Perm / Beard（依該理髮師提供的 `services.category` 過濾）。
> - 每張卡可點進該理髮師的 `/barbers/[id]` 詳細頁。
> - 這是**公開可瀏覽**的頁面（登入與否都能看理髮師清單；要按下 Book 才需要登入）。

> **Note for Claude Code:** match a **standard marketplace listing layout** (navbar + responsive product-card grid) but swap "products" for "barbers". `/barbers` reads M1.1's `barbers` (+ joins `services` for the category filter, + the rep photo from `barber_photos`). It is **distinct** from the barber-management page `/shop/bookings` and from `/shop/earnings` — those are gated to `role = 'shop'`. The `[id]` route below is a **uuid/int barber id**, so `/barbers/<uuid>` never collides with `/shop/bookings`. ([[m1.1-seller-setup]] owns the data this page reads.)

---

### Step 4 — Build `/barbers/[id]` (detail — services + available slots + Book)

> **版型參考 / Layout reference:** 做成**電商 marketplace 商品詳細頁**的版型 —— **特別要做出「照片輪播 carousel」**。版型由上而下：navbar → 頂部一個**大圖照片輪播（主圖 + 可左右切換 + 縮圖/圓點導覽 + 點圖可放大）** → 標題與價格 → 賣家資訊 → 規格 → 描述 → 動作按鈕。我們把「商品照」換成理髮師的**作品照**、把「賣家」換成這位**理髮師**、把「購買」換成 **Book**。
>
> 做 `/barbers/[id]` 這個**理髮師詳細頁**（`[id]` 是該理髮師的 uuid/int id）：
>
> - **頂部：作品照輪播 / Hairstyle photo carousel（最重要）** — 讀 `barber_photos`（屬於這間 `barber_id`），做成一個**電商商品詳細頁式的照片輪播 carousel**：一張大主圖、左右切換箭頭、下方縮圖或圓點導覽、點主圖可放大（lightbox）。排序依 `sort_order`，**精選（`is_featured`）排前面**。圖片從公開的 `barber-photos` Storage bucket 取 URL。這是顧客判斷「這個理髮師手藝如何」的主要依據。
> - **理髮師資訊**：名稱、簡介、地址。
> - 一段 **Services** 區塊：列出這位理髮師的 `services`（名稱、分類、`price`、需要幾個時段 `required_slots`），每個服務可被選取。
> - 一段 **Available slots** 區塊：用 **anti-join** 只顯示這位理髮師「還沒被佔用」的 `bookable_slots` —— 也就是 `not exists (select 1 from booking_slots bs where bs.slot_id = s.id)`。時段本身沒有 status 欄位；可預約與否是「有沒有 `booking_slots` 指到它」推導出來的（取消預約會刪掉 `booking_slots`，時段就自動釋出）。**重要**：因為一筆預約會佔用 N 個連續時段（**N = 該服務的 `required_slots`**，直接用，不是 ceil(duration/30)），選「開始時段」時要確認它後面有 **連續 N 個**都還沒被佔用的時段，才把它列為可選的開始時間。
> - 一顆明顯的 **Book** 按鈕。**按下 Book 不要換頁** — 它會打開一個 pop-up dialog（下一步做）。
> - 若使用者未登入，按 Book 時導去 `/login`，登入後回到本頁。

> **Note for Claude Code:** match a **standard marketplace product-detail layout**, and in particular **build the photo carousel** at the top (main image + prev/next arrows + thumbnail/dot nav + click-to-zoom lightbox) — the user explicitly called this out. Use the project's existing carousel primitive (shadcn `Carousel`/embla, Radix, etc.). The **carousel reads `barber_photos`** (created in M1.1), ordered by `sort_order` with `is_featured` first; files come from the public `barber-photos` bucket, so a plain public URL works (no auth to view). The available-slots list filters **via the anti-join against `booking_slots`** (a slot is offered when **no `booking_slots` row** references it) — there is **no `slot.status`** to read, and **no `status` filter** on the booking (the row-existence in `booking_slots` IS the hold, independent of `pending_payment`/`paid`). Because Step 2 inserts a `pending_payment` booking (+ its `booking_slots` rows) for the chosen slot, a freshly-booked slot **disappears** on the next read (intended; no double-offering). Keep the page on `/barbers/[id]`; the Book action is a dialog, not a navigation. The `is_featured` photos are the same set the **M4 egg unit** feeds to AI to draft the bio.

---

### Step 5 — Build the Book pop-up dialog (the core UX)

This is the signature interaction of M1.2. **The customer never leaves `/barbers/[id]`.**

> 在 `/barbers/[id]` 上做一個**預約 pop-up dialog（modal）**，這是 M1.2 的核心 UX：
>
> 1. 客人按 **Book** → 彈出一個 modal（**頁面不跳轉，背景仍是 `/barbers/[id]`**）。
> 2. modal 裡讓客人選：**服務（service，決定價格與長度）** → **日期** → 該日期下的**可選「開始時段」**。先選服務，因為（a）價格來自 `service.price`、（b）服務要佔幾個時段：**N = service.required_slots**（時段長度是 `platform_settings.slot_minutes`，目前 30 分鐘）。選好服務後把**價格清楚顯示在 modal 上**（整數，幣別由 `platform_settings.currency` 決定）。
> 3. **可選的「開始時段」要過濾**：只有「它自己＋後面連續 N−1 個時段都還沒被佔用」的時段，才能當開始時間（例如 `required_slots=3` 的燙髮，就只列出後面有連續 3 個空檔的開始時間）。判斷「被佔用」用 anti-join：該時段沒有任何 `booking_slots` 指到它。
> 4. modal 底部有 **Confirm** 與 **Cancel**。
> 5. 按 **Confirm** → 呼叫 Step 2 的 **`create_booking(service_id, start_slot_id)` RPC**（它在一個交易裡建立 `pending_payment` booking ＋ N 筆 `booking_slots`、快照 `price`，**不要改任何時段狀態**）→ **關閉 modal** → **留在 `/barbers/[id]`** → 跳一個成功 toast（中性字樣，因為這個里程碑還不收錢）。若 RPC 因為時段剛被搶走而失敗（`UNIQUE(slot_id)` 衝突），顯示「這些時段剛剛被搶走了，請換一個開始時間」。
> 6. 那 N 個時段在重新整理後會從可預約清單消失 —— 因為它們現在都有 `booking_slots` 指到，anti-join 會排除它們。
>
> ⚠️ 這一步**還不要接 Stripe**。這筆預約停在 `pending_payment`（`paid_at` 為 NULL）就是 M1.2 完成的狀態。請把「Confirm」的行為寫得乾淨、可被替換 —— 因為 **M2.1 會把這顆 Confirm 從「只呼叫 `create_booking`」改成「`create_booking` 拿到 `booking_id` 後開 Stripe Checkout」**，付款成功後 webhook 才把 booking 變 `paid` 並蓋上 `paid_at`。

> **Note for Claude Code:** the dialog is a controlled modal over `/barbers/[id]` (shadcn `Dialog`, Radix, etc.). Picking the service fixes **N = service.required_slots** (read it directly — not ceil(duration/30)); only offer **start slots that have N contiguous free slots** after them (a free slot = no `booking_slots` row references it). On confirm: call the **`create_booking` RPC** (it creates the `pending_payment` booking + N `booking_slots` rows in one transaction — **no slot-status flip**, slots have no status), then `close()` the dialog and fire a toast; **do not** `router.push`. The held slots disappear from the available list on the next read because the anti-join excludes any slot with a `booking_slots` row. This "stay on the page" contract is what makes M2.1's swap trivial — M2.1 keeps the same `create_booking` call and just adds "→ create Checkout Session → redirect" after it. Keep the toast payment-neutral in M1.2 since no money moves yet (the booking is complete at `pending_payment`).

---

### Step 6 — Build `/bookings` (my-bookings) → push → run the checklist

> 做 `/bookings` 這個**客人的「我的預約」頁面**：
>
> - 列出**目前登入客人自己的**所有 `bookings`（靠 RLS：`auth.uid() = customer_id`），顯示理髮師名稱（透過 `service_id → services.barber_id → barbers` join 取得，`bookings` 本身沒有 `barber_id`）、服務、**起始時間（讀 `bookings_with_start` view 的 `starts_at` —— 即該預約 `booking_slots` 的 `MIN(starts_at)`，`bookings` 沒有 `start_slot_id` 欄位）**、`price`、以及 `status`（M1.2 都會是 `pending_payment`）。
> - 依**起始時間（`bookings_with_start.starts_at`）**排序，最新的在上。
> - 客人看不到別人的預約（這由 RLS 強制，不是只靠前端過濾）。

Then have Claude Code **push to `main`** (recall the GitHub PAT from Secrets Manager `barber-project/github` — don't re-paste), let Vercel redeploy, and verify:

> **ask:** "Run the `m1.2-buyer-setup-checklist` skill."

---

## Things to watch out for (common mistakes)

0. **Skipping the prereq** — run `[[m1.2-buyer-setup-prerequisite]]` first. If M1.1's data isn't really there (no barber with a service + enough future free slots), the Book dialog has nothing to offer and you can't smoke-test the flow — you'll build blind against an empty schedule. (The `role='customer'` account you book as is **not** a prereq — you sign it up as the first act of this build; M1.2 is where the customer side is first built.)
1. **Reading `services.price` later instead of snapshotting it** — always snapshot `price` onto the booking row at creation (Step 1/2). A later price edit must not retroactively change an existing booking or its M2.1 charge.
2. **`price` × 100** — the default currency (TWD) is **zero-decimal**. Store the whole-unit amount in `platform_settings.currency`; the ×10^minor_units math lives in M2.1's Stripe `unit_amount` (driven by `platform_settings.currency_minor_units`) — see [[m2.1-buyer-to-admin-payments]] / [[stripe-best-practice]].
3. **Adding a `barber_id` (or `start_slot_id`) column to `bookings`** — there is **none of either**. A booking references **only** `service_id`; its slots (including the first) live in `booking_slots`, and its start time is **derived** as `MIN(starts_at)` over them (via the `bookings_with_start` view). The barber/shop is reached via `bookings → services(barber_id) → barbers(shop_id)`. Don't store `barber_id`, `shop_id`, or `start_slot_id` on the booking, and don't put `barber_id` in M2.1's Stripe metadata as a source of truth.
4. **Trying to "flip the slot" / read `slot.status`** — slots have **no status**. Availability is **derived** via the anti-join (a slot is offered when no `booking_slots` row references it — independent of booking status). Don't write a `bookable_slots` status UPDATE and don't filter on `status='available'`.
5. **Offering already-booked slots** — the `/barbers` + `/barbers/[id]` available-slots query must use the **NOT EXISTS anti-join against `booking_slots`** (`not exists (select 1 from booking_slots bs where bs.slot_id = s.id)`), and a START slot is only offered if the next **N = service.required_slots** slots are all free. A held slot must not appear, or you invite double-books.
6. **Skipping the double-book guard** — the **`UNIQUE(slot_id)` on `booking_slots`** is the cheap DB-level guarantee that a slot is in at most one (live) booking's hold. Keep it; surface its unique-violation as "those times were just taken". (Cancelling a booking deletes its `booking_slots` rows — the free-on-cancel trigger — so the UNIQUE means *live*-only.)
7. **Navigating away from `/barbers/[id]` on Book** — the spec is a **pop-up dialog**, not a new page. The customer stays put; the dialog closes back to the same page with a toast. (This is the seam M2.1 reuses.)
7a. **Dropping the photo carousel / gallery** — `/barbers/[id]` must lead with a **photo carousel** of the barber's `barber_photos` (main image + prev/next + thumbnail/dot nav + zoom), like a marketplace product-detail page; `/barbers` cards each show a rep photo. The portfolio is the customer's main signal — don't ship a text-only detail page.
7b. **Not picking the service first in the dialog** — price comes from `service.price`, so the dialog selects **service → date → slot** and shows the price before Confirm. A slot carries no price on its own.
8. **Building the `paid` transition / payment now / gating on `paid`** — that's M2.1. In M1.2, `pending_payment` is terminal and a complete booking; `paid_at` stays NULL. Don't add Stripe, don't stamp `paid_at`, don't gate any logic on `status='paid'` or `paid_at`, don't add `/bookings/success`.
9. **Over-engineering the concurrency race** — do NOT add row-locks / retry loops / TTL expiry for simultaneous clicks. Deferred until ~1,000 concurrent customers/barber. The **`UNIQUE(slot_id)` on `booking_slots`** + the atomic `create_booking` RPC are all the safety M1.2 needs.
10. **Weak RLS on `bookings`** — a customer must read/write only their own rows (`auth.uid() = customer_id`); the owning **shop** gets read-only on bookings for its barbers' services via the join `services → barbers` (`b.shop_id = auth.uid()`). Verify with `get_advisors` ([[supabase-best-practice]]).
11. **Raw ad-hoc SQL against prod** — the `bookings` table and any later change go through a **migration / `apply_migration`**, never a console `INSERT`/`UPDATE`.
12. **Client-side-only `insert`** — prefer a server route / RPC so the price snapshot runs on the trusted side (Step 2).

## Expected duration

40–70 minutes — most of it on the `/barbers/[id]` detail page + the Book dialog UX. The migration is quick; the dialog "stay-on-page + slot-flip" wiring is the part to get right.

## Next step

When `m1.2-buyer-setup-checklist` is green, tell the student:
「M1.2 完成了！客人現在可以瀏覽所有理髮師、進到某位理髮師的詳細頁、按 Book 開啟 pop-up dialog 選日期＋時段，確認後就建立了一筆 `pending_payment` 預約（時段沒有 status 欄位，但因為 `booking_slots` 已經寫入，anti-join 會讓它從可預約清單消失；`booking_slots` 上的 `UNIQUE(slot_id)` 也擋掉同一時段的第二筆 live 預約），並能在 `/bookings` 看到自己的預約。**這筆 `pending_payment`、`paid_at` 為 NULL 的預約就是 M1.2 的完整成果——還沒收錢**，佔位是靠 `booking_slots` 的存在、與付款無關。準備好的話跟我說『啟動 M2.1』，我們來接 Stripe：把那顆 Confirm 改成開 Stripe Checkout，付款成功後 webhook 才把**這筆預約**從 `pending_payment` 變 `paid`、蓋上 `paid_at`，而且**誰先付款誰就贏得這個時段**；平台 20%／理髮師 80% 的拆帳是月底結算（M2.2）才從 `paid` 預約推導出來的。」
Then load `m2.1-buyer-to-admin-payments` (run `m2.1-buyer-to-admin-payments-prerequisites` first — it sets up the Stripe sandbox auth and promotes your admin account).

## Reference

- Supabase RLS: https://supabase.com/docs/guides/auth/row-level-security
- Supabase migrations: https://supabase.com/docs/guides/deployment/database-migrations
- Postgres `gen_random_uuid()`: https://www.postgresql.org/docs/current/functions-uuid.html
- shadcn/ui Dialog (modal): https://ui.shadcn.com/docs/components/dialog
- Cross-skills: [[m1.2-buyer-setup-prerequisite]] · [[m1.2-buyer-setup-checklist]] · [[m1.1-seller-setup]] · [[m2.1-buyer-to-admin-payments]] · [[supabase-best-practice]] · [[lovable-best-practice]]
