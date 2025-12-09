# Руководство по миграции на раздельные репозитории

## Шаг 1: Создание публичного репозитория API

1. Создайте новый публичный репозиторий: `github.com/pyw0w/AniApi`

2. Структура публичного репозитория:
```
AniApi/
├── shared/
│   ├── plugin.go
│   ├── core_api.go
│   ├── event.go
│   ├── types.go
│   └── commands.go
├── go.mod
├── README.md
└── examples/
    └── simple-plugin/
```

3. Скопируйте все файлы из `shared/` в публичный репозиторий

4. Создайте `go.mod` в публичном репозитории:
```go
module github.com/pyw0w/AniApi

go 1.25.5

require github.com/bwmarrin/discordgo v0.28.1
```

## Шаг 2: Создание публичного репозитория Runtime

1. Создайте новый публичный репозиторий: `github.com/pyw0w/AniRuntime`

2. Структура публичного репозитория:
```
AniRuntime/
├── cmd/
│   └── runtime/
│       └── main.go
├── go.mod
├── README.md
└── config.example.json
```

3. Скопируйте `cmd/runtime/` в публичный репозиторий

4. Создайте `go.mod` в публичном репозитории:
```go
module github.com/pyw0w/AniRuntime

go 1.25.5

require (
    github.com/pyw0w/AniApi v0.1.0
    github.com/pyw0w/AniCore v0.1.0  // через replace для локальной разработки
)
```

## Шаг 3: Обновление приватного репозитория ядра

1. Обновите `go.mod` в приватном репозитории:
```go
module github.com/pyw0w/AniCore

go 1.25.5

require github.com/pyw0w/AniApi v0.1.0

require github.com/bwmarrin/discordgo v0.28.1
```

2. Обновите импорты в `core/discord/main.go`:
```go
import (
    "github.com/pyw0w/AniApi/shared"
    // ...
)
```

3. Обновите импорты в `plugins/*/main.go`:
```go
import (
    "github.com/pyw0w/AniApi/shared"
    // ...
)
```

4. Удалите директорию `cmd/runtime/` из приватного репозитория (она теперь в AniRuntime)

5. Удалите директорию `shared/` из приватного репозитория (она теперь в AniApi)

## Шаг 4: Обновление Runtime репозитория

1. Обновите импорты в `cmd/runtime/main.go`:
```go
import (
    "github.com/pyw0w/AniApi/shared"
    // internal пакеты из AniCore через replace
)
```

2. Для локальной разработки используйте `go.work` или `replace`:
```go
replace github.com/pyw0w/AniCore => ../AniCore
```

## Шаг 5: Публикация API

1. Создайте релиз в публичном репозитории AniApi
2. Добавьте теги версий (v0.1.0, v0.2.0 и т.д.)
3. Обновите зависимости:
```bash
go get github.com/pyw0w/AniApi@latest
go mod tidy
```

## Шаг 6: Создание репозитория документации

1. Создайте новый публичный репозиторий: `github.com/pyw0w/AniDocs`

2. Запустите скрипт настройки:
```bash
./scripts/setup-docs-repo.sh
```

3. Инициализируйте git репозиторий:
```bash
cd ../AniDocs
git init
git add .
git commit -m "Initial documentation"
git remote add origin https://github.com/pyw0w/AniDocs.git
git push -u origin main
```

4. Добавьте как submodule в другие репозитории:
```bash
# В AniCore
./scripts/add-docs-submodule.sh

# В AniRuntime
./scripts/add-docs-submodule.sh
```

## Структура после миграции

```
AniApi/ (публичный)
  └── shared/

AniCore/ (приватный)
  ├── internal/
  ├── core/
  ├── plugins/
  └── docs/ (submodule -> AniDocs)

AniRuntime/ (публичный)
  ├── cmd/runtime/
  └── docs/ (submodule -> AniDocs)

AniDocs/ (публичный)
  ├── plugin-development.md
  ├── repository-structure.md
  └── ...
```
