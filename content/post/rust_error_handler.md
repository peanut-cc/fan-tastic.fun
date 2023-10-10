---
title: "Rust笔记----探索rust错误处理最佳实践"
date: 2023-10-09T16:05:11+08:00
tags: ["Rust", "rust错误处理"]
categories: ["Rust"]
keywords: ["rust","rust错误处理", "rust error"]
draft: true
---

最近在通过Rust的Axum web框架写web服务进行rust编码实践，在这个过程中对于rust编码中如何更好的处理错误，不一定是最好的办法，但是是一次不错的学习过程，后续也会持续探索rust中错误处理的最佳实践。

在这次web demo中，请求的流程为`API -> Model` 即请求进来之后到 api 处理逻辑并操作Model进行相关的数据库操作，整体的目录结构:

```rust
./src
├── _dev_utils
├── config.rs
├── crypt
│   ├── error.rs
│   ├── mod.rs
│   ├── pwd.rs
│   └── token.rs
├── ctx
│   ├── error.rs
│   └── mod.rs
├── error.rs
├── log
├── main.rs
├── model
│   ├── base.rs
│   ├── error.rs
│   ├── mod.rs
│   ├── store
│   │   ├── error.rs
│   │   └── mod.rs
│   ├── task.rs
│   └── user.rs
├── utils
│   ├── error.rs
│   └── mod.rs
└── web
    ├── error.rs
    ├── mod.rs
    ├── mw_auth.rs
    ├── mw_res_map.rs
    ├── routes_login.rs
    ├── routes_static.rs
    └── rpc
```

因为主要是通过这个熟悉Axum的使用，代码目录相对来说比较简单。在这个项目中基本各级别的目录都定义了自己的error,并且都是用枚举的方式。并手动为每个error 实现 Display(`core::fmt::Display`) 和 标准库的Error(`std::error::Error`)

## Wrap Error

在通常Rust开发中会使用 枚举的方式 定义不同的错误类型，如：

```rust
#[derive(Debug)]
enum Error {
    EmptyVec,
    Parse(ParseIntError),
}
```

通常为了可以方便的打印日志，会为定义的错误枚举类型实现 `core::fmt::Display` trait:

```rust
impl core::fmt::Display for Error {
    fn fmt(&self, fmt: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(fmt, "{self:?}")
    }
}
```

对于这个错误我们最终会放到Result中:

```rust
type Result<T> = std::result::Result<T, Error>;
```

业务代码：

```rust
fn doubule_first(vec: Vec<&str>) -> Result<i32> {
    let first = vec.first().ok_or(Error::EmptyVec)?;
    let parsed = first.parse::<i32>()?;
    Ok(2 * parsed)
}

fn print(result: Result<i32>) {
    match result {
        Ok(n) => println!("The first doubled is {}", n),
        Err(e) => println!("Error: {}", e),
    }
}

fn main() {
    let numbers = vec!["12", "23", "100"];
    let empty = vec![];
    let strings = vec!["fan-tasitc", "2", "23"];
    print(doubule_first(numbers));
    print(doubule_first(strings));
    print(doubule_first(empty));
}
```

这个时候编译代码会看到如下错误：

```bash
error[E0277]: `?` couldn't convert the error to `Error`
  --> src/main.rs:29:38
   |
27 | fn doubule_first(vec: Vec<&str>) -> Result<i32> {
   |                                     ----------- expected `Error` because of this
28 |     let first = vec.first().ok_or(Error::EmptyVec)?;
29 |     let parsed = first.parse::<i32>()?;
   |                                      ^ the trait `From<ParseIntError>` is not implemented for `Error`
   |
   = note: the question mark operation (`?`) implicitly performs a conversion on the error value using the `From` trait
   = help: the following other types implement trait `FromResidual<R>`:
             <std::result::Result<T, F> as FromResidual<Yeet<E>>>
             <std::result::Result<T, F> as FromResidual<std::result::Result<Infallible, E>>>
   = note: required for `std::result::Result<i32, Error>` to implement `FromResidual<std::result::Result<Infallible, ParseIntError>>`
```

这是因为 `first.parse::<i32>()`返回的是 `std::result::Result<i32, ParseIntError>`，当我们通过`?`  操作符传播错误时，就会发现我们定义`doubule_first`函数的返回类型是`Result<i32>`，其中的Error是我们定义的枚举类型`Error`,所以就会提示不能将`ParseIntError` 转换为我们定义的`Error`。

不过从编译错误提示中，也看到提示我们

```bash
the trait `From<ParseIntError>` is not implemented for `Error`
```

按照提示我们为我们定义的枚举类型的`Error`实现 `From`:

```rust
impl From<ParseIntError> for Error {
    fn from(value: ParseIntError) -> Self {
        Error::Parse(value)
    }
}
```

再次编译代码成功。

当我们给枚举类型`Error`实现了`From trait`之后，代码编译成功了。
实现 `From trait`的作用在我们上面的代码中的作用就是帮助我们把`ParseIntError`转换为我们定义的枚举类型错误 `Error`，
在使用 `?` 传播错误，或者一个将 `ParseIntError` 需要转换成 `DoubleError` 时，它会被自动调用。

## 相关链接

- <https://rustwiki.org/zh-CN/rust-by-example/error/multiple_error_types/wrap_error.html>
- <https://doc.rust-lang.org/std/convert/trait.From.html>
