---
name: aws-secrets-best-practice
description: The AWS operating SOP for the 抽成制理髮師預約平台 — a TWO-job AWS footprint: Secrets Manager (for OPERATIONAL/DEV secrets only, the `barber-project/*` namespace: the GitHub PAT and any local-dev API key) and Route 53 (DNS record management for the M3 custom domain). Covers region pinning (us-east-1), the explicit rule that APP-RUNTIME keys (Stripe, Supabase) live in VERCEL ENV not AWS, hosted-zone basics + CNAME/A/ALIAS UPSERT for M3, AWS MCP (`call_boto3`) conventions, and why Supabase Vault is NOT used here. Use whenever a student is storing the GitHub PAT, setting up AWS credentials/MCP, or wiring Route 53 DNS for the domain. Apply proactively.
---

# AWS Secrets Best Practice (Barber Booking — Secrets Manager + Route 53)

This course uses AWS for **exactly two jobs**, and the discipline is in keeping them straight:

1. **Secrets Manager** — stores **operational / developer** secrets only, under the **`barber-project/*`** namespace: the **GitHub PAT** (`barber-project/github`) that lets Claude Code push to the repo, and any **API key you use during local development**. These are reached during the *build*, not at app request time.
2. **Route 53** — holds the **DNS records** for the M3 custom domain. We do **not** register a domain via AWS; we take the **Vercel-provided DNS records** and enter them into Route 53.

