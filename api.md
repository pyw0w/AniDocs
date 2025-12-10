# Публичный API (AniApi/shared)

Этот документ описывает ключевые интерфейсы и типы из пакета `github.com/pyw0w/AniApi/shared`.

Пакет `shared` — это единственный публичный контракт между:

- плагинами и ядром,
- core-модулями и runtime,
- публичными расширениями и приватным кодом AniCore.

## Обзор

Основные элементы API:

- `Plugin` — интерфейс плагина.
- `CoreAPI` — API для плагинов.
- `CommandProvider` — интерфейс для плагинов, предоставляющих slash-команды Discord.
- `Event` и типы событий — модель событий.
- `VersionInfo`, `CompatibilityRange`, `APIVersion` — версионирование и проверка совместимости.
- `DatabaseAPI` — интерфейс работы с БД для плагинов.
- `Core`, `RuntimeAPI` — интерфейсы для core-модулей (описаны в `types.go`).

## Plugin

```go
// Plugin интерфейс, который должен реализовать каждый плагин
type Plugin interface {
    // Name возвращает имя плагина
    Name() string

    // Version возвращает версию плагина
    Version() string

    // CoreVersionCompatibility возвращает информацию о совместимости с версиями ядра
    // Возвращает минимальную версию ядра и максимальную (пустая строка = без ограничения)
    // Формат версий: "MAJOR.MINOR.PATCH" (например, "1.0.0")
    CoreVersionCompatibility() CompatibilityRange

    // Init инициализирует плагин с предоставленным CoreAPI
    Init(core CoreAPI) error

    // OnEvent вызывается при получении события от runtime
    OnEvent(e Event) error
}
```

См. подробности и рекомендации в `plugins.md`.

## CoreAPI

```go
// CoreAPI предоставляет API для плагинов для взаимодействия с runtime и core-модулями
type CoreAPI interface {
    // Log логирует сообщение через runtime
    Log(format string, args ...interface{})

    // Emit отправляет событие в event bus
    Emit(e Event) error

    // CallCore вызывает команду в указанном core-модуле
    CallCore(coreName string, cmd Event) error

    // Database возвращает API для работы с базой данных
    Database() DatabaseAPI
}
```

Реализация `CoreAPIImpl` находится в `AniRuntime/cmd/runtime/main.go`.

## CommandProvider

```go
import "github.com/bwmarrin/discordgo"

// CommandProvider интерфейс для плагинов, которые предоставляют slash commands
// Это опциональный интерфейс - плагины могут его реализовать
type CommandProvider interface {
    // GetCommands возвращает список команд, которые плагин хочет зарегистрировать
    GetCommands() []*discordgo.ApplicationCommand
}
```

Runtime собирает команды от всех плагинов, реализующих `CommandProvider`, через метод `GetPluginCommands()` и передаёт их core-модулю (например, Discord core) для регистрации.

## Версионирование и совместимость

```go
// APIVersion версия публичного API
const APIVersion = "1.0.0"

// VersionInfo содержит информацию о версии
type VersionInfo struct {
    Major int
    Minor int
    Patch int
}

// CompatibilityRange описывает диапазон совместимости версий
type CompatibilityRange struct {
    MinVersion string // Минимальная версия ядра (включительно)
    MaxVersion string // Максимальная версия ядра (включительно, пустая строка = без ограничения)
}
```

Функции:

- `ParseVersion` — парсит строку `MAJOR.MINOR.PATCH` в структуру `VersionInfo`.
- `(*VersionInfo).Compare(other)` — сравнивает версии.
- `(*VersionInfo).GreaterOrEqual(other)` / `LessOrEqual(other)` — удобные проверки.
- `(*CompatibilityRange).IsCompatible(coreVersion)` — проверяет, совместима ли версия ядра с диапазоном.

Runtime использует `CompatibilityRange.IsCompatible` при загрузке плагинов, чтобы не загружать несовместимые версии.

Плагины должны:

- возвращать разумный `MinVersion` и при необходимости `MaxVersion`,
- обновлять диапазон при изменениях API ядра.

## События

Тип `Event` и константы типов событий определены в `event.go`/`types.go`. Общая идея:

- Каждое событие имеет тип (строка или enum),
- полезную нагрузку (данные события),
- источник (`source` — кто сгенерировал событие).

Поток событий:

1. Core-модули создают и публикуют `Event` через `RuntimeAPI.Emit`.
2. Runtime отправляет события в `eventbus`.
3. `eventbus` рассылает события всем подписчикам — плагинам (`OnEvent`).
4. Плагины могут генерировать новые события через `CoreAPI.Emit`.

Конкретные типы событий (например, `EventTypeMessageCreate`, `EventTypeCommand`, `EventTypePluginLoaded`) описаны в `AniApi/shared`.

## DatabaseAPI

Интерфейс `DatabaseAPI` определён в `database.go` и предоставляет плагинам доступ к БД:

- выполнение запросов к SQLite с учётом `pluginID`,
- изолированное хранение данных каждого плагина.

Реализация находится в `AniCore/pkg/database` и используется runtime через `Runtime.GetDatabaseAPI(pluginID)`.

## Core и RuntimeAPI

Для core-модулей в `types.go` определены:

- `Core` — интерфейс core-модуля (методы `Name`, `Init`, `Start`, `Stop`, `OnCommand` и т.п.),
- `RuntimeAPI` — интерфейс, который runtime предоставляет core-модулям.

Реализация `RuntimeAPIImpl` находится в `AniRuntime/cmd/runtime/main.go` и предоставляет:

- логирование с префиксом core-модуля,
- доступ к конфигурации core-модуля,
- доступ к БД,
- получение команд плагинов,
- доступ к версии ядра.

Подробнее об использовании этих интерфейсов см. в `cores.md` и `runtime.md`.
