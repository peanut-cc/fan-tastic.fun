---
title: "《Zero To Production In Rust》笔记(二) "
date: 2023-02-01T20:00:36+08:00
draft: false
tags: ["rust"]
categories: ["《Zero To Production In Rust》笔记"]
---

作者在关于这个项目的编写过程应该是目前看到的比较完善的，不管是继承测试，还是写代码的方式，作者最开始说：

- Make a change
- Compile the application
- Run tests
- Run the application

这个过程贯穿项目的整个过程，也是非常值得自己学习的地方。也让自己看到了测试对于项目的持续开发的重要性。

## 关于 Actix

### 一个简单的 web server

```rust
use actix_web::{web, App, HttpRequest, HttpServer, Responder};

async fn greet(req: HttpRequest) -> impl Responder {
    let name = req.match_info().get("name").unwrap_or("World");
    format!("Hello {}!", &name)
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .route("/", web::get().to(greet))
            .route("/{name}", web::get().to(greet))
    })
    .bind("127.0.0.1:8000")?
    .run()
    .await
}

```

这里推荐一个工具 `cargo-expand` 可以将代码中宏的部分进行展开，上述代码我们执行`cargo expand`，可以看到如下结果：

```rust
#![feature(prelude_import)]
#[prelude_import]
use std::prelude::rust_2021::*;
#[macro_use]
extern crate std;
use actix_web::{web, App, HttpRequest, HttpServer, Responder};
async fn greet(req: HttpRequest) -> impl Responder {
    let name = req.match_info().get("name").unwrap_or("World");
    {
        let res = ::alloc::fmt::format(format_args!("Hello {0}!", & name));
        res
    }
}
fn main() -> std::io::Result<()> {
    let body = async {
        HttpServer::new(|| {
                App::new()
                    .route("/", web::get().to(greet))
                    .route("/{name}", web::get().to(greet))
            })
            .bind("127.0.0.1:8000")?
            .run()
            .await
    };
    #[allow(clippy::expect_used, clippy::diverging_sub_expression)]
    {
        return tokio::runtime::Builder::new_multi_thread()
            .enable_all()
            .build()
            .expect("Failed building the Runtime")
            .block_on(body);
    }
}

```

从展开的代码看到 main  函数是同步的，不过最后又这么一段代码：

```rust
{
    return tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .build()
        .expect("Failed building the Runtime")
        .block_on(body);
}
```

启动了一个 tokio 的异步运行时，使用它来驱动 HttpServer::run，其实换就是`#[tokio::main]` 是给我们提供了一种方便些异步代码的方式。

### Actix Application State

> actix-web gives us the possibility to attach to the application other pieces of data that are not related to the lifecycle of a single incoming request - the so-called application state

`Application State` 一个最常用的地方应该就是数据连接了。下面是代码例子：

```rust
pub fn run(listener: TcpListener, db_pool: PgPool) -> Result<Server, std::io::Error> {
    let db_pool = web::Data::new(db_pool);
    let server = HttpServer::new(move || {
        App::new()
            .wrap(TracingLogger::default())
            .route("/health_check", web::get().to(health_check))
            .route("/subscriptions", web::post().to(subscribe))
            .app_data(db_pool.clone())
    })
    .listen(listener)?
    .run();
    Ok(server)
}
```

### actix URL-Encoded Forms

> "A URL-encoded form body can be extracted to a struct, much like `Json<T>`. This type must implement `serde::Deserialize`."

```rust
#[derive(serde::Deserialize)]
pub struct FormData {
    email: String,
    name: String,
}

/// extract form data using serde
/// this handler gets called only if the content type is *x-www-form-urlencoded*
/// and the content of the request could be deserialized to a `FormData` struct
pub async fn subscribe(form: web::Form<FormData>) -> HttpResponse {
}
```

### listen 127.0.0.1:0

在测试代码中使用确实一个比较不错的方式，不用再总是担心要用的端口已经被占用了。
> Port 0 is special-cased at the OS level: trying to bind port 0 will trigger an OS scan for an available port which will then be bound to the application

```rust
let listener = TcpListener::bind("127.0.0.1:0").expect("Failed to bind random port");
let port = listener.local_addr().unwrap().port();
let address = format!("http://127.0.0.1:{}", port);
```

### 关于日志

## 关于测试

在Rust 中通常把测试分为两种：单元测试和集成测试

单元测试目标是测试某段代码（一般是函数），验证该单元是否能按照预期工作。在Rust中，单元测试的代码通常是和待测试的代码放在同一个文件中。

```rust
pub fn add_two(a: i32) -> i32 {
    a + 2
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() {
        assert_eq!(add_two(2), 4);
    }
}
```

