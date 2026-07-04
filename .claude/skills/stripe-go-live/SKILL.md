---
name: stripe-go-live
description: The sandbox→live cut-over checklist for Stripe on the 抽成制理髮師預約平台 — activate the Stripe account (phone + bank + business info), swap `pk_test_`/`sk_test_` for `pk_live_`/`sk_live_` in Vercel env, create a NEW live webhook endpoint on the custom domain (a new `whsec_`), redeploy, and smoke-test with a REAL card + refund. Use when the student says "go live", "上線收真錢", "切到 live mode", "switch Stripe to production", after M3 attaches the custom domain, or when a live payment fails. Simpler than a fixed-price product because our prices are dynamic — there are NO Price rows to re-create. Cross-ref [[m3-domain]] and [[stripe-best-practice]].
---

# Stripe Go-Live (Barber Booking — sandbox → live)

Going live means switching the platform from **test mode** (test cards, fake money) to **live mode** (real cards, real money into the platform's Stripe balance). For this project the cut-over is **deliberately small** because our Checkout uses **dynamic `price_data`** — there are **no fixed Stripe Price/Product rows to re-create** in live mode (the credits-style reference course had to migrate price tiers; we don't). What actually changes is: **the account must be activated, the keys swap, and a brand-new live webhook endpoint is created on the custom domain.** Everything else is identical to what M2.1 already built.

Run this **after M3** has attached the custom domain (the live webhook should point at the real domain, not a `.vercel.app` URL). When you (Claude Code) walk a student through this, do it as an ordered checklist and **don't skip the activation step** — live keys don't work until the account is activated.

> **The mental model:** test mode and live mode are **two parallel worlds** in one Stripe account. They have **separate keys, separate webhook endpoints, separate event logs, and separate data**. Nothing you created in test mode (bookings, payments, the test webhook endpoint) exists in live mode. You are re-pointing the same app code at the live world by swapping env vars — the code does not change.

---

## Execution mode: Cowork vs CLI

| Operation | CLI mode | Cowork mode |
|---|---|---|
| Activate the account | dashboard → **Activate account** (phone + bank + business) | dashboard step (no MCP) |
| Reveal live keys | dashboard → Developers → API keys (toggle **View live**) | dashboard step |
| Set `*_live_*` in Vercel env | dashboard → Settings → Environment Variables | **dashboard step — the Vercel MCP does not set env vars (2026)** |
| Create the live webhook endpoint | dashboard → Developers → Webhooks → Add endpoint | **dashboard step — the Stripe MCP does not manage webhook endpoints (2026)** |
| Smoke-test a payment | a real card on the live site | a real card on the live site |
| Refund the smoke-test | dashboard → Payments → the charge → Refund | dashboard step (or Stripe MCP in **live** mode — be deliberate) |

Almost every go-live action is a **dashboard step** — neither the Stripe MCP nor the Vercel MCP automates webhook endpoints or env vars. Plan for clicking, not calling.

---

## Section 1 — Activate the Stripe account

Live keys exist but **return errors until the account is activated**. Activation is Stripe's KYC: it needs real business + payout details.

1. Dashboard → **Activate account** (or Settings → Account).
2. Provide: **business type / legal name**, a **phone number** (verified), the **business website** (your custom domain from M3 is ideal), a short product description ("online barber appointment booking"), and a **bank account** for payouts — this is **the platform's** bank account (where the 100% you collect settles), *not* a barber's.
3. Submit and wait for "Your account is active" / payouts **enabled**. Some regions clear instantly; others take review time.

> **Why the platform's bank, not a shop's:** there is **no Stripe Connect** here — the platform collects 100% into its **own** Stripe balance, and the admin bank-transfers each shop's 80% manually via per-shop payout batches (M2.2). So Stripe only ever needs the **platform's** payout bank account. ([[stripe-best-practice]] — money flow.)

---

