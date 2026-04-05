# GitLab Chatbot

An AI-powered chatbot for GitLab's Handbook and Direction pages.

---

## Project Structure

```
gitlab-chatbot/
├── scraper/
│   └── scrape.py          # Crawls handbook.gitlab.com & about.gitlab.com/direction
├── pipeline/
│   ├── chunker.py         # Splits raw pages into overlapping text chunks
│   ├── ingest.py          # Embeds chunks via Gemini API → stores in ChromaDB
│   └── retriever.py       # Query-time semantic search (used by the backend)
├── data/
│   ├── raw/               # JSON files: one per scraped page
│   ├── chunks/            # JSON files: chunked text with metadata
│   └── chroma/            # ChromaDB persistent vector store
├── requirements.txt
└── README.md
```

---

## Phase 1 Setup — Data Pipeline

### 1. Install dependencies

```bash
pip install -r requirements.txt
```

### 2. Scrape GitLab pages

```bash
python -m scraper.scrape
```

This crawls up to 500 pages from:
- `https://handbook.gitlab.com/`
- `https://about.gitlab.com/direction/`

Raw pages are saved as JSON under `data/raw/`. Expect this to take
5–15 minutes depending on your connection speed (it's polite — 0.5s
delay between requests).

### 3. Chunk the pages

```bash
python -m pipeline.chunker
```

Splits each page into ~400-token overlapping chunks. Results go to `data/chunks/`.

### 4. Embed and ingest into ChromaDB

```powershell
$env:GEMINI_API_KEY = "your_key_here"   # Get free at aistudio.google.com
python -m pipeline.ingest
```

⚠️ **Note**: The Gemini free tier has strict rate limits. The ingest script includes
automatic retry logic with exponential backoff. Full ingestion (~4700 chunks) takes 30-45
minutes on the free tier. Safe to re-run — already-embedded chunks are skipped.

Embeds every chunk using Gemini `embedding-001` and upserts into
a local ChromaDB collection. Safe to re-run — already-embedded chunks
are skipped.

### 5. Test retrieval

```bash
python -m pipeline.retriever "What is GitLab's approach to remote work?"
```

---

## Environment Variables

| Variable        | Required | Description                                      |
|-----------------|----------|--------------------------------------------------|
| `GEMINI_API_KEY`| Yes      | Gemini API key (free tier: 1,500 req/day)        |
| `BATCH_SIZE`    | No       | Chunks per embedding API call (default: 50)      |

---

## Notes

- The scraper respects `handbook.gitlab.com` and only crawls `/direction/` paths on `about.gitlab.com`.
- Chunking uses a sliding window with 80-token overlap to preserve context across chunk boundaries.
- ChromaDB is stored locally — no external database setup required.
- The Gemini free tier (1,500 embedding calls/day) is sufficient for the full corpus.
