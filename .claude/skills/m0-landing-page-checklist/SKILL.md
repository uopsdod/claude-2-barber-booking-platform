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
  Expect the just-created user with `role = 'shop'` (and a separately-created buyer with `role='buyer'`).
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
| D4 sign-in + sign-out loop | ✅ / ❌ | |

**Verdict:**
- All ✅ → 「M0 驗收通過 ✅ READY for M1.1。跟我說『啟動 M1.1』，我們來讓理髮師開店、建立預約排程。」
- Any ❌ → list the failed items + the recovery step, and tell the student to fix then re-run `驗收 M0`.
