# API-контракт

REST/JSON, префикс `/api/v1`. Источник правды — [api/openapi.yaml](../api/openapi.yaml) (при расхождении прав он); этот файл — прозаический компаньон. Термины — по [CONTEXT.md](../CONTEXT.md), модель данных — по [C4.md](C4.md).

Авторизация: `Authorization: Bearer <access_token>` (в токене — `user_id`). Access живёт коротко (~15 мин), обновляется по refresh (~30 дней) через `POST /auth/refresh`; логаут — клиентский (удалить токены). Токены на фронте хранятся в `localStorage` — это осознанный компромисс MVP (простота против XSS-риска); при появлении чувствительного контура кандидат на httpOnly-cookie. Разбор идёт в фоне — статус **поллится**, а не ждётся в запросе ([ADR 0003](../adr/0003-async-review-worker.md)).

## Общие коды ошибок

| Код | Когда |
|---|---|
| 401 | Нет/невалидный/протухший токен |
| 402 | Исчерпан лимит бесплатных Разборов |
| 404 | Сущность не найдена **или не принадлежит** пользователю (владение проверяется на сервере, US-8) |
| 409 | Недопустимое состояние (например, запрос Отчёта у Разбора не в `done`) |
| 413 | Файл больше 50 МБ |
| 422 | Ошибка валидации тела запроса |

Тело ошибки: `{ "detail": "...", "code": "LIMIT_REACHED" | null }`.

## DTO (кратко)

```jsonc
// User
{ "id": "uuid", "email": "a@b.ru", "plan": "free|paid",
  "free_reviews_left": 2, "email_verified": true, "is_admin": false, "created_at": "..." }

// ReviewOut (карточка/статус)
{ "id": "uuid", "status": "queued|processing|done|failed",
  "score": 74 | null,              // заполнен при done
  "fail_reason": "..." | null,     // заполнен при failed
  "deck_filename": "q3.pptx", "n_slides": 20 | null,
  "has_audio": true, "has_data": false,
  "created_at": "...", "finished_at": "..." | null }

// Finding
{ "id": "uuid", "slide_num": 7 | null,   // null = уровень деки
  "category": "TYPOGRAPHY|HIERARCHY|READABILITY|CONSISTENCY|CHART|NARRATIVE|SPEECH_MISMATCH|DELIVERY",
  "severity": "CRITICAL|MAJOR|MINOR",
  "title": "≤80 символов", "description": "...", "fix_suggestion": "...",
  "bbox": {"x":0.1,"y":0.2,"w":0.3,"h":0.1} | null,   // нормировано 0..1
  "screenshot_asset_id": "uuid" | null,  // аннотированный PNG
  "screenshot_url": "/api/v1/files/{uuid}?sig=..." | null,  // готовая ссылка, подписана заново на каждый GET /reviews/{id}/report — не нужен отдельный запрос за подписью
  "auto_fixable": true, "auto_fixed": false,
  "source": "SlideAnalyzer", "user_flag": false, "user_like": false }

// ReportOut
{ "review_id": "uuid", "score": 74, "n_slides": 20,
  "findings": [ /* Finding[] */ ],
  "delivery": { "words_per_minute": 138.5,
                "filler_words": {"ээ": 12, "как бы": 5},
                "long_pauses": [45.2, 132.8] } | null,   // только если было аудио
  "auto_fixed_count": 4,
  "pdf_asset_id": "uuid" | null, "fixed_pptx_asset_id": "uuid" | null,
  "fixed_pptx_filename": "q3-review_исправленная_версия№2.pptx" | null }
```

## Эндпоинты

### Auth

| Метод и путь | Доступ | Тело → Ответ |
|---|---|---|
| `POST /auth/register` | публичный | `{email, password}` → `201 {access, refresh, user}`; создаётся `plan=free`, `free_reviews_left=2` (сразу можно работать, без verify) |
| `POST /auth/login` | публичный | `{email, password}` → `200 {access, refresh, user}` |
| `POST /auth/refresh` | публичный | `{refresh_token}` → `200 {access, refresh, user}` |
| `GET /auth/me` | аутентиф. | → `200 User` |

### Reviews

| Метод и путь | Доступ | Тело → Ответ |
|---|---|---|
| `GET /reviews` | аутентиф. | → `200 [ReviewOut]` (только мои, новые сверху) |
| `POST /reviews` | аутентиф. | `multipart`: `deck` (обяз., PPTX/PDF ≤ 50 МБ, ≤ 60 слайдов) + опц. `audio` + `data` → `202 ReviewOut` (`queued`). Лимит исчерпан → `402` |
| `GET /reviews/{id}` | владелец | → `200 ReviewOut` (фронт поллит каждые 5 с до `done`/`failed`) |
| `GET /reviews/{id}/report` | владелец | только при `done` → `200 ReportOut`; иначе `409` |

**Лимит и `failed`.** `free_reviews_left` резервируется при `POST /reviews` (атомарно, `LimitService`). Если Разбор завершается `failed` (ошибка рендера/пайплайна), **кредит возвращается** — падение по нашей вине не должно стоить пользователю попытки. Отдельного эндпоинта ретрая нет: повторить = загрузить файл заново (это новый Разбор и новый резерв кредита).

**Администраторы** (`user.is_admin`, email из `ADMIN_EMAILS`) обходят `free_reviews_left` целиком — `LimitService.check_and_reserve` возвращает пользователя без резерва/декремента, независимо от `plan`.

### Findings / Files / Events

| Метод и путь | Доступ | Тело → Ответ |
|---|---|---|
| `POST /findings/{id}/flag` | владелец Разбора | 👎 → `204`; `user_flag=true`, `user_like=false` + score в Langfuse |
| `POST /findings/{id}/like` | владелец Разбора | 👍 → `204`; `user_like=true`, `user_flag=false` + score в Langfuse |
| `POST /findings/{id}/apply_fix` | владелец Разбора | точечный автофикс → `204`; переген `fixed.pptx` из оригинала по набору `auto_fixed`; не-`auto_fixable`/чужая → `404` |
| `GET /files/{asset_id}` | владелец | отдача файла через Storage. Для `<img>` (скриншоты) — короткоживущий `?sig=` вместо заголовка (тег не шлёт Authorization); прочие — по bearer. Чужой asset → `404` |
| `POST /events` | аутентиф. | `EventIn[]` (батч фронтовых событий: `report_opened`, `finding_expanded`, `pdf_downloaded`…) → `204` |

### Служебное

| Метод и путь | Доступ | Ответ |
|---|---|---|
| `GET /health` | публичный | `200 {"status":"ok"}` — смоук после деплоя |

## Правила видимости и приватности (сервер обязан обеспечивать)

- Разбор, Находки, файлы одного пользователя недоступны другому — проверка владельца на сервере, чужое → `404` (не `403`, чтобы не палить существование).
- `GET /files/{asset_id}` со скриншотом использует signed-`sig` с коротким TTL; истёк → `401`.
- Файлы (`FileAsset`) имеют `expires_at`; после истечения удаляются периодической задачей воркера и `404` на выдаче (US-8).
- Антифрод бесплатного лимита — через `free_reviews_left` (без обязательного подтверждения почты в MVP).
