# ТЗ: Quiz-платформа (FastAPI + PostgreSQL 18 + Centrifugo + React)

> Источник: https://claude.ai/artifact/1SC9WkU7pxcN4Tfncow7bE (выгружено 2026-11-08). Источник истины для агента теперь этот файл. Отклонения записываются в `docs/decisions.md`.

2026-11-04 · @maxim

## 1. Обзор продукта

Веб-платформа для создания квизов и их синхронного прохождения группой игроков в реальном времени. MVP: регистрация, личный кабинет, конструктор тестов, публичная библиотека вопросов, игровые комнаты с live-синхронизацией и итоговой статистикой.

### Роли

| Роль | Что может |
|---|---|
| Гость | Регистрация, вход, просмотр публичной библиотеки вопросов |
| Пользователь | Создавать тесты и вопросы, публиковать вопросы, импортировать чужие публичные вопросы в свои тесты, вступать в комнаты по приглашению |
| Хост комнаты | Пользователь, создавший комнату: приглашает игроков, стартует игру, может завершить её досрочно или исключить игрока |
| Игрок | Участник комнаты: отмечает «Готов», отвечает на вопросы, видит итоги |

### Глоссарий

- **Тест (Quiz)** — упорядоченный набор вопросов с настройками (время на вопрос, перемешивание).
- **Вопрос (Question)** — текст, тип проверки, варианты ответов или эталонные текстовые ответы. Живёт отдельно от теста и может входить в несколько тестов.
- **Тип вопроса** — `single` (один верный), `multiple` (несколько верных), `text` (свободный ввод, проверка на бэкенде).
- **Категория** — одна на вопрос, выбирается из списка или создаётся пользователем.
- **Тэг** — свободные метки, много на вопрос.
- **Комната (Room)** — экземпляр игры по конкретному тесту с набором игроков.
- **Раунд** — показ одного вопроса всем игрокам комнаты с общим таймером.

### Ключевые сценарии

1. Пользователь регистрируется и попадает в личный кабинет.
2. Создаёт тест, добавляет вопросы трёх типов, задаёт категорию и тэги.
3. Отмечает вопросы как публичные — они появляются в библиотеке с фильтрами по категории и тэгам.
4. Создаёт комнату из теста, приглашает пользователей (по нику или по ссылке-коду).
5. Все игроки в лобби нажимают «Готов», хост стартует игру.
6. Вопросы показываются синхронно, у каждого игрока свой ответ, он видит только свои действия.
7. После финала каждый видит свои ответы, ответы других, время ответов, рейтинг и график хода игры.

### Вне рамок MVP

- Оплата, рейтинги между играми, медиа в вопросах (картинки/видео) — только задел в схеме данных.
- Мобильное приложение — только адаптивная вёрстка.

## 2. Стек и архитектура

Четыре сервиса: React SPA, FastAPI, Centrifugo и PostgreSQL 18. Клиент меняет состояние только через REST, бэкенд пишет в PostgreSQL и после коммита публикует событие в Centrifugo, Centrifugo рассылает его подписчикам по WebSocket.

> Диаграмма «архитектура · 4 компонента, 4 связи» есть только в исходном документе.

В проде перед всем стоит Nginx: `/api` → FastAPI, `/connection` → Centrifugo, `/` → статика фронта. Для нескольких инстансов Centrifugo подключается Redis как брокер — в MVP не нужен.

| Слой | Технология | Роль |
|---|---|---|
| Frontend | React 19, TypeScript, Vite | SPA: кабинет, конструктор, библиотека, комнаты, итоги |
| Backend | Python 3.13, FastAPI, SQLAlchemy 2.0 async | REST API, бизнес-логика, игровой движок, таймеры |
| Realtime | Centrifugo v6 | WebSocket-доставка событий, presence, восстановление после обрыва |
| БД | PostgreSQL 18 | всё постоянное состояние, включая состояние игры |
| Инфраструктура | Docker Compose, Nginx | локальный запуск одной командой, прокси в проде |

### Архитектурные принципы

- **Сервер — источник истины.** Таймеры, время ответа, проверка и очки считаются только на бэкенде.
- **Команды через REST, события через Centrifugo.** Клиент не публикует в каналы, что упрощает авторизацию и валидацию.
- **Состояние в БД, а не в памяти процесса.** Рестарт бэкенда не теряет игру; `GET /rooms/{id}` всегда возвращает полную картину.
- **Снимки вопросов.** Раунд хранит копию вопроса, поэтому правка теста не влияет на идущие и прошедшие игры.
- **Модульный монолит.** Один сервис FastAPI с чёткими доменными модулями — проще для MVP и для работы ИИ-агента.

## 3. Модель данных (PostgreSQL 18)

Первичные ключи — UUIDv7 через встроенную в PostgreSQL 18 функцию `uuidv7()`: они сортируемы по времени и хорошо ложатся в B-tree. Миграции — Alembic, схема ниже — эталон для первой миграции.

