# Deploy

Prod — Docker Compose on a VPS, Caddy on a single domain ([ADR 0004](../adr/0004-stack-fastapi-react.md)). HTTPS automatic, no CORS, no nginx.

## Infrastructure

- **VPS:** ~4 vCPU / 8 GB RAM / SSD (Timeweb / Selectel, ~1500–2000 ₽/mo). 8 GB leaves headroom for LibreOffice render, faster-whisper **medium** (~1.5 GB), weasyprint, `app`, `worker`, Postgres, Redis. On 4 GB without whisper you can hit OOM. Railway/Render — an option at the very start.
- **Domain + HTTPS:** Caddy obtains and renews the certificate itself (Let's Encrypt).
- **File storage:** local disk (volume) for MVP → S3-compatible (Cloudflare R2) as we grow; path abstracted behind `StorageBackend`.
- **Secrets** (`LLM_API_KEY`, `LANGFUSE_*`, `SMTP_*`, `SENTRY_DSN`, `DATABASE_URL`) — in `.env` on the host, **outside git**. `.env.example` in the repo lists every variable.

## docker-compose (one file: local and prod)

| Service | Role |
|---|---|
| `caddy` | HTTPS / HTTP, single domain → `app` (API + baked SPA) |
| `app` | FastAPI (uvicorn): REST API + file intake + Report static |
| `worker` | ARQ worker: Review pipeline (ingest → analysis → report), cron `cleanup_expired_files` |
| `db` | PostgreSQL 16 (volume) |
| `redis` | ARQ queue |

`app` and `worker` — one image, different start commands. `restart: unless-stopped` — come back after reboot. Observability (Grafana/Loki/Prometheus) for MVP is not in compose — configs may live under `deploy/` separately ([ADR 0007](../adr/0007-three-layer-observability.md)).

## Docker image (multi-stage)

One root `Dockerfile`; two top-level modules (`backend/`, `frontend/`) build into one image:

- **Stage 1 (build SPA):** `node` → `COPY frontend/` → `npm ci && vite build` → `frontend/dist`.
- **Stage 2 (backend):** `python:3.12-slim` + **LibreOffice headless**, **ffmpeg**, **RU fonts** (ttf-mscorefonts, PT Sans/Serif — otherwise Russian decks render as tofu squares). Dependencies via **uv** (`COPY backend/pyproject.toml backend/uv.lock` → `uv sync --frozen --no-dev`), then `COPY backend/`. Built SPA is placed in the image: `COPY --from=frontend /frontend/dist ./static` — **FastAPI serves both API and static** (as in the reference). The same image runs as `app` (uvicorn) and as `worker` (arq).

Caddy in front of the image only terminates TLS and proxies to app (single domain → no CORS). Post-build check: inside the container `soffice --headless --convert-to pdf` renders a Russian PPTX without tofu instead of letters.

## Database

SQLAlchemy 2.0 + **Alembic** (unlike one-shot prototypes — the product lives long, migrations are required). Deploy applies `alembic upgrade head`. Backups: `pg_dump` on cron + upload to S3; restore is verified on a clean machine at least once.

## CI/CD

Tracker — GitHub Issues/Projects (tickets in [tasks/](../tasks/)). Push to `main` → **GitHub Actions**:
1. lint (`ruff`, `eslint`) + fast pipeline unit tests (with mocked `LLMClient`);
2. image build;
3. deploy to VPS (SSH: `git pull && docker compose up -d --build && alembic upgrade head`).

Expensive integration tests (real VLM on a small deck) are marked `@pytest.mark.expensive` and do **not** run in CI — run manually.

## Emergency manual redeploy

```bash
ssh user@<vps>
cd ~/slidelens && git pull && docker compose up -d --build && docker compose exec app alembic upgrade head
```

## Privacy in prod

- `cleanup_expired_files` (ARQ cron, daily) deletes `FileAsset` rows with expired `expires_at` from Storage (US-8, [ADR 0007](../adr/0007-three-layer-observability.md)).
- Terms of service: files are deleted after N days, not used for training. Separately we state that slides are sent for analysis to an external VLM (Claude API), which under Anthropic’s API access terms does not train on submitted data — this must be explicit in the ToS for corporate users.

## Cost (MVP ballpark)

VPS ~1000 ₽/mo, domain ~1500 ₽/year. Main cost line — VLM API: ~$0.3–1.5 per Review of a 20-slide Deck with zooms. Measured from day one in Langfuse; on threshold breach — Telegram alert.
