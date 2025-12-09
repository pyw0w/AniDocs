# AniDocs

Централизованная документация для платформы AniCore.

## Структура

- [Руководство по разработке плагинов](plugin-development.md)
- [Структура репозиториев](repository-structure.md)
- [Справочник API](api-reference.md)
- [Руководство по миграции](migration-guide.md)
- [Разработка публичных плагинов](public-plugins.md)
- [Структура API репозитория](api-repository-structure.md)

## Использование

Эта документация используется как git submodule в других репозиториях:

```bash
# Добавление submodule
git submodule add https://github.com/pyw0w/AniDocs.git docs

# Обновление submodule
git submodule update --remote docs
```

## Репозитории

- **AniApi**: [github.com/pyw0w/AniApi](https://github.com/pyw0w/AniApi) - Публичный API
- **AniCore**: [github.com/pyw0w/AniCore](https://github.com/pyw0w/AniCore) - Приватное ядро
- **AniRuntime**: [github.com/pyw0w/AniRuntime](https://github.com/pyw0w/AniRuntime) - Публичный запускатель

## Вклад в документацию

Документация открыта для улучшений. Создавайте issues и pull requests!
