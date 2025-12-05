# AniAPI - Документация для разработчиков

AniAPI - это библиотека для создания плагинов для Discord бота AniDev. Она предоставляет интерфейсы и типы для разработки динамически загружаемых плагинов.

## Версия

Текущая версия: **0.2.0**

## Быстрый старт

### Добавление зависимости

В `Cargo.toml` вашего плагина:

```toml
[dependencies]
aniapi = { path = "../aniapi" }
async-trait = "0.1"
serenity = { version = "0.12", default-features = false, features = ["client", "gateway", "rustls_backend", "model"] }
```

### Минимальный пример плагина

```rust
use aniapi::{export_plugin, Context, Event, Plugin, Result};
use async_trait::async_trait;

#[derive(Default)]
pub struct MyPlugin;

#[async_trait]
impl Plugin for MyPlugin {
    fn name(&self) -> &str {
        "MyPlugin"
    }

    async fn on_load(&mut self, _ctx: &Context) -> Result<()> {
        Ok(())
    }

    async fn on_unload(&mut self, _ctx: &Context) -> Result<()> {
        Ok(())
    }

    async fn on_event(&mut self, _event: &Event, _ctx: &Context) -> Result<()> {
        Ok(())
    }
}

export_plugin!(MyPlugin);
```

## Основные концепции

- **Plugin** - основной трейт, который должен реализовать каждый плагин (DLL)
- **Module** - трейт для встроенных расширений ядра, компилирующихся вместе с AniCore
- **Service** - трейт для сервисов, загружаемых динамически (DLL)
- **Context** - контекст выполнения, предоставляющий доступ к системным сервисам
- **Event** - события, которые плагин может обрабатывать
- **CommandSpec** - спецификация команды Discord
- **BackendService** - сервис для работы с SQLite базой данных (статистика, пользователи, расширяемые таблицы)

### Различия между плагинами, модулями и сервисами

| Тип | Способ загрузки | Назначение | Расположение |
|-----|----------------|------------|--------------|
| **Plugin** | Динамическая (DLL) | Команды Discord, обработка событий | `plugins/*/` |
| **Module** | Компилируется с ядром | Backend-интеграция, унифицированные интерфейсы | `anicore/src/modules/` |
| **Service** | Динамическая (DLL) | Предоставление сервисов плагинам | `plugins/*/` |

## Документация

- [Руководство по разработке плагинов](./PLUGIN_GUIDE.md) - пошаговое руководство по созданию плагинов
- [Руководство по созданию модулей](./MODULE_GUIDE.md) - руководство по созданию модулей для расширения ядра
- [Справочник API](./API_REFERENCE.md) - полная документация API
- [Примеры](./EXAMPLES.md) - примеры использования

## Лицензия

См. основной проект AniDev.

