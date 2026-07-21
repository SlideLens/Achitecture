# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is the **architecture/design documentation** repo for **SlideLens** — a web platform where a user uploads a presentation (Дека, PPTX/PDF), optionally a pitch recording and an Excel of the underlying data, and a multimodal agent returns a senior-designer-level review: annotated slide problems, honesty checks on charts, a "speech ↔ slides" cross-check, and an auto-fixed copy of the file. There is **no application code here yet** — this repo is the spec the (future) implementation must follow. All content is in Russian; keep documentation edits in Russian to match.

There are no build/lint/test commands, except linting the API contract: `redocly lint api/openapi.yaml` (requires the `redocly` CLI), and rendering diagrams/report: `cd report && make diagrams pdf html` (requires `d2`, `pandoc`, headless `chromium` — none needed just to read the docs, since [docs/C4.md](docs/C4.md) has the same diagrams in Mermaid). Everything else is Markdown + embedded Mermaid / d2 diagram sources.

## Document map and reading order

Start from [README.md](README.md), which indexes everything. Read in this order when picking up context:

1. [TASK.md](TASK.md) — original product statement (problem, MVP scope)
2. [CONTEXT.md](CONTEXT.md) — **the glossary — binding.** Canonical terms for people, artifacts, review lifecycle, findings, pipeline, with explicit "avoid these synonyms" notes (e.g. always «Разбор» never «анализ/аудит»; «Находка» never «замечание/issue»; «Дека» never «презентация» as a field name). All other docs and future code identifiers use these terms exactly.
3. [docs/PRD.md](docs/PRD.md) — user stories with acceptance criteria, AI functions (the 10 pipeline steps), data model summary, scope (MVP / phases / out-of-scope), test set, success metrics, risks
4. [docs/C4.md](docs/C4.md) — C1–C3 diagrams + pipeline flow + ERD (replaces classic C4 level 4) + sequence diagram (Mermaid)
5. [api/openapi.yaml](api/openapi.yaml) — REST contract (OpenAPI 3.1, **source of truth**): DTOs, endpoints, error codes, visibility rules. [docs/API.md](docs/API.md) is the human-readable companion — keep both in sync, openapi.yaml wins on conflict.
6. [docs/PROMPTS.md](docs/PROMPTS.md) — analyzer prompts, JSON response schemas, no-LLM fallback behavior
7. [docs/DEPLOY.md](docs/DEPLOY.md) — VPS, Docker Compose, Caddy single-domain, CI/CD, privacy/auto-deletion
8. [docs/DESIGN.md](docs/DESIGN.md) — UI design system and screens (the Report page is the product's storefront)
9. [docs/BACKLOG.md](docs/BACKLOG.md) — phase 0–4 plan and target repository structure
10. [adr/](adr/) — accepted architectural decisions (see below)

## Core domain model (don't relitigate without an ADR)

- **Review core is a pure library.** `backend/core/` never imports `backend/app/` and never touches the DB directly; `backend/worker/tasks.py` is the only thing that wires them together. This lets the core be developed and tested from the CLI through all of phase 1, without web/DB. Locked by [ADR 0001](adr/0001-pipeline-pure-library.md). Do not "simplify" by importing an ORM session into an analyzer.
- **Analysis is multi-analyzer with graceful degradation.** Independent analyzers over a common `BaseAnalyzer` (`SlideAnalyzer`, `ZoomAgent`, `DeckAnalyzer`, `ChartChecker`, `CrossModalAnalyzer`), each with its own versioned prompt. A **failing analyzer is skipped and logged, the Review continues** — a partial report beats `failed`. The `ZoomAgent` is a deliberate two-phase step (cheap screening → crop ×2 → analyze big), capped at 3 zooms/slide for cost. Locked by [ADR 0002](adr/0002-vlm-pipeline-hybrid-analyzers.md).
- **All VLM calls go through a single `LLMClient`.** Nobody imports `anthropic` directly — this keeps structured-output parsing, retries, Langfuse tracing and cost accounting in one place. Prompts live as versioned md files in `backend/core/prompts/` (frontmatter `version`, `tier`); the version is attached to the Langfuse trace.
- **Review runs async.** A Review takes 2–5 minutes → `POST /reviews` accepts the file and returns `202 queued`; a background worker (ARQ + Redis, from the start) runs the pipeline; the frontend polls status; email on `done`. Not a blocking HTTP request. Locked by [ADR 0003](adr/0003-async-review-worker.md).
- **Finding taxonomy is fixed and shared.** `Category` (`TYPOGRAPHY | HIERARCHY | READABILITY | CONSISTENCY | CHART | NARRATIVE | SPEECH_MISMATCH | DELIVERY`) and `Severity` (`CRITICAL | MAJOR | MINOR`) are single-source-of-truth enums used by core, DB, API and UI. `bbox` is normalized `0..1` (dpi-independent). `Finding` (pydantic in `backend/core/`) is mirrored by `FindingRow` (ORM in `backend/app/`); the converter lives in one place.
- **Auto-fix is a narrow, safe strategy set.** `PptxFixer` applies only `MinFontSizeRule` / `ContrastRule` / `AlignmentRule` to `auto_fixable` findings on a copy of the deck, with a control re-render afterward. No slide re-layout. Locked by [ADR 0006](adr/0006-pptx-autofix-strategy.md).
- **Cost per Review is a first-class metric.** Traced in Langfuse from day one; there is an alert on a threshold. Observability is three separate layers (product analytics `Event` table / Langfuse LLM-obs / Sentry+Grafana+Loki+Prometheus). Locked by [ADR 0007](adr/0007-three-layer-observability.md).

## ADRs — read before proposing architectural changes

- [0001](adr/0001-pipeline-pure-library.md) — `backend/core/` is a pure library, isolated from `backend/app/` and the DB
- [0002](adr/0002-vlm-pipeline-hybrid-analyzers.md) — multi-stage VLM pipeline of independent analyzers + zoom-agent, graceful per-analyzer failure, structured JSON via a single `LLMClient`
- [0003](adr/0003-async-review-worker.md) — asynchronous background worker, not a synchronous HTTP request
- [0004](adr/0004-stack-fastapi-react.md) — FastAPI JSON API + React/Vite/TS SPA, LibreOffice+pdf2image rendering, PostgreSQL + SQLAlchemy 2.0 + Alembic, JWT (fastapi-users), single-domain Caddy
- [0005](adr/0005-crossmodal-delivery-analysis.md) — cross-modal speech↔slides check + delivery metrics via faster-whisper; rehearsal mode is phase 4
- [0006](adr/0006-pptx-autofix-strategy.md) — PPTX auto-fix as a strategy pattern, minimal reliable rule set
- [0007](adr/0007-three-layer-observability.md) — three-layer observability; cost-per-Review is the key unit-economics metric

When a new architectural decision is made, add a new numbered ADR under `adr/` (`NNNN-slug.md`, don't edit past ones — supersede them) and cross-link it from README.md's ADR list and from the relevant docs.

## Repository layout (two top-level modules)

The repo has exactly **two top-level modules — `backend/` and `frontend/`** — and everything else lives inside them (same shape as the reference project):

- **`backend/`** — the Python project (uv + `pyproject.toml` + `uv.lock`). Inside it: `app/` (web layer: `main.py`, `config.py`, `db.py`, `deps.py`, `security.py`, `api/v1/`, `services/`, `schemas/`, `models/`, `seed.py`), `core/` (pure library — analyzers & review run), `worker/` (`tasks.py` — the only bridge between `app` and `core`), `observability/`, `migrations/` (Alembic), `tests/` (`unit/`, `integration/`, `golden/`). The `app` / `core` / `worker` split (ADR 0001) lives **inside** `backend/`, not at the repo root. Python import names stay `app.*`, `core.*`, `worker.*` (the package root is `backend/`); only filesystem paths carry the `backend/` prefix.
- **`frontend/`** — the React/Vite/TS SPA (`src/` with `pages/`, `components/`, `api/`, `hooks/`, `auth/`, `lib/`).
- **Repo root** — `Dockerfile`, `docker-compose.yml`, `Makefile`, `CLAUDE.md`, `README.md`, and `tasks/` (dev tickets).

Full tree — [docs/BACKLOG.md](docs/BACKLOG.md#структура-репозитория-целевая).

## Planned implementation stack (for when code is added)

FastAPI (clean JSON API `/api/v1`, JWT bearer via fastapi-users) + React/Vite/TS/Tailwind/TanStack Query SPA (types generated from OpenAPI, never hand-written). PostgreSQL + SQLAlchemy 2.0 + Alembic migrations. Background worker: **ARQ + Redis from the start** (separate process, same image as `app` — ADR 0003). Rendering: LibreOffice headless + pdf2image (RU fonts baked into the image — critical). VLM: Claude API (vision) via `LLMClient` only. faster-whisper (transcription), python-pptx (auto-fix), Pillow (annotations), weasyprint (PDF report). Single multi-stage Docker image: stage 1 `vite build` → `frontend/dist`, stage 2 the `backend/` image (uv) with `dist` baked into `./static` and served by FastAPI; the same image runs as `app` and as `worker`. Docker Compose on a VPS, Caddy on one domain terminates TLS and proxies to the app (no CORS, no nginx). Target repo layout and phase plan — [docs/BACKLOG.md](docs/BACKLOG.md).

## Editing conventions

- Keep terminology consistent with [CONTEXT.md](CONTEXT.md) exactly — it's the single source of truth for names used in both docs and future code identifiers.
- Diagrams come in two synced forms: **Mermaid embedded in [docs/C4.md](docs/C4.md)** (render-free, the primary read) and **[docs/diagrams/](docs/diagrams/) d2 sources** rendered into `docs/images/*.svg` for the [report/](report/) PDF. When you change one, change the other. Rendered images are generated (`make -C report diagrams`), not committed.
- DTO/endpoint shapes in [api/openapi.yaml](api/openapi.yaml) (source of truth) and its prose companion [docs/API.md](docs/API.md) must stay in sync with each other, with the ERD in [docs/C4.md](docs/C4.md) / [docs/diagrams/er_diagram.d2](docs/diagrams/er_diagram.d2), and with the data-model summary in [docs/PRD.md](docs/PRD.md) — when changing one, check the others.
- The `Category`/`Severity` enums and the `Finding` shape are the contract seam between core, DB and API; change them in all four places (PROMPTS, openapi, C4/ERD, PRD) together.
