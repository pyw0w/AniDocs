# Примеры использования AniAPI

## Пример 1: Простой плагин с командой

```rust
use aniapi::{export_plugin, CommandSpec, Context, Event, Plugin, Result};
use async_trait::async_trait;
use serenity::builder::{
    CreateInteractionResponse, CreateInteractionResponseMessage,
};
use serenity::model::application::Interaction;

#[derive(Default)]
pub struct HelloPlugin;

#[async_trait]
impl Plugin for HelloPlugin {
    fn name(&self) -> &str {
        "HelloPlugin"
    }

    fn commands(&self) -> Vec<CommandSpec> {
        vec![CommandSpec::new("hello", "Приветствует пользователя")]
    }

    async fn on_load(&mut self, ctx: &Context) -> Result<()> {
        if let Some(logger) = ctx.logger {
            log::set_logger(logger).ok();
            log::set_max_level(log::LevelFilter::Info);
        }
        log::info!("HelloPlugin загружен!");
        Ok(())
    }

    async fn on_unload(&mut self, _ctx: &Context) -> Result<()> {
        log::info!("HelloPlugin выгружен!");
        Ok(())
    }

    async fn on_event(&mut self, event: &Event, ctx: &Context) -> Result<()> {
        match event {
            Event::Interaction(interaction) => {
                match interaction.as_ref() {
                    Interaction::Command(command) => {
                        if command.data.name == "hello" {
                            let response = CreateInteractionResponseMessage::new()
                                .content("Привет! 👋");
                            let builder = CreateInteractionResponse::Message(response);
                            
                            if let Some(responder) = &ctx.responder {
                                responder.respond(
                                    command.id.get(),
                                    &command.token,
                                    builder,
                                )
                                .await?;
                            }
                        }
                    }
                    _ => {}
                }
            }
            _ => {}
        }
        Ok(())
    }
}

export_plugin!(HelloPlugin);
```

## Пример 2: Плагин с кэшем

```rust
use aniapi::{export_plugin, CommandSpec, Context, Event, Plugin, Result};
use async_trait::async_trait;
use serenity::builder::{
    CreateInteractionResponse, CreateInteractionResponseMessage,
};
use serenity::model::application::Interaction;

#[derive(Default)]
pub struct CounterPlugin {
    counter_key: String,
}

#[async_trait]
impl Plugin for CounterPlugin {
    fn name(&self) -> &str {
        "CounterPlugin"
    }

    fn commands(&self) -> Vec<CommandSpec> {
        vec![
            CommandSpec::new("increment", "Увеличить счётчик"),
            CommandSpec::new("get", "Получить значение счётчика"),
        ]
    }

    async fn on_load(&mut self, ctx: &Context) -> Result<()> {
        self.counter_key = format!("{}:counter", self.name());
        
        // Инициализация счётчика, если его нет
        if ctx.cache.get(&self.counter_key).await.is_none() {
            ctx.cache.set(&self.counter_key, "0").await;
        }
        
        Ok(())
    }

    async fn on_unload(&mut self, _ctx: &Context) -> Result<()> {
        Ok(())
    }

    async fn on_event(&mut self, event: &Event, ctx: &Context) -> Result<()> {
        match event {
            Event::Interaction(interaction) => {
                match interaction.as_ref() {
                    Interaction::Command(command) => {
                        match command.data.name.as_str() {
                            "increment" => {
                                let current = ctx
                                    .cache
                                    .get(&self.counter_key)
                                    .await
                                    .and_then(|v| v.parse::<u64>().ok())
                                    .unwrap_or(0);
                                
                                let new_value = current + 1;
                                ctx.cache
                                    .set(&self.counter_key, &new_value.to_string())
                                    .await;
                                
                                let response = CreateInteractionResponseMessage::new()
                                    .content(&format!("Счётчик: {}", new_value));
                                let builder = CreateInteractionResponse::Message(response);
                                
                                if let Some(responder) = &ctx.responder {
                                    responder.respond(
                                        command.id.get(),
                                        &command.token,
                                        builder,
                                    )
                                    .await?;
                                }
                            }
                            "get" => {
                                let value = ctx
                                    .cache
                                    .get(&self.counter_key)
                                    .await
                                    .unwrap_or_else(|| "0".to_string());
                                
                                let response = CreateInteractionResponseMessage::new()
                                    .content(&format!("Текущее значение: {}", value));
                                let builder = CreateInteractionResponse::Message(response);
                                
                                if let Some(responder) = &ctx.responder {
                                    responder.respond(
                                        command.id.get(),
                                        &command.token,
                                        builder,
                                    )
                                    .await?;
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
}

export_plugin!(CounterPlugin);
```

