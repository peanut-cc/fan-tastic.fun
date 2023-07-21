---
title: "从零搭建Rust语言开发环境"
date: 2023-07-20T22:39:38+08:00
tags: ["Rust", "rust安装"]
categories: ["Rust"]
keywords: ["rust","rust环境安装", "rust安装"]
draft: false
---

## 安装Rust

官网地址: <https://www.rust-lang.org/learn/get-started>

这里`Linux`环境为例:

```bash
root@iZ2ze7nnhgeigtbnvj4av2Z:~# curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
info: downloading installer

Welcome to Rust!

......
......
You can uninstall at any time with rustup self uninstall and
these changes will be reverted.

Current installation options:


   default host triple: x86_64-unknown-linux-gnu
     default toolchain: stable (default)
               profile: default
  modify PATH variable: yes

1) Proceed with installation (default)
2) Customize installation
3) Cancel installation
>

info: profile set to 'default'
......
......

Rust is installed now. Great!

To get started you may need to restart your current shell.
This would reload your PATH environment variable to include
Cargo's bin directory ($HOME/.cargo/bin).

To configure your current shell, run:
source "$HOME/.cargo/env"
root@iZ2ze7nnhgeigtbnvj4av2Z:~# 
```

安装好撞击后我就有了如下几个常用的命令:

- cargo: rust的编译管理器,包管理器. 可以用于创建新项目,编译和运行程序,以及管理代码依赖的外部库.
- rustc: rust的编译器,不过通常我们都是使用cargo 编译项目.
- rustup: 如果之前已经安装了`Rust`,通过`rustup update`也可以很方便的更新`Rust`的版本.

可以通过`cargo version` 查看自己当前的版本:

```bash
root@iZ2ze7nnhgeigtbnvj4av2Z:~# cargo version
cargo 1.71.0 (cfd3bbd8f 2023-06-08)`
```

### Hello World

通过`cargo new hello` 创建一个hello 的项目

```bash
hello/
├── Cargo.toml
└── src
    └── main.rs
```

项目的根目录中:

- `Cargo.toml`: 描述项目的元数据,如项目名称,项目的版本,以及依赖.
- `src`: src目录用于存放项目的源代码.

通过`cargo run` 即可运行程序:

```bash
root@iZ2ze7nnhgeigtbnvj4av2Z:/opt/dev/hello# cargo run
   Compiling hello v0.1.0 (/opt/dev/hello)
    Finished dev [unoptimized + debuginfo] target(s) in 2.08s
     Running `target/debug/hello`
Hello, world!
root@iZ2ze7nnhgeigtbnvj4av2Z:/opt/dev/hello# 
```

## cargo 常用命令

- `cargo version`: 显示版本信息
- `cargo new`: 创建一个新项目
- `cargo test`: 运行项目中的单元测试
- `cargo fmt`: 自动格式化代码
- `cargo build`: 编译项目代码
- `cargo run`: 运行项目代码
- `cargo check`: 快速编译项目，无需生成二进制文件来检查错误

更详细的可以看: <https://doc.rust-lang.org/stable/cargo/index.html>

### cargo 扩展

#### Faster Linking

在我们实际项目开发中,我们会不断的重复如下步骤

- 增加代码
- 编译代码
- 运行测试
- 运行程序

而在这个过程中我们非常关注增量编译的性能,在我们进行一些小的改动之后,需要多久才能编译出新的二进制
在编译大型项目时,`Rust` 的编译速度可能就会成为我们的痛点,针对这个我们可以通过一些扩展来进行加快编译的速度.

在我们做了代码改动之后,编译的很多时间是花费在链接阶段(<https://en.wikipedia.org/wiki/Linker_(computing)>),即组装二进制文件. 默认的链接器做的也很好,不过我们可以根据不同的系统设置更快的链接器:

- lld on Windows and Linux, a linker developed by the LLVM project;
- zld on MacOS.

Ubuntu: `apt-get install lld clang`
Arch: `pacman -S lld clang`
MacOS: `brew install michaeleisel/zld/zld`
Windows: `cargo install -f cargo-binutils`, `rustup component add llvm-tools-preview`

Rust中使用,需要在项目中创建文件`.config/config.toml`

```toml
[target.x86_64-pc-windows-msvc]
rustflags = ["-C", "link-arg=-fuse-ld=lld"]
[target.x86_64-pc-windows-gnu]
rustflags = ["-C", "link-arg=-fuse-ld=lld"]


[target.x86_64-unknown-linux-gnu]
rustflags = ["-C", "linker=clang", "-C", "link-arg=-fuse-ld=lld"]

[target.x86_64-apple-darwin]
rustflags = ["-C", "link-arg=-fuse-ld=/usr/local/bin/zld"]
[target.aarch64-apple-darwin]
rustflags = ["-C", "link-arg=-fuse-ld=/usr/local/bin/zld"]
```

同时最近有另外一个据称比 `lld` 更快的链接器 `mold`(<https://github.com/rui314/mold>)
rust使用的方式和`lld`一样,也是在`.config/config.toml`根据自己的系统添加如下内容:

```toml
[target.x86_64-unknown-linux-gnu]
rustflags = ["-C", "link-arg=-fuse-ld=/path/to/mold"]
```

#### cargo-watch

用于监控当前项目的文件的变化,并执行`cargo`的相关命令,可以通过如下命令安装

`cargo install cargo-watch`

在实际项目开发中可以根据自己的需要来设置要执行的命令,如:

- `cargo watch -x check`: 每次文件的变动就会触发执行 `cargo check`
- `cargo watch -x check -x test -x run`: 每次文件变动触发执行`cargo check`, `cargo test`, `cargo run`

#### Linting

在安装`Rust` 时如果是默认安装,clippy 包含在`rustup` 安装的组件中.
如果没有安装也可以通过`rustup component add clippy` 安装.
可以在项目目录使用
`cargo clippy`

如果 `Clippy` 发出任何警告，我们希望 linter 检查失败。
`cargo clippy -- -D warnings`

## 更换源

可以通过更换国内源来加快依赖的下载速度,添加配置文件并编辑

```bash
touch ~/.cargo/config
vim ~/.cargo/config
```

配置文件内容为:

```bash
[source.crates-io]
registry = "https://github.com/rust-lang/crates.io-index"
# 指定镜像
replace-with = 'tuna' # 如：tuna、sjtu、ustc，或者 rustcc

# 注：以下源配置一个即可，无需全部
# 中国科学技术大学
[source.ustc]
registry = "git://mirrors.ustc.edu.cn/crates.io-index"

# 上海交通大学
[source.sjtu]
registry = "https://mirrors.sjtug.sjtu.edu.cn/git/crates.io-index"

# 清华大学
[source.tuna]
registry = "https://mirrors.tuna.tsinghua.edu.cn/git/crates.io-index.git"

# rustcc社区
[source.rustcc]
registry = "https://code.aliyun.com/rustcc/crates.io-index.git"
```

## IDE

`IDE`的选择可以根据自己的习惯, 不管是 `JetBrains`家族的 `IntelliJ` 还是`vscode` 都可以.
我个人写`Rust` 更喜欢用 `vscode` + `rust-analyzer` 的方式.
如果你使用的是`IntelliJ`,则在插件里安装 `rust` 插件即可.

## 相关链接

- <https://www.rust-lang.org/learn/get-started>
- <https://doc.rust-lang.org/stable/cargo/index.html>
- <https://lld.llvm.org/>
- <https://github.com/llvm/llvm-project>
- <https://github.com/rui314/mold>
