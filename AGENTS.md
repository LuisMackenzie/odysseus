# AGENTS.md — Odysseus

Quick-ramp context for AI agents working in this repo. Every line answers: "would an agent miss this without help?"

---

## Architecture

```
app.py          FastAPI entry point (slim orchestrator; loads dotenv, registers routes)
core/           auth, database, middleware, session_manager — re-exports from src/ where noted
src/            business logic: llm_core, agent_loop, agent_tools, chat_processor, constants, …
routes/         HTTP route handlers (one file per feature area)
services/       heavier service modules (research, search, memory, hwfit/Cookbook, …)
static/         index.html + app.js + style.css + static/js/ (modular front-end; no build step)
tests/          flat pytest suite (~500+ files); phased restructure in progress
scripts/        CLI tools (odysseus-*, odysseus-backup, update_database.py, …)
mcp_servers/    built-in MCP servers (email, image_gen, memory, rag)
companion/      companion-mode pairing routes
```

`core/constants.py` is a **shim** that re-exports everything from `src/constants.py`. The single source of truth for all paths and config is **`src/constants.py`**. Import from either; never define a second copy.

---

## Developer commands

**Setup (Linux/macOS native):**
```bash
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
python setup.py          # creates data/, admin account, first-boot config
python -m uvicorn app:app --host 127.0.0.1 --port 7000
```
macOS M-series: use `./start-macos.sh` instead (sets up Homebrew deps, starts on port 7860).

**Docker (recommended for testing):**
```bash
cp .env.example .env
docker compose up -d --build
docker compose logs --tail=120 odysseus
```

**Syntax checks (what CI runs):**
```bash
python -m compileall -q app.py core routes src services scripts tests
node --check static/js/<file>.js         # skip static/lib/ (vendored)
```

**Run tests:**
```bash
# Always use the venv interpreter — system python3 may lack pinned deps (e.g. nh3)
mkdir -p data                            # sqlite DB expects ./data/ to exist
.venv/bin/python -m pytest               # or: source venv/bin/activate && python -m pytest
```

**Focused test runs (preferred over full suite):**
```bash
python3 tests/run_focus.py --area security
python3 tests/run_focus.py --area services --sub-area cookbook
python3 tests/run_focus.py --fast                         # excludes slow-marked tests
python3 tests/run_focus.py --last-failed
python3 tests/run_focus.py --area services -- --maxfail=1 -q

# Or directly with pytest markers:
python -m pytest -m area_security
python -m pytest -m "area_services and sub_cookbook"
```

Areas: `security`, `routes`, `services`, `cli`, `js`, `helpers`, `unit`, `uncategorized`.

---

## Testing quirks

- **pytest is `continue-on-error: true` in CI** — the suite is informational, not a hard gate yet. A red run does not block merge; tracked under ROADMAP "fresh install smoke tests."
- **`mkdir -p data` before running locally** — the in-memory SQLite default in `conftest.py` handles most tests, but some need `./data/app.db` on disk.
- **JS tests skip when `node` is absent** — treat a skip as a coverage gap, not a pass.
- **No `sys.modules` mutation at module scope** — use `monkeypatch.setitem` or the helpers in `tests/helpers/import_state.py`. Restoring `sys.modules` alone is insufficient; parent-package attributes must be restored too.
- **`slow` marker is opt-in and evidence-driven** — only mark a test slow with `--durations` output, not by guessing.
- **Taxonomy markers** are added automatically at collection time by `tests/conftest.py` (via `tests/_taxonomy.py`) — you never need to add `area_*`/`sub_*` markers manually.

---

## Code conventions

**Paths — never hardcode:**
- Every persisted file or directory has a named constant in `src/constants.py` (e.g. `AUTH_FILE`, `CHROMA_DIR`, `TTS_CACHE_DIR`). Import and use that constant.
- `DATA_DIR` is the only place that reads `ODYSSEUS_DATA_DIR`. Use it only for dynamic paths with no fixed name (e.g. per-owner files).
- Never build paths from `Path(__file__)`, hardcode `/app/…`, or use `"data/..."` relative strings.
- If a new data file or directory has no constant yet, add one to `src/constants.py`.

**Internal loopback URLs:**
- Never hardcode `http://localhost:7000`. Use `internal_api_base()` from `src.constants` — it honors `ODYSSEUS_INTERNAL_BASE` / `APP_PORT`.

**Commit style:** Conventional Commits — `type(scope): summary` (e.g. `fix(search): …`, `feat(notes): …`). Keep subject short and imperative.

---

## UI/visual conventions (enforced in review)

- **No Unicode emoji** anywhere in UI or code. Use inline SVG matching the monochrome icon style in `static/index.html`.
- CSS variables only: `--red`, `--fg`, `--bg`, `--card`, `--border`, etc. Do not introduce new color values, font sizes, or spacing units.
- Primary font: `Fira Code` (monospaced). Do not override.
- Dark theme is the default; light-mode work must go through the existing theme system.
- Reuse existing button/input/card/border classes — do not write parallel components.
- UI PRs require a screenshot or clip of the change running in the app. PRs that change rendering without one are closed.

---

## Branch and PR rules

- **All PRs target `dev`**, not `main`. `main` is curated by the maintainer at each release.
- Link every PR to an issue (`Fixes #NNN` or `Part of #NNN`).
- PRs must include manual test steps from running the actual app — unit test results alone are not enough.
- Keep PRs small and single-purpose: no mixing bug fixes, refactors, formatting, and feature work in one PR.
- **LLM agent PRs**: open an issue describing the problem first. Bulk auto-generated PRs that don't match the project's visual style are closed on sight.

---

## Optional dependencies

`requirements-optional.txt` is not installed by default. It unlocks:
- `faster-whisper` — local STT
- `duckduckgo-search` — DDG search provider
- `PyMuPDF` (AGPL) — PDF rendering/forms
- `markitdown` — Office/EPUB text extraction

Docker: add `--build-arg INSTALL_OPTIONAL=true` to include these in the image.

---

## Common gotchas

- **`chromadb-client` conflict**: if the lightweight HTTP-only package is installed alongside full `chromadb`, memory silently degrades. Fix: `pip uninstall chromadb-client -y && pip install --force-reinstall chromadb`.
- **macOS port**: `start-macos.sh` uses port `7860` (not 7000) because AirPlay holds 7000.
- **`data/` is gitignored** — never commit anything under `data/`. Contains `app.db`, `memory.json`, uploads, chroma embeddings, auth state, API keys, etc.
- **Docker GPU**: `nvidia-smi` passing inside the container confirms Docker passthrough, not a CUDA-enabled llama.cpp build. Reinstall the serve engine via Cookbook → Dependencies if CUDA tensor layers are missing.
- **Ollama with Docker**: use `http://host.docker.internal:11434/v1` as the endpoint, and start Ollama with `OLLAMA_HOST=0.0.0.0:11434 ollama serve`.
