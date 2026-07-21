# Бэклог и план по фазам

Соло-разработка по вечерам с AI-помощниками (5–10 ч/нед): задача = 1–2 вечера, спринты по 2 недели. Модули — по [struktura проекта](#структура-репозитория-целевая) ниже. Эпики = milestone'ы, задачи = issues.

**Критический путь:** каркас → `LLMClient` → ингест → пер-слайдовый анализ → **метрика качества** (recall/мусор). Пока качество ядра не дотянуто до цели — веб вокруг слабого ядра не строим.

## Фазы

| Фаза | Содержание | Оценка |
|---|---|---|
| 0. Фундамент | репозиторий, [CLAUDE.md](../CLAUDE.md), Docker (FastAPI+Postgres+LibreOffice), spike VLM на 3 слайдах, тестовый набор дек | ~12 ч |
| 1. Ядро пайплайна ⭐ | ингест → анализаторы → агрегация → аннотация → автофиксы → отчёт; **самое важное**, ~60 % усилий | ~50 ч |
| 2. Платформа | auth, кабинет, загрузка, воркер, страница Отчёта, PDF, лендинг, деплой + observability | ~35 ч |
| 3. Валидация и оплата | раздача бесплатных Разборов, обратная связь, тарифы + ЮKassa | ~10 ч |
| 4. Режим репетиции | запись питча в браузере, точный тайминг по слайдам, динамика между прогонами | ~14 ч |

## Фаза 0 — Фундамент

- **Каркас репозитория + CLAUDE.md.** Дерево из [struktura](#структура-репозитория-целевая), правило «`backend/core/` не импортирует `backend/app/`» ([ADR 0001](../adr/0001-pipeline-pure-library.md)), контракты `Finding`, соглашение о промптах.
- **config.py + database.py.** `Settings` (все ENV с валидацией), async engine, `get_db()`, `.env.example`.
- **Docker Compose (dev).** app / worker / db / redis; Dockerfile с LibreOffice + ffmpeg + RU-шрифтами; `GET /health`.
- **Observability с первого дня** ([ADR 0007](../adr/0007-three-layer-observability.md)): structlog (JSON, `review_id` в contextvars), Sentry, Langfuse-обёртка вызовов.
- **Spike:** 3 слайда через VLM вручную → `docs/spike-notes.md` (что ловит, $/слайд, таксономия Категорий).
- **Тестовый набор:** 10–15 дек (≥5 RU, ≥3 с графиками, 2–3 плохих); 3 размечены в `backend/tests/golden/`.

## Фаза 1 — Ядро пайплайна ⭐

- **`backend/core/schemas.py`** — все контракты (`Finding`, `Category`, `Severity`, `BBox`, `TranscriptSegment`, `DeliveryMetrics`, `ChartReading`, `SuspiciousRegion`, `ReviewResult`). Только pydantic.
- **`LLMClient` + `PromptRegistry` + `ReviewContext`** — vision + structured output, retry, backoff, Langfuse-span, стоимость; промпты-файлы с frontmatter.
- **`DeckIngestor` + `AudioExtractor` + `Transcriber`** — PPTX/PDF → PNG; аудио → WAV → faster-whisper; `compute_delivery`.
- **`BaseAnalyzer` + `PipelineOrchestrator` + CLI** — graceful degradation, порядок из конфига, `python -m core.run`.
- **Анализаторы:** `SlideAnalyzer` (v1 промпт) → `ZoomAgent` → `DeckAnalyzer` → `ChartChecker` → `CrossModalAnalyzer` ([ADR 0002](../adr/0002-vlm-pipeline-hybrid-analyzers.md), [ADR 0005](../adr/0005-crossmodal-delivery-analysis.md)).
- **Сборка:** `Aggregator` + `DeckScorer`, `Annotator` (Pillow), `PptxFixer` ([ADR 0006](../adr/0006-pptx-autofix-strategy.md)), `ReportBuilder` + `PdfExporter`.
- **Eval-скрипт** `backend/tests/golden/eval.py` → `docs/quality-log.md`. **Критерий готовности фазы: recall ≥ 70 %, мусор < 20 %.**

## Фаза 2 — Платформа

- ORM-модели + Alembic (User, Review, FindingRow, FileAsset, Event, Rehearsal-пустая).
- Auth API (fastapi-users, JWT access+refresh, верификация почты).
- Сервисы: `Storage`, `EmailService`, `EventTracker`, `LimitService` (SELECT FOR UPDATE, 402).
- `ReviewService` + `backend/worker/tasks.py` ([ADR 0003](../adr/0003-async-review-worker.md)); Reviews/Findings/Files/Events API ([API.md](API.md)).
- Frontend: каркас SPA, генерация типов из OpenAPI, auth-страницы, Кабинет + загрузка, **страница Отчёта** (витрина), лендинг с живым примером ([DESIGN.md](DESIGN.md)).
- Деплой на VPS + Grafana/Loki/Prometheus + алерты ([DEPLOY.md](DEPLOY.md)).

## Фаза 3 — Валидация и монетизация

- Раздать 30–50 бесплатных Разборов (консультанты/аналитики/стартаперы), собрать «за что заплатил бы».
- Контент-маркетинг: публичные разборы известных презентаций.
- Тарифы + ЮKassa (идемпотентный webhook), воронка-дашборд, пополнение golden-разметки из 👎.

## Фаза 4 — Режим репетиции

- Страница репетиции: показ слайдов + `MediaRecorder` (аудио + таймкоды переключений → точные `SlideTiming`).
- `CrossModalAnalyzer` на точном тайминге: карта тайминга, слайды-«болота» (>2 мин) и «заглушки» (<5 с), mismatch с таймкодами.
- Динамика между прогонами (`attempt_num`) → повод для подписки, а не разовой оплаты.

## Структура репозитория (целевая)

Два верхнеуровневых модуля — **`backend/`** и **`frontend/`**, всё остальное внутри них (как в референсном проекте). Сборка — один multi-stage `Dockerfile`: стадия 1 собирает SPA (`vite build`), стадия 2 — backend-образ, куда `dist` кладётся в `./static`; тот же образ запускается как `app` и как `worker`.

```
slidelens/
├── backend/                  # Python-проект (uv + pyproject.toml + uv.lock)
│   ├── app/                  # веб-слой: main.py, config.py, db.py, deps.py, security.py,
│   │                         #   api/v1/ (роуты), services/, schemas/, models/, seed.py
│   ├── core/                 # чистая библиотека: run.py, context.py, schemas.py, llm/, ingest.py,
│   │                         #   transcribe.py, analyzers/, aggregate.py, annotate.py, fix.py,
│   │                         #   report.py, prompts/ (версионированные md)
│   ├── worker/               # tasks.py: process_review, cleanup_expired_files (связывает app и core)
│   ├── observability/        # setup.py: structlog, Sentry, метрики
│   ├── migrations/           # Alembic
│   └── tests/                # unit/ · integration/ · golden/ (деки + разметка + eval)
├── frontend/                 # React + Vite + TS SPA
│   └── src/                  # pages/, components/, api/, hooks/, auth/, lib/, App.tsx, main.tsx
├── Dockerfile                # multi-stage: frontend build → backend-образ (dist → ./static)
├── docker-compose.yml · Makefile · CLAUDE.md · README.md
└── tasks/                    # тикеты разработки (см. tasks/README.md)
```

Ключевое правило: `backend/core/` **не** импортирует `backend/app/` — их связывает только `backend/worker/tasks.py` ([ADR 0001](../adr/0001-pipeline-pure-library.md)). Разделение на `app` / `core` / `worker` живёт внутри `backend/`, а не на верхнем уровне репозитория.
