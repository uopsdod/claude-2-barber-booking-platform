---
name: m1.1-seller-setup-prerequisite
description: One-time carryover check before Milestone 1.1 of the barber-booking course — an INTERACTIVE, agent-driven walkthrough. M1.1 is the FIRST milestone where Supabase holds real application data, so this skill introduces the Supabase DATA-TABLE + RLS discipline (the flight course kept Supabase auth-only) and confirms the M0 foundation is still good: M0 green (auth + `profiles.role` works), the Supabase MCP reachable, and the GitHub PAT cached in Secrets Manager. Ends with one verified read against the new/empty schema. Use when the student starts M1.1 for the first time, or when `m1.1-seller-setup` / `-checklist` detects M0 auth, the Supabase MCP, or the GitHub token is missing.
---

# M1.1 Prerequisites — interactive carryover check (the agent drives)

**You are the Cowork agent running this skill. Drive the student through it one part at a time** — don't dump the whole thing. For each part: say what's about to happen, **check first** (Secrets Manager / Supabase MCP), ask the student for a value **only if something is actually missing**, run the work yourself, confirm it worked, and move on.

> **M1.1 introduces nothing new to set up — it introduces a new *discipline*.** M0 already wired AWS access, cached the GitHub PAT, swapped auth to the student's own Supabase, and created the `profiles.role` stub. So this prereq is a **carryover check**, not an account-setup. The genuinely new thing M1.1 brings is that **Supabase now holds real application tables** (`barbers` / `services` / `bookable_slots`), each protected by **Row Level Security** — the flight course kept Supabase auth-only, so this is the moment to internalize the data-table + RLS rules in [[supabase-best-practice]].

> ⚠️ **Don't assume the secret *name*.** M0 was run by an AI in a separate session, and the name it chose for the GitHub PAT is **not guaranteed** (`barber-project/github`, `github-pat`, `barber/github`, …). Never hard-code `--secret-id barber-project/github` and treat a miss as "absent." **Discover by listing + scanning** (below); only treat it as missing after the scan finds nothing.

## Architecture (what this check unlocks)

