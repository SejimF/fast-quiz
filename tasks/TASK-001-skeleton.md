# TASK-001: Каркас

## Зачем

Подготовить окружение и проверки. Без них не работает TDD в следующих этапах.

## Контекст

`docs/SPEC.md`: раздел 2 (стек), раздел 9 (структура репозитория, `docker-compose.yml`, `.env.example`), раздел 8 («Наблюдаемость», «Качество кода»). `docs/decisions.md`: записи 2–4.

## Что сделать

- Структура монорепозитория из раздела 9: `backend/`, `frontend/`, `infra/`, `docker-compose.yml`, `.env.example`.
- `backend`: Dockerfile на Python 3.13 с uv, `pyproject.toml` с настройками ruff, mypy (strict), pytest (`asyncio_mode = "auto"`) и pytest-cov, `create_app`, `core/config.py` (pydantic-settings), `core/db.py`, настроенный асинхронный Alembic с пустой первой миграцией, эндпоинты `/healthz` и `/readyz`, логи structlog в формате JSON.
- Фикстуры тестов: база `quiz_test` создаётся при старте тестовой сессии, миграции накатываются, каждый тест работает в откатываемой транзакции. Клиент `httpx.AsyncClient`.
- `frontend`: Vite, React 19, TypeScript (strict), ESLint, Vitest и Testing Library, Tailwind, скрипты `lint`, `typecheck`, `test`. Стартовая страница с запросом к `/healthz`.
- `infra/centrifugo/config.json` по разделу 5. Секреты задаются через переменные окружения.
- Сервисы `postgres`, `redis`, `centrifugo`, `backend` и `frontend` в compose (`docs/cache.md`, раздел «Реализация»).
- `app/core/cache.py` и `app/core/cache_keys.py`: клиент, `get_or_load`, `invalidate`, `bump`, хук после коммита. Тестовая база Redis `/15`. Каталоги кода монтируются в контейнеры, контейнеры работают от UID и GID хоста.
- `.github/workflows/ci.yml`: те же проверки через `docker-compose`.
- Удалить `main.py` и `test_main.http`.

## Критерии приёмки

- [ ] **AC1.** Даны чистые тома и `cp .env.example .env`, когда выполняется `docker-compose up --build`, тогда поднимаются 5 сервисов, фронт открывается на `:5173`, а `GET :8080/healthz` возвращает 200.
- [ ] **AC2.** `GET /readyz` возвращает 200, когда доступны PostgreSQL и Centrifugo, и 503 с указанием недоступной зависимости, когда одна из них остановлена. Недоступный Redis даёт 200 со статусом `degraded`.
- [ ] **AC8.** `get_or_load` при промахе вызывает загрузчик один раз даже для 20 одновременных запросов, при попадании не вызывает его вовсе, а при падающем Redis возвращает результат загрузчика.
- [ ] **AC9.** `invalidate` после отката транзакции не выполняется.
- [ ] **AC3.** Все команды из раздела «Окружение» в `CLAUDE.md` проходят с кодом 0.
- [ ] **AC4.** Два теста, которые пишут в базу, не видят данных друг друга.
- [ ] **AC5.** Переменная, заданная на хосте, но не в `.env`, не попадает в контейнер `backend`.
- [ ] **AC6.** Файлы, которые создают контейнеры (миграции, `node_modules`, результат форматирования), принадлежат пользователю хоста.
- [ ] **AC7.** CI запускается на push и pull request и падает, если падает любая проверка.

## Тесты в порядке написания (TDD)

1. `integration`: `GET /healthz` возвращает 200 и `{"status": "ok"}`.
2. `integration`: `GET /readyz` возвращает 200 при доступных зависимостях и 503, когда проверка базы или Centrifugo подменена недоступной.
3. `integration`: изоляция данных между двумя тестами (AC4).
4. `unit`: конфиг падает с понятной ошибкой, если нет обязательной переменной.
5. `integration`: `get_or_load`: промах, попадание, защита от наплыва, падающий Redis. Затем инвалидация после коммита и её отсутствие при откате.
6. `vitest`: стартовая страница показывает статус бэкенда (запрос подменён).

## Граничные случаи

- PostgreSQL 18 хранит данные в `/var/lib/postgresql`, а не в `/var/lib/postgresql/data`. Это нужно проверить по описанию образа.
- `backend` ждёт готовности `postgres` по healthcheck.

## Вне области этапа

- Модели предметной области, авторизация, Nginx для прода.

## Как проверить вручную

```bash
cp .env.example .env
docker-compose up --build
curl -i localhost:8080/healthz
curl -i localhost:8080/readyz
docker-compose stop centrifugo && curl -i localhost:8080/readyz   # 503
docker-compose run --rm backend pytest
docker-compose run --rm frontend npm test
```
