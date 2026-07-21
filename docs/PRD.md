# PRD: SlideLens — presentation review agent

> Solo evening project with AI assistants; realistic public MVP timeline ~3–4 months. Original brief: [TASK.md](../TASK.md).
> Shared vocabulary (terms are binding): [CONTEXT.md](../CONTEXT.md).
> Architecture decisions: [ADR 0001](../adr/0001-pipeline-pure-library.md) … [ADR 0007](../adr/0007-three-layer-observability.md).

---

## 1. Problem and goal

People who regularly present slides to leadership and clients prepare decks blind: a senior designer's eye is expensive, and existing tools are either cosmetic or checklist-level critique that ignores numbers and speech.

**MVP goal:** a platform where the user uploads a Deck (+ optional pitch Recording and Excel), and a multimodal agent returns a senior-designer-level Review: annotated issues, chart honesty checks, speech ↔ slides alignment, and a Fixed deck.

**Product nail:** the moment “the agent showed an annotated slide with a real problem + suggested a fix, and on a chart caught a truncated axis.” Everything else is scaffolding around core quality.

**Quality priority** (when decisions conflict, bias this way): 1) **Finding quality** of the core (recall ≥ 70 %, junk < 20 % on the golden set — otherwise there is no product; see [ADR 0002](../adr/0002-vlm-pipeline-hybrid-analyzers.md)) → 2) **unit economics** (Review cost under control, [ADR 0007](../adr/0007-three-layer-observability.md)) → 3) **reliability** (a partial report beats `failed`) → 4) feature completeness — cut first.

## 2. Target audience

People who regularly present slides to leadership and clients: consultants, analysts, sales, PMs, founders. Corporate presentations, not VC pitch decks. Russian-speaking market as the starting beachhead.

## 3. AI features (core)

All VLM calls run only from the backend, through a single `LLMClient` ([ADR 0002](../adr/0002-vlm-pipeline-hybrid-analyzers.md)); structured JSON output (Pydantic); every call is traced in Langfuse with cost. The pipeline is a pure library — it knows nothing about the web or the DB ([ADR 0001](../adr/0001-pipeline-pure-library.md)).

