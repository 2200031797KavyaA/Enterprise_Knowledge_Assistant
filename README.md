# Enterprise Knowledge Assistant

A RAG-powered internal assistant that answers employee questions about HR, IT, and Finance policy - grounded in your actual company documents, with sources cited on every answer.

## Problem Statement

Employees waste time hunting through scattered HR policies, IT procedures, and handbooks for simple answers (leave balance, reimbursement rules, password resets). This assistant ingests those documents, retrieves the passages that actually answer a question, and generates a cited, grounded response instead of a generic or hallucinated one.

## Solution

A full-stack app: employees upload policy documents once, then ask questions in a chat interface. Every answer shows exactly which document and section it came from, and a developer/debug view exposes the entire retrieval pipeline for a technical audience.

## Features

- Document ingestion for PDF, DOCX, TXT, and MD, with structure-aware chunking (splits on headings first, then by size with overlap)
- Hybrid retrieval: BM25 keyword search + NVIDIA semantic embeddings, fused with Reciprocal Rank Fusion (RRF)
- Second-stage reranking via NVIDIA's cross-encoder reranking model
- Vector storage and semantic search via **ChromaDB**
- Grounded generation via a **Key Gateway** chat model (OpenAI-compatible), with inline `[1]` citations
- Source cards on every answer - expandable to see the exact excerpt used
- Multi-turn conversation memory, per-conversation, persisted in SQLite
- A full RAG debug/trace panel: original query → improved query → every retrieved candidate with its BM25/vector/rerank scores → final context → prompt preview → which backend answered
- Feedback (thumbs up/down) on any answer
- **Works with zero external credentials**: every AI stage (chat, embeddings, reranking) has a transparent local fallback, so the whole pipeline is demoable before either API key exists

## Additional Features (beyond the brief)

- Honest backend-switching: the UI and debug trace always show which backend actually answered (`key_gateway`/`nvidia` vs `local_fallback`) rather than pretending
- Weighted RRF: when running on the local fallback embedding (not semantically meaningful), the fusion automatically leans on BM25 more heavily rather than letting noisy vector scores win
- Document manager page with per-chunk preview, so you can see exactly how a document was split before it's ever queried
- Drag-and-drop upload with category tagging (HR / IT / Finance / General)

## Technology Stack

**Frontend:** React (Vite), Tailwind CSS, lucide-react
**Backend:** FastAPI, Pydantic, SQLAlchemy
**Database:** SQLite (documents, chunks, conversations, messages, feedback)
**Vector Store:** ChromaDB (persistent, local)
**AI - split across two providers:**
- **Key Gateway** (OpenAI-compatible endpoint you provide) → chat/answer generation + query rewriting
- **NVIDIA NIM** (`build.nvidia.com` / `integrate.api.nvidia.com`) → NV-Embed embeddings + NV-RerankQA reranking

## Architecture

```
React Frontend
      │  REST (fetch)
      ▼
FastAPI Routes  (documents / chat / conversations / feedback / system)
      │
      ▼
Service Layer  (document_service, chat_service)
      │                              │
      ▼                              ▼
Repositories (SQLite)         RAG Pipeline
                                     │
        ┌────────────┬──────────────┼───────────────┬─────────────┐
        ▼            ▼              ▼                ▼             ▼
  Query Improver   BM25       ChromaDB Vector    NVIDIA Rerank   Context
  (Key Gateway     (SQLite    Search (NVIDIA      (cross-        Builder
  chat, or rule-   chunk      embeddings, or      encoder, or    (dedup,
  based fallback)  text)      local hash          lexical        threshold,
                               fallback)           fallback)      limit)
                                     │
                                     ▼
                              Prompt Template
                                     │
                                     ▼
                        Key Gateway Chat Model (or
                        extractive local fallback)
                                     │
                                     ▼
                          Grounded Answer + Sources
```

## RAG Architecture

