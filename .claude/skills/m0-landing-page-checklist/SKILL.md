---
name: m0-landing-page-checklist
description: 抽成制理髮師預約平台 Milestone 0 verification — checks every artifact (GitHub repo, Vercel deploy, the hair-marketplace landing contents, Customer↔Shop role-tab auth, Supabase auth loop + the profiles.role stub) is real and correctly wired. Use when the student says "驗收 M0", "check M0", "M0 done?", or after the `m0-landing-page` skill completes Step 9.
---

# M0 — Landing + Sign-in Checklist

## What this skill does

Verifies the student actually completed M0 — not just *thinks* they did. People (and LLMs) skip steps. This skill tests every artifact and reports pass/fail per item, then emits a `READY for M1.1` verdict.

**Run this AFTER `m0-landing-page` Step 9, or any time the student claims M0 is done.**

## Execution mode: Cowork vs CLI (read this first)

| Section | CLI mode tool | Cowork mode equivalent |
|---|---|---|
| A — GitHub repo | `gh repo view` / `gh api` | GitHub MCP, or open repo URL in browser |
| B — Vercel deploy | `curl` | `mcp__vercel__*`, or open URL in browser |
| C — Landing page contents | `curl … \| grep` | Playwright MCP, or student inspects in browser |
| D — Auth + role | browser + Supabase MCP | Supabase MCP (preferred both modes) |

In Cowork mode every Bash block below is CLI-only — use the equivalent. Don't try to install `gh`/`curl` in Cowork.

## How to run

The student invokes this directly (e.g. types `驗收 M0`). You (Claude Code) **actively run** each check and report results — don't just describe them.

### Step 1: Collect URLs (one message)

Ask the student for:
1. GitHub repo URL (`https://github.com/<user>/<repo>`)
2. Vercel deploy URL (`https://<app>.vercel.app`)
3. Supabase project ref (`https://<ref>.supabase.co`)

### Step 2: Run the checklist

#### Section A — GitHub repo
- **A1** Repo exists, is **public**, has Lovable's files:
  ```bash
  gh repo view <owner>/<repo> --json name,visibility,defaultBranchRef
  ```
- **A2** Recent commit (Lovable sync + the deployable-build push from Step 6):
  ```bash
  gh api repos/<owner>/<repo>/commits --jq '.[0].commit.message' | head -1
  ```
  *Recovery if missing:* re-connect GitHub in Lovable (M0 Step 3) / re-push (Step 6).

#### Section B — Vercel deploy
- **B1** Live URL returns 200:
  ```bash
  curl -sS -o /dev/null -w "%{http_code}\n" https://<app>.vercel.app
  ```
- **B2** Auto-deploying from GitHub — confirm the Vercel project's Git connection points at the repo from A1. *Recovery:* re-import the repo (M0 Step 7).
- **B3** **Deep links work (SSR-vs-SPA trap):** `/login` does NOT 404:
  ```bash
  curl -sS -o /dev/null -w "%{http_code}\n" https://<app>.vercel.app/login
  curl -sS -o /dev/null -w "%{http_code}\n" https://<app>.vercel.app/barbers
  ```
  Expect `200` on `/login`; `/barbers` 200 or redirect to sign-in. A **404** on `/login` = SSR shipped without a SPA fallback. *Recovery:* M0 Step 6.

#### Section C — Landing page contents (the marketplace look, NOT 3 cards)
- **C1** Hero/search + Login present:
  ```bash
  curl -sS https://<app>.vercel.app | grep -oiE "login|sign in|sign up|style|barber|popular|search" | sort | uniq -c
  ```
  (A client-rendered SPA may return little — fall back to a browser/Playwright check.)
- **C2** Browser check (Cowork or if C1 empty): confirm **all four marketplace sections** are visible — (1) serif hero with split photos + a search bar, (2) trust logo strip, (3) a **4-icon feature row**, (4) a **"Popular" grid** — and that it is **NOT** the old generic 3-feature-card hero.

