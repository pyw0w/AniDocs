# Конфигурация

Эта страница описывает основные параметры конфигурации runtime и core-модулей.

Файл конфигурации по умолчанию — `config.json` в репозитории `AniRuntime` (см. `config.example.json`).

## Общая структура config.json

Ниже приведён пример типичной структуры (схематично):

```json
{
  "log_level": "info",
  "cores": ["discord", "backend"],
  "core_directory": "./build/cores",
  "core_configs": {
    "discord": {
      "config": {
        "token": "DISCORD_BOT_TOKEN",
        "guild_id": "..."
      }
    }
  },
  "plugins": {
    "directory": "./build/plugins",
    "enabled": ["example", "commands"]
  },
  "backend": {
    "enabled": true,
    "database": {
      "path": "./data/anicore.db"
    }
  },
  "security": {
    "allowed_paths": ["./build/cores", "./build/plugins"],
    "verify_hashes": true
  }
}
```

Точные поля могут отличаться, ориентируйтесь на актуальную версию `config.example.json` и структуру `config.Config` в `AniCore/pkg/config`.

## Логирование

- `log_level` — уровень логирования runtime и core-модулей.
- Возможные значения определяются в `pkg/logger` (например, `debug`, `info`, `warn`, `error`).

Логи пишутся через `pkg/logger`, стандартный пакет `log` **не используется**.

## Core-модули

### Список core-модулей

- `cores` — список имён core-модулей, которые нужно загрузить.
- Для каждого имени runtime ищет файл `<coreName>.so` в директории `core_directory`.

### Директория core-модулей

- `core_directory` — основная директория для `.so` файлов core-модулей.
- Для обратной совместимости может использоваться `cores_directory` (старое имя поля) — runtime сначала берёт `core_directory`, если оно не задано — `cores_directory`.

### Конфигурация core-модулей

- `core_configs` — объект с конфигами по имени core-модуля.
- Для каждого core-модуля передаётся `map[string]interface{}` в `RuntimeAPIImpl`.
- Конкретный формат зависит от самого core-модуля (например, токен бота, идентификаторы каналов и т.п.).

## Плагины

### Директория плагинов

- `plugins.directory` — путь к директории с `.so` файлами плагинов.

### Включённые плагины

- `plugins.enabled` — список имён плагинов, которые нужно загрузить.
- Runtime сканирует директорию `plugins.directory`, берёт только `.so` файлы и загружает те, чьи имена попали в `enabled`.

Если плагин реализует интерфейс `CommandProvider` из `AniApi/shared`, его slash-команды будут автоматически собраны и переданы core-модулям, которые их регистрируют (например, Discord core).

## Backend и база данных

Backend реализован как core-модуль и использует SQLite базу данных.

- `backend.enabled` — включает/выключает backend и инициализацию БД.
- `backend.database.path` — путь к файлу базы данных.
  - Если путь не указан, по умолчанию используется `./data/anicore.db`.

Runtime инициализирует БД через `database.NewDB` и передаёт подключение в core-модули и плагины через API:

- Core-модули могут получить `*sql.DB` через `RuntimeAPI.GetDatabase()`.
- Плагины получают `shared.DatabaseAPI` через `CoreAPI.Database()` (сайндбокс по pluginID).

## Безопасность

Блок `security` управляет проверкой загружаемых `.so` файлов:

- `security.allowed_paths` — список разрешённых директорий, из которых можно загружать плагины и core-модули.
- `security.verify_hashes` — включить/выключить проверку хэшей.

Проверка выполняется через `pkg/security.Verifier` перед загрузкой каждого `.so`. При ошибке проверки файл не загружается, а в логи пишется предупреждение.

## Переменные окружения и секреты

- Никогда не храните токены и пароли в репозитории.
- Для локальной разработки можно использовать `config.json` (который в `.gitignore`).
- Для CI/CD рекомендуется использовать переменные окружения и секреты CI.

## Версионирование и совместимость

- Версия ядра хранится в файле `VERSION` в корне AniCore.
- Runtime читает этот файл (или `../VERSION`) и передаёт версию в загрузчик плагинов.
- Плагины реализуют `CoreVersionCompatibility()` и возвращают `CompatibilityRange` из `AniApi/shared/version.go`.
- При загрузке плагинов проверяется, совместима ли версия ядра с диапазоном, указанным плагином.

Подробнее про версионирование см. в `api.md` и `development.md`.
