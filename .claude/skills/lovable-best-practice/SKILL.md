---
name: lovable-best-practice
description: Hard rules and workflow tips for working with Lovable in the 抽成制理髮師預約平台 course. Use whenever a student is generating, re-rolling, or editing the M0 Lovable landing page + role-tab auth — and any later Lovable re-roll of the front-end. Covers not connecting your own Supabase too early (v1 = Lovable Cloud), shipping a clean modern build (avoid the SSR 404 trap), keeping the hair-themed Novara design tokens consistent (serif hero + beige/cream + the 4 marketplace sections, NOT a 3-card hero), the Customer↔Shop role-tab auth UX, capturing role in auth metadata only in M0, PascalCase components, and no secrets in commits. Apply proactively — stop the student before they break one.
---

# Lovable Best Practice (Barber Booking — the M0 hair-marketplace build)

This skill is loaded any time the student is interacting with Lovable — which in this course is **M0** (the hair-themed landing page + role-tab sign-in) and any later re-roll of the front-end. It lives at two levels:

1. **Hard rules** — non-negotiable. Each has a real incident behind it; skipping one costs the student hours later (or ships a broken deploy / a leaked secret).
2. **General workflow tips** — Vercel framework-preset gotchas, re-roll discipline, git commit habits.

When you (Claude Code) guide a student through Lovable steps, **apply these rules proactively** — don't wait for the student to ask. If you see them about to break one, stop them and explain why.

> **Architecture note that shapes these rules:** in M0 the app uses **Lovable Cloud auth** for v1, then swaps to the **student's own Supabase** (M0 Step 8). From M1.1 on, Supabase holds **real app data** (barbers/bookings/payouts) with RLS — but **none of that exists in M0**. So in M0 you capture the chosen role in **auth user metadata only**; you do **not** create `profiles`/`barbers`/`bookings` tables yet. See [[supabase-best-practice]].

---

## Execution mode: Cowork vs CLI

