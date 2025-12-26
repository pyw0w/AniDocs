# Настройка Domain Controller

## Быстрый старт

### 1. Структура frontend приложений

В проекте используется один frontend, который собирается в две разные папки для админской и пользовательской панелей:

```
project/
├── frontend/          # Исходный код frontend
│   ├── src/
│   ├── dist/          # Временная папка сборки (не используется для production)
│   └── package.json
├── frontend-admin/    # Собранные файлы админской панели
│   └── dist/          # Генерируется командой make build-frontend-admin
├── frontend-user/     # Собранные файлы пользовательского интерфейса
│   └── dist/          # Генерируется командой make build-frontend-user
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
        "user_domain": "anidev.com",
        "api_domain": "api.anidev.com",
        "admin_frontend_path": "./frontend-admin/dist",
        "user_frontend_path": "./frontend-user/dist"
      }
    }
  }
}
```

### 3. Сборка frontend

Используйте команды Makefile для сборки frontend:

```bash
# Собрать оба frontend (admin и user)
make build-frontend

# Или собрать отдельно:
make build-frontend-admin  # Собирает в frontend-admin/dist
make build-frontend-user   # Собирает в frontend-user/dist
```

Пути сборки можно переопределить через переменные окружения:

```bash
FRONTEND_ADMIN_PATH=./custom-admin/dist make build-frontend-admin
FRONTEND_USER_PATH=./custom-user/dist make build-frontend-user
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

Подробная инструкция по настройке Nginx находится в [`docs/nginx-setup.md`](nginx-setup.md).

Пример конфигурации: [`docs/nginx.example.conf`](nginx.example.conf)

Быстрый старт:

```bash
# Скопируйте пример конфигурации
sudo cp docs/nginx.example.conf /etc/nginx/sites-available/anidev

# Отредактируйте домены и пути
sudo nano /etc/nginx/sites-available/anidev

# Проверьте конфигурацию
sudo nginx -t

# Включите сайт и перезагрузите Nginx
sudo ln -s /etc/nginx/sites-available/anidev /etc/nginx/sites-enabled/
sudo systemctl reload nginx
```

**Важно**: Убедитесь, что в конфигурации Nginx передается заголовок `Host`:
```nginx
proxy_set_header Host $host;
```

Это необходимо для правильной работы Domain Controller.

### 5. Запуск

```bash
# Запустите backend
./anidev -config config.json

# Backend будет обслуживать три домена:
# - admin.anidev.com -> frontend-admin/dist + API
# - anidev.com -> frontend-user/dist + API
# - api.anidev.com -> только API (без frontend)
```

## Как это работает

1. **Domain Middleware** определяет домен из заголовка `Host`
2. **Static Handler** раздает соответствующий frontend (только на admin/user доменах)
3. **API маршруты** (`/api/v1/*`) доступны с любого домена
4. **API домен** (`api.domain.ru`) - только API, frontend не отдается
5. **SPA роутинг** - все не-API запросы на admin/user доменах отдают `index.html`

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