```sql
CREATE EXTENSION IF NOT EXISTS citext;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS fuzzystrmatch;

CREATE TYPE question_type AS ENUM ('single', 'multiple', 'text');
CREATE TYPE room_status   AS ENUM ('lobby', 'countdown', 'in_progress', 'finished', 'cancelled');
CREATE TYPE invite_status AS ENUM ('pending', 'accepted', 'declined');

-- Пользователи и сессии
CREATE TABLE users (
  id            uuid PRIMARY KEY DEFAULT uuidv7(),
  email         citext UNIQUE NOT NULL,
  username      citext UNIQUE NOT NULL,        -- 3..32, [a-z0-9_]
  display_name  text NOT NULL,
  password_hash text NOT NULL,                 -- argon2id
  created_at    timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE refresh_tokens (
  id          uuid PRIMARY KEY DEFAULT uuidv7(),
  user_id     uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token_hash  text NOT NULL UNIQUE,
  expires_at  timestamptz NOT NULL,
  revoked_at  timestamptz
);

-- Справочники
CREATE TABLE categories (
  id          uuid PRIMARY KEY DEFAULT uuidv7(),
  name        text NOT NULL,
  slug        text NOT NULL UNIQUE,            -- нормализованное имя, защита от дублей
  is_system   boolean NOT NULL DEFAULT false,  -- предзаполненные
  created_by  uuid REFERENCES users(id) ON DELETE SET NULL,
  created_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE tags (
  id    uuid PRIMARY KEY DEFAULT uuidv7(),
  name  citext NOT NULL UNIQUE                 -- lower-case, 1..32 символа
);

-- Вопросы
CREATE TABLE questions (
  id                 uuid PRIMARY KEY DEFAULT uuidv7(),
  author_id          uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  type               question_type NOT NULL,
  text               text NOT NULL,            -- до 2000 символов
  explanation        text,                     -- показывается в итогах
  category_id        uuid REFERENCES categories(id) ON DELETE SET NULL,
  is_public          boolean NOT NULL DEFAULT false,
  points             int NOT NULL DEFAULT 1000 CHECK (points > 0),
  time_limit_sec     int CHECK (time_limit_sec BETWEEN 5 AND 300), -- NULL = из настроек теста
  text_check         jsonb,                    -- для type='text': {case_sensitive, fuzzy_distance}
  source_question_id uuid REFERENCES questions(id) ON DELETE SET NULL, -- если скопирован из библиотеки
  created_at         timestamptz NOT NULL DEFAULT now(),
  updated_at         timestamptz NOT NULL DEFAULT now(),
  deleted_at         timestamptz
);

CREATE TABLE question_options (       -- для single / multiple
  id          uuid PRIMARY KEY DEFAULT uuidv7(),
  question_id uuid NOT NULL REFERENCES questions(id) ON DELETE CASCADE,
  text        text NOT NULL,
  is_correct  boolean NOT NULL DEFAULT false,
  position    int NOT NULL
);

CREATE TABLE question_text_answers (  -- для text: список допустимых ответов
  id          uuid PRIMARY KEY DEFAULT uuidv7(),
  question_id uuid NOT NULL REFERENCES questions(id) ON DELETE CASCADE,
  answer      text NOT NULL,
  normalized  text NOT NULL             -- результат normalize(answer)
);

CREATE TABLE question_tags (
  question_id uuid REFERENCES questions(id) ON DELETE CASCADE,
  tag_id      uuid REFERENCES tags(id) ON DELETE CASCADE,
  PRIMARY KEY (question_id, tag_id)
);

-- Тесты
CREATE TABLE quizzes (
  id                     uuid PRIMARY KEY DEFAULT uuidv7(),
  author_id              uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  title                  text NOT NULL,
  description            text,
  default_time_limit_sec int NOT NULL DEFAULT 30,
  shuffle_questions      boolean NOT NULL DEFAULT false,
  shuffle_options        boolean NOT NULL DEFAULT true,
  created_at             timestamptz NOT NULL DEFAULT now(),
  updated_at             timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE quiz_questions (
  quiz_id     uuid REFERENCES quizzes(id) ON DELETE CASCADE,
  question_id uuid REFERENCES questions(id) ON DELETE RESTRICT,
  position    int NOT NULL,
  PRIMARY KEY (quiz_id, question_id)
);

-- Игра
CREATE TABLE rooms (
  id                  uuid PRIMARY KEY DEFAULT uuidv7(),
  quiz_id             uuid NOT NULL REFERENCES quizzes(id),
  host_id             uuid NOT NULL REFERENCES users(id),
  code                text NOT NULL UNIQUE,     -- 6 символов, для ссылки-приглашения
  status              room_status NOT NULL DEFAULT 'lobby',
  current_round_index int,
  settings            jsonb NOT NULL,           -- снимок настроек теста на момент создания
  created_at          timestamptz NOT NULL DEFAULT now(),
  started_at          timestamptz,
  finished_at         timestamptz
);

CREATE TABLE room_invites (
  id          uuid PRIMARY KEY DEFAULT uuidv7(),
  room_id     uuid NOT NULL REFERENCES rooms(id) ON DELETE CASCADE,
  user_id     uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  invited_by  uuid NOT NULL REFERENCES users(id),
  status      invite_status NOT NULL DEFAULT 'pending',
  created_at  timestamptz NOT NULL DEFAULT now(),
  UNIQUE (room_id, user_id)
);

CREATE TABLE room_players (
  room_id     uuid REFERENCES rooms(id) ON DELETE CASCADE,
  user_id     uuid REFERENCES users(id) ON DELETE CASCADE,
  is_ready    boolean NOT NULL DEFAULT false,
  joined_at   timestamptz NOT NULL DEFAULT now(),
  left_at     timestamptz,
  final_score int,
  final_rank  int,
  PRIMARY KEY (room_id, user_id)
);

CREATE TABLE room_rounds (
  id                uuid PRIMARY KEY DEFAULT uuidv7(),
  room_id           uuid NOT NULL REFERENCES rooms(id) ON DELETE CASCADE,
  index             int NOT NULL,
  question_id       uuid REFERENCES questions(id) ON DELETE SET NULL,
  question_snapshot jsonb NOT NULL,   -- текст, тип, варианты, верные ответы, лимит, очки
  started_at        timestamptz NOT NULL,
  deadline_at       timestamptz NOT NULL,
  ended_at          timestamptz,
  UNIQUE (room_id, index)
);

CREATE TABLE answers (
  id                  uuid PRIMARY KEY DEFAULT uuidv7(),
  round_id            uuid NOT NULL REFERENCES room_rounds(id) ON DELETE CASCADE,
  room_id             uuid NOT NULL REFERENCES rooms(id) ON DELETE CASCADE,
  user_id             uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  selected_option_ids uuid[],          -- single / multiple
  text_answer         text,            -- text
  is_correct          boolean NOT NULL,
  partial_ratio       numeric(4,3),    -- для multiple: доля верности 0..1
  score               int NOT NULL,
  response_time_ms    int NOT NULL,    -- answered_at - round.started_at, считает сервер
  answered_at         timestamptz NOT NULL DEFAULT now(),
  UNIQUE (round_id, user_id)           -- один ответ на раунд, без переответа
);

-- Индексы
CREATE INDEX ON questions (author_id) WHERE deleted_at IS NULL;
CREATE INDEX ON questions (category_id, created_at DESC) WHERE is_public AND deleted_at IS NULL;
CREATE INDEX questions_text_trgm ON questions USING gin (text gin_trgm_ops);
CREATE INDEX ON question_tags (tag_id);
CREATE INDEX ON quiz_questions (quiz_id, position);
CREATE INDEX ON room_players (user_id);
CREATE INDEX ON room_invites (user_id, status);
CREATE INDEX ON answers (room_id, user_id);
```

