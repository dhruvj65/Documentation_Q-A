# Dailoqa

Dailoqa is a documentation site with an AI assistant built in. You can read the LangChain, LangGraph and Deep Agents docs, and ask the assistant questions about them. The assistant answers only from the docs and cites the sections it used. It can also turn a bug report or feature request into a Jira ticket, after checking that the issue isn't already filed.

## What it does

- **Documentation Q&A.** Answers questions using hybrid retrieval (vector search plus keyword search) over the docs, and cites the sections it used. If the docs don't cover a question, it says so instead of guessing.
- **Bug reports.** Asks only for missing details, collects screenshots or recordings, searches Jira for duplicates, then either links the user to an existing issue or files a new one.
- **Feature requests.** The same flow as bug reports, for new ideas.
- **Human approval.** Nothing is written to Jira until the user approves the exact ticket shown in the chat.
- **Live updates.** Replies stream to the browser over Server-Sent Events, and users get notified when their ticket is filed.

## Architecture

```
Frontend (Next.js) ──proxy + service JWT──▶ Backend API (FastAPI) ──▶ LangChain agent
       │                                          │                      │
  Google OAuth                               PostgreSQL ◀────────────────┤
  docs pages (MDX)                     (app data, checkpoints,           │
                                        pgvector doc chunks)             │
                                                  ▲                      ▼
                                           Worker process ─────────▶ Jira Cloud
                                      (dedupe → create/link → notify)
```

| Part | Stack | What it does |
|---|---|---|
| `Frontend/` | Next.js 16, React 19, Tailwind 4 | Docs pages, the "Ask AI" chat, Google sign-in, proxy routes to the backend |
| `Backend/` | FastAPI, LangChain / LangGraph, SQLAlchemy | Agent, chat and SSE APIs, RAG pipeline, Jira webhook |
| `Backend/src/app/worker` | Python | Background job queue that creates or links Jira tickets and uploads evidence |
| `Database/` | PostgreSQL + pgvector | Schema and setup scripts |

### The agent

The agent is built in `Backend/src/app/agent/factory.py` with `create_agent`, and uses these middleware:

- **Skills** load step-by-step playbooks from `Backend/skills/` (`documentation-qa`, `bug-report-creation`, `feature-request`) when they're needed.
- **Grounding** makes the model search the docs before it answers a docs question.
- **Human-in-the-loop** pauses `create_ticket` and `link_to_existing` until the user approves or rejects.
- **Summarization, tool retry and a model-call limit** keep long conversations stable.
- **Stream publisher** sends tool status updates to the chat.

Conversation state is saved in Postgres with the LangGraph Postgres checkpointer.

### Retrieval

- The source docs are in `Backend/docs-corpus/raw` (Mintlify MDX). `docs-formatter/format_docs.py` turns them into plain Markdown in `docs-corpus/formatted`.
- `python -m app.rag.ingest` splits the docs by heading and token count, embeds them locally with `BAAI/bge-small-en-v1.5`, and stores them in pgvector.
- Search combines vector and keyword results with reciprocal rank fusion. Distance and rank cut-offs let the agent decline questions the docs don't cover, and "diversity slots" stop one page from filling every result.
- `python -m app.rag.evaluate` tests the cut-off settings against `eval/golden_set.json` and reports hit@k, MRR and decline rate.

## Getting started

### Prerequisites

- Python 3.12+
- Node.js 20+
- PostgreSQL with the `pgvector` extension
- An LLM endpoint (an OpenAI-compatible LiteLLM gateway by default, or Anthropic)
- A Jira Cloud site and API token (for ticket creation)
- Google OAuth client credentials (for sign-in)

### 1. Database

Create `Backend/.env` first (see step 2). Then, on Windows:

```powershell
./Database/scripts/setup.ps1
```

This resets and creates the `dailoqa` database and applies the files in `Database/schema/` in order. On other systems, run the same files with `psql`:

```bash
psql -U <user> -d postgres -f Database/scripts/reset.sql
psql -U <user> -d dailoqa  -f Database/scripts/provision.sql
```

### 2. Backend

