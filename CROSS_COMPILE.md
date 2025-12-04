# Кросс-компиляция для Linux из Windows

## Быстрый старт

Для сборки проекта под Linux из Windows используйте:

```powershell
.\build.ps1 release -Target linux
```

## Требования

1. **Rust и rustup** должны быть установлены
2. **Target для Linux** будет установлен автоматически при первом запуске

## Варианты сборки

### Стандартная сборка (GNU toolchain)

```powershell
# Установка target (выполняется автоматически)
rustup target add x86_64-unknown-linux-gnu

# Сборка
.\build.ps1 release -Target linux
```

**Примечание:** Для этого варианта может потребоваться установка линкера для Linux. Если сборка не работает, используйте вариант с musl (см. ниже).

### Статическая сборка (musl - рекомендуется)

Для статической сборки без зависимостей от системных библиотек:

```powershell
# Установка musl target
rustup target add x86_64-unknown-linux-musl

# Вручную измените в build.ps1 строку 35:
# $rustTarget = if ($Target -eq "linux") { "x86_64-unknown-linux-musl" } else { "x86_64-pc-windows-msvc" }

# Или используйте cargo напрямую:
cargo build --release --target x86_64-unknown-linux-musl
```

## Результаты сборки

После успешной сборки:
- **Бинарник:** `target/x86_64-unknown-linux-gnu/release/aniruntime` (без расширения для Linux)
- **Плагины:** `plugins/*.so` (вместо `.dll`)

## Альтернативные методы

### Использование Docker

Если кросс-компиляция не работает, можно использовать Docker:

```dockerfile
FROM rust:latest
WORKDIR /app
COPY . .
RUN cargo build --release
```

### Использование WSL

Если у вас установлен WSL (Windows Subsystem for Linux), можно собрать проект напрямую в Linux окружении:

```bash
# В WSL
./build.sh release
```

## Устранение проблем

### Ошибка линкера

Если появляется ошибка линкера при сборке для Linux:
1. Попробуйте вариант с `musl` (статическая сборка)
2. Или установите MinGW-w64 с поддержкой Linux toolchain
3. Или используйте Docker/WSL

### Плагины не копируются

Убедитесь, что:
- Плагины собраны успешно (проверьте `target/x86_64-unknown-linux-gnu/release/`)
- Имена плагинов определены правильно в `Cargo.toml`

