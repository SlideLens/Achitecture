# ADR 0003: Async background worker instead of synchronous HTTP

**Status:** Accepted
**Date:** 10 June 2026
**Decision context:** Review processing, frontend–backend interaction

## 1. Context

A Review of a 20-slide Deck takes **2–5 minutes**: render, dozens of VLM calls, transcription, autofixes. Holding an HTTP request open that long is not viable — proxy/browser timeouts, dropped connections, no progress UX, and risk of losing work on restart.

## 2. Decision

Separate file acceptance from processing:

1. **Acceptance (fast):** `POST /reviews` validates MIME/size, stores files via `Storage`, creates `Review(status=queued)`, enqueues a job, and **immediately** responds `202` + `ReviewOut`.
2. **Processing (background):** worker `process_review` — downloads files into a workdir, runs `PipelineOrchestrator.run()`, writes Findings/Score/artifacts to DB and Storage, sets `status=done` and sends email. Pipeline exceptions → `status=failed` + human-readable `fail_reason` + Sentry.
3. **Status:** the frontend polls `GET /reviews/{id}` (TanStack Query, `refetchInterval=5000`), stopping on `done`/`failed`. Real-time step progress is deliberately not implemented — status + email are enough.
4. **Queue: ARQ + Redis from day one.** The worker is a separate process (in MVP the same Docker image as `app`, started with a different command; see [DEPLOY.md](../docs/DEPLOY.md)); jobs are pushed to Redis. Periodic task `cleanup_expired_files` (ARQ cron) deletes `FileAsset` rows past `expires_at`.

## 3. Alternatives considered

- **Synchronous response in one request.** Rejected: timeouts, no progress, work lost on restart.
- **WebSocket/SSE with step-level progress streaming.** Rejected for MVP: heavier infrastructure, little value — the user leaves the tab for minutes anyway; email on completion solves it.
- **In-process `FastAPI BackgroundTasks` at the start.** Rejected: a Review runs 2–5 minutes and dies on uvicorn restart/deploy, loads the web worker, and does not allow scaling processing separately. Saving “no Redis” does not pay off — Redis is needed anyway, and rewriting jobs for ARQ later costs more than starting with it.
- **Celery.** Rejected: heavier than ARQ; for an async FastAPI codebase ARQ fits better and is easier to operate.

## 4. Consequences

### Positive
- Instant response on upload; processing survives closing the tab.
- The worker scales separately from the API; on crash the job is retried.
- `failed` status with `fail_reason` — honest UX instead of a hung request.

### Negative and risks
- Queue infrastructure (Redis) and shared Storage between API and worker are required. *Mitigation:* on MVP everything on one host — `app` and `worker` from one image, Redis and local disk in the same docker-compose; move Storage to S3/R2 as we grow.
- The user waits for email/polling instead of seeing the result immediately. *Mitigation:* explicit statuses in the cabinet, email on readiness.
- Queue depth is a degradation risk. *Mitigation:* `queue_depth` metric and alert ([ADR 0007](0007-three-layer-observability.md)).
