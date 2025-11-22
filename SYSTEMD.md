# Интеграция с systemd

Этот модуль обеспечивает интеграцию AniDev с systemd для управления сервисом.

## Возможности

- Автоматическое определение работы под systemd
- Уведомления systemd о состоянии сервиса (READY, STOPPING, STATUS)
- Поддержка watchdog для мониторинга работоспособности
- Обработка сигналов SIGTERM для корректного завершения
- Логирование в systemd journal

## Установка

### 1. Сборка проекта

```bash
cargo build --release
```

### 2. Установка бинарного файла

```bash
sudo cp target/release/anicore /usr/local/bin/anicore
sudo chmod +x /usr/local/bin/anicore
```

### 3. Создание пользователя (опционально, для безопасности)

```bash
sudo useradd -r -s /bin/false anicore
```

### 4. Настройка переменных окружения

Создайте файл `/etc/anicore/env`:

```bash
sudo mkdir -p /etc/anicore
sudo nano /etc/anicore/env
```

Добавьте:
```
DISCORD_TOKEN=your_discord_token_here
```

Или установите переменную напрямую в service файле.

### 5. Установка service файла

Скопируйте `anicore.service` в `/etc/systemd/system/`:

```bash
sudo cp anicore.service /etc/systemd/system/
```

Отредактируйте файл, указав правильный путь к бинарному файлу и токен Discord:

```bash
sudo nano /etc/systemd/system/anicore.service
```

### 6. Перезагрузка systemd и запуск сервиса

```bash
sudo systemctl daemon-reload
sudo systemctl enable anicore
sudo systemctl start anicore
```

## Управление сервисом

### Проверка статуса

```bash
sudo systemctl status anicore
```

### Просмотр логов

```bash
sudo journalctl -u anicore -f
```

### Остановка сервиса

```bash
sudo systemctl stop anicore
```

### Перезапуск сервиса

```bash
sudo systemctl restart anicore
```

### Перезагрузка конфигурации (если поддерживается)

```bash
sudo systemctl reload anicore
```

## Настройка service файла

Основные параметры в `anicore.service`:

- `Type=notify` - сервис уведомляет systemd о готовности через sd_notify
- `WatchdogSec=60` - интервал watchdog (в секундах)
- `Restart=always` - автоматический перезапуск при сбое
- `RestartSec=10` - задержка перед перезапуском
- `User` и `Group` - пользователь и группа для запуска (для безопасности)

## Требования

- Linux система с systemd
- libsystemd установлен в системе
- Rust toolchain для сборки

## Примечания

- Модуль автоматически определяет, запущен ли процесс под systemd
- Если не под systemd, все функции работают без ошибок, но уведомления игнорируются
- Watchdog обновляется каждые 30 секунд
- Сервис корректно обрабатывает SIGTERM для graceful shutdown

