# Руководство по созданию модулей для AniCore

Это руководство поможет вам понять, что такое модули и как создать свой модуль для расширения функциональности ядра AniCore.

## Что такое модули?

Модули - это **встроенные расширения ядра**, которые компилируются вместе с AniCore. В отличие от плагинов (DLL) и сервисов (DLL), модули:

- ✅ Компилируются вместе с ядром (не загружаются динамически)
- ✅ Имеют прямой доступ к внутренним компонентам AniCore
- ✅ Предоставляют backend-интеграцию (веб-сервисы, API, панели управления)
- ✅ Обеспечивают унифицированные интерфейсы для плагинов (БД, утилиты, абстракции)
- ✅ Регистрируются и управляются через ModuleManager

## Отличия модулей от плагинов и сервисов

| Характеристика | Модули | Плагины | Сервисы |
|----------------|--------|---------|---------|
| Способ загрузки | Компилируются вместе с ядром | Динамическая загрузка (DLL) | Динамическая загрузка (DLL) |
| Доступ к ядру | Прямой доступ | Только через API | Только через API |
| Использование | Backend-интеграция, унифицированные интерфейсы | Команды Discord, обработка событий | Предоставление сервисов плагинам |
| Расположение | `anicore/src/modules/` | `plugins/*/` | `plugins/*/` |

## Структура модуля

Модули размещаются в каталоге `anicore/src/modules/`. Каждый модуль должен находиться в отдельной подпапке:

```
anicore/src/modules/
  ├── mod.rs                    # Регистрация всех модулей
  ├── dashboard/                # Пример модуля Dashboard
  │   ├── mod.rs               # Реализация Module для Dashboard
  │   └── server.rs            # Axum сервер
  └── my_module/                # Ваш новый модуль
      ├── mod.rs               # Реализация Module
      └── ...
```

## Создание модуля

### 1. Создание структуры

Создайте новую папку для вашего модуля в `anicore/src/modules/`:

```
anicore/src/modules/my_module/
  ├── mod.rs
```

### 2. Реализация трейта Module

В `mod.rs` реализуйте трейт `Module`:

```rust
use aniapi::{Context, Module, Result, ServiceProvider};
use async_trait::async_trait;
use std::sync::Arc;

pub struct MyModule {
    // Ваши поля
}

impl MyModule {
    pub fn new() -> Self {
        Self {
            // Инициализация
        }
    }
}

#[async_trait]
impl Module for MyModule {
    fn name(&self) -> &str {
        "MyModule"
    }
    
    fn description(&self) -> &str {
        "Описание моего модуля"
    }
    
    fn version(&self) -> &str {
        "1.0.0"
    }
    
    async fn initialize(&mut self, ctx: &Context) -> Result<()> {
        // Инициализация модуля
        // Здесь можно получить сервисы из ServiceProvider
        // и запустить необходимые компоненты
        Ok(())
    }
    
    async fn shutdown(&mut self, _ctx: &Context) -> Result<()> {
        // Завершение работы модуля
        // Остановка серверов, закрытие соединений и т.д.
        Ok(())
    }
    
    async fn register_services(&self, ctx: &Context, provider: &mut dyn ServiceProvider) -> Result<()> {
        // Регистрация сервисов, которые модуль предоставляет плагинам
        // Например:
        // let my_service = Arc::new(MyService::new());
        // provider.register_service_any("MyService", my_service);
        Ok(())
    }
}

impl Default for MyModule {
    fn default() -> Self {
        Self::new()
    }
}
```

### 3. Регистрация модуля

Добавьте модуль в `anicore/src/modules/mod.rs`:

```rust
pub mod dashboard;
pub mod my_module;  // Добавьте ваш модуль

pub use dashboard::DashboardModule;
pub use my_module::MyModule;  // Добавьте экспорт
```

### 4. Регистрация в ModuleManager

Зарегистрируйте модуль в `anicore/src/lib.rs` в методе `Bot::new()`:

