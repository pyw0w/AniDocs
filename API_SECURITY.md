# Защита API

## Обзор

API защищено двумя уровнями безопасности:
1. **CORS и Origin проверка** - только разрешенные домены могут обращаться к API
2. **JWT токены** - все защищенные endpoints требуют валидный токен авторизации

## Конфигурация

### Настройка разрешенных доменов

В `config.json` добавьте секцию `allowed_origins`:

```json
{
  "core_configs": {
    "api": {
      "domains": {
        "admin_domain": "admin.pycore.ru",
        "user_domain": "pycore.ru",
        "api_domain": "api.pycore.ru",
        "allowed_origins": [
          "https://admin.pycore.ru",
          "https://pycore.ru",
          "https://api.pycore.ru"
        ],
        "strict_origin_check": true
      }
    }
  }
}
```

### Параметры конфигурации

- `allowed_origins` - список разрешенных доменов для CORS и Origin проверки
  - Если не указан, автоматически используются домены из `admin_domain`, `user_domain`, `api_domain`
  - Поддерживаются протоколы `http://` и `https://`
- `strict_origin_check` - строгая проверка Origin заголовка
  - `true` - блокировать запросы без Origin заголовка
  - `false` - разрешать запросы без Origin (проверяется Host заголовок)

## Публичные endpoints

Следующие endpoints доступны без токена:

- `GET /health` - проверка здоровья сервиса
- `GET /auth/github` - OAuth авторизация через GitHub
- `GET /auth/github/callback` - OAuth callback GitHub
- `GET /auth/discord` - OAuth авторизация через Discord
- `GET /auth/discord/callback` - OAuth callback Discord
- `GET /news` - получение новостей
- `GET /news/:id` - получение новости по ID
- `GET /news/rss` - RSS лента новостей

## Защищенные endpoints

Все остальные endpoints требуют JWT токен в заголовке `Authorization`:

```
Authorization: Bearer <token>
```

### Примеры защищенных endpoints:

- `GET /auth/me` - информация о текущем пользователе
- `POST /auth/logout` - выход из системы
- `GET /plugins` - список плагинов
- `GET /webhooks` - список webhooks
- `GET /guilds` - список гильдий
- `GET /tickets` - список тикетов
- И другие...

## Как это работает

### 1. CORS защита

- Проверяется заголовок `Origin` запроса
- Если Origin не в списке разрешенных, запрос блокируется
- CORS заголовки устанавливаются только для разрешенных доменов

### 2. Origin проверка

- Дополнительная проверка Origin/Host/Referer заголовков
- Защита от CSRF атак
- Применяется только к API endpoints (`/api/v1/*` и `/v1/*`)

### 3. JWT токены

- Все защищенные endpoints требуют валидный JWT токен
- Токен получается через OAuth авторизацию
- Токен проверяется на каждом запросе

## Разработка

Для разработки можно отключить строгую проверку:

```json
{
  "domains": {
    "strict_origin_check": false
  }
}
```

Или не указывать `allowed_origins` - тогда будут разрешены все домены (только для разработки!).

## Безопасность

⚠️ **Важно для продакшн:**

1. Всегда указывайте `allowed_origins` в продакшн
2. Используйте `strict_origin_check: true` в продакшн
3. Используйте HTTPS для всех доменов
4. Храните JWT секрет в безопасном месте
