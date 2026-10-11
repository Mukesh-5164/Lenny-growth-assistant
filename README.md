# The Lenny Growth Assistant

A grounded, conversational assistant over **Lenny's Podcast transcripts** (302 episodes) for product and growth teams.
It answers questions with citations, writes **Ship 30 for 30-style essays** from what guests actually said, and generates
**Markdown / HTML artifacts** that render beside the chat in a sandboxed viewer. Runs fully locally on **Ollama**; can use
**Anthropic Claude** as a cloud model. One toggle in the UI.

| Doc | What it covers |
|---|---|
| [docs/PRD.md](docs/PRD.md) | Discovery brief, scope, flows, acceptance criteria, risks, plan |
| [docs/architecture.md](docs/architecture.md) | Schema, API, components, ingestion/retrieval, routing, security, topology |
| [docs/design.md](docs/design.md) | UX principles, states, responsive + accessibility decisions |
| [docs/manual-test-plan.md](docs/manual-test-plan.md) | UI test script for evaluators |
| [agent_transcripts/](agent_transcripts/) | Build log incl. failures and corrections |

## Quick start (about 5 minutes + first-run ingest)

**Prerequisites:** Docker with Compose v2, [Ollama](https://ollama.com), ~6 GB free RAM for the default 7B model.

```bash
ollama pull qwen2.5:7b          # or a smaller model, see "Choosing a local model"
# Linux only: let containers reach Ollama ->  OLLAMA_HOST=0.0.0.0 ollama serve
docker compose up --build       # or: make up
open http://localhost:8000
```

That's it, no `.env` needed. On first start the app clones the public transcript repo and builds the index in the
background (~1 minute; progress in the sidebar and `GET /health/ready`). Questions asked before it finishes get a clear
`KB_EMPTY` message, not a crash.

## Choosing the model (no code changes)

Everything is environment-driven (see `.env.example`). Copy it to `.env` and edit.

| Goal | Settings |
|---|---|
| Local demo (default) | `LLM_PROVIDER=ollama`, `OLLAMA_MODEL=qwen2.5:7b` |
| Lighter machine | `OLLAMA_MODEL=llama3.2:3b` (faster; weaker essays) |
| Cloud | `ANTHROPIC_API_KEY=sk-ant-...`, `LLM_PROVIDER=anthropic` |
| Cloud as safety net | keep `LLM_PROVIDER=ollama`, add the key and `FALLBACK_PROVIDER=anthropic` |

The provider selector in the header switches the model per chat message; its dot shows whether the provider is reachable.
**Fallback behaviour:** if the selected provider errors or times out and `FALLBACK_PROVIDER` is set (and differs), the request
is retried once on the fallback and the message is badged "fallback used". With no fallback the user sees a structured error
with a Retry button. Missing keys, an unpulled model, and an unreachable Ollama each produce a specific actionable message.

## Environment variables

| Variable | Default | Notes |
|---|---|---|
| `DATABASE_URL` | set by compose | PostgreSQL (`postgresql+psycopg://…`); Supabase/Railway URLs work as-is |
| `LLM_PROVIDER` | `ollama` | `ollama` or `anthropic` |
| `FALLBACK_PROVIDER` | empty | optional second provider |
| `OLLAMA_BASE_URL` / `OLLAMA_MODEL` / `OLLAMA_NUM_CTX` | `http://host.docker.internal:11434` / `qwen2.5:7b` / `8192` | |
| `ANTHROPIC_API_KEY` / `ANTHROPIC_MODEL` | empty / `claude-sonnet-5-5` | optional |
| `LLM_TIMEOUT_S` | `180` | per model call |
| `AUTO_INGEST`, `KB_PATH`, `KB_REPO_URL` | `true`, … | first-run clone + index |
| `RETRIEVAL_TOP_K`, `MIN_QUERY_COVERAGE` | `6`, `0.5` | evidence gate (see architecture.md) |
| `ADMIN_TOKEN` | empty | enables `POST /api/kb/reload` |
| `LOG_LEVEL` | `INFO` | JSON logs to stdout |

## Using it

* Ask a question → grounded answer with `[1]` citations; click a citation or source chip for the excerpt, timestamp and a deep link to the episode moment.
* "Write a Ship 30 for 30 essay about pricing experiments" → ~1,250-word essay opens in the Artifact Viewer.
* "Create an HTML one-pager summarising that" / "make a markdown checklist" → artifact beside the chat (Preview / Source, Copy, Download).
* Anything the transcripts don't support gets an explicit "not in the transcripts" reply instead of a guess.

## Without Docker

```bash
docker compose up -d db                       # just Postgres (add `ports: ["5432:5432"]`), or use any Postgres/Supabase URL
cd backend && python -m venv .venv && . .venv/bin/activate && pip install -r requirements.txt
export DATABASE_URL=postgresql+psycopg://lenny:lenny@localhost:5432/lenny OLLAMA_BASE_URL=http://localhost:11434 KB_PATH=../data/lennys-podcast-transcripts/episodes
python -m app.kb.ingest --fetch               # optional; the server also does this on first start
uvicorn app.main:create_app --factory --reload
```

## Tests

```bash
make test        # or: cd backend && pip install -r requirements-dev.txt && python -m pytest -q
```
48 automated tests (SQLite + fake LLM, no Docker/Ollama needed) cover chunking/parsing (three transcript formats), BM25 and the
evidence gate, routing, artifact parsing, HTML/Markdown sanitisation, session independence and user scoping, persistence across
app restarts, provider failure/timeout/fallback, essay length repair, empty KB, and database failure. UI checks: `docs/manual-test-plan.md`.

## API (OpenAPI at `/docs`)

`GET /health` · `GET /health/ready` · `GET /api/config` · `POST/GET /api/sessions` · `GET/DELETE /api/sessions/{id}` ·
`POST /api/sessions/{id}/messages` · `GET /api/artifacts/{id}` · `GET /api/artifacts/{id}/download` · `GET /api/search?q=` · `POST /api/kb/reload`.
All session/artifact calls need an `X-User-Id` header (a pseudonymous browser id). Errors: `{"error":{"code","message","details","request_id"}}`.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `LLM_UNAVAILABLE: Cannot reach Ollama` | Start `ollama serve`. In Docker on Linux, bind it to `0.0.0.0` (`OLLAMA_HOST=0.0.0.0`). |
| `LLM_NOT_CONFIGURED: model … not installed` | `ollama pull <OLLAMA_MODEL>` |
| `LLM_TIMEOUT` | Smaller model, or raise `LLM_TIMEOUT_S`; first call loads the model into memory. |
| `KB_EMPTY` / sidebar says "ingesting" | Wait for first-run ingest; check `docker compose logs app`; or `make ingest`. No internet → clone the repo into `./data` manually. |
| `DB_UNAVAILABLE` | `docker compose ps`; the app retries DB connection at boot and `/health/ready` shows detail. |
| Essay outside 1,000–1,500 words | Small local models drift; the app does one repair pass and shows the actual count. Use a larger model. |
| Anything else | Every response carries `X-Request-ID`; `docker compose logs app \| grep <id>` shows retrieval (query, coverage), model (provider, ms, tokens) and artifact events. |

## Known limitations (read before trusting it)

* The evidence gate is lexical (BM25 + IDF-weighted term coverage). It reliably catches clear misses; borderline off-topic questions
  rely on the model returning `NOT_SUPPORTED`, which small local models follow imperfectly. A golden-set eval is the first next step.
* Retrieval is keyword-based, so paraphrase-heavy questions can miss. Adding embeddings (hybrid) is the planned upgrade.
* Not verified in this build environment: Docker image build, real PostgreSQL, a real Ollama model, the Anthropic API and a real browser.
  See `agent_transcripts/BUILD_LOG.md` for exactly what was verified and how.
* 1 of 303 transcripts (`ryan-hoover`) has no parsable speaker turns and is skipped (logged as a warning).

## Extending

New skill: add `backend/app/skills/<name>/SKILL.md` (frontmatter config + instructions), a branch in `agent/router.py`, and a
method in `agent/orchestrator.py`. New provider: implement `chat()`/`status()` in `llm/` and register it in `llm/registry.py`.
