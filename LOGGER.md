# Система логирования AniCore

AniCore использует централизованную систему логирования на основе `tracing`, которая доступна для всех модулей и плагинов.

## Использование в основном коде

### Инициализация

Логгер инициализируется автоматически при запуске приложения:

```rust
use anicore::init_logger;

init_logger();
```

### Использование макросов

```rust
use tracing::{info, warn, error, debug, trace};

tracing::info!("Информационное сообщение");
tracing::warn!("Предупреждение");
tracing::error!("Ошибка");
tracing::debug!("Отладочное сообщение");
tracing::trace!("Трассировочное сообщение");
```

### Форматирование

```rust
tracing::info!("Плагин {} версии {} загружен", name, version);
tracing::error!("Ошибка загрузки плагина: {}", error);
```

## Использование в плагинах

### Через Rust API (рекомендуется)

Если плагин написан на Rust и использует `aniapi` как зависимость:

**Важно**: Плагин должен иметь `aniapi` в зависимостях в `Cargo.toml`:

```toml
[dependencies]
aniapi = { path = "../path/to/aniapi" }  # или из crates.io
```

Затем в коде плагина:

```rust
use aniapi::logger::PluginLogger;

// В функции плагина
PluginLogger::info("Плагин успешно загружен");
PluginLogger::warn("Предупреждение: функция устарела");
PluginLogger::error("Критическая ошибка");
PluginLogger::debug("Отладочная информация");
```

**Как это работает**: 
- Плагин компилируется с зависимостью на `aniapi`
- `PluginLogger` использует глобальный `tracing` subscriber
- Subscriber инициализируется в основном процессе (AniCore)
- Плагины просто используют уже настроенный subscriber
- Это работает даже если плагин - отдельный проект!

### Через FFI (для C/C++ плагинов или динамического связывания)

Для плагинов, написанных на C/C++ или когда нужно динамическое связывание:

**Важно**: FFI функции должны быть доступны через динамическую библиотеку aniapi или статически связаны.

```c
extern void aniapi_log_info(const char* message);
extern void aniapi_log_warn(const char* message);
extern void aniapi_log_error(const char* message);
extern void aniapi_log_debug(const char* message);

// Использование
aniapi_log_info("Плагин загружен");
aniapi_log_warn("Предупреждение");
```

Или в Rust через FFI:

```rust
use std::ffi::CString;

extern "C" {
    fn aniapi_log_info(msg: *const std::os::raw::c_char);
}

let msg = CString::new("Сообщение").unwrap();
unsafe { aniapi_log_info(msg.as_ptr()); }
```

### Пример использования в Rust плагине

```rust
use aniapi::{BotPluginVTable, PluginVTablePtr, CommandBuilder, Commands};
use aniapi::logger::PluginLogger;
use std::ffi::{CStr, CString};
use std::os::raw::c_char;

unsafe extern "C" fn plugin_on_message(content: *const c_char, _channel_id: u64) -> *const c_char {
    if content.is_null() {
        PluginLogger::error("Получен null указатель в on_message");
        return std::ptr::null();
    }
    
    let content_str = match CStr::from_ptr(content).to_str() {
        Ok(s) => s,
        Err(e) => {
            PluginLogger::error(&format!("Ошибка преобразования строки: {}", e));
            return std::ptr::null();
        }
    };
    
    PluginLogger::debug(&format!("Получено сообщение: {}", content_str));
    
    if content_str.trim() == "!ping" {
        PluginLogger::info("Обработка команды ping");
        // ... обработка команды
    }
    
    std::ptr::null()
}
```

## Настройка уровня логирования

Уровень логирования настраивается через переменную окружения `RUST_LOG`:

```bash
# Только ошибки
export RUST_LOG=error

# Информация и выше
export RUST_LOG=info

# Все сообщения включая debug
export RUST_LOG=debug

# Только для конкретного модуля
export RUST_LOG=anicore::plugins=debug
```

## Формат логов

Логи выводятся в компактном формате:

```
14:47 info AniHub запущен на платформе: Windows
14:47 info Расширение файлов плагинов: .dll
14:47 info [Plugin] Плагин успешно загружен
14:47 warn [Plugin] Предупреждение: функция устарела
14:47 error [Plugin] Критическая ошибка
```

## Преимущества

1. **Единый формат**: Все логи имеют одинаковый формат
2. **Централизованное управление**: Уровень логирования настраивается в одном месте
3. **Производительность**: Используется эффективная библиотека `tracing`
4. **Гибкость**: Поддержка структурированного логирования
5. **Доступность**: API доступен для всех модулей и плагинов

