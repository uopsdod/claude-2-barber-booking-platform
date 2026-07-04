---
name: m3-domain
description: 抽成制理髮師預約平台 Milestone 3 — bind the student's OWN custom domain to the Vercel-hosted booking app, with DNS hosted in AWS Route 53. The student brings a domain (no registration step); Vercel shows the exact DNS records, and Claude creates them in the Route 53 hosted zone via the already-connected AWS API MCP (call_aws), then waits for Vercel's green ✓ + HTTPS cert, and updates the Stripe webhook URL if Stripe is already live. Use when the student says "啟動 M3", "start M3", "綁網域", "綁自己的網域", "把預約網站掛到我的網域", "go live", "上線", or "正式開張".
---

# M3 — Custom Domain via Route 53（把預約網站掛上你自己的網域）

## What this skill does

Takes the working booking platform (live on `*.vercel.app`) and puts it on **the student's own domain** — `barber.yourdomain.com` instead of `barber-platform.vercel.app`. The student **already owns a domain**; M3 does **no registration**. The only moving part is **DNS**: Vercel tells you exactly which records to create, and you create them in **AWS Route 53** (the same AWS account you connected in M0 — no new auth).

By the end the student has:

1. The custom domain (a **semantic subdomain** — `barber.yourdomain.com`) **added to the Vercel project**, with Vercel showing the exact DNS records it needs.
2. Those **Vercel-provided records created in the Route 53 hosted zone** — a `CNAME` → `cname.vercel-dns.com` for the subdomain (plus, if Vercel asks, a merged `_vercel` ownership TXT), or `A`/`ALIAS` records for an apex — via the AWS API MCP (`call_aws` → `route53 change-resource-record-sets`, always **`UPSERT`**).
3. Vercel showing the domain **Valid / green ✓** with an **automatic HTTPS cert** issued.
4. The live booking app served over **`https://barber.yourdomain.com`** — landing, `/login`, `/barbers`, the booking pop-up — all working on the new host, with **Supabase Auth's Site URL pointed at the new host** so sign-up confirmation emails link there.
5. *(If Stripe is already live)* the **Stripe webhook endpoint URL updated** to the custom domain, so payment confirmations keep flowing (forward-ref [[stripe-go-live]]).

