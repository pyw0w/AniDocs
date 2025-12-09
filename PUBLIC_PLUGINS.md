# Разработка публичных плагинов

Публичные плагины могут разрабатываться в отдельных репозиториях и использовать только публичный API.

## Создание публичного плагина

### 1. Создание репозитория

```bash
mkdir my-anicore-plugin
cd my-anicore-plugin
go mod init github.com/yourusername/my-anicore-plugin
```

### 2. Установка зависимостей

```bash
go get github.com/pyw0w/AniApi@latest
```

### 3. Структура плагина

```
my-anicore-plugin/
├── main.go
├── go.mod
├── go.sum
└── README.md
```

### 4. Пример кода

```go
package main

import (
    "fmt"
    "github.com/pyw0w/AniApi/shared"
)

type MyPlugin struct {
    name    string
    version string
    core    shared.CoreAPI
}

var Plugin shared.Plugin = &MyPlugin{
    name:    "my-plugin",
    version: "1.0.0",
}

func (p *MyPlugin) Name() string {
    return p.name
}

func (p *MyPlugin) Version() string {
    return p.version
}

func (p *MyPlugin) Init(core shared.CoreAPI) error {
    p.core = core
    p.core.Log("My plugin initialized")
    return nil
}

func (p *MyPlugin) OnEvent(e shared.Event) error {
    if e.Type == string(shared.EventTypeMessageCreate) {
        p.core.Log("Received message: %s", e.Data["content"])
    }
    return nil
}
```

### 5. Сборка плагина

```bash
CGO_ENABLED=1 go build -buildmode=plugin -o my-plugin.so main.go
```

### 6. Использование

1. Скопируйте `.so` файл в директорию плагинов runtime
2. Добавьте плагин в `config.json`:
```json
{
  "plugins": {
    "enabled": ["my-plugin"],
    "directory": "./build/plugins"
  }
}
```

## Плагины с командами

Если плагин предоставляет slash commands:

```go
import (
    "github.com/bwmarrin/discordgo"
    "github.com/pyw0w/AniApi/shared"
)

func (p *MyPlugin) GetCommands() []*discordgo.ApplicationCommand {
    return []*discordgo.ApplicationCommand{
        {
            Name:        "mycommand",
            Description: "My custom command",
        },
    }
}
```

Плагин автоматически реализует интерфейс `shared.CommandProvider`.

## Распространение плагинов

Публичные плагины могут:
- Быть опубликованы в отдельных репозиториях
- Распространяться через релизы GitHub
- Использоваться любым пользователем AniCore

## Ограничения

Публичные плагины:
- Не имеют доступа к внутренним пакетам runtime
- Могут использовать только публичный API
- Должны быть собраны с той же версией Go, что и runtime

