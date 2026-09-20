# My Learning AI Assistant

> RAG-powered AI assistant with a cosmic glassmorphism UI.
> Upload PDFs, ask questions with text or voice, get intelligent answers grounded in your documents.

---

## Tech Stack

| Layer        | Technology                                                           |
| ------------ | -------------------------------------------------------------------- |
| Frontend     | Next.js 16, Tailwind CSS, Framer Motion, Web Speech API              |
| Backend      | Python 3.11+, FastAPI, LangChain                                     |
| LLM          | Gemini 2.5 Flash via OpenRouter                                      |
| Embeddings   | `sentence-transformers/all-MiniLM-L6-v2` (runs locally — no API key) |
| Vector Store | FAISS (persisted to disk)                                            |
| Re-ranking   | Cross-Encoder `ms-marco-MiniLM-L-6-v2`                               |
| Evaluation   | RAGAS                                                                |
| DevOps       | Docker, GitHub Actions                                               |

---

## Prerequisites

| Requirement             | Version      | Install            |
| ----------------------- | ------------ | ------------------ |
| Python                  | 3.11 or 3.12 | https://python.org |
| Node.js + npm           | 18+          | https://nodejs.org |
| Docker + Docker Compose | 20+          | https://docker.com |

> Docker is only needed for the Docker path. Local dev does not require it.

---

## API Keys You Need

Only **one** API key is required:

| Key                  | Where to get it            | What it's for          |
| -------------------- | -------------------------- | ---------------------- |
| `OPENROUTER_API_KEY` | https://openrouter.ai/keys | LLM (Gemini 2.5 Flash) |

Embeddings run **locally** via `sentence-transformers` — no additional key needed.

---

## Usage

1. **Upload a PDF** — Drag and drop a PDF (up to 50 MB) into the upload panel and add a short description (5–1000 characters).
2. **Ask a question** — Type in the chat box or click the **microphone** button to speak.
3. **Hear the answer** — Responses are read aloud automatically via Text-to-Speech (can be toggled off).
4. **Explore sources** — Click "N sources" on any AI message to see the retrieved chunks and page numbers.
5. **Use suggestion chips** — After uploading, the app generates document-specific questions — click any to jump straight in.
6. **Browse topics** — The sidebar shows extracted topics from your document for quick navigation.

---

## API Reference

| Endpoint           | Method | Description                                                       |
| ------------------ | ------ | ----------------------------------------------------------------- |
| `/health`          | GET    | Health check — returns `{"status":"ok"}`                          |
| `/api/upload`      | POST   | Upload a PDF + description; returns `document_id` and chunk count |
| `/api/query`       | POST   | RAG query; returns answer + source chunks                         |
| `/api/suggestions` | POST   | Generate question suggestions for a document                      |
| `/api/topics`      | POST   | Extract topic list from a document                                |
| `/api/tts`         | POST   | Text-to-speech synthesis                                          |
| `/api/evaluate`    | POST   | RAGAS evaluation of an answer                                     |
| `/docs`            | GET    | Interactive Swagger UI                                            |

---

## Project Structure

```
AgenticAI_SSR_Project1/
├── backend/
│   ├── app/
│   │   ├── main.py                    # FastAPI entry point, CORS, routers
│   │   ├── config.py                  # Pydantic settings (reads .env)
│   │   ├── models.py                  # Request/response schemas
│   │   ├── routes/
│   │   │   ├── upload.py              # POST /api/upload
│   │   │   ├── query.py               # POST /api/query
│   │   │   ├── suggestions.py         # POST /api/suggestions
│   │   │   ├── topics.py              # POST /api/topics
│   │   │   ├── tts.py                 # POST /api/tts
│   │   │   └── evaluate.py            # POST /api/evaluate
│   │   ├── services/
│   │   │   ├── document_processor.py  # PDF → chunks → FAISS
│   │   │   ├── embeddings.py          # sentence-transformers + retry
│   │   │   ├── rag_pipeline.py        # Full RAG orchestration
│   │   │   ├── reranker.py            # Cross-Encoder re-ranking
│   │   │   └── evaluator.py           # RAGAS metrics + JSONL logging
│   │   └── utils/retry.py             # Tenacity decorators
│   ├── tests/test_api.py
│   ├── requirements.txt
│   ├── Dockerfile
│   ├── .env.example                   # Copy to .env and fill in your key
│   └── .env                           # Your secrets (git-ignored)
├── frontend/
│   ├── public/                        # Static assets (served at /)
│   ├── src/
│   │   ├── app/                       # Next.js App Router (layout, page)
│   │   ├── components/                # ChatInterface, UploadZone, etc.
│   │   ├── hooks/                     # useSpeechRecognition, useTextToSpeech
│   │   ├── lib/api.ts                 # Typed API client
│   │   └── types/index.ts
│   ├── .env.local.example             # Copy to .env.local and set your backend URL
│   └── .env.local                     # Your local frontend config (git-ignored)
├── docker-compose.yml
└── README.md
```

