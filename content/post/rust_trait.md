---
title: "Rust笔记----Trait"
date: 2023-07-26T21:06:00+08:00
tags: ["Rust", "Trait"]
categories: ["Rust"]
keywords: ["rust","Trait","Trait object"]
draft: true
---

trait 是 Rust 中的一个非常重要的概念, 在面向对象语言中一般都是叫做接口(interface).

在rust中,trait是在行为上对类型的约束, 这种约束让trait有如下用法:

- 接口抽象: 接口是对类型行为的统一约束
- 泛型约束: 泛型的行为被trait限定在更有限的范围内
- 抽象类型: 在运行时作为一种间接的抽象类型去使用,动态的分发给具体的类型
- 标签trait: 对类型的约束,可以直接作为一种"标签"使用
