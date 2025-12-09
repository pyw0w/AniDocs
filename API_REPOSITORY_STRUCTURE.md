# Структура публичного API репозитория

## Репозиторий: github.com/pyw0w/AniApi

### Структура директорий

```
AniApi/
├── shared/                    # Публичный API пакет
│   ├── plugin.go             # Интерфейс Plugin
│   ├── core_api.go           # Интерфейс CoreAPI
│   ├── event.go              # Тип Event
│   ├── types.go              # Дополнительные типы и интерфейсы
│   └── commands.go           # Интерфейс CommandProvider
├── examples/                 # Примеры плагинов
│   ├── simple-plugin/       # Простой пример плагина
│   └── commands-plugin/     # Пример плагина с командами
├── go.mod
├── go.sum
├── README.md
└── LICENSE
```

### Содержимое shared/

#### plugin.go
```go
package shared

// Plugin интерфейс, который должен реализовать каждый плагин
type Plugin interface {
    Name() string
    Version() string
    Init(core CoreAPI) error
    OnEvent(e Event) error
}
```

#### core_api.go
```go
package shared

// CoreAPI предоставляет API для плагинов
type CoreAPI interface {
    Log(format string, args ...interface{})
    Emit(e Event) error
    CallCore(coreName string, cmd Event) error
}
```

#### event.go
```go
package shared

import "time"

type Event struct {
    Type      string
    Data      map[string]string
    Source    string
    Timestamp time.Time
}

func NewEvent(eventType string, data map[string]string, source string) Event
```

#### types.go
```go
package shared

import "github.com/bwmarrin/discordgo"

// EventType, Capabilities, RuntimeAPI, Core
```

#### commands.go
```go
package shared

import "github.com/bwmarrin/discordgo"

// CommandProvider для плагинов с командами
type CommandProvider interface {
    GetCommands() []*discordgo.ApplicationCommand
}
```

### Версионирование

API репозиторий использует семантическое версионирование:
- v0.1.0 - начальная версия
- v0.2.0 - добавление новых интерфейсов (обратная совместимость)
- v1.0.0 - стабильный API

### Использование в плагинах

```go
package main

import "github.com/pyw0w/AniApi/shared"

var Plugin shared.Plugin = &MyPlugin{...}
```

### Зависимости

API репозиторий должен иметь минимальные зависимости:
- `github.com/bwmarrin/discordgo` - только для типов команд (не для runtime)
