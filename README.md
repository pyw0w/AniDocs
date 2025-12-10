# AniDocs — документация платформы AniCore

AniDocs — это публичный репозиторий документации для модульной платформы ботов **AniCore**.

Платформа разделена на несколько репозиториев:
- **AniCore** (приватный) — ядро, core-модули, плагины, модули, backend
- **AniApi** (публичный) — публичные интерфейсы для плагинов и core-модулей
- **AniRuntime** (публичный) — запускатель платформы
- **AniDocs** (публичный) — этот репозиторий документации

## Основные разделы документации

- `architecture.md` — архитектура платформы и взаимосвязь репозиториев
- `getting-started.md` — установка и быстрый старт
- `configuration.md` — конфигурация runtime и core-модулей
- `runtime.md` — устройство и работа AniRuntime
- `plugins.md` — разработка плагинов
- `cores.md` — разработка core-модулей и backend
- `api.md` — описание публичного API из `AniApi/shared`
- `development.md` — правила разработки и работы с репозиториями
- `deployment.md` — сборка, релизы и деплой

## Где что находится

Краткая структура после разделения на репозитории:

```text
anidev/
├── AniCore/          # Приватное ядро
│   ├── core/         # Core-модули (discord, telegram, backend)
│   ├── plugins/      # Приватные плагины
│   ├── modules/      # Расширения (core/runtime/plugin)
│   ├── pkg/          # Публичные пакеты для AniRuntime
│   └── ...
├── AniApi/           # Публичный API (shared)
├── AniRuntime/       # Публичный runtime (cmd/runtime)
└── AniDocs/          # Документация (этот репозиторий)
```

Подробная структура описана в `architecture.md` и `development.md`.
