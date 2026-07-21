# SlideLens — architecture and documentation

**SlideLens** is a web platform where a user uploads a presentation (Deck, PPTX/PDF), optionally a pitch Recording and an Excel file with data, and a multimodal agent returns a **senior-designer-level Review**: annotated slide problems, honesty checks on charts, a "speech ↔ slides" cross-check, and an **auto-fixed version of the file**.

This repository is an **architecture specification** (there is no application code here yet): implementation is written against it. All documentation is in English; keep edits in English and strictly follow the terms in [CONTEXT.md](CONTEXT.md).

## What the product does

- **Slide analysis** — typography, visual hierarchy, readability; zoom into small text and charts.
- **Chart honesty checks** — truncated axes, caption vs. data, reconciliation against attached Excel.
- **Cross-modality** — "what the speaker says ↔ what is on the slide" plus Delivery metrics (pace, pauses, filler words).
- **Auto-fix** — Fixed deck (PPTX) with safe edits (font/contrast/alignment) plus a PDF report.

## Reading order

1. [TASK.md](TASK.md) — original product statement (problem, what MVP can do).
2. [CONTEXT.md](CONTEXT.md) — **glossary, shared language — required.** Canonical terms (Review, Deck, Finding, Category…) with "avoid these synonyms" notes. All other documents and code use them strictly.
3. [docs/PRD.md](docs/PRD.md) — user stories with acceptance criteria, AI features, scope (MVP / phases), risks, metrics.
4. [docs/C4.md](docs/C4.md) — C1–C3 diagrams + pipeline + ERD + sequence (Mermaid, readable on GitHub).
5. [api/openapi.yaml](api/openapi.yaml) — REST contract (**source of truth**); prose companion — [docs/API.md](docs/API.md) (keep in sync; on conflict, openapi.yaml wins).
6. [docs/PROMPTS.md](docs/PROMPTS.md) — analyzer prompts, JSON schemas, no-LLM fallback behavior.
7. [docs/DEPLOY.md](docs/DEPLOY.md) — VPS, Docker Compose, Caddy, CI/CD, privacy.
8. [docs/DESIGN.md](docs/DESIGN.md) — design system and screens (the Report page is the storefront).
9. [docs/BACKLOG.md](docs/BACKLOG.md) — phase 0–4 plan and target repository structure.
10. [adr/](adr/) — accepted architectural decisions (below).

## Architectural decisions (ADR)

- [0001](adr/0001-pipeline-pure-library.md) — pipeline in `backend/core/` as a pure library, isolated from `backend/app/` and the DB.
- [0002](adr/0002-vlm-pipeline-hybrid-analyzers.md) — multi-stage VLM pipeline of independent analyzers + zoom agent, graceful degradation.
- [0003](adr/0003-async-review-worker.md) — asynchronous background worker (a Review takes minutes, not an HTTP request).
- [0004](adr/0004-stack-fastapi-react.md) — FastAPI + React SPA stack, rendering via LibreOffice, single Caddy domain.
- [0005](adr/0005-crossmodal-delivery-analysis.md) — cross-modal check and Delivery metrics; rehearsal is phase 4.
- [0006](adr/0006-pptx-autofix-strategy.md) — PPTX auto-fixes as a "strategy", minimal reliable rule set.
- [0007](adr/0007-three-layer-observability.md) — three-layer observability; cost per Review is the primary metric.

## Diagrams

Two formats of the same content:

- **Mermaid in [docs/C4.md](docs/C4.md)** — renders on GitHub with no tooling. Primary way to read.
- **[docs/diagrams/](docs/diagrams/) (d2)** + building a polished PDF report in [report/](report/) — requires `d2`, `pandoc`, headless `chromium`:
  ```bash
  cd report && make diagrams pdf html
  ```

## Contract check

```bash
redocly lint api/openapi.yaml   # requires redocly CLI
```

There is no application code in the repository — documentation only. Conventions for AI assistants and developers are in [CLAUDE.md](CLAUDE.md).
