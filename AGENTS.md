# AGENTS.md

## Cursor Cloud specific instructions

Durable, non-obvious notes for working on this repo in a Cloud Agent. Standard
commands live in [`README.md`](README.md); this section only records the gotchas.

### Toolchain

- Python must be **3.13**. The repo requires `>=3.13` (see `pyproject.toml`) and
  CI pins 3.13, but `uv` on this VM defaults to the newest interpreter it finds
  (3.14), which pulls in bleeding-edge tool versions. The startup update script
  therefore syncs the venv with `--python 3.13`. When invoking uv manually, run
  `uv sync --python 3.13 ...` / `uv run --python 3.13 ...` if the venv was not
  already created on 3.13.
- `uv` is installed at `~/.local/bin/uv` and symlinked into `/usr/local/bin` so
  non-interactive shells (and the update script) find it without a profile edit.
- `pnpm` (9.15.4) is the JS package manager. Only `apps/web` is a pnpm workspace
  member; root `package.json` just holds orchestration scripts.

### Services (run in separate terminals)

- API — `pnpm api:dev` (FastAPI + uvicorn `--reload`, http://localhost:8000).
  Endpoints under `/api/v1` plus `/health`. Set `FANTASY_DRAFT_CORS_ORIGINS`
  (see `.env.example`) if serving the web app from a non-default origin.
- Web — `pnpm dev` (Vite PWA, http://localhost:5173). In dev the API client
  auto-targets `http://localhost:8000` (`import.meta.env.DEV`), so the app runs
  in **"Cloud DVS"** engine mode when the API is up. Override with
  `VITE_API_BASE_URL` in `apps/web/.env.local`.
- `pnpm dev` uses esbuild and does **not** type-check, so it starts even when
  `tsc` would fail (see below).

### Known-broken on `master` (not caused by your changes)

The `Add DVS formula v2` commit bumped the engine to `0.2.0` and added
`marginalValue`/`waitLoss` to the score breakdown, but left several call sites,
tests, and types stale. As of environment setup the following fail on a clean
`master`; verify against your base branch before assuming a change is yours:

- `uv run pytest` — `apps/api/tests/test_api.py::test_health_and_version` expects
  engine `0.1.0` but the code reports `0.2.0` (53 pass, 1 fails).
- `uv run ruff check .` — 15 import-ordering errors (mostly in tests).
- `uv run mypy packages/dvs-engine/src apps/api/src` — 1 error in `csv_import.py`.
- `pnpm build` — `tsc` fails because `apps/web/src/engine/fallback.ts` omits the
  new `marginalValue`/`waitLoss` breakdown fields. This also blocks
  `pnpm test:e2e`, whose Playwright `webServer` runs `npm run build` first.

Clean and passing today: `pnpm lint`, `pnpm test` (vitest), the Python unit
tests except the one above, and both dev servers + the online recommendation
flow.

### Offline (Pyodide) path

The offline engine (`apps/web/src/workers`, Pyodide) needs a built wheel at
`apps/web/public/engine/` (run `pnpm build:engine`) plus a cached Pyodide
distribution; see [`docs/offline-readiness.md`](docs/offline-readiness.md). In
the Vite dev server the app shows an "Offline Python engine is not ready" banner
(a worker `importScripts`/module-worker limitation) — this is expected in dev
and does **not** affect the online "Cloud DVS" path.