![Barber platform architecture (M1.1) — the carryover pieces this prereq confirms before the barber side is built: Cowork (claude code) pushes UI to the GitHub repo (token recalled from AWS Secrets Manager) → Vercel-hosted Product Site; and the Supabase project that held auth + profiles.role in M0 is about to gain three RLS-protected data tables (barbers, services, bookable_slots). This skill verifies M0's auth + profiles.role loop still works, the Supabase MCP can reach the project, and the GitHub PAT is cached, then does one read against the (still empty) new schema.](assets/architecture-m1.png)

The three things you confirm here are what M1.1 builds on:
- **M0 auth + `profiles.role`** → the role gate (`shop`) every M1.1 surface checks.
- **Supabase MCP reachable** → how Step 2 applies the `platform_settings`/`barbers`/`services`/`bookable_slots` migration + runs `get_advisors`.
- **GitHub PAT cached** → the push loop that ships the onboarding form + `/shop/bookings`.

## When to load this skill

- The student says "啟動 M1.1 前置" / "M1.1 prerequisites" / "check M1.1 setup".
- The `[[m1.1-seller-setup]]` build skill (or its checklist) detects M0 auth, the Supabase MCP connector, or the GitHub token is missing.

**Opening line to the student (say something like):**
> "M1.1 is the first milestone where Supabase holds *real* data — barbers, services, schedule slots — each locked down with Row Level Security. Almost everything it needs was already set up in M0, so I'll just confirm three carryovers are still good (your auth + `profiles.role`, the Supabase connection, and your cached GitHub token), do one quick read against the new tables' home, then we build. Let me check what's already there."

---

## Step 1 — M0 is green (auth + `profiles.role` works)

M1.1 gates every barber surface on `profiles.role`, so confirm the M0 seam still works **before** building on it.

**1a — `profiles` exists and carries a role.** Via the Supabase MCP (`mcp__claude_ai_Supabase__execute_sql`):
```sql
select id, email, role from public.profiles order by created_at desc limit 5;
```
- Returns rows with a `role` of `customer` / `shop` → ✅ the M0 stub works.
- **No `profiles` table / no `role` column** → M0 Step 9 didn't land. Stop and send the student back to `[[m0-landing-page]]` Step 9.

**1b — the role allowlist is right.** Confirm the check constraint allows `customer | shop | admin` (M1.1 writes `shop`; `admin` is promoted later):
```sql
select pg_get_constraintdef(c.oid)
from pg_constraint c
join pg_class t on t.oid = c.conrelid
where t.relname = 'profiles' and c.contype = 'c';
```
- See `role in ('customer','shop','admin')` → ✅. If `admin` is missing from the allowlist, note it; the M2.1 prereq needs it (it's harmless for M1.1).

> If the student has no `shop` row yet, that's fine — M1.1 Step 1 is where a user becomes a shop. You only need the **table + column + constraint** to be correct here.

---

## Step 2 — Supabase MCP is reachable (the migration + advisor path)

M1.1 applies schema and RLS through the Supabase MCP. Confirm the connector can actually reach the student's project — one read is enough:
```text
mcp__claude_ai_Supabase__list_tables   →  should list at least public.profiles
```
- Returns the table list (you should see `profiles`) → ✅ the Supabase Connector is wired to the right project (the one M0 swapped auth onto).
- **Auth error / wrong project / empty** → the Supabase Connector isn't installed or points at a different org/project. Have the student re-install it (Cowork → Connectors → **Supabase Connector**) and pick the **barber-platform** project from M0, then re-run.

> **Note for Claude Code:** also confirm `mcp__claude_ai_Supabase__get_advisors` is callable (you'll run it in M1.1 Step 4 after the RLS migration). If `list_tables` works, the advisor will too — they're the same connector. Don't apply any migration in this prereq; M1.1 owns the schema.

---

## Step 3 — GitHub PAT is cached (the push loop) — discover, don't re-ask

The onboarding form + `/shop/bookings` ship via the same push loop M0 set up. The PAT was cached in M0 **under whatever name that run chose** — so **scan, don't assume the name**.

Run the canonical list once and scan it:
```bash
aws secretsmanager list-secrets --region us-east-1 --query "SecretList[].Name"
```
1. Pick the name that looks like the GitHub token — anything containing **`github`** (e.g. `barber-project/github`, `github-pat`, `barber/github`).
2. Read it and pull out the token — **accept either shape**:
   ```bash
   aws secretsmanager get-secret-value --secret-id "<the name you found>" --region us-east-1 --query SecretString --output text
   ```
   - JSON (`{"pat":"github_pat_…"}`) → use the `pat` field.
   - Bare string (`github_pat_…` with no JSON wrapper) → use the whole value. Don't `json.load`-and-fail.
   - Sanity-check it starts with `github_pat_` (fine-grained) or `ghp_` (classic).
   - Found → "✅ GitHub token still cached from M0 (`<name>`) — I'll recall it for pushes, never re-ask." Remember the exact name for the build skill.
   - **No `github`-ish secret anywhere** (rare — M0 partial) → ask:
     > "I don't see a GitHub token cached. Paste your **GitHub fine-grained PAT** — github.com/settings/personal-access-tokens → *Generate new token (fine-grained)* → *Only select repositories* → this repo → *Repository permissions → Contents → Read and write* → copy the `github_pat_…`."

     Wait (⚠️ write-credential — don't echo it back), then cache under the course-standard name:
     ```bash
     aws secretsmanager create-secret --name barber-project/github \
       --secret-string '{"pat":"<their token>"}' --region us-east-1
     ```

> **Note for Claude Code:** also confirm AWS itself is still wired (`aws sts get-caller-identity --query Account --output text --region us-east-1` returns an account id). If it errors, the `[default]` profile expired — send the student back to `[[m0-landing-page]]` Step 4 to re-write it. App-runtime Supabase keys live in **Vercel env**, not here ([[aws-secrets-best-practice]]).

---

## Step 4 — One verified read against the new/empty schema

The new tables don't exist yet (M1.1 creates them), so the "verified read" here is the **negative** read that proves a clean slate — exactly what the build skill expects to start from. Via the Supabase MCP:
```sql
select to_regclass('public.barbers')          as barbers,
       to_regclass('public.services')        as services,
       to_regclass('public.bookable_slots')  as bookable_slots;
```
- All three return **NULL** → ✅ clean slate, M1.1 will create them fresh.
- One already exists (a partial earlier run) → read its columns (`\d public.barbers` equivalent) and tell the student; the M1.1 migration uses `create table if not exists`, so it's safe to re-run, but flag it so you don't double-build the UI.

---

## Verify (all must pass)

- ✅ **M0 auth + `profiles.role`** — `profiles` exists, `role` column present, allowlist `customer|shop|admin` (Step 1).
- ✅ **Supabase MCP reachable** — `list_tables` returns the project's tables (incl. `profiles`); `get_advisors` is callable (Step 2).
- ✅ **GitHub PAT cached** — discovered in Secrets Manager (whatever name), shape tolerated; AWS `[default]` still wired (Step 3).
- ✅ **Clean schema slate** — `barbers` / `services` / `bookable_slots` are NULL (don't exist yet), ready for M1.1's migration (Step 4).

## Next step

When all four are ✅, tell the student:
「前置檢查通過 ✅ — M0 的登入與 `profiles.role` 正常、Supabase MCP 連得上、GitHub token 還在 Secrets Manager，而且 barbers/services/bookable_slots 是乾淨的空白狀態。接下來我會用 migration 建這些資料表（含 platform_settings 與 RLS），再做理髮店上架表單和 `/shop/bookings`。跟我說『啟動 M1.1』就開始。」
Then return to the build skill `[[m1.1-seller-setup]]` (Step 1).
