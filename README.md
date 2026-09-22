## 1. Что нужно сделать

В итоге у тебя должен получиться проект:

```text
hello-rust/
├── .github/
│   └── workflows/
│       └── ci.yml
├── src/
│   ├── main.rs
│   └── lib.rs
├── tests/
│   └── integration.rs
├── .gitignore
├── .dockerignore
├── Dockerfile
└── Cargo.toml
```

Задание знакомит с Cargo, тестами Rust, Docker Multi-stage и GitHub Actions. 

---

# 2. Создание проекта

Открой **Git Bash**, WSL или терминал Linux.

Перейди в домашний каталог:

```bash
cd ~
```

Создай проект:

```bash
mkdir -p hello-rust/.github/workflows
mkdir -p hello-rust/src
mkdir -p hello-rust/tests

cd hello-rust
```

Теперь структура будет:

```text
hello-rust/
├── .github/
│   └── workflows/
├── src/
└── tests/
```

---

# 3. Файл `Cargo.toml`

Создай файл:

```text
Cargo.toml
```

Вставь:

```toml
[package]
name = "hello-rust"
version = "0.1.0"
edition = "2021"

[dependencies]

[profile.release]
opt-level = "z"
lto = true
strip = true
```

Здесь:

* `name` — название проекта;
* `version` — версия;
* `edition` — версия Rust Edition;
* `opt-level = "z"` — оптимизация размера;
* `lto = true` — Link Time Optimization;
* `strip = true` — удаление отладочной информации.

Это соответствует содержимому задания. 

---

# 4. Файл `src/lib.rs`

Создай:

```text
src/lib.rs
```

Вставь:

```rust
pub fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}

pub fn sum_range(from: i64, to: i64) -> i64 {
    (from..=to).sum()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_greet() {
        assert_eq!(greet("Docker"), "Hello, Docker!");
    }

    #[test]
    fn test_sum_range() {
        assert_eq!(sum_range(1, 10), 55);
    }
}
```

Здесь находятся две функции:

```rust
greet()
```

выводит приветствие.

А:

```rust
sum_range()
```

считает сумму чисел от `from` до `to`.

Также здесь находятся два **unit-теста**. 

---

# 5. Файл `src/main.rs`

Создай:

```text
src/main.rs
```

Вставь:

```rust
use hello_rust::{greet, sum_range};

fn main() {
    println!("Hello from Rust in Docker! 🦀🐳");
    println!("OS: {}", std::env::consts::OS);
    println!("Arch: {}", std::env::consts::ARCH);
    println!("{}", greet("Docker"));
    println!("Sum 1..10 = {}", sum_range(1, 10));

    let args: Vec<String> = std::env::args().skip(1).collect();

    if !args.is_empty() {
        println!("Аргументы:");

        for (i, arg) in args.iter().enumerate() {
            println!("  {}: {}", i + 1, arg);
        }
    }
}
```

Программа:

1. выводит приветствие;
2. показывает операционную систему;
3. показывает архитектуру;
4. вызывает `greet()`;
5. вызывает `sum_range()`;
6. выводит аргументы командной строки, если они переданы.

Это предусмотрено исходным заданием. 

---

# 6. Интеграционные тесты

Создай:

```text
tests/integration.rs
```

Вставь:

```rust
use hello_rust::{greet, sum_range};

#[test]
fn integration_test_greet() {
    assert_eq!(greet("Rust"), "Hello, Rust!");
    assert!(greet("CI").contains("CI"));
}

#[test]
fn integration_test_sum_large() {
    assert_eq!(sum_range(1, 100), 5050);
}
```

Здесь находятся **integration tests**.

Они проверяют функции библиотеки уже как внешний тестовый модуль. 

---

# 7. Dockerfile

Теперь создаём:

```text
Dockerfile
```

Вставь:

```dockerfile
# Этап 1: сборка
FROM rust:1-slim AS builder

WORKDIR /build

# Копируем манифест
COPY Cargo.toml Cargo.lock ./

# Фиктивный main, чтобы собрать зависимости отдельным слоем
RUN mkdir src && echo "fn main() {}" > src/main.rs

# Собираем зависимости
RUN cargo build --release

# Удаляем фиктивный бинарник
RUN rm -f target/release/hello-rust target/release/deps/hello_rust-*

# Копируем настоящий исходник
COPY src ./src
COPY tests ./tests

# Собираем финальный бинарник
RUN cargo build --release


# Этап 2: запуск
FROM debian:stable-slim

# Непривилегированный пользователь
RUN useradd --create-home appuser

WORKDIR /home/appuser

COPY --from=builder /build/target/release/hello-rust ./hello-rust

USER appuser

ENTRYPOINT ["./hello-rust"]
```