集成测试的代码是在一个单独的目录下，只能通过调用通过 `pub` 定义的 `API`。集成测试通常是对某一个功能或者接口进行测试。在Rust 中，通常是在根目录下建立一个 `tests` 目录，用于存放继承测试的代码。注意：`tests` 目录下的每个文件都是一个单独的包。

对于目前的这个程序来说，可以通过集成测试来测试对外提供的功能，在后续实现中，也可以根据需要对需要测试的函数添加单元测试。
作者在这本书中非常的注重测试，这是自己需要改进的地方。

## 关于Config.toml 中的 lib 和 bin

作者在config.toml 中添加了如下配置：

```toml
[lib]
path = "src/lib.rs"

[[bin]]
path = "src/main.rs"
name = "zero2prod"
```

先看一下一个典型的 Package 目录结构为：

```bash
.
├── Cargo.lock
├── Cargo.toml
├── src/
│   ├── lib.rs
│   ├── main.rs
│   └── bin/
│       ├── named-executable.rs
│       ├── another-executable.rs
│       └── multi-file-executable/
│           ├── main.rs
│           └── some_module.rs
├── benches/
│   ├── large-input.rs
│   └── multi-file-bench/
│       ├── main.rs
│       └── bench_module.rs
├── examples/
│   ├── simple.rs
│   └── multi-file-example/
│       ├── main.rs
│       └── ex_module.rs
└── tests/
    ├── some-integration-tests.rs
    └── multi-file-test/
        ├── main.rs
        └── test_module.rs

```

- `Cargo.toml` 和 `Cargo.lock` 保存在 `package` 根目录下
- 源代码放在 `src` 目录下
- 默认的 `lib` 包根是 `src/lib.rs`
- 默认的二进制包根是 `src/main.rs`
  - 其它二进制包根放在 `src/bin/` 目录下
- 基准测试 `benchmark` 放在 `benches` 目录下
- 示例代码放在 `examples` 目录下
- 集成测试代码放在 `tests` 目录下

Cargo 项目中包含有一些对象，它们包含的源代码文件可以被编译成相应的包，这些对象被称之为 `Cargo Target`.
这里重点看一下关于 `lib`  和 `bin` 的配置

库对象用于定义一个库，该库可以被其它的库或者可执行文件所链接。该对象包含的默认文件名是 `src/lib.rs`，且默认情况下，库对象的名称跟项目名是一致的
二进制对象在被编译后可以生成可执行的文件，默认的文件名是 `src/main.rs`，二进制对象的名称跟项目名也是相同的。
一个项目是可以拥有多个二进制文件，因此一个项目项目是可以拥有多个二进制对象。当拥有多个对象时，对象的文件默认会放在 `src/bin/` 目录下。所以配置文件中 lib 是使用 `[lib]` 而 bin 是使用 `[[bin]]`。

## 本地开发Docker相关

在本地开发时快速使用脚本用于去创建一个postgres的Docker容器

```bash
#!/usr/bin/env bash
set -x
set -eo pipefail

if ! [ -x "$(command -v psql)" ]; then
    echo >&2 "Error: psql is not installed."
    exit 1
fi

if ! [ -x "$(command -v sqlx)" ]; then
    echo >&2 "Error: sqlx is not installed."
    echo >&2 "Use:"
    echo >&2 " cargo install sqlx-cli"
    echo >&2 "to install it." 
    exit 1
fi

# Check if a custom user has been set, otherwise default to 'postgres'
DB_USER=${POSTGRES_USER:=postgres}
# Check if a custom password has been set, otherwise default to 'password'
DB_PASSWORD="${POSTGRES_PASSWORD:=password}"
# Check if a custom database name has been set, otherwise default to 'newsletter'
DB_NAME="${POSTGRES_DB:=newsletter}"
# Check if a custom port has been set, otherwise default to '5432'
DB_PORT="${POSTGRES_PORT:=5432}"


# Launch postgres using Docker
docker run \
    -e POSTGRES_USER=${DB_USER} \
    -e POSTGRES_PASSWORD=${DB_PASSWORD} \
    -e POSTGRES_DB=${DB_NAME} \
    -p "${DB_PORT}":5432 \
    -d postgres \
    postgres -N 1000
    # Încreased maximum number of connections for testing purposes

# Keep pinging Postgres until it's ready to accept commands 
export PGPASSWORD="${DB_PASSWORD}"
until psql -h "localhost" -U "${DB_USER}" -p "${DB_PORT}" -d "postgres" -c '\q'; do 
    >&2 echo "Postgres is still unavailable - sleeping" 
    sleep 1
done

>&2 echo "Postgres is up and running on port ${DB_PORT}!"

export DATABASE_URL=postgres://${DB_USER}:${DB_PASSWORD}@localhost:${DB_PORT}/${DB_NAME}
sqlx database create
```