### Правила целостности (проверяются в сервисном слое)

- `single`: минимум 2 варианта, ровно 1 верный.
- `multiple`: минимум 2 варианта, минимум 1 верный.
- `text`: минимум 1 допустимый ответ в `question_text_answers`, вариантов нет.
- Публичный вопрос обязан иметь категорию. Редактировать и удалять вопрос может только автор; чужой публичный вопрос добавляется в свой тест через копирование (`source_question_id`).
- Вопрос, который уже использовался в комнате, удаляется мягко (`deleted_at`), а в раунде остаётся `question_snapshot` — история игр не ломается при правке теста.
- Новая категория: имя нормализуется в `slug` (lower, trim, схлопывание пробелов, ё→е); при совпадении slug возвращается существующая категория вместо создания дубля.

## 4. Backend (FastAPI)

Монолит на FastAPI, разбитый на доменные модули с трёхслойной структурой: `router` (HTTP, схемы Pydantic) → `service` (бизнес-логика, транзакции) → `repository` (SQLAlchemy 2.0 async). Весь ввод-вывод асинхронный. Состояние игры хранится в PostgreSQL, сервер — единственный источник истины для таймеров, проверки ответов и очков.

### Библиотеки

| Назначение | Выбор |
|---|---|
| Python | 3.13 |
| Web | FastAPI, Uvicorn |
| ORM / драйвер | SQLAlchemy 2.0 (async), asyncpg |
| Миграции | Alembic |
| Схемы и конфиг | Pydantic v2, pydantic-settings |
| Пароли | argon2-cffi (argon2id) |
| JWT | PyJWT |
| HTTP к Centrifugo | httpx (server API) или `cent` SDK |
| Фоновые таймеры раундов | APScheduler / собственный asyncio-воркер + `SELECT … FOR UPDATE SKIP LOCKED` |
| Rate limit | slowapi |
| Тесты | pytest, pytest-asyncio, httpx AsyncClient, testcontainers-postgres |
| Линт / типы | ruff, mypy |

### Модули

- `auth` — регистрация, логин, refresh/logout, выпуск токенов подключения Centrifugo.
- `users` — профиль, поиск пользователей по нику для приглашений.
- `catalog` — категории и тэги (список, автодополнение, создание своей категории).
- `questions` — CRUD своих вопросов, публикация, публичная библиотека, копирование в свой тест.
- `quizzes` — CRUD тестов, порядок вопросов.
- `rooms` — создание комнаты, приглашения, лобби, готовность, старт.
- `game` — движок раундов: показ вопроса, приём ответа, проверка, завершение раунда и игры.
- `results` — итоги комнаты, статистика, данные для графиков.
- `realtime` — обёртка над Centrifugo server API (`publish`, `broadcast`, `disconnect`) и генерация JWT для клиента.

### REST API (префикс `/api/v1`)

