---
title: "Rust笔记----错误处理"
date: 2023-07-30T10:23:02+08:00
tags: ["Rust", "rust错误处理"]
categories: ["Rust"]
keywords: ["rust","rust错误处理", "rust error"]
draft: false
---


## 关于rust 的错误

rust 有两种不同的错误处理机制: `panic` 和 `Result<T, E>`

```rust
pub enum Result<T, E> {
    /// Contains the success value
    Ok(T),
    /// Contains the error value
    Err(E),
}
```

Result是Rust提供的一个枚举类, 它里面应当包含程序成功运行时返回的值T,或者是程序运行失败时返回的错误类型E

panic 是线程级别的,一个线程panic 其他线程可以正常运行自己的业务逻辑.

一个 `Result<T, E>`的例子:

```rust
fn get_weather(location: LatLng) -> Result<WeatherReport,io::Error> {...}
```

get_weather函数 要么返回一个成功的结果 `Ok(weather)`,即weather就是一个新的WeatherReport值, 要么返回一个错误的结果`Err(err_value)`, error_value就是一个具体的io:Error错误的值.

Rust 要求在函数返回是Result的时候,必须写相应的处理逻辑:

通过match表达式处理:

```rust
match get_weather(hometown) {
    Ok(report) => {
        ....
    }
    Err(err) => {
        println!("error querying the weather: {}", err);
        ....
    }
}
```

同时Result 提供了一下非常方便的方法

- `result.is_ok()` 和 `result.is_err()` 返回 bool 值,告诉我们 result 是成功的结果还是错误的结果
- `result.ok()` 返回 `Option<T>` 类型的成功值(如果有的话),如果 result 是一个成功的结果,就返回 `Some(success_value)`,否则返回 None,而丢弃错误值
- `result.err()` 返回 `Option<E>` 类型的错误值(如果有的话)
- `result.unwrap()` 也会返回成功值(如果 result 是成功的结果的话). 不过如果 result 是错误的结果,这个方法则会panic
- `result.unwrap_or(fallback)` 如果 result 是成功的结果的话返回成功值, 否则返回 fallback，丢弃错误值
- `result.expect(message)` 与 `.unwrap()` 相同,只不过你需要自己提供panic时打印到控制台的消息

## 错误传播

rust使用`?` 操作符可以传播错误, 可以在任何产生Result的表达式后面添加`?`
`?` 操作符的行为取决与这个函数是返回一个成功的结果还是返回一个错误的结果:

- 如果是成功的值,那么会获取Result 其中成功的值
- 如果是错误的值, 将错误的结果沿调用链向上传播

### 自定义错误类型

在自己刚开始使用`?`的时候,你可能会碰到下面这种情况:

```rust
use std::io::{self, BufRead};

fn read_numbers(file: &mut dyn BufRead) -> Result<Vec<i64>, io::Error> {
    let mut numbers = vec![];
    for line_result in file.lines() {
        let line = line_result?;
        numbers.push(line.parse()?);
    }
    Ok(numbers)
}
```

编译代码会看到如下错误:

```bash
error[E0277]: `?` couldn't convert the error to `std::io::Error`
 --> src/main.rs:7:34
  |
3 | fn read_numbers(file: &mut dyn BufRead) -> Result<Vec<i64>, io::Error> {
  |                                            --------------------------- expected `std::io::Error` because of this
...
7 |         numbers.push(line.parse()?);
  |                                  ^ the trait `From<ParseIntError>` is not implemented for `std::io::Error`
  |
```

`line_result` 的类型是 `Result<String,std::io::Error>`
`line.parse()`的返回值是:`Result<i64, std::num::ParseIntError>`
而函数read_numbers的返回类型只能容纳`std::io::Error`
Rust 会尝试把 `std::num::ParseIntError` 转换为`std::io::Error`,但是失败了. 因为 trait `From<ParseIntError>` is not implemented for `std::io::Error`

可以通过一个简单方法解决,rust 中所有标准库的错误类型都可以转换为 `Box<std::error::Error>` 类型,即表示"任何错误"

