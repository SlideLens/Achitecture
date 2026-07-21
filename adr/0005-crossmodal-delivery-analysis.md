# ADR 0005: Cross-modal reconciliation and Delivery metrics; rehearsal — phase 4

**Status:** Accepted
**Date:** 15 June 2026
**Decision context:** speech analysis, transcript-to-slide alignment, MVP boundaries

## 1. Context

A key product differentiator is not only slides but also **speech**: reconciling “what the speaker says ↔ what is on the slide” and Delivery metrics. That requires a timed transcript and its binding to slides. Precise binding only comes from a recording with known slide-change moments — and MVP has no such recording (the user uploads a finished video/audio from phone or Zoom).

## 2. Decision

**Transcription — faster-whisper locally** (medium, RU) → `list[TranscriptSegment]` with timestamps. From video we take only the audio track (ffmpeg → WAV 16k mono); frames are not analyzed.

**Delivery metrics** (`DeliveryMetrics`) are computed from Whisper timestamps **without a separate model:** pace (words/min), filler words (RU dictionary in constants), long pauses (> 3 s). Cheap and deterministic.

**Transcript-to-slide alignment — two-level:**
- **MVP (heuristic):** bind by mentions of slide titles/numbers + even distribution of the remainder. Imprecise, but enough to catch gross mismatches.
- **Phase 4 (precise):** Rehearsal mode — the user flips slides in the browser, `MediaRecorder` records audio + slide-change timestamps → precise `SlideTiming`. The same `CrossModalAnalyzer` accepts precise binding instead of the heuristic.

**Speech Findings:** LLM reconciliation of a “speech ↔ slide” chunk → `SPEECH_MISMATCH` (speaker claims X, slide shows Y). Plus `DELIVERY` Findings from metrics and Deck recommendations (“slide 7 — 3 min of speech → split”, “slide 12 flipped in 5 s → remove”).

**Rehearsal mode is deferred to phase 4** (after core validation): the data model reserves an empty `Rehearsal` table so the future feature does not break the schema. That turns the product from a “one-off audit” into a “trainer before every talk” (dynamics across runs → a reason to subscribe).

## 3. Alternatives considered

- **Diarization + precise alignment from an uploaded recording.** Rejected for MVP: complex and unreliable; heuristics suffice, and precision is better obtained for free in phase 4 from browser timestamps.
- **Cloud STT instead of local whisper.** Rejected at the start: privacy (pitch recording is personal data) and cost; local whisper on CPU fits within the Review time budget.
- **Video-frame analysis (gestures, on-screen slide).** Rejected: expensive and outside the value core; audio + slides cover 90% of the benefit.

## 4. Consequences

### Positive
- Cross-modality — the unique feature — exists already in MVP on an uploaded recording.
- Delivery metrics are nearly free (computed from timestamps, no model).
- The DB schema is ready for phase 4 without a breaking migration.

### Negative and risks
- Heuristic alignment errs on slides without clear titles/numbers. *Mitigation:* `SPEECH_MISMATCH` requires LLM confirmation; disputed bindings do not become confident Findings.
- Whisper adds minutes to a Review with audio. *Mitigation:* small/medium model; transcription in parallel with slide render.
- The RU filler dictionary is incomplete. *Mitigation:* dictionary in constants, extended as we operate.