| Метод | Путь | Назначение |
|---|---|---|
| POST | `/auth/register` | email, username, password, display_name → пользователь + токены |
| POST | `/auth/login` | email/username + пароль → access (15 мин) + refresh (30 дней, httpOnly cookie) |
| POST | `/auth/refresh` | ротация refresh-токена |
| POST | `/auth/logout` | отзыв refresh-токена |
| GET | `/auth/centrifugo-token` | JWT подключения к Centrifugo (`sub` = user_id, TTL 10 мин) |
| GET | `/users/me` | профиль |
| GET | `/users/search?q=` | поиск по нику для приглашений (до 10 результатов) |
| GET | `/categories?q=` | список и автодополнение |
| POST | `/categories` | создать свою категорию (дедупликация по slug) |
| GET | `/tags?q=` | автодополнение тэгов |
| GET | `/questions/mine` | мои вопросы, фильтры: type, category, tag, q |
| POST | `/questions` | создать вопрос с вариантами/ответами и тэгами |
| GET / PATCH / DELETE | `/questions/{id}` | чтение / правка / удаление (только автор) |
| POST | `/questions/{id}/publish` | сделать публичным (обязательна категория) |
| POST | `/questions/{id}/unpublish` | снять с публикации |
| GET | `/library/questions` | публичные вопросы: category, tags[], q, sort, курсорная пагинация; верные ответы скрыты |
| POST | `/library/questions/{id}/copy` | скопировать в свои, опционально сразу в `quiz_id` |
| GET / POST | `/quizzes` | мои тесты / создать |
| GET / PATCH / DELETE | `/quizzes/{id}` | тест с вопросами / правка настроек / удаление |
| PUT | `/quizzes/{id}/questions` | полный порядок вопросов: `[question_id, …]` |
| POST | `/rooms` | создать комнату из `quiz_id` (тест должен содержать ≥ 1 вопрос) |
| GET | `/rooms/{id}` | состояние комнаты: статус, игроки, готовность, текущий раунд (без верных ответов) |
| GET | `/rooms/by-code/{code}` | найти комнату по коду ссылки |
| POST | `/rooms/{id}/invites` | пригласить `user_ids[]` (только хост) |
| GET | `/invites` | мои входящие приглашения |
| POST | `/invites/{id}/accept` · `/decline` | ответ на приглашение |
| POST | `/rooms/{id}/join` | вход по коду (лобби, не больше `max_players`) |
| POST | `/rooms/{id}/leave` | выйти |
| POST | `/rooms/{id}/ready` | `{ready: bool}` |
| POST | `/rooms/{id}/start` | только хост, все игроки готовы, игроков ≥ 1 |
| POST | `/rooms/{id}/rounds/{index}/answer` | ответ игрока: `option_ids[]` или `text` |
| POST | `/rooms/{id}/cancel` | хост отменяет игру |
| DELETE | `/rooms/{id}/players/{user_id}` | хост исключает игрока в лобби |
| GET | `/rooms/{id}/results` | итоги: рейтинг, ответы всех, тайминги, серии для графиков |
| GET | `/rooms/history` | мои сыгранные комнаты |

Все мутации игры идут через REST, а не через сообщения клиента в Centrifugo: так проще валидация, авторизация и тесты. Centrifugo только доставляет события.

### Формат ошибок

Единый JSON: `{"error": {"code": "ROUND_CLOSED", "message": "…", "details": {}}}`. Коды: `VALIDATION_ERROR` 422, `UNAUTHORIZED` 401, `FORBIDDEN` 403, `NOT_FOUND` 404, `ROOM_NOT_IN_LOBBY` 409, `NOT_ALL_READY` 409, `ALREADY_ANSWERED` 409, `ROUND_CLOSED` 409, `ROOM_FULL` 409, `RATE_LIMITED` 429.

### Пример: создание вопроса

```json
POST /api/v1/questions
{
  "type": "multiple",
  "text": "Какие из этих СУБД реляционные?",
  "category": {"id": null, "name": "Базы данных"},
  "tags": ["sql", "backend"],
  "options": [
    {"text": "PostgreSQL", "is_correct": true},
    {"text": "MongoDB",    "is_correct": false},
    {"text": "MySQL",      "is_correct": true}
  ],
  "time_limit_sec": 20,
  "points": 1000,
  "explanation": "MongoDB — документная СУБД."
}
```

Для `text`: вместо `options` передаётся `"text_answers": ["Москва", "г. Москва"]` и `"text_check": {"case_sensitive": false, "fuzzy_distance": 1}`.

### Проверка текстовых ответов

1. `normalize(s)`: Unicode NFKC, trim, lower (если не `case_sensitive`), ё→е, схлопывание пробелов, удаление пунктуации по краям.
2. Совпадение, если нормализованный ответ равен любому `normalized` из допустимых.
3. Иначе, если `fuzzy_distance > 0`, засчитывается при расстоянии Левенштейна ≤ `fuzzy_distance` (только для ответов длиной ≥ 5 символов).
4. Логика — чистая функция в `game/checker.py` с юнит-тестами на каждый тип.

## 5. Realtime (Centrifugo)

Centrifugo v6 работает как отдельный сервис доставки событий: бэкенд публикует через server HTTP API, клиент только подписывается. Клиент ничего не публикует в каналы — все действия идут через REST.

### Подключение и авторизация

1. Фронт получает JWT подключения: `GET /auth/centrifugo-token` (HS256, `sub` = user_id, `exp` 10 мин), передаёт его в `centrifuge-js` через `getToken` — SDK сам обновляет токен.
2. Для комнаты фронт запрашивает токен подписки: `GET /rooms/{id}/subscription-token` (claims `sub`, `channel`, `exp`). Бэкенд выдаёт его только участнику комнаты.
3. Личный канал `personal:#<user_id>` — user-limited канал: подписаться может только пользователь с этим `sub`, токен подписки не нужен.

### Каналы

| Канал | Кто подписан | Назначение | Опции namespace |
|---|---|---|---|
| `room:<room_id>` | участники комнаты | состояние лобби и игры, общие события | presence, join_leave, history 100 сообщений / 30 мин, force_recovery |
| `personal:#<user_id>` | сам пользователь | приглашения, личный результат ответа, личные итоги раунда | allow_user_limited_channels, history 50 / 10 мин, force_recovery |

### События

Каждое сообщение — `{"type": "...", "seq": <int>, "server_time": "<ISO>", "data": {...}}`. `seq` монотонно растёт в рамках комнаты, клиент отбрасывает устаревшие события.

