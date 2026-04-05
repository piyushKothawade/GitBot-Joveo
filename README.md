# GitBot — GitLab Knowledge Assistant

An AI-powered chatbot that answers questions about GitLab's
[Handbook](https://handbook.gitlab.com/) and [Direction](https://about.gitlab.com/direction/)
pages. Built with a RAG pipeline, Gemini LLM, Jina embeddings, ChromaDB, FastAPI, and React.

---

## Live Demo

| Service | URL |
|---------|-----|
| Frontend | `https://your-app.vercel.app` |
| Backend API | `https://your-hf-space.hf.space` |
| API Docs | `https://your-hf-space.hf.space/docs` |

---

## Architecture

```
User
 │
 ▼
React Frontend (Vercel)
 │  SSE streaming over /chat/stream
 │  Vercel rewrites → HF Space
 ▼
FastAPI Backend (HF Spaces — Docker)
 ├── Guardrail check   →  Gemini Flash (relevance classifier)
 ├── Retrieval         →  Jina embeddings + ChromaDB (4,697 vectors)
 └── Generation        →  Gemini Flash (RAG answer + streaming)
```

**Data pipeline (run once locally):**
```
GitLab Handbook + Direction pages
 │  scraper/scrape.py   (BeautifulSoup crawler)
 ▼
data/raw/               (JSON, one file per page)
 │  pipeline/chunker.py (sliding window, ~400 tok, 80 tok overlap)
 ▼
data/chunks/            (JSON, overlapping text chunks)
 │  pipeline/ingest.py  (Jina embeddings → ChromaDB upsert)
 ▼
data/chroma/            (persistent vector store — committed to repo)
```

---

## Project Structure

```
gitlab-chatbot/
├── scraper/
│   └── scrape.py           Crawls handbook + direction pages
├── pipeline/
│   ├── chunker.py          Splits pages into overlapping chunks
│   ├── ingest.py           Embeds chunks via Jina → ChromaDB
│   └── retriever.py        Query-time semantic search
├── backend/
│   ├── main.py             FastAPI app (3 endpoints)
│   ├── llm.py              Gemini wrapper (stream + guardrail)
│   └── prompts.py          System prompt + RAG prompt builder
├── frontend/
│   ├── src/
│   │   ├── App.jsx         Root layout + empty state
│   │   ├── hooks/
│   │   │   └── useChat.js  SSE streaming hook
│   │   ├── components/
│   │   │   ├── Message.jsx     Chat bubble + markdown renderer
│   │   │   ├── ChatInput.jsx   Input + source filter controls
│   │   │   └── Sidebar.jsx     Brand + source legend
│   │   └── index.css       Design tokens + animations
│   ├── package.json
│   └── vite.config.js
├── Dockerfile              HF Spaces Docker deployment
├── vercel.json             Vercel rewrite rules
├── requirements.txt
├── app.py                  HF Spaces entrypoint
└── .gitignore
```

---

## Local Setup

### Prerequisites

- Python 3.11+ (3.14 works — see note on `lxml` below)
- Node.js 18+
- API keys: [Gemini](https://aistudio.google.com) (free) · [Jina](https://jina.ai) (free)

### 1. Clone and install

```bash
git clone https://github.com/your-username/gitlab-chatbot
cd gitlab-chatbot
pip install -r requirements.txt
```

> **Python 3.14 note:** `lxml` has no wheels for 3.14 yet.
> The scraper uses `html.parser` instead — no action needed.

### 2. Set environment variables

```bash
export GEMINI_API_KEY=your_gemini_key
export JINA_API_KEY=your_jina_key
```

### 3. Run the data pipeline

```bash
# Step 1 — Scrape GitLab pages (~10 min, up to 500 pages)
python -m scraper.scrape

# Step 2 — Chunk the pages
python -m pipeline.chunker

# Step 3 — Embed and ingest into ChromaDB
python -m pipeline.ingest

# Step 4 — Test retrieval
python -m pipeline.retriever "What is GitLab's mission?"
```

### 4. Start the backend

```bash
uvicorn backend.main:app --reload --port 8000
```

Visit `http://localhost:8000/docs` for the interactive API explorer.

### 5. Start the frontend

```bash
cd frontend
npm install
npm run dev
```

Visit `http://localhost:3000`.

---

## API Reference

### `GET /health`

```json
{
  "status": "ok",
  "retriever_ready": true,
  "collection_size": 4697
}
```

### `POST /chat`

```json
// Request
{
  "query": "What is GitLab's PTO policy?",
  "history": [],
  "top_k": 6,
  "source_filter": "handbook",
  "use_hybrid": false
}

// Response
{
  "answer": "GitLab offers ...",
  "sources": [
    { "title": "...", "url": "...", "source": "handbook", "score": 0.91 }
  ],
  "query": "What is GitLab's PTO policy?"
}
```

### `POST /chat/stream`

Same request body as `/chat`. Returns `text/event-stream`:

```
data: {"type": "token",   "content": "GitLab "}
data: {"type": "token",   "content": "offers "}
data: {"type": "sources", "content": [{...}]}
data: {"type": "done"}
```

Error events:
```
data: {"type": "error", "content": "Query is off-topic..."}
```

---

## Deployment

### Backend → Hugging Face Spaces

1. Create a new Space at [huggingface.co/spaces](https://huggingface.co/spaces)
   - SDK: **Docker**
   - Visibility: Public

2. Add secrets in **Space Settings → Variables and Secrets**:
   ```
   GEMINI_API_KEY = your_key
   JINA_API_KEY   = your_key
   CORS_ORIGINS   = https://your-app.vercel.app
   ```

3. Commit and push the repo (including `data/chroma/` via Git LFS):
   ```bash
   git lfs install
   git lfs track "data/chroma/**"
   git add .gitattributes data/chroma/
   git commit -m "Add ChromaDB vector store"
   git remote add hf https://huggingface.co/spaces/your-username/gitlab-chatbot-backend
   git push hf main
   ```

4. HF Spaces builds the Docker image automatically. Watch the build logs in the Space UI.
   Once the Space is running, note your Space URL: `https://your-hf-space.hf.space`

### Frontend → Vercel

1. Update `vercel.json` — replace the placeholder backend URL:
   ```json
   "destination": "https://your-hf-space.hf.space/chat/:path*"
   ```

2. Push to GitHub, then import the repo at [vercel.com/new](https://vercel.com/new):
   - **Root Directory:** `frontend`
   - **Framework:** Vite
   - No environment variables needed (API calls are proxied via `vercel.json`)

3. Vercel deploys automatically on every push to `main`.

---

## Key Design Decisions

### Why RAG over fine-tuning?
GitLab's Handbook is a living document — pages update frequently. RAG retrieves
the latest indexed content at query time without retraining. Fine-tuning would
bake in a snapshot that goes stale.

### Why Jina embeddings over Gemini?
Gemini's embedding API has strict per-minute rate limits that caused failures
during bulk ingestion. Jina's API is more permissive, and `jina-embeddings-v4`
performs comparably on retrieval benchmarks.

### Why chunking with overlap?
A fact often spans a paragraph boundary. With pure non-overlapping chunks, the
retriever might return a chunk that starts mid-sentence, losing critical context.
80-token overlap ensures adjacent chunks share enough text to cover boundary cases.

### Why SSE over WebSockets?
SSE is unidirectional (server → client), which is all we need for streaming
LLM responses. It works over standard HTTP/1.1, requires no upgrade handshake,
and is trivially supported by `fetch()` + `ReadableStream` in the browser.

### Guardrail design (fail-open)
The relevance classifier uses a separate low-temperature Gemini call before the
main RAG call. It's designed to fail open — if the classifier itself errors,
the query is allowed through. This avoids blocking valid questions due to
classifier API hiccups.

### Vercel rewrite vs CORS
Rather than configuring CORS on the backend for every possible Vercel preview
URL, the `vercel.json` rewrites route `/chat/*` and `/health` requests to the
HF Space backend server-side. From the browser's perspective, all requests are
same-origin — no CORS headers needed.

---

## Bonus Features Implemented

- **Streaming responses** — tokens appear as they're generated, not all at once
- **Source citations** — every answer links back to the exact Handbook/Direction page
- **Relevance guardrail** — off-topic queries are rejected before hitting the LLM
- **Source filtering** — users can scope search to Handbook, Direction, or hybrid
- **Suggested questions** — contextual prompts on the empty state lower the barrier to start
- **Copy button** — one-click copy for any assistant response
- **Multi-turn context** — last 3 conversation exchanges are included in each LLM call
- **Hybrid search** — separate retrieval per source with configurable weighting

---

## Limitations & Future Improvements

- **Re-ingestion cadence:** The vector store is a point-in-time snapshot. A cron job
  re-running the scraper + ingest weekly would keep it current.
- **Re-ranking:** Adding a Jina reranker pass after retrieval would improve precision
  on complex queries where the top-6 chunks aren't all equally relevant.
- **Authentication:** The API has no auth layer. For internal GitLab use,
  adding GitLab OAuth would scope access to employees only.
- **Evaluation:** A held-out QA dataset benchmarking retrieval recall and answer
  faithfulness (e.g. with RAGAS) would give objective quality metrics.
