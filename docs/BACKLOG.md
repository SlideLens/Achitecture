# Backlog and phase plan

Solo evening development with AI assistants (5–10 h/week): a task = 1–2 evenings, sprints of 2 weeks. Modules — per [repository structure](#repository-structure-target) below. Epics = milestones, tasks = issues.

**Critical path:** scaffold → `LLMClient` → ingest → per-slide analysis → **quality metric** (recall/junk). Until core quality hits the target — do not build a web shell around a weak core.

## Phases

| Phase | Content | Estimate |
|---|---|---|
| 0. Foundation | repository, [CLAUDE.md](../CLAUDE.md), Docker (FastAPI+Postgres+LibreOffice), VLM spike on 3 slides, test deck set | ~12 h |
| 1. Pipeline core ⭐ | ingest → analyzers → aggregation → annotation → autofixes → report; **most important**, ~60 % of effort | ~50 h |
| 2. Platform | auth, cabinet, upload, worker, Report page, PDF, landing, deploy + observability | ~35 h |
| 3. Validation and payments | free Review giveaways, feedback, pricing + YooKassa | ~10 h |
| 4. Rehearsal mode | in-browser pitch recording, precise per-slide timing, cross-run dynamics | ~14 h |

## Phase 0 — Foundation

- **Repository scaffold + CLAUDE.md.** Tree from [structure](#repository-structure-target), rule “`backend/core/` does not import `backend/app/`” ([ADR 0001](../adr/0001-pipeline-pure-library.md)), `Finding` contracts, prompt conventions.
- **config.py + database.py.** `Settings` (all ENV with validation), async engine, `get_db()`, `.env.example`.
- **Docker Compose (dev).** app / worker / db / redis; Dockerfile with LibreOffice + ffmpeg + RU fonts; `GET /health`.
- **Observability from day one** ([ADR 0007](../adr/0007-three-layer-observability.md)): structlog (JSON, `review_id` in contextvars), Sentry, Langfuse call wrapper.
- **Spike:** 3 slides through VLM by hand → `docs/spike-notes.md` (what it catches, $/slide, Category taxonomy).
- **Test set:** 10–15 decks (≥5 RU, ≥3 with charts, 2–3 bad); 3 labeled in `backend/tests/golden/`.

## Phase 1 — Pipeline core ⭐

- **`backend/core/schemas.py`** — all contracts (`Finding`, `Category`, `Severity`, `BBox`, `TranscriptSegment`, `DeliveryMetrics`, `ChartReading`, `SuspiciousRegion`, `ReviewResult`). Pydantic only.
- **`LLMClient` + `PromptRegistry` + `ReviewContext`** — vision + structured output, retry, backoff, Langfuse span, cost; prompt files with frontmatter.
- **`DeckIngestor` + `AudioExtractor` + `Transcriber`** — PPTX/PDF → PNG; audio → WAV → faster-whisper; `compute_delivery`.
- **`BaseAnalyzer` + `PipelineOrchestrator` + CLI** — graceful degradation, config-driven order, `python -m core.run`.
- **Analyzers:** `SlideAnalyzer` (v1 prompt) → `ZoomAgent` → `DeckAnalyzer` → `ChartChecker` → `CrossModalAnalyzer` ([ADR 0002](../adr/0002-vlm-pipeline-hybrid-analyzers.md), [ADR 0005](../adr/0005-crossmodal-delivery-analysis.md)).
- **Assembly:** `Aggregator` + `DeckScorer`, `Annotator` (Pillow), `PptxFixer` ([ADR 0006](../adr/0006-pptx-autofix-strategy.md)), `ReportBuilder` + `PdfExporter`.
- **Eval script** `backend/tests/golden/eval.py` → `docs/quality-log.md`. **Phase exit criterion: recall ≥ 70 %, junk < 20 %.**

## Phase 2 — Platform

- ORM models + Alembic (User, Review, FindingRow, FileAsset, Event, empty Rehearsal).
- Auth API (fastapi-users, JWT access+refresh, email verification).
- Services: `Storage`, `EmailService`, `EventTracker`, `LimitService` (SELECT FOR UPDATE, 402).
- `ReviewService` + `backend/worker/tasks.py` ([ADR 0003](../adr/0003-async-review-worker.md)); Reviews/Findings/Files/Events API ([API.md](API.md)).
- Frontend: SPA scaffold, OpenAPI type generation, auth pages, Cabinet + upload, **Report page** (showcase), landing with live example ([DESIGN.md](DESIGN.md)).
- VPS deploy + Grafana/Loki/Prometheus + alerts ([DEPLOY.md](DEPLOY.md)).

## Phase 3 — Validation and monetization

- Give away 30–50 free Reviews (consultants/analysts/founders), collect “what I would pay for.”
- Content marketing: public reviews of well-known presentations.
- Pricing + YooKassa (idempotent webhook), funnel dashboard, grow golden labels from 👎.

## Phase 4 — Rehearsal mode

- Rehearsal page: slide playback + `MediaRecorder` (audio + switch timestamps → precise `SlideTiming`).
- `CrossModalAnalyzer` on precise timing: timing map, “swamp” slides (>2 min) and “stubs” (<5 s), mismatch with timestamps.
- Cross-run dynamics (`attempt_num`) → reason for subscription, not one-off payment.

## Repository structure (target)

Two top-level modules — **`backend/`** and **`frontend/`**, everything else inside them (as in the reference project). Build — one multi-stage `Dockerfile`: stage 1 builds the SPA (`vite build`), stage 2 is the backend image with `dist` placed in `./static`; the same image runs as `app` and as `worker`.

```
slidelens/
├── backend/                  # Python project (uv + pyproject.toml + uv.lock)
│   ├── app/                  # web layer: main.py, config.py, db.py, deps.py, security.py,
│   │                         #   api/v1/ (routes), services/, schemas/, models/, seed.py
│   ├── core/                 # pure library: run.py, context.py, schemas.py, llm/, ingest.py,
│   │                         #   transcribe.py, analyzers/, aggregate.py, annotate.py, fix.py,
│   │                         #   report.py, prompts/ (versioned md)
│   ├── worker/               # tasks.py: process_review, cleanup_expired_files (bridges app and core)
│   ├── observability/        # setup.py: structlog, Sentry, metrics
│   ├── migrations/           # Alembic
│   └── tests/                # unit/ · integration/ · golden/ (decks + labels + eval)
├── frontend/                 # React + Vite + TS SPA
│   └── src/                  # pages/, components/, api/, hooks/, auth/, lib/, App.tsx, main.tsx
├── Dockerfile                # multi-stage: frontend build → backend image (dist → ./static)
├── docker-compose.yml · Makefile · CLAUDE.md · README.md
└── tasks/                    # development tickets (see tasks/README.md)
```

Key rule: `backend/core/` does **not** import `backend/app/` — only `backend/worker/tasks.py` bridges them ([ADR 0001](../adr/0001-pipeline-pure-library.md)). The `app` / `core` / `worker` split lives inside `backend/`, not at the repository top level.
