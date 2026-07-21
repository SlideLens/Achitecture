# ADR 0002: Multi-stage VLM pipeline of independent analyzers with a zoom agent

**Status:** Accepted
**Date:** 8 June 2026
**Decision context:** analysis core, Finding quality and cost

## 1. Context

The naive approach “one slide image → one prompt → findings” produced “plastic” critique on early spikes: the model poorly reads fine text from PNGs, misses chart issues, cannot see cross-slide consistency, and gets confused when a slide is dense. Core quality is existential for the product (see priorities in [PRD.md](../docs/PRD.md) §1): without recall ≥ 70% and noise < 20%, there is no product.

At the same time VLM is the main cost driver, and “run everything at maximum” turns a Review of a 20-slide Deck into an expensive operation.

## 2. Decision

Split analysis into a **set of independent analyzers** on top of a shared `BaseAnalyzer`, each with a narrow responsibility and its own versioned prompt:

1. **`SlideAnalyzer`** — per-slide analysis (hierarchy, typography, readability). PNG + python-pptx–extracted slide text are provided together (the model reads fine text from images poorly). Concurrent (`asyncio` + `Semaphore(4)`).
2. **`ZoomAgent`** — two-phase: cheap screening marks suspicious regions (`SuspiciousRegion[]`: bbox + reason) → crop by bbox with ×2 upscale → re-analyze at large scale. Cap **3 zooms/slide** (cost control).
3. **`DeckAnalyzer`** — contact sheet (grid of all slide thumbnails) + texts → font/color consistency, narrative, duplicates; Deck-level Findings (`slide_num = None`).
4. **`ChartChecker`** — for slides with charts: zoom → `ChartReading` → honesty checks; reconcile with Excel when attached.
5. **`CrossModalAnalyzer`** — speech ↔ slides (see [ADR 0005](0005-crossmodal-delivery-analysis.md)).

Shared rules:
- **Single `LLMClient`** — all VLM calls (vision + structured output via tool use), retry of invalid JSON (once, with the error text in the prompt), retries on 429/529 with backoff, Langfuse span per call, cost increment in `ReviewContext`. Parameter `tier` (screening/full) for a future two-tier model.
- **Structured output:** response strictly follows the pydantic schema `Finding`; shared enums `Category`/`Severity` — source of truth for code and UI.
- **Graceful degradation:** `BaseAnalyzer` wraps `run()` — timing, logging, **exception catching**. A failed analyzer is skipped and logged; the Review continues (a partial report is better than `failed`).
- **Prompt versioning:** prompts are md files under `backend/core/prompts/` with frontmatter (`version`, `tier`); the version lands in the Langfuse trace → “version ↔ quality” link in `quality-log.md`.

## 3. Alternatives considered

- **One mega-prompt per slide.** Rejected: does not read fine text, misses charts and cross-slide issues, cannot version by sub-task.
- **Classic CV heuristics (OpenCV) instead of VLM.** Rejected for semantics (hierarchy, narrative, “caption contradicts the data”) — those need a model; partially retained in autofixes ([ADR 0006](0006-pptx-autofix-strategy.md)) and in Delivery fallback metrics.
- **Fine-tune a custom model.** Rejected: no data or budget at MVP; value is in the pipeline, not in weights.

## 4. Consequences

### Positive
- Each analyzer is iterated and tested in isolation; enabled/disabled via a list in the orchestrator config.
- The zoom agent raises recall on “fine” issues missed by the per-slide pass — at controlled cost.
- Cost of each step is visible in Langfuse as a separate span → clear what to optimize.

### Negative and risks
- More VLM calls → higher cost. *Mitigation:* zoom cap, `Semaphore`, screening tier, alert on `review_cost_usd`.
- Duplicate Findings from different analyzers. *Mitigation:* `Aggregator` deduplicates by `slide_num` + bbox IoU > 0.5 + LLM confirmation “same issue?”.
- VLM non-determinism complicates eval. *Mitigation:* `temperature: 0`, golden set + LLM judge, Score stability ±5 on a re-run as an acceptance test.
