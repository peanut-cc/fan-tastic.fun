---
title: "Rust笔记----泛型"
date: 2023-07-27T22:06:57+08:00
tags: ["Rust", "泛型"]
categories: ["Rust"]
keywords: ["rust","泛型","generic"]
draft: false
---

泛型 是一种参数化多态.
对于Rust来说,在编译阶段，泛型会被替换为它代表的数据类型, 所以一个函数可能会变成多个具体数据类型的函数,这种膨胀会导致编译后的程序文件变大.不过好好处就是运行时不用消消耗额外的资源计算泛型所代表的具体类型.

在Struct、Enum、impl、Trait等地方都可以使用泛型.

struct 使用泛型:

```rust
struct container_named<T: std::fmt::Display> {
  field: T,
}
```

Enum使用泛型:

```rust
enum Option<T> {
  Some(T),
  None,
}
```

impl实现类型的方法时使用泛型:

```rust
struct Container<T>{
  item: T,
}

// impl后的T是声明泛型T
// Container<T>的T对应Struct类型Container<T>
impl<T> Container<T> {
  fn new(item: T) -> Self {
    Container {item}
  }
}
```

Trait使用泛型:

```rust
// 表示将某种类型T转换为当前类型
trait From<T> { 
  fn from(T) -> Self; 
}
```

某数据类型impl实现Trait时使用泛型：

```rust
use std::fmt::Debug;

trait Eatable {
  fn eat_me(&self);
}

#[derive(Debug)]
struct Food<T>(T);

impl<T: Debug> Eatable for Food<T> {
  fn eat_me(&self) {
    println!("Eating: {:?}", self);
  }
}
```

上面impl时指定了`T: Debug`,它表示了`Food<T>`类型的T必须实现了`Debug`
通常,应当尽量不在定义类型时限制泛型的范围,除非确实有必要去限制.这是因为,泛型本就是为了描述更为抽象、更为通用的类型而存在的,限制泛型将使得类型变得更具体化,适用场景也更窄.但是在impl类型时,应当去限制泛型,并且遵守缺失什么功能就添加什么限制的规范,这样可以使得所定义的方法不会过度泛化.也不会过度具体化.

## 泛型的基本使用

通过泛型系统,可以减少很多冗余代码.

```rust
use std::ops::Add;
fn double<T>(i: T) -> T
where
    T: Add<Output = T> + Clone + Copy,
{
    i + i
}

fn main() {
    println!("{}", double(12_u32));
    println!("{}", double(10_i32));
    println!("{}", double(100_i64));
}
```

函数名称后面的`<T>`表示在函数作用域内定义一个泛型T.这个泛型只能在函数签名和函数体内使用
参数部分`i: T`表示参数i的类型是泛型T
返回值部分`-> T`表示该函数的返回值类型是泛型T

这里我们希望的是double 函数是对数值进行加法操作,而泛型表示的是各种类型,所以这里需要对泛型T进行限制:

- 在double的函数体内需要对泛型T的值i进行加法操作,及实现了 `std::ops::Add` 类型
- 同时限制泛型T实现了Clone,Copy trait

这种限制泛型就是在trait中提到的 `trait Bound`.

## Trait Object 和 泛型

- Trait Object 可以被看作一种数据类型,它总是以引用的方式被使用,在运行期间,它在栈中保存了具体类型的实例数据和实现自该Trait的方法
- 泛型不是一种数据类型,它可被看作是数据类型的参数形式或抽象形式,在编译期间会被替换为具体的数据类型
- Trait Objecct方式也称为动态分派(dynamic dispatch),它在程序运行期间动态地决定具体类型.而Rust泛型是静态分派