## Пример 3: Плагин с конфигурацией

```rust
use aniapi::{export_plugin, Context, Event, Plugin, Result};
use async_trait::async_trait;
use serde::Deserialize;
use std::sync::Arc;
use tokio::sync::Mutex;

#[derive(Deserialize, Clone, Debug)]
pub struct Config {
    pub welcome_message: String,
    pub channel_id: u64,
}

impl Default for Config {
    fn default() -> Self {
        Self {
            welcome_message: "Добро пожаловать!".to_string(),
            channel_id: 0,
        }
    }
}

#[derive(Default)]
pub struct WelcomePlugin {
    config: Arc<Mutex<Option<Config>>>,
}

#[async_trait]
impl Plugin for WelcomePlugin {
    fn name(&self) -> &str {
        "WelcomePlugin"
    }

    async fn on_load(&mut self, ctx: &Context) -> Result<()> {
        if let Some(logger) = ctx.logger {
            log::set_logger(logger).ok();
            log::set_max_level(log::LevelFilter::Info);
        }

        if let Some(config_mgr) = &ctx.config {
            match config_mgr.get_config(self.name()).await? {
                Some(toml_value) => {
                    let toml_str = toml::to_string(&toml_value)?;
                    match toml::from_str::<Config>(&toml_str) {
                        Ok(config) => {
                            let mut stored = self.config.lock().await;
                            *stored = Some(config.clone());
                            log::info!("Конфигурация загружена: {:?}", config);
                        }
                        Err(e) => {
                            log::warn!("Ошибка парсинга конфигурации: {}. Используются значения по умолчанию.", e);
                            let mut stored = self.config.lock().await;
                            *stored = Some(Config::default());
                        }
                    }
                }
                None => {
                    log::warn!("Конфигурация не найдена. Используются значения по умолчанию.");
                    let mut stored = self.config.lock().await;
                    *stored = Some(Config::default());
                }
            }
        }

        Ok(())
    }

    async fn on_unload(&mut self, _ctx: &Context) -> Result<()> {
        Ok(())
    }

    async fn on_event(&mut self, event: &Event, ctx: &Context) -> Result<()> {
        let config = {
            let stored = self.config.lock().await;
            stored.clone().unwrap_or_else(Config::default)
        };

        match event {
            Event::Message(author, content) => {
                if content == "!welcome" {
                    log::info!("Отправка приветствия для {}", author);
                    // Здесь можно отправить сообщение в канал
                }
            }
            _ => {}
        }

        Ok(())
    }
}

export_plugin!(WelcomePlugin);
```

## Пример 4: Плагин с кнопками

