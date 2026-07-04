---
name: m3-domain-prerequisites
description: 抽成制理髮師預約平台 Milestone 3 prerequisites — a LIGHTWEIGHT CARRYOVER CHECK (no new account setup) before binding a custom domain. Confirms the booking app is deployed and green on Vercel, the AWS API MCP / call_aws is connected and can reach Route 53 (same account from M0), and the student HAS a domain in hand (confirming WHICH registered domain if the account has several — the host format is fixed policy, `barber.<domain>`, never asked). If Stripe is live, flags that the webhook URL will need updating. Use when the student starts M3, or when m3-domain / -checklist detects a missing carryover piece.
---

# M3 Prerequisites — Vercel + Route 53 carryover check

## What this skill does

M3 adds **no new account** — it reuses the **Vercel deployment from M0** and the **AWS account from M0** (the one connected for Secrets Manager, which also holds the Route 53 hosted zone). This skill is a **lightweight carryover check**: confirm the pieces M3 depends on are already in place before touching DNS, rather than setting anything up.

It verifies four things: (1) the app is **deployed and green on Vercel**, (2) the **AWS API MCP / `call_aws` is connected and can reach Route 53**, (3) the student **has a domain in hand** (and *which* one, if the account has several — the host format is fixed policy, `barber.<domain>`, never a question), and (4) *(if Stripe is live)* a note that the **webhook URL will need updating**.

## Architecture

![Barber platform architecture (M3) — the full M2 booking + payout system these prerequisites confirm before the domain is bound. The Product Site box is still labeled *.vercel.app at this point (M3 will relabel it book.yourdomain.com once the domain attaches). DNS for the student's domain lives in an AWS Route 53 hosted zone in the same AWS account M0 connected for Secrets Manager — this prereq confirms call_aws can reach it (a list-hosted-zones read returns without an auth error). The downstream system is unchanged and assumed working: Next.js API routes + Supabase (auth + barbers/services/schedules/bookings/payouts), Stripe Checkout + webhook confirming slots, and the 20/80 commission payout. These prerequisites verify the Vercel deploy is green, call_aws reaches Route 53, and the student has a domain — then hand back to the M3 build skill. Legend: orange = manual input, teal = main component, pink = user data.](assets/architecture-m3.png)

## When to load this skill

- "M3 環境準備" / any time `m3-domain` or `-checklist` detects a missing carryover piece (Vercel not green, `call_aws` can't reach Route 53, or no domain in hand).

This is a carryover check, **not** a setup skill — there is no new account to register. Everything here was provisioned in M0 (Vercel + AWS) and built up through M2.

## Step 1 — The booking app is deployed and green on Vercel

M3 only **re-points a domain** at an already-working deployment — it changes the *address*, not the logic. So the live `*.vercel.app` URL must already serve the booking app.

```bash
# the live Vercel site responds on the key routes (CLI mode)
curl -sS -o /dev/null -w "%{http_code}\n" https://<app>.vercel.app          # 200
curl -sS -o /dev/null -w "%{http_code}\n" https://<app>.vercel.app/login     # 200
curl -sS -o /dev/null -w "%{http_code}\n" https://<app>.vercel.app/barbers   # 200 or redirect to /login
```

> **Cowork: the sandbox `curl` is proxy-blocked** — verify the `200` with a **URL-fetch tool** (e.g. the Vercel URL-fetch MCP `web_fetch_vercel_url`), or just open the URL in a browser. A sandbox `curl` failure does NOT mean the site is down.

If the booking flow isn't working on the Vercel URL yet, **finish M1.2 / M2.x first** — M3 is purely the domain cutover.

## Step 2 — The AWS API MCP / `call_aws` is connected and can reach Route 53

The Route 53 hosted zone lives in the **same AWS account you connected in M0** (for Secrets Manager). M3 needs no new auth — just confirm `call_aws` can read Route 53. One read is enough:

```bash
# returns the hosted zones without an auth error → call_aws reaches Route 53 (same account from M0)
aws route53 list-hosted-zones \
  --query "HostedZones[].{name:Name,id:Id,private:Config.PrivateZone}"
```

- **A public zone whose `Name` == `<domain>.`** (with the trailing dot) → that's the zone M3 will edit; note its `HostedZoneId`.
- **No matching public zone** → the domain's DNS isn't in Route 53 yet. Either it's at an external registrar (in which case M3 Step 2 will create the zone and have the student delegate nameservers), or the student gave the wrong domain. Surface this now, don't block on it.
- **An auth error** (not just "no zones") → the AWS connector isn't reaching the M0 account; reconnect the AWS API MCP (M0 Step 4.2) before proceeding.

> **Note for Claude Code:** this is the **same AWS account and `[default]` profile** as the M0 Secrets Manager setup — you are NOT setting up new credentials. If `list-hosted-zones` errors on auth, it's the connector, not a permissions gap for a new account. (See [[aws-secrets-best-practice]].)

## Step 3 — The student has a domain in hand (confirm WHICH domain — never the format)

This is the **one** genuinely un-discoverable piece — the domain string comes from the student.

- Confirm the student **owns a registered domain** (anywhere — Namecheap / Cloudflare / GoDaddy / Route 53 itself). M3 does **no registration**.
- **The host format is fixed policy — do NOT ask the student to choose it.** M3 always binds a **semantic subdomain `barber.<domain>`** (e.g. `barber.svuncle.com`): one clean `CNAME`, no apex special-casing, no collision. A **path** (`<domain>/barber`) is not a thing (the app binds to a host, not a path); bare **apex** is only an escape hatch if the student *unprompted* insists. Don't surface a "subdomain vs apex vs path" question.
- **The one thing you DO confirm: which registered domain**, when the account has several. If `list-hosted-zones` (Step 2) shows more than one zone (e.g. both `svuncle.com` and `learncodebypicture.com`), ask the student which one the booking app should live under — that's the un-discoverable choice, not the format.

## Step 4 — (If Stripe is live) note the webhook URL will need updating

If the student has already run [[stripe-go-live]] (a real Stripe webhook pointing at `*.vercel.app`), flag that **the Stripe webhook endpoint URL will need repointing to the custom domain** after it attaches (M3 Step 4). The webhook is the source of truth for the slot state machine, so it can't be left dangling. If Stripe is still on sandbox keys, or not wired at all, there's nothing to repoint — note it and move on.

## Verify (all must pass)

- The live `*.vercel.app` URL serves the booking app — `/`, `/login`, `/barbers` all respond (200 or a sign-in redirect) ✅
- `call_aws` / the AWS API MCP returns `list-hosted-zones` **without an auth error** — same account from M0 — and you've identified the hosted zone for the domain (or noted it doesn't exist yet) ✅
- The student **owns a domain**, and if the account has several zones you've confirmed **which registered domain** to use — the host is always `barber.<domain>` (fixed policy; you did **not** ask the student to pick a format) ✅
- *(If Stripe is live)* you've noted the **webhook URL will need updating** after the domain attaches — flag [[stripe-go-live]] ✅
- **No new account or credential setup is needed** — everything reuses M0's Vercel + AWS connection ✅

Anything missing → go back to that milestone first (Vercel deploy → M0/M1.2/M2.x; AWS connector → M0 Step 4.2). Don't start the DNS cutover until all five pass.

## Next step

Return to `m3-domain` Step 1.
