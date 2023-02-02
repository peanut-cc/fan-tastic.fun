---
title: "《Zero To Production In Rust》笔记(一) ---- 开发前期准备"
date: 2023-02-01T14:29:37+08:00
draft: false
tags: ["rust"]
categories: ["《Zero To Production In Rust》笔记"]
---

这本书和之前看过的一些书籍不一样，在最开始的部分，从开发流程到常用Rust小工具的说明都写的非常清楚，包括后面的CI持续集成。是可以通过书中的完整项目好好学一下Rust web的开发流程。

## 开发流程

在最开始的部分，作者说了这个项目的开发流程：

- Make a change
- Compile the application
- Run tests
- Run the application

这个是自己需要改进的地方，应该注重测试代码的编写。

## 编译速度优化

对于比较大的Rust项目，编译速度可能会比较慢，这里是可以通过一些方法进行优化：

- lld on Windows and Linux, a linker developed by the LLVM project;
- zld on MacOS

对于Windows需要安装的相关依赖：
`cargo install -f cargo-binutils`
对于Linuxx需要安装的相关依赖:
`apt install lld clang` or `pacman -S lld clang`
对于Mac需要安装的相关依赖：
`brew install michaeleisel/zld/zld`

在项目中使用需要在项目的.cargo/config.toml中添加相关配置为:

```toml
# mac
[target.x86_64-apple-darwin]
rustflags = ["-C", "link-arg=-fuse-ld=/usr/local/bin/zld"]

[target.aarch64-apple-darwin] rustflags = ["-C", "link-arg=-fuse-ld=/usr/local/bin/zld"

# windows
[target.x86_64-pc-windows-msvc] rustflags = ["-C", "link-arg=-fuse-ld=lld"] [target.x86_64-pc-windows-gnu] rustflags = ["-C", "link-arg=-fuse-ld=lld"]

# linux
[target.x86_64-unknown-linux-gnu] rustflags = ["-C", "linker=clang", "-C", "link-arg=-fuse-ld=lld"]

```

### cargo 子命令工具推荐

Rust 项目的依赖管理是可以通过cargo子命令方便的管理的, 安装cargo-edit 工具`cargo install cargo-edit`
如添加依赖可以使用`cargo add actix-web`, 或者使用 `cargo add actix-web --vers 4.0.0` 指定版本，执行之后可以在项目的Cargo.toml文件中看到

```toml
[dependencies]
actix-web = "4.3.0"
```

## 关于持续集成

在项目开发中比较好的做法是，在基于main分支的开发中，我们应该能够在任何时间节点部署我们的main分支。
团队的的每个人都可以基于main分支checkout新的分支开发新的功能或者fix bug 并合并到main.
CI 可以帮助我们很好的检查我们提交的代码，在CI 中我们做代码的lint 检查，test 的检查等等，从而保证我们的代码的正确性。

比较好的CI中应该包含如下几部分：

- Tests
- Code Coverage
- Code Linting
- Formatting
- Security Vulnerabilities

对于Test 来说，Rust也非常方便：`cargo test` 即可以运行单元测试和集成测试。

对于 Code Coverage,`https://github.com/xd009642/tarpaulin` 工具也提供非常方便的方法，通过`cargo install cargo-tarpaulin` 安装, 使用 `cargo tarpaulin --ignore-tests`

对于 Code Linting, linter 将尝试发现不合常理的代码、过于复杂的结构和常见的错误。Rust 团队维护着 clippy（<https://github.com/rust-lang/rust-clippy>） ，即官方的 Rust linter，如果没有安装也可以通过命令快速安装：`rustup component add clippy`，使用：`cargo clippy`。
注意：clippy 的静态代码分析并非万无一失的，clippy 可能会提示可能不正确，这种情况可以在受影响的代码上使用`#[allow(clippy::lint_name)]`来取消对该代码的错误提示，或者使用clippy.toml中配置全局的配置。

对于Formatting， rustfmt 是官方的 Rust 格式化程序。就像 clippy 一样，rustfmt 包含在 rustup 安装的默认组件集中。如果没有安装或者丢失，可以通过 `rustup component add rustfmt` 安装。使用： `cargo fmt`, 在CI 中 使用： `cargo fmt -- --check`

对于 Security Vulnerabilities，Rust 安全代码工作组维护着一个数据库——在 crates.io 上发布的 crates 报告漏洞的最新集合。提供了 cargo-audit (<https://github.com/rustsec/rustsec/tree/main/cargo-audit>) 这是一个方便的 cargo 子命令，用于检查项目依赖树中是否报告了漏洞。可以通过 `cargo install cargo-audit` 进行安装，使用`cargo audit`。社区中也有一个更加强大的工具<https://github.com/EmbarkStudios/cargo-deny>

作者提供了常用CI 配置文件例子：

- github action: <https://gist.github.com/LukeMathWalker/5ae1107432ce283310c3e601fac915f3>
- gitlab ci: <https://gist.github.com/LukeMathWalker/d98fa8d0fc5394b347adf734ef0e85ec>
