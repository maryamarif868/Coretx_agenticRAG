# Cortex — Agentic RAG Document Assistant

Cortex is a full-stack, multi-user, agentic Retrieval-Augmented Generation (RAG) system. Users sign up, add their own knowledge sources (PDF/TXT files, pasted text, or a website URL), and ask questions, by text or voice, that are answered strictly from the added content, with inline citations, guardrails, and a self-evaluating pipeline. A built-in admin panel lets admins manage users and monitor platform usage.

---

## Tech Stack

**Backend:** Python, FastAPI, Uvicorn
**Agent orchestration:** LangChain, LangGraph
**Vector store:** ChromaDB (persisted to disk, one isolated collection per user)
**LLM provider:** Groq (with multi-key failover)
**Embeddings:** HuggingFace `sentence-transformers`
**Database:** PostgreSQL on Supabase (chat sessions, messages, user accounts)
**Ingestion:** PDF/TXT upload, pasted text, website URL scraping
**Auth:** JWT (python-jose), bcrypt password hashing (passlib), Google OAuth
**Evaluation:** RAGAS (LLM-as-judge)
**Speech:** Groq Whisper (speech-to-text), gTTS (text-to-speech)
**Observability:** LangSmith tracing
**Frontend:** React + Vite
**Deployment:** Docker, Railway

---

## Models in Use

| Purpose | Model |
|---|---|
| Main agent LLM (routing, grading, answering) | `openai/gpt-oss-120b` (via Groq) |
| RAGAS judge LLM (evaluation scoring) | `openai/gpt-oss-20b` (via Groq) |
| Reranker | `cross-encoder/ms-marco-MiniLM-L-6-v2` |
| Embeddings | `sentence-transformers/all-MiniLM-L6-v2` |
| Speech-to-text | `whisper-large-v3-turbo` (via Groq) |
| Text-to-speech | gTTS |

---

## Architecture Overview

```
React (Vite) frontend
        │  JWT bearer token on every request
        ▼
FastAPI backend
        │
        ├── Auth layer (JWT, bcrypt, Google OAuth, password reset)
        │
        ├── Admin panel (role-based: user management, usage stats)
        │
        ├── Multi-source ingestion (PDF / TXT / pasted text / website URL)
        │
        ├── Per-user document store
        │     ├── ChromaDB collection (unique per user, per upload)
        │     └── Supabase PostgreSQL (chat sessions, messages, scoped by user_id)
        │
        └── LangGraph agentic pipeline (per question)
              │
              ├── route            → direct / retrieve / recall
              ├── select_files     → which source(s) to search
              ├── decide_context   → summary-only vs. full chunk search
              ├── condense         → rewrite follow-up as standalone query
              ├── retrieve         → hybrid search (vector + BM25)
              ├── rerank           → cross-encoder re-scoring
              ├── grade            → relevance check, retry loop (max 2 attempts)
              └── answer           → cited, guardrail-checked response
```

All LLM calls are traced to LangSmith for observability and debugging.

---

## Features

### 1. Core RAG Pipeline
Documents are chunked (`RecursiveCharacterTextSplitter`), embedded, and stored in ChromaDB. Answers are generated strictly from retrieved context, with inline bracket citations (`[1]`, `[2]`) mapped back to the source document, page, or URL.

### 2. Multi-Source Ingestion
Users can build their knowledge base from three kinds of sources:
- **File upload:** PDF and TXT files
- **Pasted text:** raw text typed or pasted directly into the app
- **Website URL:** the user enters a site URL, the page content is fetched, cleaned, chunked, and indexed like any other document, so they can ask questions about that site

All sources go through the same chunking, embedding, and ChromaDB pipeline, and answers carry citations back to the original file or URL.

### 3. Agentic Workflow (LangGraph)
Rather than a single fixed search-then-answer flow, each question moves through a graph of decision nodes:
- **Route** — classifies the question as small talk, a document question, or a question about a previous chat session
- **Select files** — when multiple sources are indexed, decides which are relevant
- **Decide context** — chooses between answering from a document summary or searching full chunks
- **Condense** — rewrites context-dependent follow-ups ("what about the second one?") into standalone queries using chat history
- **Retrieve → Rerank → Grade** — a retry loop that re-searches with a reworded query if the first retrieval isn't relevant, up to 2 attempts

### 4. Hybrid Search
Retrieval combines **vector similarity search** (semantic) with **BM25 keyword search** (lexical), deduplicated into a single candidate set, so queries that rely on exact terms or rare keywords aren't missed by embeddings alone.

### 5. Reranking
A cross-encoder (`ms-marco-MiniLM-L-6-v2`) re-scores a wider pool of retrieved candidates against the query directly (rather than via cosine similarity on separately-encoded vectors), and only the top-k most relevant chunks are passed to the answering model.

