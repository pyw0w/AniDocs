# Руководство по разработке плагинов

Это руководство описывает, как создать плагин для AniCore.

## Требования

- Go 1.25.5 (та же версия, что и runtime)
- Linux (плагины работают только на Linux)
- Публичный API репозиторий: `github.com/pyw0w/AniApi`

## Установка зависимостей

```bash
go mod init your-plugin-name
go get github.com/pyw0w/AniApi@latest
```

## Структура плагина

Создайте директорию для вашего плагина:

```
plugins/
└── myplugin/
    └── main.go
```

## Реализация интерфейса Plugin

Каждый плагин должен реализовать интерфейс `shared.Plugin`:

```go
package main

import (
    "github.com/pyw0w/AniApi/shared"
)

type MyPlugin struct {
    name    string
    version string
    core    shared.CoreAPI
}

// Plugin должен быть экспортирован как var Plugin
var Plugin shared.Plugin = &MyPlugin{
    name:    "myplugin",
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
    p.core.Log("MyPlugin initialized")
    return nil
}

func (p *MyPlugin) OnEvent(e shared.Event) error {
    // Обработка событий
    p.core.Log("Received event: %s", e.Type)
    return nil
}
```

## Экспорт плагина

**Важно**: Плагин должен экспортировать переменную `Plugin` типа `shared.Plugin`:

```go
var Plugin shared.Plugin = &MyPlugin{...}
```

Runtime ищет именно эту переменную при загрузке плагина.

## Использование CoreAPI

Плагин получает `CoreAPI` в методе `Init()`. Через него можно:

### Логирование

```go
p.core.Log("Message: %s", "Hello")
```

### Отправка событий

```go
event := shared.NewEvent("my_event", map[string]string{
    "key": "value",
}, p.name)
p.core.Emit(event)
```

### Вызов команд в core-модулях

```go
cmd := shared.NewEvent("send_message", map[string]string{
    "content": "Hello from plugin",
}, p.name)
p.core.CallCore("discord", cmd)
```

## Обработка событий

Метод `OnEvent()` вызывается для каждого события, на которое подписан плагин. Runtime автоматически подписывает плагины на события типа `message_create` и `command`.

Пример обработки сообщений:

```go
func (p *MyPlugin) OnEvent(e shared.Event) error {
    if e.Type == string(shared.EventTypeMessageCreate) {
        content := e.Data["content"]
        source := e.Source // Имя core-модуля
        
        // Обработка сообщения
        p.core.Log("Message from %s: %s", source, content)
        
        // Отправка ответа
        response := shared.NewEvent("send_message", map[string]string{
            "content": "Echo: " + content,
        }, p.name)
        
        return p.core.CallCore(source, response)
    }
    return nil
}
```

## Сборка плагина

Соберите плагин как .so файл:

```bash
cd plugins/myplugin
CGO_ENABLED=1 go build -buildmode=plugin -o ../../build/plugins/myplugin.so
```

⚠️ **Важно**: Для сборки плагинов требуется включенный CGO (`CGO_ENABLED=1`).

Или используйте Makefile (CGO включен автоматически):

```bash
make build-plugins
```

## Требования к версии Go

⚠️ **Критически важно**: Runtime и все плагины должны быть собраны **одной и той же версией Go** (1.25.5).

Проверка версии:

```bash
go version  # Должно быть go1.25.5
```

## Module path

Плагины должны использовать публичный API репозиторий.

В `go.mod` вашего плагина:

```go
module your-plugin-name

require github.com/pyw0w/AniApi v0.1.0
```

## Тестирование плагина

1. Соберите плагин: `go build -buildmode=plugin -o build/plugins/myplugin.so plugins/myplugin/main.go`
2. Добавьте плагин в `config.json`:
   ```json
   {
     "plugins": {
       "enabled": ["myplugin"],
       "directory": "./build/plugins"
     }
   }
   ```
3. Запустите runtime: `./build/runtime -config config.json`
4. Проверьте логи на наличие сообщений от плагина

## Примеры событий

### События от core-модулей

- `message_create` - новое сообщение
  - `channel` / `chat` - канал/чат
  - `author` / `user` - автор
  - `content` - содержимое

- `command` - команда от пользователя
  - `command` - имя команды
  - `args` - аргументы

### Системные события

- `plugin_loaded` - плагин загружен
- `core_started` - core-модуль запущен
- `core_stopped` - core-модуль остановлен

## Отладка

Используйте `core.Log()` для логирования:

```go
p.core.Log("Debug: variable = %v", variable)
```

Логи будут видны в выводе runtime с префиксом `[Plugin:name]`.

## Ограничения

- Плагины нельзя выгрузить в runtime (ограничение Go)
- Для обновления плагина требуется перезапуск runtime
- Плагин выполняется в процессе runtime (полный доступ к процессу)
- Плагины работают только на Linux

## Безопасность

- Загружайте плагины только из доверенных источников
- Используйте проверку хешей в конфигурации
- Ограничьте пути загрузки через `allowed_paths`

## Полный пример

См. `plugins/example/main.go` для полного рабочего примера плагина.

