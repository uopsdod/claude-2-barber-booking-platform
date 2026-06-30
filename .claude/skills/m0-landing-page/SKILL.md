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

## 🔑 Canonical role values (pinned — do not let a build drift)

The three roles are **`customer` · `shop` · `admin`**. The sign-up tab labels are **"Customer"** and **"Barber"**, but the *values* written to `options.data.role` / `profiles.role` are **`customer`** and **`shop`** — **never `buyer`, never `barber`** as a value. `admin` is never written at sign-up (promoted in the M2.1 prereq). If a generated build emits `buyer`, it fails checklist **D3** (role-value mismatch) — **fix the APP to emit `customer`, don't relax the DB.** (Full fix note in the `.txt` Step 9 + [[supabase-best-practice]].)

## How to run

1. **Read [`m0-landing-page.txt`](m0-landing-page.txt) in this folder.** It is the complete, ordered M0 playbook (Steps 1–10) with the verbatim Lovable prompt, all copy-paste prompts, and the Step 9 `profiles` migration SQL.
2. Drive the student through its **10 steps in order** — after each step, **wait for confirmation** before moving on. Don't dump all steps at once.
3. When the build is done, **run the `m0-landing-page-checklist` skill** to verify (it also holds the same copy-paste prompts as an operational appendix).

## Next step

When `m0-landing-page-checklist` is green, tell the student:
「M0 完成了！你現在有一個能用 Customer / Shop 兩種身分註冊登入的線上理髮預約入口網站，而且 Cowork 專案、AWS / Vercel / Supabase 連接器、GitHub token、`profiles.role` 都備好了。準備好的話跟我說『啟動 M1.1』，我們來讓理髮店開店、建立預約排程。」
Then load `[[m1.1-barber-shop-and-schedule]]` (run `[[m1.1-barber-shop-and-schedule-prerequisites]]` first — it reuses the AWS access, GitHub token, and Supabase project set up here).

## Reference

- **Full M0 playbook: [`m0-landing-page.txt`](m0-landing-page.txt)** ← the source of truth
- Best-practice rules this milestone assumes: [[lovable-best-practice]] · [[supabase-best-practice]] · [[aws-secrets-best-practice]]
- Verification: `m0-landing-page-checklist`
- Lovable: https://docs.lovable.dev/ · Supabase Auth: https://supabase.com/docs/guides/auth · Vercel: https://vercel.com/docs/deployments · AWS Secrets Manager: https://docs.aws.amazon.com/secretsmanager/
