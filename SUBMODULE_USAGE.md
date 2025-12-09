# Использование AniDocs как Git Submodule

## Что такое Git Submodule?

Git submodule позволяет включить один git репозиторий как поддиректорию другого репозитория. Это идеально для документации, которая должна быть доступна в нескольких репозиториях.

## Добавление AniDocs как submodule

### В репозитории AniCore

```bash
cd /path/to/AniCore
git submodule add https://github.com/pyw0w/AniDocs.git docs
git commit -m "Add AniDocs as submodule"
```

### В репозитории AniRuntime

```bash
cd /path/to/AniRuntime
git submodule add https://github.com/pyw0w/AniDocs.git docs
git commit -m "Add AniDocs as submodule"
```

### Использование скрипта

```bash
./scripts/add-docs-submodule.sh
```

## Клонирование репозитория с submodules

### При клонировании

```bash
git clone --recursive https://github.com/pyw0w/AniCore.git
```

### Если уже клонирован

```bash
git clone https://github.com/pyw0w/AniCore.git
cd AniCore
git submodule update --init --recursive
```

## Обновление submodule

### Обновить до последней версии

```bash
git submodule update --remote docs
git add docs
git commit -m "Update documentation"
```

### Обновить до конкретной версии

```bash
cd docs
git checkout v0.1.0
cd ..
git add docs
git commit -m "Update documentation to v0.1.0"
```

## Работа с submodule

### Перейти в submodule

```bash
cd docs
# Теперь вы в репозитории AniDocs
```

### Внести изменения в документацию

```bash
cd docs
# Внести изменения
git add .
git commit -m "Update documentation"
git push

# Вернуться в основной репозиторий
cd ..
git add docs
git commit -m "Update docs submodule"
git push
```

### Удалить submodule

```bash
git submodule deinit docs
git rm docs
rm -rf .git/modules/docs
```

## Проблемы и решения

### Submodule показывает как измененный после клонирования

```bash
git submodule update --init --recursive
```

### Submodule не обновляется

```bash
cd docs
git fetch
git pull
cd ..
git add docs
git commit -m "Update docs"
```

### Конфликт версий submodule

Если разные репозитории используют разные версии документации:

```bash
# Проверить текущую версию
cd docs
git log --oneline -1

# Обновить до нужной версии
git checkout <commit-hash>
cd ..
git add docs
git commit -m "Sync docs version"
```

## Best Practices

1. **Версионирование**: Используйте теги для версионирования документации
2. **Синхронизация**: Обновляйте submodule в всех репозиториях одновременно
3. **Коммиты**: Коммитьте изменения в submodule отдельно от основного репозитория
4. **CI/CD**: Настройте автоматическое обновление submodule в CI/CD пайплайнах

## Структура после добавления

```
AniCore/
├── internal/
├── core/
├── plugins/
├── docs/          # <- submodule (AniDocs)
│   ├── .git
│   ├── README.md
│   └── ...
└── .gitmodules    # <- конфигурация submodules
```

Файл `.gitmodules` содержит:
```ini
[submodule "docs"]
    path = docs
    url = https://github.com/pyw0w/AniDocs.git
```

