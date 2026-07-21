# Деплой

Прод — Docker Compose на VPS, Caddy на одном домене ([ADR 0004](../adr/0004-stack-fastapi-react.md)). HTTPS автоматом, без CORS и nginx.

## Инфраструктура

- **VPS:** ~4 vCPU / 8 GB RAM / SSD (Timeweb / Selectel, ~1500–2000 ₽/мес). 8 ГБ — с запасом под LibreOffice-рендер, faster-whisper **medium** (~1.5 ГБ), weasyprint, `app`, `worker`, Postgres, Redis. На 4 ГБ без whisper можно упереться в OOM. Railway/Render — вариант на самом старте.
- **Домен + HTTPS:** Caddy получает и продлевает сертификат сам (Let's Encrypt).
- **Хранилище файлов:** локальный диск (volume) на MVP → S3-совместимое (Cloudflare R2) по мере роста; путь абстрагирован за `StorageBackend`.
- **Секреты** (`LLM_API_KEY`, `LANGFUSE_*`, `SMTP_*`, `SENTRY_DSN`, `DATABASE_URL`) — в `.env` на хосте, **вне git**. `.env.example` в репо перечисляет все переменные.

## docker-compose (один файл: локально и прод)

| Сервис | Роль |
|---|---|
| `caddy` | HTTPS / HTTP, один домен → `app` (API + baked SPA) |
| `app` | FastAPI (uvicorn): REST API + приём файлов + статика Отчёта |
| `worker` | ARQ-воркер: пайплайн Разбора (ingest → анализ → отчёт), cron `cleanup_expired_files` |
| `db` | PostgreSQL 16 (volume) |
| `redis` | очередь ARQ |

`app` и `worker` — один образ, разные команды запуска. `restart: unless-stopped` — поднимаются после ребута. Observability (Grafana/Loki/Prometheus) для MVP не в compose — конфиги могут лежать в `deploy/` отдельно ([ADR 0007](../adr/0007-three-layer-observability.md)).

## Docker-образ (multi-stage)

Один `Dockerfile` в корне; два верхнеуровневых модуля (`backend/`, `frontend/`) собираются в один образ:

- **Stage 1 (build SPA):** `node` → `COPY frontend/` → `npm ci && vite build` → `frontend/dist`.
- **Stage 2 (backend):** `python:3.12-slim` + **LibreOffice headless**, **ffmpeg**, **RU-шрифты** (ttf-mscorefonts, PT Sans/Serif — иначе русские деки рендерятся квадратами). Зависимости — через **uv** (`COPY backend/pyproject.toml backend/uv.lock` → `uv sync --frozen --no-dev`), затем `COPY backend/`. Собранный SPA кладётся в образ: `COPY --from=frontend /frontend/dist ./static` — **FastAPI отдаёт и API, и статику** (как в референсе). Тот же образ запускается как `app` (uvicorn) и как `worker` (arq).

Caddy перед образом только терминирует TLS и проксирует на app (один домен → без CORS). Проверка после сборки: внутри контейнера `soffice --headless --convert-to pdf` рендерит русский PPTX без квадратов вместо букв.

## База данных

SQLAlchemy 2.0 + **Alembic** (в отличие от одноразовых прототипов — продукт живёт долго, миграции нужны). Деплой применяет `alembic upgrade head`. Бэкапы: `pg_dump` по cron + выгрузка в S3; восстановление проверяется на чистой машине хотя бы раз.

## CI/CD

Трекер — GitHub Issues/Projects (тикеты — в [tasks/](../tasks/)). Пуш в `main` → **GitHub Actions**:
1. lint (`ruff`, `eslint`) + быстрые юнит-тесты пайплайна (с замоканным `LLMClient`);
2. сборка образа;
3. деплой на VPS (SSH: `git pull && docker compose up -d --build && alembic upgrade head`).

Дорогие интеграционные тесты (реальный VLM на маленькой деке) помечены `@pytest.mark.expensive` и **не** идут в CI — гоняются вручную.

## Аварийный ручной редеплой

```bash
ssh user@<vps>
cd ~/slidelens && git pull && docker compose up -d --build && docker compose exec app alembic upgrade head
```

## Приватность в проде

- `cleanup_expired_files` (cron ARQ, ежедневно) удаляет `FileAsset` с истёкшим `expires_at` из Storage (US-8, [ADR 0007](../adr/0007-three-layer-observability.md)).
- Оферта: файлы удаляются через N дней, не используются для обучения. Отдельно фиксируем, что слайды уходят на анализ во внешний VLM (Claude API), который по условиям API-доступа Anthropic не обучается на переданных данных, — это должно быть явно в оферте для корпоративных пользователей.

## Стоимость (ориентир MVP)

VPS ~1000 ₽/мес, домен ~1500 ₽/год. Основная статья — VLM API: ~0.3–1.5 $ за Разбор 20-слайдовой Деки с зумами. Меряется с первого дня в Langfuse; при превышении порога — алерт в Telegram.