Ten Review steps (diagram — [C4.md](C4.md#review-pipeline)):

1. **Ingest** — PPTX/PDF → PDF (LibreOffice headless) → slide PNGs (pdf2image, 150 dpi); extract slide text (python-pptx).
2. **Transcription** — pitch Recording → WAV → faster-whisper → Transcript with timestamps; Delivery metrics (pace, pauses, fillers).
3. **Per-slide analysis** (`SlideAnalyzer`) — hierarchy, typography, readability → Findings.
4. **Zoom agent** (`ZoomAgent`) — flag suspicious regions → crop ×2 → close-up analysis (≤ 3 zooms/slide).
5. **Cross-slide analysis** (`DeckAnalyzer`) — font/color consistency, narrative, duplicates.
6. **Chart checks** (`ChartChecker`) — truncated axes, pie ≠ 100 %, caption vs data; reconcile with Excel.
7. **Cross-modal alignment** (`CrossModalAnalyzer`) — speech ↔ slides (`SPEECH_MISMATCH`) + Deck recommendations from speech.
8. **Aggregation** (`Aggregator` + `DeckScorer`) — dedupe, prioritization, Score 0–100.
9. **Annotation** (`Annotator`, Pillow) — BBox frames on screenshots.
10. **Autofixes** (`PptxFixer`, python-pptx) — contrast/alignment/font size → Fixed deck; **Report** (HTML + PDF).

**Reliability:** failure of any Analyzer does not kill the Review — it is skipped and logged (a partial report beats `failed`). Ingest render failure → `failed` with a human-readable `fail_reason` and the hint “attach a PDF.”

## 4. Data model (brief)

Full ERD — in [C4.md](C4.md#level-4--data-model-erd). Key entities: **User** (plan, free_reviews_left), **Review** (status, score, fail_reason, deck_filename, n_slides, delivery_metrics), **Finding** (slide_num, category, severity, bbox, auto_fixable), **FileAsset** (kind, expires_at — auto-delete), **Event** (product analytics), **Rehearsal** (empty stub for phase 4). Pipeline contracts (`Finding`, `Category`, `Severity`, `BBox`, `DeliveryMetrics`) — pydantic, single source of truth; see [PROMPTS.md](PROMPTS.md).

## 5. User stories and acceptance criteria

### US-1. Registration and login
*As a new user, I register and confirm my email to get free Reviews.*
- ✅ Registration (email + password), confirmation email, login (JWT access + refresh).
- ✅ New account: `plan = free`, `free_reviews_left = 2`.
- ✅ Unconfirmed email → cannot create a Review (403).

### US-2. Upload Deck and start Review
*As a user, I upload a Deck and optionally a pitch Recording and Excel.*
- ✅ Drag-and-drop Deck (PPTX/PDF, ≤ 50 MB, ≤ 60 slides), optional audio/video and `.xlsx`; validate type/size **before** upload.
- ✅ `POST /reviews` → `202` + Review in `queued`; pipeline runs in the background ([ADR 0003](../adr/0003-async-review-worker.md)).
- ✅ Limit exhausted → `402` with a clear message.

### US-3. Cabinet and Review status
*As a user, I see my Reviews and their progress.*
- ✅ Review card list with statuses; frontend polls status (every 5 s) until `done`/`failed`.
- ✅ `failed` shows a human-readable `fail_reason`.
- ✅ On `failed`, the reserved free Review **is refunded** (`free_reviews_left` incremented back) — a failure on our side does not consume an attempt. No separate retry: retry = new upload.
- ✅ On completion, an email arrives with a link to the Report.

### US-4. Report (product nail)
*As a user, I open the Report and understand what is wrong with my Deck and why.*
- ✅ Overall Score; Findings per slide with **annotated** screenshots (click a frame → scroll to Finding).
- ✅ Filter by Category/Severity (state in URL, shareable via link).
- ✅ Blocks: “whole deck,” “charts,” “Delivery,” “speech ↔ slides” (last two — if a pitch Recording was attached).
- ✅ Each Finding: title, description, fix suggestion, 👎 button (junk finding).

### US-5. Download artifacts
*As a user, I take the result in a convenient form.*
- ✅ “Download PDF report” button.
- ✅ “Download fixed PPTX” button (Autofixes applied to `auto_fixable` Findings; file opens in PowerPoint/LibreOffice without errors).

### US-6. Chart honesty checks
*As a user, I want the agent to catch misleading charts.*
- ✅ For slides with charts: truncated Y axis, pie share sum ≠ 100 %, caption contradicts data → `CHART` Finding.
- ✅ If Excel is attached — reconcile chart values with the source.

### US-7. Delivery and speech ↔ slides
*As a user, I attached a pitch recording and want a review of the talk, not only the slides.*
- ✅ Delivery metrics: pace (words/min), long pauses, filler words.
- ✅ `SPEECH_MISMATCH`: speaker claims something that contradicts the slide.
- ✅ Deck recommendations from speech (“slide 7 — 3 min of talk → split it”).

### US-8. Privacy
*As a user, I upload confidential slides and want guarantees.*
- ✅ Files auto-delete after N days (`FileAsset.expires_at`), not used for training — stated in the terms of service.
- ✅ Another user's Review/file is inaccessible (owner check on the server, 404).

## 6. Scope

### MVP (required)
US-1 … US-8. Pipeline core (steps 1–10) with golden-set quality (recall ≥ 70 %, junk < 20 %); auth + limits; cabinet and Report (the showcase — spend time here); PDF and Fixed deck; landing with a live example; VPS deploy with observability stack ([DEPLOY.md](DEPLOY.md)). Autofixes — minimal reliable set (font/contrast/alignment).

### Stretch (if time remains, in priority order)
1. Two-tier model (cheap screening → expensive analysis) to cut Review cost.
2. Broader recording format support (mov/m4a), independent of Resume upload.
3. Report UI polish: Finding expand animations, before/after autofix diffs.

### Out of scope (deliberately not in MVP)
- **Rehearsal mode** (in-browser recording) — deferred to phase 4; `Rehearsal` stays an empty table for now.
- **Payments** — phase 3; `plan` field exists, billing does not.
- Teams, sharing, comments; realtime per-step progress (status + email is enough).
- Landing-page reviews, brand-book compliance, Google Slides integration.
- Full slide reflow via autofix (only targeted safe edits, [ADR 0006](../adr/0006-pptx-autofix-strategy.md)).

### Backlog (after public MVP)
Rehearsal mode and cross-run dynamics (phase 4, [ADR 0005](../adr/0005-crossmodal-delivery-analysis.md)); pricing and YooKassa (phase 3); content marketing via public reviews. Full plan — [BACKLOG.md](BACKLOG.md).

## 7. Stack

FastAPI (pure JSON API) + React/Vite/TS/Tailwind/TanStack Query, PostgreSQL + SQLAlchemy 2.0 (Alembic migrations), fastapi-users (JWT), background worker (ARQ + Redis), LibreOffice + pdf2image (render), Claude API (vision) via `LLMClient`, faster-whisper (transcription), python-pptx (autofixes), Pillow (annotations), weasyprint (PDF). Details and rejected alternatives — [ADR 0004](../adr/0004-stack-fastapi-react.md).

## 8. Test and golden set

- 10–15 public Decks (≥ 5 Russian-language, ≥ 3 with charts that have data, 2–3 deliberately bad).
- 3 Decks hand-labeled (“golden” Finding list: slide + category) — used to compute recall/junk (`backend/tests/golden/eval.py` → `docs/quality-log.md`).

## 9. MVP success metrics

- Core quality: recall ≥ 70 %, junk < 20 % on the golden set.
- Cost of a Review for a 20-slide Deck under the threshold (measured in phase 1, alert in Langfuse).
- ≥ 50 % of people who got a Review opened the Report fully; ≥ 5 people said “I would pay” with a concrete amount.

## 10. Risks

| Risk | Mitigation |
|---|---|
| Review quality feels “plastic,” like competitors | Core (phase 1) — 60 % of effort: golden set, prompt iterations in `backend/core/prompts/`, zoom agent, data reconciliation ([ADR 0002](../adr/0002-vlm-pipeline-hybrid-analyzers.md)) |
| Expensive VLM per user | Limits; cost traced from day 1; zoom cap; two-tier model in stretch ([ADR 0007](../adr/0007-three-layer-observability.md)) |
| Large models will soon do this “out of the box” | Value is in the pipeline (zoom, data reconciliation, autofix) and the product, not the model alone |
| PPTX zoo breaks render (fonts, animations) | LibreOffice + RU fonts in the image; typed ingest exceptions; “attach a PDF” fallback ([ADR 0004](../adr/0004-stack-fastapi-react.md)) |
| Confidential data in third-party presentations | Auto-delete files after N days, not for training; stated in ToS (US-8) |
| Review takes minutes — HTTP timeout | Async worker, status polling, email on completion ([ADR 0003](../adr/0003-async-review-worker.md)) |