## 关于 rust config 库

<https://docs.rs/config/latest/config/>

这个库还是非常方便的，可以处理多种文件格式的配置文件，并支持通过环境变量获取配置，下面是代码中使用的方式

```rust
use config::Config;
use secrecy::{ExposeSecret, Secret};
use serde_aux::field_attributes::deserialize_number_from_string;
use sqlx::{
    postgres::{PgConnectOptions, PgSslMode},
    ConnectOptions,
};

#[derive(serde::Deserialize)]
pub struct Settings {
    pub database: DatabaseSettings,
    pub application: ApplicationSettings,
}

#[derive(serde::Deserialize)]
pub struct ApplicationSettings {
    #[serde(deserialize_with = "deserialize_number_from_string")]
    pub port: u16,
    pub host: String,
}

#[derive(serde::Deserialize)]
pub struct DatabaseSettings {
    pub username: String,
    pub password: Secret<String>,
    #[serde(deserialize_with = "deserialize_number_from_string")]
    pub port: u16,
    pub host: String,
    pub database_name: String,
    pub require_ssl: bool,
}

impl DatabaseSettings {
    pub fn without_db(&self) -> PgConnectOptions {
        let ssl_mode = if self.require_ssl {
            PgSslMode::Require
        } else {
            PgSslMode::Prefer
        };
        PgConnectOptions::new()
            .host(&self.host)
            .username(&self.username)
            .password(self.password.expose_secret())
            .port(self.port)
            .ssl_mode(ssl_mode)
    }

    pub fn with_db(&self) -> PgConnectOptions {
        let mut options = self.without_db().database(&self.database_name);
        options.log_statements(tracing::log::LevelFilter::Trace);
        options
    }
}

pub fn get_configuration() -> Result<Settings, config::ConfigError> {
    let base_path = std::env::current_dir().expect("Failed to determine the current directory");
    let configuration_directory = base_path.join("configuration");
    let base_source = config::File::from(configuration_directory.join("base"));

    let environment: Environment = std::env::var("APP_ENVIRONMENT")
        .unwrap_or_else(|_| "local".into())
        .try_into()
        .expect("Failed to parse APP_ENVIRONMENT");
    let environment_base = config::File::from(configuration_directory.join(environment.as_str()));

    let builder = Config::builder()
        .add_source(base_source)
        .add_source(environment_base)
        .add_source(config::Environment::with_prefix("app").separator("__"));
    builder.build()?.try_deserialize()
}

#[derive(Debug)]
pub enum Environment {
    Local,
    Production,
}

impl Environment {
    pub fn as_str(&self) -> &'static str {
        match self {
            Environment::Local => "local",
            Environment::Production => "production",
        }
    }
}

impl TryFrom<String> for Environment {
    type Error = String;
    fn try_from(s: String) -> Result<Self, Self::Error> {
        match s.to_lowercase().as_str() {
            "local" => Ok(Self::Local),
            "production" => Ok(Self::Production),
            other => Err(format!(
                "{other} is not a supported environment. Use either `localòr `production`.",
            )),
        }
    }
}

```

## 关于测试隔离

There are two techniques I am aware of to ensure test isolation when interacting with a relational database in a test:

- wrap the whole test in a SQL transaction and rollback at the end of it
- spin up a brand-new logical database for each integration test

The first is clever and will generally be faster: rolling back a SQL transaction takes less time than spinning up a new logical database. It works quite well when writing unit tests for your queries but it is tricky to pull off in an integration test like ours: our application will borrow a PgConnection from a PgPool and we have no way to “capture” that connection in a SQL transaction context.
Which leads us to the second option: potentially slower, yet much easier to implemen

```rust
// Lauch our application in the background
async fn spawn_app() -> TestApp {
    Lazy::force(&TRACING);

    let listener = TcpListener::bind("127.0.0.1:0").expect("Failed to bind random port");
    let port = listener.local_addr().unwrap().port();
    let address = format!("http://127.0.0.1:{}", port);

    let mut configuration = get_configuration().expect("Failed to read configuration.");
    configuration.database.database_name = Uuid::new_v4().to_string();
    let connection_pool = configure_database(&configuration.database).await;

    let server = run(listener, connection_pool.clone()).expect("Failed to bind address");
    let _ = tokio::spawn(server);
    TestApp {
        address,
        db_pool: connection_pool,
    }
}
```

上述代码是在tests 中 程序使用的，这种用法还是挺方便的，每次测试是时的 `database_name` 都是使用`uuid`生成.
