# Справочник API AniAPI

## Типы ошибок

### `Error`

Перечисление ошибок, которые могут возникнуть при работе с API.

```rust
pub enum Error {
    Plugin(String),        // Ошибка плагина
    System(String),        // Системная ошибка
    Other(anyhow::Error),  // Прочие ошибки
    Io(std::io::Error),    // Ошибки ввода/вывода
    LibLoading(String),    // Ошибка загрузки библиотеки
    Serenity(String),      // Ошибка Serenity (Discord API)
}
```

### `Result<T>`

Алиас для `std::result::Result<T, Error>`.

## Команды

### `CommandSpec`

Структура для описания команды Discord.

```rust
pub struct CommandSpec {
    pub name: String,                    // Имя команды
    pub description: String,             // Описание команды
    pub permission: Option<Permissions>,  // Требуемые права
    pub cooldown: Option<u64>,           // Кулдаун в секундах
}
```

#### Методы

- `new(name: &str, description: &str) -> Self` - создать новую спецификацию
- `permission(self, p: Permissions) -> Self` - установить требуемые права
- `cooldown(self, seconds: u64) -> Self` - установить кулдаун
- `to_create_command(&self) -> CreateCommand` - преобразовать в Serenity команду

## События

### `Event`

Перечисление событий, которые может обрабатывать плагин.

```rust
pub enum Event {
    Message(String, String),                    // Сообщение (автор, содержимое)
    Discord(Arc<GatewayIntents>),               // Настройки Discord
    SerenityEvent(Arc<serenity::model::event::Event>), // Событие Serenity
    Interaction(Arc<Interaction>),              // Взаимодействие (команда/кнопка)
    VoiceStateUpdate(Arc<VoiceState>),          // Обновление голосового состояния
    System(SystemEvent),                        // Системное событие
}
```

### `SystemEvent`

Системные события жизненного цикла.

```rust
pub enum SystemEvent {
    Load,   // Плагин загружен
    Unload, // Плагин выгружен
    Start,  // Система запущена
    Stop,   // Система остановлена
}
```

## Трейты

### `Plugin`

Основной трейт, который должен реализовать каждый плагин.

```rust
#[async_trait]
pub trait Plugin: Any + Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str { "" }
    fn commands(&self) -> Vec<CommandSpec> { Vec::new() }
    
    async fn on_load(&mut self, ctx: &Context) -> Result<()>;
    async fn on_unload(&mut self, ctx: &Context) -> Result<()>;
    async fn on_event(&mut self, event: &Event, ctx: &Context) -> Result<()>;
}
```

#### Методы

- `name(&self) -> &str` - уникальное имя плагина
- `description(&self) -> &str` - описание плагина (опционально)
- `commands(&self) -> Vec<CommandSpec>` - список команд, регистрируемых плагином
- `on_load(&mut self, ctx: &Context) -> Result<()>` - вызывается при загрузке плагина
- `on_unload(&mut self, ctx: &Context) -> Result<()>` - вызывается при выгрузке плагина
- `on_event(&mut self, event: &Event, ctx: &Context) -> Result<()>` - вызывается при каждом событии

### `Cache`

Интерфейс для работы с кэшем.

```rust
#[async_trait]
pub trait Cache: Send + Sync {
    async fn get(&self, key: &str) -> Option<String>;
    async fn set(&self, key: &str, value: &str);
    async fn remove(&self, key: &str);
}
```

### `Responder`

Интерфейс для отправки ответов на взаимодействия.

```rust
#[async_trait]
pub trait Responder: Send + Sync {
    async fn respond(
        &self, 
        interaction_id: u64, 
        token: &str, 
        response: CreateInteractionResponse
    ) -> Result<()>;
}
```

### `GuildManager`

Интерфейс для управления гильдией (сервером Discord).

```rust
#[async_trait]
pub trait GuildManager: Send + Sync {
    async fn create_voice_channel(
        &self, 
        guild_id: u64, 
        name: &str, 
        category_id: Option<u64>
    ) -> Result<u64>;
    
    async fn delete_channel(&self, channel_id: u64) -> Result<()>;
    
    async fn move_member(
        &self, 
        guild_id: u64, 
        user_id: u64, 
        channel_id: u64
    ) -> Result<()>;
}
```

### `Dashboard`

Интерфейс для отправки метрик в дашборд.

```rust
#[async_trait]
pub trait Dashboard: Send + Sync {
    async fn update_latency(&self, latency_ms: f64);
    async fn record_command_usage(&self, command_name: &str);
}
```

### `ConfigManager`

Интерфейс для работы с конфигурацией плагинов.

