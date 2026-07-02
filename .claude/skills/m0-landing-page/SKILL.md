---
name: m0-landing-page
description: 抽成制理髮師預約平台 Milestone 0 — generate the v1 hair-themed landing page with Lovable (a Novara-style marketplace, NOT a 3-card hero), push to GitHub, set up the Cowork project + AWS/Vercel/Supabase connectors, cache the GitHub token in AWS Secrets Manager, deploy to Vercel, and swap auth to the student's own Supabase with a Customer↔Shop role tab + a `profiles.role` stub. Use when the student says "啟動 M0", "start M0", "build the M0 landing page", "理髮師預約平台的入口網站", or any variant that maps to "先做出一個能登入的理髮預約網站".
---

# M0 — Landing Page + Sign-in（先有一個能登入的理髮預約入口網站）

> **📄 The full playbook lives in [`m0-landing-page.txt`](m0-landing-page.txt)** (this skill's own folder) — that plain-text file is the **single source of truth** for M0. **Before doing anything, READ that file and follow its 10 steps verbatim.** This `SKILL.md` is a thin loader so the milestone still triggers and cross-links resolve; it deliberately does **not** duplicate the steps (duplication is how the two drift apart). When you change M0, change the `.txt` — not this stub.

## When to load this skill

Trigger phrases:
- "啟動 M0" / "start M0" / "begin M0"
- "幫我蓋 M0 的 landing page"
- "理髮師預約平台的入口網站" / "理髮預約網站"
- Any prompt referencing "先做出一個能登入的網站"

Do NOT load this skill for M1.1+ — they have their own skills.

## What M0 produces (one-paragraph orientation)

The v1 barber booking platform: a **Novara-style hair-marketplace landing** (serif hero + split photos + search, trust logo strip, 4-icon feature row, "Popular" grid of **featured barbers**) + a `/login` auth page with a **Customer / Barber role tab** + a **role-aware `/barbers` post-login shell** — built on **Lovable → GitHub (public) → Cowork project + AWS/Vercel/Supabase connectors → GitHub PAT cached in Secrets Manager → deployable build pushed → Vercel deploy → auth swapped to the student's own Supabase → a `profiles.role` stub (trigger + backfill)**. Out of scope: barber/schedule tables, booking, Stripe, admin payouts, custom domain (M1.1+). **The exact steps, prompts, and SQL are in [`m0-landing-page.txt`](m0-landing-page.txt) — read it.**

## Architecture

![Barber platform architecture (M0) — the build-and-deploy loop. The student drives Cowork (claude code), which pulls the GitHub PAT from AWS Secrets Manager (the API Keys / aws connector) and pushes the Lovable-generated UI to the GitHub Repo. From the Repo, the Product Site deploys to its Vercel host and the Database (Supabase) is wired via its connector. Lovable (the Landing Page source) is struck through because after M0 the student swaps auth onto their own Supabase — Lovable's role ends once the code is in GitHub. The dashed "Connector" lines are the AWS / Vercel / Supabase connectors set up in this milestone.](assets/architecture-m0.png)

How the diagram maps to M0:
- **You → Cowork (claude code):** the student drives the whole loop from the Cowork project.
- **Cowork → API Keys (aws) → Repo (GitHub):** the GitHub PAT is cached in AWS Secrets Manager and recalled to push the Lovable build to the repo (the **Personal Token** arrow).
- **Repo → Product Site (Vercel) + Database (Supabase):** the pushed build deploys to Vercel; auth is then swapped onto the student's own Supabase (the `profiles.role` stub).
- **Landing Page (Lovable) struck through:** Lovable generates the v1 UI, but once the code lives in GitHub the student moves off Lovable Cloud — it's a starting point, not a runtime dependency.

## 🔑 Canonical role values (pinned — do not let a build drift)

The three roles are **`customer` · `shop` · `admin`**. The sign-up tab labels are **"Customer"** and **"Barber"**, but the *values* written to `options.data.role` / `profiles.role` are **`customer`** and **`shop`** — **never `buyer`, never `barber`** as a value. `admin` is never written at sign-up (promoted in the M2.1 prereq). If a generated build emits `buyer`, it fails checklist **D3** (role-value mismatch) — **fix the APP to emit `customer`, don't relax the DB.** (Full fix note in the `.txt` Step 9 + [[supabase-best-practice]].)

## How to run

1. **Read [`m0-landing-page.txt`](m0-landing-page.txt) in this folder.** It is the complete, ordered M0 playbook (Steps 1–10) with the verbatim Lovable prompt, all copy-paste prompts, and the Step 9 `profiles` migration SQL.
2. Drive the student through its **10 steps in order** — after each step, **wait for confirmation** before moving on. Don't dump all steps at once.
3. When the build is done, **run the `m0-landing-page-checklist` skill** to verify (it also holds the same copy-paste prompts as an operational appendix).

## Next step

When `m0-landing-page-checklist` is green, tell the student (milestone-scoped — celebrate M0, don't surface the next milestone):
「M0 完成了！你現在有一個能用 Customer / Shop 兩種身分註冊登入的線上理髮預約入口網站，而且 Cowork 專案、AWS / Vercel / Supabase 連接器、GitHub token、`profiles.role` 都備好了。入口網站這一步到這裡就完整了 ✅」

> **Note for Claude Code (author-only — do NOT volunteer to the student):** the next milestone is the shop/seller setup (list barbers, services, schedule slots, payout bank info). When the instructor advances the student (or the student asks "what's next"), run `[[m1.1-seller-setup-prerequisite]]` first (it reuses the AWS access, GitHub token, and Supabase project set up here), then load `[[m1.1-seller-setup]]`. Don't name or seed the next milestone into the student's mental model at the end of M0.

## Reference

- **Full M0 playbook: [`m0-landing-page.txt`](m0-landing-page.txt)** ← the source of truth
- Best-practice rules this milestone assumes: [[lovable-best-practice]] · [[supabase-best-practice]] · [[aws-secrets-best-practice]]
- Verification: `m0-landing-page-checklist`
- Lovable: https://docs.lovable.dev/ · Supabase Auth: https://supabase.com/docs/guides/auth · Vercel: https://vercel.com/docs/deployments · AWS Secrets Manager: https://docs.aws.amazon.com/secretsmanager/
