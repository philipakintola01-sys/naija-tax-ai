# Naija Tax AI 🇳🇬
## 🛠️ Tech Stack

![n8n](https://img.shields.io/badge/n8n-EA4B71?style=for-the-badge&logo=n8n&logoColor=white)
![Supabase](https://img.shields.io/badge/Supabase-3ECF8E?style=for-the-badge&logo=supabase&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/pgvector-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)
![Google Gemini](https://img.shields.io/badge/Google%20Gemini-4285F4?style=for-the-badge&logo=google&logoColor=white)
![LangChain](https://img.shields.io/badge/LangChain-1C3C3C?style=for-the-badge&logo=langchain&logoColor=white)
![Telegram](https://img.shields.io/badge/Telegram-2CA5E0?style=for-the-badge&logo=telegram&logoColor=white)
![Render](https://img.shields.io/badge/Render-46E3B7?style=for-the-badge&logo=render&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white)
![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=for-the-badge&logo=javascript&logoColor=black)
A RAG-powered Nigerian tax AI agent. Users ask tax questions via Telegram and receive cited answers sourced exclusively from official Nigerian tax documents (NTA 2025, NTAA, NRSA, JRBA).

Built with n8n, Supabase pgvector, Google Gemini, and Telegram.

---

## Architecture

```
INGESTION PIPELINE (run once)
Google Drive (.md files)
      ↓
n8n Loop (one file at a time)
      ↓
Download → Extract Text → Split Out → Edit Fields (rename text → pageContent)
      ↓
Supabase Vector Store (insert)
  ├── Embeddings: gemini-embedding-001 (3072 dims)
  ├── Default Data Loader (JSON mode, {{ $json.pageContent }})
  └── Recursive Character Text Splitter (chunk: 1500, overlap: 200)

QUERY PIPELINE (live)
Telegram Message
      ↓
AI Agent (Gemini 2.5 Flash)
  ├── Tool: Supabase Vector Store (retrieve-as-tool, Top K: 15)
  │     └── Embeddings: gemini-embedding-001
  └── Memory: Simple Memory (session = chat.id)
      ↓
Telegram Reply
```

---

## Stack

| Component | Service |
|---|---|
| Workflow automation | n8n self-hosted (Render) |
| Vector database | Supabase pgvector |
| Embeddings | Google Gemini (gemini-embedding-001) |
| LLM | Google Gemini 2.5 Flash |
| Document storage | Google Drive |
| Bot interface | Telegram |
| Document parsing | Docling (local CLI) |

---

## Prerequisites

- n8n self-hosted on Render (or any platform)
- Supabase project (free tier works)
- Google API key from [aistudio.google.com](https://aistudio.google.com)
- Google Drive OAuth2 credentials
- Telegram bot token from @BotFather

---

## Step 1 — Supabase Setup

### Create documents table

Go to **Supabase → SQL Editor** and run:

```sql
create extension if not exists vector;

create table documents (
  id bigserial primary key,
  content text,
  metadata jsonb,
  embedding vector(3072)
);
```

### Create match_documents function

```sql
create or replace function match_documents (
  query_embedding vector(3072),
  match_count int default 5,
  filter jsonb default '{}'
)
returns table (
  id bigint,
  content text,
  metadata jsonb,
  similarity float
)
language sql stable
as $$
  select
    id, content, metadata,
    1 - (embedding <=> query_embedding) as similarity
  from documents
  where metadata @> filter
  order by embedding <=> query_embedding
  limit match_count;
$$;
```

> **Critical:** The `filter jsonb default '{}'` parameter is required. n8n's LangChain integration always passes this parameter — without it the query pipeline throws a PGRST202 schema cache error.

### Get credentials

- **Settings → Data API → Project URL** → copy (format: `https://xxxx.supabase.co`)
- **Settings → Data API → Project API Keys → service_role** → copy (NOT the anon key)

---

## Step 2 — n8n on Render

### Required environment variables

Set these before first deploy. Never change them after:

```
N8N_ENCRYPTION_KEY=your_permanent_random_string
DB_TYPE=postgresdb
DB_POSTGRESDB_HOST=your_postgres_host
DB_POSTGRESDB_PORT=5432
DB_POSTGRESDB_DATABASE=n8n
DB_POSTGRESDB_USER=postgres
DB_POSTGRESDB_PASSWORD=your_password
```

> **Critical:** `N8N_ENCRYPTION_KEY` must be permanent. If Render redeploys without it, a new key is generated and all saved credentials become unreadable ("bad decrypt" error).

### Credentials to add in n8n

1. **Google Drive OAuth2** — for document access
2. **Supabase API** — Project URL + service_role key
3. **Google Gemini(PaLM) API** — key from aistudio.google.com
   - Use **two separate keys** — one for ingestion, one for queries
   - Free tier limit is 1,000 embedding requests/day per key

---

## Step 3 — Document Preparation

Convert PDFs to markdown using Docling:

```bash
docling --input-path ./pdfs --output-dir ./output --output-format markdown
```

Upload the `.md` files to a Google Drive folder.

---

## Step 4 — Import the Workflow

1. In n8n → **Workflows → Import**
2. Upload `NAIJA_AI_TAX_clean.json`
3. Re-connect all credentials (Google Drive, Supabase, Gemini x2, Telegram)
4. The credential IDs in the JSON are instance-specific and will not transfer — you must reselect them

---

## Step 5 — Run Ingestion

1. Make sure the Supabase table exists (Step 1)
2. Click **Execute Workflow** on the Manual Trigger
3. With 8 documents expect ~1,500 chunks, ~15 minutes on Render free tier
4. Verify completion:

```sql
select count(*) from documents;
-- Should be 1000+
```

5. Clean markdown artifacts after ingestion:

```sql
update documents 
set content = trim(regexp_replace(content, '<!--.*?-->', '', 'g'))
where content like '%<!--%';
```

---

## Step 6 — Activate and Test

1. Toggle workflow **Active** in n8n
2. Message your Telegram bot:

```
What is the VAT rate in Nigeria under the 2025 tax reform?
```

Expected: VAT maintained at 7.5%, zero-rated essential goods, citation to NTA 2025.

---

## Node Configuration Reference

### AI Agent

| Field | Value |
|---|---|
| Prompt | `={{ $('Telegram Trigger').item.json.message.text }}` |
| System Message | See below |

**System Message:**
```
You are a Nigerian tax expert AI assistant. Answer questions using ONLY the context retrieved from official Nigerian tax documents. Always cite the relevant act and section number. If the answer is not in the context, say "I don't have enough information on that topic in my knowledge base."
```

### Supabase Vector Store1 (retrieve tool)

| Field | Value |
|---|---|
| Mode | retrieve-as-tool |
| Top K | 15 |
| Tool Description | `Use this tool to search official Nigerian tax documents including the Nigeria Tax Act 2025, Nigeria Tax Administration Act, Nigeria Revenue Service Act, and expert analysis. Call this tool for EVERY tax-related question before answering.` |

> The tool description is what tells the agent WHEN to search. Wrong description = agent answers from its own knowledge without ever touching Supabase.

### Simple Memory

| Field | Value |
|---|---|
| Session Key Type | Custom Key |
| Session Key | `={{ $('Telegram Trigger').item.json.message.chat.id }}` |

> Must be dynamic per user. Static key = all users share one memory = agent answers from conversation history instead of searching Supabase.

### Send a text message (Telegram)

| Field | Value |
|---|---|
| Chat ID | `={{ $('Telegram Trigger').item.json.message.chat.id }}` |
| Text | `={{ $json.output }}` |
| Append Attribution | OFF |

---

## Errors Encountered and Fixes

### Error 1 — Split Out wrong field name
**Error:** Nothing passing downstream after Extract from File  
**Cause:** "Fields To Split Out" was set to `data` but Extract from File outputs `text`  
**Fix:** Changed to `text`

---

### Error 2 — Supabase table wrong dimensions
**Error:** `Error inserting: expected 768 dimensions, not 3072`  
**Cause:** Table created with `vector(768)` but `gemini-embedding-001` outputs 3072 dimensions  
**Fix:** Dropped and recreated table with `vector(3072)`

---

### Error 3 — Default Data Loader wrong mode
**Error:** `vector must have at least 1 dimension 400 Bad Request` — Gemini receiving empty text  
**Cause:** Default Data Loader set to Binary then JSON "Load All Input Data" mode. LangChain's Default Data Loader in JSON mode expects a field called `pageContent` not `text`  
**Fix:** Added Edit Fields node to rename `text` → `pageContent`. Set Data Loader to JSON mode, Load Specific Data, expression `{{ $json.pageContent }}`

---

### Error 4 — Include Other Input Fields was OFF
**Error:** pageContent mapped but downstream nodes received incomplete data  
**Cause:** Edit Fields node "Include Other Input Fields" toggle was OFF — only `pageContent` passed through  
**Fix:** Toggled ON

---

### Error 5 — googlePalmApi credential silently returns empty embeddings
**Error:** `vector must have at least 1 dimension` — Gemini running successfully but returning `[]` for every chunk  
**Cause:** `text-embedding-004` model not available on this API key. Returned empty arrays with no error instead of failing loudly  
**Fix:** Switched to `models/gemini-embedding-001` which IS available on the key and outputs 3072 dimensions  
**Debug method:** Inspected `data.ai_embedding[0][0].json.response` in execution data — showed all `[]` arrays

---

### Error 6 — N8N_ENCRYPTION_KEY changed on Render redeploy
**Error:** `Credentials could not be decrypted — bad decrypt`  
**Cause:** Render redeployed n8n (2.4.8 → 2.12.3) without a fixed encryption key. New random key generated, all saved credentials unreadable  
**Fix:** Added permanent `N8N_ENCRYPTION_KEY` env var on Render. Deleted and re-entered all credentials  
**Prevention:** Always set this before first run. Never change it.

---

### Error 7 — match_documents missing filter parameter
**Error:** `PGRST202 Could not find function public.match_documents(filter, match_count, query_embedding)`  
**Cause:** n8n's LangChain integration passes a `filter` parameter that the original function didn't have  
**Fix:** Recreated function with `filter jsonb default '{}'` parameter

---

### Error 8 — Rate limit 429 during ingestion
**Error:** `429 Too Many Requests — quota exceeded for embed_content_free_tier_requests, limit 1000`  
**Cause:** All 1,000 free daily embedding requests consumed by repeated ingestion runs during debugging  
**Fix:** Created second API key from aistudio.google.com for query pipeline. Reduced embedding batch size to 5. Added Wait node between loop iterations.

---

### Error 9 — Agent not using Supabase tool
**Error:** Bot answering from Gemini's own knowledge, never calling Supabase  
**Cause 1:** Tool description was the system prompt pasted by mistake — agent didn't know what the tool was for  
**Cause 2:** Prompt field had system instructions instead of `{{ $('Telegram Trigger').item.json.message.text }}`  
**Fix:** Corrected both fields (see Node Configuration Reference above)

---

### Error 10 — All users sharing one memory
**Error:** Agent answering from old conversation history instead of searching Supabase  
**Cause:** Simple Memory node used static session key `"go"` — all users shared one conversation  
**Fix:** Changed to `={{ $('Telegram Trigger').item.json.message.chat.id }}` — each user gets isolated memory

---

### Error 11 — Telegram webhook stuck on old message
**Error:** New Telegram messages not appearing in executions, same old message replaying  
**Cause:** After redeployment the webhook URL changed but Telegram wasn't notified  
**Fix:** Deleted and re-added Telegram Trigger node. Re-activating workflow re-registers the webhook with Telegram

---

### Error 12 — VAT question returning "no information"
**Error:** Bot says "I don't have enough information" for VAT rate despite data existing in Supabase  
**Cause 1:** Chunks had `<!-- image -->` tags polluting the content, lowering similarity scores  
**Cause 2:** Top K was 4 — correct chunk ranked below retrieval cutoff  
**Fix 1:** SQL cleanup — `update documents set content = trim(regexp_replace(content, '<!--.*?-->', '', 'g')) where content like '%<!--%'`  
**Fix 2:** Increased Top K to 15

---

## Known Limitations

- Gemini free tier: 1,000 embedding requests/day — shared between ingestion and queries
- Render free tier: spins down after 15 min inactivity, ~60s cold start on first message
- Chunk quality depends on Docling markdown conversion — image-heavy PDFs produce noisy chunks with `<!-- image -->` artifacts

---

## Roadmap

- [ ] Error handler workflow with Telegram alert on failure
- [ ] Web frontend on Netlify
- [ ] Add source filename to chunk metadata for better citations
- [ ] Upgrade to paid Gemini API for faster responses
- [ ] Support follow-up questions with richer conversation context
- [ ] SaaS monetization layer
