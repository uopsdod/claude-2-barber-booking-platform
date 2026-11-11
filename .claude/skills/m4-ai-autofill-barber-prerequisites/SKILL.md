---
name: m4-ai-autofill-barber-prerequisites
description: One-time setup before Milestone 4 (the egg unit「AI 自動填寫賣家商鋪資訊」) of the barber-booking course — an INTERACTIVE, agent-driven walkthrough with TWO parts. (A) The only NEW runtime wiring: the student has already stored their OpenAI API key in AWS Secrets Manager as `openai-api-key` (us-east-1, done from the student step sheet before this skill runs); the agent verifies it exists and is an `sk-…` secret, then copies it into Vercel env as `OPENAI_API_KEY` (Sensitive, Production, server-only, NEVER `VITE_`-prefixed) and adds `OPENAI_MODEL` (a vision-capable chat model) via the Vercel MCP — never printing the value. (B) A data-readiness carryover check of exactly what M4 consumes: at least one barber with portfolio photos in `barber_photos` (the shop picks which ones the AI reads with a per-photo「AI」checkbox at press time), the public `barber-photos` Storage bucket (thumbnails and public pages read photos from it; the function downloads them server-side and sends them to OpenAI inline), the M1.1 RLS that lets a shop update its own `barbers.intro` + `barber_photos.caption` (the "套用" write path), `SUPABASE_SECRET_KEY` already in Vercel env from M2.1, the `pgvector` extension available, and a clean slate (no `style_refs` table / `match_style_refs` function yet — M4 creates them). Use when the student starts M4 for the first time, says "啟動 M4 前置" / "M4 prerequisites" / "check M4 setup", or when `m4-ai-autofill-barber` / `-checklist` detects a missing OpenAI env var, no photos, or no pgvector.
---

# M4 Prerequisites — interactive setup + carryover check (the agent drives)

**You are the Cowork agent running this skill. Drive the student through it one part at a time** — don't dump the whole thing. For each part: say what's about to happen, **check first**, tell the student only what actually needs doing, confirm it worked, and move on.

