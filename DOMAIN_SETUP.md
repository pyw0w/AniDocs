# Настройка Domain Controller

## Быстрый старт

### 1. Структура frontend приложений

Создайте два отдельных frontend приложения:

```
project/
├── frontend-admin/     # Админская панель
│   ├── src/
│   ├── dist/          # Собранные файлы
│   └── package.json
├── frontend-user/     # Пользовательский интерфейс
│   ├── src/
│   ├── dist/          # Собранные файлы
│   └── package.json
└── cmd/anidev/        # Backend
```

### 2. Конфигурация

Добавьте в `config.json` секцию `domains`:

```json
{
  "core_configs": {
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
}
```

### 3. Сборка frontend

```bash
# Админская панель
cd frontend-admin
npm install
npm run build

# Пользовательский интерфейс
cd frontend-user
npm install
npm run build
```

### 4. Настройка доменов

#### Для разработки (localhost)

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

#### Для продакшена (nginx)

Настройте nginx для проксирования на backend:

```nginx
# Админский домен
server {
    listen 80;
    server_name admin.anidev.com;
    
    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

# Пользовательский домен
server {
    listen 80;
    server_name app.anidev.com;
    
    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### 5. Запуск

```bash
# Запустите backend
./anidev -config config.json

# Backend будет обслуживать оба домена
# - admin.anidev.com -> frontend-admin/dist
# - app.anidev.com -> frontend-user/dist
```

## Как это работает

1. **Domain Middleware** определяет домен из заголовка `Host`
2. **Static Handler** раздает соответствующий frontend
3. **API маршруты** (`/api/v1/*`) доступны с любого домена
4. **SPA роутинг** - все не-API запросы отдают `index.html`

## Примеры использования

### Получение типа домена в handler

```go
import "github.com/pyw0w/anidev/internal/anapi/middleware"

func MyHandler(c *gin.Context) {
    domainType := middleware.GetDomainType(c)
    
    if domainType == middleware.DomainTypeAdmin {
        // Логика для админ панели
    }
}
```

### Ограничение доступа по домену

```go
// Только для админского домена
adminGroup := router.Group("/admin-panel")
adminGroup.Use(middleware.RequireDomainType(middleware.DomainTypeAdmin))
```

## Важные замечания

- Пути к frontend должны быть абсолютными или относительными от рабочей директории
- Убедитесь, что директории `dist` существуют и содержат `index.html`
- API маршруты работают независимо от домена
- Для SPA роутинга все не-API запросы отдают `index.html`
