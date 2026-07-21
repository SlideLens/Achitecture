# SlideLens — shared language (glossary)

This file is the **source of truth for terminology**. All other documents and future code (class names, fields, endpoints, categories) use these terms exactly as defined here. Where a term has a common synonym we avoid, it is marked _Avoid_.

Product: a web platform where a user uploads a presentation (and optionally a pitch recording + Excel with data), and a multimodal agent returns a senior-designer-level review — annotated problems, honesty checks on charts, a "speech ↔ slides" cross-check, and an auto-fixed version of the file.

## People

**User**:
A registered person who uploads presentations and reads Reviews (in code: `User`). An external SaaS customer, not a company employee — so "user" is appropriate here (unlike internal systems).
_Avoid_: client (except in a business context), customer

**Plan**:
The user's plan tier (in code: `plan`): `free` (limit of free Reviews) or `paid`. Billing is phase 3, but the field exists from MVP.
_Avoid_: subscription (as a field name), tariff (ok in UI copy, but the canonical field is `plan`)

**Administrator** (formerly «Администратор»):
A user flag (in code: `User.is_admin`, `bool`) — not a plan tier; orthogonal to `plan`. Set from emails in `ADMIN_EMAILS` (`Settings.admin_email_set`) at registration and synced on every login (`app/auth.py`), so an account registered before appearing on the list is promoted to Administrator without a manual DB edit. The only MVP effect: `LimitService` bypasses `free_reviews_left` entirely, regardless of `plan`. Do not confuse with an admin-panel role — there is no management UI in MVP.
_Avoid_: superuser, root, superuser (as a UI/domain term — `is_admin` / "Administrator" is canonical)

## User artifacts

**Deck** (formerly «Дека»):
An uploaded presentation — a PPTX or PDF file. The unit around which a Review is built.
_Avoid_: presentation (ok in UI copy), slide deck, document, file (a file is `FileAsset`, see below)

**Slide**:
One page of a Deck. In the pipeline it is rendered to PNG (`slide_001.png`…). Numbering starts at 1.

**Pitch recording**:
An optional audio or video recording of a talk that the user attaches to a Deck. From video, only the audio track is used (frames are not analyzed in MVP). Source of the Transcript and Delivery metrics.
_Avoid_: audio (ok as field name `audio`), video, clip

**Data (Excel)**:
An optional `.xlsx` with source numbers that ChartChecker uses to verify values on Deck charts.
_Avoid_: table, dataset

## Review

**Review** (formerly «Разбор»):
A full analysis of one Deck: a pipeline run from ingest to report (in code: `Review`). Has a status, a Score, and a set of Findings. One user creates many Reviews.
_Avoid_: analysis, audit, check, review (the last confuses with status naming in Russian docs; prefer Review as the entity name)

**Review status**:
Lifecycle: `queued → processing → done`, plus terminal `failed`. A Review takes 2–5 minutes in the background, so status is polled rather than waited on in the HTTP request.
- **queued** — files accepted, Review enqueued.
- **processing** — worker is running the pipeline.
- **done** — Findings, Score, and artifacts are ready; email sent to the user.
- **failed** — pipeline crashed; `fail_reason` is set (human-readable cause).

**Score** (formerly «Скор»):
Final Deck score 0–100 (in code: `score`). Computed from Findings: `100 − Σ weights (critical 12, major 5, minor 1)`, normalized by slide count, floor 5. Formula lives in one place (`DeckScorer`), configurable. Weights are a **starting default**; calibrated on a golden set in phase 1.
_Avoid_: rating, points, grade (ok in UI copy)

## Findings

**Finding** (formerly «Находка»):
One concrete problem found in a Deck (in code: `Finding`). Tied to a slide (or to the Deck as a whole if `slide_num = None`), has a Category, Severity, title, description, and suggested fix.
_Avoid_: remark (ok colloquially), error, issue, problem (as an entity name)

**Category**:
Finding type (in code: enum `Category`). Canonical set is fixed — single source of truth for all code and UI:
`TYPOGRAPHY` (typography) · `HIERARCHY` (visual hierarchy) · `READABILITY` (readability) · `CONSISTENCY` (deck consistency) · `CHART` (chart problem) · `NARRATIVE` (narrative/structure) · `SPEECH_MISMATCH` (speech contradicts the slide) · `DELIVERY` (delivery: pace/pauses/fillers).
_Avoid_: type, tag, class

**Severity**:
Finding priority (in code: enum `Severity`): `CRITICAL` (critical) · `MAJOR` (serious) · `MINOR` (minor). Drives border color and weight in the Score.
_Avoid_: importance, priority, severity level

**BBox**:
Normalized `0..1` coordinates of the problem rectangle on the slide (`x, y, w, h`) — dpi-independent. Annotator draws a frame from them. May be `None` (Finding without a precise region).
_Avoid_: coordinates, frame (the frame is the visualization of a BBox), bounding box