The **bright line** that prevents most confusion: **app-runtime keys** — `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `SUPABASE_SERVICE_ROLE_KEY`, the Supabase URL + publishable key — live in **Vercel environment variables**, **not** AWS. They're read by Next.js API routes at request time; routing them through AWS would add a request-time AWS dependency for no benefit. AWS holds the secrets you reach for *while building*; Vercel holds the secrets the *running app* reads.

AWS access in Cowork is the official **AWS MCP** connector, signed in through the browser (OAuth — no IAM user, no access key; set up in [[m0-landing-page]] Step 4.2). Its `aws___run_script` tool runs a short Python script that calls AWS with `call_boto3(...)`. **Every** call pins **`region_name="us-east-1"`** (CLI: `--region us-east-1`). When you (Claude Code) guide a student through AWS work, **apply these rules proactively** — stop them before they break one.

> **AWS here is small and deliberate — no compute stack.** This course runs **no** Lambda / DynamoDB / SQS / API-Gateway — all app logic is Next.js API routes (Vercel serverless functions) + Supabase, so any Lambda/DDB/VPC/IAM-role rules **do not apply**. AWS does exactly two jobs: the **two-kinds-of-secret** discipline (the GitHub PAT in Secrets Manager) and **Route 53** DNS for the M3 custom domain — both with **region pinning**. That's it.

---

## Execution mode: Cowork vs CLI

The hard rules apply identically in both — only the command surface differs.

| Operation | CLI mode | Cowork mode (AWS MCP) |
|---|---|---|
| Run any AWS API call | `aws <service> <verb> … --region us-east-1` | `aws___run_script` → `await call_boto3(service_name="<service>", operation_name="<PascalCaseOperation>", region_name="us-east-1", params={…})` |
| Store/read a secret | `aws secretsmanager create-secret / get-secret-value …` | `secretsmanager` `CreateSecret` / `PutSecretValue` / `GetSecretValue` / `ListSecrets` |
| Manage a DNS record | `aws route53 change-resource-record-sets …` | `route53` `ChangeResourceRecordSets` |
| List hosted zones (sanity check) | `aws route53 list-hosted-zones --region us-east-1` | `route53` `ListHostedZones` |

**Region:** **every** call passes `us-east-1` — `region_name="us-east-1"` in Cowork, `--region us-east-1` in CLI. Forgetting the region is the #1 "works for me but not in the script" gap (Rule 1). CLI mode authenticates however the student set up their local `aws` (`aws sso login` / `aws configure`); Cowork needs nothing beyond the AWS MCP sign-in.

> **AWS MCP conventions (`call_boto3`):**
> - `operation_name` is the **PascalCase API operation** (`CreateSecret`, `ChangeResourceRecordSets`) — not the CLI spelling (`create-secret`) and not the snake_case boto3 method.
> - Each `aws___run_script` call is **self-contained** — no variables survive between calls, so each script re-discovers what it needs (secret name, hosted-zone id).
> - End the script with `result = {…}` then `result` on its own line so the tool returns it. Filter in Python (`[s["Name"] for s in resp["SecretList"]]`) — there's no `--query`.
> - Verify by re-reading state (`GetSecretValue`, `ListResourceRecordSets`) in the same or a follow-up script.
> - If AWS tools disappear or calls fail with a sign-in / auth error, the session expired — reconnect the AWS MCP connector and sign in again. Don't fall back to an access key. If the old **AWS API MCP** (`call_aws`) is still enabled, disable it.

---

## Hard rules

### Rule 1 — Pin `us-east-1` on every call (`region_name` in Cowork, `--region` in CLI).

> **The rule:** No `call_boto3(...)` without `region_name="us-east-1"`, no `aws …` without `--region us-east-1`. Note: **Route 53 is a global service** but pass `us-east-1` there too — one habit, no exceptions.

**Why:** A Secrets Manager call without an explicit region either errors (`You must specify a region`) or silently hits a *different* default region where your secret doesn't exist — and you get `ResourceNotFoundException` for a secret you demonstrably created (in us-east-1). Pin it everywhere and "not found" stops being a region mystery.

**How to apply:** Put `us-east-1` on every call. When something returns "not found" right after you created it, **check the region first** before assuming it wasn't created.

---

### Rule 2 — Secrets Manager is for OPERATIONAL/DEV secrets only (`barber-project/*`). APP-RUNTIME keys go to Vercel env, not AWS.

> **The rule:** Store in Secrets Manager **only** the secrets you use *while building*: the **GitHub PAT** (`barber-project/github`) and any **local-dev API key** — under the **`barber-project/*`** namespace. **Never** put `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, or `SUPABASE_SERVICE_ROLE_KEY` in AWS — those are **app-runtime** keys and live in **Vercel environment variables**.

**Why:** There are **two kinds of secret with two homes**, and mixing them is the central confusion:
- **Operational/dev secrets** (GitHub PAT, local-dev keys) are read by **you/Claude Code during the build** — to push code, to run a script locally. Caching them once in Secrets Manager means a fresh Cowork session **recalls the same token** instead of re-pasting it.
- **App-runtime secrets** (Stripe, Supabase service-role) are read by **Next.js API routes at request time, on Vercel**. The idiomatic, lowest-latency place for those is **Vercel env** — the server reads them directly. Routing them through AWS would make every webhook/checkout call do an AWS `GetSecretValue` first: a needless request-time dependency, more latency, more IAM surface, for zero benefit.

Putting a Stripe/Supabase runtime key in AWS isn't a security win — it's an architecture smell that adds a moving part. Keep the line clean.

**How to apply — check-then-collect (the SOP every prereq follows):**
```python
# does it already exist? (a returning session almost always: yes — don't re-collect)
resp = await call_boto3(service_name="secretsmanager", operation_name="ListSecrets",
                        region_name="us-east-1")
names = [s["Name"] for s in resp["SecretList"]]
result = {"github_secrets": [n for n in names if "github" in n.lower()]}
result
#   → ["barber-project/github"]  ⇒ SKIP collection, reuse it
#   → []                         ⇒ collect the PAT, then (next script):
#      await call_boto3(service_name="secretsmanager", operation_name="CreateSecret",
#                       region_name="us-east-1",
#                       params={"Name": "barber-project/github",
#                               "SecretString": "<the github_pat_… token>"})
# update an existing one → PutSecretValue (replaces the whole value)
```
(CLI mode: `aws secretsmanager list-secrets --region us-east-1 --query "SecretList[].Name"`, then `aws secretsmanager create-secret --name barber-project/github --secret-string '<token>' --region us-east-1`.)
- The canonical operational set: **`barber-project/github`** (the GitHub PAT — a real write-credential; scope it Contents:RW on the one repo) and **`barber-project/<dev-tool>`** for any local-dev API key.
- **PAT secret name can differ** — that's why the check above lists all names and matches `github` rather than looking up one exact name (a real build had it under `github/personal-access-token`). Use whichever exists; don't re-collect.
- **App-runtime keys:** set them in **Vercel → Settings → Environment Variables** (Production scope). Stripe live-key handling: [[stripe-go-live]]. Supabase key split: [[supabase-best-practice]] Rule 4.
- **Chat-retention caveat:** a key briefly appears in the transcript on its way to `CreateSecret`; because it's stored **once** (not re-pasted every session) exposure is small, but rotate at course end.

---

### Rule 3 — Route 53 (M3): take the Vercel-provided records and UPSERT them into the hosted zone. We don't register domains.

> **The rule:** For M3, **Vercel shows the exact DNS records** to create when you add the custom domain — typically a **CNAME** → `cname.vercel-dns.com` for a **subdomain** (`book.yourdomain.com`), or **A / ALIAS** records for an **apex**. Open the domain's **hosted zone** in Route 53 and create **exactly those records**, using **`UPSERT`** (create-or-replace, idempotent). We do **not** register a domain via AWS.

**Why:** Route 53's API is all-or-nothing per change-batch and isn't forgiving of guesses. The **subdomain CNAME** path is the simplest and least error-prone — prefer it. Using **`UPSERT`** (rather than `CREATE`) makes the change **idempotent**: re-running won't fail with "record already exists," and it cleanly overwrites a stale value. The whole job is "copy Vercel's records verbatim into the right hosted zone" — the failure modes are (a) wrong hosted zone, (b) the zone's nameservers not actually delegated at the registrar, (c) a typo'd target.

**How to apply:**
1. **Find the hosted zone** for the domain: `route53` `ListHostedZones` → `[(z["Name"], z["Id"]) for z in resp["HostedZones"]]` (CLI: `aws route53 list-hosted-zones --region us-east-1 --query "HostedZones[].[Name,Id]"`). If the zone doesn't exist, create it (`CreateHostedZone`) **and point the registrar's nameservers at the four Route 53 NS records** first — DNS won't resolve until delegation is in place.
2. **In Vercel:** add the custom domain (subdomain default) → copy the **exact** records Vercel displays.
3. **UPSERT into Route 53** (subdomain CNAME example):
   ```python
   resp = await call_boto3(
       service_name="route53", operation_name="ChangeResourceRecordSets", region_name="us-east-1",
       params={"HostedZoneId": "<Z...>",
               "ChangeBatch": {"Changes": [{"Action": "UPSERT", "ResourceRecordSet": {
                   "Name": "book.yourdomain.com", "Type": "CNAME", "TTL": 300,
                   "ResourceRecords": [{"Value": "cname.vercel-dns.com"}]}}]}})
   result = resp["ChangeInfo"]["Status"]   # PENDING → INSYNC within ~60 s
   result
   ```
   (CLI: `aws route53 change-resource-record-sets --hosted-zone-id <Z...> --region us-east-1 --change-batch '{"Changes":[{"Action":"UPSERT",…}]}'`.)
   (Apex: Vercel gives an A record / ALIAS target — use that instead. ALIAS-to-CloudFront-style targets use `AliasTarget`, not `ResourceRecords`.)
4. **Back in Vercel:** wait for the domain to verify (green ✓) and the HTTPS cert to issue.
5. **Verify resolution:** `dig +short book.yourdomain.com` (CLI) should return the Vercel target; then load the site over HTTPS and confirm auth + booking still work.
6. **If Stripe is live**, update the **Stripe webhook endpoint URL to the custom domain** afterward — [[stripe-go-live]], [[m3-domain]].

> The AWS MCP is **already connected from M0** — M3 adds **no new AWS setup**, just Route 53 record creation. ([[m3-domain-prerequisites]] is a lightweight carryover check, not a new account.)

---

## Why not Supabase Vault (decision recorded)

A reasonable question: Supabase **Vault** is an encrypted Postgres table, and in Cowork the Supabase MCP **can** read a Vault secret in-session via `execute_sql` (`SELECT decrypted_secret FROM vault.decrypted_secrets WHERE name = '…'`) and use it — e.g. to fetch the GitHub PAT for a `git push`. Functionally that's **equivalent** to fetching the PAT from AWS via the AWS MCP. So why AWS Secrets Manager?

**Because we need an AWS account for Route 53 (M3) regardless.** Given AWS is already in the stack for DNS, **reusing the same account for Secrets Manager adds no new dependency** — whereas Vault would be the *only* reason to lean on Supabase for secrets, and it wouldn't remove AWS from the picture (DNS still requires it). On top of "no new dependency," AWS Secrets Manager gives:

- **Finer, purpose-built IAM** — scope `GetSecretValue` to `arn:aws:secretsmanager:us-east-1:<acct>:secret:barber-project/*`, separate from database access.
- **Plaintext reads kept off the DB connection** — fetching the PAT doesn't run a `SELECT decrypted_secret …` over the same connection the app uses for live data; secret reads are a distinct, CloudTrail-logged AWS API path.
- **One mental model for "build-time secrets"** — they all live in `barber-project/*`, found the same way every session.

**Net:** Vault would mainly earn its keep if we wanted to **drop AWS entirely** — which we can't, because of DNS. Since AWS is non-negotiable for Route 53, Secrets Manager is the lower-friction home for the operational secrets. (This says nothing about *app* data, which lives in Supabase tables with RLS — see [[supabase-best-practice]].)

---

## Things to actively watch out for

1. **`ResourceNotFoundException` on a secret you just created** → Rule 1 (no `--region us-east-1` → wrong default region).
2. **A Stripe/Supabase service-role key in Secrets Manager** → Rule 2 (app-runtime keys go to **Vercel env**, not AWS — move it).
3. **Re-pasting the GitHub PAT every session** → Rule 2 (it's cached in `barber-project/github`; check-then-collect, and grep both possible names).
4. **DNS won't resolve after creating the Route 53 record** → Rule 3 (the registrar's nameservers aren't delegated to this hosted zone, or you edited the wrong zone).
5. **`ChangeResourceRecordSets` fails "record already exists"** → Rule 3 (use **`UPSERT`**, not `CREATE`).
6. **AWS MCP call fails** → check `operation_name` is PascalCase (`ListSecrets`, not `list-secrets`) and `region_name="us-east-1"` is set. If *every* call fails with a sign-in / auth error, the session expired — reconnect the AWS MCP connector (Rule 1 box above).
7. **HTTPS cert never issues in Vercel** → the CNAME/A target doesn't match what Vercel asked for, or DNS hasn't propagated — re-check the record against Vercel's exact value and `dig` it.
8. **GitHub PAT pasted into a commit / the front-end** → it's a write-credential; rotate immediately and store only in Secrets Manager ([[lovable-best-practice]] Rule 6a).

---

## Out of scope for this course (not part of the AWS footprint here)

- **Lambda / DynamoDB / SQS / API Gateway / EC2 / VPC** — none of it; all app logic is Next.js API routes + Supabase.
- **Domain registration via Route 53** — the user brings a domain; we only wire DNS records.
- **Per-service least-privilege IAM roles, KMS customer-managed keys + rotation policies, multi-account separation** — course uses one AWS MCP sign-in (the account owner, full access) + the default KMS for Secrets Manager; harden in a prod pass.
- **Routing app-runtime secrets through AWS** — deliberately not done (Rule 2); they live in Vercel env.

When a student asks "shouldn't we put the Stripe key in Secrets Manager too / split the IAM role?" → "Not for this course — app-runtime keys belong in Vercel env (Rule 2), and one AWS MCP sign-in is the teaching-grade floor. Both are production hardening passes."

---

## Cross-references

- [[m0-landing-page]] — connects the AWS MCP (Step 4.2, browser sign-in) and caches the GitHub PAT in `barber-project/github` (Rule 2).
- [[m3-domain]] — the M3 build that creates the Vercel-provided DNS records in Route 53 (Rule 3).
- [[m3-domain-prerequisites]] — the lightweight carryover check that the AWS MCP reaches Route 53 (no new setup).
- [[supabase-best-practice]] — where the app-runtime Supabase keys live (Vercel env), and why Vault isn't used for secrets here.
- [[stripe-go-live]] — Stripe live keys go in Vercel env; update the webhook URL to the custom domain after M3.
