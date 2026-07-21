# UI design

Design system = **Tailwind + shadcn/ui** (components edited live in code). Product showcase = **Report page**; spend most design time there. All UI copy is in Russian and uses terms from [CONTEXT.md](../CONTEXT.md) (“Finding,” not “remark”; “Review,” not “analysis”).

## Direction — “Precision Review”

Light theme, a calm precision-review tool: the UI stays neutral (black/white/gray), and color appears only where it carries meaning — Finding Severity, Review status, interactive highlight. Primary accent (`--primary`, black) is reserved for action buttons; `--accent` (blue) — only for interactivity/selection (focus, links, active filters, progress bar), never as a decorative “brand color.”

**Report signature element:** numbered pins on the slide (`SlideViewer`) — a colored circle with the Finding number sits at the corner of its `BBox`; the same number opens the `FindingCard` in the right-hand “field notes” column. The frame around `BBox` is not a solid outline, but 4 corner crop marks (as in crop tools), so it does not fight the slide content. Findings without `BBox` (whole Deck) use a `◆` marker instead of a number.

| Token | Role |
|---|---|
| `--background` | near-white page background |
| `--card` | Review and Finding cards |
| `--primary` | black — solid action buttons (CTA) |
| `--accent` | blue — interactivity/selection: focus rings, links, active filters, progress bar, `processing` indicator |
| `--foreground` / `--muted-foreground` | primary / secondary text |

- **Fonts:** **Golos Text** (RU-native grotesque, variable, 400–800) — UI and headings; **IBM Plex Mono** — Score, Finding numbers, Category/Severity tags, Delivery metrics (data visually separated from prose). Both self-hosted in `src/assets/fonts/` (no external Google Fonts requests in prod). Icons — `lucide-react`.
- **Style:** cards `rounded-lg`, thin borders (`--border`), shadow only on hover for interactive cards — not by default. Badges/tags — `font-mono`, uppercase, no fully filled “pills” (except statuses).

## Finding Severity colors

One color code — in badges, in pins/corners on slides (Annotator), and in filters:

| Severity | Color |
|---|---|
| `CRITICAL` | red |
| `MAJOR` | orange (bright, not brown — clearly distinct from `CRITICAL`) |
| `MINOR` | gray |

## Review status colors

| Status | Look |
|---|---|
| queued | gray (outline) |
| processing | blue (`--accent`), with spinner indicator |
| done | green |
| failed | red, muted, with `fail_reason` |

## Screens

1. **Landing** — offer + **live Report example** (frozen JSON fixture rendered by production Report components → the example always stays in sync with the product). CTA to register. Plausible/Yandex.Metrica.
2. **Login / Registration** — email + password; “confirm your email” screen.
3. **Cabinet (Dashboard)** — Review card grid with statuses + `UploadDropzone` (drag-and-drop Deck + optional audio/xlsx, validate type/size **before** upload, upload progress). Poll status until `done`/`failed`.
4. **Report (ReviewReport)** — main screen:
   - `ScoreGauge` — Score 0–100 as a measuring dial (tick marks + progress arc, color by range);
   - `SlideViewer` — annotated slide PNG with crop marks on `BBox` and numbered pins; pin click scrolls to the matching `FindingCard`;
   - `FindingFilters` — by Category/Severity, state in URL params (shareable via link), active filters — accent (blue) fill;
   - `FindingCard` — marker with Finding number (matches pin), color by Severity, `fix_suggestion`, 👎 button;
   - `DeliveryPanel` + `MismatchPanel` — “Delivery” and “speech ↔ slides” blocks (only if audio was present);
   - “Download PDF report” and “Download fixed PPTX” buttons.
5. **Rehearsal** (phase 4) — stub in MVP.

Navigation — header: Cabinet · Profile/Plan · Log out.

## Rules

- Mobile responsiveness — not an MVP priority (demo and laptop work, minimum 1280px).
- **Empty/intermediate states are required:** empty cabinet, Review `processing` (skeletons + “SlideLens is studying your deck…”), `failed` with a human-readable `fail_reason`, Report with no Findings (“no serious issues found”).
- All client actions important to the funnel (`report_opened`, `finding_expanded`, `pdf_downloaded`, `fixed_pptx_downloaded`) are sent to `POST /events`.
- 👎 on a Finding — noticeable but not pushy: it is the dataset source for prompt improvement ([ADR 0007](../adr/0007-three-layer-observability.md)).