**Auto-fix**:
Automatic correction of a Finding in the Deck itself via python-pptx (in code: `auto_fixable` / `auto_fixed`). Applied only to safe rules (font size, contrast, alignment) → output is the Fixed deck.
_Avoid_: auto-edit, fix, autocorrect

**Fixed deck**:
A copy of the uploaded Deck with Auto-fixes applied (`fixed.pptx`) that the user can download.
_Avoid_: fixed version, edited file

## Pipeline and analyzers

**Pipeline**:
Sequence of Review steps: ingest → transcription ∥ Excel parse → analyzers → aggregation → annotation → auto-fixes → report (in code: `core/` package, orchestrator `PipelineOrchestrator`). Pure library: **does not import `app/` and does not touch the DB directly** — only the worker wires them together.
_Avoid_: conveyor, flow, workflow

**Analyzer**:
A pipeline module that produces Findings (in code: subclass of `BaseAnalyzer`). Failure of one Analyzer does not fail the Review — it is skipped and logged (a partial report is better than `failed`). Set: SlideAnalyzer, ZoomAgent, DeckAnalyzer, ChartChecker, CrossModalAnalyzer.
_Avoid_: check, checker (except the name `ChartChecker`), detector

**Zoom agent**:
Analyzer (`ZoomAgent`) that uses a cheap VLM call to mark suspicious slide regions (small text, dense table, chart), crops them (crop + upscale ×2), and analyzes them at larger scale. Max 3 zooms per slide (cost control).
_Avoid_: zoomer, magnifier

**VLM**:
Multimodal (vision-language) model used for slide and chart analysis. In MVP — Claude API (vision). All calls go only through `LLMClient`; `anthropic` is never imported directly in code.
_Avoid_: LLM (narrow case — ok), neural net, model (ok in general context)

## Delivery and cross-modality

**Transcript**:
Pitch-recording text with timestamps (in code: `list[TranscriptSegment]`), produced via faster-whisper. Basis for cross-modal checking and Delivery metrics.
_Avoid_: transcript dump, subtitles

**Delivery** (formerly «Подача»):
How the person speaks (in code: `DeliveryMetrics`): pace (words/min), Filler words, long pauses (> 3 s), per-slide timing. Produces Findings of category `DELIVERY` and recommendations for the Deck.
_Avoid_: speech (speech is what is checked against slides), performance, diction

**Filler words**:
Filler tokens from the RU dictionary ("ээ", "как бы", "то есть", "на самом деле", "короче"…), counted by `compute_delivery`.
_Avoid_: fillers, junk words

**Cross-modal check**:
Matching Transcript to slides (in code: `CrossModalAnalyzer`): speaker claims X, slide shows Y → Finding `SPEECH_MISMATCH`. Plus Deck recommendations based on speech ("slide 7 — 3 minutes of talk → split it").
_Avoid_: multimodal check (ok), alignment (that is a separate sub-step)

**Rehearsal mode**:
Phase 4: user records a pitch in the browser while advancing slides (MediaRecorder: audio + slide-switch timestamps → precise `SlideTiming`). In code: `Rehearsal`. Yields a timing map, "bog" slides (> 2 min) and "stubs" (< 5 s), and dynamics across runs.
_Avoid_: trainer, live recording

## Assembling the result

**Report**:
The Review storefront: Score, Findings per slide with annotated PNGs, blocks for "whole deck", "charts", "Delivery", "speech ↔ slides", "Auto-fixes". A serializable view model (`ReportOut`) stored in the DB, returned to the frontend (`GET /reviews/{id}/report`), and exported to PDF (weasyprint).
_Avoid_: report (as a loose synonym for result), result (ok in general context)

**Annotation**:
Overlaying frames from BBoxes onto the slide PNG (color by Severity) plus Finding number (in code: `Annotator`, Pillow). One slide with several Findings = one image with several frames.
_Avoid_: markup, overlay

**FileAsset**:
Any file tied to a Review (in code: `FileAsset`, field `kind`): `deck_original | slide_png | annotated_png | fixed_pptx | audio | data_xlsx | report_pdf`. Has `expires_at` — auto-delete after N days (privacy).
_Avoid_: file (as a vague term), asset, attachment

## Analytics and cost

**Event**:
A product-analytics record (in code: `Event`): `user_id`, `name`, `properties` (JSON), `created_at`. MVP events: `signup`, `review_created`, `review_done`, `report_opened`, `finding_expanded`, `pdf_downloaded`, `fixed_pptx_downloaded`, `limit_hit`, etc. Written from services, not from routes.
_Avoid_: event (as a loose synonym), metric (a metric is Prometheus)

**Review cost**:
Sum of VLM-call spend for one Review in USD (in code: `ReviewContext.total_cost_usd`). Primary unit-economics metric; traced in Langfuse, with an alert on threshold breach.
_Avoid_: price, costs