Lovable itself works the same in both (it's a hosted web app), but the **verification + git steps around it** differ.

| Operation | CLI mode | Cowork mode |
|---|---|---|
| Confirm a Lovable iteration synced to GitHub | `gh api repos/<owner>/<repo>/commits --jq '.[0].commit.message'` | GitHub MCP, Lovable's Git panel, or open `https://github.com/<owner>/<repo>/commits` |
| Secret/PII grep over changed files (Rule 6) | `grep -rE 'sk_(test\|live)_\|service_role\|github_pat_' .` locally | No local checkout — review the Lovable diff in the UI before saving, or scan the pushed commit in GitHub web |
| Edit Lovable's generated files directly | Claude Code Edit/Write on a local clone | Edit through Lovable's own editor; there's no local working copy |

The **hard rules** apply identically in both modes — only the verification mechanics change.

---

## Hard rules (apply to every Lovable session in this course)

### Rule 1 — Do NOT connect your own Supabase (or GitHub) on the first Lovable prompt. v1 runs on Lovable Cloud.

> **The rule:** In M0, generate v1 on **Lovable Cloud** (its built-in auth). Connect **GitHub in Step 3** and the **student's own Supabase in Step 8** — never on the initial generation.

**Why:** Connecting your own Supabase too early makes Lovable scaffold auth (and sometimes tables) against a backend before the page even exists, and the cleanup — re-pointing to the student's project, removing the premature tables — gets tangled with the initial generation. Connecting late means the swap is one clean step. v1 on **Lovable Cloud** is the fastest path to a working sign-up to demo, and the swap to the student's own Supabase (M0 Step 8) is deliberate and isolated.

**How to apply:** If the student (or Lovable) tries to "Connect Supabase" during the first generation, stop them: 「先不要連自己的 Supabase，v1 用 Lovable Cloud，我們在 Step 8 一次接乾淨。」

---

### Rule 2 — Ship a clean, modern build (Vite SPA or standard Next.js) — avoid the SSR 404 trap.

> **The rule:** The deployed app must be either a **plain Vite SPA** (with a SPA fallback so deep links resolve) or a **standard Next.js app** (Vercel's native target). Do **not** ship an exotic SSR scaffold (TanStack Start / Cloudflare-wrangler-targeted) that "builds successfully" on Vercel but **404s every route**.

**Why:** Lovable sometimes scaffolds an SSR/edge-targeted app that compiles fine on Vercel but, because the routing/runtime doesn't match Vercel's target, **every route — including `/login` and `/barbers` — returns 404** after a green build. It's the single most confusing M0 deploy failure: the build log is clean, but the live site is dead. A plain Vite SPA (with a `dist/` SPA fallback) or a standard Next.js app deploys to Vercel with zero surprises.

**How to apply:**
- Attaching this skill to the **first** Lovable prompt (M0 Step 1) usually prevents the SSR scaffold — do it.
- At M0 Step 6, have Claude Code convert to a clean **Vite SPA** (remove TanStack Start / SSR / `wrangler.toml`; React Router; static `dist/` + SPA fallback) or keep a **standard Next.js** app, then confirm `vite build` / `next build` is clean.
- **Verify after deploy:** `curl` `/`, `/login`, `/barbers` → `200` (or a redirect to `/login`). A **404 on `/login`** means the SSR trap is still there.

---

### Rule 3 — Keep the hair-themed Novara design tokens consistent: serif hero + beige/cream + the 4 marketplace sections — NOT a 3-card hero.

> **The rule:** The landing page is a **Novara-style hair marketplace**: a **serif display headline** over a warm **beige/cream** palette, **split hairstyle/barber photos** flanking a centered **search bar**, then — in order — a **trust logo strip**, a **4-icon feature row** (Verified Barbers / Instant Booking / Secure Payment / Top-Rated Styles), and a **"Popular" grid** of styles/top barbers. It is **NOT** a generic "3 feature cards" hero. Keep these tokens consistent across every re-roll.

**Why:** The marketplace look is the product's whole first impression and a deliberate departure from the default SaaS "3 cards under a headline" hero Lovable reaches for. If a re-roll drifts back to the 3-card layout or swaps the serif/beige palette for generic Tailwind blue, the page stops looking like the intended hair marketplace and every later screenshot/demo is off-brand. The four sections (hero+search, logo strip, 4-icon row, popular grid) are the spec — re-rolling for layout is the most common Lovable time-sink, so pin them up front.

**How to apply:**
- Use the **verbatim M0 prompt** (M0 Step 1) — it names every section, the serif/beige tokens, and the 4 feature labels, so v1 obeys on the first generation.
- If a generation comes back as a 3-card hero or with a generic palette, **re-prompt for the specific sections** ("serif hero with split photos + centered search; logo strip; exactly 4 icon cells; a Popular grid") rather than nudging colors one at a time.
- Pick the **brand wordmark once** (e.g. "Barberly") and keep it across milestones — don't rename mid-course.

---

### Rule 4 — The Customer↔Shop role tab belongs on the SIGN-UP form, captured into auth metadata.

> **The rule:** The `/login` sign-up form must have a **role selector as a tab / segmented toggle** at the top: **"I'm a Customer"** / **"I'm a Shop"** (default Customer). The choice is passed into the sign-up call's **user metadata** (`options.data.role`). Sign-in (existing users) has no role tab.

**Why:** This toggle is the **seam** the whole role model hangs on — M0 Step 9 turns the captured metadata into `profiles.role` (via the signup trigger), M1.1 gates the shop pages on `role = 'shop'`, and the M2.1 prereq promotes an `admin`. If the tab is missing, or the role isn't passed into the sign-up call, there's no role to persist and the customer/shop split breaks downstream. It must be on **sign-up** (where you choose who you are), not sign-in.

**How to apply:**
- The sign-up call carries `options: { data: { role } }` with `role` ∈ `customer | shop` (the Customer tab maps to `customer`, the Shop tab to `shop`; the default is `customer`).
- **Never** offer "Sign up as Admin" — `admin` is promoted manually later ([[supabase-best-practice]]), never self-served.
- If a re-roll drops the tab or stops passing the role, restore it before moving on — Step 9 depends on it.

---

### Rule 5 — Capture role in auth metadata ONLY in M0 — no custom tables yet.

> **The rule:** In M0, the chosen role lives **only** in Supabase auth user metadata (`raw_user_meta_data.role`). Do **not** let Lovable create `profiles` / `barbers` / `services` / `bookings` tables, RPCs, or client-side `.from('…')` data queries in M0. The `profiles` table is added by **you** as a migration in M0 Step 9; the rest are M1.1+.

**Why:** M0 is landing page + auth only. If Lovable scaffolds app tables in M0, they collide with the real schema (with RLS) that M1.1+ introduces via migrations — you'd have a Lovable-made `profiles` fighting the one your Step 9 trigger creates, plus RLS gaps Lovable doesn't set. Keeping M0 to auth-metadata-only means the data layer is built **deliberately, as versioned migrations** ([[supabase-best-practice]] Rule 1), not improvised by the UI generator.

**How to apply:**
- The M0 prompt explicitly says: *do NOT create `barbers`/`bookings`/`profiles` tables; only use `auth.users`; capture the role in auth user metadata.*
- Reviewing a Lovable diff, watch for `supabase.from(...)`, new `.sql` migrations, or "create table" prompts → **block them** in M0: 「M0 只用 auth metadata 存 role，資料表等 M1.1 用 migration 建。」
- The one exception is **your** Step 9 `profiles` migration — that's intentional and trigger-driven, not a Lovable table.

---

### Rule 6 — No secrets in commits (and PascalCase components).

> **The rule (6a — secrets):** Never let a Stripe key (`sk_test_`/`sk_live_`/`whsec_`), a Supabase **service-role** key, a `NEXT_PUBLIC_`-misprefixed secret, or a GitHub PAT reach a Lovable-generated file or a commit. Real-looking demo emails/names → swap to obvious fakes (`user@example.com`) before committing.
>
> **The rule (6b — PascalCase):** React component files are `PascalCase.tsx` (`BookingDialog.tsx`, `BarberCard.tsx`) — not `booking-dialog.tsx`, not `bookingDialog.tsx`.

**Why (6a):** Once a secret hits the GitHub repo it's in git history forever — even after deletion, every clone has it, and GitHub's secret scanner indexes it instantly. The service-role key bypasses all RLS; a leaked Stripe live key can move real money. A public course repo makes any leak immediate. (App-runtime keys belong in **Vercel env**, the GitHub PAT in **AWS Secrets Manager** — never in code; see [[supabase-best-practice]], [[aws-secrets-best-practice]].)
**Why (6b):** Lovable's default naming is inconsistent (kebab/camel/Pascal, sometimes mixed in one project). Once the project grows past M1, hunting for `<BookingDialog />` among three casings is a real tax. Pick PascalCase and tell Lovable to enforce it.

**How to apply:**
- Before any Lovable-triggered commit, scan changed files:
  - **CLI:** `grep -rE 'sk_(test|live)_|whsec_|service_role|github_pat_|@(gmail|yahoo|hotmail|outlook)\.com' .`
  - **Cowork:** review the diff in Lovable's UI before saving, or scan the pushed commit in GitHub web.
- Component files (export a React component): `PascalCase.tsx`. Hooks/utils/types: keep Lovable's default (`useAuthState.ts`). If Lovable makes a kebab-case `.tsx`, tell it: "rename to PascalCase to match convention."
- The student's **own** test sign-up email living in **Supabase auth.users** is fine — that's runtime auth data, not committed code.

---

## General Lovable workflow tips

### Tip 1 — Vercel framework-preset auto-detection sometimes picks wrong

Lovable's output is Vite + React (or sometimes Next.js). Vercel usually auto-detects, but occasionally guesses wrong and the deploy fails with a cryptic build error. **Fix:** Vercel project → Settings → General → **Framework Preset** → set to **Vite** (or **Next.js**) explicitly → redeploy.

### Tip 2 — Free-tier credit budget — re-roll discipline

Lovable's free tier limits generations per day (verify your current quota). Each "fix and regenerate" eats one; you can burn the daily budget in 15 minutes.
- First prompt = the **full** M0 prompt verbatim (Step 1) with the two rule skills attached. Don't try shorter versions first.
- Small issues (color, copy) → Lovable's edit-in-place, not a full regenerate.
- If you must regenerate, batch **all** desired changes into one follow-up prompt — not three small re-rolls (especially for the Rule 3 sections).

### Tip 3 — Always commit to GitHub before iterating in Lovable

Lovable two-way-syncs with GitHub once connected (M0 Step 3). If a later generation overwrites something you liked, you want a commit to roll back to.
- **CLI:** `gh api repos/<owner>/<repo>/commits --jq '.[0].commit.message'`
- **Cowork:** open `https://github.com/<owner>/<repo>/commits`, or use the GitHub MCP / Lovable's Git panel.

---

## What this skill is NOT

- Not a generic Lovable tutorial — it's opinionated guidance for **this** course (the barber booking platform).
- Not a replacement for [[m0-landing-page]] — that skill has the step-by-step + the verbatim prompt; this provides the *rules* it assumes.
- Not Lovable documentation — see https://docs.lovable.dev.

## TODO (fill in after running M0 a few times)

- [ ] Confirm the current Lovable free-tier generation limit.
- [ ] Capture any new M0 pitfalls from the first cohort (e.g. whether Lovable emits Vite or Next.js by default this month).
- [ ] Lock the exact prompt wording that reliably yields the 4-section marketplace hero (Rule 3) + the working role tab (Rule 4) on the first generation.
- [ ] Confirm the Lovable "Connect Supabase" UI flow for the M0 Step 8 swap to the student's own project.

## Cross-references

- [[m0-landing-page]] — the M0 build (the verbatim prompt + the 10 steps these rules support).
- [[supabase-best-practice]] — auth-metadata-only in M0 (Rule 5), where the service-role key goes (Rule 6a), and the migration-driven `profiles`/data tables.
- [[aws-secrets-best-practice]] — where the GitHub PAT lives (not in commits) and the app-runtime-keys-in-Vercel split.
