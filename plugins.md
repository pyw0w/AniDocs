# Разработка плагинов

Плагины — это расширения платформы AniCore, которые реализуют бизнес-логику бота. Плагины:

- собираются как `.so` файлы,
- загружаются runtime динамически,
- используют только публичный API из `github.com/pyw0w/AniApi/shared`.

## Базовый интерфейс Plugin

Каждый плагин обязан реализовать интерфейс `shared.Plugin`:

```go
package shared

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

Ключевые моменты:

- `Name()` — уникальное имя плагина, используется в логах и для идентификации в БД.
- `Version()` — версия плагина в формате `MAJOR.MINOR.PATCH`.
- `CoreVersionCompatibility()` — диапазон версий ядра, с которыми плагин совместим.
- `Init(core CoreAPI)` — инициализация плагина, сохраняйте `CoreAPI` для дальнейшей работы.
- `OnEvent(e Event)` — основной обработчик событий.

## CoreAPI

В `Init` плагину передаётся интерфейс `CoreAPI`:

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

Типичные сценарии:

- Логирование: `core.Log("received event: %v", e)`.
- Отправка событий другим компонентам: `core.Emit(shared.NewEvent(...))`.
- Взаимодействие с core-модулями (например, Discord): `core.CallCore("discord", cmdEvent)`.
- Работа с БД: `db := core.Database()` (конкретный интерфейс описан в `AniApi/shared/database.go`).

## События

Плагины получают события через `OnEvent(e Event)`. Типы событий и структура `Event` определены в `AniApi/shared/event.go` и `types.go`.

Примеры типов событий:

- сообщение от пользователя (message create),
- команда (slash command),
- системные события (загрузка плагина и т.п.).

Runtime автоматически подписывает плагины на нужные типы событий через `eventbus`.

## Slash-команды (CommandProvider)

Если плагин хочет предоставлять slash-команды Discord, он может дополнительно реализовать интерфейс `CommandProvider`:

```go
// CommandProvider интерфейс для плагинов, которые предоставляют slash commands
// Это опциональный интерфейс - плагины могут его реализовать
type CommandProvider interface {
    // GetCommands возвращает список команд, которые плагин хочет зарегистрировать
    GetCommands() []*discordgo.ApplicationCommand
}
```

Runtime:

- при загрузке плагинов проверяет, реализует ли плагин `CommandProvider`,
- если да — вызывает `GetCommands()` и собирает все команды,
- передаёт их core-модулю (например, Discord core) для регистрации в Discord.

## Версионирование и совместимость

Каждый плагин должен корректно описывать совместимость с ядром через `CoreVersionCompatibility()` и `CompatibilityRange` из `version.go`:

```go
type CompatibilityRange struct {
    MinVersion string // Минимальная версия ядра (включительно)
    MaxVersion string // Максимальная версия ядра (включительно, пустая строка = без ограничения)
}
```

Runtime при загрузке вызывает `CompatibilityRange.IsCompatible(coreVersion)` и не загружает плагины, несовместимые с текущей версией ядра.

Рекомендуется:

- обновлять диапазон при изменении API ядра,
- использовать `APIVersion` из `AniApi/shared` как основу для проверки.

## Работа с базой данных

Через `CoreAPI.Database()` плагин получает `DatabaseAPI`, привязанный к его идентификатору:

- позволяет хранить данные плагина в общей SQLite БД,
- даёт изолированное пространство по pluginID.

Конкретный интерфейс `DatabaseAPI` описан в `AniApi/shared/database.go` и реализован в `AniCore/pkg/database`.

## Логирование и ошибки

Следуйте общим правилам проекта:

- Используйте `CoreAPI.Log` (внутри — `pkg/logger`), а не стандартный `log`.
- Всегда проверяйте ошибки и по возможности оборачивайте через `fmt.Errorf("...")` с `%w`.
- Сообщения об ошибках начинаются со строчной буквы.

## Сборка плагина

Плагины собираются как `.so` файлы, чтобы runtime мог их динамически загружать.

Обычно в AniCore есть `Makefile` с таргетом `make build-plugins`, который:

- компилирует каждый плагин в `./build/plugins/<name>.so`,
- использует `CGO_ENABLED=1` и нужные флаги компиляции.

После сборки убедитесь, что:

- путь к плагину указан в `config.json` (параметр `plugins.directory`),
- имя плагина присутствует в `plugins.enabled`.

## Пример жизненного цикла плагина

1. Runtime загружает `.so` плагина.
2. Внутри loader ищет реализацию `shared.Plugin`.
3. Вызывается `CoreVersionCompatibility()` и проверяется совместимость.
4. Создаётся `CoreAPIImpl` и вызывается `Init(coreAPI)`.
5. Плагин подписывается на события через `eventbus`.
6. При каждом событии вызывается `OnEvent(e)`.

Подробнее о типах событий и API см. в `api.md`.