```bash
cd Backend
cp .env.example .env          # fill in DATABASE_URL, SERVICE_JWT_SECRET, LLM_*, JIRA_*
pip install -e ".[dev]"

python -m app.rag.ingest      # build the docs index (safe to re-run)
python run.py                 # API on http://127.0.0.1:8000
python -m app.worker          # in a second terminal: ticket worker
```

Or run the API and the worker with Docker:

```bash
cd Backend
docker compose up --build
```

`docker-compose.yml` reads `DATABASE_URL_DOCKER` (usually pointing at `host.docker.internal`) and mounts `Frontend/tmp` as the shared evidence folder.

### 3. Frontend

```bash
cd Frontend
cp .env.example .env.local    # fill in DATABASE_URL, SESSION_SECRET, ENCRYPTION_KEY, GOOGLE_*, SERVICE_JWT_SECRET
npm install
npm run dev                   # http://localhost:3000
```

`SERVICE_JWT_SECRET` must be the same in the frontend and the backend. The frontend uses it to sign the short-lived tokens it sends to the API.

## Configuration

Main backend settings (all listed in `Backend/.env.example`):

| Variable | Purpose |
|---|---|
| `DATABASE_URL` | Postgres connection (`postgresql+psycopg://…`) |
| `SERVICE_JWT_SECRET` | Shared secret for frontend → backend auth |
| `LLM_PROVIDER`, `LLM_KEY`, `LLM_BASE_URL` | Model provider (`litellm` by default, or `anthropic`) |
| `LLM_MODEL_AGENT` / `_CLASSIFIER` / `_TITLER` | Optional model override for each role |
| `JIRA_SITE_URL`, `JIRA_EMAIL`, `JIRA_API_TOKEN`, `JIRA_PROJECT_KEY` | Jira connection |
| `JIRA_WEBHOOK_SECRET` | Checks incoming Jira webhooks |
| `EVIDENCE_ROOT` | Folder for uploaded screenshots and recordings |
| `RAG_ENABLED`, `DOCS_PATH`, `DOCS_LANGUAGE`, `EMBEDDING_MODEL` | Docs index settings |
| `RAG_TOP_K`, `SEMANTIC_MAX_DISTANCE`, `LEXICAL_MIN_RANK`, `RAG_DIVERSITY_SLOTS` | Search tuning (re-run the evaluation after changing the docs) |

## API

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Health check |
| `POST` | `/chat/send` | Send a chat message (the reply arrives over SSE) |
| `GET` | `/events` | Server-Sent Events stream |
| `GET` | `/conversations` | List conversations |
| `GET` | `/conversations/{id}/messages` | Get a conversation's messages |
| `POST` | `/conversations/abandon` | Abandon a pending conversation |
| `GET` | `/notifications`, `/notifications/history` | Ticket notifications |
| `POST` | `/notifications/mark-delivered`, `/notifications/mark-all-read` | Update notification status |
| `POST` | `/jira/webhook` | Jira issue updates |

## Testing

```bash
# Backend (skip live tests, which call the real model and cost tokens)
cd Backend && pytest -m "not live"

# Frontend
cd Frontend && npm test && npm run lint
```

## Project structure

```
.
├── Backend/
│   ├── src/app/
│   │   ├── agent/        # agent factory, middleware, tools, prompts
│   │   ├── api/          # FastAPI routers
│   │   ├── rag/          # ingest, embeddings, search, evaluation
│   │   ├── jira/         # Jira client, ADF builder, similar-report search
│   │   ├── worker/       # background ticket jobs
│   │   ├── sse/          # event stream registry and Postgres LISTEN/NOTIFY
│   │   └── db/           # models and repositories
│   ├── skills/           # agent skill playbooks (SKILL.md)
│   ├── docs-corpus/      # raw and formatted documentation
│   ├── docs-formatter/   # MDX → Markdown converter
│   ├── eval/             # retrieval golden set
│   └── tests/
├── Frontend/
│   └── src/
│       ├── app/          # Next.js routes: landing, docs, ask-ai, API proxies
│       └── features/     # auth, chat, docs, dashboard, backend-client
└── Database/
    ├── schema/           # SQL schema files
    └── scripts/          # setup, provision, reset
```
