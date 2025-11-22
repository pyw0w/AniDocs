# Разработка плагинов для AniCore

## Введение

AniCore использует современный Rust API на основе traits для разработки плагинов. Плагины реализуют trait `Plugin` и используют `PluginContext` для взаимодействия с системой.

## Создание плагина как отдельного проекта

Плагины могут быть созданы как полностью отдельные проекты. Вот как это сделать:

### 1. Создание нового проекта плагина

```bash
cargo new --lib my-plugin
cd my-plugin
```

### 2. Настройка Cargo.toml

```toml
[package]
name = "my-plugin"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]  # Важно: компилируем как динамическую библиотеку

[dependencies]
# Добавляем aniapi как зависимость
# Если aniapi опубликован в crates.io:
# aniapi = "0.1.0"

# Если aniapi локальный (рекомендуется для разработки):
aniapi = { path = "../path/to/aniapi" }

# Дополнительные зависимости
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
serenity = { workspace = true }  # Если нужен доступ к типам Serenity
```

### 3. Базовая структура плагина

```rust
use aniapi::{Plugin, PluginContext};
use std::error::Error;

pub struct MyPlugin {
    name: String,
    version: String,
}

impl MyPlugin {
    pub fn new() -> Self {
        Self {
            name: "My Plugin".to_string(),
            version: "1.0.0".to_string(),
        }
    }
}

impl Plugin for MyPlugin {
    fn name(&self) -> &str {
        &self.name
    }
    
    fn version(&self) -> &str {
        &self.version
    }
    
    fn initialize(&mut self, ctx: PluginContext) -> Result<(), Box<dyn Error>> {
        aniapi::logger::PluginLogger::info(&format!("Плагин {} v{} инициализируется", self.name, self.version));
        
        // Регистрируем обработчики событий, функции и команды здесь
        
        Ok(())
    }
    
    fn shutdown(&mut self) -> Result<(), Box<dyn Error>> {
        aniapi::logger::PluginLogger::info(&format!("Плагин {} v{} завершает работу", self.name, self.version));
        Ok(())
    }
}

// Экспортируем функцию для инициализации плагина
#[no_mangle]
pub extern "C" fn init_plugin() -> *mut std::ffi::c_void {
    let plugin: Box<dyn aniapi::Plugin> = Box::new(MyPlugin::new());
    // Оборачиваем Box<dyn Plugin> в еще один Box, чтобы получить *mut Box<dyn Plugin>
    Box::into_raw(Box::new(plugin)) as *mut std::ffi::c_void
}
```

### 4. Регистрация обработчиков событий

#### Обработчик сообщений

```rust
use aniapi::{MessageEvent, PluginContext};
use serde_json::Value;

fn initialize(&mut self, ctx: PluginContext) -> Result<(), Box<dyn Error>> {
    // Регистрируем обработчик сообщений
    ctx.register_message_handler(Box::new(|event: MessageEvent| -> Result<Option<Value>, Box<dyn Error>> {
        aniapi::logger::PluginLogger::debug(&format!("Получено сообщение: {}", event.content));
        
        if event.content.trim() == "!hello" {
            let response = serde_json::json!({
                "content": "Привет!"
            });
            Ok(Some(response))
        } else {
            Ok(None)  // Плагин не обрабатывает это сообщение
        }
    }));
    
    Ok(())
}
```

#### Обработчик взаимодействий (slash команд)

```rust
use aniapi::{InteractionEvent, PluginContext};
use serde_json::Value;

fn initialize(&mut self, ctx: PluginContext) -> Result<(), Box<dyn Error>> {
    // Регистрируем обработчик взаимодействий
    ctx.register_interaction_handler(Box::new(|event: InteractionEvent| -> Result<Option<Value>, Box<dyn Error>> {
        aniapi::logger::PluginLogger::debug(&format!("Обработка команды: {}", event.command_name));
        
        if event.command_name == "hello" {
            let response = serde_json::json!({
                "content": "Привет из плагина!",
                "ephemeral": false
            });
            Ok(Some(response))
        } else {
            Ok(None)
        }
    }));
    
    Ok(())
}
```

