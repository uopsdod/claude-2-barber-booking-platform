---
name: m2.1-buyer-to-admin-payments-prerequisites
description: One-time setup before Milestone 2.1 of the barber booking platform — an INTERACTIVE, agent-driven walkthrough with TWO parts. (1) Connect a Stripe SANDBOX account via the Stripe MCP / CLI and verify `livemode:false` (the consent page defaults to LIVE — you must switch to sandbox), then put its `STRIPE_SECRET_KEY` (`sk_test_…`) in Vercel env — all the pre-code Stripe setup lives here, not in the build (the build's Step 3 only confirms it; `STRIPE_WEBHOOK_SECRET` is created later in build Step 8). (2) Promote ONE existing account to `role='admin'` — there is no public admin sign-up; the person signs up normally via /login, you find them with SELECT id,email,role FROM profiles, and promote them with a ONE-OFF Supabase migration (UPDATE public.profiles SET role='admin') applied via `mcp__claude_ai_Supabase__apply_migration`. Admin lives in THIS prereq because admin only matters once payment exists (the later payout/settlement work consumes it). Two things need a MANUAL confirmation from the student: (a) WHICH email to promote to admin, and (b) that `STRIPE_SECRET_KEY` is saved in Vercel. Use when the student starts M2.1, says "啟動 M2.1 的前置作業", "set up Stripe sandbox", "connect Stripe", "put the Stripe secret key in Vercel", "promote my admin user", "create an admin account", or when `m2.1-buyer-to-admin-payments` / `-checklist` detects Stripe isn't connected or no admin exists.
---

# M2.1 Prerequisites — Stripe sandbox + promote your admin user (the agent drives)

**You are the Cowork agent running this skill. Drive the student through it one part at a time** — don't dump the whole thing. For each part: say what's about to happen, do the connector/MCP work yourself, confirm it worked, and move on.

This prereq does **two** things, and only two:

1. **Connect a Stripe SANDBOX account** (verify you're really in sandbox, `livemode:false`) and **put its `STRIPE_SECRET_KEY` in Vercel env** — all the pre-code Stripe setup lives here.
2. **Promote one existing account to `role='admin'`** via a one-off migration.

> **Two manual confirmations the student MUST give you.** Everything else you do yourself, but two things depend on a human decision/action you can't perform:
> - **Which email to promote to admin** — you can't guess it; ask, then promote exactly that one account (Part B).
> - **That `STRIPE_SECRET_KEY` is saved in Vercel** — Vercel MCP doesn't manage env vars, so you can't set or read it; the student pastes it in the dashboard and confirms (A4).

> **Why admin lives in THIS prereq.** The `admin` role only becomes *meaningful once money exists* — an admin's entire job is the month-end payout reconciliation, which needs paid bookings to reconcile. So the course promotes the admin here, in the payment milestone, right before the settlement work consumes it. There is **no public "sign up as admin" UI** — promotion happens exactly once, through a migration *you* run, which is also why a user can never self-escalate.

**Opening line to the student (say something like):**
> "M2.1 needs two bits of setup before we wire payments: first I'll connect your **Stripe sandbox** account (test mode — no real money), then we'll **promote one account to admin**. I'll need two things from you: which email should become the admin, and a quick confirmation that your Stripe secret key is saved in Vercel. Two parts, one at a time. Let's start with Stripe."

---

## Part A — Stripe sandbox account + MCP/CLI auth

### A1 — Make sure the student has a Stripe account

> "Do you have a Stripe account? If not, sign up free at https://dashboard.stripe.com/register — you do **not** need to activate the account or submit business details for sandbox/test mode. Going live with real cards is a separate, much-later step."

No activation needed for sandbox. Wait for a yes before connecting.

### A2 — Connect Stripe — and SWITCH TO SANDBOX (the foot-gun)

**Cowork mode (Stripe MCP):** install the Stripe connector → authenticate (`mcp__claude_ai_Stripe__authenticate` → `mcp__claude_ai_Stripe__complete_authentication`).

> ⚠️ **The #1 foot-gun: the Stripe consent page defaults to LIVE.** When the OAuth/consent page opens, there is an **account / mode selector** — it lands on the **live** account by default. **Switch it to a sandbox** (test) account *before* you approve. If you approve on live, every key and call below is a live key and you risk touching real money. If you're not sure which you approved, treat it as wrong and re-auth into sandbox.

**CLI mode (Stripe CLI):** the student runs `stripe login`, which opens the browser pairing/consent page — **same switch-to-sandbox warning applies** — then pastes the confirmation back. The CLI's `stripe listen` is for **local** webhook testing (it prints a rotating `whsec_…` for `localhost`); the **deployed** Vercel webhook uses the **dashboard endpoint's** stable secret instead (created in M2.1 build Step 8). Don't confuse the two ([[stripe-best-practice]] Rule 4).

| | Cowork (Stripe Connector) | CLI (Stripe CLI) |
|---|---|---|
| Connect | install connector → `authenticate` → `complete_authentication` | `stripe login` → approve in browser → paste back |
| Switch to sandbox | **on the consent page, pick the sandbox account before approving** | **same — pick sandbox on the pairing page** |
| Local webhook test | (test against deployed Vercel URL) | `stripe listen --forward-to localhost:3000/api/stripe/webhook` (rotating secret) |

### A3 — Verify you're actually in SANDBOX (`livemode:false`)

Don't trust the consent screen — **prove** it. Make any read call through the Stripe MCP / CLI and confirm the returned objects carry **`livemode: false`** (and that the API key in play is an `sk_test_…`, not `sk_live_…`):

- Cowork: ask the Stripe MCP to list a couple of recent objects (e.g. balance / a test charge) and check the `livemode` field on the response.
- CLI: `stripe balance retrieve` → the account/object shows test-mode; `stripe config --list` shows the test key in use.

> **Note for Claude Code:** if anything reads `livemode: true` or you see an `sk_live_…`, **stop** — you're on the live account. Re-authenticate and pick the sandbox account. Going live (new `pk_live_`/`sk_live_` keys + a new webhook endpoint on the custom domain) is a separate, later step handled well after this milestone.

### A4 — Put the sandbox `STRIPE_SECRET_KEY` in Vercel env  ← manual confirmation #2

Now that sandbox is connected, stash its secret key in Vercel so M2.1's build has it ready — this is a pure "copy the key into env" action with no dependency on any M2.1 code, so it belongs here with the rest of the Stripe setup (not in the build). **This is the second of the two things the student does by hand:** Vercel MCP does not manage env vars in 2026, so you can't set or read it — the student pastes it in the dashboard and confirms back to you.

> 到 Stripe dashboard，**確認右上角在 sandbox / test mode**（A2/A3 已切過），Developers → API keys → 複製 **Secret key**（`sk_test_…`）。
> 然後到 **Vercel → Settings → Environment Variables**，新增 `STRIPE_SECRET_KEY = sk_test_…`（Production scope），存檔。之後任何新環境變數生效都要 **redeploy** 一次。

This is an **app-runtime key → Vercel env, NOT AWS Secrets Manager** ([[aws-secrets-best-practice]]; AWS holds only operational/dev secrets like the GitHub PAT). The Stripe secret key is server-only — it never ships in the browser bundle. `STRIPE_WEBHOOK_SECRET` is **not** set here — its `whsec_…` doesn't exist until the webhook endpoint is created in M2.1 build Step 8.

**Tell the student:** "✅ Stripe connected in **sandbox** (`livemode:false`) — test cards only, no real money — and `STRIPE_SECRET_KEY` is in Vercel env. Now let's make your admin account."

---

## Part B — Promote your admin user (one-off migration, no self-escalation)

The barber platform has three roles — `customer`, `shop`, `admin`. M0's sign-up tab only ever writes `customer` or `shop`; **`admin` is never self-served.** You promote exactly one account, once, by running a migration. Here's the full sequence:

### B1 — The person signs up normally through `/login` (and tells you WHICH email) ← manual confirmation #1

> "Decide which account will be the admin (your own email is the obvious choice) and **tell me that email** — I can't guess it, and I'll promote exactly that one account. **Sign up / log in normally on the live site via `/login`** — as a customer or shop, doesn't matter. That creates a `profiles` row with `role` = `customer`/`shop`. We'll flip just that one row to `admin`."

**This is the first of the two things the student decides by hand:** which email becomes the admin. Wait until they give you the email AND confirm the account exists. (If they already have an account from earlier milestones, they can reuse it — no need to make a new one.)

### B2 — Find them in `profiles`

Ask the Supabase MCP (`mcp__claude_ai_Supabase__execute_sql`) to look the account up by email:

```sql
SELECT id, email, role FROM public.profiles WHERE email = '<your-email>';
```

Confirm exactly one row comes back and note its current `role` (expected `buyer` or `shop`). If zero rows → they haven't finished sign-up (redo B1); if the email's wrong → fix it before promoting.

### B3 — Promote with a ONE-OFF MIGRATION (never a raw ad-hoc UPDATE)

Per [[supabase-best-practice]], **all** schema/data changes go through a migration file — *never* a raw ad-hoc prod `UPDATE` in the SQL console. Apply this via `mcp__claude_ai_Supabase__apply_migration` (suggested name `promote_admin_<you>`):

```sql
-- one-off: promote a single, known account to admin
update public.profiles
set role = 'admin'
where email = '<your-email>';
```

> **Note for Claude Code:** keep the `WHERE email = '...'` tight — this migration touches exactly one row. Apply it through `apply_migration` so it's recorded in the migration history (auditable, reproducible), not typed into the console. **Local/dev shortcut:** a `seed.sql` can promote a known dev account locally (`update public.profiles set role='admin' where email='dev@example.com';`) so a fresh local DB always has an admin — but the **remote** promotion is always a migration.

### B4 — Verify the promotion

```sql
SELECT id, email, role FROM public.profiles WHERE email = '<your-email>';   -- role = 'admin'
```

> "Now **log out and back in** on the site so the app re-reads `role='admin'`. The admin-only payout page that uses this role is built in a later milestone — but the admin account must exist *before* that page is useful, which is why we do it now."

**Why this can't be self-escalated (say it once):** there is no UI that writes `admin`; the sign-up tab only writes `buyer`/`shop`; the only path to `admin` is a migration *you* run with the service-role connection. The later admin-only pages additionally keep `role` changes off the client path and gate the admin route server-side ([[supabase-best-practice]]).

---

## Verify (all must pass)

- ✅ **Stripe connected in sandbox** — a read call returns `livemode: false`; the key in play is `sk_test_…` (NOT `sk_live_…`).
- ✅ **`STRIPE_SECRET_KEY` in Vercel env (student-confirmed)** — the sandbox `sk_test_…` is saved as a Production env var (A4, **manual confirmation #2** — you can't read it, the student confirms it). (`STRIPE_WEBHOOK_SECRET` is NOT here — it's created with the webhook endpoint in M2.1 build Step 8.)
- ✅ **You know the two webhook-secret sources** — `stripe listen` (rotating, local) ≠ dashboard endpoint (stable, Vercel prod); the build skill uses the dashboard one ([[stripe-best-practice]] Rule 4).
- ✅ **The student told you WHICH email to promote** — **manual confirmation #1**; you promoted exactly that one account.
- ✅ **Exactly one account promoted** — `SELECT ... WHERE email=...` shows `role='admin'`, applied via `apply_migration` (in the migration history), not a console edit.
- ✅ **Sign-up still only writes `buyer`/`shop`** — `admin` was reached only by your migration; no self-escalation path exists.

## Next step

When all are green, tell the student:
「前置作業完成 ✅ — Stripe 已連到 sandbox（`livemode:false`，只用測試卡）、`STRIPE_SECRET_KEY` 也已放進 Vercel env，而且你已經有一個 `role='admin'` 的帳號了。回到 `m2.1-buyer-to-admin-payments`，跟我說『啟動 M2.1』，我們來接 Stripe Checkout、做付款 webhook，把預約變成『付款成功才確認』。」
Then return to the build skill `m2.1-buyer-to-admin-payments`.

## Reference

- Stripe test/sandbox mode: https://stripe.com/docs/test-mode
- Stripe API keys (test vs live): https://stripe.com/docs/keys
- Stripe CLI / `stripe listen`: https://stripe.com/docs/stripe-cli
- Supabase migrations: https://supabase.com/docs/guides/deployment/database-migrations
- Cross-skill: [[m2.1-buyer-to-admin-payments]] · [[stripe-best-practice]] · [[supabase-best-practice]]
