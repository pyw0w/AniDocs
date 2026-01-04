# AniDev API

Этот документ описывает основные REST эндпоинты AniDev API. База URL по умолчанию:

- Локально: `http://localhost:8080`
- Продакшн: домен API настраивается через `config.json` (`core_configs.api` и `domains.api_domain`).

Все пути ниже предполагают префикс `/api/v1` (или алиас `/v1`).

## Аутентификация

Большинство управленческих эндпоинтов требуют JWT-токен, полученный через OAuth (GitHub / Discord).

- OAuth-роуты доступны по префиксу `/api/v1/auth` и дублируются без префикса `/auth`.
- Токен передаётся в заголовке:
  - `Authorization: Bearer <jwt>`

Публичные эндпоинты (health, часть anime/news и т.п.) не требуют токена.

---

## Health

### `GET /health`

Проверка состояния сервиса.

**Ответ 200**

```json
{
  "status": "ok",
  "version": "1.0.0",
  "uptime": "00:10:42"
}
```

---

## Plugins

Работа с загруженными плагинами и .so файлами.

### `GET /plugins`

Список всех плагинов: загруженных в Manager и присутствующих в директории плагинов.

**Ответ 200**

```json
{
  "plugins": [
    {
      "id": "myplugin",
      "name": "My Plugin",
      "version": "1.0.0",
      "author": "Your Name",
      "description": "Описание плагина",
      "enabled": true,
      "created_at": "2024-01-01T12:00:00Z",
      "updated_at": "2024-01-01T12:00:00Z"
    }
  ],
  "total": 1
}
```

### `GET /plugins/{id}`

Информация о конкретном плагине (или загруженном, или только найденном в .so файлах).

Коды ответов:
- `200` — плагин найден
- `404` — плагин не найден

### `POST /plugins`

Загрузка плагина из .so файла.

**Тело запроса**

```json
{
  "name": "myplugin"
}
```

`name` — имя файла плагина в директории плагинов (расширение `.so` можно не указывать).

Коды ответов:
- `201` — плагин успешно загружен
- `400` — ошибка валидации или конфигурации
- `404` — файл плагина не найден

### `PUT /plugins/{id}`

Перезагрузка уже загруженного плагина.

Коды ответов:
- `200` — плагин перезагружен
- `404` — плагин не найден или не загружен

### `DELETE /plugins/{id}`

Выгрузка плагина из Manager.

Коды ответов:
- `204` — успешно выгружен
- `404` — плагин не найден или не загружен

---

## Webhooks

Позволяют подписываться на события (например, публикация новостей, статистика плагинов).

### `GET /webhooks`

Список зарегистрированных вебхуков.

### `GET /webhooks/{id}`

Информация о конкретном вебхуке.

### `POST /webhooks`

Создание нового вебхука.

**Тело запроса**

```json
{
  "url": "https://example.com/webhook",
  "secret": "optional-secret",
  "events": ["news.published", "stat.created"],
  "enabled": true
}
```

### `DELETE /webhooks/{id}`

Удаление вебхука.

Коды ответов:
- `204` — успешно удалён
- `404` — не найден

---

## Stats

Работа со статистикой использования плагинов.

### `GET /stats`

Агрегированная статистика.

Пример ответа:

```json
{
  "enabled_plugins": 3,
  "total_plugins": 5,
  "total_events": 42,
  "plugin_stats": [
    {
      "plugin_id": "myplugin",
      "plugin_name": "My Plugin",
      "event_count": 10,
      "last_event": "2024-01-01T12:00:00Z"
    }
  ],
  "recent_events": [
    {
      "id": 1,
      "plugin_id": "myplugin",
      "event_type": "command.used",
      "event_data": "...",
      "created_at": "2024-01-01T12:00:00Z"
    }
  ]
}
```

### `POST /stats`

Создание события статистики.

**Тело запроса**

```json
{
  "plugin_id": "myplugin",
  "event_type": "command.used",
  "event_data": "optional payload"
}
```

---

## Anime

Эндпоинты для работы с аниме (Kodik + локальная БД).

### `GET /anime`

Список аниме с фильтрацией и пагинацией.

Параметры:
- `q` — поисковый запрос (опционально)
- `genre` — жанр (опционально)
- `sort` — поле сортировки (`popularity`, `rating`, `release_date`, `title`)
- `order` — порядок (`asc` / `desc`)
- `page` — страница (по умолчанию 1)
- `limit` — размер страницы (по умолчанию 20)

### `GET /anime/{id}`

Детальная информация об аниме. `id` может быть:
- локальным ID,
- `shikimori_id`,
- `kodik_id` (в зависимости от того, как сохранено в БД).

### `GET /anime/search`

Поиск аниме по строке `q` (через Kodik, с fallback в локальную БД).

### `GET /anime/{id}/video`

Получение ссылки на плеер Kodik для выбранного аниме.

---

## Auth и пользовательские сущности

Здесь приведён только обзор; детали авторизации и форматов см. в отдельных документах AniDocs.

### `/auth` и `/api/v1/auth`

- `GET /auth/github`, `GET /auth/github/callback`
- `GET /auth/discord`, `GET /auth/discord/callback`
- `GET /auth/token` — получить JWT из cookie
- `GET /auth/me` — информация о текущем пользователе (требует токен)
- `POST /auth/logout` — выход

### Прочие группы

- `/admin/users` — управление пользователями панели
- `/news` и `/admin/news` — новости и changelog
- `/shikimori` — связка с Shikimori и импорт списков
- `/user/lists` — пользовательские списки аниме
- `/config` — чтение и обновление `config.json`
- `/guilds` — управление гильдиями Discord
- `/tickets` — тикеты поддержки

Подробные описания этих эндпоинтов рекомендуется вынести в отдельные Markdown-файлы в AniDocs (например, `auth.md`, `anime.md`, `webhooks.md` и т.д.).
