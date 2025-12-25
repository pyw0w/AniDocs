# Настройка Nginx для Domain Controller

## Быстрая установка

### 1. Установка Nginx

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install nginx

# CentOS/RHEL
sudo yum install nginx
```

### 2. Копирование конфигурации

```bash
# Скопируйте пример конфигурации
sudo cp docs/nginx.example.conf /etc/nginx/sites-available/anidev

# Или создайте символическую ссылку
sudo ln -s /etc/nginx/sites-available/anidev /etc/nginx/sites-enabled/
```

### 3. Редактирование конфигурации

Отредактируйте файл `/etc/nginx/sites-available/anidev`:

```bash
sudo nano /etc/nginx/sites-available/anidev
```

Измените:
- `admin.anidev.com` на ваш админский домен
- `app.anidev.com` на ваш пользовательский домен
- `api.anidev.com` на ваш API домен
- `127.0.0.1:8080` на адрес вашего backend (если отличается)

### 4. Проверка конфигурации

```bash
# Проверка синтаксиса
sudo nginx -t

# Если всё ок, перезагрузите Nginx
sudo systemctl reload nginx
```

### 5. Настройка DNS

Настройте DNS записи для ваших доменов:

```
A    admin.anidev.com    ->  YOUR_SERVER_IP
A    app.anidev.com       ->  YOUR_SERVER_IP
A    api.anidev.com       ->  YOUR_SERVER_IP
```

## Настройка SSL (HTTPS)

### Использование Let's Encrypt (Certbot)

```bash
# Установка Certbot
sudo apt install certbot python3-certbot-nginx

# Получение сертификатов для всех доменов
sudo certbot --nginx -d admin.anidev.com -d app.anidev.com -d api.anidev.com

# Certbot автоматически обновит конфигурацию Nginx
```

После получения сертификатов:

1. Раскомментируйте HTTPS блоки в конфигурации
2. Раскомментируйте редирект с HTTP на HTTPS
3. Перезагрузите Nginx: `sudo systemctl reload nginx`

### Автоматическое обновление сертификатов

```bash
# Проверка автоматического обновления
sudo certbot renew --dry-run

# Certbot автоматически обновляет сертификаты через cron
```

## Оптимизация производительности

### 1. Кеширование статических файлов

Если хотите раздавать статику напрямую через Nginx (рекомендуется):

```nginx
location /assets/ {
    alias /path/to/frontend-admin/dist/assets/;
    expires 1y;
    add_header Cache-Control "public, immutable";
    access_log off;
}
```

### 2. Gzip сжатие

Добавьте в основной конфиг Nginx (`/etc/nginx/nginx.conf`):

```nginx
gzip on;
gzip_vary on;
gzip_min_length 1024;
gzip_types text/plain text/css text/xml text/javascript 
           application/x-javascript application/xml+rss 
           application/json application/javascript;
```

### 3. Увеличение лимитов

Если нужно обрабатывать больше запросов:

```nginx
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=50r/s;
limit_req_zone $binary_remote_addr zone=general_limit:10m rate=100r/s;
```

## Мониторинг и логи

### Просмотр логов

```bash
# Логи доступа
sudo tail -f /var/log/nginx/admin.anidev.com.access.log
sudo tail -f /var/log/nginx/app.anidev.com.access.log

# Логи ошибок
sudo tail -f /var/log/nginx/admin.anidev.com.error.log
sudo tail -f /var/log/nginx/app.anidev.com.error.log
```

### Статистика

```bash
# Установка модуля статистики (опционально)
sudo apt install nginx-module-njs

# Или используйте внешние инструменты:
# - GoAccess для анализа логов
# - Grafana + Prometheus для мониторинга
```

## Безопасность

### 1. Ограничение доступа к админской панели

Можно ограничить доступ по IP:

```nginx
location / {
    # Разрешаем доступ только с определенных IP
    allow 192.168.1.0/24;
    allow 10.0.0.0/8;
    deny all;
    
    proxy_pass http://anidev_backend;
    # ... остальные настройки
}
```

### 2. Защита от DDoS

Уже включено в конфигурацию через `limit_req_zone`. При необходимости увеличьте лимиты.

### 3. Скрытие версии Nginx

В `/etc/nginx/nginx.conf`:

```nginx
server_tokens off;
```

## Troubleshooting

### Проблема: 502 Bad Gateway

**Причина**: Backend не запущен или недоступен

**Решение**:
```bash
# Проверьте, что backend запущен
sudo systemctl status anidev

# Проверьте порт
sudo netstat -tlnp | grep 8080
```

### Проблема: Домен не определяется

**Причина**: Заголовок Host не передается правильно

**Решение**: Убедитесь, что в конфигурации есть:
```nginx
proxy_set_header Host $host;
```

### Проблема: Статические файлы не загружаются

**Причина**: Неправильный путь к frontend

**Решение**: Проверьте пути в `config.json`:
```json
{
  "domains": {
    "admin_frontend_path": "/var/www/anidev/frontend-admin/dist",
    "user_frontend_path": "/var/www/anidev/frontend-user/dist"
  }
}
```

### Проблема: CORS ошибки

**Причина**: Backend должен обрабатывать CORS правильно

**Решение**: Убедитесь, что в backend включен CORS middleware (уже включен по умолчанию)

## Пример полной конфигурации для продакшена

См. `docs/nginx.example.conf` - там есть полный пример с HTTPS, безопасностью и оптимизацией.
