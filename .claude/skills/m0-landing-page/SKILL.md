---
name: m0-landing-page
description: 抽成制理髮師預約平台 Milestone 0 — generate the v1 hair-themed landing page with Lovable (a Novara-style marketplace, NOT a 3-card hero), push to GitHub, set up the Cowork project + AWS/Vercel/Supabase connectors, cache the GitHub token in AWS Secrets Manager, deploy to Vercel, and swap auth to the student's own Supabase with a Customer↔Shop role tab + a `profiles.role` stub. Use when the student says "啟動 M0", "start M0", "build the M0 landing page", "理髮師預約平台的入口網站", or any variant that maps to "先做出一個能登入的理髮預約網站".
---

# M0 — Landing Page + Sign-in（先有一個能登入的理髮預約入口網站）

## What this skill does

Walks the student through Milestone 0 end-to-end, in the order the course actually runs it: **Lovable (hair-themed marketplace landing + role-tab auth) → GitHub → make repo public → set up the Cowork project + connectors (AWS / Vercel / Supabase) → cache the GitHub token in Secrets Manager → convert to a deployable build & push → deploy to Vercel → swap auth to the student's own Supabase + add the `profiles.role` stub → bootstrap the skills & run the checklist.**

A shop here is the aggregator account that runs one-or-more barbers (a one-man shop works too).

By the end the student has:

1. A v1 landing page (`/`) for the **barber booking platform** — a **Novara-style hair marketplace** look (NOT the old 3-feature-card hero): a serif hero with split hairstyle photos + a search bar, a trust logo strip, a 4-icon feature row, and a **"Popular" grid of featured barbers** (barbers are the product — not separate "styles") — plus a `/login` auth page with a **Customer / Barber role tab** (the labels; the values written are `customer` / `shop`) and a **role-aware** authenticated post-login shell at `/barbers` (a "coming soon" placeholder that differs for a customer vs a barber, with a small "barber" pill shown for shop accounts).
2. A **GitHub repo** (created by Lovable) holding that code, made **public**.
3. A **Cowork desktop project** with **AWS API MCP**, **Vercel Connect**, and **Supabase Connector** installed — the workbench every later milestone uses.
4. The **GitHub PAT cached in AWS Secrets Manager** (`barber-project/github`), so this and every later session **recall the same token to modify the repo** instead of re-pasting one.
5. A **live Vercel URL** that auto-deploys the repo on every push.
6. **Auth wired up** — register / log in / log out. v1 starts on **Lovable Cloud** (fastest path to a working sign-up), then **swaps to the student's own Supabase project**.
7. A **`profiles` table with a `role` column** (`customer` default, set `shop` via the sign-up tab) plus a `display_name` column — created by an `on_auth_user_created` trigger — the seam M1.1 (shop) and the M2.1 prereq (admin) build on. Unlike the flight course, **Supabase here will hold real app data** from M1.1 on (barbers, bookings, payouts), not auth-only.

**Out of scope for M0:** the barber/schedule tables, the booking flow, Stripe, the admin payout page, and the custom domain. Those are M1.1 and later.

## When to load this skill

Trigger phrases:
- "啟動 M0" / "start M0" / "begin M0"
- "幫我蓋 M0 的 landing page"
- "理髮師預約平台的入口網站" / "理髮預約網站"
- Any prompt referencing "先做出一個能登入的網站"

Do NOT load this skill for M1.1+ — they have their own skills.

## Execution mode (Cowork-first)

This milestone is written for **Cowork on Desktop** — the course's primary environment, and Steps 4–6 set up the Cowork project and its connectors. The **one** part that needs a local shell is writing `~/.aws/credentials` (Step 4), and you hand the student a Claude Code CLI / desktop prompt for exactly that. Everywhere else, the AWS/Vercel/Supabase **connectors** do the work from inside Cowork.

