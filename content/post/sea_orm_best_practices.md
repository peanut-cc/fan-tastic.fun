---
title: "sea-orm 最佳实践"
date: 2022-09-29T19:24:18+08:00
draft: true
---

## 希望解决的问题

对比了 `Rust` 中的几个ORM， 最后选择了`sea-orm` 作为学习 `Rust web` 开发的一个入口。
那么问题来了，`sea-orm` 在项目中最佳的使用方法？
对于自己的需求：

- 希望通过 `sea-orm` 生成数据库表结构
- 希望通过`sea-orm` 维护 `migration`记录
- 希望通过 `sea-orm` 的 表结构（Model）变动时，可以在 `migration` 时添加一下自定义数据操作

