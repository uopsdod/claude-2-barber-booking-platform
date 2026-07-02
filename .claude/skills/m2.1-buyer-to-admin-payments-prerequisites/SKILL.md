---
name: m2.1-buyer-to-admin-payments-prerequisites
description: One-time setup before Milestone 2.1 of the barber booking platform — an INTERACTIVE, agent-driven walkthrough with TWO parts. (1) Set up ALL the pre-code Stripe wiring so the build stays pure code: connect a Stripe SANDBOX account via the Stripe MCP / CLI and verify `livemode:false` (the consent page defaults to LIVE — you must switch to sandbox), put its `STRIPE_SECRET_KEY` (`sk_test_…`) in Vercel env (A4), and create the webhook endpoint against `https://<vercel-url>/api/stripe/webhook` (event `checkout.session.completed`) + put `STRIPE_WEBHOOK_SECRET` (`whsec_…`) in Vercel env (A5) — the endpoint won't deliver until the build ships the route, tested in M2.1 Step 9, which is expected. (2) Promote ONE existing account to `role='admin'` — there is no public admin sign-up; the person signs up normally via /login, you find them with SELECT id,email,role FROM profiles, and promote them with a ONE-OFF Supabase migration (UPDATE public.profiles SET role='admin') applied via `mcp__claude_ai_Supabase__apply_migration`. Admin lives in THIS prereq because admin only matters once payment exists (the later payout/settlement work consumes it). Manual, dashboard-only actions the student MUST do (you can't): WHICH email to promote to admin, and — since Vercel MCP doesn't manage env vars and Stripe MCP doesn't manage webhook endpoints — the two Vercel env vars + the webhook endpoint. Use when the student starts M2.1, says "啟動 M2.1 的前置作業", "set up Stripe sandbox", "connect Stripe", "put the Stripe secret key in Vercel", "set up the Stripe webhook endpoint", "promote my admin user", "create an admin account", or when `m2.1-buyer-to-admin-payments` / `-checklist` detects Stripe isn't connected or no admin exists.
---

# M2.1 Prerequisites — Stripe sandbox + promote your admin user (the agent drives)

**You are the Cowork agent running this skill. Drive the student through it one part at a time** — don't dump the whole thing. For each part: say what's about to happen, do the connector/MCP work yourself, confirm it worked, and move on.

This prereq does **two** things, and only two:

1. **Set up all the pre-code Stripe wiring** — connect a Stripe SANDBOX account (verify `livemode:false`), then do the dashboard/Vercel work so the build stays pure code: put `STRIPE_SECRET_KEY` in Vercel env (A4), and create the webhook endpoint + put `STRIPE_WEBHOOK_SECRET` in Vercel env (A5).
2. **Promote one existing account to `role='admin'`** via a one-off migration.

