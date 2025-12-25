# Domain Controller - Контроллер доменов

Система разделения пользовательского и админского интерфейсов на основе доменов.

## Описание

Domain Controller позволяет обслуживать два разных frontend приложения (пользовательский и админский) на разных доменах через один backend API сервер.

## Архитектура

- **Domain Middleware** (`internal/anapi/middleware/domain.go`) - определяет тип домена из запроса и сохраняет его в контексте
- **Static Handler** (`internal/anapi/handlers/static.go`) - раздает соответствующий frontend в зависимости от типа домена
- **Domain Config** - конфигурация доменов и путей к frontend в `APIConfig`

## Конфигурация

Добавьте в конфигурацию API следующие параметры:

```json
{
  "api": {
    "enabled": true,
    "host": "0.0.0.0",
    "port": 8080,
    "domains": {
      "admin_domain": "admin.anidev.com",
      "user_domain": "app.anidev.com",
      "admin_frontend_path": "./frontend-admin/dist",
      "user_frontend_path": "./frontend-user/dist"
    }
  }
}
```

### Параметры конфигурации

- `admin_domain` - домен для админ панели (например: `admin.anidev.com`)
- `user_domain` - домен для пользовательского интерфейса (например: `app.anidev.com` или `anidev.com`)
- `admin_frontend_path` - путь к собранному админскому frontend (директория с `index.html` и статическими файлами)
- `user_frontend_path` - путь к собранному пользовательскому frontend

## Типы доменов

- `DomainTypeAdmin` - админский домен
- `DomainTypeUser` - пользовательский домен
- `DomainTypeUnknown` - неизвестный домен (по умолчанию используется пользовательский)

## Использование в коде

### Получение типа домена в handler

```go
import "github.com/pyw0w/anidev/internal/anapi/middleware"

func MyHandler(c *gin.Context) {
    domainType := middleware.GetDomainType(c)
    
    switch domainType {
    case middleware.DomainTypeAdmin:
        // Логика для админ панели
    case middleware.DomainTypeUser:
        // Логика для пользовательского интерфейса
    }
}
```

### Ограничение доступа по домену

```go
// Только для админского домена
adminGroup := router.Group("/admin-panel")
adminGroup.Use(middleware.RequireDomainType(middleware.DomainTypeAdmin))
{
    // маршруты админ панели
}
```

## Маршрутизация

1. **API маршруты** (`/api/v1/*`) - обрабатываются независимо от домена
2. **Статические файлы** - раздаются из соответствующего frontend в зависимости от домена
3. **SPA роутинг** - все не-API запросы отдают `index.html` соответствующего frontend

## Примеры использования

### Разработка

Для разработки можно использовать разные порты или поддомены:

```json
{
  "domains": {
    "admin_domain": "localhost:3001",
    "user_domain": "localhost:3002",
    "admin_frontend_path": "./frontend-admin/dist",
    "user_frontend_path": "./frontend-user/dist"
  }
}
```

### Продакшн

В продакшене используйте реальные домены:

```json
{
  "domains": {
    "admin_domain": "admin.anidev.com",
    "user_domain": "app.anidev.com",
    "admin_frontend_path": "/var/www/anidev/frontend-admin/dist",
    "user_frontend_path": "/var/www/anidev/frontend-user/dist"
  }
}
```

## Структура frontend

Каждый frontend должен быть собран в отдельную директорию со следующей структурой:

```
frontend-admin/dist/
├── index.html
├── assets/
│   ├── index-abc123.js
│   └── index-def456.css
└── ...

frontend-user/dist/
├── index.html
├── assets/
│   ├── index-xyz789.js
│   └── index-uvw012.css
└── ...
```

## Безопасность

- Domain Controller проверяет домен из заголовка `Host`
- Защита от path traversal при раздаче статических файлов
- API маршруты доступны с любого домена (если не ограничены middleware)

## Логирование

Domain Controller логирует определение домена на уровне DEBUG:

```
Domain detected: admin.anidev.com -> admin
Domain detected: app.anidev.com -> user
```