#### Section D — Auth + role
- **D1** The `/login` page shows a **Customer ↔ Shop tab/toggle** on sign-up.
- **D2** **The decisive test:** sign up a brand-new email **as a Shop** on the live site, then check the student's OWN Supabase → Authentication → Users (the new user appears there, not a Lovable-default backend).
- **D3** **The `profiles.role` stub works** — the new shop sign-up created a `profiles` row with `role = 'shop'`:
  ```sql
  -- via Supabase MCP execute_sql, or the SQL editor:
  select id, email, role from public.profiles order by created_at desc limit 3;
  ```
  Expect the just-created user with `role = 'shop'` (and a separately-created customer with `role='customer'`).
- **D3b** **No orphan auth users** — every existing `auth.users` row has a matching `profiles` row (the Step 9 backfill caught users who signed up during earlier M0 testing, not just future signups):
  ```sql
  select (select count(*) from auth.users) as users,
         (select count(*) from public.profiles) as profiles;
  ```
  The two counts must be **equal**. If `profiles < users`, the backfill didn't run — re-apply the Step 9 migration (the `insert … select from auth.users on conflict do nothing` block).
- **D4** Sign-in AND sign-out both work on the live site (close the loop).
  *Recovery:* redo M0 Step 8 (auth swap) and/or Step 9 (the `profiles` trigger).

## Reporting

Emit a table:

| Check | Status | Notes |
|---|---|---|
| A1 repo exists + public | ✅ / ❌ | |
| A2 recent commit(s) | ✅ / ❌ | |
| B1 Vercel 200 | ✅ / ❌ | |
| B2 auto-deploy wired | ✅ / ❌ | |
| B3 /login deep-link not 404 | ✅ / ❌ | SSR-vs-SPA trap |
| C1/C2 marketplace sections (NOT 3-card) | ✅ / ⚠️ / ❌ | hero+search / logo strip / 4-icon row / Popular grid |
| D1 Customer/Shop role tab | ✅ / ❌ | |
| D2 new user in student's Supabase | ✅ / ❌ | the key one |
| D3 profiles.role stub populated | ✅ / ❌ | role copied from sign-up tab |
| D3b no orphan auth users (counts match) | ✅ / ❌ | Step 9 backfill caught pre-existing signups |
| D4 sign-in + sign-out loop | ✅ / ❌ | |

**Verdict:**
- All ✅ → 「M0 驗收通過 ✅ READY for M1.1。跟我說『啟動 M1.1』，我們來讓理髮師開店、建立預約排程。」
- Any ❌ → list the failed items + the recovery step, and tell the student to fix then re-run `驗收 M0`.

---

## Appendix — Copy-paste setup prompts (M0 operational walkthrough)

The verbatim prompts the student pastes while running M0, in order. This is the operational companion to the build skill [[m0-landing-page]] — when a check above fails, the recovery often means re-running one of these. **Replace every `XXXXX` placeholder before pasting.** Role values are `customer` / `shop`; routes are `/` and `/login` and `/barbers` (there is no `/app`).

### Change GitHub repo to public

Go to https://github.com/ → Your repo (ex. `https://github.com/uopsdod/barberly-launchpad-78.git`) → **Settings** → **Change repository visibility: Public**.

### Sign in to Cowork on Desktop

**Set up Cowork**
- Create a new project (ex. `barber-booking-platform`).
- Turn on **Act without asking**.
- Turn on memory — **Settings → Capabilities**:
  - Search and reference chats
  - Generate memory from chat history
  - Connector discovery

`===== - ===== - =====`

**Set up Connector / Vercel**
- Sign in to vercel.com first.
- Install Vercel Connect → Cowork → **Customize → Vercel Connect**.

Prompt """
what projects are in my vercel account ?
"""

`===== - ===== - =====`

**Set up Connector / Supabase**
- Sign in to supabase.com first → create org.
- Install Supabase Connector → Cowork → **Customize → Supabase Connector**.

Prompt """
what org and projects do I have in my supabase organization?
"""

`===== - ===== - =====`

**Set up Connector / AWS**

*Set up the AWS credential*
- AWS email (ex. `uopsfof+barber@gmail.com`).
- Sign in https://aws.amazon.com/console/ → Root user → click **Forgot password** → go to email and set a new password → sign in again.
- Go to **IAM** → **Users** → Create `admin-for-project-barber-booking-platform-001` user → attach **AdministratorAccess** policy → Create → **access key: CLI**.
- Click the new IAM user → **Security credentials** tab → **Create access key** → click **CLI** → Create key.
- Replace the AWS access key and AWS secret key below.

> Claude Code / 桌面版

