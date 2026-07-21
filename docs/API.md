# API contract

REST/JSON, prefix `/api/v1`. Source of truth — [api/openapi.yaml](../api/openapi.yaml) (it wins on conflict); this file is the prose companion. Terms — per [CONTEXT.md](../CONTEXT.md); data model — per [C4.md](C4.md).

Auth: `Authorization: Bearer <access_token>` (token carries `user_id`). Access is short-lived (~15 min), refreshed via refresh (~30 days) through `POST /auth/refresh`; logout is client-side (delete tokens). Tokens on the frontend live in `localStorage` — a deliberate MVP tradeoff (simplicity vs XSS risk); when a sensitive surface appears, httpOnly cookies are a candidate. The Review runs in the background — status is **polled**, not awaited in the request ([ADR 0003](../adr/0003-async-review-worker.md)).

## Common error codes

| Code | When |
|---|---|
| 401 | Missing/invalid/expired token |
| 402 | Free Review limit exhausted |
| 404 | Entity not found **or does not belong** to the user (ownership checked on the server, US-8) |
| 409 | Illegal state (e.g. requesting a Report for a Review not in `done`) |
| 413 | File larger than 50 MB |
| 422 | Request body validation error |

Error body: `{ "detail": "...", "code": "LIMIT_REACHED" | null }`.

## DTOs (brief)

```jsonc
// User
{ "id": "uuid", "email": "a@b.ru", "plan": "free|paid",
  "free_reviews_left": 2, "email_verified": true, "is_admin": false, "created_at": "..." }

// ReviewOut (card/status)
{ "id": "uuid", "status": "queued|processing|done|failed",
  "score": 74 | null,              // set when done
  "fail_reason": "..." | null,     // set when failed
  "deck_filename": "q3.pptx", "n_slides": 20 | null,
  "has_audio": true, "has_data": false,
  "created_at": "...", "finished_at": "..." | null }

// Finding
{ "id": "uuid", "slide_num": 7 | null,   // null = deck level
  "category": "TYPOGRAPHY|HIERARCHY|READABILITY|CONSISTENCY|CHART|NARRATIVE|SPEECH_MISMATCH|DELIVERY",
  "severity": "CRITICAL|MAJOR|MINOR",
  "title": "≤80 characters", "description": "...", "fix_suggestion": "...",
  "bbox": {"x":0.1,"y":0.2,"w":0.3,"h":0.1} | null,   // normalized 0..1
  "screenshot_asset_id": "uuid" | null,  // annotated PNG
  "screenshot_url": "/api/v1/files/{uuid}?sig=..." | null,  // ready URL, re-signed on every GET /reviews/{id}/report — no separate signature request
  "auto_fixable": true, "auto_fixed": false,
  "source": "SlideAnalyzer", "user_flag": false, "user_like": false }

// ReportOut
{ "review_id": "uuid", "score": 74, "n_slides": 20,
  "findings": [ /* Finding[] */ ],
  "delivery": { "words_per_minute": 138.5,
                "filler_words": {"um": 12, "like": 5},
                "long_pauses": [45.2, 132.8] } | null,   // only if audio was present
  "auto_fixed_count": 4,
  "pdf_asset_id": "uuid" | null, "fixed_pptx_asset_id": "uuid" | null,
  "fixed_pptx_filename": "q3-review_fixed_v2.pptx" | null }
```

## Endpoints

### Auth

| Method and path | Access | Body → Response |
|---|---|---|
| `POST /auth/register` | public | `{email, password}` → `201 {access, refresh, user}`; creates `plan=free`, `free_reviews_left=2` (usable immediately, without verify) |
| `POST /auth/login` | public | `{email, password}` → `200 {access, refresh, user}` |
| `POST /auth/refresh` | public | `{refresh_token}` → `200 {access, refresh, user}` |
| `GET /auth/me` | authenticated | → `200 User` |

### Reviews

| Method and path | Access | Body → Response |
|---|---|---|
| `GET /reviews` | authenticated | → `200 [ReviewOut]` (mine only, newest first) |
| `POST /reviews` | authenticated | `multipart`: `deck` (required, PPTX/PDF ≤ 50 MB, ≤ 60 slides) + optional `audio` + `data` → `202 ReviewOut` (`queued`). Limit exhausted → `402` |
| `GET /reviews/{id}` | owner | → `200 ReviewOut` (frontend polls every 5 s until `done`/`failed`) |
| `GET /reviews/{id}/report` | owner | only when `done` → `200 ReportOut`; otherwise `409` |

**Limit and `failed`.** `free_reviews_left` is reserved on `POST /reviews` (atomically, `LimitService`). If the Review ends as `failed` (render/pipeline error), **the credit is refunded** — a failure on our side must not cost the user an attempt. There is no separate retry endpoint: retry = upload the file again (a new Review and a new credit reservation).

**Admins** (`user.is_admin`, email from `ADMIN_EMAILS`) bypass `free_reviews_left` entirely — `LimitService.check_and_reserve` returns the user without reserve/decrement, regardless of `plan`.

### Findings / Files / Events

| Method and path | Access | Body → Response |
|---|---|---|
| `POST /findings/{id}/flag` | Review owner | 👎 → `204`; `user_flag=true`, `user_like=false` + score in Langfuse |
| `POST /findings/{id}/like` | Review owner | 👍 → `204`; `user_like=true`, `user_flag=false` + score in Langfuse |
| `POST /findings/{id}/apply_fix` | Review owner | targeted autofix → `204`; regen `fixed.pptx` from original over the `auto_fixed` set; non-`auto_fixable`/foreign → `404` |
| `GET /files/{asset_id}` | owner | serve file via Storage. For `<img>` (screenshots) — short-lived `?sig=` instead of a header (the tag does not send Authorization); others — bearer. Foreign asset → `404` |
| `POST /events` | authenticated | `EventIn[]` (batch of frontend events: `report_opened`, `finding_expanded`, `pdf_downloaded`…) → `204` |

### Ops

| Method and path | Access | Response |
|---|---|---|
| `GET /health` | public | `200 {"status":"ok"}` — smoke after deploy |

## Visibility and privacy rules (server must enforce)

- One user's Review, Findings, and files are inaccessible to another — owner check on the server; foreign → `404` (not `403`, to avoid leaking existence).
- `GET /files/{asset_id}` for screenshots uses a signed `sig` with a short TTL; expired → `401`.
- Files (`FileAsset`) have `expires_at`; after expiry they are deleted by a periodic worker job and return `404` on serve (US-8).
- Free-limit antifraud — via `free_reviews_left` (email confirmation not required in MVP).