1. **Ingest** - PDF/DOCX/TXT/MD → text extraction → structure-aware chunking (headings first, then sliding window with overlap for long sections)
2. **Embed** - each chunk embedded via NVIDIA's `nv-embedqa-e5-v5` (asymmetric query/passage embeddings) and upserted into ChromaDB, keyed by the SQLite chunk id
3. **Retrieve** - on a query: BM25 over chunk text (stopword-filtered) + ChromaDB cosine similarity search, fused with weighted Reciprocal Rank Fusion
4. **Rerank** - NVIDIA's `nv-rerankqa-mistral-4b-v3` cross-encoder scores each candidate against the actual query (a real second-stage reranker, not just re-sorting the fusion score)
5. **Context build** - relevance threshold filter, near-duplicate removal, limited to the top N chunks
6. **Generate** - context + conversation history + question assembled via a dedicated prompt template, sent to your Key Gateway's chat model, instructed to answer only from context and cite sources inline
7. **Respond** - answer + source cards (document, section, similarity, excerpt) + full debug trace

### Local fallback mode (no credentials configured)

Every AI-dependent stage degrades gracefully and *transparently*, and the two providers fall back independently of each other:
- **Chat generation & query rewriting** (Key Gateway) → extractive answer (returns the top retrieved passage directly) / rule-based abbreviation expansion
- **Embeddings** (NVIDIA) → deterministic hashing vector (weak semantically, but stable)
- **Reranking** (NVIDIA) → lexical term-overlap scoring

The debug panel and `/api/system/status` always report which backend is actually live for each stage, so nothing is silently faked. If you have your NVIDIA key but not the Key Gateway yet (or vice versa), the app runs correctly with a mixed pipeline.

## Folder Structure

```
eka/
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── api/routes/        # documents, chat, conversations, feedback, system
│   │   ├── core/config.py     # env-driven settings
│   │   ├── models/            # SQLAlchemy models
│   │   ├── schemas/           # Pydantic request/response schemas
│   │   ├── services/          # document_service, chat_service (orchestration)
│   │   ├── repositories/      # DB access
│   │   ├── db/database.py
│   │   ├── rag/                # loaders, chunking, embeddings, vector_store,
│   │   │                        # retriever (BM25+RRF), context_builder, query_improver
│   │   ├── llm/
│   │   │   ├── gateway_client.py   # Key Gateway - chat only
│   │   │   └── nvidia_client.py    # NVIDIA - embeddings + reranking only
│   │   └── prompts/templates.py
│   ├── tests/
│   ├── data/sample_docs/       # generated sample HR/IT/Finance documents
│   ├── seed.py
│   ├── requirements.txt
│   ├── .env                    # your actual credentials (gitignored)
│   └── .env.example
└── frontend/
    ├── src/
    │   ├── components/         # Sidebar, MessageBubble, SourceCard, DebugPanel, ...
    │   ├── pages/               # ChatPage, DocumentsPage
    │   ├── services/            # apiClient, documentService, chatService
    │   └── context/ToastContext.jsx
    └── tailwind.config.js
```

## Database Design

