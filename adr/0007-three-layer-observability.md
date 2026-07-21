# ADR 0007: Three-layer observability; Review cost — primary metric

**Status:** Accepted
**Date:** 20 June 2026
**Decision context:** monitoring, analytics, unit-economics control

## 1. Context

The product has three distinct observability questions that must not be mixed:
1. **What users do** (product: funnel, what they read in the Report).
2. **What the agent does** (LLM: prompts, responses, latency, and — critically — **cost of each Review**).
3. **Whether the service is alive** (infra: errors, RPS, queue, share of `failed`).

Review cost is the main expense line and the primary unit-economics risk: without measuring it from day one it is easy to build a product that costs more than customers pay.

## 2. Decision

Three independent layers, each with its own tool:

**Layer 1. Product analytics.** `Event` table in Postgres (`user_id`, `name`, `properties` JSON, `created_at`); events are written from services, not from routes. Funnel `signup → review_created → review_done → report_opened → *_downloaded → second_review`. Funnel dashboard — in Grafana via a Postgres datasource (no separate BI needed). On the landing — Plausible / Yandex.Metrica.

**Layer 2. LLM observability — Langfuse** (from day one):
- trace of every Review: all VLM calls by step, prompts, responses, latency;
- **Review cost in USD** — unit-economics metric; alert when a threshold is exceeded;
- prompt versions from `backend/core/prompts/` attached to traces → which version yields better Findings;
- 👎 “junk Finding” flag from the Report is written back to Langfuse — a dataset for prompt iteration.

**Layer 3. Infrastructure** (as we grow):
1. **Sentry** — from day one (phase 1): pipeline and backend errors with stack traces.
2. **structlog** (JSON, `review_id`/`user_id` via contextvars) — from day one, so Loki can parse later without rework.
3. **Grafana + Loki + Prometheus** — at deploy (phase 2): end-to-end search by `review_id`; custom metrics `pipeline_step_duration_seconds{step}`, `review_cost_usd`, `queue_depth`, `reviews_total{status}`; 3 dashboards (API, pipeline, funnel); Telegram alerts (service down, `queue_depth > 10`, `failed > 10%`, cost above threshold).

## 3. Alternatives considered

- **One tool for everything (e.g. only Grafana or only Amplitude).** Rejected: product funnel, prompt tracing with cost, and infra metrics are different data models; Langfuse is specialized for LLM and does not replace APM.
- **Jaeger/Tempo (distributed tracing).** Rejected: a monolith does not need it; for the pipeline Langfuse fills that role.
- **Elasticsearch for logs.** Rejected: heavy; Loki is enough.
- **Defer observability to “after MVP”.** Rejected for Sentry/structlog/Langfuse: without cost and errors from day one you cannot iterate on core quality and economics.

## 4. Consequences

### Positive
- Review cost is visible from the first call → unit economics under control, something to alert on.
- 👎 labeling from the Report closes the loop “product → dataset → prompts”.
- End-to-end `review_id` ties log, metric, and trace of one Review.

### Negative and risks
- Three systems = operational load. *Mitigation:* full infra layer in docker-compose; Sentry/Langfuse — free cloud tiers at the start.
- Self-hosted Langfuse adds services to compose. *Mitigation:* start with Langfuse cloud tier; self-host later if desired.