Здесь используется **Multi-stage Docker build**:

```text
rust:1-slim
     ↓
компиляция Rust
     ↓
готовый бинарник
     ↓
debian:stable-slim
     ↓
маленький финальный образ
```

Так как Rust-компиляция создаёт самостоятельный бинарник, для запуска не требуется JVM или другой runtime. 

---

# 8. Важный момент с `Cargo.lock`

В Dockerfile есть:

```dockerfile
COPY Cargo.toml Cargo.lock ./
```

Поэтому сначала нужно получить `Cargo.lock`.

Если Rust установлен:

```bash
cargo generate-lockfile
```

После этого появится:

```text
Cargo.lock
```

Он должен находиться рядом с `Cargo.toml`.

В итоге:

```text
hello-rust/
├── Cargo.toml
└── Cargo.lock
```

---

# 9. `.gitignore`

Создай:

```text
.gitignore
```

Содержимое:

```gitignore
/target
```

Каталог `target` содержит результаты компиляции, поэтому в Git его добавлять не нужно. 

---

# 10. `.dockerignore`

Создай:

```text
.dockerignore
```

Вставь:

```dockerignore
target/
.git/
.github/
*.md
.gitignore
.dockerignore
```

Это уменьшает количество файлов, отправляемых Docker в build context. 

---

# 11. Проверка проекта без установленного Rust

По заданию Rust на компьютере вообще не обязателен.

Можно использовать Docker.

### Git Bash / Linux / WSL

Выполни:

```bash
mkdir -p ~/.cargo-docker-cache

docker run --rm \
  -u "$(id -u):$(id -g)" \
  -e HOME=/tmp \
  -e CARGO_HOME=/tmp/.cargo \
  -v "$(pwd)":/app \
  -v ~/.cargo-docker-cache:/tmp/.cargo \
  -w /app \
  rust:1-slim \
  cargo test
```

### PowerShell

Если используешь Windows PowerShell:

```powershell
docker run --rm `
  -e CARGO_HOME=/tmp/.cargo `
  -v "${PWD}:/app" `
  -w /app `
  rust:1-slim `
  cargo test
```

В Windows должен быть запущен **Docker Desktop**. Если проект находится на диске, доступ к которому Docker Desktop не имеет, его нужно разрешить в настройках Docker Desktop. 

---

# 12. Что должно произойти после `cargo test`

Docker скачает Rust-образ и выполнит:

```bash
cargo test
```

Будут запущены:

### Unit tests

```text
test_greet
test_sum_range
```

### Integration tests

```text
integration_test_greet
integration_test_sum_large
```

Если всё правильно, тесты завершатся успешно.

---

# 13. Сборка Docker-образа

Теперь перейди в проект:

```bash
cd ~/hello-rust
```

Собери Docker image:

```bash
docker build -t hello-rust .
```

То есть:

```text
Dockerfile
     ↓
docker build
     ↓
hello-rust image
```

Команда соответствует заданию. 

---

# 14. Проверка Docker image

Посмотри созданный image:

```bash
docker images
```

Ты должен увидеть примерно:

```text
REPOSITORY   TAG       IMAGE ID       CREATED
hello-rust   latest    ........       ...
```

---

# 15. Запуск контейнера

Выполни:

```bash
docker run --rm hello-rust
```

Ожидаемый результат:

```text
Hello from Rust in Docker! 🦀🐳
OS: linux
Arch: x86_64
Hello, Docker!
Sum 1..10 = 55
```
<img width="1894" height="189" alt="image" src="https://github.com/user-attachments/assets/19634d70-184f-46db-91bc-d744aa6ff747" />

Это указанный в задании ожидаемый вывод. 

---

# 16. Проверка аргументов

Можно дополнительно проверить аргументы:

```bash
docker run --rm hello-rust one two three
```

Программа должна вывести:

```text
Hello from Rust in Docker! 🦀🐳
OS: linux
Arch: x86_64
Hello, Docker!
Sum 1..10 = 55
Аргументы:
  1: one
  2: two
  3: three