> **Source of truth = GitHub `main` + the Supabase project, not a local checkout.** Have the student **`git pull` on `main`** first, and confirm the migration list is current — M4 edits the `/shop` page, so a stale local `src/` would have you editing the wrong file ([[supabase-best-practice]] watch-out #11).

> **What's new in M4 = exactly one outside service: OpenAI.** M0–M2 already wired AWS + Supabase MCP + Vercel + the GitHub PAT (+ Stripe). M4 adds an **OpenAI API key** — used for three things in one function: GPT **looks at the photos**, OpenAI **embeddings** turn text into vectors, and GPT **writes** the intro/captions. One key covers all three. Everything else (the vector database) lives in the **same Supabase project** via the `pgvector` extension — no new database, no Edge Function, no new npm dependency.

> ⚠️ **Where the OpenAI key lives: AWS Secrets Manager is the source, Vercel env is the runtime copy, and it is NEVER `VITE_`.** Same pattern the student already used for their other credentials: they stored the key in **AWS Secrets Manager** as **`openai-api-key`** (us-east-1) from the step sheet, so you can reference it without asking again. The app itself runs on Vercel, so the serverless function reads it from **Vercel env `OPENAI_API_KEY`** (Sensitive). A `VITE_`-prefixed var is inlined into the browser bundle — anyone could read the key and run up the student's OpenAI bill. **Never print, echo or log the key value** — not in chat, not in a commit, not in a file.

## Architecture (what this check unlocks)

M4 adds an **AI draft** to the shop side. On `/shop`, a shop presses「AI 自動填寫」on one barber → the Vercel function `/api/ai/autofill` (server-only, holds `OPENAI_API_KEY` + `SUPABASE_SECRET_KEY`) reads the portfolio photos the shop **ticked for AI** (a per-photo「AI」checkbox, no count limit) from the `barber-photos` bucket → **GPT describes each hairstyle** → the descriptions are turned into **vectors** (OpenAI `text-embedding-3-small`, 1536 dims) → **`match_style_refs`** finds the closest entries in a **hairstyle knowledge base** (`style_refs`, a pgvector table in the same Supabase DB) → **GPT writes** the barber's intro + a caption per photo using the matched hairstyle names → the function returns a **draft**. The shop edits it and presses「套用」, which writes `barbers.intro` + `barber_photos.caption` through the **existing M1.1 RLS**. That's the full RAG loop: **vectorize → similarity search → generate with the retrieved context**.

What this prereq confirms is exactly what that flow consumes:
- **`OPENAI_API_KEY` + `OPENAI_MODEL` in Vercel env** (Part A) → the function can call OpenAI.
- **`SUPABASE_SECRET_KEY` in Vercel env** (from M2.1) → the function reads photos and writes the knowledge-base vectors past RLS.
- **Photos to look at** (`barber_photos` rows; the shop ticks which ones at press time) + **the public bucket** → the draft dialog and public pages can show each photo; the function itself downloads the photos server-side and sends them to OpenAI inline.
- **The M1.1 write policies** on `barbers` + `barber_photos` → the「套用」button can save the draft as the shop.
- **`pgvector` available** + **a clean slate** → M4's migration can create `style_refs` + `match_style_refs`.

## When to load this skill

- The student says "啟動 M4 前置" / "M4 prerequisites" / "check M4 setup".
- `[[m4-ai-autofill-barber]]` (or its checklist) detects a missing `OPENAI_API_KEY`, no photos, or no `vector` extension.

**Opening line to the student (say something like):**
> 「M4 要做的是『AI 自動填寫』：店家按一個按鈕，AI 看你勾選的作品照，參考平台的髮型範本資料庫（這就是 RAG 向量資料庫），幫他寫好自我介紹和每張照片的說明，店家確認後才存檔。開始之前只有一件新的事：你已經把 **OpenAI API 金鑰**存進 AWS Secrets Manager 了，我會確認它在、然後幫你放進 Vercel 的環境變數。其他東西我也一起檢查現在的狀態。」

---

## Part A — The OpenAI key: AWS Secrets Manager → Vercel env (the only new setup)

### A1 — The key is in AWS Secrets Manager and is a real `sk-` secret

Via the AWS MCP (`aws___run_script`):
```python
d = await call_boto3(service_name="secretsmanager", operation_name="DescribeSecret",
                     region_name="us-east-1", params={"SecretId": "openai-api-key"})
v = await call_boto3(service_name="secretsmanager", operation_name="GetSecretValue",
                     region_name="us-east-1", params={"SecretId": "openai-api-key"})
val = v["SecretString"].strip()
result = {"arn": d["ARN"], "starts_with_sk": val.startswith("sk-") or '"sk-' in val,
          "looks_like_json": val.startswith("{")}      # never return the value itself
result
```
(CLI mode: `aws secretsmanager describe-secret --secret-id openai-api-key --region us-east-1`.)
- Found → note the ARN. Then check the value **without printing it** (`GetSecretValue`, as above) and check only two things: the value starts with **`sk-`** (usually `sk-proj-…`), and it is a bare string (or a JSON with one obvious key field — accept either, like the GitHub PAT).
- **Not found** → the student skipped the step sheet's「Set Up OpenAI Credential」part. Tell them (in their language) to go back to that part: top up billing (≥ US$5), create the key, and ask you to store it as `openai-api-key`. Then re-run A1.
- **Value starts with `key_`** (or anything other than `sk-`) → that's the key's **ID**, not the secret, and every call would fail with `401 invalid_api_key`. Ask the student to create a new key, copy the `sk-…` string shown once, and ask you to **update** `openai-api-key` with it.

> **Note for Claude Code:** the secret name/region can drift between installs (the course has seen this with the GitHub PAT). If `openai-api-key` isn't in us-east-1, run `ListSecrets` and match by name (`openai`), then use what you find.

### A2 — Copy it into Vercel env + add the model name (Vercel MCP)

First list the project's env var **names** (`filter_project_envs`, project + team id from earlier milestones — never decrypt):

| Variable | Needed | Where it comes from |
|---|---|---|
| `SUPABASE_SECRET_KEY` | ✅ | M2.1 prerequisite A6 |
| `OPENAI_API_KEY` | ✅ | **this step** — the value of `openai-api-key` |
| `OPENAI_MODEL` | ✅ | **this step** — default `gpt-4o-mini` |

- `SUPABASE_SECRET_KEY` **missing** → blocker: send the student to the M2.1 prerequisite A6, then come back.
- Any **`VITE_OPENAI…`** variable → remove it first (it would ship the key to every browser).
- Create **`OPENAI_API_KEY`** with the value from A1: type **sensitive**, target **production** (`create_project_env`). If it already exists, overwrite it with the A1 value (`edit_project_env`) so Vercel and AWS stay in sync.
- Create **`OPENAI_MODEL`** = `gpt-4o-mini` (or the course's chosen model), target **production**.

> **Note for Claude Code:** `OPENAI_MODEL` must be a chat model that accepts **image input** and supports **`response_format: { type: "json_schema", strict: true }`** (Structured Outputs). Embeddings always use `text-embedding-3-small` — hard-coded in the function on purpose, because the database column is `vector(1536)` and the dimension must match.
>
> If your session's Vercel MCP can't create env vars, fall back to the dashboard: tell the student to add the two variables in **Vercel → Settings → Environment Variables** (key marked **Sensitive**, Production), pasting the same `sk-…` they stored in AWS, then confirm back.

### A3 — Verify the names landed

Re-list the env var names. `OPENAI_API_KEY` (**sensitive**, production) and `OPENAI_MODEL` present, no `VITE_OPENAI…` → ✅.

Env vars only reach a function **after a new deployment**. M4's build pushes code (which redeploys), so no manual redeploy is needed now — but if the key ever changes later, update **both** the AWS secret and the Vercel variable, then **Redeploy**.

**Tell the student:** 「✅ 你的 OpenAI 金鑰在 AWS Secrets Manager（`openai-api-key`），我也把它放進 Vercel 的 `OPENAI_API_KEY`（Sensitive，只有伺服器讀得到），模型設成 `gpt-4o-mini`。這是 M4 唯一需要的新設定。」

---

## Part B — Data-readiness carryover check (Supabase MCP)

All reads run through the Supabase MCP **`execute_sql`**. Refer to tools by their bare names (`list_tables`, `execute_sql`, `apply_migration`, `list_extensions`) — the namespace varies per session.

> **Note for Claude Code:** a single `execute_sql` with several `select`s returns only the **last** result set — run them as separate calls or wrap them in one `json_build_object(...)`. **Apply no migration in this prereq** — M4 owns its schema.

### B1 — There are photos to look at

```sql
select b.id as barber_id, b.name, b.shop_id,
       count(p.id)                                as photos,
       count(p.id) filter (where p.is_featured)   as featured,
       coalesce(length(b.intro), 0)               as intro_len
from public.barbers b
left join public.barber_photos p on p.barber_id = b.id
group by b.id
order by photos desc;
```
- **At least one barber with `photos >= 1`** → ✅ there's something for the AI to look at. Note that barber (you'll test with them).
- `featured` is informational only. 精選 / Featured still just orders the public carousel; the AI reads the photos the student ticks with the「AI」checkbox next to 精選 / Featured under each photo (or「全部選取」) right before pressing the button, with no count limit.
- **No photos anywhere** → have the student upload **3–6 real hairstyle photos** for one barber on `/shop` (M1.1's uploader) then re-run B1. Without photos the「AI 自動填寫」button stays disabled.

> **Note for Claude Code:** good test photos are clear shots of a haircut (side/back views, a fade, a perm, a beard). Random non-hair images will still produce a draft, but the retrieval demo (A3 in the checklist) is much more convincing with real hairstyles.

### B2 — The bucket is public (thumbnails + public pages)

```sql
select id, public from storage.buckets where id = 'barber-photos';
```
- `public = true` → ✅ the draft dialog and `/barbers/:id` can show photos by public URL. (OpenAI does **not** fetch these URLs: the function downloads each photo with the service role and sends it inline, because OpenAI's own download can time out on an uncached photo.)
- `public = false` or no row → M1.1 Step 3a didn't land as designed. Send the student to `[[m1.1-seller-setup]]` Step 3a. (Don't switch to signed URLs here — the whole course treats portfolio photos as public.)

### B3 — The「套用」write path exists (M1.1 RLS)

The draft is saved **by the shop's own session**, through the policies M1.1 created — M4 adds no new write policy for these tables.

```sql
select tablename, policyname, cmd
from pg_policies
where schemaname = 'public' and tablename in ('barbers','barber_photos')
order by tablename, policyname;
```
- `barbers` has an **UPDATE** policy for the owning shop (M1.1: `shops_update_own`), and `barber_photos` has a write policy covering UPDATE (M1.1: `photos_write_own`, `for all`) → ✅.
- Missing → re-apply M1.1's RLS migration (`[[m1.1-seller-setup]]` Step 3) before building M4.

Also confirm the two target columns exist:
```sql
select table_name, column_name from information_schema.columns
where table_schema = 'public'
  and ((table_name = 'barbers' and column_name = 'intro')
    or (table_name = 'barber_photos' and column_name in ('caption','is_featured','storage_path')));
```
Expect **4 rows**.

### B4 — `pgvector` is available + `is_admin()` exists

```sql
select name, default_version, installed_version
from pg_available_extensions where name = 'vector';
```
- A row with a `default_version` → ✅ M4's migration can `create extension vector with schema extensions`. (`installed_version` may be NULL — that's fine; M4 installs it.)
- No row → extremely unlikely on Supabase; tell the student and stop.

```sql
select to_regproc('public.is_admin') as is_admin_fn;
```
- Non-NULL → ✅ (M1.1 created it; M4 doesn't strictly need it, but the checklist uses the admin account to read the knowledge base).

### B5 — Clean slate

```sql
select to_regclass('public.style_refs')      as style_refs,
       to_regproc('public.match_style_refs') as match_style_refs;
```
- Both **NULL** → ✅ clean slate; M4 Step 1 creates them.
- Either exists (a partial earlier run) → read its shape (`style_refs.embedding` must be `vector(1536)`) and tell the student; the M4 migration uses `if not exists` / `create or replace`, so it is safe to re-run.

> **Prereq scope note:** these are **static** checks. Nothing here calls OpenAI — the first real call happens when the student presses the button after the build. The behavioral proofs (draft shape, 403 for another shop, no auto-write, retrieval quality, key not in the bundle) live in **`[[m4-ai-autofill-barber-checklist]]`**.

---

## Verify (all must pass)

- ✅ **OpenAI wired** — `openai-api-key` exists in AWS Secrets Manager and is an `sk-…` secret; `OPENAI_API_KEY` (Sensitive) + `OPENAI_MODEL` exist in Vercel env; no `VITE_OPENAI…` variable; `SUPABASE_SECRET_KEY` is still there (Part A).
- ✅ **Photos to look at** — at least one barber has ≥ 1 photo (B1).
- ✅ **Public bucket** — `barber-photos` is public (B2).
- ✅ **Write path** — M1.1's shop UPDATE policies on `barbers` + `barber_photos` exist, and `intro` / `caption` / `is_featured` / `storage_path` columns are there (B3).
- ✅ **pgvector available + clean slate** — `vector` is in `pg_available_extensions`; `style_refs` / `match_style_refs` don't exist yet (B4, B5).

## Next step

When all five are ✅, tell the student:
「前置檢查通過 ✅ —— OpenAI 金鑰在 AWS Secrets Manager，也已經放進 Vercel（只在伺服器端讀得到）；`<理髮師名稱>` 有 N 張作品照，按按鈕前勾選照片「精選」旁邊的「AI」就能決定要讓 AI 看哪幾張；照片 bucket 是公開的，頁面上看得到照片；店家本來就能改自己理髮師的介紹和照片說明，所以『套用』不用新增權限；Supabase 也可以啟用 pgvector。接下來我會：建一張**髮型範本知識庫**（pgvector 向量表）並放進 24 種常見髮型、寫一支伺服器端的 `/api/ai/autofill`（看圖 → 轉向量 → 找相似範本 → 寫草稿），再在 `/shop` 加上『AI 自動填寫』按鈕和草稿對話框。跟我說『啟動 M4』就開始。」
Then return to the build skill `[[m4-ai-autofill-barber]]` (Step 0/1).

## Reference

- OpenAI API keys: https://platform.openai.com/api-keys
- Vercel environment variables (Sensitive): https://vercel.com/docs/environment-variables/sensitive-environment-variables
- Supabase pgvector: https://supabase.com/docs/guides/database/extensions/pgvector
- `[[m1.1-seller-setup]]` — where `barber_photos`, the public bucket and the shop write policies come from.
- `[[m2.1-buyer-to-admin-payments-prerequisites]]` — where `SUPABASE_SECRET_KEY` was put in Vercel env (A6).
- `[[aws-secrets-best-practice]]` — why app-runtime keys go in Vercel env, not AWS.
- `[[m4-ai-autofill-barber]]` / `[[m4-ai-autofill-barber-checklist]]` — the build this prereq gates, and its verification.