| type | Канал | Когда | data |
|---|---|---|---|
| `invite.received` | personal | хост пригласил | room_id, code, quiz_title, host |
| `lobby.player_joined` | room | игрок вошёл | user (id, display_name) |
| `lobby.player_left` | room | вышел или исключён | user_id, reason |
| `lobby.ready_changed` | room | нажал / снял «Готов» | user_id, is_ready, ready_count, total |
| `game.countdown` | room | хост нажал «Старт» | starts_at (через 3 с) |
| `round.started` | room | начало раунда | index, total, question (без верных ответов), started_at, deadline_at |
| `round.progress` | room | кто-то ответил | answered_count, total (без содержимого ответов) |
| `answer.accepted` | personal | ответ принят | round_index, response_time_ms (результат скрыт до конца раунда) |
| `round.ended` | room | дедлайн или ответили все | index, correct (option_ids или допустимые ответы), explanation |
| `round.my_result` | personal | конец раунда | is_correct, score, total_score, rank |
| `game.finished` | room | последний раунд закрыт | room_id → фронт загружает `GET /rooms/{id}/results` |
| `game.cancelled` | room | хост отменил | reason |

Во время игры игрок видит только свои действия и обезличенный прогресс «ответили 3 из 5». Чужие ответы и таблица результатов открываются после `game.finished`.

### Синхронизация времени

Таймер на клиенте считается от серверных `started_at` / `deadline_at`. При подключении и в каждом событии приходит `server_time`, фронт хранит смещение `offset = server_time − Date.now()` и рисует таймер по `deadline_at − (Date.now() + offset)`. Время ответа фиксирует только сервер.

### Переподключение

При обрыве `centrifuge-js` восстанавливает пропущенные сообщения из истории (recovery). Если восстановление не удалось (`wasRecovering && !recovered`), фронт заново грузит `GET /rooms/{id}` — этот ответ всегда содержит полное актуальное состояние, включая текущий раунд и свой ответ на него.

### Пример конфигурации (v6)

```json
{
  "client": {
    "allowed_origins": ["http://localhost:5173"],
    "token": { "hmac_secret_key": "<CENTRIFUGO_TOKEN_SECRET>" }
  },
  "http_api": { "key": "<CENTRIFUGO_API_KEY>" },
  "channel": {
    "namespaces": [
      {
        "name": "room",
        "presence": true,
        "join_leave": true,
        "force_push_join_leave": true,
        "history_size": 100,
        "history_ttl": "30m",
        "force_recovery": true
      },
      {
        "name": "personal",
        "allow_user_limited_channels": true,
        "history_size": 50,
        "history_ttl": "10m",
        "force_recovery": true
      }
    ]
  }
}
```

Секреты задаются через переменные окружения, а не в файле. Перед реализацией агент сверяет ключи с документацией установленной версии Centrifugo: структура конфига менялась между v5 и v6.

## 6. Игровой процесс

Комната — конечный автомат из пяти статусов, переходы выполняет только сервис `game`. Любой запрос, недопустимый в текущем статусе, получает 409.

> Диаграмма «статусы комнаты · 5 состояний» есть только в исходном документе.

### Лобби

- Хост создаёт комнату и сразу становится игроком. В `rooms.settings` сохраняется снимок настроек теста.
- Игроки попадают через принятое приглашение или ссылку с кодом. Вход возможен только в `lobby`.
- Старт доступен, только когда `is_ready = true` у всех активных игроков, включая хоста.
- Хост вышел из лобби → комната `cancelled`. Лобби без активности 24 часа → `cancelled` фоновой задачей.

### Цикл раунда

1. `start`: статус `countdown`, публикация `game.countdown` с `starts_at = now() + 3 с`; фиксируется порядок вопросов (с перемешиванием, если включено).
2. На `starts_at` воркер создаёт `room_rounds[0]` со снимком вопроса, `deadline_at = started_at + time_limit`, статус `in_progress`, событие `round.started`. Варианты перемешиваются одинаково для всех игроков.
3. Игрок отправляет один ответ. Сервер считает `response_time_ms`, проверяет, начисляет очки, сохраняет и публикует `answer.accepted` в личный канал и `round.progress` в комнату.
4. Раунд закрывается, когда ответили все активные игроки или наступил `deadline_at`. Не ответившим записывается «нет ответа» с 0 очков (отдельной строкой не хранится, выводится в итогах).
5. Публикуются `round.ended` (верный ответ и пояснение) и `round.my_result` каждому игроку. Пауза 5 с на разбор.
6. Следующий раунд — снова шаг 2. После последнего раунда считаются `final_score` и `final_rank`, статус `finished`, событие `game.finished`.

### Подсчёт очков

Верный ответ даёт от 50% до 100% очков вопроса в зависимости от скорости: P — `points`, t — время ответа, T — лимит раунда, r — доля верности.

$$
\text{score} = \operatorname{round}\left(P \cdot r \cdot \left(1 - \frac{t}{2T}\right)\right)
$$

| Тип | r | `is_correct` |
|---|---|---|
| `single` | 1, если выбран верный вариант, иначе 0 | r = 1 |
| `multiple` | max(0, (верно выбрано − неверно выбрано) / всего верных) — частичный зачёт | r = 1; при 0 < r < 1 статус «частично» |
| `text` | 1, если ответ прошёл проверку из раздела 4, иначе 0 | r = 1 |

Пример: P = 1000, T = 20 с, верный ответ за 4 с → 1000 × (1 − 4/40) = 900 очков. Рейтинг: сумма очков по убыванию, при равенстве — меньшее суммарное время верных ответов.

### Крайние случаи

