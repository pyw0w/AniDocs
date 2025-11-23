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
}
```

**Важно**: Все поля кроме `cache` опциональны. Всегда проверяйте наличие сервиса перед использованием.

## Макросы

### `export_plugin!`

Макрос для экспорта функции создания плагина.

```rust
export_plugin!(MyPlugin);
```

Генерирует функцию `_create_plugin()`, которая используется системой загрузки плагинов.