#### Обработчик событий голосовых каналов

```rust
use aniapi::{VoiceStateUpdateEvent, PluginContext};

fn initialize(&mut self, ctx: PluginContext) -> Result<(), Box<dyn Error>> {
    // Регистрируем обработчик событий голосовых каналов
    ctx.register_voice_state_handler(Box::new(|event: VoiceStateUpdateEvent| -> Result<Option<Value>, Box<dyn Error>> {
        aniapi::logger::PluginLogger::info(&format!(
            "Пользователь {} изменил голосовое состояние в гильдии {}",
            event.user_id, event.guild_id
        ));
        
        // Обработка события...
        
        Ok(None)  // Voice state handlers обычно не возвращают ответ
    }));
    
    Ok(())
}
```

### 5. Регистрация команд

```rust
use aniapi::{CommandBuilder, Commands, CommandOptionBuilder, CommandOptionType, PluginContext};

fn initialize(&mut self, ctx: PluginContext) -> Result<(), Box<dyn Error>> {
    // Создаем команды
    let hello_command = CommandBuilder::new("hello", "Приветствие от плагина")
        .build();
    
    let greet_command = CommandBuilder::new("greet", "Поприветствовать пользователя")
        .add_option(
            CommandOptionBuilder::new(CommandOptionType::User, "user", "Пользователь для приветствия")
                .required(true)
                .build()
        )
        .build();
    
    // Регистрируем команды
    let commands = Commands::new()
        .add(hello_command)
        .add(greet_command);
    
    ctx.register_commands(commands)?;
    
    Ok(())
}
```

### 6. Регистрация функций для межплагинного взаимодействия

```rust
use aniapi::{PluginContext, PluginFunction};
use serde_json::Value;
use std::sync::Arc;

fn initialize(&mut self, ctx: PluginContext) -> Result<(), Box<dyn Error>> {
    // Регистрируем функцию, которую могут вызывать другие плагины
    let my_function: PluginFunction = Arc::new(|args: Value| -> Result<Value, Box<dyn Error>> {
        aniapi::logger::PluginLogger::info("Вызвана функция my_function");
        
        // Обработка аргументов
        let input = args.get("input")
            .and_then(|v| v.as_str())
            .ok_or("Отсутствует поле 'input'")?;
        
        // Возвращаем результат
        Ok(serde_json::json!({
            "result": format!("Обработано: {}", input)
        }))
    });
    
    ctx.register_function("my_function", my_function)?;
    
    Ok(())
}
```

### 7. Вызов функций из других плагинов

```rust
use aniapi::PluginContext;
use serde_json::Value;

fn initialize(&mut self, ctx: PluginContext) -> Result<(), Box<dyn Error>> {
    // Получаем функцию из другого плагина
    if let Some(func) = ctx.get_function("other_plugin::some_function") {
        let args = serde_json::json!({
            "input": "test"
        });
        
        match func(args) {
            Ok(result) => {
                aniapi::logger::PluginLogger::info(&format!("Результат: {:?}", result));
            }
            Err(e) => {
                aniapi::logger::PluginLogger::error(&format!("Ошибка вызова функции: {}", e));
            }
        }
    }
    
    Ok(())
}
```

### 8. Работа с конфигурацией

```rust
use aniapi::PluginContext;
use serde_json::Value;

fn initialize(&mut self, ctx: PluginContext) -> Result<(), Box<dyn Error>> {
    // Получаем конфигурацию плагина
    if let Some(config) = ctx.get_config()? {
        aniapi::logger::PluginLogger::info(&format!("Загружена конфигурация: {:?}", config));
        
        // Используем конфигурацию...
    } else {
        aniapi::logger::PluginLogger::info("Конфигурация не найдена, используем значения по умолчанию");
    }
    
    // Сохраняем конфигурацию
    let new_config = serde_json::json!({
        "setting1": "value1",
        "setting2": 42
    });
    
    ctx.save_config(&new_config)?;
    
    Ok(())
}
```