- Игрок потерял соединение — остаётся в игре, после переподключения может ответить в текущем раунде. Для условия «ответили все» офлайн-игроки не учитываются (по presence Centrifugo).
- Игрок нажал «Выйти» во время игры — `left_at` заполняется, в итогах он остаётся с набранными очками.
- Хост отключился посреди игры — игра продолжается, таймерами управляет сервер.
- Все игроки вышли — `cancelled`.
- Ответ пришёл после дедлайна с учётом допуска 300 мс — `ROUND_CLOSED`.

### Данные для итогов (`GET /rooms/{id}/results`)

```json
{
  "room": {"id": "…", "quiz_title": "…", "started_at": "…", "finished_at": "…"},
  "leaderboard": [
    {"rank": 1, "user": {"id": "…", "display_name": "Аня"}, "score": 4120,
     "correct": 4, "partial": 1, "wrong": 0, "no_answer": 0, "avg_time_ms": 5320}
  ],
  "rounds": [
    {"index": 0, "question": {"text": "…", "type": "single", "options": […], "correct": […], "explanation": "…"},
     "answers": [
       {"user_id": "…", "status": "correct", "value": ["opt-id"], "time_ms": 4100, "score": 898}
     ],
     "stats": {"correct_ratio": 0.6, "avg_time_ms": 7200}}
  ],
  "series": {
    "cumulative_score": [{"user_id": "…", "points": [898, 1650, 2600, 3400, 4120]}]
  }
}
```

`status` ∈ `correct` | `partial` | `wrong` | `no_answer`. Итоги доступны только участникам комнаты и только в статусе `finished`.

## 7. Frontend (React)

SPA на React 19 + TypeScript, собирается Vite. Серверное состояние живёт в TanStack Query, realtime-события Centrifugo обновляют его кэш напрямую, локальное состояние игры — в небольшом Zustand-сторе.

### Библиотеки

| Назначение | Выбор |
|---|---|
| Сборка | Vite, TypeScript (strict) |
| Роутинг | React Router 7 |
| Серверное состояние | TanStack Query |
| Клиентское состояние | Zustand (сессия, игровой стор) |
| Формы и валидация | React Hook Form + Zod |
| UI | Tailwind CSS + shadcn/ui |
| Realtime | centrifuge-js |
| Графики | Recharts |
| Drag-and-drop порядка вопросов | dnd-kit |
| API-клиент | генерация типов из OpenAPI FastAPI (`openapi-typescript`) + fetch-обёртка |
| Тесты | Vitest, Testing Library, Playwright (e2e) |

### Страницы

| Маршрут | Экран | Содержимое |
|---|---|---|
| `/login`, `/register` | Вход / регистрация | формы с валидацией, ошибки с бэка в полях |
| `/` | Кабинет | мои тесты, входящие приглашения, история игр, кнопка «Создать тест» |
| `/quizzes/:id/edit` | Конструктор теста | название, настройки, список вопросов с перетаскиванием, «Добавить вопрос», «Из библиотеки», «Создать комнату» |
| `/questions` | Мои вопросы | таблица с фильтрами type / category / tag, поиск |
| `/questions/new`, `/questions/:id` | Редактор вопроса | тип, текст, варианты с переключателем верности или список допустимых ответов, категория (комбобокс с «Создать “…”»), тэги (chip-инпут с автодополнением), публичность |
| `/library` | Библиотека | публичные вопросы, фильтры категория + тэги + поиск, бесконечная прокрутка, «Добавить в тест» |
| `/rooms/:id` | Комната | один маршрут, экран выбирается по `status`: лобби / игра / итоги |
| `/join/:code` | Вход по ссылке | находит комнату по коду и делает join |
| `/history` | История | сыгранные комнаты со ссылками на итоги |

### Экран комнаты

**Лобби.** Список игроков с индикатором онлайн (presence) и галочкой готовности, кнопка «Готов / Не готов», код и ссылка-приглашение с копированием, поиск пользователей для приглашения (только хост), кнопка «Старт» активна у хоста, когда готовы все.

**Обратный отсчёт.** 3-2-1 по `starts_at`.

**Раунд.** Номер вопроса «4 / 10», полоса таймера, текст вопроса. Для `single` — радиокнопки, `multiple` — чекбоксы и кнопка «Ответить», `text` — инпут и Enter. После отправки ввод блокируется, показывается «Ответ принят за 4,2 с» и счётчик «Ответили 3 из 5». По `round.ended` подсвечиваются верные варианты и свой результат: верно / неверно / частично, +очки.

**Итоги.**

- Пьедестал и таблица рейтинга: место, игрок, очки, верных ответов, среднее время.
- Мои ответы: по каждому вопросу — мой ответ, верный ответ, время, очки, пояснение.
- Ответы других: матрица вопрос × игрок, в ячейке статус (верно / частично / неверно / нет ответа) цветом и время ответа.
- График «Ход игры»: линия накопленных очков каждого игрока по раундам.
- График «Время ответа»: сгруппированные столбцы по вопросам, столбец на игрока, цвет по статусу ответа.
- Сводка по вопросам: доля верных ответов и среднее время — видно самые сложные вопросы.

### Работа с realtime

- Хук `useCentrifuge()` — один клиент на приложение, токен через `getToken` → `/auth/centrifugo-token`.
- Хук `useRoomChannel(roomId)` — подписка на `room:<id>` с токеном подписки и обработчиком событий, который обновляет кэш `['room', id]` в TanStack Query через `setQueryData`.
- Хук `usePersonalChannel()` — `personal:#<userId>`: тосты о приглашениях, личные результаты.
- Хук `useServerClock()` — смещение часов и таймер раунда.
- При неудачном recovery — `invalidateQueries(['room', id])`.

