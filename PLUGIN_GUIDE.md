# Руководство по разработке плагинов для AniAPI

Это руководство поможет вам создать свой первый плагин для AniDev.

## Создание проекта плагина

### 1. Структура проекта

Создайте новую папку в `plugins/` и структуру проекта:

```
plugins/my_plugin/
  Cargo.toml
  src/
    lib.rs
```

### 2. Настройка Cargo.toml

```toml
[package]
name = "my_plugin"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
aniapi = { path = "../../aniapi" }
async-trait = "0.1"
serenity = { version = "0.12", default-features = false, features = ["client", "gateway", "rustls_backend", "model"] }
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1.0", features = ["macros"] }
log = "0.4"
toml = "0.8"
```

**Важно**: `crate-type = ["cdylib"]` необходим для динамической загрузки.

### 3. Базовый шаблон плагина

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

    fn description(&self) -> &str {
        "Описание моего плагина"
    }

    async fn on_load(&mut self, ctx: &Context) -> Result<()> {
        // Инициализация плагина
        Ok(())
    }

    async fn on_unload(&mut self, _ctx: &Context) -> Result<()> {
        // Очистка ресурсов
        Ok(())
    }

    async fn on_event(&mut self, event: &Event, ctx: &Context) -> Result<()> {
        // Обработка событий
        Ok(())
    }
}

export_plugin!(MyPlugin);
```

## Регистрация команд

### Простая команда

```rust
use aniapi::CommandSpec;

fn commands(&self) -> Vec<CommandSpec> {
    vec![
        CommandSpec::new("hello", "Приветствует пользователя")
    ]
}
```

### Команда с правами

```rust
use serenity::model::Permissions;

fn commands(&self) -> Vec<CommandSpec> {
    vec![
        CommandSpec::new("admin", "Админская команда")
            .permission(Permissions::ADMINISTRATOR)
    ]
}
```

### Команда с кулдауном

```rust
fn commands(&self) -> Vec<CommandSpec> {
    vec![
        CommandSpec::new("spam", "Команда с кулдауном")
            .cooldown(5) // 5 секунд
    ]
}
```

## Обработка взаимодействий

### Обработка команд

```rust
async fn on_event(&mut self, event: &Event, ctx: &Context) -> Result<()> {
    match event {
        Event::Interaction(interaction) => {
            match interaction.as_ref() {
                Interaction::Command(command) => {
                    match command.data.name.as_str() {
                        "hello" => {
                            let response = CreateInteractionResponseMessage::new()
                                .content("Привет!");
                            let builder = CreateInteractionResponse::Message(response);
                            
                            if let Some(responder) = &ctx.responder {
                                responder.respond(
                                    command.id.get(), 
                                    &command.token, 
                                    builder
                                ).await?;
                            }
                        }
                        _ => {}
                    }
                }
                _ => {}
            }
        }
        _ => {}
    }
    Ok(())
}
```

### Обработка кнопок

```rust
use serenity::builder::{CreateButton, CreateActionRow};
use serenity::model::application::ComponentInteractionDataKind;

Event::Interaction(interaction) => {
    match interaction.as_ref() {
        Interaction::Component(component) => {
            if let ComponentInteractionDataKind::Button = &component.data.kind {
                if component.data.custom_id == "my_button" {
                    let response = CreateInteractionResponseMessage::new()
                        .content("Кнопка нажата!");
                    let builder = CreateInteractionResponse::Message(response);
                    
                    if let Some(responder) = &ctx.responder {
                        responder.respond(
                            component.id.get(), 
                            &component.token, 
                            builder
                        ).await?;
                    }
                }
            }
        }
        _ => {}
    }
}
```

## Работа с кэшем

```rust
async fn on_event(&mut self, event: &Event, ctx: &Context) -> Result<()> {
    // Сохранение значения
    ctx.cache.set("my_key", "my_value").await;
    
    // Получение значения
    if let Some(value) = ctx.cache.get("my_key").await {
        println!("Значение: {}", value);
    }
    
    // Удаление значения
    ctx.cache.remove("my_key").await;
    
    Ok(())
}
```

## Работа с конфигурацией

### Определение структуры конфигурации

```rust
use serde::Deserialize;

#[derive(Deserialize, Clone, Debug)]
pub struct Config {
    pub setting1: String,
    pub setting2: u64,
}

