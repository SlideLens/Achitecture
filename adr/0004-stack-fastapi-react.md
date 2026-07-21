# ADR 0004: Stack — FastAPI (JSON API) + React SPA, render via LibreOffice

**Status:** Accepted
**Date:** 12 June 2026
**Decision context:** technology stack, frontend↔backend contract, deploy

## 1. Context

We need a stack for solo development with AI assistants (consistent codegen and a type-safe contract matter), for a multimodal Python pipeline (VLM, whisper, python-pptx — Python ecosystem), and for a showcase Report with interactivity (annotated slides, filters). Plus reliable server-side PPTX-to-image rendering.

## 2. Decision

**Backend — FastAPI, pure JSON API** (`/api/v1`, JWT bearer). The contract is an auto-generated OpenAPI schema; frontend types are generated from it (`openapi-typescript`), never written by hand: change a DTO → regenerate types → TypeScript shows what broke.

**Frontend — separate SPA:** React + Vite + TypeScript + Tailwind + TanStack Query + React Router. The Report is the product’s main screen.

**Data — PostgreSQL + SQLAlchemy 2.0 + Alembic** (SQLite locally for a fast start). Unlike pure hackathon projects, this product lives for a long time → migrations are needed from day one.

**Auth — fastapi-users** (email+password, email confirmation, JWT access+refresh). We do not write auth ourselves.

**Slide render — LibreOffice headless** (`soffice --convert-to pdf`) + pdf2image (PDF→PNG 150 dpi). Works in Docker, handles PPTX and PDF. The image includes RU fonts (ttf-mscorefonts, PT Sans/Serif) — critical, otherwise Russian decks render as tofu squares.

**Other core pieces:** faster-whisper (transcription), python-pptx (autofixes), Pillow (annotations), weasyprint (PDF report from a Jinja template — the only place with an HTML template; the frontend does not generate PDFs).

**Prod — Docker Compose + Caddy:** the built SPA (`frontend/dist`) is baked into the backend image (`./static`) and served by FastAPI itself (`StaticFiles`) — as in the reference project. Caddy in front of the image only terminates TLS and reverse-proxies all traffic to app (**one domain → no CORS in prod, cookies/tokens are simpler**), HTTPS automatically. Details — [DEPLOY.md](../docs/DEPLOY.md).

## 3. Alternatives considered

- **Server-rendered (Jinja2 + HTMX) instead of SPA.** Considered (early draft was HTMX) and rejected: a Report with annotated slides, clicks on bounding boxes, and URL filters is rich client state where React with TanStack Query wins; a typed OpenAPI contract is more valuable for solo development with AI.
- **Node/TS on the backend.** Rejected: the entire multimodal stack (whisper, python-pptx, VLM SDK, pdf2image) is Python; moving the core to Node does not pay off.
- **Render via native PowerPoint / cloud converter.** Rejected: does not run in a Linux container / external dependency and privacy concerns; LibreOffice headless is self-contained.
- **nginx + separate frontend container.** Rejected: Caddy is simpler (auto-HTTPS, less config); one domain removes CORS.

## 4. Consequences

### Positive
- A single type-safe frontend↔backend contract from OpenAPI; AI assistants generate consistent code against the structure.
- Python core and web in one ecosystem; render and all ML dependencies live in one image.
- One origin in prod — no CORS, simpler tokens.

### Negative and risks
- Heavy Docker image (LibreOffice + ffmpeg + fonts + optional CUDA whisper). *Mitigation:* multi-stage build, CPU whisper (small/medium), cached layers.
- LibreOffice is finicky with exotic PPTX (animations, rare fonts). *Mitigation:* render timeout + retry, typed ingest exceptions, fallback “attach a PDF”.
- Two codebases (Python + TS). *Mitigation:* OpenAPI type generation keeps them in sync.