> **What the student MUST do by hand (you can't perform these).** Everything else you do yourself, but these depend on a human decision/action:
> - **Which email to promote to admin** — you can't guess it; ask, then promote exactly that one account (Part B).
> - **The two Vercel env vars + the webhook endpoint** — Vercel MCP doesn't manage env vars and Stripe MCP doesn't manage webhook endpoints (2026), so the student does these in the dashboards and confirms back: `STRIPE_SECRET_KEY` (A4), the webhook endpoint against the Vercel URL, and `STRIPE_WEBHOOK_SECRET` (A5).
>
> Doing the endpoint + both secrets here means M2.1's build is pure code — no dashboard trips mid-build. The webhook won't *deliver* until the build ships the route (tested in M2.1 Step 9); that's expected.

> **Why admin lives in THIS prereq.** The `admin` role only becomes *meaningful once money exists* — an admin's entire job is the month-end payout reconciliation, which needs paid bookings to reconcile. So the course promotes the admin here, in the payment milestone, right before the settlement work consumes it. There is **no public "sign up as admin" UI** — promotion happens exactly once, through a migration *you* run, which is also why a user can never self-escalate.

**Opening line to the student (say something like):**
> "M2.1 needs some setup before we wire payments: first I'll connect your **Stripe sandbox** account (test mode — no real money) and we'll get all the Stripe keys + the webhook endpoint into Vercel up front, then we'll **promote one account to admin**. I'll need a few things from you along the way — which email should become the admin, and confirmations that the two Stripe secrets + the webhook endpoint are saved (those live in dashboards I can't touch). Two parts, one at a time. Let's start with Stripe."

---

## Part A — Stripe sandbox account + MCP/CLI auth

### A1 — Make sure the student has a Stripe account

> "Do you have a Stripe account? If not, sign up free at https://dashboard.stripe.com/register — you do **not** need to activate the account or submit business details for sandbox/test mode. Going live with real cards is a separate, much-later step."

No activation needed for sandbox. Wait for a yes before connecting.

### A2 — Connect the Stripe connector in Cowork — and SWITCH TO SANDBOX (the foot-gun)

**Walk the student through connecting the Stripe connector.** Say something like:

> "In Cowork, let's add the Stripe connector so I can talk to your Stripe:
> 1. **Add the Stripe connector** (Connectors → Stripe).
> 2. **Enable MCP.**
> 3. When it asks for permissions, click **"Write"** for all permissions.
> 4. Click **Authorize**."

Then do the MCP handshake yourself (`mcp__claude_ai_Stripe__authenticate` → `mcp__claude_ai_Stripe__complete_authentication`).

> ⚠️ **The #1 foot-gun: the Stripe consent page defaults to LIVE.** When the OAuth/consent page opens, there is an **account / mode selector** — it lands on the **live** account by default. **Switch it to a sandbox** (test) account *before* you approve. If you approve on live, every key and call below is a live key and you risk touching real money. If you're not sure which you approved, treat it as wrong and re-auth into sandbox.

**Once connected, ask the student which sandbox to use** (a Stripe account can have several) — say something like:

> "What Stripe Sandbox do I have? / 你想用哪一個 Stripe Sandbox？"

Confirm you're pointed at that sandbox before doing anything else. (This also sets up the `livemode:false` proof in A3.)

**CLI alternative (Stripe CLI):** the student runs `stripe login`, which opens the browser pairing/consent page — **same switch-to-sandbox warning applies** — then pastes the confirmation back. The CLI's `stripe listen` is for **local** webhook testing (it prints a rotating `whsec_…` for `localhost`); the **deployed** Vercel webhook uses the **dashboard endpoint's** stable secret instead (created here in the prereq, A5). Don't confuse the two ([[stripe-best-practice]] Rule 4).

| | Cowork (Stripe Connector) | CLI (Stripe CLI) |
|---|---|---|
| Connect | add connector → enable MCP → click **Write** for all permissions → **Authorize** → `complete_authentication` | `stripe login` → approve in browser → paste back |
| Switch to sandbox | **on the consent page, pick the sandbox account before approving** | **same — pick sandbox on the pairing page** |
| Which sandbox | **ask the student "what Stripe Sandbox do I have?" and confirm** | **same — confirm the sandbox in play** |
| Local webhook test | (test against deployed Vercel URL) | `stripe listen --forward-to localhost:3000/api/stripe/webhook` (rotating secret) |

### A3 — Verify you're actually in SANDBOX (`livemode:false`)

Don't trust the consent screen — **prove** it. Make any read call through the Stripe MCP / CLI and confirm the returned objects carry **`livemode: false`** (and that the API key in play is an `sk_test_…`, not `sk_live_…`):

- Cowork: ask the Stripe MCP to list a couple of recent objects (e.g. balance / a test charge) and check the `livemode` field on the response.
- CLI: `stripe balance retrieve` → the account/object shows test-mode; `stripe config --list` shows the test key in use.

> **Note for Claude Code:** if anything reads `livemode: true` or you see an `sk_live_…`, **stop** — you're on the live account. Re-authenticate and pick the sandbox account. Going live (new `pk_live_`/`sk_live_` keys + a new webhook endpoint on the custom domain) is a separate, later step handled well after this milestone.

### A4 — Put the sandbox `STRIPE_SECRET_KEY` in Vercel env

Now that sandbox is connected, stash its secret key in Vercel so M2.1's build has it ready — this is a pure "copy the key into env" action with no dependency on any M2.1 code, so it belongs here with the rest of the Stripe setup (not in the build). This is one of the manual, dashboard-only actions: Vercel MCP does not manage env vars in 2026, so you can't set or read it — the student pastes it in the dashboard and confirms back to you.

> **在 Stripe 拿 Secret key：** 進 **Stripe Sandbox**（確認右上角在 sandbox / test mode，A2/A3 已切過）→ 搜尋 **"API Keys"** → 複製 **Secret key**（`sk_test_xxxxx`）。
> **加到 Vercel 環境變數：** 進 **Vercel → 你的專案 → Settings → Environment Variables** → 新增：
> - **Key**：`STRIPE_SECRET_KEY`
> - **Value**：`sk_test_xxxxx`
> - **Scope**：Production
> 存檔後**點 Redeploy**（Vercel 在 build/啟動時才讀環境變數，不重新部署不會生效）。

This is an **app-runtime key → Vercel env, NOT AWS Secrets Manager** ([[aws-secrets-best-practice]]; AWS holds only operational/dev secrets like the GitHub PAT). The Stripe secret key is server-only — it never ships in the browser bundle.

### A5 — Create the webhook endpoint + put `STRIPE_WEBHOOK_SECRET` in Vercel env

Create the Stripe webhook endpoint now and stash its `whsec_…` in Vercel too — so **all** the Stripe-dashboard clicking and Vercel-env pasting is done up front, and M2.1's build stays pure code. The endpoint's path (`/api/stripe/webhook`) is a **fixed convention** and the app has been Vercel-deployed since M0, so the URL is fully known now — even though the *route itself* isn't built until M2.1 Step 6/7. That's fine: Stripe registers an endpoint regardless of whether the path yet exists, and hands you the signing secret immediately. Stripe MCP does not manage webhook endpoints in 2026, so this is a **manual dashboard step** too.

> 到 Stripe dashboard（**確認右上角在 sandbox / test mode**）→ Developers → **Webhooks → Add endpoint**：
> - **Endpoint URL**：`https://<your>.vercel.app/api/stripe/webhook`（用你 M0/M1 的正式 Vercel 網址；路徑固定就是 `/api/stripe/webhook`）
> - **Events to send**：只勾 **`checkout.session.completed`**（`charge.refunded` 是 v2，先不要加）
> - 建立後在端點頁點 **Reveal** 拿到 **Signing secret**（`whsec_…`）。
> 然後到 **Vercel → Settings → Environment Variables**，新增 `STRIPE_WEBHOOK_SECRET = whsec_…`（Production scope），存檔。

> **Note for Claude Code:** you are **not** testing event delivery here — you can't, because the `/api/stripe/webhook` route doesn't exist until M2.1 Step 6/7 ships and Vercel redeploys. Until then the endpoint will show failed/404 deliveries in the Stripe dashboard, which is **expected**; the real end-to-end delivery test (test card → `paid`) is M2.1 **Step 9**. This dashboard-endpoint `whsec_…` is **stable** (rolls only if you click "Roll"), and is **different** from the `stripe listen` CLI secret, which rotates each restart and is only for local dev ([[stripe-best-practice]] Rule 4). When M3 later attaches a custom domain, you create a *new* endpoint on that domain — this Vercel-URL one doesn't auto-follow.

**Tell the student:** "✅ Stripe connected in **sandbox** (`livemode:false`) — test cards only, no real money — and both `STRIPE_SECRET_KEY` and `STRIPE_WEBHOOK_SECRET` are in Vercel env, with the webhook endpoint registered against your Vercel URL. (It won't successfully deliver until M2.1 builds the route — that's expected.) Now let's make your admin account."

---

## Part B — Promote your admin user (one-off migration, no self-escalation)

The barber platform has three roles — `customer`, `shop`, `admin`. M0's sign-up tab only ever writes `customer` or `shop`; **`admin` is never self-served.** You promote exactly one account, once, by running a migration. Here's the full sequence:

### B1 — Sign up the account that will become admin (and tell you WHICH email) ← student does this by hand

**Guide the student through it, then wait.** Say something like:

> "Let's make the account that will become your admin. Decide which email to use (your own is the obvious choice) and **tell me that email** — I can't guess it, and I'll promote exactly that one account. On your **live site** (`/login`):
> 1. **Sign up a new user** as a **Customer** (e.g. `uopspop@gmail.com`) — customer or shop, doesn't matter; we only flip the role afterward.
> 2. **Confirm the email** (check the inbox for the confirmation link and click it).
> 3. **Sign in** with that account.
> That creates a `profiles` row with `role = 'customer'`. We'll flip just that one row to `admin`."

Wait until the student gives you the email **and** confirms they've signed up + confirmed + signed in. (If they already have an account from earlier milestones, they can reuse it — no new signup needed; just tell me the email.)

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
- ✅ **`STRIPE_SECRET_KEY` in Vercel env (student-confirmed)** — the sandbox `sk_test_…` is saved as a Production env var (A4 — you can't read it, the student confirms it).
- ✅ **Webhook endpoint created + `STRIPE_WEBHOOK_SECRET` in Vercel env (student-confirmed)** — the dashboard endpoint points at `https://<vercel-url>/api/stripe/webhook`, subscribes only `checkout.session.completed`, and its `whsec_…` is saved as a Production env var (A5). **It won't successfully deliver until M2.1 builds the route + redeploys — that's expected**; the delivery test is M2.1 Step 9, NOT here.
- ✅ **You know the two webhook-secret sources** — `stripe listen` (rotating, local) ≠ this dashboard endpoint (stable, Vercel prod); the build uses the dashboard one you just created ([[stripe-best-practice]] Rule 4).
- ✅ **The student told you WHICH email to promote** — you promoted exactly that one account.
- ✅ **Exactly one account promoted** — `SELECT ... WHERE email=...` shows `role='admin'`, applied via `apply_migration` (in the migration history), not a console edit.
- ✅ **Sign-up still only writes `buyer`/`shop`** — `admin` was reached only by your migration; no self-escalation path exists.

## Next step

When all are green, tell the student:
「前置作業完成 ✅ — Stripe 已連到 sandbox（`livemode:false`，只用測試卡），`STRIPE_SECRET_KEY`、`STRIPE_WEBHOOK_SECRET` 都放進 Vercel env、webhook 端點也建好指向你的 Vercel 網址（在 build 把路由做出來、重新部署後才會真的收到事件，先失敗是正常的），而且你已經有一個 `role='admin'` 的帳號了。回到 `m2.1-buyer-to-admin-payments`，跟我說『啟動 M2.1』，我們來接 Stripe Checkout、做付款 webhook，把預約變成『付款成功才確認』。」
Then return to the build skill `m2.1-buyer-to-admin-payments`.

## Reference

- Stripe test/sandbox mode: https://stripe.com/docs/test-mode
- Stripe API keys (test vs live): https://stripe.com/docs/keys
- Stripe CLI / `stripe listen`: https://stripe.com/docs/stripe-cli
- Supabase migrations: https://supabase.com/docs/guides/deployment/database-migrations
- Cross-skill: [[m2.1-buyer-to-admin-payments]] · [[stripe-best-practice]] · [[supabase-best-practice]]