```rust
#[async_trait]
pub trait ConfigManager: Send + Sync {
    async fn get_config(&self, plugin_name: &str) -> Result<Option<toml::Value>>;
}
```

### `BackendService`

Сервис для работы с базой данных (SQLite). Предоставляет функционал для статистики, пользователей, плагинов и расширяемых таблиц.

```rust
#[async_trait]
pub trait BackendService: Send + Sync {
    // Статистика
    async fn save_command_stats(&self, command_name: &str, count: u64) -> Result<()>;
    async fn get_command_stats(&self, command_name: &str) -> Result<Option<u64>>;
    async fn save_latency(&self, latency_ms: f64) -> Result<()>;
    async fn get_latency_history(&self, limit: Option<usize>) -> Result<Vec<f64>>;
    
    // Пользователи
    async fn create_user(&self, user_id: u64, username: &str) -> Result<()>;
    async fn update_user(&self, user_id: u64, username: &str) -> Result<()>;
    async fn get_user(&self, user_id: u64) -> Result<Option<(u64, String)>>;
    async fn get_all_users(&self) -> Result<Vec<(u64, String)>>;
    
    // Расширяемые таблицы для плагинов
    async fn create_plugin_table(&self, plugin_name: &str, table_name: &str, columns: &str) -> Result<()>;
    async fn insert_plugin_data(&self, plugin_name: &str, table_name: &str, values: HashMap<String, String>) -> Result<()>;
    async fn query_plugin_data(&self, plugin_name: &str, table_name: &str, query: &str) -> Result<Vec<HashMap<String, String>>>;
    async fn delete_plugin_data(&self, plugin_name: &str, table_name: &str, condition: &str) -> Result<()>;
    
    // Произвольные SQL запросы
    async fn execute_query(&self, query: &str) -> Result<Vec<HashMap<String, String>>>;
}
```

#### Методы статистики

- `save_command_stats(command_name, count)` - сохранить статистику использования команды
- `get_command_stats(command_name)` - получить статистику использования команды
- `save_latency(latency_ms)` - сохранить значение латентности
- `get_latency_history(limit)` - получить историю латентности (последние N записей)

#### Методы работы с пользователями

- `create_user(user_id, username)` - создать нового пользователя
- `update_user(user_id, username)` - обновить данные пользователя
- `get_user(user_id)` - получить пользователя по ID
- `get_all_users()` - получить всех пользователей

#### Методы работы с таблицами плагинов

- `create_plugin_table(plugin_name, table_name, columns)` - создать таблицу для плагина
- `insert_plugin_data(plugin_name, table_name, values)` - вставить данные в таблицу плагина
- `query_plugin_data(plugin_name, table_name, query)` - выполнить запрос к таблице плагина
- `delete_plugin_data(plugin_name, table_name, condition)` - удалить данные из таблицы плагина

#### Произвольные SQL запросы

- `execute_query(query)` - выполнить произвольный SQL запрос для расширенных сценариев

**Доступ к BackendService**: Получается через `Context.services` используя `get_service_any("BackendService")`.

## Контекст

### `Context`

Контекст выполнения плагина, предоставляющий доступ к системным сервисам.

```rust
pub struct Context {
    pub cache: Arc<dyn Cache>,                    // Кэш (всегда доступен)
    pub responder: Option<Arc<dyn Responder>>,    // Отвечать на взаимодействия
    pub guild: Option<Arc<dyn GuildManager>>,    // Управление гильдией
    pub dashboard: Option<Arc<dyn Dashboard>>,   // Дашборд
    pub logger: Option<&'static dyn log::Log>,   // Логгер
    pub config: Option<Arc<dyn ConfigManager>>,  // Конфигурация
    pub services: Option<Arc<std::sync::Mutex<dyn ServiceProvider>>>, // Провайдер сервисов
}
```

**Важно**: Все поля кроме `cache` опциональны. Всегда проверяйте наличие сервиса перед использованием.

**Доступ к сервисам**: Используйте `Context.services` для получения сервисов через `ServiceProvider`:
```rust
if let Some(services) = &ctx.services {
    if let Ok(services_guard) = services.lock() {
        if let Some(backend_any) = services_guard.get_service_any("BackendService") {
            if let Some(backend) = backend_any.downcast_ref::<Arc<dyn BackendService>>() {
                // Используйте backend
            }
        }
    }
}
```

## Макросы

### `export_plugin!`

Макрос для экспорта функции создания плагина.

```rust
export_plugin!(MyPlugin);
```

Генерирует функцию `_create_plugin()`, которая используется системой загрузки плагинов.

