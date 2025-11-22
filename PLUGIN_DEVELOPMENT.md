# Разработка плагинов для AniCore

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
```

### 3. Использование логирования в плагине

#### Вариант 1: Через Rust API (рекомендуется)

Если плагин использует `aniapi` как зависимость, можно использовать `PluginLogger`:

```rust
use aniapi::{BotPluginVTable, PluginVTablePtr, CommandBuilder, Commands};
use aniapi::logger::PluginLogger;
use std::ffi::{CStr, CString};
use std::os::raw::c_char;

unsafe extern "C" fn plugin_on_message(content: *const c_char, _channel_id: u64) -> *const c_char {
    PluginLogger::info("Обработка сообщения");
    
    if content.is_null() {
        PluginLogger::error("Получен null указатель");
        return std::ptr::null();
    }
    
    let content_str = match CStr::from_ptr(content).to_str() {
        Ok(s) => s,
        Err(e) => {
            PluginLogger::error(&format!("Ошибка преобразования: {}", e));
            return std::ptr::null();
        }
    };
    
    PluginLogger::debug(&format!("Получено сообщение: {}", content_str));
    
    // ... обработка сообщения
    
    std::ptr::null()
}
```

#### Вариант 2: Через FFI функции

Если плагин написан на C/C++ или не может использовать Rust зависимости:

```rust
use std::ffi::CString;

extern "C" {
    fn aniapi_log_info(msg: *const std::os::raw::c_char);
    fn aniapi_log_warn(msg: *const std::os::raw::c_char);
    fn aniapi_log_error(msg: *const std::os::raw::c_char);
    fn aniapi_log_debug(msg: *const std::os::raw::c_char);
}

unsafe extern "C" fn plugin_on_message(content: *const std::os::raw::c_char, _channel_id: u64) -> *const std::os::raw::c_char {
    let msg = CString::new("Обработка сообщения").unwrap();
    aniapi_log_info(msg.as_ptr());
    
    // ... обработка
    
    std::ptr::null()
}
```

**Важно**: При использовании FFI функций плагин должен быть скомпилирован с линковкой к библиотеке aniapi или функции должны быть доступны через динамическое связывание.

### 4. Полный пример плагина

```rust
use aniapi::{BotPluginVTable, PluginVTablePtr, CommandBuilder, Commands};
use aniapi::logger::PluginLogger;
use std::ffi::{CStr, CString};
use std::os::raw::c_char;

const PLUGIN_NAME: &str = "My Plugin";
const PLUGIN_VERSION: &str = "1.0.0";

unsafe extern "C" fn plugin_name() -> *const c_char {
    c"My Plugin".as_ptr()
}

unsafe extern "C" fn plugin_version() -> *const c_char {
    c"1.0.0".as_ptr()
}

unsafe extern "C" fn plugin_on_message(content: *const c_char, _channel_id: u64) -> *const c_char {
    PluginLogger::info("Плагин обрабатывает сообщение");
    
    if content.is_null() {
        return std::ptr::null();
    }
    
    let content_str = match CStr::from_ptr(content).to_str() {
        Ok(s) => s,
        Err(_) => return std::ptr::null(),
    };
    
    if content_str.trim() == "!hello" {
        PluginLogger::info("Обработка команды !hello");
        // ... обработка команды
    }
    
    std::ptr::null()
}

unsafe extern "C" fn plugin_get_commands() -> *const c_char {
    PluginLogger::debug("Получение списка команд");
    
    let command = CommandBuilder::new("hello", "Приветствие")
        .build();
    
    let commands = Commands::new().add(command);
    let json = match commands.to_json() {
        Ok(j) => j,
        Err(_) => return std::ptr::null(),
    };
    
    CString::new(json).unwrap().into_raw()
}

unsafe extern "C" fn plugin_on_interaction(
    _command_name: *const c_char,
    _interaction_json: *const c_char,
) -> *const c_char {
    PluginLogger::info("Обработка взаимодействия");
    std::ptr::null()
}

static PLUGIN_VTABLE: BotPluginVTable = BotPluginVTable {
    name: plugin_name,
    version: plugin_version,
    on_message: plugin_on_message,
    get_commands: plugin_get_commands,
    on_interaction: plugin_on_interaction,
};

#[no_mangle]
pub extern "C" fn _get_vtable() -> PluginVTablePtr {
    PluginLogger::info(&format!("Плагин {} v{} загружается", PLUGIN_NAME, PLUGIN_VERSION));
    PluginVTablePtr::new(&PLUGIN_VTABLE)
}
```

### 5. Компиляция плагина

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

### 6. Размещение плагина

Скопируйте скомпилированную библиотеку в директорию плагинов AniCore (по умолчанию `target/debug` или `target/release`).

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
- Даже если плагин - полностью отдельный проект, логирование будет работать, если:
  - Плагин использует `aniapi` как зависимость
  - Плагин компилируется с теми же версиями зависимостей (особенно `tracing`)

### Зависимости

- Плагины могут использовать `aniapi` как обычную зависимость
- Все типы из `aniapi` доступны в плагинах
- Плагины могут иметь свои собственные зависимости

### Безопасность

- FFI функции помечены как `unsafe` - используйте их осторожно
- Всегда проверяйте указатели на `null` перед использованием
- Используйте `CString` для безопасной работы с C-строками

### Производительность

- Используйте `PluginLogger::debug()` для отладочных сообщений
- В production используйте уровень логирования `info` или выше
- Избегайте избыточного логирования в горячих путях

## Примеры использования логирования

```rust
// Информационное сообщение
PluginLogger::info("Плагин успешно загружен");

// Предупреждение
PluginLogger::warn("Функция устарела, используйте новую");

// Ошибка
PluginLogger::error(&format!("Ошибка обработки: {}", error));

// Отладка
PluginLogger::debug(&format!("Значение переменной: {:?}", value));
```

## Публикация плагина

Если вы хотите опубликовать плагин:

1. Убедитесь, что `aniapi` доступен (опубликован в crates.io или через git)
2. Укажите правильную зависимость в `Cargo.toml`
3. Документируйте использование логирования
4. Предоставьте примеры использования