**Out of scope for M3:** registering a domain (the student brings one); the sandbox→live Stripe switch itself (that's [[stripe-go-live]]); any app-code or Supabase changes (M3 only changes the *address*, not the logic).

## When to load this skill

Trigger phrases:
- "啟動 M3" / "start M3" / "begin M3"
- "綁網域" / "綁自己的網域" / "把預約網站掛到我的網域"
- "go live" / "上線" / "正式開張"

Requires M2.2 done (the booking + payout product works on the Vercel URL). Run **[[m3-domain-prerequisites]]** first — it's a lightweight carryover check (Vercel green, `call_aws` reaches Route 53, domain in hand), not a new account setup.

## Execution mode (Cowork-first)

This milestone is **DNS + two dashboards** (Vercel, plus the Stripe dashboard only if live). The Route 53 record creation runs through the **AWS API MCP (`call_aws`)** you connected in M0 — no new auth.

| Action | Cowork mode | Pure CLI mode |
|---|---|---|
| Add domain to Vercel project | **manual dashboard step** — Settings → Domains → Add (no Vercel-domain MCP tool) | `vercel domains add <host>` |
| Create the records in Route 53 | AWS API MCP → `call_aws` (`route53 change-resource-record-sets`) | `aws route53 change-resource-record-sets …` |
| Confirm domain serves HTTPS | Vercel dashboard ✓ + a URL-fetch tool (sandbox `curl` is proxy-blocked) | `curl -sSI https://<host>` |
| Update Stripe webhook (if live) | **manual dashboard step** in Stripe (Stripe MCP doesn't manage webhook endpoints) | Stripe dashboard |

`aws` / `call_aws` commands here run against the **same account from M0**; pin `--region us-east-1` for the Secrets-Manager-adjacent calls (Route 53 is global, but stay consistent).

## Architecture

### Full-platform flow overview (all milestones)

By M3 the whole system is in place, so this is a good moment to see it end-to-end. This diagram is the complete data + money flow the course builds across M0–M2.2 — the thing the custom domain in M3 finally fronts:

![Full barber-booking-platform flow structure — the end-to-end data and money model the course builds, now with the M3 custom domain bound. Each of the three Product Site (Vercel-host) boxes — for the shop, customer, and admin — is labeled with the bound custom domain ({domain}.com), the M3 change: all three roles reach the same Vercel deployment through the student's own domain instead of the *.vercel.app URL. LEFT (the SHOP side, profiles.role='shop'): a shop owner signs in and manages a Product Site that writes to the Supabase database — one shop runs MANY barbers (barbers.shop_id, not unique); each barber has services (price + required_slots, the count of consecutive slots a service needs), sample-work style_photos, and bookable_slots (plain time windows, no status). The shop-level payout bank account lives on profiles. CENTER (the database): the 9 tables — profiles (with role customer/shop/admin, display_name, bank account), platform_settings (single-row currency + slot-minutes config; all money columns are integers in this currency, no _twd suffix), barbers, services, bookable_slots, booking_slots (the join mapping one booking to its N consecutive slots, with a UNIQUE(slot_id) no-double-book guard), bookings (3-state pending_payment→paid→cancelled; a price snapshot; paid_at; a payout_id FK that is NULL until the booking is paid out; NO start_slot_id — the start time is derived as MIN(starts_at) over its booking_slots), commission_rates (the versioned 20% platform / 80% shop ratio, default effective 2026-01-01), and payouts (a flexible settlement BATCH for ONE shop — shop_id NOT unique so a shop has many payouts over time; status pending_transfer→transferred or cancelled; snapshotting gross/platform_cut/shop_cut/shop_name at build time). RIGHT (the CUSTOMER side, role='customer'): a customer browses barbers, opens a barber detail page, and books a service in a pop-up dialog — creating a pending_payment booking that holds its N slots; Stripe Checkout + the webhook (top) then flip that booking to paid and stamp paid_at (no split is computed here, no transactions table). SETTLEMENT (admin): the admin reads the live owed pool (paid bookings WHERE payout_id IS NULL, split derived from commission_rates, attributed per shop via bookings→services→barbers→shop_id), hand-picks a shop's bookings into a payout batch (stamping their payout_id), does the manual bank transfer, and marks the payout transferred — or cancels a pending one to return its bookings to the owed pool. Settlement state is derived from bookings.payout_id, never a booking status.](assets/barber_booking_platform_flow_structure.jpg)

### M3 deployment view

![Barber platform architecture (M3) — the full M2 booking + payout system with the custom domain bound. The Product Site box, previously labeled *.vercel.app, is now labeled book.yourdomain.com (still Vercel-hosted) — that relabel IS the M3 change. DNS for the domain lives in an AWS Route 53 hosted zone (the same AWS account M0 connected for Secrets Manager); the student adds the domain in Vercel, Vercel emits a CNAME (subdomain) or A/ALIAS (apex) record, and Claude UPSERTs that record into the Route 53 zone via the call_aws MCP. Everything downstream is unchanged: the front-end still calls the Next.js API routes + Supabase (auth + barbers/services/schedules/bookings/payouts tables), and Stripe Checkout + the Stripe webhook still mark bookings paid — except, if Stripe is live, the webhook endpoint URL is repointed from *.vercel.app to book.yourdomain.com. AWS Secrets Manager (barber-project/*) and the 20/80 commission monthly-settlement flow are untouched. Legend: orange = manual input, teal = main component, pink = user data.](assets/architecture-m3.png)

How the diagram maps to M3:
- **Product Site relabel `*.vercel.app` → `barber.yourdomain.com`** (top): Step 1 adds the domain in Vercel; Step 2 makes DNS point at Vercel via Route 53; Step 3 turns it green + HTTPS. The hosting target does **not** move — it's the same Vercel deployment, just a new address.
- **Route 53 hosted zone (AWS, from M0):** Step 2 creates the Vercel-provided record here via `call_aws`. **No new AWS auth** — it's the account you set up in M0 for Secrets Manager.
- **Stripe webhook → `barber.yourdomain.com`** (right): Step 4, *only if Stripe is already live* — repoint the webhook endpoint so confirmations keep landing ([[stripe-go-live]]).
- **Everything else unchanged:** Next.js API routes, Supabase tables, the booking lifecycle, the 20/80 monthly settlement — M3 touches no app code.

## Conversational flow

You (Claude Code) drive the student through **4 steps**, in order. Don't dump them all at once — after each step, **wait for confirmation** before moving on. (And run [[m3-domain-prerequisites]] before Step 1.)

1. Add the custom domain (`barber.<domain>`) in Vercel → read off the DNS records Vercel shows (incl. the `_vercel` TXT if the domain is linked elsewhere)
2. Create those records in the Route 53 hosted zone (`call_aws` → `UPSERT`; **merge** the shared `_vercel` TXT)
3. Wait for Vercel to verify (green ✓) + the HTTPS cert → point Supabase Auth Site URL at the new host
4. *(If Stripe is live)* update the Stripe webhook endpoint URL → then **offer** the checklist (don't auto-run it)

---

### Step 1 — Add the custom domain in Vercel and read off the records

> **Bind a short SEMANTIC SUBDOMAIN — `barber.<domain>` — and DON'T ask the student to choose the format.** The host style is fixed policy, not a decision to surface:
> - ✅ **use** `barber.<domain>` (a semantic subdomain — e.g. `barber.svuncle.com`). A subdomain takes a single clean `CNAME`, never collides with anything else on the domain, and skips the apex-`A`/`ALIAS` special-casing.
> - ❌ **never** a **path** like `<domain>/barber` — the app binds to a **host**, not a path; a path isn't a DNS record and isn't an option.
> - Bare **apex** (`<domain>`) is only an escape hatch if the student *unprompted* insists on the root — don't offer it.
>
> **The one thing you DO confirm** (it's genuinely un-discoverable): **which registered domain** to use, when the account has several (e.g. Route 53 holds both `svuncle.com` and `learncodebypicture.com`). Ask *that* — the registered domain — never the host format.

⚠️ **Cowork: adding the domain to the project is a MANUAL dashboard step** — there is no Vercel-domain MCP tool. Have the student do:

> 到 **Vercel → 你的 project → Settings → Domains → Add**，輸入 **`barber.<你的網域>`**（例如 `barber.svuncle.com`；用 `barber.` 這個語意子網域，最單純）→ Add。

**CLI equivalent:**
```bash
vercel domains add barber.yourdomain.com
```

Then Vercel **displays the exact DNS records to create** — read them off the dashboard and create them in Step 2. What it shows depends on subdomain vs apex:

- **Subdomain (default)** → a single **`CNAME`**: `barber.yourdomain.com` → **`cname.vercel-dns.com`**.
  ⚠️ **Use the EXACT target Vercel gives you.** For a host that was ever linked to another Vercel account, Vercel sometimes shows a **project-specific** target (e.g. `…vercel-dns-017.com`) instead of the generic `cname.vercel-dns.com`. Copy whatever Vercel prints — don't assume.
- **Apex (only if the student insisted on `yourdomain.com`)** → an **`A`** record to Vercel's anycast IP (Vercel shows `76.76.21.21`), or — since the zone is in Route 53 — a **Route 53 `ALIAS`** A-record pointing at Vercel. Apex `CNAME` is not allowed at the zone root; use `A`/`ALIAS`.

> ⚠️ **If Vercel says "This domain is linked to another Vercel account" → there's a SECOND record: a `_vercel` ownership-verification TXT.** When the domain (or another host on it) is already attached to a *different* Vercel account/project, Vercel won't verify on the `CNAME` alone — it also shows a message like *"add a TXT record at `_vercel.<domain>` to verify ownership,"* with a value like:
> ```
> TXT  _vercel.yourdomain.com  vc-domain-verify=barber.yourdomain.com,cdfbc229669df39491c6
> ```
> You must create **BOTH** records (the `CNAME` **and** the `_vercel` TXT) or the green ✓ never comes and the student has no idea why. The TXT can be **removed after** Vercel verifies. **This `_vercel` TXT needs the special merge handling in Step 2 — read that before UPSERTing it.**

> Have the student **paste the exact host + record type + target Vercel shows** back to you — **including the `_vercel` TXT if Vercel asked for it**. That string is the source of truth for Step 2.

---

### Step 2 — Create the records in the Route 53 hosted zone

The DNS for the domain lives in a **Route 53 hosted zone** in the **same AWS account you connected in M0** (the prereq's `list-hosted-zones` read already confirmed this). No new auth — just create the record Vercel gave you.

**First, confirm the zone and its id** (read-only):
```bash
# via the AWS API MCP (call_aws) or CLI — find the public zone whose Name == "<domain>." (trailing dot)
aws route53 list-hosted-zones \
  --query "HostedZones[].{name:Name,id:Id,private:Config.PrivateZone}"
```

> **If there is NO hosted zone for the domain yet:** create one, then point the **registrar's nameservers at Route 53** before any record will resolve.
> ```bash
> aws route53 create-hosted-zone --name yourdomain.com --caller-reference $(date +%s)
> # then read the 4 NS values Route 53 assigned…
> aws route53 get-hosted-zone --id <zone-id> --query "DelegationSet.NameServers"
> ```
> Have the student copy those 4 NS records into **their registrar's nameserver settings** (this delegates the whole domain to Route 53). NS delegation can take time to propagate — say so, and continue once it's in.

**Then create the Vercel record — always an `UPSERT`, never a blind overwrite.** For the **subdomain default** (the Vercel `CNAME` from Step 1):
```bash
aws route53 change-resource-record-sets --hosted-zone-id <zone-id> --change-batch '{
  "Changes": [{
    "Action": "UPSERT",
    "ResourceRecordSet": {
      "Name": "barber.yourdomain.com",
      "Type": "CNAME",
      "TTL": 300,
      "ResourceRecords": [{ "Value": "cname.vercel-dns.com" }]
    }
  }]
}'
```
Swap in **whatever exact target Vercel printed** (it may be a project-specific `…vercel-dns-NNN.com`).

> **If Vercel asked for a `_vercel` ownership TXT (Step 1) — it is a SHARED, domain-wide, MULTI-VALUE record. LIST it first and MERGE, never blind-UPSERT.** The `_vercel.<domain>` TXT is a **single record set shared across every Vercel project on that domain** — a naive `UPSERT` of just your one new token **replaces the whole set and knocks the other projects' domains offline**. So:
> 1. **LIST the existing `_vercel` TXT values first:**
>    ```bash
>    aws route53 list-resource-record-sets --hosted-zone-id <zone-id> \
>      --query "ResourceRecordSets[?Name=='_vercel.yourdomain.com.' && Type=='TXT']"
>    ```
>    It may already hold other projects' tokens, e.g.:
>    ```
>    "vc-domain-verify=ai-video-speed-reader.yourdomain.com,9a414e4c759a2678f507"
>    "vc-domain-verify=fly.yourdomain.com,605045c210b0c51f90f2"
>    ```
> 2. **`UPSERT` with ALL values merged** — every existing value **plus** your new one (each a separate quoted string in `ResourceRecords`):
>    ```bash
>    aws route53 change-resource-record-sets --hosted-zone-id <zone-id> --change-batch '{
>      "Changes": [{
>        "Action": "UPSERT",
>        "ResourceRecordSet": {
>          "Name": "_vercel.yourdomain.com",
>          "Type": "TXT",
>          "TTL": 300,
>          "ResourceRecords": [
>            { "Value": "\"vc-domain-verify=ai-video-speed-reader.yourdomain.com,9a414e4c759a2678f507\"" },
>            { "Value": "\"vc-domain-verify=fly.yourdomain.com,605045c210b0c51f90f2\"" },
>            { "Value": "\"vc-domain-verify=barber.yourdomain.com,<your-new-token>\"" }
>          ]
>        }
>      }]
>    }'
>    ```
> 3. After Vercel goes green, the `_vercel` TXT for *your* host can be dropped — but **only your value; re-UPSERT the merged set minus yours**, never delete the whole record.
>
> This is a stronger, more specific case of the "never touch unrelated records" rule below: here the collision is **inside the same record set**, so "don't touch unrelated records" isn't enough — you must **preserve unrelated values within the record you're editing**.

> **Apex case (only if the student chose the bare apex):** Vercel showed an `A` to `76.76.21.21`. `UPSERT` an `A` record at the zone root with that IP, **or** a Route 53 `ALIAS` A-record targeting Vercel — `Type: "A"`, `Name: "yourdomain.com"`. Do **not** create a `CNAME` at the apex.

> **Note for Claude Code:** **always `UPSERT`, never a blind create that could clobber an existing record set.** If the zone already has records for that exact host (e.g. an old parked `CNAME`), `UPSERT` cleanly replaces just that one record set; a manual delete-then-create risks a window of broken DNS. And **never touch unrelated record sets** in the zone — they may belong to the student's email or another project. (See [[aws-secrets-best-practice]] for the Route 53 SOP — same `call_aws` shorthand gotchas as the M0 Secrets Manager calls.)

---

### Step 3 — Wait for Vercel to verify (green ✓) + the HTTPS cert

Once the Route 53 record propagates, **Vercel auto-detects it, marks the domain Valid (green ✓), and issues a TLS cert automatically** — no extra action.

> 回到 **Vercel → Settings → Domains**，等 `barber.<你的網域>` 從「Pending / Invalid Configuration」變成 **綠色的 ✓ Valid**，憑證會自動簽發。DNS 傳播可能要幾分鐘 — 沒馬上變綠很正常，等一下再看。

Confirm the live host actually serves the app over HTTPS:
```bash
# CLI mode:
curl -sSI https://barber.yourdomain.com | head -1     # HTTP/2 200
```

> **Note for Claude Code (Cowork):** the sandbox `curl` is **proxy-blocked** — outbound HTTPS to arbitrary hosts returns 403/000, so a sandbox `curl` failure does NOT mean the site is down. Verify the `200` with a **URL-fetch tool** (e.g. the Vercel URL-fetch MCP `web_fetch_vercel_url` / `get_access_to_vercel_url`), which fetches from outside the sandbox. **Don't trust Vercel `get_project`'s `domains` array for the attach check either — it lags.** Ground truth is *fetching the host and getting a `200`*, not reading the API field. (Propagation: if the first fetch fails, wait a minute and retry before concluding anything.)

**3b — Point Supabase Auth at the new host (the sign-up confirmation email).** The custom domain changes the app's address, but **Supabase Auth still points at the old host** until you update it — and this is silent, because **plain email+password login does no redirect round-trip, so it works on any host regardless of this setting** (which is exactly why the cutover *looks* done without touching Auth). What actually breaks is the **first-time sign-up confirmation email** (and password-reset / magic-link emails): those embed the **Site URL**, so if it still points at `*.vercel.app` or localhost, a brand-new user's "confirm your email" link sends them to the **old host**. Fix it at cutover:

> 到 **Supabase → Authentication → URL Configuration**：
> - 把 **Site URL** 設成 `https://barber.<你的網域>`（這是**註冊確認信**和重設密碼／magic-link 信裡連結的基底 URL）。
> - **Redirect URLs** 加一條 `https://barber.<你的網域>/**`（只放正式網域；`/**` 萬用字元涵蓋所有登入後路徑）。
> - 存檔。

> **Note for Claude Code:** plain **email+password login needs neither** of these (that's the tell — the domain works before you touch this) — but the **sign-up confirm link** and any **OAuth / magic-link / password-reset** email use them, so set them at cutover. **Trade-off of production-only redirect URLs:** OAuth/magic-link/email-confirm redirects **from local dev or the old `*.vercel.app`** are no longer allow-listed — fine for a go-live cutover; add a `http://localhost:<port>/**` entry back **temporarily** only if you later test those email flows locally. **Operational aside:** if **Save** fails, check the Supabase **status page** (there can be an active incident) and retry after it clears — it's not a config error on the student's side.

---

### Step 4 — Update the Stripe webhook URL (only if Stripe is already live)

If you've already done [[stripe-go-live]] (a real Stripe webhook pointing at `*.vercel.app`), the webhook endpoint URL still references the **old** host. The app keeps working from the new domain regardless — but **Stripe will keep POSTing confirmations to the old `*.vercel.app` URL**, which is fine as long as that URL still resolves to the same deployment. To make the brand consistent (and to be safe if you later retire the `.vercel.app` alias), repoint it:

⚠️ **Cowork: updating the webhook endpoint is a MANUAL Stripe dashboard step** — the Stripe MCP does **not** manage webhook endpoints (as of 2026).

> 到 **Stripe Dashboard → Developers → Webhooks → 你的 endpoint → Update details**，把 endpoint URL 從 `https://<app>.vercel.app/api/stripe/webhook` 改成 `https://barber.<你的網域>/api/stripe/webhook`（路徑保持一致）。改完做一筆測試付款，確認 booking 還是會被 webhook 設成 `paid`。

> **Note for Claude Code:** the webhook is the **source of truth** for recording a booking as paid — it's the event that flips the **booking** `pending_payment → paid` (slots have no status). If you repoint it to a host that 404s the webhook path, **payments succeed but bookings never flip to `paid`**. Confirm the new URL resolves (Step 3's `200`) *before* switching, and smoke-test one payment after. (Full sandbox→live migration lives in [[stripe-go-live]].)

If Stripe is **not** live yet (still on sandbox keys with a sandbox webhook, or no Stripe at all), **skip this step** — there's nothing to repoint. When the student later runs [[stripe-go-live]], it sets the live webhook at the custom domain from the start.

**Setup is done.** Tell the student the domain cutover is complete and that a verification checklist (`m3-domain-checklist`) is available **when they want it** — it confirms the Route 53 record matches Vercel, the domain serves HTTPS, auth + booking still work on the new host, and the Stripe webhook (if live) points at the new domain. **Do NOT run it automatically — wait for the student to ask** (e.g. 「驗收 M3」/「上線檢查」), then load the `-checklist` skill.

---

## Things to watch out for (common mistakes)

1. **Bind `barber.<domain>` — don't ask the student to choose a format.** Use the semantic subdomain `barber.yourdomain.com` by default; it's a single clean `CNAME`, never collides with other records, and skips the apex special-casing. A **path** (`<domain>/barber`) is **wrong** (the app binds to a host, not a path) — never offer it; bare apex only if the student unprompted insists. The only thing you confirm is **which registered domain** when the account has several.
2. **Use the EXACT record Vercel prints — don't assume `cname.vercel-dns.com`.** A previously-linked host can need a project-specific target (`…vercel-dns-017.com`). Copy whatever the Vercel dashboard shows; a generic guess silently fails verification.
3. **Apex can't be a `CNAME`.** If the student insists on the bare apex, use an `A` record to Vercel's IP, or a Route 53 `ALIAS` A-record — never a `CNAME` at the zone root.
4. **Always `UPSERT`, never a blind overwrite.** `route53 change-resource-record-sets` with `UPSERT` replaces only that one record set; never delete-then-create (broken-DNS window) and never touch unrelated records in the zone (they may be the student's email/other projects).
5. **No hosted zone yet?** Create it, then point the **registrar's nameservers** at the 4 Route 53 NS values — otherwise nothing resolves no matter how perfect the records are. NS delegation propagates slowly.
6. **DNS propagation is normal latency.** Vercel won't go green, and `curl`/fetch won't `200`, until the record propagates (minutes). A first-try failure is usually just propagation — wait and retry, don't re-edit the record.
7. **Cowork sandbox `curl` is proxy-blocked** — a sandbox `curl` failure is NOT proof the site is down. Verify with a URL-fetch tool from outside the sandbox; and **don't trust Vercel's `domains` array** (it lags) — fetch the host for ground truth.
8. **Stripe webhook URL (if live)** — repoint it to the custom domain via the **Stripe dashboard** (no MCP for webhook endpoints), keep the path identical, and confirm the new URL `200`s before switching, or payments succeed but slots never confirm.
9. **Missing the `_vercel` ownership TXT when the domain is linked elsewhere.** If Vercel says the domain is linked to another account, the `CNAME` alone will never verify — you also need the `_vercel.<domain>` TXT. And that TXT is a **shared, domain-wide, multi-value** record: **LIST it and merge all values** before UPSERTing, or you knock other Vercel projects on that domain offline. (Step 1 + Step 2.)
10. **Forgetting Supabase Auth's Site URL.** Email+password login works on the new host without any Auth change (so the cutover looks done) — but the **sign-up confirmation email** links to whatever **Site URL** is set. Leave it on `*.vercel.app`/localhost and new users get a confirm link to the old host. Set **Site URL** + one `https://barber.<domain>/**` redirect at cutover (Step 3b).

## Expected duration

15–40 minutes — mostly DNS propagation + waiting for Vercel's green ✓ and cert. The Route 53 record itself is one `UPSERT`.

## Next step

Once the domain is green + Supabase Site URL is set, tell the student setup is done and **offer** the checklist (don't run it unprompted). When they later run `m3-domain-checklist` and it's green, celebrate (milestone-scoped — don't surface the next step):
「M3 完成了！你的理髮預約平台現在掛在你自己的網域 `barber.<你的網域>` 上，HTTPS 憑證自動簽好，登入和預約都能用 — 整套抽成制預約 SaaS 從 landing、開店排程、線上付款、到月底撥款後台都跑通了，恭喜上線 🎉」

> **Note for Claude Code (author-only — do NOT volunteer to the student):** the optional follow-on is taking Stripe from test mode to live (swap to `sk_live_`/`pk_live_` keys, point a live webhook at `barber.yourdomain.com`, smoke-test with a real card + refund). Only bring it up if the student asks about charging real money; then load `[[stripe-go-live]]`. Don't seed go-live into the student's mental model at the end of M3.

## Reference

- Vercel custom domains: https://vercel.com/docs/projects/domains
- Vercel + Route 53: https://vercel.com/docs/projects/domains/working-with-domains
- AWS Route 53 `change-resource-record-sets`: https://docs.aws.amazon.com/Route53/latest/APIReference/API_ChangeResourceRecordSets.html
- Route 53 ALIAS records: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-choosing-alias-non-alias.html
- [[m3-domain-prerequisites]] — the lightweight carryover check (Vercel green, `call_aws` reaches Route 53, domain in hand) before touching DNS.
- [[m3-domain-checklist]] — all verification lives here.
- [[aws-secrets-best-practice]] — the Route 53 record-management SOP + `call_aws` shorthand gotchas (same account as M0 Secrets Manager).
- [[stripe-go-live]] — the optional sandbox→live Stripe switch (sets the live webhook on the custom domain).
