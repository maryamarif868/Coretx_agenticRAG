# Cortex — Agentic RAG Document Assistant

Cortex is a full-stack, multi-user, agentic Retrieval-Augmented Generation (RAG) system. Users sign up, upload their own documents (PDF/TXT or pasted text), and ask questions, by text or voice, that are answered strictly from the uploaded content, with inline citations, guardrails, and a self-evaluating pipeline.

---

## Tech Stack

- **Backend:** Python, FastAPI, Uvicorn
- **Agent orchestration:** LangChain, LangGraph
- **Vector store:** ChromaDB (persisted to disk, one isolated collection per user)
- **LLM provider:** Groq (with multi-key failover)
- **Embeddings:** HuggingFace `sentence-transformers`
- **Database:** PostgreSQL on Supabase (chat sessions, messages, user accounts)
- **Ingestion:** PDF/TXT upload, pasted text, website URL scraping
- **Auth:** JWT (python-jose), bcrypt password hashing (passlib), Google OAuth
- **Evaluation:** RAGAS (LLM-as-judge)
- **Speech:** Groq Whisper (speech-to-text), gTTS (text-to-speech)
- **Observability:** LangSmith tracing
- **Frontend:** React + Vite
- **Deployment:** Docker, Railway

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
        ├── Per-user document store
        │     ├── ChromaDB collection (unique per user, per upload)
        │     └── SQLite-backed chat sessions (scoped by user_id)
        │
        └── LangGraph agentic pipeline (per question)
              │
              ├── route            → direct / retrieve / recall
              ├── select_files     → which document(s) to search
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
Documents are chunked (`RecursiveCharacterTextSplitter`), embedded, and stored in ChromaDB. Answers are generated strictly from retrieved context, with inline bracket citations (`[1]`, `[2]`) mapped back to source document and page.

### 2. Agentic Workflow (LangGraph)
Rather than a single fixed search-then-answer flow, each question moves through a graph of decision nodes:
- **Route** — classifies the question as small talk, a document question, or a question about a previous chat session
- **Select files** — when multiple documents are indexed, decides which are relevant
- **Decide context** — chooses between answering from a document summary or searching full chunks
- **Condense** — rewrites context-dependent follow-ups ("what about the second one?") into standalone queries using chat history
- **Retrieve → Rerank → Grade** — a retry loop that re-searches with a reworded query if the first retrieval isn't relevant, up to 2 attempts

### 3. Hybrid Search
Retrieval combines **vector similarity search** (semantic) with **BM25 keyword search** (lexical), deduplicated into a single candidate set, so queries that rely on exact terms or rare keywords aren't missed by embeddings alone.

### 4. Reranking
A cross-encoder (`ms-marco-MiniLM-L-6-v2`) re-scores a wider pool of retrieved candidates against the query directly (rather than via cosine similarity on separately-encoded vectors), and only the top-k most relevant chunks are passed to the answering model.

### 5. Guardrails
- **Input:** blocks common prompt-injection phrasings (e.g. "ignore previous instructions", "reveal your system prompt"), enforces a length limit, and redacts card/ID-like number patterns before they reach the model
- **Output:** normalizes malformed citation brackets, strips citations pointing to chunks that don't exist, and flags (with an inline note) any email, link, or number in the answer that isn't actually present in the retrieved context
- Verified with a red-team script of adversarial prompts alongside a set of normal questions to confirm no false blocks

### 6. Multi-Session Recall
A dedicated routing path lets users ask about earlier conversations in *other* chat sessions ("what did I ask you yesterday?"), answered from a summarized cross-session history rather than the document index.

### 7. Automated RAG Evaluation (RAGAS)
A fully self-contained evaluation pipeline:
1. An LLM reads random chunks from the indexed document and generates a question + ground-truth answer pair
2. The real agent pipeline answers each generated question
3. A separate judge LLM scores the results on four RAGAS metrics: **faithfulness**, **answer relevancy**, **context precision**, and **context recall**

This closes the loop without requiring a hand-maintained test set, while keeping the question-generating model separate from the answering model to reduce circularity.

### 8. Multi-Key Groq Failover
LLM calls automatically fail over across multiple configured Groq API keys if one hits a rate limit, using LangChain's `with_fallbacks`, so a single exhausted quota doesn't take down the whole pipeline.

### 9. Authentication & Per-User Isolation
- JWT-based signup/login with bcrypt-hashed passwords, plus Google OAuth and a forgot/reset-password flow
- Every chat session, message, and document index is scoped strictly to the logged-in user, enforced at the database query level, not just in application logic
- Each user's documents live in their own isolated ChromaDB collection and their own on-disk folder, so uploading a new document never affects another user's data
- A new signup always lands on an empty upload screen; nothing is inherited from other accounts

### 10. Persistence
Chat sessions and messages are stored in SQLite (not in-memory), using a fresh connection per call for thread safety under FastAPI's streaming responses. Document indexes and uploaded files are persisted to disk per user and automatically restored on server restart.

### 11. Voice Chat
- Speech-to-text via Groq Whisper, locked to English to avoid auto-detect misfires
- The transcribed question runs through the same agentic pipeline as typed questions
- The answer is cleaned of markdown and citation markers, then converted to speech via gTTS and streamed back to the browser as playable audio

### 12. Observability (LangSmith)
Every agent run is tagged and traced to LangSmith, giving visibility into each node's input/output, routing decisions, and latency, useful for debugging retrieval or routing issues that wouldn't be visible from the chat UI alone.

---

## Key API Endpoints

| Endpoint | Purpose |
|---|---|
| `POST /api/auth/signup` | Create an account |
| `POST /api/auth/login` | Email/password login |
| `POST /api/auth/google` | Google OAuth login |
| `POST /api/auth/forgot-password` / `reset-password` | Password reset flow |
| `GET /api/state` | Current user's indexed-document status |
| `POST /api/index` | Upload and index document(s) |
| `POST /api/index/add` | Add more documents to an existing index |
| `DELETE /api/index` | Remove the current user's documents |
| `GET/POST /api/sessions` | List / create chat sessions |
| `GET /api/sessions/{id}` | Load a session's message history |
| `POST /api/chat` | Streamed, agentic Q&A |
| `POST /api/voice-chat` | Voice question → voice + text answer |
| `POST /api/evaluate/auto` | Run the automated RAGAS evaluation |

---

## Frontend

Built with React and Vite. Features a dark, animated UI (network-graph background, an illustrated robot assistant that reflects agent state), a sidebar with document stats and chat history, streaming token-by-token answers, voice input/output, and a glassmorphic login/signup screen.