### Структура `frontend/src`

```
api/          # сгенерированные типы, fetch-клиент, query-хуки по модулям
realtime/     # centrifuge-клиент, useRoomChannel, usePersonalChannel, типы событий
features/
  auth/  quizzes/  questions/  library/  rooms/  game/  results/
components/   # общие UI-компоненты
stores/       # zustand: session, game
routes/       # конфиг роутера, guard авторизации
lib/          # utils, форматирование времени
```

## 8. Нефункциональные требования и безопасность

### Безопасность

- Верные ответы никогда не уходят на клиент до `round.ended`: отдельные Pydantic-схемы `QuestionPublic` и `QuestionWithAnswers`, тест проверяет отсутствие `is_correct` в ответах `/rooms/{id}` и `round.started`.
- Публичная библиотека тоже скрывает верные ответы; они видны только после копирования вопроса к себе.
- Пароли — argon2id; минимум 8 символов. Access-токен в памяти фронта, refresh — httpOnly, Secure, SameSite=Lax cookie с ротацией.
- Авторизация на уровне сервиса: правка вопросов и тестов — только автор, управление комнатой — только хост, ответ — только участник комнаты в статусе `in_progress`.
- Rate limit: логин и регистрация 5/мин на IP, ответы 10/мин на пользователя, приглашения 30/мин.
- CORS и `allowed_origins` Centrifugo — только домен фронта. API-ключ Centrifugo — только на бэкенде.
- Санитизация: тексты вопросов рендерятся как plain text, без HTML.

### Корректность и конкурентность

- Приём ответа — одна транзакция: `SELECT … FOR UPDATE` раунда, проверка `now() < deadline_at` с допуском 300 мс на сеть, вставка с `UNIQUE (round_id, user_id)`.
- Закрытие раунда идемпотентно: `UPDATE room_rounds SET ended_at = now() WHERE id = … AND ended_at IS NULL RETURNING …`; только тот, кто получил строку, публикует `round.ended` и запускает следующий раунд.
- Таймеры раундов переживают рестарт бэкенда: воркер раз в 250 мс выбирает просроченные раунды `deadline_at < now() AND ended_at IS NULL` через `FOR UPDATE SKIP LOCKED` — работает и с несколькими инстансами.
- Публикация в Centrifugo — после коммита транзакции. Ошибка публикации логируется и не откатывает игру: клиенты дотянут состояние через REST.

### Производительность (MVP)

- До 50 игроков в комнате, до 200 одновременных комнат на одном инстансе.
- Задержка от дедлайна до `round.ended` у клиентов — до 500 мс.
- p95 REST-ответов — до 200 мс при этой нагрузке.

### Наблюдаемость

- Структурные JSON-логи (structlog) с `request_id`, `user_id`, `room_id`.
- `/healthz` (живость) и `/readyz` (БД и Centrifugo доступны).
- Опционально: Prometheus-метрики (`prometheus-fastapi-instrumentator`), встроенные метрики Centrifugo.

### Качество кода

- Покрытие бэкенда тестами ≥ 80% для модулей `game`, `questions`, `rooms`.
- Обязательные тесты: проверка каждого типа ответа, подсчёт очков, гонка двух одновременных ответов, закрытие раунда по таймеру и по «ответили все», утечка верных ответов.
- CI (GitHub Actions): ruff, mypy, pytest с Postgres 18 в сервисе; eslint, tsc, vitest; сборка Docker-образов.

## 9. Репозиторий, Docker, конфигурация

Монорепозиторий: бэкенд, фронтенд и инфраструктура рядом, весь стек поднимается одной командой `docker compose up`.

```
quiz/
├── AGENTS.md                 # правила для ИИ-агента (см. раздел 10)
├── docker-compose.yml
├── .env.example
├── infra/
│   ├── centrifugo/config.json
│   └── nginx/nginx.conf      # прод: /api → backend, /connection → centrifugo, / → статика
├── backend/
│   ├── pyproject.toml        # uv
│   ├── alembic/
│   ├── app/
│   │   ├── main.py           # create_app, роутеры, middleware
│   │   ├── core/             # config, db, security, errors, logging
│   │   ├── realtime/         # centrifugo client, tokens, event schemas
│   │   ├── modules/
│   │   │   ├── auth/  users/  catalog/  questions/  quizzes/
│   │   │   ├── rooms/  game/  results/
│   │   │   │   # в каждом: router.py, schemas.py, service.py, repository.py, models.py
│   │   └── workers/round_timer.py
│   ├── scripts/seed.py       # системные категории, демо-пользователи и тест
│   └── tests/
└── frontend/
    ├── package.json
    ├── vite.config.ts
    └── src/                  # см. раздел 7
```

### docker-compose.yml

