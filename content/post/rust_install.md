---
title: "从零搭建Rust语言开发环境"
date: 2023-07-20T22:39:38+08:00
draft: true
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

如果之前已经安装了`Rust`,通过`rustup update`也可以很方便的更新`Rust`的版本.

`cargo` 是`Rust`构建工具和包管理器,安装好之后可以通过`cargo version` 查看自己当前的版本:

```bash
root@iZ2ze7nnhgeigtbnvj4av2Z:~# cargo version
cargo 1.71.0 (cfd3bbd8f 2023-06-08)`
```

## cargo 常用命令

- `cargo version`: 显示版本信息
- `cargo new`: 创建一个新项目
- `cargo test`: 运行项目中的单元测试
- `cargo fmt`: 自动格式化代码
- `cargo build`: 编译项目代码
- `cargo run`: 运行项目代码

更详细的可以看: <https://doc.rust-lang.org/stable/cargo/index.html>

### cargo 扩展

在我们实际项目开发中,我们会不断的重复如下步骤

- 增加代码
- 编译代码
- 运行测试
- 运行程序

而在这个过程中我们非常关注增量编译的性能,在我们进行一些小的改动之后,需要多久才能编译出新的二进制
在编译大型项目时,`Rust` 的编译速度可能就会成为我们的痛点,针对这个我们可以通过一些扩展来进行加速编译的速度.

Faster Linking

在我们做了代码改动之后,编译的很多时间是花费在链接阶段(<https://en.wikipedia.org/wiki/Linker_(computing)>),即组装二进制文件. 默认的链接器做的也很好,不过我们可以根据不同的系统设置更快的链接器:

- lld on Windows and Linux, a linker developed by the LLVM project;
- zld on MacOS.

Ubuntu: `apt-get install lld clang`
Arch: `pacman -S lld clang`
MacOS: `brew install michaeleisel/zld/zld`

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

## 相关链接

- <https://www.rust-lang.org/learn/get-started>
- <https://doc.rust-lang.org/stable/cargo/index.html>