```rust
type GenError = Box<std::error::Error>;
type GenResult<T> = Result<T, GenError>;
fn read_numbers(file: &mut dyn BufRead) -> GenResult<Vec<i64>> {}
```

在实际业务开发中通常会使用定义错误的枚举类型:

```rust
impl Error for MyError {

}

/// MyError属于当前自定义的枚举，其中包含了多种错误类型
/// MyError也包含了从下层传递而来的错误类型，统一归纳
#[derive(Debug)]
pub enum MyError {
    BadSchema(String, String, String),
    IO(io::Error),
    Read,
    Receive,
    Send,
}

//实现Display
impl fmt::Display for MyError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            MyError::BadSchema(s1, s2, s3) => {
                write!(f, "BadSchema Error:{}, {}, {}", s1, s2, s3)
            }
            MyError::IO(e) => {
                write!(f, "IO Error: {}", e)
            }
            MyError::Read => {
                write!(f, "Read Error")
            }
            MyError::Receive => {
                write!(f, "Receive Error")
            }
            MyError::Send => {
                write!(f, "Send Error")
            }
        }
    }
}

// 当我们为一个错误类型的转换实现了From方法, 就可以使用?来进行自动转换
impl From<io::Error> for MyError {
    fn from(err: io::Error) -> MyError {
        MyError::IO(err)
    }
}
```

在定义MyError时,其中包括了多种错误类型,有当前模块产生的错误（比如Read, Receive, Send）. 也有从下层模块传递上来的错误,比如IO(`io::Error`),针对从下层传递而来的这种错误, 我们需要将它归纳到自己的MyError中，统一传递给上层,当我们为一个错误类型的转换实现了From方法, 就可以使用?来进行自动转换.

### 使用trait Object传递错误

这种方式是直接使用 `Box<dyn Error>` 来统一错误类型. 使用 `?` 来传递错误,自动把Error 转换成 `Box<dyn Error>`

```rust
fn test_error() -> Result<i32, Box<dyn Error>> {
    let s = std::fs::read_to_string("test123.txt")?;
    let n = s.trim().parse::<i32>()?;
    Ok(n)
}
```

这种方式的好处在于不用再一个个地去定义错误类型,编写From方法,坏处在于丢失了具体的错误类型的信息,如果要对于不同的错误类型进行不同处理,就会遇到麻烦

## anyhow 和 thiserror

rust的错误处理有两个优秀的第三方库: `anyhow` 和 `thiserror`, 并且经常可以看到这句话:

> anyhow is for applications, thiserror is for libraries

anyhow 可以把用户自定义的，所有实现了`std::error::Error` trait的结构体，统一转换成它定义的`anyhow::Error`
这样在传递错误的过程中就可以使用统一的一个结构体,不用在定义各种各样的错误.

thiserror: 如果我们自定义一个MyError结构体,需要实现很多内容,Error trait,Display,Debug以及各种From函数,手动写可能比较麻烦,而thiserror这个库可以简化这个过程

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum DataStoreError {
    #[error("data store disconnected")]
    Disconnect(#[from] io::Error),
    #[error("the data for key `{0}` is not available")]
    Redaction(String),
    #[error("invalid header (expected {expected:?}, found {found:?})")]
    InvalidHeader {
        expected: String,
        found: String,
    },
    #[error("unknown data store error")]
    Unknown,
}
```

我们自定义的结构体前加上`#[derive(Error)]`,就可以自动`impl Error`
`#[error("invalid header (expected {expected:?}, found {found:?})")]`这条语句代表如何实现`Display`,后面的字符串就代表Display会输出的字符,同时支持格式化参数.比如这条语句里的expected就是代表结构体里面的元素,如果是元组则可以通过`.0`或者`.1`的方式来表示元素
`#[from]`表示会自动实现From方法


## 相关链接

- <https://docs.rs/thiserror/latest/thiserror/>
- <https://docs.rs/anyhow/latest/anyhow/>
- <https://rustmagazine.github.io/rust_magazine_2021/chapter_2/rust_error_handle_and_log.html>