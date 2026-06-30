---
name: m1.1-seller-setup
description: 抽成制理髮師預約平台 Milestone 1.1 — let a shop list ONE OR MORE barbers (the `name`/intro/address barber profile), list services, upload sample hairstyle photos, publish bookable schedule slots, and save shop-level payout (bank) details. Adds a "Become a shop" flow + wires the sign-up role tab to `profiles.role = 'shop'`, creates the `platform_settings` (single-row currency + slot-length config) / `barbers` (shop_id NOT unique — many barbers per shop) / `services` / `bookable_slots` / `barber_photos` tables via Supabase `apply_migration` with RLS (a shop may only CRUD their own rows; bank fields live on `profiles` and are readable only by the shop + admin) plus a public `barber-photos` Storage bucket (shop-write, public-read), and builds the shop onboarding (payout settings + create/manage barbers + photo uploader) + the `/shop/bookings` schedule & service editor. Use when the student says "啟動 M1.1", "start M1.1", "讓理髮師上架", "barber profile and schedule", "建立預約排程", "上傳作品照", or any variant of "上架理髮師、發布可預約時段". (對應 course unit 4-7.)
---

# M1.1 — 理髮店上架與預約排程（上架理髮師、建立預約排程）

## What this skill does

Walks the student through Milestone 1.1 — the **shop side**: a customer becomes a shop, lists **one or more barbers**, lists services with prices, uploads **sample hairstyle photos** (the barber's portfolio of past work), publishes bookable time slots, and saves their **shop-level payout (bank) details**. (A shop is the aggregator account that runs one-or-more barbers — a one-man shop works too.) This is the **first milestone where Supabase holds real application data** (M0 was auth-only): you create the single-row `platform_settings` config plus four tenant tables + a Storage bucket with RLS via `apply_migration`, then build two shop surfaces on top of them.

By the end the student has:

1. A **"Become a shop" path** — an existing customer can upgrade to a shop, and the M0 sign-up role tab now persists `profiles.role = 'shop'` so the shop surfaces unlock.
2. A **`platform_settings` table** — a single-row global config (`currency` default `twd`, `currency_minor_units` default `0`, `slot_minutes` default `30`). It replaces the hard-coded "TWD zero-decimal" + "30-min" constants: money columns are integers in `platform_settings.currency`, and a bookable slot is a `slot_minutes`-long window. World-readable (the UI needs it), admin-only write.
3. A **`barbers` table** (`shop_id → profiles`, **`name`**, intro, address) — **one shop can list MANY barbers** (`shop_id` is **not** unique; there's an index `idx_barbers_shop`). **No bank columns on barbers** — the payout target lives on `profiles` (shop level), since one shop has one bank account.
4. **Shop-level payout details on `profiles`** — `bank_account_name` / `bank_account_number` (added to `profiles` in M0, filled here). Readable only by the shop + an admin; this is what M2.2 pays out, rolled up per shop.
5. A **`services` table** (`barber_id`, name, **category** `cut / color / perm / beard`, **price**, **required_slots**) — the menu a customer books from. `price` is an integer in `platform_settings.currency`; `required_slots` is how many consecutive `bookable_slots` the service needs.
6. A **`bookable_slots` table** (`barber_id`, `starts_at`, `ends_at`) — published time windows, **with no status column**. Each slot is a **`platform_settings.slot_minutes`-long unit** (configurable, default 30 min): the barber publishes those windows, and in M1.2 a booking spans **N = service.required_slots** consecutive slots. A slot is bookable until a booking references it (the booking lifecycle lives on `bookings` + the `booking_slots` join table, M1.2). 

7. A **`barber_photos` table + a `barber-photos` Supabase Storage bucket** — the barber's **sample hairstyle photos** (their past work). Each row is `barber_id`, `storage_path`, `caption`, `is_featured`, `sort_order`. The shop uploads/deletes; **anyone may view** (it's the public portfolio shown on `/barbers/[id]` in M1.2). The `is_featured` flag is what the **M4 egg unit** will feed to AI to auto-write the barber's `intro`.
8. **RLS on every table** — a shop can only CRUD **their own** barbers / services / slots / photos; the **bank fields on `profiles` are readable only by the shop + an admin**, never world-readable; photo files are public-read but shop-write.
9. A **shop onboarding flow** — create/manage **one or more barbers** (barber profile) + a **shop-level "payout settings"** form with **REQUIRED bank info** (account name + number; *"use test data first"*) + a **sample hairstyle photo uploader**.
10. A **`/shop/bookings` page** — publish/edit bookable slots **and** a service & price editor (per barber).

**Out of scope for M1.1:** the customer browse/booking flow (`/barbers`, `/barbers/[id]`, the booking dialog, `bookings` table) is **M1.2** (which also renders the photo gallery on the detail page); Stripe payments are **M2.1**; the shop's **`/shop/earnings`** page ships in **M2.2** (it needs paid bookings to show numbers); the admin payout page is **M2.2**. **AI auto-writing the barber's `intro` from the featured photos is the M4 egg unit** (`m4-ai-autofill-barber`, course 4-19) — M1.1 only stores the photos + the `is_featured` flag that M4 will consume.

## When to load this skill

Trigger phrases:
- "啟動 M1.1" / "start M1.1" / "begin M1.1"
- "上架理髮師" / "理髮師檔案" / "barber profile and schedule"
- "建立預約排程" / "發布可預約時段" / "publish bookable slots"
- Any prompt mapping to "理髮店上架理髮師、列服務、排可預約時段"

Run **`m1.1-seller-setup-prerequisite` first** — it confirms M0 is green (auth + `profiles.role`), the Supabase MCP is reachable, and the GitHub token is cached, then does one verified read against the new/empty schema. Do NOT load this skill for M1.2 (the buyer side) — that has its own skill `[[m1.2-buyer-setup]]`.

## Execution mode (Cowork-first)

This milestone runs in **Cowork on Desktop**, the workbench M0 set up. Two kinds of work:

| Work | Cowork mode | CLI mode |
|---|---|---|
| UI changes (onboarding form, `/shop/bookings`) | Prompt **Lovable**, then push to GitHub via the cached token | Same Lovable prompt; `git push` with the PAT from Secrets Manager |
| Schema + RLS (`platform_settings`/`barbers`/`services`/`bookable_slots`) | **Supabase MCP** `mcp__claude_ai_Supabase__apply_migration` | `supabase db push` / SQL editor (but the course always uses `apply_migration`) |

| RLS sanity check | **Supabase MCP** `mcp__claude_ai_Supabase__get_advisors` | same MCP call |

Every Supabase change goes through a **migration** (`apply_migration`), never a raw ad-hoc `UPDATE` in the SQL editor — see [[supabase-best-practice]]. RLS is **on by default** for these multi-tenant tables.

## Architecture

![Barber platform architecture (M1.1) — the shop side. The student drives Cowork (claude code), which pushes UI to the GitHub repo (→ Vercel-hosted Product Site) and applies migrations to the student's own Supabase. New in M1.1: the single-row platform_settings config plus RLS-protected tables — barbers (shop_id → profiles, with shop-only bank fields on profiles), services (barber_id, category, price, required_slots), and bookable_slots (barber_id, starts_at/ends_at — no status column; a slot is just a slot_minutes-long time window). A shop signs in (profiles.role = 'shop'), fills the onboarding form (barber profile + required bank info), and uses /shop/bookings to publish slots and edit services. RLS scopes every row to its shop; bank fields are readable only by the shop + admin. The customer-facing browse/booking surfaces and the bookings table are greyed-out (they arrive in M1.2).](assets/architecture-m1.png)

How the diagram maps to M1.1:
- **You → Cowork (claude code) → Repo (GitHub) → Product Site (Vercel):** the onboarding form + `/shop/bookings` ship through the same push loop M0 established (token recalled from Secrets Manager).
- **Cowork → Database (Supabase):** `platform_settings` + the new tenant tables + their RLS policies are applied via `apply_migration` (Step 3); `get_advisors` checks them (Step 4).
- **`profiles.role = 'shop'`:** the seam from M0 — the role tab + "Become a shop" flow set it; the shop surfaces gate on it.
- **Greyed-out (M1.2):** `bookings`, `/barbers`, `/barbers/[id]`, the booking dialog. Don't build them here.

## Conversational flow

You (Claude Code) drive the student through **6 steps**, in order. Don't dump them all at once — after each step, **wait for confirmation** before moving on.

1. Wire the shop role: the sign-up tab + a "Become a shop" flow set `profiles.role = 'shop'`
2. Apply the `platform_settings` / `barbers` / `services` / `bookable_slots` / `barber_photos` migration (`apply_migration`)
3. Add the RLS policies (own-row CRUD; shop-level bank fields on `profiles`, shop + admin only; `platform_settings` world-read/admin-write)
3a. Create the `barber-photos` Storage bucket + its Storage policies
4. RLS sanity check with `get_advisors`
5. Build the shop onboarding (shop payout settings + create/manage one-or-more barbers + photo uploader)
6. Build `/shop/bookings` — slot publisher + service & price editor → run the checklist

---

### Step 1 — Wire the shop role (sign-up tab + "Become a shop")

M0 already captures `customer` / `shop` from the sign-up role tab into `profiles.role`. M1.1 makes that role **mean something** and adds a path for an existing customer to upgrade. Paste this into **Lovable**:

> 在現有的網站上加入「開店 / Become a shop」的流程，並讓註冊頁的角色分頁真正生效：
>
> 1. 登入後的導覽列／個人選單裡，若使用者目前是 **customer**，顯示一個 **「開店 / Become a shop」** 按鈕；點下去帶他到理髮店上架表單（Step 5 會建）。
> 2. 註冊頁那個 **Customer ↔ Shop 分頁**：選 Shop 註冊的人（`role = 'shop'`），登入後應該直接看到理髮店後台入口（而不是顧客的瀏覽頁）。
> 3. 角色判斷一律讀 **Supabase `profiles.role`**（M0 已建好的欄位），不要在前端自己存一份角色狀態。`shop` 才能看到 `/shop/bookings`；`customer` 看到的是「開店」入口。
>
> 注意：這一步**先不要**動任何資料表 schema（barbers/services 等我會用 migration 建）；你只要接 UI 與 `profiles.role` 的判斷。

**Note for Claude Code:** the actual flip of `customer → shop` for an upgrading user should happen through Supabase, **gated by RLS** (a user updating their **own** `profiles.role` to `shop`). It must **never** allow `admin` — that allowlist stays `customer | shop | admin` and the sign-up/upgrade path only ever writes `customer | shop`; `admin` is promoted only by a one-off migration in the M2.1 prereq ([[supabase-best-practice]]). The "Become a shop" button is just the UI seam; the table that makes it real is `barbers`, built next.

---

### Step 2 — Apply the schema migration (`platform_settings` / `barbers` / `services` / `bookable_slots`)

This is the milestone's core. Apply it as **one Supabase migration** via `mcp__claude_ai_Supabase__apply_migration` (name it e.g. `m1_1_barber_shop_schema`). **Never** type these into the SQL editor as ad-hoc statements — [[supabase-best-practice]] requires a migration file so the change is reviewable and replayable.

```sql
-- ── platform_settings: ONE row of platform-wide config (currency + slot length).
--    The single_row CHECK + a fixed boolean PK keep it to exactly one row. It replaces
--    the hard-coded "TWD is zero-decimal" + "30-min" constants — money columns are
--    integers in platform_settings.currency, and a bookable slot is slot_minutes long. ──
create table if not exists public.platform_settings (
  id            boolean primary key default true check (id),   -- always true → at most one row
  currency      text not null default 'twd',                   -- ISO-ish currency code; money columns are integers in THIS currency
  currency_minor_units integer not null default 0,             -- 0 = zero-decimal (TWD/JPY); 2 = cents (USD/EUR). Drives Stripe unit_amount math.
  slot_minutes  integer not null default 30 check (slot_minutes > 0),  -- the bookable_slots unit length
  updated_at    timestamptz not null default now()
);
insert into public.platform_settings (id) values (true) on conflict (id) do nothing;  -- seed the single row

-- ── barbers: one SHOP can run MANY barbers. No bank columns here — the payout
--    target lives on profiles (shop level), since one shop = one bank account. ──
create table if not exists public.barbers (
  id          uuid primary key default gen_random_uuid(),
  shop_id      uuid not null references public.profiles(id) on delete cascade,  -- NOT unique: many barbers per shop
  name text not null,           -- display name of the barber
  intro       text,
  address     text,
  created_at  timestamptz not null default now()
);
create index if not exists idx_barbers_shop on public.barbers(shop_id);  -- look up all of a shop's barbers

-- ── services: the bookable menu for a barber ──
create table if not exists public.services (
  id           uuid primary key default gen_random_uuid(),
  barber_id      uuid not null references public.barbers(id) on delete cascade,
  name         text not null,
  category     text not null check (category in ('cut','color','perm','beard')),
  price          integer not null check (price >= 0),          -- integer amount in platform_settings.currency (e.g. TWD = whole units)
  required_slots integer not null check (required_slots >= 1),  -- # of CONSECUTIVE bookable_slots this service needs (decoupled from slot minutes)
  created_at     timestamptz not null default now()
);

-- ── bookable_slots: published time windows. NO status column — a slot is just a
--    bookable time. Its availability is DERIVED: a slot is free unless a LIVE
--    booking references it (M1.2 anti-join against booking_slots). Each slot is a
--    platform_settings.slot_minutes-long window (configurable, default 30 min). ──
create table if not exists public.bookable_slots (
  id         uuid primary key default gen_random_uuid(),
  barber_id  uuid not null references public.barbers(id) on delete cascade,
  starts_at  timestamptz not null,
  ends_at    timestamptz not null,
  created_at timestamptz not null default now(),
  check (ends_at > starts_at)
);

-- ── barber_photos: the barber's sample hairstyle portfolio (past work). ──
--    Files live in the 'barber-photos' Storage bucket; this table holds metadata.
create table if not exists public.barber_photos (
  id           uuid primary key default gen_random_uuid(),
  barber_id      uuid not null references public.barbers(id) on delete cascade,
  storage_path text not null,           -- e.g. '<barber_id>/<uuid>.jpg' in the barber-photos bucket
  caption      text,
  is_featured  boolean not null default false,  -- the set M4's AI bio-writer will read
  sort_order   integer not null default 0,
  created_at   timestamptz not null default now()
);

create index if not exists idx_services_barber   on public.services(barber_id);
create index if not exists idx_bookable_slots_barber       on public.bookable_slots(barber_id);
create index if not exists idx_photos_barber      on public.barber_photos(barber_id);
create index if not exists idx_photos_featured  on public.barber_photos(barber_id, is_featured);
```

> **Note for Claude Code:** the photo **files** go in a Supabase **Storage** bucket named `barber-photos` (created in Step 3a below, not via this SQL); this `barber_photos` table only stores the **path + metadata**. Keeping the `is_featured` flag here (not in Storage) is deliberate — M4's AI bio-writer reads `where is_featured = true` to pick which work to describe. Don't store image bytes in Postgres.

> **Note for Claude Code:** `price` is a plain **integer in `platform_settings.currency`** — no `_twd` suffix, because the currency is config, not baked into the column name. The default currency is `twd` with `currency_minor_units = 0` (zero-decimal), so when M2.1 builds the Stripe line item, `unit_amount = price * 10^currency_minor_units` (TWD default: ×1; USD would be ×100). The webhook/checkout reads `currency_minor_units` from `platform_settings` instead of assuming TWD ([[supabase-best-practice]], and the Stripe rule in M2.1). Note `bookable_slots` has **no status column** — a slot is just a `slot_minutes`-long time window; whether it's bookable is derived in M1.2 from whether a live booking references it (the `bookings.status` lifecycle, not a slot field).

---

### Step 3 — Add the RLS policies (own-row CRUD; bank fields shop + admin only; platform_settings world-read/admin-write)

RLS is the heart of multi-tenancy here: **barber A must never touch barber B's data**, and **bank numbers must not leak**. Apply this as a **second migration** (`m1_1_barber_rls`):

```sql
alter table public.platform_settings enable row level security;
alter table public.barbers          enable row level security;
alter table public.services       enable row level security;
alter table public.bookable_slots enable row level security;
alter table public.barber_photos    enable row level security;

-- helper: is the current user an admin?
create or replace function public.is_admin()
returns boolean language sql stable security definer set search_path = public as $$
  select exists (select 1 from public.profiles p
                 where p.id = auth.uid() and p.role = 'admin');
$$;

-- ── platform_settings: world-readable (it's just config the UI needs), admin-only write. ──
create policy "platform_settings_read"  on public.platform_settings for select using (true);
create policy "platform_settings_admin" on public.platform_settings for all using (public.is_admin());

-- ── barbers ──
-- everyone may read a barber's public fields (name/intro/address) for browsing
-- in M1.2. There are no bank columns on barbers anymore — the payout target is on
-- profiles (shop level), guarded by the profiles bank-field policy below.
create policy "shops_select_public" on public.barbers
  for select using (true);
create policy "shops_insert_own" on public.barbers
  for insert with check (auth.uid() = shop_id);
create policy "shops_update_own" on public.barbers
  for update using (auth.uid() = shop_id);
create policy "shops_delete_own" on public.barbers
  for delete using (auth.uid() = shop_id);

-- ── services: a barber CRUDs only services under a barber they own; anyone may read ──
create policy "services_select_public" on public.services
  for select using (true);
create policy "services_write_own" on public.services
  for all using (
    exists (select 1 from public.barbers s where s.id = services.barber_id and s.shop_id = auth.uid())
  ) with check (
    exists (select 1 from public.barbers s where s.id = services.barber_id and s.shop_id = auth.uid())
  );

-- ── bookable_slots: same ownership rule; anyone may read (M1.2 needs to show open slots) ──
create policy "slots_select_public" on public.bookable_slots
  for select using (true);
create policy "slots_write_own" on public.bookable_slots
  for all using (
    exists (select 1 from public.barbers s where s.id = bookable_slots.barber_id and s.shop_id = auth.uid())
  ) with check (
    exists (select 1 from public.barbers s where s.id = bookable_slots.barber_id and s.shop_id = auth.uid())
  );

-- ── barber_photos: anyone may read (public portfolio); only the owning shop writes ──
create policy "photos_select_public" on public.barber_photos
  for select using (true);
create policy "photos_write_own" on public.barber_photos
  for all using (
    exists (select 1 from public.barbers s where s.id = barber_photos.barber_id and s.shop_id = auth.uid())
  ) with check (
    exists (select 1 from public.barbers s where s.id = barber_photos.barber_id and s.shop_id = auth.uid())
  );

-- ── public browse projection of barbers (no sensitive columns; barbers has none now) ──
create or replace view public.barbers_public as
  select id, shop_id, name, intro, address, created_at
  from public.barbers;

-- ── bank fields live on PROFILES (shop level) — shop + admin only. ──
-- M0 already enabled RLS on profiles with profiles_select_own (auth.uid() = id).
-- Add an admin read path so the M2.2 payout page can see a shop's bank account.
create policy "profiles_select_admin" on public.profiles
  for select using (public.is_admin());
-- Result: a profile's bank_account_* is readable ONLY by that shop (select_own)
-- or an admin (select_admin). No public path exposes it — and barbers_public has
-- no bank columns at all, so the customer browse never touches them.
```

> **Note for Claude Code:** the decisive rule is **bank fields readable only by shop + admin** — and they now live on **`profiles`** (shop level), not `barbers`, because one shop can run many barbers but has a single payout bank account. Two facts enforce it: (1) `barbers` / `barbers_public` have **no bank columns at all**, so the customer browse can't leak them; (2) `profiles.bank_account_*` is gated by `profiles_select_own` (the shop) + `profiles_select_admin` (the payout admin). Never surface `bank_account_number` in any list a customer can load. The checklist's "bank fields not world-readable" test verifies exactly this ([[supabase-best-practice]]).

---

### Step 3a — Create the `barber-photos` Storage bucket (+ its Storage policies)

The `barber_photos` table holds metadata; the image **files** live in a Supabase **Storage** bucket. Create a **public** bucket `barber-photos` (public so the M1.2 gallery and the M4 AI step can read the images by URL), with policies that let **only the owning shop upload/delete** under their own `barber_id` prefix.

Create the bucket (Supabase MCP, or dashboard → Storage → New bucket → name `barber-photos`, **Public** ✓). Then apply the Storage RLS as a migration (`m1_1_barber_photos_storage`):

```sql
-- Public READ for the bucket (anyone can view a barber's portfolio image by URL)
create policy "barber_photos_read" on storage.objects
  for select using ( bucket_id = 'barber-photos' );

-- Only the owning shop may UPLOAD/UPDATE/DELETE, and only under their own
-- barber's path prefix '<barber_id>/...'. (path is stored as storage.objects.name;
-- the first path segment must be a barber the caller owns.)
create policy "barber_photos_write_own" on storage.objects
  for all using (
    bucket_id = 'barber-photos'
    and exists (
      select 1 from public.barbers s
      where s.shop_id = auth.uid()
        and s.id::text = split_part(storage.objects.name, '/', 1)
    )
  ) with check (
    bucket_id = 'barber-photos'
    and exists (
      select 1 from public.barbers s
      where s.shop_id = auth.uid()
        and s.id::text = split_part(storage.objects.name, '/', 1)
    )
  );
```

> **Note for Claude Code:** the upload path convention is **`<barber_id>/<uuid>.<ext>`** — the first segment is the barber id, which the Storage policy checks against ownership, so barber A can't write into barber B's folder even though the bucket is public-read. Store that same path in `barber_photos.storage_path`. The bucket is **public-read on purpose** (portfolio images are meant to be seen, and M4's AI step needs URL access) — there's nothing sensitive in a haircut photo; the sensitive data (bank fields) is on `profiles`, not here.

---

### Step 4 — RLS sanity check with `get_advisors`

Before building UI on top, confirm Supabase agrees the tables are locked down. Run `mcp__claude_ai_Supabase__get_advisors` (type **`security`**) and read the report:

```text
mcp__claude_ai_Supabase__get_advisors  →  type: "security"
```

What you want to see:
- **No `rls_disabled_in_public` warning** for `platform_settings`, `barbers`, `services`, `bookable_slots`, `barber_photos` (RLS is enabled — Step 3), and the `barber-photos` Storage bucket has its read/write policies (Step 3a).
- If the advisor flags the `barbers_public` **view** as `security_definer` / exposed, that's expected for a read-only public projection (it deliberately drops bank fields). Note it; it's not a leak.
- Resolve any **real** finding (e.g. a table with RLS off, or a policy that's `using (true)` for `update`/`delete`) before moving on.

> **Note for Claude Code:** `get_advisors` is the cheap, authoritative gate — run it after **every** schema/RLS migration in this course, not just here. A green security advisor + the checklist's cross-barber-edit test together prove the tenant isolation actually holds.

---

### Step 5 — Build the shop onboarding (shop payout settings + create/manage barbers + sample photos)

Now the UI. A shop can run **many barbers**, so this is two pieces: a **shop-level payout settings** form (writes `profiles.bank_account_*`) and a **barber create/manage** form (writes one or more `barbers` rows). Paste into **Lovable**:

> 建立「理髮店上架 / Shop onboarding」區，只有登入且 `profiles.role = 'shop'`（或剛從「開店」進來的人）能看到。**一個 shop 可以開很多位理髮師。**
>
> **A. 撥款設定 / Payout settings（shop 層級，全部理髮師共用）** — 寫到 **`profiles`**（目前登入者那一筆）：
> - **匯款戶名 bank_account_name**（**必填**）
> - **匯款帳號 bank_account_number**（**必填**）
> - 標註 **「先用測試資料即可 / use test data first」**。這是 shop 的收款帳號，月結時 admin 會把這個 shop 名下所有理髮師的 80% 合併成一筆轉給他（M2.2）。
>
> **B. 我的理髮師檔案 / My barbers（可有多位）** — CRUD **`barbers`**（`shop_id = 目前登入者`）：
> - 列出我名下所有理髮師，提供 **「新增一位理髮師 / Add another barber」**。
> - 每位理髮師的欄位：**名稱 name**（必填）、**簡介 intro**（選填，多行；提示「之後可用 AI 從作品照自動產生」M4 彩蛋）、**地址 address**（選填）。
> - 可編輯 / 刪除我自己的理髮師。**不要**有「只能一位理髮師」的限制——同一個 shop 可以建立多筆 `barbers`。
>
> **C. 作品照上傳 / Sample hairstyle photos（每位理髮師各自一組）** — 讓理髮師上傳過去的髮型作品照：
> - 在某一位理髮師底下上傳多張圖片，存到 Supabase Storage 的 **`barber-photos`** bucket，路徑用 **`<barber_id>/<uuid>.<ext>`**。
> - 每上傳一張，在 `barber_photos` 寫一筆（`barber_id`、`storage_path`、選填 `caption`、`sort_order`）。
> - 可勾選 **「精選 / Featured」**（寫 `is_featured=true`）——之後 M4 會用精選作品讓 AI 自動寫 bio。
> - 可刪除自己的作品照（同時刪 Storage 檔案與 `barber_photos` 那一筆）。顯示縮圖牆管理排序與精選。
>
> **規則：**
> - **撥款銀行資訊為必填** — shop 沒填就不能收款（月結靠它撥款）。
> - **銀行欄位在 `profiles`，絕對不要**出現在任何公開頁面或清單；只有本人和 admin（M2.2 撥款頁）能讀。作品照則相反——是公開的，會在 M1.2 的理髮師詳情頁展示。
> - 所有讀寫都走 Supabase client + Storage + RLS；不要用 service-role key 在前端。

**Note for Claude Code:** the model is **one shop → many barbers**, so do NOT add a "one barber only" guard (the old `shop_id UNIQUE` is gone). Two opposite visibility rules: **bank fields are on `profiles`, shop+admin-only** (the shop's single payout account, never public), while **`barber_photos` are public-read** (the portfolio is meant to be seen; M1.2 renders it on `/barbers/[id]`). Upload to the `barber-photos` bucket under the `<barber_id>/...` prefix so the Step 3a Storage policy authorizes the write. The **`is_featured`** flag is the set the **M4 egg unit** (`m4-ai-autofill-barber`) feeds to a vision model to draft the bio. Don't wire any real payout integration here; payouts are a manual admin bank transfer recorded in M2.2, **rolled up per shop**.

---

### Step 6 — Build `/shop/bookings` (slot publisher + service & price editor) → run the checklist

The shop's control room. Paste into **Lovable**:

> 建立理髮店後台頁 **`/shop/bookings`**，只有 `profiles.role = 'shop'` 且擁有理髮師的人能進。兩個區塊：
>
> **A. 服務與價格編輯 / Services & price editor** — CRUD `services`（屬於我的 `barber_id`）：
> - 欄位：**名稱 name**、**分類 category**（下拉：`cut` / `color` / `perm` / `beard`）、**價格 price**（整數，金額單位是 `platform_settings.currency`，預設 TWD 是 zero-decimal，**不要乘 100**）、**所需時段數 required_slots**（整數，這個服務需要幾個連續的時段，例如一個時段預設 30 分鐘、要 90 分鐘的服務就填 `3`）。
> - 可新增 / 編輯 / 刪除我自己的服務。
>
> **B. 可預約時段發布 / Publish bookable slots** — CRUD `bookable_slots`（屬於我的 `barber_id`）：
> - 選日期 + 起訖時間建立時段。**每個時段是 `platform_settings.slot_minutes` 長度的單位（可設定，預設 30 分鐘）** —— 想開一段較長的可預約時間，就發布一連串連續的時段（UI 可協助一次產生一天份）。M1.2 的一筆預約會佔掉 **N = service.required_slots** 個連續的時段（直接用 required_slots，不是 ceil(時長/30)）。
> - **`bookable_slots` 沒有 status 欄位** —— 一個時段就是一段「可被預約的時間窗」。
> - 列出我已發布的時段，可編輯/刪除 **還沒有人預約** 的時段（也就是還沒有 live `bookings` 透過 `booking_slots` 指到它的時段；這個判斷在 M1.2 用 anti-join 做）。
> - **先不要**做付款或預約 —— 預約與其狀態（`pending_payment` → `paid`）是 M1.2 / M2.1 在 `bookings` 上做的；M1.1 只發布時間窗。
>
> 全部透過 Supabase client + RLS：我只看得到、改得了 **我自己** 的服務與時段，別人的看不到也改不了。

Then push to GitHub (recall the token from Secrets Manager — don't re-paste) and let Vercel redeploy. Finally verify M1.1:

> **ask:** "Run the `m1.1-seller-setup-checklist` skill."

---

## Things to watch out for (common mistakes)

1. **Skipping the prereq** — run `[[m1.1-seller-setup-prerequisite]]` first. If M0's `profiles.role` isn't actually working, every gate in M1.1 silently fails open or closed.
2. **Editing the DB without a migration** — every schema/RLS change goes through `apply_migration`. No ad-hoc SQL-editor `UPDATE`s ([[supabase-best-practice]]).
3. **Forgetting to enable RLS** — a `create table` without `enable row level security` is world-open. `get_advisors` (Step 4) catches this; don't skip it.
4. **Bank fields leaking into a public view** — the single most important RLS rule here. The bank fields live on **`profiles`** (shop level) and are gated to shop + admin; `barbers` / `barbers_public` have **no bank columns at all**. Never put `bank_account_number` in a customer-loadable list.
5. **`price` × 100** — the default currency (`twd`) is **zero-decimal** (`currency_minor_units = 0`). Store the whole-unit `price` in `platform_settings.currency`; M2.1's Stripe `unit_amount = price * 10^currency_minor_units` (TWD default: ×1).
6. **Re-adding a one-barber-per-shop limit** — the model is now **one shop → MANY barbers**. `barbers.shop_id` is **NOT** unique (there's an index `idx_barbers_shop` instead). The onboarding "My barbers" list must allow "Add another barber"; don't make the form edit-a-single-barber or block a second insert.
6a. **Photo upload path not matching the Storage policy** — files MUST go under `<barber_id>/...` in the `barber-photos` bucket, or the Step 3a `barber_photos_write_own` policy rejects the upload. Store that same path in `barber_photos.storage_path`. Deleting a photo must delete BOTH the Storage object and the `barber_photos` row.
7. **Letting `role` flip to `admin`** — the upgrade path writes `shop` only. `admin` is promoted via a one-off migration in the M2.1 prereq, never self-served.
8. **Building M1.2 here** — no `bookings` table, no `/barbers/[id]`, no booking dialog. Those are `[[m1.2-buyer-setup]]`.
9. **Service-role key in the front-end** — never. The barber UI uses the publishable key + RLS; service-role stays in Vercel server env ([[supabase-best-practice]], [[lovable-best-practice]]).
10. **Re-pasting the GitHub token** — it's cached in Secrets Manager from M0; recall it for the push.

## Expected duration

45–70 minutes — most of it in Lovable (the onboarding form with the photo uploader + `/shop/bookings`) and verifying the RLS isolation. The migrations + the Storage bucket + `get_advisors` are fast.

## Next step

When `m1.1-seller-setup-checklist` is green, tell the student:
「M1.1 完成了！一個 shop 現在可以上架**多位理髮師**、各自列出服務與價格、上傳作品髮型照、發布可預約的時間窗（時段本身沒有狀態），並在 shop 層級填好撥款銀行帳號，而且 RLS 確保每個人只能改自己的資料、銀行欄位（在 profiles）只有本人和 admin 讀得到、作品照只有本人能上傳但大家都看得到。準備好的話跟我說『啟動 M1.2』，我們來做顧客端：瀏覽所有理髮師、進理髮師詳情頁（含作品照輪播 carousel）、用彈出視窗選服務挑時段下訂（先不收款，金流是 M2.1）。」
Then load `[[m1.2-buyer-setup]]`.

## Reference

- Supabase RLS: https://supabase.com/docs/guides/database/postgres/row-level-security
- Supabase migrations: https://supabase.com/docs/guides/deployment/database-migrations
- Supabase advisors (security lints): https://supabase.com/docs/guides/database/database-advisors
- Postgres views: https://www.postgresql.org/docs/current/sql-createview.html
- [[supabase-best-practice]] · [[lovable-best-practice]] · [[m1.1-seller-setup-prerequisite]] · [[m1.2-buyer-setup]]
