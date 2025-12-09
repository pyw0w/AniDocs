# Структура репозитория документации AniDocs

## Репозиторий: github.com/pyw0w/AniDocs

### Структура директорий

```
AniDocs/
├── README.md                    # Главная страница документации
├── plugin-development.md        # Руководство по разработке плагинов
├── repository-structure.md      # Структура репозиториев
├── api-reference.md             # Справочник API
├── migration-guide.md           # Руководство по миграции
├── public-plugins.md            # Разработка публичных плагинов
├── api-repository-structure.md  # Структура API репозитория
├── getting-started.md           # Быстрый старт
├── examples/                    # Примеры
│   ├── simple-plugin/
│   ├── commands-plugin/
│   └── custom-core/
├── guides/                      # Подробные руководства
│   ├── building-plugins.md
│   ├── creating-cores.md
│   └── security.md
└── api/                         # API документация
    ├── shared/
    │   ├── plugin.md
    │   ├── core-api.md
    │   └── events.md
    └── runtime/
        └── configuration.md
```

### Основные документы

#### README.md
- Обзор платформы
- Ссылки на основные разделы
- Быстрая навигация

#### plugin-development.md
- Как создать плагин
- Примеры кода
- Требования и ограничения

#### repository-structure.md
- Описание всех репозиториев
- Структура зависимостей
- Использование submodules

#### api-reference.md
- Полный справочник API
- Интерфейсы и типы
- Примеры использования

### Версионирование

Документация версионируется вместе с API:
- v0.1.0 - начальная версия
- v0.2.0 - обновления для новых версий API
- v1.0.0 - стабильная документация

### Использование как submodule

Документация подключается к другим репозиториям:

```bash
# В AniCore
git submodule add https://github.com/pyw0w/AniDocs.git docs

# В AniRuntime
git submodule add https://github.com/pyw0w/AniDocs.git docs
```

### Обновление документации

1. Внести изменения в AniDocs
2. Закоммитить и запушить
3. Обновить submodule в других репозиториях:
   ```bash
   git submodule update --remote docs
   git add docs
   git commit -m "Update documentation"
   ```

### Формат документации

- Markdown (.md)
- Поддержка GitHub Flavored Markdown
- Возможность генерации статического сайта (MkDocs, Docusaurus и т.д.)

