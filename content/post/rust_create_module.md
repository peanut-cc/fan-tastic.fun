---
title: "Rust笔记----包和模块"
date: 2023-07-28T21:23:42+08:00
tags: ["Rust", "rust包管理"]
categories: ["Rust"]
keywords: ["rust","rust包管理", "rust模块"]
draft: false
---

## 包管理

在Rust中最基本的单位是包(crate), Rust包管理器Cargo.
通过`cargo new hello --lib`创建包 hello, `-lib` 创建的是库文件.
`cargo new hello --bin` 或者 省略 `--bin` 创建可执行文件爱你

```bash
❯ tree ./hello                         
./hello
├── Cargo.toml
└── src
    └── lib.rs
```

`Cargo.toml` 是包配置文件.内容为:

```toml
[package]
name = "hello"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]

```

目前生成的的`lib.rs`的初始化内容为:

```rust
pub fn add(left: usize, right: usize) -> usize {
    left + right
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() {
        let result = add(2, 2);
        assert_eq!(result, 4);
    }
}
```

rust中使用`mod` 来定义模块, `#[cfg(test)]` 属性为条件编译,告诉编译器只在运行测试`cargo test`时才编译执行.

```rust
❯ cargo test
   Compiling hello v0.1.0 (/home/peanut_fan/dev/rust_code/hello)
    Finished test [unoptimized + debuginfo] target(s) in 0.29s
     Running unittests src/lib.rs (target/debug/deps/hello-f36de81fe94c7d9a)

running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests hello

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

```

当运行了`cargo test`,`cargo build`或者`cargo run`之后的,项目目录会增加如下内容:

```bash
❯ tree ./hello     
./hello
├── Cargo.lock
├── Cargo.toml
├── src
│   └── lib.rs
└── target
```

`Cargo.lock` 只记录依赖包的详细信息,不需要开发者维护,而是由Cargo自动维护.
`target`文件夹是存储编译后的目标文件爱你.默认为Debug模式, 也可以使用`--release`参数使用发布模式.

### 关于`Cargo.toml`

`[package]` 包含了包的信息,名字,版本,作者,rust edition
`[dependencies]` 是包的依赖的相关库的,如果我们添加一个`regex`的依赖则是:

```toml
[dependencies]
regex = "1.9.1"
```

也可以直接使用git 仓库地址

```toml
[dependencies]
rand = { git="https://github.com/rust-lang-nursery/rand" }
```

也可以使用path来指定本地包

```toml
[dependencies]
xxx = { path ="xxx" }
```

rust 包使用的是语义化版本号,格式为:"X.Y.Z"

- X, 主版本,做了不兼容或者颠覆性更新
- Y, 次版本号,做了向下兼容的功能性修改
- Z, 修订版本号, 做了向下兼容性的问题修正

依赖包中指定依赖的版本号范围有以下几种:
`^`:允许新版本号在不修改`X.Y.Z`中最左边`X`非零数字的情况下才能更新
`*`: 允许在`X.Y.Z` 的任何一个上面
`~`: 允许修改`X.Y.Z`中没有明确指定的版本号
`手动指定`: 使用 `＞、＞=、＜、＜=、=`来指定版本号

实际的例子:

- `^1.2.3` 表示 `>=1.2.3` `<2.0.0`
- `^1.2` 表示 `>=1.2.0` `<2.0.0`
- `^1` 表示 `>=1.0.0` `<2.0.0`
- `^0.2.3` 表示 `>=0.2.3` `<0.3.0`
- `^0.0.3` 表示 `>=0.0.3` `<0.0.4`
- `^0` 表示 `>=0.0.0` `<1.0.0`
- `1.*` 表示 `>=1.0.0` `<2.0.0`
- `1.2.*` 表示 `>=1.2.0` `<1.3.0`
- `~1.2.3` 表示 `>=1.3.4` `<1.3.0`
- `~1.2` 表示 `>=1.2.0` `<1.3.0`

`Cargo.toml` 中可以使用`[workspace]` 设置工作空间,如:

```toml
[workspace]
members = [
    "bin",
    "app",
    "config",
    "domain",
    "migration"
]
```

工作空间是指在同一个根包下包含多个子包.每个子包有自己的Cargo.toml配置, 各自独立,互不影响.包括跟包的`Cargo.toml`也不会影响其他子包的依赖.不过整个工作空间只允许有一个`Cargo.lock`文件

通过 `Cargo.toml` 可以通过`[dev-dependencies]`来设置 tests, examples, benchmarks 的依赖,即执行`cargo test` 或者 `cargo bench`时使用的依赖

## 模块系统

在单个文件中,可以使用`mod`来声明一个模块.
在Rust中单个文件同时也是一个默认的模块,文件名就是模块名.每个包都拥有一个顶级模块`src/lib.rs` 或者 `src/main.rs`

如果我们在`lib.rs` 定义所有的模块信息

```rust
mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}

        fn seat_at_table() {}
    }

    mod serving {
        fn take_order() {}

        fn serve_order() {}

        fn take_payment() {}
    }
}
```

模块可以嵌套,这里我们在 `front_of_house` 模块里嵌套了两个模块`hosting` 和 `serving`, 但是在实际项目中我们很少这样做,更多的是通过不同的文件处理:

```bash
./src
├── front_of_house
│   ├── hosting
│   │   └── mod.rs
│   ├── mod.rs
│   └── serving
│       └── mod.rs
└── lib.rs
```

lib.rs的内容为:

```rust
pub mod front_of_house;
```

`front_of_house/mod.rs` 中内容为:

```rust
pub mod hosting;
pub mod serving;
```

### 用路径引用模块

绝对路径: 从包根开始,路径名以包名或者 `crate` 作为开头
相对路径: 从当前模块开始, 以 `self`,`super` 或当前模块的标识符作为开头.

```rust
mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}
    }
}

pub fn eat_at_restaurant() {
    // 绝对路径
    crate::front_of_house::hosting::add_to_waitlist();

    // 相对路径
    front_of_house::hosting::add_to_waitlist();
}
```

而在我们拆分的不同文件之后我们如果在根目录下main.rs想要使用,则需要如下:

```rust
use hello::front_of_house::hosting;

fn main() {
    hosting::add_to_waitlist()
}
```

这里的`hello` 是因为最开始我是使用`cargo new hello`创建的项目

### 使用super引用模块

`super` 代表的是父模块为开始的引用方式

```rust
fn serve_order() {}

// 厨房模块
mod back_of_house {
    fn fix_incorrect_order() {
        cook_order();
        super::serve_order();
    }

    fn cook_order() {}
}
```

### 使用self引用模块

self 其实就是引用自身模块中的项:

```rust
fn serve_order() {
    self::back_of_house::cook_order()
}

mod back_of_house {
    fn fix_incorrect_order() {
        cook_order();
        crate::serve_order();
    }

    pub fn cook_order() {}
}
```
