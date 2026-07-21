# ADR 0001: Review pipeline — pure library, isolated from web and DB

**Status:** Accepted
**Date:** 5 June 2026
**Decision context:** repository structure, module boundaries

## 1. Context

The product core is the Review pipeline (ingest → analyzers → aggregation → report). It is the most complex and valuable part: its quality is iterated for weeks (prompts, zoom agent, data reconciliation). At the same time we need a web layer (auth, upload, cabinet, Report) and a database. The temptation is to write analyzers directly in FastAPI routes/services, calling ORM models.

Problems with that coupling:
1. You cannot run and debug the pipeline without a running web stack and DB — yet phase 1 (3–4 weeks) is entirely about core quality, before the platform exists.
2. Analyzer unit tests pull in backend/app/DB/sessions.
3. VLM calls spread across the codebase; cost and tracing are lost.

## 2. Decision

Extract the pipeline into a separate package `backend/core/` as a **pure library** with hard boundaries:

1. **Zero imports from `backend/app/`.** Pipeline input — file paths + `ReviewContext`; output — `list[Finding]` + on-disk artifacts. Only `backend/worker/tasks.py` connects `backend/core/` and `backend/app/`.
2. **`ReviewContext`** — the single object flowing through every step: paths (workdir, deck, audio, xlsx), results (slide_pngs, transcript, excel_data, findings), metadata (review_id, cost counter). Serialized to JSON (`ctx.dump()`) for debugging.
3. **CLI as the phase-1 interface:** `python -m core.run --deck x.pptx [--audio y.mp4] [--data z.xlsx] --out dir/` — the full pipeline runs from the terminal with no web layer.
4. **Single VLM entry point** — `LLMClient` (see [ADR 0002](0002-vlm-pipeline-hybrid-analyzers.md)); `anthropic` is never imported directly in application code.
5. The web layer (`backend/app/`) knows about DB and HTTP, but **not** about analysis details; it only accepts files, enqueues work, and reads results.

## 3. Consequences

### Positive
- The core is developed and tested throughout phase 1 from the CLI, without web or DB — a fast prompt-iteration loop.
- Analyzer unit tests do not need backend/app/DB (only `LLMClient` is mocked).
- The boundary between “happy path ↔ errors/cost” is visible in one place.
- The pipeline is portable: the same code drives golden-eval, CLI, and the production worker.

### Negative and risks
- Contract duplication: `Finding` (pydantic in the pipeline) and `FindingRow` (ORM in app) — two models for one entity. *Mitigation:* fields mirror one-to-one; conversion lives in one place (`ReviewService`), covered by a test.
- Temptation to “cut corners” and import a DB session into an analyzer for convenience. *Mitigation:* the rule is fixed in [CLAUDE.md](../CLAUDE.md) and checked in review (grep `from app` under `backend/core/`).
