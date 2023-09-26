---
title: "Rust笔记----Axum 写 Middleware"
date: 2023-09-26T16:11:13+08:00
tags: ["Rust", "Axum"]
categories: ["Rust"]
keywords: ["rust", "Axum", "Axum Middleware"]
draft: false
---


最近看了 <https://www.youtube.com/watch?v=XZtlD_m59sM> 关于Rust Axum 开发web的介绍，在代码中有些之前不了解的地方，通过这个笔记进行整理。

## Axum Middleware

假设我们目前有如下几个自定义中间件

```rust
pub async fn mw_test1<B>(req: Request<B>, next: Next<B>) -> Result<Response> {
    println!("->> {:<12} - mw_test1", "MIDDLEWARE");
    Ok(next.run(req).await)
}
pub async fn mw_test2<B>(req: Request<B>, next: Next<B>) -> Result<Response> {
    println!("->> {:<12} - mw_test2", "MIDDLEWARE");
    Ok(next.run(req).await)
}

pub async fn mw_test3<B>(req: Request<B>, next: Next<B>) -> Result<Response> {
    println!("->> {:<12} - mw_test3", "MIDDLEWARE");
    Ok(next.run(req).await)
}

async fn main_response_mapper(_ctx: Option<Ctx>, res: Response) -> Response {
    println!("->> {:<12} - main_response_mapper", "RES_MAPPER");
    res
}

async fn main_response_mapper2(_ctx: Option<Ctx>, res: Response) -> Response {
    println!("->> {:<12} - main_response_mapper2", "RES_MAPPER");
    res
}
```

并按照如下方式进行添加：

```rust
let routes_all = Router::new()
    .merge(routes_hello())
    .layer(middleware::from_fn(mw_test1))
    .layer(middleware::from_fn(mw_test2))
    .layer(middleware::from_fn(mw_test3))
    .layer(middleware::map_response(main_response_mapper))
    .layer(middleware::map_response(main_response_mapper2))

    .layer(CookieManagerLayer::new());
```

请求hello接口打印如下：

```bash
->> EXTRACTOR    - Ctx
->> EXTRACTOR    - Ctx
->> MIDDLEWARE   - mw_test3
->> MIDDLEWARE   - mw_test2
->> MIDDLEWARE   - mw_test1
->> HANDLER      - handler_hello - HelloParams { name: None }
->> RES_MAPPER   - main_response_mapper
->> RES_MAPPER   - main_response_mapper2
```

从结果中可以知道如下内容：
通过 `layer(middleware::from_fn(xxx))`添加的中间件，每个请求经过的中间件的顺序正好我们代码中添加的顺序相反
通过 `layer(middleware::map_response(xxx))` 添加的中间件，是用于对响应的处理，不过这个经过的顺序和我们代码中添加的顺序保持一致。
同时需要知道我们在 `layer(middleware::map_response(xxx))` 的中间件是执行 `extractors`, 也可以运行实现 `FromRequestParts` 的`extractors`, 这些将在调用`handler`之前运行。

例如我们在这里为Ctx 实现了 `FromRequestParts`

```rust
#[derive(Debug, Clone)]
pub struct Ctx;

// region:    --- Ctx Extractor
#[async_trait]
impl<S: Send + Sync> FromRequestParts<S> for Ctx {
    type Rejection = Error;

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self> {
        println!("->> {:<12} - Ctx", "EXTRACTOR");

        parts
            .extensions
            .get::<Result<Ctx>>()
            .ok_or(Error::AuthFailCtxNotInRequestExt)?
            .clone()
    }
}
```

定义的`main_response_mapper`和 `main_response_mapper2`的代码为：

```rust
async fn main_response_mapper(_ctx: Option<Ctx>, res: Response) -> Response {
    println!("->> {:<12} - main_response_mapper", "RES_MAPPER");
    res
}

async fn main_response_mapper2(_ctx: Option<Ctx>, res: Response) -> Response {
    println!("->> {:<12} - main_response_mapper2", "RES_MAPPER");
    res
}
```

所以当我调用hello接口的时候会看到的是两个 `->> EXTRACTOR    - Ctx`，正是因为调用`extractors`是在处理handler之前。
这种实现 `from_request_parts`的方式可以帮助我们写出非常简洁的代码，类似我们对于需要需要认证的中间件中加入Ctx 参数，在`from_request_parts` 写相关的header 字段的提取和验证，类似如下代码：

```rust
#[async_trait]
impl<S> FromRequestParts<S> for User
    where
        S: Send + Sync,
{
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(parts: &mut Parts, _: &S) -> Result<Self, Self::Rejection> {
        let auth_header = parts.headers.get("X-Auth-Token")
            .and_then(|header| header.to_str().ok())
            .ok_or((StatusCode::UNAUTHORIZED, "Unauthorized"))?;

        verify_auth_token(auth_header).await
            .map_err(|_| (StatusCode::UNAUTHORIZED, "Unauthorized"))
    }
}
```

## 相关链接

- <https://docs.rs/axum/latest/axum/middleware/fn.map_response.html>
- <https://docs.rs/axum/latest/axum/middleware/index.html>
- <https://www.propelauth.com/post/clean-code-with-rust-and-axum>
- <https://www.youtube.com/watch?v=XZtlD_m59sM>
