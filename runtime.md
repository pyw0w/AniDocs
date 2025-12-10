# AniRuntime

`AniRuntime` — публичный запускатель платформы AniCore. Он отвечает за:

- чтение конфигурации,
- инициализацию логирования и БД,
- загрузку core-модулей и плагинов из `.so` файлов,
- маршрутизацию событий между core-модулями и плагинами,
- корректное завершение работы по сигналам.

## Структура репозитория AniRuntime

```text
AniRuntime/
├── cmd/runtime/
│   └── main.go      # Точка входа runtime
├── config.example.json
├── go.mod
└── README.md
```

## Основной поток выполнения

Упрощённо `cmd/runtime/main.go` делает следующее:

1. Читает путь к конфигу из флага `-config` (по умолчанию `config.json`).
2. Загружает конфигурацию через `config.LoadConfig` из `AniCore/pkg/config`.
3. Создаёт `Runtime` через `NewRuntime(cfg)`.
4. Настраивает обработку сигналов (`os.Interrupt`, `SIGTERM`).
5. Вызывает `runtime.Run(ctx)` и ждёт завершения.

```text
Runtime.Run(ctx):
- LoadCores()   — загрузка и инициализация core-модулей
- StartCores()  — запуск core-модулей
- LoadPlugins() — загрузка и инициализация плагинов
- ожидание ctx.Done()
- StopCores()   — остановка core-модулей
```

## Структура Runtime

Основная структура `Runtime` содержит:

- `config *config.Config` — конфигурация.
- `eventBus *eventbus.EventBus` — шина событий.
- `pluginLoader *pluginloader.Loader` — загрузчик плагинов.
- `coreLoader *coreloader.Loader` — загрузчик core-модулей.
- `verifier *security.Verifier` — проверка безопасности `.so` файлов.
- `plugins map[string]shared.Plugin` — загруженные плагины.
- `cores map[string]shared.Core` — загруженные core-модули.
- `coreAPIs map[string]*CoreAPIImpl` — API для плагинов.
- `log *logger.Logger` — логгер runtime.
- `coreVersion string` — версия ядра (из файла `VERSION`).
- `database *database.DB` — обёртка над БД (если включён backend).

### Загрузка core-модулей

`LoadCores()`:

- Берёт список имён core-модулей из `config.Cores`.
- Определяет директорию для `.so` файлов: `config.CoreDirectory` или `config.CoresDirectory`.
- Для каждого core-модуля:
  - строит путь `<coreDir>/<coreName>.so`,
  - загружает модуль через `coreloader.Loader`,
  - формирует конфиг для модуля из `config.CoreConfigs[coreName]`,
  - создаёт `RuntimeAPIImpl` и вызывает `core.Init(runtimeAPI)`,
  - сохраняет core в `r.cores`.

`StartCores()` и `StopCores()` вызывают `Start()`/`Stop()` у каждого core-модуля и логируют результат.

### Загрузка плагинов

`LoadPlugins()`:

- Берёт директорию плагинов из `config.Plugins.Directory`.
- Строит множество включённых плагинов по `config.Plugins.Enabled`.
- Сканирует директорию, отбирает только `.so` файлы.
- Для каждого плагина:
  - проверяет, включён ли он в `enabled`,
  - проверяет файл через `security.Verifier`,
  - загружает через `pluginloader.Loader.LoadPlugin(pluginPath, coreVersion)` — там проверяется совместимость версий,
  - создаёт `CoreAPIImpl` и вызывает `plugin.Init(coreAPI)`,
  - подписывает плагин на события через `eventBus.Subscribe` (например, `EventTypeMessageCreate`, `EventTypeCommand`),
  - отправляет событие `EventTypePluginLoaded`.

### Команды плагинов

`GetPluginCommands()` собирает slash-команды от всех плагинов, которые реализуют интерфейс `shared.CommandProvider`:

- Для каждого плагина проверяется type-assertion к `CommandProvider`.
- Если плагин его реализует, вызывается `GetCommands()`.
- Все команды собираются в один список и далее могут быть зарегистрированы core-модулем (например, Discord core).

### Работа с базой данных

Runtime инициализирует БД, если в конфиге включён backend:

- Путь к БД берётся из `cfg.Backend.Database.Path` или по умолчанию `./data/anicore.db`.
- Создаётся `database.DB` через `database.NewDB(dbPath)`.
- Плагины получают `shared.DatabaseAPI` через `Runtime.GetDatabaseAPI(pluginID)`.
- Core-модули получают `*sql.DB` через `RuntimeAPI.GetDatabase()`.

## Завершение работы

Runtime использует `context.Context` и канал сигналов:

- При получении `os.Interrupt` или `SIGTERM` вызывается `cancel()`.
- `Runtime.Run` выходит из ожидания `ctx.Done()`, останавливает core-модули и завершает работу.
- После этого main даёт время на graceful shutdown (`time.Sleep`).

## Запуск

Примеры запуска runtime:

```bash
# Из репозитория AniRuntime
cd /home/USER/anidev/AniRuntime

# Используя go run
go run ./cmd/runtime -config config.json

# Либо через заранее собранный бинарник (если есть)
./bin/anicore-runtime -config config.json
```

Подробнее про настройку конфигурации — в `configuration.md`, про API для плагинов и core-модулей — в `api.md`, `plugins.md` и `cores.md`.
