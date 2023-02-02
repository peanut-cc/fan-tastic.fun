---
title: "《Zero To Production In Rust》笔记(二) "
date: 2023-02-01T20:00:36+08:00
draft: true
---


## 一个简单的 web server

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

## 关于测试

关于集成测试，嵌入式测试等不同测试代码的说明？需要补充整理