Prompt """
I have new AWS credentials I want to configure. Please write them to my AWS credentials file. Here are the values:
Access key ID: XXXXX
Secret access key: XXXXX
First, detect whether I'm on Mac/Linux or Windows to determine the correct credentials file path 
(~/.aws/credentials on Mac/Linux, %USERPROFILE%\.aws\credentials on Windows), 
then write the [default] profile with the new values — preserving any other existing profiles in the file. 
Once done, test the connection using aws sts get-caller-identity.
"""

*Add the AWS connector*
- Go back to the Cowork tab → **Customize → Connectors** → search for **"AWS API MCP"** → **Install**.

Prompt """
How many IAM users do I have now on my aws account? 
"""

`===== - ===== - =====`

### Migrate deployment host from Lovable to Vercel

**Convert the project to be Vercel-compatible**

*Convert the GitHub repo to public if not yet (see above).*

*Pass the GitHub token to AWS for Cowork*
- Get a GitHub token → https://github.com/settings/personal-access-tokens → select the single repo → click **Contents** permission → enable **Read and write**.

Prompt """ 
Here is my GitHub Personal Access Token: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX 
Use aws mcp to store it in secrets manager so we can re-use it in a new session 
"""

Prompt """ (source: https://github.com/uopsdod/claude-2-barber-booking-platform/blob/m0-landing-page/.claude/skills/m0-landing-page-checklist/SKILL.md)
GITHUB REPO: XXXXX
Make this project cleanly deployable on Vercel.
If it's a Vite/React app, convert it to a plain Vite SPA suitable for static hosting: remove any TanStack Start / SSR / server-side rendering and any Cloudflare/wrangler config; use React Router for client-side routing (/, /login, /barbers); the build output must be a static SPA (vite build → dist/) with a SPA fallback so deep links like /login and /barbers resolve client-side.
If it's a Next.js app, keep it as a standard Next.js project (Vercel's native target) and just confirm the build is clean and that /login and /barbers both resolve.
Keep all existing UI, auth, the Customer/Barber role tab, and styling unchanged.
"""

Prompt """
push the new Vercel-Compatible structure change to my github repo, overriding the main branch.
Recall the GitHub token from Secrets Manager (barber-project/github) to authenticate the push — don't ask me for it.
"""

**Migrate the deployment host**
- Go to https://vercel.com/ → **Import** the project → **Connect GitHub account** → import the GitHub repo → wait for the deployment.
- Get the Vercel deployed site URL (ex. `https://fly-low-alert.vercel.app/`).
- Check the landing-page style.
- Check the sign-up / sign-in features.

`===== - ===== - =====`

### Migrate database from Lovable to Supabase

- Go to https://supabase.com/ → create a project (ex. `https://supabase.com/dashboard/project/pmvtdxbelbgglpalxype/`).
- Wait for it to become **Healthy** → get the **Project URL** → get the **Publishable API Key**.
- Replace the URL and publishable-key values in the prompt (the env-var names are `NEXT_PUBLIC_SUPABASE_*` for a Next.js app, `VITE_SUPABASE_*` for a Vite app — the prompt covers both).

Prompt """ (source: https://github.com/uopsdod/claude-2-flight-price-notifier/blob/m0-landing-page/.claude/skills/m0-landing-and-signin/SKILL.md)
Switch this project's backend from Lovable Cloud to the user's own Supabase project. Do NOT keep any Lovable Cloud references.

Specifically:

Find every place the project currently uses Lovable Cloud's Supabase client (Lovable's auto-provisioned supabase client, typically in src/integrations/supabase/client.ts for Vite, or the Supabase client setup for Next.js). Point it at these credentials instead (use the NEXT_PUBLIC_* names for a Next.js app, the VITE_* names for a Vite app):

NEXT_PUBLIC_SUPABASE_URL (or VITE_SUPABASE_URL) = <SUPABASE_URL>

NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY (or VITE_SUPABASE_PUBLISHABLE_KEY) = <PUBLISHABLE_KEY>

Note: Supabase's newer "publishable key" (sb_publishable_*) replaces what used to be called the "anon key". They're the same role (browser-safe, RLS-gated). If the codebase already uses an *_ANON_KEY name, you can either:

Rename to the *_PUBLISHABLE_KEY name for consistency with current Supabase naming, OR
Keep the *_ANON_KEY variable name but put the new sb_publishable_* value in it (works fine; it's just an env var name). Pick one and apply consistently across .env, the client init code, and any docs.
Update the .env file (or .env.local) to use the values above. Remove any Lovable Cloud env vars (e.g. anything prefixed with LOVABLE_CLOUD_* or that points to a Lovable-owned Supabase ref).

Make sure the Supabase client is initialized exactly once and reads from env — no hardcoded URLs.

Keep the Sign Up / Sign In / Sign Out flow and the Customer/Barber role tab exactly as they are. Only the backend target changes. The sign-up call must still pass the chosen role in user metadata (options.data.role).

After this change, sign-up should create users in the user's own Supabase auth.users table — verify by signing up a NEW test user in the Lovable preview, then checking the Supabase dashboard → Authentication → Users — the new email should appear there.

As last, reminder users to manually add the same Supabase URL + publishable-key environment variables to Vercel (Settings → Environment Variables), then redeploy.
"""