```rust
use aniapi::{export_plugin, CommandSpec, Context, Event, Plugin, Result};
use async_trait::async_trait;
use serenity::builder::{
    CreateActionRow, CreateButton, CreateInteractionResponse,
    CreateInteractionResponseMessage,
};
use serenity::model::application::{ComponentInteractionDataKind, Interaction};

#[derive(Default)]
pub struct ButtonPlugin;

#[async_trait]
impl Plugin for ButtonPlugin {
    fn name(&self) -> &str {
        "ButtonPlugin"
    }

    fn commands(&self) -> Vec<CommandSpec> {
        vec![CommandSpec::new("show_buttons", "Показать пример кнопок")]
    }

    async fn on_load(&mut self, _ctx: &Context) -> Result<()> {
        Ok(())
    }

    async fn on_unload(&mut self, _ctx: &Context) -> Result<()> {
        Ok(())
    }

    async fn on_event(&mut self, event: &Event, ctx: &Context) -> Result<()> {
        match event {
            Event::Interaction(interaction) => {
                match interaction.as_ref() {
                    Interaction::Command(command) => {
                        if command.data.name == "show_buttons" {
                            let button1 = CreateButton::new("btn_1")
                                .label("Кнопка 1")
                                .style(serenity::model::application::component::ButtonStyle::Primary);
                            
                            let button2 = CreateButton::new("btn_2")
                                .label("Кнопка 2")
                                .style(serenity::model::application::component::ButtonStyle::Success);
                            
                            let row = CreateActionRow::Buttons(vec![button1, button2]);
                            
                            let response = CreateInteractionResponseMessage::new()
                                .content("Выберите кнопку:")
                                .components(vec![row]);
                            
                            let builder = CreateInteractionResponse::Message(response);
                            
                            if let Some(responder) = &ctx.responder {
                                responder.respond(
                                    command.id.get(),
                                    &command.token,
                                    builder,
                                )
                                .await?;
                            }
                        }
                    }
                    Interaction::Component(component) => {
                        if let ComponentInteractionDataKind::Button = &component.data.kind {
                            let content = match component.data.custom_id.as_str() {
                                "btn_1" => "Вы нажали кнопку 1!",
                                "btn_2" => "Вы нажали кнопку 2!",
                                _ => "Неизвестная кнопка",
                            };
                            
                            let response = CreateInteractionResponseMessage::new()
                                .content(content);
                            let builder = CreateInteractionResponse::Message(response);
                            
                            if let Some(responder) = &ctx.responder {
                                responder.respond(
                                    component.id.get(),
                                    &component.token,
                                    builder,
                                )
                                .await?;
                            }
                        }
                    }
                    _ => {}
                }
            }
            _ => {}
        }
        Ok(())
    }
}

export_plugin!(ButtonPlugin);
```

## Пример 5: Плагин с метриками

```rust
use aniapi::{export_plugin, CommandSpec, Context, Event, Plugin, Result};
use async_trait::async_trait;
use serenity::builder::{
    CreateInteractionResponse, CreateInteractionResponseMessage,
};
use serenity::model::application::Interaction;

#[derive(Default)]
pub struct MetricsPlugin;

#[async_trait]
impl Plugin for MetricsPlugin {
    fn name(&self) -> &str {
        "MetricsPlugin"
    }

    fn commands(&self) -> Vec<CommandSpec> {
        vec![CommandSpec::new("stats", "Показать статистику")]
    }

    async fn on_load(&mut self, _ctx: &Context) -> Result<()> {
        Ok(())
    }

    async fn on_unload(&mut self, _ctx: &Context) -> Result<()> {
        Ok(())
    }

    async fn on_event(&mut self, event: &Event, ctx: &Context) -> Result<()> {
        match event {
            Event::Interaction(interaction) => {
                match interaction.as_ref() {
                    Interaction::Command(command) => {
                        if command.data.name == "stats" {
                            // Записываем использование команды
                            if let Some(dashboard) = &ctx.dashboard {
                                dashboard.record_command_usage("stats").await;
                            }
                            
                            let response = CreateInteractionResponseMessage::new()
                                .content("Статистика отправлена в дашборд!");
                            let builder = CreateInteractionResponse::Message(response);
                            
                            if let Some(responder) = &ctx.responder {
                                responder.respond(
                                    command.id.get(),
                                    &command.token,
                                    builder,
                                )
                                .await?;
                            }
                        }
                    }
                    _ => {}
                }
            }
            _ => {}
        }
        Ok(())
    }
}

export_plugin!(MetricsPlugin);
```

## Пример 6: Комплексный плагин

Полный пример плагина, использующего все возможности API, можно найти в:
- `plugins/utils_plugin/src/lib.rs` - пример с командами и кнопками
- `plugins/voicetemp_plugin/src/lib.rs` - пример с голосовыми каналами и конфигурацией