### 6. Guardrails
- **Input:** blocks common prompt-injection phrasings (e.g. "ignore previous instructions", "reveal your system prompt"), enforces a length limit, and redacts card/ID-like number patterns before they reach the model
- **Output:** normalizes malformed citation brackets, strips citations pointing to chunks that don't exist, and flags (with an inline note) any email, link, or number in the answer that isn't actually present in the retrieved context
- Verified with a red-team script of adversarial prompts alongside a set of normal questions to confirm no false blocks

### 7. Multi-Session Recall
A dedicated routing path lets users ask about earlier conversations in *other* chat sessions ("what did I ask you yesterday?"), answered from a summarized cross-session history rather than the document index.

### 8. Automated RAG Evaluation (RAGAS)
A fully self-contained evaluation pipeline:
1. An LLM reads random chunks from the indexed content and generates a question + ground-truth answer pair
2. The real agent pipeline answers each generated question
3. A separate judge LLM scores the results on four RAGAS metrics: **faithfulness**, **answer relevancy**, **context precision**, and **context recall**

This closes the loop without requiring a hand-maintained test set, while keeping the question-generating model separate from the answering model to reduce circularity.

### 9. Multi-Key Groq Failover
LLM calls automatically fail over across multiple configured Groq API keys if one hits a rate limit, using LangChain's `with_fallbacks`, so a single exhausted quota doesn't take down the whole pipeline.

### 10. Authentication & Per-User Isolation
- JWT-based signup/login with bcrypt-hashed passwords, plus Google OAuth and a forgot/reset-password flow
- Every chat session, message, and document index is scoped strictly to the logged-in user, enforced at the database query level, not just in application logic
- Each user's documents live in their own isolated ChromaDB collection and their own on-disk folder, so adding a new source never affects another user's data
- A new signup always lands on an empty upload screen; nothing is inherited from other accounts

### 11. Persistence (Supabase PostgreSQL)
Chat sessions, messages, and user accounts are stored in PostgreSQL hosted on Supabase, so data survives restarts and redeploys. Document indexes and uploaded files are persisted per user and automatically restored on server restart.

### 12. Admin Panel
A protected admin dashboard (role-based access) for managing the platform:
- View and manage registered users
- Monitor usage (chats, uploads, indexed sources)
- Review platform activity

### 13. Voice Chat
- Speech-to-text via Groq Whisper, locked to English to avoid auto-detect misfires
- The transcribed question runs through the same agentic pipeline as typed questions
- The answer is cleaned of markdown and citation markers, then converted to speech via gTTS and streamed back to the browser as playable audio

### 14. Observability (LangSmith)
Every agent run is tagged and traced to LangSmith, giving visibility into each node's input/output, routing decisions, token usage, and latency. Useful for debugging retrieval or routing issues that aren't visible from the chat UI alone.

---

## Key API Endpoints

| Endpoint | Purpose |
|---|---|
| `POST /api/auth/signup` | Create an account |
| `POST /api/auth/login` | Email/password login |
| `POST /api/auth/google` | Google OAuth login |
| `POST /api/auth/forgot-password` / `reset-password` | Password reset flow |
| `GET /api/state` | Current user's indexed-source status |
| `POST /api/index` | Upload and index file(s) or pasted text |
| `POST /api/index/url` | Index a website by URL |
| `POST /api/index/add` | Add more sources to an existing index |
| `DELETE /api/index` | Remove the current user's sources |
| `GET/POST /api/sessions` | List / create chat sessions |
| `GET /api/sessions/{id}` | Load a session's message history |
| `POST /api/chat` | Streamed, agentic Q&A |
| `POST /api/voice-chat` | Voice question → voice + text answer |
| `POST /api/evaluate/auto` | Run the automated RAGAS evaluation |
| `GET /api/admin/users` | Admin: list users |
| `GET /api/admin/stats` | Admin: platform usage stats |

---

## Frontend

Built with React and Vite. Features a dark, animated UI (network-graph background, an illustrated robot assistant that reflects agent state), a sidebar with source stats and chat history, streaming token-by-token answers, voice input/output, a glassmorphic login/signup screen, and an admin dashboard for admin users.

---

## Environment Variables

Create a `.env` file in the project root (never commit it):

| Variable | Purpose |
|---|---|
| `GROQ_API_KEY` | Groq LLM + Whisper (add extra keys for failover) |
| `DATABASE_URL` | Supabase PostgreSQL connection string |
| `JWT_SECRET` | JWT token signing |
| `GOOGLE_CLIENT_ID` | Google OAuth |
| `LANGCHAIN_TRACING_V2` | Enable LangSmith tracing (`true`) |
| `LANGCHAIN_API_KEY` | LangSmith API key |
| `LANGCHAIN_PROJECT` | LangSmith project name |

---

## Run Locally

```bash
# Backend
pip install -r requirements.txt
uvicorn main:app --reload

# Frontend
cd frontend
npm install
npm run dev
```

## Run with Docker

```bash
docker build -t cortex .
docker run -p 8000:8000 --env-file .env cortex
```

## Deployment

The app is containerized with Docker and deployed on Railway. Environment variables are set in the Railway service's **Variables** tab, and a Railway volume is used to persist the ChromaDB data directory across redeploys.