If a student is on a pure local CLI instead, the `aws …` commands shown here apply directly (they're the same calls the AWS MCP makes); the connector-install bullets become "make sure `gh` / `vercel` / `aws` are authenticated locally."

## Required external accounts

| # | Service | Used for |
|---|---|---|
| 1 | Lovable (`lovable.dev`) | Generates the v1 hair-marketplace UI + scaffolds role-tab auth |
| 2 | GitHub | Lovable creates a repo here; you make it public; Vercel imports from it. A **fine-grained PAT** (single repo, Contents: Write) gets cached in Secrets Manager (Step 5) |
| 3 | AWS | **Secrets Manager** holds the GitHub token so future sessions reuse it; **Route 53** is used in M3. Reached via the **AWS API MCP** connector (Cowork) or the `[default]` profile (CLI) |
| 4 | Vercel | Auto-deploys the GitHub repo; reached via the **Vercel Connect** connector |
| 5 | Supabase (`supabase.com`) | `auth.users` + the app's real data tables (from M1.1); reached via the **Supabase Connector** |

If any of these is missing, **stop and ask the student to register first.**

## The product being built

M0 builds the **barber booking platform** — the brand is **"Barberly"** (use it consistently from here on; only change it if the student explicitly wants their own name). The v1 is a real, working signed-in SaaS: the landing page sells the booking experience, and the auth page lets a user sign up as a **customer** or a **shop** (the aggregator account that runs one-or-more barbers). After signing in, the user lands on `/barbers` (browse) or a placeholder shell. The barber/schedule and booking features come in M1.1+.

## Architecture

![Barber platform architecture (M0) — the student drives Cowork (claude code), which pushes to the GitHub repo; from the repo the code flows out to the Vercel-hosted Product Site and the student's own Supabase (auth + a profiles.role stub). AWS Secrets Manager caches the GitHub token. The Lovable landing page is the upstream generator; M0 migrates hosting onto Vercel and auth onto the student's own Supabase.](assets/architecture-m0.png)

How the diagram maps to M0:
- **You → Cowork (claude code) → Repo (GitHub):** the student drives Cowork; Cowork recalls the cached GitHub token (Step 5) and pushes to the repo (Step 6).
- **Repo → Product Site (Vercel host):** the repo auto-deploys to Vercel on every push (Step 7).
- **Repo → Database (Supabase):** the front-end talks to the student's own Supabase project for `auth.users` and the `profiles` stub after the swap (Step 8–9).
- **AWS Secrets Manager:** caches the GitHub PAT (`barber-project/github`) for reuse across sessions (Step 5).

## Conversational flow

You (Claude Code) drive the student through **10 steps**, in order. Don't dump them all at once — after each step, **wait for confirmation** before moving on.

1. Build v1 on Lovable (hair marketplace + role-tab auth)
2. Check v1 on Lovable
3. Migrate code → GitHub, make repo public
4. Set up the Cowork project + connectors (AWS / Vercel / Supabase)
5. Cache the GitHub token in Secrets Manager (`barber-project/github`)
6. Convert to a deployable build + push to GitHub
7. Deploy to Vercel
8. Swap auth → the student's own Supabase (+ Vercel env vars)
9. Add the `profiles.role` stub (trigger on signup)
10. Bootstrap the skills into the project → run the checklist

---

### Step 1 — Build v1 on Lovable (attach the rule skills + paste the full prompt)

There is **no separate "create a blank project" step** — that just burns a Lovable generation. Do it all in the **first** message:

1. 到 https://lovable.dev/ 登入，**+ New project**.
2. **Attach the two rule files as context** to this first prompt: `lovable-best-practice` and `supabase-best-practice`. They tell Lovable the non-negotiables up front — ship a clean modern build, PascalCase components, don't wire your own Supabase yet — so v1 obeys them on the first generation.
3. Paste the **full prompt below verbatim as that same first message** and generate. One generation → the whole landing page + role-tab auth + post-login shell.

⚠️ **Don't click Connect Supabase / Connect GitHub from inside this first generation.** v1 uses Lovable Cloud; GitHub comes in Step 3, your own Supabase in Step 8. (Why Lovable Cloud first: [[lovable-best-practice]] Rule 1.) Re-rolls burn the free quota ([[lovable-best-practice]] Tip 2).

**The full prompt (paste verbatim):** — replace `Barberly` with your chosen brand.

> Build a SaaS landing page + authenticated app shell for **Barberly**, a **barber / hair-stylist booking marketplace** where customers find a stylist and book an appointment online — and where barbers list a profile and publish their schedule. Targeted at people who want to discover a good barber and book a slot in a few taps. The site is **barber-centric: barbers are the product.** A barber offers services (Cut / Color / Perm / Beard) at a starting price; "styles" are not separate listings.
>
> Design language — match this reference look closely (a clean, modern, editorial e-commerce marketplace):
> - Warm **beige / cream** palette on near-white backgrounds, with a dark near-black accent for buttons.
> - **Serif display headlines** (refined fashion-editorial serif) paired with a clean sans-serif body (Inter or similar). Generous whitespace, rounded cards.
> - Mobile responsive, tasteful subtle fade-in animations.
>
> The site must include:
>
> 1. A public landing page (`/`) with, in this order:
>    - **Navbar**: the wordmark **"Barberly"** top-left; a search field; and a dark rounded-pill **"Login"** button top-right. **No other nav links in v1** (no *Styles / Barbers / Book / Contact* tabs — booking and barber browsing arrive in later milestones, and dead in-page anchors should not appear).
>    - **Hero**: an eyebrow label "New Look", a big serif headline **"Style with Confident Hair"**, **two styled-hair / barber photos split left and right** of the headline, and a centered **search bar** with placeholder "Find your stylist or search a style", plus filter chips: *All / Cut / Color / Perm / Beard*.
>    - **Trust logo strip**: a horizontal row of partner-salon / brand logos directly under the hero.
>    - **Feature row (exactly 4 icon + label cells)** under a heading "Best booking experience": **Verified Barbers**, **Instant Booking**, **Secure Payment**, **Top-Rated Styles**.
>    - **"Popular" grid**: a heading "Popular" and a card grid of **featured barbers (not styles)**. Each card shows a **barber portrait**, the **barber's name**, **shop / location**, a row of **service chips** (subset of Cut / Color / Perm / Beard), a **star rating with review count**, a **"from $—" starting price**, and a small **"Popular" badge**. Cards are clickable blocks with a hover lift (destination is a placeholder for v1).
>    - **Footer** with copyright "© 2026 Barberly".
>
> 2. Authentication using Lovable's built-in Supabase-style auth (Lovable Cloud is fine for v1; we'll swap to a user-owned Supabase project later):
>    - A combined **Sign Up / Sign In page at `/login`** with email + password.
>    - On the **Sign Up** form, include a **role selector as a TAB / segmented toggle at the top of the form** with two options labeled **"Customer"** (role value `customer`) and **"Barber"** (role value `shop`). Default to Customer. Capture the choice in form state and pass it into the sign-up call's user metadata (`options.data.role`).
>    - Sign Out functionality.
>    - Email confirmation disabled for v1.
>
> 3. After signing in, land the user on **`/barbers`**, a simple authenticated shell:
>    - A **header** with the **Barberly wordmark** on the left, and on the right: the greeting **Hi {user.email}**, then — **only if** the account's `user_metadata.role === "shop"` — a small rounded **pill tag** rendered next to the email reading **"barber"** (no tag is rendered for Customer accounts), then a **Sign Out** button.
>    - **Body content is role-aware:**
>      - **Customer:** 「附近的理髮師即將上線 — 下一個里程碑會加上瀏覽與預約功能。」 / "Barbers near you are coming soon — browse & booking arrive in the next milestone."
>      - **Barber:** 「理髮師後台即將上線 — 下一個里程碑會加上個人檔案、服務項目與排班管理。」 / "Your barber dashboard is coming soon — profile, services & schedule arrive in the next milestone."
>
> Out of scope for v1: the barber onboarding form, the barber / services / schedule tables, the booking flow, payments, and any custom database tables (do NOT create `barbers` / `bookings` / `profiles` tables yet — only use Supabase's default `auth.users`; capture the chosen role in auth user metadata only). Those come in later milestones. Stick to landing page + role-tab auth + the **role-aware `/barbers` placeholder shell**.

**Why this prompt is shaped this way:** it pins the **Novara-style marketplace sections** (hero+search, logo strip, 4-icon row, popular grid) so you don't re-roll for layout; it puts the **role tab** on sign-up (the seam Step 9 turns into `profiles.role`); and it captures the role in **auth metadata only** (no custom tables yet — those are M1.1, see [[supabase-best-practice]]).

---

### Step 2 — Check the v1 on Lovable

Before touching GitHub, confirm the generation is good:

- Get the **Lovable preview URL** (e.g. `https://id-preview--<uuid>.lovable.app/`).
- Check the **landing-page style** — serif hero with the two split photos + search bar, the logo strip, the **4-icon feature row**, the **Popular grid of featured barbers** (barber cards — portrait, name, shop/location, service chips, rating, "from $—"), footer. It must look like the marketplace reference, **NOT** a 3-feature-card layout. The navbar has **only** the wordmark + search + Login (no dead *Styles / Barbers / Book / Contact* anchors).
- Check the **sign-up tab works** — the **Customer / Barber** toggle is visible on sign-up; signing up as either lands you on `/barbers` with 「Hi {email}」. Confirm the post-login shell is **role-aware**: a **Barber** account shows the **"barber" pill** next to the email + the dashboard-coming-soon copy; a **Customer** account shows no pill + the browse-coming-soon copy.

Only move on once the preview looks right and auth works — re-prompting in Lovable now is cheaper than after GitHub/Vercel are wired.

---

### Step 3 — Migrate code → GitHub, make repo public

> 在 Lovable 點 **GitHub** 按鈕（左下角選單）→ **Connect** → 授權 → 讓 Lovable 建立一個新的 repo（名稱可用 `barber-platform`）。

Then confirm the repo exists and make it public:
- Open the new repo (e.g. `https://github.com/<you>/barber-platform`) and confirm Lovable's files are there.
- **Settings → Danger Zone → Change repository visibility → Public → confirm.** A public repo keeps the Vercel import and later tooling simple.
- **CLI:** `gh repo view <owner>/<repo> --json name,visibility,defaultBranchRef`.

Paste the **GitHub repo URL** back.

---

### Step 4 — Set up the Cowork project + connectors

This is where the course moves into **Cowork on Desktop** — the workbench every later milestone runs in.

**4.1 — Create the Cowork project.**
- New project (e.g. `barber-platform`).
- Turn on **Act without asking**.
- Settings → **Capabilities**: turn on **Search and reference chats**, **Generate memory from chat history**, **Connector discovery**.

**4.2 — AWS connector (the only part that touches a local shell).**
1. In the **AWS Console as the root user** of the course account: **IAM → Users → Create user** (suggested `admin-for-barber-platform-001`) → attach **`AdministratorAccess`** → create. Then **Security credentials → Create access key → "Command Line Interface (CLI)"** → copy the **Access key ID + Secret** (shown once).
2. **Open a Claude Code CLI / desktop session** (not Cowork) and paste this, filling in the two values:
   > I have new AWS credentials I want to configure. Please write them to my AWS credentials file. Access key ID: `XXXXX`. Secret access key: `XXXXX`. First detect Mac/Linux vs Windows for the correct path (`~/.aws/credentials` vs `%USERPROFILE%\.aws\credentials`), then write the `[default]` profile — preserving any other existing profiles. Then test with `aws sts get-caller-identity`.
3. Back in **Cowork → Customize → Connectors → search "AWS API MCP" → Install.**
4. Sanity-check from Cowork: *"How many IAM users do I have on my AWS account?"* — a number means the connector reads the `[default]` creds.

> ⚠️ Root is used **once** (to make the admin user); never use root keys after. Revoke the access key at course end.

**4.3 — Vercel connector.** Sign in to vercel.com first → Cowork → **Connectors → Vercel Connect → Install** → sanity-check: *"What projects are in my Vercel account?"*

**4.4 — Supabase connector.** Sign in to supabase.com first → Cowork → **Connectors → Supabase Connector → Install** → sanity-check: *"How many projects are in my Supabase organization?"*

---

### Step 5 — Cache the GitHub token in Secrets Manager (the re-use point)

From here on, **Claude Code itself edits the repo** (Step 6 pushes a deployable build; later milestones push API routes + migrations). It needs a **GitHub write token** — and you don't want to paste a fresh one into every new session. So **store it once in Secrets Manager and recall the same token every session.**

**5.1 — Create a scoped GitHub PAT.**
> 到 https://github.com/settings/personal-access-tokens → **Generate new token** → **Fine-grained** → **Only select repositories** → 選這個 repo → **Repository permissions → Contents → Read and write** → 產生 token（`github_pat_…`）。Contents:Write 就足以 push；不要給更大的權限。

**5.2 — Hand it to Claude Code and have it store the secret.** In Cowork the AWS MCP does the write:
> 「Here is my GitHub Personal Access Token: `github_pat_…`. Use the AWS MCP to store it in Secrets Manager so we can **re-use it in a new session** (secret name `barber-project/github`, region `us-east-1`).」

**CLI equivalent:**
```bash
aws secretsmanager create-secret --name barber-project/github \
  --secret-string '<the github_pat_… token>' --region us-east-1
# already exists → aws secretsmanager put-secret-value --secret-id barber-project/github --secret-string '<token>' --region us-east-1
```

> **This is the operational-secret pattern** ([[aws-secrets-best-practice]]): the GitHub PAT and any local-dev API key live in `barber-project/*` in Secrets Manager — **app-runtime keys (Stripe, Supabase) go to Vercel env instead** (Steps 8 + M2.1). Later sessions **discover** the token by scanning `list-secrets` for a name containing `github` and reading whatever shape they find.

**Verify:** ask the AWS MCP to list secrets and confirm a github-ish one is there (CLI: `aws secretsmanager list-secrets --region us-east-1 --query "SecretList[].Name"`).

> **Note for Claude Code:** the GitHub PAT is a **real write-credential** — keep it scoped to Contents:Write on the one repo and **never** commit it or echo it into the front-end bundle. Secrets Manager is KMS-encrypted, IAM-scoped, and `GetSecretValue` is CloudTrail-logged. ([[aws-secrets-best-practice]].)

---

### Step 6 — Convert the project to a deployable build + push to GitHub

> **The #1 deploy trap.** Lovable sometimes scaffolds an SSR app (TanStack Start / Cloudflare-targeted) that builds "successfully" on Vercel but **every route 404s**. If Lovable emitted a plain Vite SPA, convert it to a clean static SPA with a SPA fallback so deep links resolve. If Lovable emitted Next.js, keep it as a standard Next.js app (Vercel's native target) — just confirm the build command and that `/login` + `/barbers` both resolve. *(Attaching `lovable-best-practice` in Step 1 usually prevents the SSR trap — do this anyway.)*

Have Claude Code apply the fix and **push it straight to `main`**, recalling the token from the secret (no re-paste):

> 「GITHUB REPO: `<owner>/<repo>`.
> Make this project cleanly deployable on Vercel. If it's a Vite/React app, convert it to a plain **Vite SPA** (remove any TanStack Start / SSR / Cloudflare-wrangler config; use React Router; static `dist/` with a SPA fallback so `/login` and `/barbers` deep-link). If it's a Next.js app, keep it as a standard Next.js project and just confirm the build is clean. Keep all UI, the role tab, and styling unchanged.
> Then **push to my GitHub repo, overriding `main`** — recall the GitHub token from Secrets Manager (`barber-project/github`) to authenticate the push; don't ask me for it.」

This is the **first reuse** of the cached token. **Verify:** the repo builds with a standard `vite build`/`next build`, and there's no `wrangler.toml`.

---

### Step 7 — Deploy to Vercel

> 到 https://vercel.com/ → **Import** 你的 GitHub repo（**Connect GitHub account** if first time）→ Framework Preset 自動偵測（Vite 或 Next.js）→ **Deploy**。完成後把上線網址貼回來（例如 `https://barber-platform.vercel.app/`）。

**CLI deep-link check:**
```bash
curl -sS -o /dev/null -w "%{http_code}\n" https://<your>.vercel.app          # 200 for /
curl -sS -o /dev/null -w "%{http_code}\n" https://<your>.vercel.app/login     # 200
curl -sS -o /dev/null -w "%{http_code}\n" https://<your>.vercel.app/barbers   # 200 or redirect to /login
```
Expect `200` on `/` and `/login`; `/barbers` returns 200 or a redirect to sign-in (both fine). A **404** on `/login` means the SPA conversion in Step 6 didn't take.

> **Note for Claude Code:** if the build **fails**, check the framework preset. If the build is **green but routes 404**, it's the SSR-vs-SPA trap from Step 6. Read the Vercel build log before guessing.

---

### Step 8 — Swap auth from Lovable Cloud to the student's own Supabase

v1 ran on **Lovable Cloud**. Move auth to the student's **own** Supabase project — they own `auth.users` going into M1.1, and (unlike the flight course) Supabase will hold the app's real tables.

**8.1 — Create the Supabase project + grab two values:**
> 1. 到 https://supabase.com/ → **New project**（名字例如 `barber-platform`，region 選離你近的，例如 `Northeast Asia (Tokyo)`）。
> 2. 等到專案變成 **Healthy**。
> 3. **Project Settings → API**，複製 **`Project URL`** 和 **Publishable API key**（`sb_publishable_*` — 新版 anon key，瀏覽器安全、RLS 保護）。

**8.2 — Paste this prompt into Lovable** (replace the two values):

> Switch this project's backend from Lovable Cloud to the user's own Supabase project. Do NOT keep any Lovable Cloud references.
>
> 1. Find every place the project uses Lovable Cloud's Supabase client (usually `src/integrations/supabase/client.ts` for Vite, or the Supabase client setup for Next.js). Point it at:
>    - `NEXT_PUBLIC_SUPABASE_URL` (or `VITE_SUPABASE_URL`) = `<SUPABASE_URL>`
>    - `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` (or `VITE_SUPABASE_PUBLISHABLE_KEY`) = `<PUBLISHABLE_KEY>`
> 2. Update `.env` / `.env.local`; remove any `LOVABLE_CLOUD_*` vars or Lovable-owned Supabase refs.
> 3. Initialize the client once, reading from env — no hardcoded URLs.
> 4. Disconnect the Lovable Cloud integration if there's a UI toggle.
> 5. Keep the Sign Up / Sign In / Sign Out flow and the Customer/Shop role tab exactly as they are — only the backend target changes. The sign-up call must still pass the chosen role in user metadata (`options.data.role`).
>
> After this, a new sign-up should create a user in the user's own Supabase `auth.users` — verify by signing up a NEW test user in the preview, then Supabase dashboard → Authentication → Users.
>
> Final reminder to the user: add the same env vars to Vercel.

**8.3 — Finish the wiring outside Lovable:**
> 1. Supabase **Authentication → Providers**：確認 **Email 已啟用**（demo 可關掉 email confirmation）。
> 2. **在 Vercel 設同名環境變數**（Settings → Environment Variables）：`NEXT_PUBLIC_SUPABASE_URL` + the publishable key（值跟 Lovable 一致），然後 **redeploy**。env 不一致是 M0 最常見的失敗原因。

**8.4 — Verify on the live Vercel URL:** sign up a brand-new email → sign in → Supabase **Authentication → Users** shows the new user in *the student's own* project.

> **Note for Claude Code:** keep the **publishable/anon key** in the front-end (correct — RLS-protected). The **service-role key** is NOT in the browser — it only goes into **Vercel server env** (`SUPABASE_SERVICE_ROLE_KEY`) for server routes/webhooks (M1.1+). If the student pastes a service-role key into the front-end, stop them ([[supabase-best-practice]]). **App-runtime Supabase keys live in Vercel env, not AWS** ([[aws-secrets-best-practice]]).

---

### Step 9 — Add the `profiles.role` stub (trigger on signup)

The role tab (Step 1) captures `customer`/`shop` in auth metadata; now persist it into a real `profiles` table so M1.1 (shop pages) and the M2.1 prereq (admin promotion) have a `role` to gate on.

Have Claude Code apply this as a **Supabase migration** (never a raw console edit — [[supabase-best-practice]]). Use the Supabase MCP `apply_migration`:

```sql
-- profiles: one row per auth user, carrying the role.
-- The bank_account_* columns are the SHOP-LEVEL payout target: one shop can run
-- many barbers (M1.1), so the bank account belongs to the PERSON, not a barber profile. They
-- stay NULL until the shop fills "payout settings" in M1.1; only the shop + an
-- admin may read them. ("use test data first" — manual month-end transfer, M2.2.)
create table if not exists public.profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  email text,
  display_name text,           -- the user's personal/business name (set at sign-up); for a shop this is the payout name
  role text not null default 'customer' check (role in ('customer','shop','admin')),
  bank_account_name   text,
  bank_account_number text,
  created_at timestamptz not null default now()
);

alter table public.profiles enable row level security;

-- a user can read/update their own profile row (this is what scopes the bank
-- fields to shop-only; admin reads them via the is_admin() path added in M1.1).
create policy "profiles_select_own" on public.profiles
  for select using (auth.uid() = id);
create policy "profiles_update_own" on public.profiles
  for update using (auth.uid() = id);

-- create the profile row on signup, copying the role from sign-up metadata
create or replace function public.handle_new_user()
returns trigger language plpgsql security definer set search_path = public as $$
begin
  insert into public.profiles (id, email, role)
  values (
    new.id,
    new.email,
    coalesce(nullif(new.raw_user_meta_data->>'role',''), 'customer')
  );
  return new;
end;
$$;

drop trigger if exists on_auth_user_created on auth.users;
create trigger on_auth_user_created
  after insert on auth.users
  for each row execute function public.handle_new_user();

-- BACKFILL: the trigger only fires on FUTURE signups, so any users who already
-- signed up during M0 testing (Steps 2 / 8.4) have NO profiles row. Create one
-- for each, reading the role from their existing sign-up metadata.
-- NOTE the fallback here is 'shop', NOT 'customer' (the trigger's default):
-- a user old enough to predate the trigger is almost certainly YOUR own early
-- barber/shop test account created before the role tab was wired, so 'shop' is
-- the safer guess for a metadata-less row. Fix any wrong guess with a one-off
-- UPDATE migration. on conflict do nothing → safe to re-run; never clobbers a
-- row the trigger already made.
insert into public.profiles (id, email, role)
select
  u.id,
  u.email,
  coalesce(nullif(u.raw_user_meta_data->>'role',''), 'shop')
from auth.users u
on conflict (id) do nothing;
```

> **Note for Claude Code:** the role allowlist is `customer | shop | admin`, but the **sign-up form only ever writes `customer` or `shop`** — `admin` is never self-served. It's promoted manually in the M2.1 prerequisite. The `update_own` policy lets a user edit their own profile but **does not let them change `role` to `admin`** in practice because the admin pages are gated server-side; for hardening, M2.2 keeps `role` changes off the client path. (See [[supabase-best-practice]].)

**Verify:** `select id, email, role from public.profiles;` returns **one row per user in `auth.users`** — the backfill caught everyone who signed up during M0 testing, with the role from their sign-up metadata (no orphan auth users left without a profile). Then sign up a NEW customer and a NEW shop on the live site and confirm the trigger adds their rows with the right roles. Cross-check the counts match: `select (select count(*) from auth.users) as users, (select count(*) from public.profiles) as profiles;` — the two numbers should be equal.

---

### Step 10 — Bootstrap the skills into the project, then run the checklist

So the Cowork project has the course's skills as context for every later milestone:

> **ask:** "Use the Git clone tool. What files do you see under the project's `.claude/skills/` folder on GitHub?"
> **ask:** "Read all those skill files as context for further development. Update the existing ones when there are the same file names."

Then verify M0:

> **ask:** "Run the `m0-landing-page-checklist` skill."

---

## Things to watch out for (common mistakes)

1. **Don't waste a generation on a blank project** — full prompt + the two attached rule skills as your very first message.
2. **The landing must be the marketplace look, NOT 3 feature cards** — if Lovable produced a generic 3-card hero, re-prompt for the hero+search / logo strip / 4-icon row / Popular grid sections.
3. **Forgetting the role tab on sign-up** — the Customer/Shop toggle is what Step 9 turns into `profiles.role`. If it's missing, the whole role model breaks.
4. **Connecting your own Supabase too early** — do it in Step 8, after Vercel (v1 runs on Lovable Cloud).
5. **Forgetting to make the repo public (Step 3)** — before the Vercel import.
6. **SSR-vs-SPA trap (Step 6)** — green build but `/login` / `/barbers` 404. Convert to a plain SPA (or a clean Next.js app) before/at import.
7. **Vercel env vars not set (Step 8.3)** — after the Supabase swap, the same `*_SUPABASE_*` vars must be added in Vercel and redeployed, or the live site still points at Lovable Cloud. Most common M0 failure.
8. **Service-role key in the front-end** — never. M0 front-end uses only the publishable key; service-role goes into Vercel server env only.
9. **Email confirmation on but no SMTP** — disable email confirmation for the demo, or the test user can't finish sign-up.
10. **Custom tables in M0 beyond `profiles`** — only the `profiles` stub in Step 9. The barber/booking/payout tables are M1.1+.
11. **Don't rename the brand mid-course** — the brand is "Barberly"; keep it consistent across milestones.
12. **Re-pasting the GitHub token every session** — don't. It's cached in Secrets Manager (Step 5); later sessions discover and reuse it.

## Expected duration

45–75 minutes for a first-timer (most of it waiting on Lovable/Vercel builds, creating accounts, and the one-time Cowork project + connectors + GitHub-token caching in Steps 4–5).

## Next step

When `m0-landing-page-checklist` is green, tell the student:
「M0 完成了！你現在有一個能用 Customer / Shop 兩種身分註冊登入的線上理髮預約入口網站，而且 Cowork 專案、AWS / Vercel / Supabase 連接器、GitHub token、`profiles.role` 都備好了。準備好的話跟我說『啟動 M1.1』，我們來讓理髮店開店、建立預約排程。」
Then load `m1.1-barber-shop-and-schedule` (run `m1.1-barber-shop-and-schedule-prerequisites` first — it reuses the AWS access, GitHub token, and Supabase project you set up here).

## Reference

- Lovable: https://docs.lovable.dev/
- Supabase Auth: https://supabase.com/docs/guides/auth
- Supabase triggers: https://supabase.com/docs/guides/auth/managing-user-data
- Vercel deploys: https://vercel.com/docs/deployments
- AWS Secrets Manager: https://docs.aws.amazon.com/secretsmanager/