---

## Environment Variables

### Backend (`backend/.env`)

| Variable             | Required | Default                   | Description                         |
| -------------------- | -------- | ------------------------- | ----------------------------------- |
| `OPENROUTER_API_KEY` | **Yes**  | —                         | OpenRouter API key                  |
| `ALLOWED_ORIGINS`    | No       | `http://localhost:3000`   | Comma-separated CORS origins        |
| `FAISS_INDEX_PATH`   | No       | `./faiss_index`           | Where FAISS index is stored         |
| `EVAL_LOG_PATH`      | No       | `./logs/ragas_eval.jsonl` | Where evaluation logs are written   |
| `LOG_LEVEL`          | No       | `INFO`                    | `DEBUG`, `INFO`, `WARNING`, `ERROR` |

### Frontend (`frontend/.env.local`)

| Variable              | Required | Default                 | Description                                                   |
| --------------------- | -------- | ----------------------- | ------------------------------------------------------------- |
| `NEXT_PUBLIC_API_URL` | No       | `http://localhost:8000` | Backend URL used by the browser for direct calls (PDF upload) |

> In Docker, `NEXT_PUBLIC_API_URL` stays as `http://localhost:8000` so the browser can reach the backend. The Next.js server uses `INTERNAL_API_URL=http://backend:8000` for server-side proxying — this is set automatically by `docker-compose.yml`.

---

## Evaluation Logs

RAGAS metrics are appended to `backend/logs/ragas_eval.jsonl` on each evaluation request:

```json
{
  "timestamp": "2026-04-02T12:00:00Z",
  "question": "What is the main topic?",
  "answer_preview": "The main topic is...",
  "metrics": {
    "faithfulness": 0.92,
    "answer_relevancy": 0.88,
    "context_precision": 0.85,
    "context_recall": null
  }
}
```

---

## Troubleshooting

**Backend won't start — `ValidationError: openrouter_api_key`**
You haven't set your API key. Open `backend/.env` and set `OPENROUTER_API_KEY=sk-or-v1-...`.

**`npm install` fails or `node_modules` is missing**
Make sure you're inside the `frontend/` directory: `cd frontend && npm install`.

**PDF upload fails in Docker**
Verify both containers are running and the backend is healthy:

```bash
docker compose ps
```

The frontend container starts only after the backend health check passes (~15 seconds).

**`http://localhost:3000` shows blank page or can't connect**
Wait ~2 minutes after `docker compose up --build` for the initial model download and Next.js build to finish. You can tail the logs to watch progress:

```bash
docker compose logs -f
```

**`backend` hostname not resolving in browser**
This means `NEXT_PUBLIC_API_URL` was set to `http://backend:8000`. The browser cannot resolve Docker-internal hostnames. Keep `NEXT_PUBLIC_API_URL=http://localhost:8000` — the Docker setup handles the internal routing automatically.

**Port already in use**
Another process is using port 3000 or 8000. Find and stop it:

```bash
# Mac/Linux:
lsof -i :8000   # or :3000
kill -9 <PID>

# Windows:
netstat -ano | findstr :8000
taskkill /PID <PID> /F
```

---

## CI/CD

The GitHub Actions workflow (`.github/workflows/main.yml`) runs on every push:

1. **Backend** — Ruff lint + format check + pytest
2. **Frontend** — ESLint + TypeScript check + Next.js build
3. **Docker** — Build both images (cached via GitHub Actions cache)
4. **Deploy** — Placeholder step (wire up to your infrastructure)