impl Default for Config {
    fn default() -> Self {
        Self {
            setting1: "default".to_string(),
            setting2: 100,
        }
    }
}
```

### Загрузка конфигурации

```rust
async fn on_load(&mut self, ctx: &Context) -> Result<()> {
    if let Some(config_mgr) = &ctx.config {
        match config_mgr.get_config(self.name()).await? {
            Some(toml_value) => {
                let toml_str = toml::to_string(&toml_value)?;
                let config: Config = toml::from_str(&toml_str)?;
                // Используйте config
            }
            None => {
                // Конфигурация не найдена, используйте значения по умолчанию
            }
        }
    }
    Ok(())
}
```

### Формат файла конфигурации

Создайте файл `config/MyPlugin.toml`:

```toml
setting1 = "value"
setting2 = 200
```

## Логирование

### Настройка логгера

```rust
async fn on_load(&mut self, ctx: &Context) -> Result<()> {
    if let Some(logger) = ctx.logger {
        log::set_logger(logger).ok();
        log::set_max_level(log::LevelFilter::Info);
    }
    Ok(())
}
```

### Использование логов

```rust
log::info!("Плагин загружен!");
log::warn!("Предупреждение: {}", message);
log::error!("Ошибка: {}", error);
log::debug!("Отладочная информация");
```

## Управление голосовыми каналами

```rust
async fn on_event(&mut self, event: &Event, ctx: &Context) -> Result<()> {
    if let Some(guild_mgr) = &ctx.guild {
        // Создание канала
        let channel_id = guild_mgr
            .create_voice_channel(
                guild_id, 
                "Новый канал", 
                Some(category_id)
            )
            .await?;
        
        // Перемещение пользователя
        guild_mgr
            .move_member(guild_id, user_id, channel_id)
            .await?;
        
        // Удаление канала
        guild_mgr.delete_channel(channel_id).await?;
    }
    Ok(())
}
```

## Обработка голосовых событий

```rust
async fn on_event(&mut self, event: &Event, ctx: &Context) -> Result<()> {
    match event {
        Event::VoiceStateUpdate(state) => {
            let user_id = state.user_id.get();
            let channel_id = state.channel_id.map(|id| id.get());
            
            if let Some(ch_id) = channel_id {
                // Пользователь присоединился к каналу
            } else {
                // Пользователь покинул все каналы
            }
        }
        _ => {}
    }
    Ok(())
}
```

## Отправка метрик в дашборд

```rust
async fn on_event(&mut self, event: &Event, ctx: &Context) -> Result<()> {
    if let Some(dashboard) = &ctx.dashboard {
        // Обновление задержки
        dashboard.update_latency(150.5).await;
        
        // Запись использования команды
        dashboard.record_command_usage("my_command").await;
    }
    Ok(())
}
```

## Обработка системных событий

```rust
async fn on_event(&mut self, event: &Event, ctx: &Context) -> Result<()> {
    match event {
        Event::System(system_event) => {
            match system_event {
                SystemEvent::Load => {
                    log::info!("Плагин загружен системой");
                }
                SystemEvent::Unload => {
                    log::info!("Плагин выгружается");
                }
                SystemEvent::Start => {
                    log::info!("Система запущена");
                }
                SystemEvent::Stop => {
                    log::info!("Система остановлена");
                }
            }
        }
        _ => {}
    }
    Ok(())
}
```

## Лучшие практики

1. **Всегда проверяйте наличие сервисов**: Не все сервисы могут быть доступны
   ```rust
   if let Some(responder) = &ctx.responder {
       // Используйте responder
   }
   ```

2. **Обрабатывайте ошибки**: Используйте `?` или `match` для обработки ошибок
   ```rust
   match some_operation().await {
       Ok(result) => { /* ... */ }
       Err(e) => {
           log::error!("Ошибка: {}", e);
           return Err(e);
       }
   }
   ```

3. **Используйте логирование**: Логи помогают отлаживать плагины
   ```rust
   log::info!("Важное событие произошло");
   ```

4. **Очищайте ресурсы в on_unload**: Освобождайте ресурсы при выгрузке
   ```rust
   async fn on_unload(&mut self, _ctx: &Context) -> Result<()> {
       // Очистка
       Ok(())
   }
   ```

5. **Используйте значения по умолчанию для конфигурации**: Всегда предоставляйте `Default` для конфигурации

## Сборка плагина

```bash
cd plugins/my_plugin
cargo build --release
```

Скомпилированная библиотека будет в `target/release/libmy_plugin.dll` (Windows) или `libmy_plugin.so` (Linux).

## Отладка

1. Используйте `log::debug!` для отладочной информации
2. Проверяйте логи системы
3. Убедитесь, что все зависимости правильно указаны в `Cargo.toml`
4. Проверьте, что `crate-type = ["cdylib"]` установлен