### 9. Доступ к HTTP клиенту Discord

```rust
use aniapi::PluginContext;
use serenity::all::Http;

fn initialize(&mut self, ctx: PluginContext) -> Result<(), Box<dyn Error>> {
    // Получаем HTTP клиент
    let http = ctx.http_client();
    
    // Используем HTTP клиент для выполнения запросов к Discord API
    // Например, создание канала, отправка сообщений и т.д.
    
    Ok(())
}
```

### 10. Полный пример плагина

```rust
use aniapi::{Plugin, PluginContext, MessageEvent, InteractionEvent, CommandBuilder, Commands};
use aniapi::logger::PluginLogger;
use serde_json::Value;
use std::error::Error;
use std::sync::{Arc, Mutex};
use std::collections::HashMap;

pub struct MyPlugin {
    name: String,
    version: String,
    // Пример внутреннего состояния
    message_count: Arc<Mutex<u64>>,
}

impl MyPlugin {
    pub fn new() -> Self {
        Self {
            name: "My Plugin".to_string(),
            version: "1.0.0".to_string(),
            message_count: Arc::new(Mutex::new(0)),
        }
    }
}

impl Plugin for MyPlugin {
    fn name(&self) -> &str {
        &self.name
    }
    
    fn version(&self) -> &str {
        &self.version
    }
    
    fn initialize(&mut self, ctx: PluginContext) -> Result<(), Box<dyn Error>> {
        PluginLogger::info(&format!("Плагин {} v{} инициализируется", self.name, self.version));
        
        // Регистрируем обработчик сообщений
        let message_count = Arc::clone(&self.message_count);
        ctx.register_message_handler(Box::new(move |event: MessageEvent| -> Result<Option<Value>, Box<dyn Error>> {
            if event.content.trim() == "!hello" {
                let mut count = message_count.lock().unwrap();
                *count += 1;
                
                let response = serde_json::json!({
                    "content": format!("Привет! Это сообщение номер {}", count)
                });
                Ok(Some(response))
            } else {
                Ok(None)
            }
        }));
        
        // Регистрируем обработчик взаимодействий
        ctx.register_interaction_handler(Box::new(|event: InteractionEvent| -> Result<Option<Value>, Box<dyn Error>> {
            if event.command_name == "hello" {
                let response = serde_json::json!({
                    "content": "Привет из плагина!",
                    "ephemeral": false
                });
                Ok(Some(response))
            } else {
                Ok(None)
            }
        }));
        
        // Регистрируем команды
        let hello_command = CommandBuilder::new("hello", "Приветствие от плагина")
            .build();
        
        let commands = Commands::new().add(hello_command);
        ctx.register_commands(commands)?;
        
        PluginLogger::info(&format!("Плагин {} v{} успешно инициализирован", self.name, self.version));
        Ok(())
    }
    
    fn shutdown(&mut self) -> Result<(), Box<dyn Error>> {
        PluginLogger::info(&format!("Плагин {} v{} завершает работу", self.name, self.version));
        Ok(())
    }
}

// Экспортируем функцию для инициализации плагина
#[no_mangle]
pub extern "C" fn init_plugin() -> *mut std::ffi::c_void {
    let plugin: Box<dyn aniapi::Plugin> = Box::new(MyPlugin::new());
    // Оборачиваем Box<dyn Plugin> в еще один Box, чтобы получить *mut Box<dyn Plugin>
    Box::into_raw(Box::new(plugin)) as *mut std::ffi::c_void
}
```

### 11. Компиляция плагина

```bash
# Debug версия
cargo build

# Release версия
cargo build --release
```

Скомпилированная библиотека будет находиться в:
- Debug: `target/debug/libmy_plugin.dll` (Windows) или `libmy_plugin.so` (Linux)
- Release: `target/release/libmy_plugin.dll` (Windows) или `libmy_plugin.so` (Linux)

**Важно**: Файл должен содержать "plugin" в имени, чтобы AniCore его распознал.

### 12. Размещение плагина

