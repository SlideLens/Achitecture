# SlideLens — presentation review agent

> Original product statement. Detailed requirements are in [docs/PRD.md](docs/PRD.md); shared terminology is in [CONTEXT.md](CONTEXT.md).

## Problem

People who regularly present slides to leadership and clients (consultants, analysts, sales, founders) prepare presentations almost blindly. A senior designer’s or experienced speaker’s eye is expensive and unavailable for every Deck. As a result:

- slides are overloaded, hard to read, and inconsistent in fonts and colors;
- charts mislead (truncated axes, caption contradicts data) — sometimes unintentionally;
- what the person **says** diverges from what is **on the slide**;
- Delivery suffers: too fast/slow, filler words, "bogs" stuck on one slide.

Existing tools are either cosmetic (templates, auto-alignment) or give checklist-level "plastic" critique and do not look at numbers or speech.

## Goal

A web platform where a user uploads a Deck (PPTX/PDF), optionally a Pitch recording and Excel with data, and a multimodal agent returns a **senior-designer-level Review**: annotated screenshots of problems, honesty checks on charts, a "what the speaker says ↔ what is on the slide" cross-check, and an **auto-fixed version of the file**.

## What MVP must do

- accept a Deck (PPTX/PDF, up to ~50 MB) and optional attachments (Pitch recording, Excel);
- render slides and analyze each: typography, visual hierarchy, readability;
- "zoom" into suspicious regions (small text, charts) and analyze them at larger scale;
- analyze the deck as a whole: font/color consistency, narrative logic, duplicates;
- check charts: truncated axes, caption vs. data, reconciliation against attached Excel;
- cross-check speech and slides, compute Delivery metrics (pace, pauses, filler words) from the uploaded recording;
- prioritize Findings, assign a Deck Score (0–100);
- show a Report: annotated slides, filters, blocks for "charts / Delivery / speech ↔ slides";
- deliver a **fixed PPTX version** and a PDF report;
- registration with email confirmation, free Review limit, public landing with a live example.

## Key differentiation

1. **Cross-modality:** slides + speech + data in one Review.
2. **Number checks on charts** (truncated axes, chart does not support the caption).
3. **PPTX auto-fix**, not critique alone.
4. Focus on **corporate** presentations, not VC pitch decks.
5. **Russian-language** market as the starting point.
6. **Rehearsal mode** (phase 4): record a pitch inside the platform → per-slide timing and a practice loop before every talk.

## Nice to have (after MVP)

- Rehearsal mode with in-browser recording and dynamics across runs (phase 4).
- Billing and plans (phase 3).
- Google Slides integration; brand-book compliance.
- Two-tier model (cheap screening → expensive analysis) for cost optimization.

## Project constraints

- Solo development on evenings/weekends (5–10 h/week) with AI assistants; realistic public MVP timeline — ~3–4 months.
- MVP language — Russian only (prompts, Report, landing).
- Main cost line — VLM API; measure unit economics from day one (Langfuse).
- Privacy: others’ presentations contain confidential data → files auto-delete after N days and are not used for training.