- `users` - name, email, password hash, role (`USER`/`ADMIN`), active flag, timestamps
- `documents` - filename, type, category, status, char/chunk counts, uploaded_by
- `document_chunks` - text + section + chunk_index (vectors live in ChromaDB, keyed by this row's id)
- `conversations` - per-user (`user_id`), with soft-delete fields (`is_deleted`, `deleted_at`, `deleted_by`)
- `messages` - sources + full debug trace stored per assistant message (trace only ever returned to admins)
- `feedback` - thumbs up/down per message
- `restore_requests` - user-submitted requests to restore a soft-deleted conversation, with admin approve/reject + reason
- `audit_logs` - actor, action, target, status, timestamp for every security-relevant action

## Authentication & Authorization

- **JWT-based auth.** `/api/auth/login` and `/api/auth/register` issue a bearer token (`sub`=user id, `role`, `exp`). The frontend sends it as `Authorization: Bearer <token>` on every request; nothing is trusted from the client body.
- **Passwords** are hashed with bcrypt (passlib) - never stored, logged, or returned in plain text. Strength is enforced server-side regardless of what the frontend does.
- **Roles**: `USER` and `ADMIN`. `require_admin`/`get_current_user` FastAPI dependencies gate every route - a non-admin hitting an admin path gets a 403 before any handler code runs.
- **Ownership**: every conversation query is scoped to `current_user.id` derived from the JWT. A user can never read, delete, or restore another user's conversation, even by guessing/forging an ID - verified by an automated cross-user test (see Testing below).
- **Soft delete + restore-request workflow**: users can delete their own conversations (soft delete only) and submit a restore request with a reason; only an admin can approve (which restores it) or reject it. Admins can also restore or permanently delete any conversation directly.
- **Document management is admin-only.** Regular users never see the document inventory - they only see per-answer source attribution (filename, section, relevance) via `/api/chat`. The RAG debug trace and any raw chunk excerpt are also stripped from the response for non-admins.

## API Endpoints

```
POST   /api/auth/register             { name, email, password, confirm_password }
POST   /api/auth/login                { email, password }
GET    /api/auth/me
PATCH  /api/auth/me                   { name }
POST   /api/auth/change-password      { current_password, new_password, confirm_new_password }
GET    /api/auth/password-requirements?password=...

GET    /api/documents                 [ADMIN]
POST   /api/documents                 [ADMIN] (multipart: file, category)
GET    /api/documents/{id}/chunks     [ADMIN]
DELETE /api/documents/{id}            [ADMIN]

POST   /api/chat                      { message, conversation_id? }  - debug trace admin-only
GET    /api/conversations             - current user's active conversations only
GET    /api/conversations/deleted     - current user's own soft-deleted conversations
GET    /api/conversations/{id}/messages
DELETE /api/conversations/{id}        - soft delete
POST   /api/conversations/{id}/restore-requests   { reason }
GET    /api/conversations/restore-requests/mine

POST   /api/feedback                  { message_id, rating, comment? }
GET    /api/system/status             which backends are actually live

GET    /api/admin/users                              [ADMIN]
PATCH  /api/admin/users/{id}/active   { is_active }   [ADMIN]
PATCH  /api/admin/users/{id}/role     { role }        [ADMIN]
GET    /api/admin/conversations                       [ADMIN] all users, incl. deleted
POST   /api/admin/conversations/{id}/restore          [ADMIN]
DELETE /api/admin/conversations/{id}/permanent        [ADMIN]
GET    /api/admin/restore-requests                    [ADMIN]
POST   /api/admin/restore-requests/{id}/resolve       [ADMIN] { approve, resolution_reason? }
GET    /api/admin/audit-logs                          [ADMIN]
GET    /api/admin/analytics/overview                  [ADMIN] real counts, never fabricated
```

## Environment Variables

See `backend/.env.example` and `backend/.env`. Key ones:

```env
# Auth
JWT_SECRET_KEY=                # generate: python -c "import secrets; print(secrets.token_hex(32))"
JWT_ALGORITHM=HS256
JWT_EXPIRE_MINUTES=720

# Chat / answer generation
KEY_GATEWAY_URL=
KEY_GATEWAY_API_KEY=
KEY_GATEWAY_CHAT_MODEL=gpt-4o-mini

# Embeddings + reranking
NVIDIA_API_KEY=                # from build.nvidia.com - leave blank to run in local fallback mode
NVIDIA_BASE_URL=https://integrate.api.nvidia.com/v1
NVIDIA_RERANK_URL=https://ai.api.nvidia.com/v1/retrieval/nvidia/nv-rerankqa-mistral-4b-v3/reranking
NVIDIA_EMBEDDING_MODEL=nvidia/nv-embedqa-e5-v5
NVIDIA_RERANK_MODEL=nvidia/nv-rerankqa-mistral-4b-v3

DATABASE_URL=sqlite:///./data/knowledge_assistant.db
CHROMA_PERSIST_DIR=./data/chroma
```

**`JWT_SECRET_KEY` must be set to a real random value before any real deployment** - the code ships with an obvious placeholder default that only exists so local dev doesn't crash on a missing `.env`.

`backend/.env` ships in this zip with these keys **blank** - fill in your actual Key Gateway URL/API key and NVIDIA API key there. It's already listed in `.gitignore` so it won't get committed if you push this to a repo.

Frontend: `frontend/.env.example` → `VITE_API_URL=http://localhost:8000`

## Installation & Running

### Backend

```bash
cd backend
python -m venv venv && source venv/bin/activate   # optional but recommended
pip install -r requirements.txt
# edit .env - set JWT_SECRET_KEY, and optionally KEY_GATEWAY_URL / KEY_GATEWAY_API_KEY / NVIDIA_API_KEY
python seed.py                                       # creates default admin/user accounts + ingests sample docs
uvicorn app.main:app --reload --port 8000
```

`seed.py` creates two accounts for local testing - **change or delete these before any real deployment**:

| Role  | Email               | Password      |
|-------|---------------------|---------------|
| ADMIN | admin@gmail.com   | Admin@12345   |
| USER  | user@gmail.com    | User@12345    |

### Frontend

```bash
cd frontend
npm install
cp .env.example .env
npm run dev                                          # http://localhost:5173
```

Visit `/login`, sign in with either seeded account (or register a new user account at `/register`). Admins land on `/admin`, regular users on `/app`.

### Running Tests

```bash
cd backend
pytest tests/ -v
```

## Adding Your Credentials

The whole app already runs end-to-end without either credential. Once you have them:

1. **Key Gateway**: set `KEY_GATEWAY_URL` and `KEY_GATEWAY_API_KEY` in `backend/.env`
2. **NVIDIA**: get a key from https://build.nvidia.com, set `NVIDIA_API_KEY` in `backend/.env`
3. Restart the backend - no code changes needed either way. `/api/system/status` and the debug panel will immediately show `key_gateway` / `nvidia` instead of `local_fallback` for whichever stages you've configured. You can set one without the other; the pipeline runs correctly either way.

## RAG Flow (for your presentation)

```
User Question
   → Query Improvement (Key Gateway rewrite, or rule-based expansion)
   → BM25 keyword search over indexed chunks
   → ChromaDB vector search (NVIDIA NV-Embed, or local fallback)
   → Reciprocal Rank Fusion (weighted toward BM25 in fallback mode)
   → NVIDIA cross-encoder reranking (or lexical fallback)
   → Relevance threshold + near-duplicate removal + top-N limit
   → Prompt template (system instructions + context + history + question)
   → Key Gateway chat model (or extractive fallback)
   → Answer with inline [1][2] citations + source cards + full debug trace
```

## Known Limitations

- Local fallback embeddings (hashing-based) are not semantically meaningful - they exist only so the app is demoable before a real NVIDIA key is added. Retrieval quality is materially better once NVIDIA embeddings are live.
- The NVIDIA reranking request/response shape in `nvidia_client.py` was built from documentation, not verified against live traffic (no key was available during development) - worth a smoke test once your key is in.
- No authentication/authorization layer - this is a single-tenant demo; a real deployment would need user accounts and per-department document access control.
- Reranking and embeddings add API latency; there's no caching layer yet for repeated queries.
- Ingestion is synchronous (upload blocks until fully chunked + embedded); fine for demo-sized documents, but a production system would background this for large files.

## Future Improvements

- Streaming chat responses (token-by-token)
- Query caching / semantic cache for repeated questions
- Role-based document access (e.g. Finance docs restricted to Finance team)
- Async/background ingestion with progress updates
- Multi-document comparison ("what changed between the old and new leave policy?")

## Presentation / Demo Flow

1. Show `/api/system/status` - explain local fallback vs. live mode, per backend
2. Upload a new HR document live → show it chunked and appear in the Documents page
3. Ask a grounded question → show the cited answer + source cards
4. Open the debug trace → walk through query improvement, BM25 vs. vector scores, what got reranked, what made the final context
5. Ask an off-topic question → show it gracefully declines rather than hallucinating
6. Explain the architecture diagram and the local-fallback design decision
