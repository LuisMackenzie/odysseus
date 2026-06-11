# AGENTS.md

Context for AI coding agents working in this repo.

---

## What this is

A self-hosted AI workspace: **Python 3.11+ FastAPI** backend + **vanilla JavaScript SPA** frontend.
Despite living in `IdeaProjects/`, this is **not** a Java/Kotlin/Android project.
No build step, no bundler, no type checker, no formatter.

---

## Run

```bash
# macOS (avoids AirPlay conflict on port 7000, auto-installs deps)
./start-macos.sh

# Other platforms (native)
python -m uvicorn app:app --host 127.0.0.1 --port 7000

# Docker (recommended for production / full stack)
docker compose up -d --build
```

Ancillary services (ChromaDB port 8100, SearXNG port 8080, ntfy port 8091) are defined in
`docker-compose.yml`. The app degrades gracefully when they are unreachable.

---

## Test

```bash
# Full suite
python -m pytest

# By taxonomy area (preferred for focused work)
python tests/run_focus.py --area security
python tests/run_focus.py --area services --sub-area cookbook

# Skip slow tests
python tests/run_focus.py --fast
python tests/run_focus.py --area routes --fast

# Single file or keyword
python -m pytest tests/test_agent_loop.py
python -m pytest -k "keyword"

# Most-recently-failed only
python tests/run_focus.py --last-failed

# Dry-run (print pytest command without running)
python tests/run_focus.py --area services --dry-run
```

Known taxonomy areas: `security`, `routes`, `services`, `cli`, `js`, `helpers`, `unit`, `uncategorized`.

Test env defaults: `DATABASE_URL=sqlite:///:memory:` (set by `tests/conftest.py`). No external
services required for most tests. Node.js tests (wrapped `*_js.py` files and `tests/streaming/*.test.mjs`)
skip cleanly when `node` is not on PATH.

CI runs pytest with `continue-on-error: true` — failures do not block the build.

---

## Lint / Syntax Check

No linter or formatter is configured. CI only runs a syntax check:

```bash
# Python (matches CI)
python -m compileall -q app.py core routes src services scripts tests

# JS (matches CI — check key files after edits)
node --check static/app.js
node --check static/js/<changed-file>.js
```

Run these before committing. There is no ruff, black, mypy, eslint, or prettier.

---

## Architecture

```
app.py              — FastAPI app: all middleware, all routers mounted, lifecycle hooks
setup.py            — First-run only: creates dirs, initialises DB, creates admin user
core/               — Cross-cutting infrastructure (DB engine, auth, middleware, exceptions)
src/                — Business logic (~93 files: LLM core, agent loop, search, memory, config, …)
routes/             — One file per HTTP feature area (chat, sessions, memory, email, …)
services/           — Service-layer abstractions (memory, search, TTS, STT, research, docs, …)
static/             — Vanilla JS SPA: index.html, app.js, js/ (~84 ES modules), style.css
tests/              — ~470+ test files; taxonomy auto-assigned from filenames
scripts/            — Git-style CLI (`scripts/odysseus` dispatcher → `scripts/odysseus-*`)
mcp_servers/        — Built-in MCP servers (email, image_gen, memory, rag)
```

**Wiring**: Managers are instantiated in `src/app_initializer.py`, then passed as arguments to route
setup functions in `app.py`. No DI container.

**SSE streaming**: Chat, agent, shell, and research responses use `text/event-stream`.

**Agent loop**: `src/agent_loop.py` (~2961 lines) — multi-round tool execution with fenced code blocks
as tool-call syntax.

**LLM core**: `src/llm_core.py` (~2164 lines) — streaming, caching, dead-host cooldown.

---

## Key conventions

### Paths and constants

`src/constants.py` is the **single source of truth** for all paths, directories, and named constants.
`core/constants.py` just re-exports it. **Never hardcode paths** (`data/...`, `/app/...`).

For internal HTTP calls, use `internal_api_base()` from `src/constants.py` — never `http://localhost:7000`.

### Database

SQLite by default (`DATABASE_URL` env var accepts any SQLAlchemy dialect). Schema is created by
`Base.metadata.create_all()` in `setup.py`. **No Alembic.** For schema changes, update
`core/database.py` and add a migration step to `scripts/update_database.py`.

### Frontend

Zero-build SPA — raw ES modules served directly. No transpilation. Cache-busting is handled via
`no-cache` headers and runtime nonces. Do not add a bundler or build step.

CSS: CSS variables only (`--red`, `--fg`, `--bg`, …). Dark theme by default. No emoji in
UI — use inline SVG. Fira Code is the primary font.

### Optional dependencies

`requirements-optional.txt` (faster-whisper, PyMuPDF-AGPL, ddgs, markitdown) is **not** installed
by default. Tests that touch these must guard with `pytest.importorskip` or a try/except import.

### Testing policy

See `tests/TESTING_STANDARD.md` (authoritative). Short version:
- Tests must be deterministic and isolated — no shared state mutation.
- Use `monkeypatch` for `os.environ`; use `tests/helpers/import_state.py` for `sys.modules` cleanup.
- Use `tests/helpers/sqlite_db.py` for a file-backed SQLite fixture.
- `tests/helpers/db_stubs.py` provides fake DB modules.
- Slow tests are opt-in only (from duration evidence, not guessing).
- `conftest.py` is intentionally minimal; do not add global fixtures there without good reason.

---

## Branch and PR model

- `dev` — default branch, unstable; **all PRs target `dev`**.
- `main` — curated, stable; periodically promoted from `dev`.
- Commit style: Conventional Commits (`type(scope): summary`).
- PR scope: small and homogeneous — do not mix refactors with behavior changes.
- Auto-generated PRs require a prior issue describing the problem.
- UI PRs require screenshots.

---

## First-time setup (not for tests)

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python setup.py   # creates data dirs, DB, admin user
```

---

## Reference docs in this repo

- `CONTRIBUTING.md` — full contributor guide including setup, pre-PR checks, PR expectations
- `tests/TESTING_STANDARD.md` — formal testing policy
- `tests/README.md` — test helper usage reference
- `.env.example` — all environment variables documented
