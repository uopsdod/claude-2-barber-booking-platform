---
name: m4-ai-autofill-barber-checklist
description: 抽成制理髮師預約平台 Milestone 4 verification (the egg unit「AI 自動填寫賣家商鋪資訊」) — checks the RAG feature is real and safe: pgvector lives in the `extensions` schema, `style_refs` has an HNSW index, RLS with public SELECT and NO client write policy, all 24 references carry a 1536-dim embedding, retrieval is sensible (a reference's nearest neighbours are related hairstyles), `match_style_refs` is service-role only; `/api/ai/autofill` rejects no token (401) and another shop's barber (403), returns a draft whose captions only reference the barber's own photos and whose ref tags only name retrieved references, and writes NOTHING until the shop presses「套用」; after「套用」the intro + captions appear on `/barbers/:id`; and `OPENAI_API_KEY` is Sensitive, server-only, never `VITE_`-prefixed and absent from the browser bundle. Use when the student says "驗收 M4", "check M4", "M4 done?", "AI 自動填寫對不對", or after `m4-ai-autofill-barber` completes Step 5.
---

# M4 — AI 自動填寫賣家商鋪資訊 Checklist

## What this skill does

Verifies the student actually completed M4 — not just that a button exists. The high-risk items are **secret handling** (is the OpenAI key really server-only?), **trust boundaries** (can someone run the AI on another shop's barber, does the AI write data on its own, does the server validate model output?), and **RAG correctness** (is there really a vector knowledge base, is it fully embedded, does retrieval return sensible neighbours?). Run every check and report pass/fail per item, then emit a verdict.

**Run this AFTER `m4-ai-autofill-barber` Step 5, or whenever the student claims M4 is done.**

## Execution mode: Cowork vs CLI (read this first)

| Section | Tool |
|---|---|
| A — Knowledge base + retrieval | **Supabase MCP `execute_sql`** |
| B — `/api/ai/autofill` behaviour | the **live app in the browser** (student) + Supabase MCP reads before/after + Vercel MCP runtime logs |
| C — Secret handling | **Vercel MCP** (env var names/types, deployment files) + URL-fetch MCP on the live bundle |
| D — 套用 end-to-end | the live app + Supabase MCP |

> **⚠️ The Supabase MCP runs as a privileged role and bypasses RLS**, so it can call the service-role-only `match_style_refs` and read every row. Use it for **structure and data** checks. The **behavioural** proofs (draft-only, 套用) are done in the **live app** — the student presses the button while you read the database before and after. The 401/403 guard (B2) is yours: verify it in the deployed source, no second account needed. The sandbox `curl` to the live URL is proxy-blocked (a `000/403` is not "down"); use the Vercel MCP / URL-fetch MCP.

## How to run

The student invokes this directly (e.g. `驗收 M4`). You **actively run** each check.

### Step 1: Collect (one message)

Don't ask the student for anything you can look up. Reuse the **live Vercel URL** from earlier milestones (Vercel MCP `get_project`), and pick the test barber yourself — the one with the most photos (query below). Tell the student up front that their only part is **one press**: sign in as that barber's shop, tick「AI」on a few photos (or「全部選取」), press「AI 自動填寫」, wait for the draft, tell you, then press「套用」when you say so.

Then confirm the newest deployment is **READY** (Vercel MCP `list_deployments`) and note the test barber's id:
```sql
select b.id, b.name, b.shop_id, b.intro,
       count(p.id) as photos, count(p.id) filter (where p.is_featured) as featured
from public.barbers b left join public.barber_photos p on p.barber_id = b.id
group by b.id
order by photos desc
limit 1;
```

### Step 2: Run the checklist

#### Section A — Knowledge base + retrieval (Supabase MCP)

- **A1 pgvector in `extensions` + HNSW index.**
  ```sql
  select json_build_object(
    'ext_schema', (select n.nspname from pg_extension e join pg_namespace n on n.oid = e.extnamespace where e.extname = 'vector'),
    'hnsw',       (select indexdef from pg_indexes where schemaname = 'public' and tablename = 'style_refs' and indexdef ilike '%hnsw%'),
    'dims',       (select format_type(atttypid, atttypmod) from pg_attribute where attrelid = 'public.style_refs'::regclass and attname = 'embedding')
  );
  ```
  Expect `ext_schema = extensions`, an `hnsw … vector_cosine_ops` indexdef, `dims = vector(1536)` (shown as `extensions.vector(1536)` or `vector(1536)`). *Recovery:* M4 Step 1.

- **A2 Every reference is embedded.**
  ```sql
  select count(*) as refs, count(embedding) as embedded,
         count(*) filter (where category = 'cut')   as cut,
         count(*) filter (where category = 'color') as color,
         count(*) filter (where category = 'perm')  as perm,
         count(*) filter (where category = 'beard') as beard
  from public.style_refs;
  ```
  Expect `refs ≥ 24` and `embedded = refs`. `embedded = 0` means the function never ran successfully — check the Vercel runtime logs (`invalid_api_key`, `insufficient_quota`, unsupported `OPENAI_MODEL`). *Recovery:* fix the env var, redeploy, press the button once.

- **A3 Retrieval is sensible.**
  ```sql
  select r.name as query, m.name as neighbour, round(m.similarity::numeric, 3) as similarity
  from public.style_refs r,
       lateral public.match_style_refs(r.embedding, 3) m
  where r.name in ('油頭 Slick Back', '霧灰色 Ash Gray', '鬍子修整 Beard Trim')
  order by r.name, m.similarity desc;
  ```
  Expect each query to return **itself first (≈ 1.000)**, and its other neighbours to be from the **same family** (油頭 → 側分油頭／飛機頭; 霧灰色 → 漂髮／特殊色／挑染; 鬍子修整 → 鬍型雕刻／熱毛巾刮鬍). ⚠️ if one neighbour is off-family (descriptions can overlap) — note it, it's not a failure. ❌ if similarities are all ~equal or random, which means vectors were written wrong (e.g. the same vector for every row).

- **A4 RLS + function grants.**
  ```sql
  select json_build_object(
    'rls_on',          (select relrowsecurity from pg_class where oid = 'public.style_refs'::regclass),
    'policies',        (select json_agg(policyname || ':' || cmd) from pg_policies where schemaname = 'public' and tablename = 'style_refs'),
    'anon_match',      has_function_privilege('anon',          'public.match_style_refs(extensions.vector,int)', 'execute'),
    'auth_match',      has_function_privilege('authenticated', 'public.match_style_refs(extensions.vector,int)', 'execute'),
    'service_match',   has_function_privilege('service_role',  'public.match_style_refs(extensions.vector,int)', 'execute')
  );
  ```
  Expect `rls_on = true`, policies = **only** a SELECT policy (no INSERT/UPDATE/DELETE/ALL), `anon_match = false`, `auth_match = false`, `service_match = true`. Then `get_advisors` (security): no `extension_in_public`, no `rls_disabled_in_public` for `style_refs`. *Recovery:* M4 Step 1 (revoke from `public, anon, authenticated`).

#### Section B — `/api/ai/autofill` behaviour

- **B1 The function is deployed.** Use the URL-fetch MCP to hit `GET <url>/api/ai/autofill` → expect **405** (deployed, POST only).

- **B2 Rejects another shop's barber (403) — the decisive trust test. You (the agent) do this, not the student.** The function uses the service role (bypasses RLS), so this check is the only thing stopping one shop from running the AI on another shop's barber. Read the **deployed** handler (Vercel MCP `get_deployment_file_contents`, or the repo at the deployed commit) and confirm, in this order, before any OpenAI call:
  1. no / invalid `Authorization` token → `401`;
  2. the barber is loaded by `barber_id` → `404` if missing;
  3. `barber.shop_id !== auth.user.id` → **`403`**.
  If your session can send a POST to the live URL (some sandboxes block it — a `000/403` from the proxy is not the app's answer), also prove it live: `POST /api/ai/autofill` with no token → expect `401`. A missing ownership check is a ❌ blocker. *Recovery:* M4 Step 3.

- **B3 + B4 — one press: the draft is right, and nothing is written until「套用」.** This is the student's only action in the checklist.
  1. You read the current state:
     ```sql
     select b.intro, json_agg(p.caption order by p.id) as captions
     from public.barbers b join public.barber_photos p on p.barber_id = b.id
     where b.id = '<test barber id>' group by b.id;
     ```
  2. The student (signed in as the owning shop) ticks「AI」on some photos, presses「AI 自動填寫」and tells you when the draft dialog is showing. Ask them to read you the ref tags (or send a screenshot).
  3. **B3:** the dialog shows an editable intro, one caption row per **ticked** photo with a thumbnail (exactly the ticked ones, no 6-photo cap), ref tags, and the "AI 參考了這些髮型範本" line. Every ref tag must be a real `style_refs.name`. Also confirm in the source that the server only loads the requested `photo_ids` that belong to this barber (`.eq("barber_id", …).in("id", photo_ids)`), filters captions to those photo ids, ref ids to the retrieved set, and caps the intro length. *Recovery:* Step 7 of the function.
  4. **B4:** with the dialog still open, re-run the query from 1 → **identical** result. ❌ if the intro or any caption already changed (the API is writing on its own).
  5. Now tell the student to press **「套用」** — that is D1.

- **B5 Errors are generic to the browser, detailed in logs.** Check the function returns a generic message on failure (`AI 服務暫時無法使用…`) and logs the detail with `console.error` — it must never echo the OpenAI response body to the client. (Structural check in the source; optionally confirm a past error in `get_runtime_logs`.)

- **B6 Photos are sent inline, not by URL.** In the source, the vision call's `image_url.url` values come from `photoDataUrl()` (a `data:` URL built from a service-role Storage download), not from a public Storage URL. ❌ if it passes `…/storage/v1/object/public/…` to OpenAI: the first press on uncached photos intermittently fails with `invalid_image_url — Unable to download content from the provided URL before the timeout` (a retry "fixes" it). If the student reports「有時候要按兩次」, search `get_runtime_logs` for `autofill failed`. *Recovery:* M4 Step 3 (`photoDataUrl`).

#### Section C — Secret handling

- **C1 Env vars are right.** Vercel MCP, names/types only: `OPENAI_API_KEY` present with type **sensitive** (or encrypted), `OPENAI_MODEL` present, **no** variable whose name starts with `VITE_` and contains `OPENAI`. Never print values.

- **C2 The key is not in the browser bundle.** With the URL-fetch MCP, fetch the live `index.html`, find the `/assets/*.js` bundle(s) it references, fetch them, and search for `sk-`, `OPENAI_API_KEY` and `api.openai.com`. Expect **no matches**. Also `grep -rn "OPENAI\|openai.com" src/` in the repo → no matches (only `api/` may mention OpenAI).

- **C3 OpenAI is only called from the server.** In the repo, `api/_openai.ts` is the only file that calls `api.openai.com`, and nothing under `src/` imports from `api/`.

#### Section D — 套用 end-to-end

- **D1 「套用」saves through RLS and is public.** After the student presses「套用」(B3 + B4 step 5), re-run the B4 query → `intro` now equals the draft (including any edit) and the captions are filled. Then confirm the public read path yourself: the `barbers` / `barber_photos` SELECT policies are `using (true)` (M1.1), so `/barbers/<id>` shows the new intro to anyone. Optionally ask the student to glance at that page. *Recovery:* M4 Step 4 (writes via the browser Supabase client; the M1.1 policies `shops_update_own` / `photos_write_own` must exist).

- **D2 The write path is the shop's own session, not a new server endpoint.** Structural: the「套用」handler uses the browser Supabase client `update` on `barbers` and `barber_photos`; there is no `/api/…` route that writes `intro`/`caption`, and no new RLS policy was added for these tables in M4's migrations (`list_migrations` shows only `m4_style_refs` and `m4_style_refs_seed`).

## Reporting

Emit a table:

| Check | Status | Notes |
|---|---|---|
| A1 pgvector in `extensions` + HNSW + vector(1536) | ✅ / ❌ | |
| A2 all references embedded | ✅ / ❌ | embedded = refs |
| A3 retrieval sensible (self first, same-family neighbours) | ✅ / ⚠️ / ❌ | |
| A4 RLS read-only for clients; `match_style_refs` service-role only; no advisor ERROR | ✅ / ❌ | |
| B1 deployed (GET → 405) | ✅ / ❌ | |
| B2 401 no token / 403 another shop's barber (deployed source) | ✅ / ❌ | **the decisive test** — agent-run |
| B3 draft shape + server-side validation of ids | ✅ / ❌ | |
| B4 nothing written until 套用 | ✅ / ❌ | unchanged while the draft is open |
| B5 generic errors to the browser, details in logs | ✅ / ❌ | |
| B6 photos sent inline (no OpenAI URL fetch) | ✅ / ❌ | prevents the「要按兩次」timeout |
| C1 `OPENAI_API_KEY` Sensitive, no `VITE_OPENAI…` | ✅ / ❌ | |
| C2 key / OpenAI host not in the browser bundle | ✅ / ❌ | |
| C3 OpenAI called only from `api/` | ✅ / ❌ | |
| D1 套用 saves and shows on `/barbers/:id` | ✅ / ❌ | |
| D2 write path = shop session + M1.1 RLS (no new endpoint/policy) | ✅ / ❌ | |

**Verdict:**
- All ✅ (⚠️ allowed on A3) → 「M4 驗收通過 ✅ —— 髮型範本知識庫（pgvector）24 筆都有向量、相似度檢索找得到同類髮型；`/api/ai/autofill` 只讓擁有這位理髮師的店家使用（別人呼叫會被 403 擋下）、會在伺服器端檢查 AI 的輸出、而且只回傳草稿，草稿顯示時資料完全不變、按『套用』才用店家自己的權限存檔並出現在公開頁面；OpenAI 金鑰只在 Vercel 伺服器端，前端打包檔裡找不到。RAG 三步（向量化 → 檢索 → 生成）都做到了，整個課程完成 🎉」
- Any ❌ → list the failed items with their recovery step and tell the student to fix and re-run `驗收 M4`. Common failures: the function was never run successfully so nothing is embedded (A2 — usually the OpenAI key/quota/model), `match_style_refs` still executable by `authenticated` because only `anon` was revoked (A4), the ownership check is missing (B2), the API writes `intro` itself (B4), photos passed to OpenAI by URL so the first press times out (B6), or the key was added as `VITE_OPENAI_API_KEY` (C1/C2).
