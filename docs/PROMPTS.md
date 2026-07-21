# Prompt specification

VLM calls go only through `LLMClient` ([ADR 0002](../adr/0002-vlm-pipeline-hybrid-analyzers.md)); `anthropic` is never imported directly in application code. Prompts are versioned md files in `backend/core/prompts/` with frontmatter (`version`, `tier`); the version lands in the Langfuse trace → “version ↔ quality” linkage in `docs/quality-log.md`.

## Shared call rules

- `temperature: 0`, response is strict JSON, validated by a pydantic schema. Invalid JSON → **one** retry with the validation error text in the prompt; second miss → analyzer is skipped (Review stays alive, [ADR 0002](../adr/0002-vlm-pipeline-hybrid-analyzers.md)).
- Structured output via tool use; defensive parsing (strip ```json fences, `json.loads` in try/except).
- Retries on 429/529 with exponential backoff; timeout from Settings.
- Every call is a Langfuse span: prompt, response, tokens, cost (incremented on `ReviewContext`).
- Finding language — **Russian (Cyrillic) only** for `title` / `description` /
  `fix_suggestion`, even when slides are in English. Enum `category`/`severity` —
  Latin (technical codes). `title` ≤ 80 characters.
- `tier` — `screening` (cheap model for zoom screening) or `full` (expensive for analysis); for a future two-tier model.

## Shared Finding contract (all analyzers)

```jsonc
{ "findings": [ {
  "slide_num": 7,                 // or null for deck level
  "category": "TYPOGRAPHY|HIERARCHY|READABILITY|CONSISTENCY|CHART|NARRATIVE|SPEECH_MISMATCH|DELIVERY",
  "severity": "CRITICAL|MAJOR|MINOR",
  "title": "...",                 // ≤ 80 characters
  "description": "...",
  "fix_suggestion": "...",
  "box_2d": [200, 100, 300, 400] | null,  // [ymin,xmin,ymax,xmax], integers 0..1000
  "auto_fixable": false
} ] }
```

**Box format.** On the wire the model returns `box_2d` — Gemini’s native box
format (`[ymin, xmin, ymax, xmax]`, integers on a 0..1000 scale). Models are trained
to emit coordinates this way and hit targets noticeably more accurately than with
arbitrary `{x,y,w,h}` floats. Inside the project there is one format — `BBox` (0..1,
x/y/w/h); conversion lives in one place, `core.geometry.box_2d_to_bbox`, and
drops anything that cannot be treated as a box (not 4 numbers, zero area).

Shared quality rule across all prompts: **“better 2 real findings than 6 stretched ones.”** Few-shot examples come from the spike (`docs/spike-notes.md`).

## 1. Per-slide analysis (`SlideAnalyzer`, tier=full)

**System (essence):**
```
You are a senior presentation designer. Review ONE slide: visual hierarchy,
typography, readability. Do not invent problems — better 2 real findings than 6 stretched ones.
For each finding, provide box_2d of the problem region ([ymin,xmin,ymax,xmax], 0..1000) and,
if the issue can be fixed automatically (small font, weak contrast,
nearly misaligned block), set auto_fixable=true.
In a separate field return has_chart: whether the slide has a chart/diagram.
Reply with strict JSON: {"findings": [...], "has_chart": bool}
```
**User:** slide PNG + slide text extracted via python-pptx (the model reads small text from the image poorly).
**has_chart** → input for `ChartChecker`.

## 2. Zoom screening (`ZoomAgent`, tier=screening)

**System (essence):**
```
Return slide regions worth inspecting closer: small text, dense table,
chart with numbers. Do not invent reasons maximally.
Reply with strict JSON: {"regions": [{"box_2d": [520,60,950,940], "reason": "small_text|dense_table|chart"}]}
```
Then for each region — crop (Pillow, upscale ×2) → re-analyze with §1 prompt (tier=full). Cap 3 zooms/slide. Findings are deduplicated against §1 by IoU.

## 3. Cross-slide analysis (`DeckAnalyzer`, tier=full)

**System (essence):**
```
You see a contact sheet (grid of all slide thumbnails) and the slide texts.
Find: inconsistent fonts/colors/margins, duplicate slides, narrative gaps
(structure: problem → solution → evidence → CTA).
Mark deck-level findings with slide_num=null; slide-bound ones — with their number.
Reply with strict JSON: {"findings": [...]}
```
**User:** 1–2 contact-sheet images + texts of all slides.

## 4. Chart check (`ChartChecker`, tier=full)

For slides with `has_chart`. First, structured chart reading:
```
Read the chart structurally. Reply with strict JSON:
{"chart_type": "bar|line|pie|...", "y_axis_starts_at_zero": bool,
 "series": [{"label": "...", "values": [...]}], "value_labels_present": bool}
```
Then deterministic checks (in code, not in the LLM): Y axis not from zero when range < 2× → `CHART/MAJOR`; pie share sum ≠ 100 %; if Excel is attached (openpyxl → sheet dict) — reconcile values with the source. Semantic check “slide caption contradicts chart data” — a separate LLM call.

## 5. Cross-modal alignment (`CrossModalAnalyzer`, tier=full)

After aligning the Transcript to slides (MVP — heuristic; phase 4 — precise `SlideTiming`, [ADR 0005](../adr/0005-crossmodal-delivery-analysis.md)):
```
Given a speech fragment and the slide shown at that moment.
If the speaker claims something that contradicts the slide — return a SPEECH_MISMATCH finding.
Also, if the speech suggests the slide should be revised (speaker spends a long time on it /
skips it instantly / adds something important missing from the slide) — return a recommendation.
Reply with strict JSON: {"findings": [...]}
```
`DELIVERY` findings (pace, pauses, fillers) are built from `DeliveryMetrics` **without an LLM** (computed from Whisper timestamps).

## 6. Deduplication (`Aggregator`, tier=screening)

When `slide_num` matches and bboxes overlap (IoU > 0.5) — a short LLM question “is this the same problem?” to collapse duplicates from different analyzers.

## Fallback without LLM

Delivery metrics (`DeliveryMetrics`) and deterministic chart checks (axis, pie sum, Excel reconcile) work **without** a VLM — they survive model unavailability. Semantic Findings (hierarchy, narrative, mismatch) simply miss from the partial report on model failure, and do not kill the Review.

## Configuration (env)

| Variable | What |
|---|---|
| `LLM_API_KEY` | VLM key (server-only) |
| `LLM_MODEL_FULL` | id of the primary vision model |
| `LLM_MODEL_SCREENING` | id of the cheap model for screening/dedupe |
| `LLM_TIMEOUT_SECONDS` | Call timeout |
| `LLM_MAX_ZOOMS_PER_SLIDE` | Zoom cap (default 3) |
| `LANGFUSE_PUBLIC_KEY` / `LANGFUSE_SECRET_KEY` | Tracing and cost |
