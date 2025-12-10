# Разработка и рабочий процесс

Этот документ описывает, как работать с кодом AniCore, AniApi, AniRuntime и AniDocs в одной директории `anidev/`.

## Структура репозиториев

```text
anidev/
├── AniCore/          # Приватное ядро
│   ├── core/        # Core-модули
│   ├── plugins/     # Приватные плагины
│   ├── modules/     # Расширения
│   ├── pkg/         # Публичные пакеты для runtime
│   └── ...
├── AniApi/          # Публичный API
│   └── shared/      # Публичные интерфейсы
├── AniRuntime/      # Публичный запускатель
│   └── cmd/runtime/ # Runtime код
└── AniDocs/         # Документация (этот репозиторий)
```

## Зависимости

- **AniApi** — независимый, не импортирует AniCore/AniRuntime.
- **AniCore** использует `github.com/pyw0w/AniApi/shared` и свои `pkg/...`.
- **AniRuntime** использует `github.com/pyw0w/AniApi/shared` и `github.com/pyw0w/AniCore/pkg/...`.
- **Плагины** используют только `github.com/pyw0w/AniApi/shared`.

## Локальная разработка с go.work

В корне `anidev/` используется `go.work`, чтобы все модули собирались вместе:

```text
go 1.25.5

use (
    .              // AniCore
    ./AniApi
    ./AniRuntime
)
```

Команды:

```bash
cd /home/USER/anidev
# Синхронизация workspace
go work sync

# Обновление зависимостей
cd /home/USER/anidev
go mod tidy            # AniCore
cd AniRuntime
go mod tidy
cd ../AniApi
go mod tidy
```

## Сборка

В `AniCore` есть `Makefile`, который управляет сборкой всех компонентов:

```bash
cd /home/USER/anidev

# Сборка всех компонентов
make build

# Отдельно:
make build-runtime   # runtime из AniRuntime
make build-cores     # core-модули из AniCore
make build-plugins   # плагины из AniCore
```

Важно:

- Убедитесь, что `CGO_ENABLED=1`, так как плагины и core-модули собираются в `.so`.

## Работа с репозиториями

Каждый репозиторий — отдельный git:

```bash
# AniCore (приватный)
cd /home/USER/anidev
# редактируете core/, plugins/, modules/, pkg/

# AniApi (публичный)
cd /home/USER/anidev/AniApi
# редактируете shared/

# AniRuntime (публичный)
cd /home/USER/anidev/AniRuntime
# редактируете cmd/runtime/

# AniDocs (публичный)
cd /home/USER/anidev/AniDocs
# редактируете документацию
```

### Пример коммитов

```bash
# AniCore
cd /home/USER/anidev
git add .
git commit -m "fix: исправлена загрузка плагинов"
git push

# AniApi
cd /home/USER/anidev/AniApi
git add .
git commit -m "feat: добавлен новый метод в CoreAPI"
git push

# AniRuntime
cd /home/USER/anidev/AniRuntime
git add .
git commit -m "refactor: упрощён runtime main"
git push

# AniDocs
cd /home/USER/anidev/AniDocs
git add .
git commit -m "docs: обновлена структура документации"
git push
```

## Правила кода и импорты

- Везде использовать публичные интерфейсы из `AniApi/shared`.
- **Не** использовать несуществующий пакет `github.com/pyw0w/AniCore/shared`.
- Плагины импортируют только `github.com/pyw0w/AniApi/shared`.
- AniRuntime и AniCore импортируют `github.com/pyw0w/AniCore/pkg/...`.

### Логирование

- Использовать `pkg/logger`.
- Не использовать стандартный пакет `log`.
- Пример: `logger.New("Runtime", logger.INFO)` или через `CoreAPI/RuntimeAPI.Log`.

### Обработка ошибок

- Всегда проверять ошибки.
- Использовать `fmt.Errorf("...: %w", err)` для оборачивания.
- Сообщения об ошибках начинать со строчной буквы.

## Версионирование

- Версия ядра хранится в файле `VERSION` в корне AniCore.
- Плагины реализуют `CoreVersionCompatibility()` и возвращают `CompatibilityRange`.
- При изменении API:
  1. Обновить `AniApi` (публичные интерфейсы).
  2. Обновить `AniCore` под новый API.
  3. При необходимости обновить `AniRuntime`.
  4. Обновить документацию в `AniDocs`.
  5. Обновить `VERSION`.

## CI/CD и проверки перед коммитом

Перед коммитом рекомендуется:

1. Проверить, что код компилируется:
   ```bash
   make build
   ```
2. Запустить линтеры (если настроены):
   ```bash
   make lint
   ```
3. Запустить тесты (если есть):
   ```bash
   make test
   ```
4. Убедиться, что в diff нет секретов (`config.json`, токены, пароли).

## Секреты и конфигурация

- `config.json` и другие файлы с секретами должны быть в `.gitignore`.
- В публичных репозиториях (AniApi, AniRuntime, AniDocs) **не** должно быть токенов и паролей.
- Для CI/CD используйте переменные окружения и секреты в настройках пайплайна.

## Документация

- Вся публичная документация живёт в `AniDocs`.
- AniCore и AniRuntime могут подключать `AniDocs` как git submodule.
- Обновление документации происходит отдельно от кода, но должно сопровождать изменения API и архитектуры.

Подробнее про архитектуру см. в `architecture.md`, про runtime — в `runtime.md`, про плагины и core-модули — в `plugins.md` и `cores.md`.
