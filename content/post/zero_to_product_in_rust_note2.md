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

对于目前开发的这个程序，就目前的测试功能来说，继承测试更加符合，模拟用户使用进行测试。