```yaml
services:
  postgres:
    image: postgres:18
    environment:
      POSTGRES_DB: quiz
      POSTGRES_USER: quiz
      POSTGRES_PASSWORD: quiz
    volumes: [pgdata:/var/lib/postgresql]
    ports: ["5432:5432"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U quiz"]
      interval: 5s

  centrifugo:
    image: centrifugo/centrifugo:v6
    command: centrifugo -c /centrifugo/config.json
    volumes: [./infra/centrifugo:/centrifugo]
    environment:
      CENTRIFUGO_CLIENT_TOKEN_HMAC_SECRET_KEY: ${CENTRIFUGO_TOKEN_SECRET}
      CENTRIFUGO_HTTP_API_KEY: ${CENTRIFUGO_API_KEY}
    ports: ["8000:8000"]

  backend:
    build: ./backend
    command: sh -c "alembic upgrade head && uvicorn app.main:app --host 0.0.0.0 --port 8080 --reload"
    env_file: .env
    depends_on:
      postgres: { condition: service_healthy }
      centrifugo: { condition: service_started }
    ports: ["8080:8080"]

  frontend:
    build: ./frontend
    command: npm run dev -- --host
    environment:
      VITE_API_URL: http://localhost:8080/api/v1
      VITE_CENTRIFUGO_URL: ws://localhost:8000/connection/websocket
    ports: ["5173:5173"]

volumes:
  pgdata:
```

У образа PostgreSQL 18 изменился путь данных по умолчанию: том монтируется в `/var/lib/postgresql`, а не в `/var/lib/postgresql/data`, — агенту стоит это проверить по описанию образа.

### Переменные окружения (`.env.example`)

| Переменная | Пример | Назначение |
|---|---|---|
| `DATABASE_URL` | `postgresql+asyncpg://quiz:quiz@postgres:5432/quiz` | подключение бэкенда |
| `JWT_SECRET` | случайные 64 байта | подпись access-токенов |
| `ACCESS_TOKEN_TTL_MIN` | 15 | срок access-токена |
| `REFRESH_TOKEN_TTL_DAYS` | 30 | срок refresh-токена |
| `CENTRIFUGO_API_URL` | `http://centrifugo:8000/api` | server API |
| `CENTRIFUGO_API_KEY` | случайная строка | ключ server API |
| `CENTRIFUGO_TOKEN_SECRET` | случайные 64 байта | HMAC для токенов клиента, тот же у Centrifugo |
| `CORS_ORIGINS` | `http://localhost:5173` | разрешённые источники |
| `ROOM_MAX_PLAYERS` | 50 | лимит игроков |

## 10. План работ для ИИ-агента и приёмка

Агент работает по этапам: каждый этап — отдельная ветка и PR, этап считается закрытым только когда проходят его критерии и весь CI. Не переходить к следующему этапу с красными тестами.

### Правила для агента (содержимое `AGENTS.md`)

- Это ТЗ — источник истины. Отклонение от него — только с записью в `docs/decisions.md` и причиной.
- Сначала тесты к бизнес-логике (`checker`, `scoring`, сервисы `game`), потом код.
- Никаких верных ответов в публичных схемах; при любом сомнении — отдельная схема.
- Время и очки считает только сервер.
- Все изменения БД — только через миграции Alembic.
- Перед коммитом: `ruff check`, `mypy`, `pytest`, `npm run lint`, `npm run typecheck`, `npm test`.
- Версии библиотек и ключи конфигурации Centrifugo сверять с актуальной документацией, а не с памятью.

### Этапы

1. **Каркас.** Монорепо, docker-compose, пустые FastAPI и Vite-приложения, Alembic, CI, `/healthz`.
  - [ ] `docker compose up` поднимает 4 сервиса, фронт открывается, `/healthz` → 200.
2. **Auth.** Регистрация, логин, refresh, logout, guard маршрутов на фронте.
  - [ ] Пользователь регистрируется, входит, переживает перезагрузку страницы, выходит.
3. **Вопросы и справочники.** Модели, CRUD вопросов трёх типов, категории с дедупликацией, тэги, редактор вопроса.
  - [ ] Нельзя сохранить `single` с двумя верными или `text` без допустимых ответов.
  - [ ] Создание категории «  Базы  данных » при существующей «базы данных» возвращает существующую.
4. **Тесты (quizzes).** CRUD, порядок вопросов drag-and-drop, кабинет.
  - [ ] Порядок сохраняется и восстанавливается после перезагрузки.
5. **Библиотека.** Публикация, фильтры по категории и тэгам, поиск, копирование в свой тест.
  - [ ] В ответе `/library/questions` нет полей верности.
6. **Realtime-основа.** Centrifugo в compose, токены подключения и подписки, личный канал, хуки на фронте.
  - [ ] Тестовое событие с бэкенда доходит в личный канал залогиненного пользователя.
7. **Комнаты и лобби.** Создание, приглашения, вход по коду, готовность, presence, старт.
  - [ ] Два браузера видят вход и готовность друг друга без перезагрузки; «Старт» доступен только при всех готовых.
8. **Игровой движок.** Раунды, приём ответов, проверка, очки, таймер-воркер, переходы статусов.
  - [ ] Раунд закрывается и по дедлайну, и когда ответили все; второй ответ → `ALREADY_ANSWERED`.
  - [ ] Рестарт бэкенда посреди раунда не ломает игру.
9. **Итоги и статистика.** `/results`, таблицы, матрица ответов, два графика, история.
  - [ ] Итоги совпадают с данными в БД; графики строятся для 1 и для 50 игроков.
10. **Полировка.** Ошибки и пустые состояния, адаптив, rate limit, seed-данные, e2e Playwright.
  - [ ] E2E: два пользователя проходят игру из 3 вопросов разных типов от регистрации до итогов.

### Итоговые критерии приёмки MVP

- [ ] Все сценарии из раздела 1 проходят вручную на чистом `docker compose up`.
- [ ] Верные ответы не видны в DevTools (REST и WebSocket) до конца раунда.
- [ ] Таймер у игроков расходится не более чем на 0,5 с.
- [ ] Переподключение посреди раунда возвращает игрока в актуальное состояние.
- [ ] CI зелёный, покрытие ключевых модулей ≥ 80%.
