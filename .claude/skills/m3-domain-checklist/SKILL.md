---
name: m3-domain-checklist
description: 抽成制理髮師預約平台 Milestone 3 GO-LIVE checklist — verifies the custom-domain cutover: the Route 53 record set matches the values Vercel asked for, the domain resolves over HTTPS, the booking app serves on the custom domain, auth + booking still work there, and (if Stripe is live) the Stripe webhook points at the new domain. Use when the student says "驗收 M3", "上線檢查", "go-live checklist", or after `m3-domain`.
---

# M3 — Go-Live Checklist（上線檢查清單）

## What this skill does

Confirms the things M3 actually changes are live: **(1) the Route 53 record matches what Vercel asked for**, **(2) the domain serves the app over HTTPS**, **(3) auth + booking still work on the custom domain**, and **(4) if Stripe is live, the webhook points at the new domain**. Reports a clear ✅/⚠️/❌ per item.

> **Scope note.** This checklist is scoped to the **domain cutover** — the only thing M3 introduces. It does **not** re-prove the commission math or the payout-admin flow (those are M2.2's checklist); it just confirms the *same* app, at the *new address*, still works end-to-end for a user.

## Architecture

![Barber platform architecture (M3) — this checklist verifies the m3-domain cutover on the full booking + payout system. The Product Site box, now labeled book.yourdomain.com (still Vercel-hosted), is reached via a Route 53 record (CNAME for a subdomain, A/ALIAS for apex) in the same AWS account M0 connected. Section A confirms the Route 53 record matches the Vercel-provided values; Section B confirms the host serves HTTPS and auth + booking still work on it; Section C confirms the Stripe webhook (if live) now points at book.yourdomain.com so booking payments keep landing. Downstream is unchanged: Next.js API routes + Supabase (auth + barbers/services/schedules/bookings/payouts) and Stripe Checkout + webhook driving the booking pending_payment → paid lifecycle (slots have no status — availability is derived from live bookings). Legend: orange = manual input, teal = main component, pink = user data.](assets/architecture-m3.png)

## Execution mode: Cowork vs CLI

| Section | CLI mode tool | Cowork mode equivalent |
|---|---|---|
| A — Route 53 record matches Vercel | `aws route53 list-resource-record-sets` | AWS API MCP (`call_aws`) |
| B — domain serves HTTPS + booking works | `curl` + browser | URL-fetch MCP (sandbox `curl` is proxy-blocked) + browser |
| C — Stripe webhook (if live) | Stripe dashboard | Stripe dashboard (no webhook MCP) |

`aws` / `call_aws` calls run against the **same account from M0**. The sandbox `curl` is proxy-blocked in Cowork — use a URL-fetch tool for any live-host check.

## How to run

Ask the student for: their **custom domain / host** (`barber.yourdomain.com`), the **Route 53 `HostedZoneId`** (or auto-detect via `list-hosted-zones`), and **whether Stripe is already live**. Then run each section and report.

### Section A — Route 53 record matches the Vercel-provided values
- **A1** The record set for the host exists and matches what Vercel asked for in [[m3-domain]] Step 1:
  ```bash
  # via call_aws or CLI — the record for the bound host
  aws route53 list-resource-record-sets --hosted-zone-id <zone-id> \
    --query "ResourceRecordSets[?Name=='barber.yourdomain.com.']"
  ```
  Expect a **`CNAME`** → the **exact** Vercel target (`cname.vercel-dns.com`, or the project-specific `…vercel-dns-NNN.com` Vercel printed). For an **apex** host, expect an **`A`/`ALIAS`** to Vercel's IP, **not** a `CNAME`. A mismatched target = Vercel will never go green. *Recovery:* `UPSERT` the record with the exact value from the Vercel dashboard ([[m3-domain]] Step 2).
  > **If Vercel required a `_vercel` ownership TXT** (the domain was linked to another Vercel account), also confirm the shared `_vercel.yourdomain.com` TXT **still contains your `vc-domain-verify=…` value alongside any others** — it's a domain-wide multi-value record, so if it now holds *only* your value, another project's token was clobbered. *Recovery:* re-`UPSERT` the `_vercel` TXT with **all** values merged ([[m3-domain]] Step 2). Check it: `... list-resource-record-sets ... --query "ResourceRecordSets[?Name=='_vercel.yourdomain.com.']"`.

### Section B — Domain serves HTTPS and the booking flow works
- **B1** The custom domain serves the app over HTTPS with a valid cert:
  ```bash
  curl -sSI https://barber.yourdomain.com | head -1     # HTTP/2 200
  ```
  (A bad/missing cert makes `curl` error out, so a clean `200` also confirms TLS. DNS can take minutes to propagate; a first-try failure may just be propagation — wait and retry.)
  > **Cowork: the sandbox `curl` is proxy-blocked** (403/000 to arbitrary hosts). Don't conclude the site is down from a sandbox `curl` — verify the `200` with a **URL-fetch tool** (e.g. `web_fetch_vercel_url`) that fetches from outside the sandbox.
  > **Don't trust Vercel `get_project`'s `domains` array for the attach check — it lags.** The fetch is ground truth; the API field is eventually-consistent.
- **B2** Deep links resolve on the new host (no SSR-vs-SPA 404). **This is a Vite SPA** with a catch-all rewrite (`"/((?!api/).*)" → /index.html`, from the M2.1 `vercel.json` fix), so **every** non-`/api` path is served by the same `index.html` shell and resolves client-side — deep-link resolution is *guaranteed by the rewrite rule*, not per-path routing.
  > **Cowork caveat (don't mis-verify this):** the Vercel URL-fetch MCP (`web_fetch_vercel_url`) **only fetches the root reliably** — for a subpath it returns *"Unable to create shareable URL for https://barber.yourdomain.com/login."* That is **not a failure**; it's a limitation of the fetch tool on SPA subpaths. So verify B2 by **(a)** the root fetch already `200`ing (B1) **+ (b)** confirming the `vercel.json` catch-all rewrite that excludes `/api/` is present (`"source": "/((?!api/).*)", "destination": "/index.html"`). Do **not** expect a per-path `200` from `web_fetch_vercel_url`.
  ```bash
  # CLI mode only (sandbox curl is proxy-blocked; in Cowork use (a)+(b) above instead):
  curl -sS -o /dev/null -w "%{http_code}\n" https://barber.yourdomain.com/login     # 200
  curl -sS -o /dev/null -w "%{http_code}\n" https://barber.yourdomain.com/barbers   # 200 or redirect to /login
  ```
  Reserve real per-path resolution proof for **actual browser navigation** (B3), which loads the SPA and routes client-side.
- **B3** **Auth + booking still work on the custom domain (the decisive test — STUDENT-PERFORMED).** This needs **real credentials + a browser**, so the assistant can't run it — have the **student** do it and report back: on `https://barber.yourdomain.com`, sign in, open a barber's page (`/barbers/[id]`), open the **booking pop-up**, pick a slot, and confirm the flow runs (Checkout opens if Stripe is wired). This proves the *same* app works at the *new* address — not just that the host serves a 200.
  *Recovery:* plain **email+password login works on any host** (no redirect round-trip), so if that passes but a **sign-up confirmation email** links to the old host, the fix is Supabase **Auth → URL Configuration → Site URL** = `https://barber.yourdomain.com` (+ a `https://barber.yourdomain.com/**` redirect URL) — see [[m3-domain]] Step 3b. If an OAuth/magic-link redirect is refused on the new host, add it to the Redirect URLs list.

### Section C — Stripe webhook points at the new domain (only if Stripe is live)
- **C1** In **Stripe → Developers → Webhooks**, the endpoint URL is `https://barber.yourdomain.com/api/stripe/webhook` (path unchanged), and a test payment flips the **booking** `pending_payment → paid`.
  - **If Stripe is NOT live** (sandbox keys, or Stripe not wired): report **✅ — "sandbox keys, no webhook to repoint"**. There is genuinely nothing to do at cutover, so this is a **pass, not a ⚠️/N/A**. (The live webhook gets set on the custom domain from the start when the student later runs [[stripe-go-live]].)
  - **If Stripe IS live** and the webhook still points at the old `*.vercel.app` host → **❌** (or **⚠️** if you can't confirm). *Recovery:* update the endpoint URL in the Stripe dashboard ([[m3-domain]] Step 4) — the Stripe MCP doesn't manage webhook endpoints. The webhook is the source of truth for recording a booking as paid, so a stale/404 URL = payments succeed but bookings never flip to `paid`. (Full sandbox→live switch: [[stripe-go-live]].)

## Reporting

| Check | Status | Notes |
|---|---|---|
| A1 Route 53 record matches Vercel value | ✅ / ⚠️ / ❌ | CNAME (subdomain) or A/ALIAS (apex), exact target; `_vercel` TXT merged if linked-elsewhere |
| B1 custom domain serves HTTPS 200 | ✅ / ❌ | valid cert |
| B2 deep links resolve (SPA catch-all rewrite present) | ✅ / ❌ | root 200 + `vercel.json` rewrite; per-path fetch N/A in Cowork |
| B3 auth + booking work on the custom domain (**student-run**) | ✅ / ❌ | the decisive test — real credentials + browser |
| C1 Stripe webhook → new domain | ✅ / ❌ | **✅ if Stripe not live (nothing to repoint)**; ❌ only if live & still on old host |

**Verdict:**
- All ✅ (C1 is ✅ "nothing to repoint" if Stripe isn't live yet) → 「🎉 M3 完成！你的理髮預約平台已經掛在自己的網域 `barber.<你的網域>` 上、HTTPS 正常、登入和預約都跑得通 — 整套抽成制預約 SaaS 正式上線了。要開始收真錢的話，下一步是 `stripe-go-live`：換 live 金鑰、把正式 webhook 指到新網域、用真卡刷一筆再退一筆驗證。」
- Any ❌ → fix and re-run `上線檢查`: A1 mismatch → `UPSERT` the exact Vercel target (and re-merge the shared `_vercel` TXT if it was clobbered); B1 not 200 → wait for DNS propagation / re-check the record + Vercel's green ✓; B3 sign-up-email/OAuth lands on the old host → set Supabase **Auth Site URL** to `https://barber.<domain>` + add a `https://barber.<domain>/**` redirect ([[m3-domain]] Step 3b); C1 (live Stripe only) wrong → repoint the Stripe webhook endpoint in the dashboard.