## Section 2 — Swap the keys (`sk_test_`/`pk_test_` → `sk_live_`/`pk_live_`) in Vercel env

1. Dashboard → Developers → **API keys** → toggle to **live** (top-right "View test data" off) → reveal:
   - **Publishable key** `pk_live_…` (browser-safe).
   - **Secret key** `sk_live_…` (server-only — never in the front-end, never committed).
2. In **Vercel → Settings → Environment Variables** (Production scope), replace:
   - `STRIPE_SECRET_KEY` = `sk_live_…`
   - the publishable key (e.g. `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY`) = `pk_live_…`
   - **leave `STRIPE_WEBHOOK_SECRET` for Section 3** — it changes too, but only after you create the live endpoint.
3. Do **not** redeploy yet — finish Section 3 so all three live values land in one redeploy.

> App-runtime Stripe keys live in **Vercel env**, never in AWS Secrets Manager (which is for the GitHub PAT + local-dev keys only). See [[aws-secrets-best-practice]].

---

## Section 3 — Create a NEW live webhook endpoint on the custom domain (new `whsec_`)

The test-mode webhook endpoint **does not carry over** to live mode — and it points at the wrong URL. Create a fresh one in the live world, on the M3 custom domain.

1. Dashboard (still in **live** mode) → Developers → **Webhooks** → **Add endpoint**.
2. **Endpoint URL** = your **custom domain** + the route: `https://book.yourdomain.com/api/stripe/webhook` (the M3 domain, **not** the `.vercel.app` URL — see [[m3-domain]]).
3. **Events to send**: at minimum `checkout.session.completed` (the event M2.1's handler keys on).
4. Save → open the new endpoint → reveal its **Signing secret** `whsec_…` (this is a **new, live** secret, different from both the test-dashboard secret and any `stripe listen` secret).
5. Set `STRIPE_WEBHOOK_SECRET` = that live `whsec_…` in **Vercel env** (Production).
6. **Now redeploy** Vercel so all three live values (`sk_live_`, `pk_live_`, live `whsec_`) take effect together.

> **Why a new endpoint, not an edit:** the live world has its own webhook registry; there's nothing to edit because nothing exists there yet. And the URL must be the **custom domain** so Stripe reaches your app over HTTPS at its real address. If you go live **before** M3, point it at the `.vercel.app` URL temporarily, then **come back and re-point it** to the domain once M3 lands. ([[stripe-best-practice]] Rule 4 — the secret per endpoint is stable but unique.)

---

## Section 4 — Smoke-test with a REAL card, then refund

Test cards **stop working in live mode** (`4242…` is declined). The only way to verify live is a real charge — so make it small and refund it immediately.

1. On the **live m3-domain site**, book a slot and pay with a **real card** (yours). Pick the cheapest service so the real charge is small.
2. Verify the full path end-to-end:
   - Stripe dashboard (**live**) → Payments shows a **succeeded** charge for the **right amount** (e.g. NT$**500**, not NT$50,000 — TWD zero-decimal config, [[stripe-best-practice]] Rule 0).
   - Dashboard → Developers → Webhooks → your live endpoint → the `checkout.session.completed` delivery shows **200**.
   - Supabase: the booking flipped to **`paid`** and `paid_at` is set, with `payout_id` still NULL (= owed). (No split is computed at payment — the 20/80 split is computed when the admin builds a payout, M2.2.)
3. **Refund it:** dashboard → Payments → the charge → **Refund** (full).

> ⚠️ **A refund in v1 does NOT auto-reverse the settlement.** Refunding the Stripe charge returns the money to the card, but it does **not** touch the `paid` booking or remove it from M2.2's owed pool / a built `payouts` batch. So a refunded smoke-test (or any real refund) would still *appear* as money owed to the shop. **For the smoke-test, also delete/void that test booking row** (or exclude it) so it doesn't pollute the payout page. **Automatic refund→settlement reversal is out of scope for v1 — flag it as v2.** (See Out of scope.)

---

## Section 5 — Final go-live verification

- [ ] Account shows **active / payouts enabled** (Section 1).
- [ ] Vercel Production env has `sk_live_…`, `pk_live_…`, and the **live** `whsec_…` — and you **redeployed** after setting them.
- [ ] The live webhook endpoint URL is the **custom domain** `/api/stripe/webhook` (not `.vercel.app`), subscribed to `checkout.session.completed`.
- [ ] A **real-card** booking went through end-to-end (succeeded charge, 200 webhook, booking `paid`, `paid_at` stamped) — and was **refunded + the row cleaned up**.
- [ ] No `sk_live_`/`whsec_` anywhere in the repo or the browser bundle (grep the deployed front-end — only `pk_live_` may appear client-side).

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Live payment errors with "account not activated" / payouts disabled | Section 1 not done (or still under review) | Complete activation (phone + bank + business); wait for "active" |
| `4242 4242 4242 4242` is declined on the live site | You're in **live mode** — test cards don't work in live | Use a **real** card; refund it (Section 4) |
| Live webhook deliveries 400 (`signature verification failed`) | `STRIPE_WEBHOOK_SECRET` is the **test** or `stripe listen` secret, not the **live endpoint's** | Copy the live endpoint's signing secret into Vercel env → redeploy ([[stripe-best-practice]] Rule 4) |
| Stripe dashboard says "delivered" but the booking never flips to `paid` | Webhook hitting the wrong URL, or middleware redirecting it | Confirm the endpoint URL is the custom domain `/api/stripe/webhook`; exempt it in middleware ([[stripe-best-practice]] Rule 5) |
| Charge amount is 100× too high (NT$50,000) | TWD zero-decimal foot-gun crept back in | `unit_amount = price × 10^currency_minor_units` — TWD config is ×1 ([[stripe-best-practice]] Rule 0) |
| Live charges still go to the test event log | App still running on `sk_test_`/`pk_test_` | Confirm Vercel Production env has the `*_live_*` keys **and** you redeployed |
| Webhook 200s but the booking stays `pending_payment` | Handler guarded on the wrong status, or a stale deploy | Confirm the `payment_status === 'paid'` branch ran and the status guard matched ([[stripe-best-practice]] Rule 9); redeploy |

---

## Out of scope for v1 (flagged for v2)

- **Refund → automatic settlement reversal.** A Stripe refund does not back a paid booking out of M2.2's owed pool / a built `payouts` batch. v1 handles refunds manually (refund in Stripe + manually correct/exclude the booking row). A real v2 would write a `refunded`/`cancelled` booking status that the settlement excludes.
- **Stripe Connect / automated shop payouts** — still manual bank transfer per payout batch (M2.2).
- **Disputes / chargebacks handling**, **Radar fraud rules**, **multi-currency** — single-currency (TWD), single-card pay-now only.
- **Re-creating Price/Product rows** — **N/A here**: prices are dynamic `price_data`, so there are no fixed Stripe Price objects to migrate from test to live (this is why our go-live is simpler than the reference's).

---

## Cross-references

- [[stripe-best-practice]] — the rules the live integration must already satisfy (currency-aware `unit_amount`, raw-body verify, idempotency, webhook secret, the webhook-records-`paid` model).
- [[m3-domain]] — attach the custom domain **first**; the live webhook endpoint URL must be that domain.
- [[m3-domain-prerequisites]] — flags that the Stripe webhook URL needs updating once the domain attaches.
- [[aws-secrets-best-practice]] — Stripe live keys go in **Vercel env**, not AWS.
- [[m2.2-admin-to-seller-payment]] — the per-shop payout settlement a refund does **not** auto-reverse in v1.
- [[stripe-mysite]] — a sibling project's live-mode cut-over checklist (same key-swap + dashboard-endpoint pattern).