```

---

# 17. Создание репозитория GitHub

Теперь нужно создать репозиторий на GitHub.

Название:

```text
hello-rust
```

Репозиторий должен быть **пустым**.

При создании не ставь галочки:

```text
☐ Add a README file
☐ Add .gitignore
☐ Choose a license
```

В задании отдельно указано создать пустой репозиторий. 

---

# 18. Инициализация Git

Находясь в:

```text
~/hello-rust
```

выполни:

```bash
git init
```

Добавь файлы:

```bash
git add .
```

Создай первый commit:

```bash
git commit -m "Initial commit: Rust app with Docker and CI"
```

Переименуй ветку:

```bash
git branch -M main
```

Можно выполнить всё одной командой:

```bash
git init && git add . && git commit -m "Initial commit: Rust app with Docker and CI" && git branch -M main
```

Именно такой порядок приведён в задании. 

---

# 19. Подключение GitHub

Добавь удалённый репозиторий:

```bash
git remote add origin https://github.com/ТВОЙ_USERNAME/hello-rust.git
```

Например:

```bash
git remote add origin https://github.com/example/hello-rust.git
```

Вместо:

```text
ТВОЙ_USERNAME
```

ставишь свой логин GitHub. 

---

# 20. Отправка проекта на GitHub

Выполни:

```bash
git push -u origin main
```

После этого проект появится на GitHub.

---

# 21. GitHub Actions

Самое важное — файл:

```text
.github/workflows/ci.yml
```

Создай его и вставь:

```yaml
name: Rust CI

on:
  push:
    branches: [ main ]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Rust
        uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt, clippy

      - name: Cache Cargo
        uses: actions/cache@v4
        with:
          path: |
            ~/.cargo/registry
            ~/.cargo/git
            target
          key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}

      - name: Format check
        run: cargo fmt --check

      - name: Lint with Clippy
        run: cargo clippy --all-targets -- -D warnings

      - name: Run tests
        run: cargo test

      - name: Build release
        run: cargo build --release

      - name: Build Docker image
        run: docker build -t hello-rust .
```

Этот workflow автоматически выполняет форматирование, Clippy, тесты, release-сборку и Docker build. 

---

# 22. Что происходит в GitHub Actions

После:

```bash
git push -u origin main
```

GitHub увидит:

```text
.github/workflows/ci.yml
```

и запустит workflow:

```text
Rust CI
```

Внутри будут следующие этапы:

```text
Checkout
   ↓
Установка Rust
   ↓
Кэш Cargo
   ↓
cargo fmt --check
   ↓
cargo clippy
   ↓
cargo test
   ↓
cargo build --release
   ↓
docker build
```

---

# 23. Проверка Actions

На GitHub открой свой репозиторий:

```text
hello-rust
```

Затем:

```text
Actions
```

Там должен появиться workflow:

```text
Rust CI
```

Если всё прошло успешно, возле запуска будет зелёная галочка:

```text
✓ Rust CI
```

В задании именно успешный запуск Actions является завершающей проверкой CI. 

---

# 24. Итоговая структура

В конце у тебя должно быть примерно так:

```text
hello-rust/
│
├── .github/
│   └── workflows/
│       └── ci.yml
│
├── src/
│   ├── main.rs
│   └── lib.rs
│
├── tests/
│   └── integration.rs
│
├── .gitignore
├── .dockerignore
├── Cargo.toml
├── Cargo.lock
└── Dockerfile
```

---

# 25. Короткий список команд

Если тебе нужно просто выполнять задание по порядку:

```bash
cd ~
```

```bash
mkdir -p hello-rust/.github/workflows
mkdir -p hello-rust/src
mkdir -p hello-rust/tests
cd hello-rust
```

После создания всех файлов:

```bash
cargo generate-lockfile
```

Проверка тестов через Docker:

```bash
docker run --rm \
  -e CARGO_HOME=/tmp/.cargo \
  -v "${PWD}:/app" \
  -w /app \
  rust:1-slim \
  cargo test
```

Сборка:

```bash
docker build -t hello-rust .
```

Запуск:

```bash
docker run --rm hello-rust
```

Git:

```bash
git init
git add .
git commit -m "Initial commit: Rust app with Docker and CI"
git branch -M main
```

GitHub:

```bash
git remote add origin https://github.com/ТВОЙ_USERNAME/hello-rust.git
git push -u origin main
```

После этого:

```text
GitHub → Actions → Rust CI
```

и проверяешь, что workflow завершился успешно.