Скопируйте скомпилированную библиотеку в директорию плагинов AniCore (по умолчанию `target/debug` или `target/release`).

## API Reference

### Plugin Trait

```rust
pub trait Plugin: Send + Sync {
    fn name(&self) -> &str;
    fn version(&self) -> &str;
    fn initialize(&mut self, ctx: PluginContext) -> Result<(), Box<dyn Error>>;
    fn shutdown(&mut self) -> Result<(), Box<dyn Error>>;
}
```

### PluginContext

`PluginContext` предоставляет доступ к функциям системы:

- `register_function(name, func)` - регистрация функции для межплагинного взаимодействия
- `get_function(name)` - получение функции из другого плагина
- `call_function(name, args)` - вызов функции по имени
- `register_message_handler(handler)` - регистрация обработчика сообщений
- `register_interaction_handler(handler)` - регистрация обработчика взаимодействий
- `register_voice_state_handler(handler)` - регистрация обработчика голосовых событий
- `get_config()` - получение конфигурации плагина
- `save_config(config)` - сохранение конфигурации плагина
- `http_client()` - доступ к HTTP клиенту Discord
- `register_commands(commands)` - регистрация команд

### Типы событий

- `MessageEvent` - событие сообщения
- `InteractionEvent` - событие взаимодействия (slash команда)
- `VoiceStateUpdateEvent` - событие обновления голосового состояния

## Важные замечания

### Инициализация логирования

**Как это работает:**

1. **Основной процесс (AniCore)** инициализирует `tracing` subscriber при запуске
2. **Плагины** используют `PluginLogger` из `aniapi`, который вызывает `tracing::info!()` и т.д.
3. `tracing` использует **глобальный subscriber** через `OnceCell`/`Lazy`
4. Когда плагин (даже отдельный проект) вызывает `PluginLogger::info()`, он использует уже инициализированный subscriber из основного процесса
5. Это работает потому что `tracing` - это глобальное состояние, доступное всем модулям

**Важно:**
- `tracing` должен быть инициализирован в основном процессе (AniCore) - это делается автоматически
- Плагины используют глобальный subscriber, который уже настроен
- Плагины **не должны** инициализировать `tracing` самостоятельно

### Зависимости

- Плагины могут использовать `aniapi` как обычную зависимость
- Все типы из `aniapi` доступны в плагинах
- Плагины могут иметь свои собственные зависимости
- Рекомендуется использовать те же версии зависимостей, что и в основном проекте

### Безопасность

- Все обработчики событий должны быть thread-safe
- Используйте `Arc` и `Mutex`/`RwLock` для разделяемого состояния
- Всегда обрабатывайте ошибки корректно

### Производительность

- Используйте `PluginLogger::debug()` для отладочных сообщений
- В production используйте уровень логирования `info` или выше
- Избегайте избыточного логирования в горячих путях
- Обработчики событий должны быть быстрыми

### Межплагинное взаимодействие

- Функции регистрируются с префиксом имени плагина: `plugin_name::function_name`
- Для вызова функции из другого плагина используйте полное имя: `other_plugin::function_name`
- Функции должны быть thread-safe и не блокирующими

## Примеры использования логирования

```rust
use aniapi::logger::PluginLogger;

// Информационное сообщение
PluginLogger::info("Плагин успешно загружен");

// Предупреждение
PluginLogger::warn("Функция устарела, используйте новую");

// Ошибка
PluginLogger::error(&format!("Ошибка обработки: {}", error));

// Отладка
PluginLogger::debug(&format!("Значение переменной: {:?}", value));
```

## Примеры плагинов

Примеры работающих плагинов можно найти в директории `plugins/`:
- `utils_plugin` - простой плагин с командами ping
- `voicetemp_plugin` - плагин для управления временными голосовыми каналами

## Публикация плагина

Если вы хотите опубликовать плагин:

1. Убедитесь, что `aniapi` доступен (опубликован в crates.io или через git)
2. Укажите правильную зависимость в `Cargo.toml`
3. Документируйте использование логирования
4. Предоставьте примеры использования
5. Укажите совместимые версии AniCore