```rust
// Initialize module manager and register modules
let mut module_manager = ModuleManager::new();
module_manager.register_module(Box::new(DashboardModule::new()));
module_manager.register_module(Box::new(MyModule::new()));  // Добавьте ваш модуль

// Initialize modules
module_manager.initialize_all(&context).await?;

// Register services from modules
{
    let mut provider = service_provider.lock().unwrap();
    module_manager.register_all_services(&context, &mut *provider).await?;
}
```

## Пример: Модуль Dashboard

Пример модуля Dashboard показывает, как создать модуль с веб-сервером:

```rust
use aniapi::{Context, Dashboard, Module, Result, BackendService, ServiceProvider};
use async_trait::async_trait;
use std::sync::Arc;

mod server;

pub struct DashboardModule {
    backend_service: Option<Arc<dyn BackendService>>,
    dashboard_impl: Arc<DashboardImpl>,
}

#[async_trait]
impl Module for DashboardModule {
    fn name(&self) -> &str {
        "Dashboard"
    }
    
    async fn initialize(&mut self, ctx: &Context) -> Result<()> {
        // Получить BackendService из ServiceProvider
        if let Some(services) = &ctx.services {
            if let Ok(mut provider) = services.lock() {
                if let Some(backend_any) = provider.get_service_any("BackendService") {
                    // Обработка получения BackendService
                }
            }
        }
        
        // Запустить веб-сервер
        server::start_dashboard_server(self.backend_service.clone());
        Ok(())
    }
    
    async fn register_services(&self, _ctx: &Context, provider: &mut dyn ServiceProvider) -> Result<()> {
        // Зарегистрировать Dashboard trait для использования другими компонентами
        provider.register_service_any("Dashboard", self.dashboard_impl.clone() as Arc<dyn std::any::Any + Send + Sync>);
        Ok(())
    }
}
```

## Предоставление сервисов плагинам

Модули могут предоставлять сервисы плагинам через ServiceProvider:

```rust
async fn register_services(&self, _ctx: &Context, provider: &mut dyn ServiceProvider) -> Result<()> {
    // Создать сервис
    let my_service = Arc::new(MyService::new());
    
    // Зарегистрировать в ServiceProvider
    provider.register_service_any("MyService", my_service as Arc<dyn std::any::Any + Send + Sync>);
    
    Ok(())
}
```

Плагины могут получать доступ к этому сервису через Context:

```rust
if let Some(services) = &ctx.services {
    if let Ok(mut provider) = services.lock() {
        if let Some(service_any) = provider.get_service_any("MyService") {
            if let Some(my_service) = service_any.downcast_ref::<Arc<MyService>>() {
                // Использовать сервис
            }
        }
    }
}
```

## Жизненный цикл модуля

1. **Регистрация** - модуль регистрируется в ModuleManager при создании Bot
2. **Инициализация** - вызывается `initialize()` после создания Context
3. **Регистрация сервисов** - вызывается `register_services()` после инициализации всех модулей
4. **Работа** - модуль работает в фоне, предоставляет сервисы плагинам
5. **Завершение** - вызывается `shutdown()` при остановке ядра

## Рекомендации

- ✅ Модули должны быть независимыми друг от друга
- ✅ Используйте только Context и ServiceProvider для взаимодействия
- ✅ Предоставляйте четкие интерфейсы через ServiceProvider
- ✅ Обрабатывайте ошибки gracefully
- ✅ Документируйте предоставляемые сервисы

- ❌ Не создавайте циклические зависимости между модулями
- ❌ Не используйте приватные API других модулей
- ❌ Не блокируйте инициализацию других модулей

## Типы модулей

### Backend-интеграция модули

Модули, которые предоставляют backend-интеграцию (веб-сервисы, API, панели управления):

- **Dashboard** - веб-панель мониторинга
- **API Gateway** - REST API для внешних сервисов (будущий)

### Интерфейсные модули

Модули, которые предоставляют унифицированные интерфейсы для плагинов:

- **Database Helper** - helper-функции для работы с БД (будущий)
- **ORM Module** - ORM-подобный интерфейс (будущий)

## Примеры использования

См. реализацию модуля Dashboard в `anicore/src/modules/dashboard/` для полного примера модуля с веб-сервером и интеграцией с BackendService.