**Check the sign-up feature on the Vercel URL to verify the database**
- Get the same Vercel deployed site URL as before.
- Check the landing-page style.
- Go to the sign-up page → sign up a new email → click the confirmation email (if email confirmation is on).
- Go to the sign-in page → sign in.
- Go to the Supabase project → **Authentication** → check whether a new user record shows up.

`===== - ===== - =====`

### Create the `profiles` table (role stub + trigger + backfill)

This is **M0 Step 9** — the one application table M0 creates. Run it **after** the Supabase swap above and **after you've signed up your first account** (so the backfill has someone to catch). The sign-up role tab only writes `customer`/`shop` into auth metadata; this migration persists that into a real `profiles.role` for M1.1 (shop pages) and the M2.1 prereq (admin promotion) to gate on. Apply it as a **migration via the Supabase MCP `apply_migration`** — never a raw SQL-editor edit ([[supabase-best-practice]]).

Prompt """
Apply a Supabase migration (via the Supabase MCP apply_migration — NOT a raw SQL-editor edit) that creates the profiles table for this barber-booking app. The migration must contain exactly this, in order:

1. A profiles table: id uuid PK referencing auth.users(id) on delete cascade; email text; display_name text; role text not null default 'customer' check (role in ('customer','shop','admin')); bank_account_name text; bank_account_number text; created_at timestamptz not null default now().

2. Enable row level security on profiles, and add two policies:
   - profiles_select_own: for select using (auth.uid() = id)
   - profiles_update_own: for update using (auth.uid() = id)

3. A SECURITY DEFINER trigger function public.handle_new_user() (set search_path = public) that, on each new auth.users insert, inserts into profiles (id, email, role) the new user's id, email, and coalesce(nullif(new.raw_user_meta_data->>'role',''), 'customer'). Then drop any existing on_auth_user_created trigger and create it: after insert on auth.users, for each row execute that function.

4. A ONE-TIME backfill at the end of the same migration, for any users who signed up before the trigger existed:
   insert into public.profiles (id, email, role)
   select u.id, u.email, coalesce(nullif(u.raw_user_meta_data->>'role',''), 'shop')
   from auth.users u
   on conflict (id) do nothing;
   IMPORTANT: the backfill fallback is 'shop', NOT 'customer' — a pre-trigger user is almost certainly my own early barber/shop test account, so 'shop' is the safer guess for a metadata-less row.

After applying, verify with: select id, email, role from public.profiles; and confirm count(auth.users) == count(public.profiles) (no orphan auth users). Do not create any other tables — only profiles.
"""

> **Note:** the trigger handles every signup **after** this migration; the backfill handles the account(s) you created **before** it (its fallback is `'shop'` because your first test account is most likely a barber/shop). If that first account was actually a customer, fix it with a one-line `UPDATE` migration.

`===== - ===== - =====`

### Set up Cowork / skills

Point this at **your own barber-platform repo** (the one Lovable created and you made public), so the course's skills load as context for every later milestone. Replace `<owner>/<repo>` with yours.

Prompt """
Use Git clone tool. What files do you see on https://github.com/<owner>/<repo>/blob/main/.claude/skills/
"""

Prompt """
read all those skill files as context for further development in my project here.
Update the existing ones when there are the same file names.
"""

Prompt """
run m0-landing-page-checklist skill
"""
