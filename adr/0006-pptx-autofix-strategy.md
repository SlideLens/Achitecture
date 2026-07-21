# ADR 0006: PPTX autofixes — strategy pattern, minimal reliable set

**Status:** Accepted
**Date:** 18 June 2026
**Decision context:** Deck auto-correction, scope and safety of edits

## 1. Context

Product differentiation is not only critique but also a **fixed Deck**: the user downloads a PPTX with fixes already applied. The temptation is to “re-layout” slides (move blocks, rewrite text). But python-pptx works at a low level, and the PPTX zoo (masters, themes, groups, animations) makes aggressive edits fragile: it is easy to “break the layout” and hand the user a broken file — worse than not fixing at all.

## 2. Decision

Autofixes use the **strategy pattern** with a narrow, reliable rule set:

1. **`FixRule` (ABC):** `applies_to(finding) -> bool`, `apply(shape, finding) -> FixResult`.
2. **MVP set** (only safe, local, meaningfully reversible edits):
   - `MinFontSizeRule` — font < 14 pt → 14 pt.
   - `ContrastRule` — text contrast < 4.5:1 (WCAG formula) → recolor to nearest dark/light.
   - `AlignmentRule` — align nearly aligned blocks (3 px tolerance).
3. **`PptxFixer.fix(deck, findings) -> Path`:** file copy (original untouched), apply rules **only** to Findings with `auto_fixable = True`, log `applied/skipped`.
4. **Control render after edits:** diff slide count + coarse check “nothing went off bounds” — if a fix broke the layout, roll back.
5. **Slide re-layout is out of MVP** (and likely out of the product entirely): that is a designer’s manual work, not automation.

## 3. Alternatives considered

- **LLM rewrites/recomposes the whole slide.** Rejected: unpredictable, breaks brand layout, cannot guarantee the file opens.
- **Export a “fixed” version as a brand-new PPTX from scratch.** Rejected: loses original styling, masters, branding — the user needs *their* file with surgical edits.
- **Show recommendations only, no file edits.** Rejected as the sole option: autofix is a stated value; but it is deliberately limited to a safe set, and the rest stays a textual recommendation (`fix_suggestion`).

## 4. Consequences

### Positive
- The fixed Deck is guaranteed to open and keeps styling; edits only where safe.
- New rules are added as separate `FixRule`s without changing orchestration.
- The `auto_fixable` flag links a Finding to autofix capability — transparent for the UI (“this one can be fixed automatically”).

### Negative and risks
- Only a subset of Findings is fixed automatically; the rest are manual recommendations. *Deliberate:* reliability over coverage.
- The control render doubles time on the autofix step. *Mitigation:* render once at the end, not after each rule.
- Rare PPTX files may still behave unexpectedly. *Mitigation:* edits run on a copy; on control-render failure — roll back to the original; Findings remain as recommendations.
