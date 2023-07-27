---
title: "Rust笔记----Trait"
date: 2023-07-26T21:06:00+08:00
tags: ["Rust", "Trait"]
categories: ["Rust"]
keywords: ["rust","Trait","Trait object"]
draft: false
---

trait 是 Rust 中的一个非常重要的概念, 在面向对象语言中一般都是叫做接口(interface).
trait 是一种方法的集合,或者是一种行为的集合.

在rust中,trait是在行为上对类型的约束, 这种约束让trait有如下一些常用用法:

- 接口抽象: 接口是对类型行为的统一约束
- 泛型约束: 泛型的行为被trait限定在更有限的范围内
- 抽象类型: 在运行时作为一种间接的抽象类型去使用,动态的分发给具体的类型

```rust
trait Animal {
    fn eat(&self);
    fn sleep(&self);
}

struct Dog;

impl Animal for Dog {
    fn eat(&self) {
        println!("dog is eating")
    }

    fn sleep(&self) {
        println!("dog sleeping")
    }
}

struct Cat;

impl Animal for Cat {
    fn eat(&self) {
        println!("cat is eating")
    }

    fn sleep(&self) {
        println!("cat is sleeping")
    }
}
```

如上面的代码,trait 只包含函数的签名,即参数与参数类型,返回值类型,但是没有函数体.
给具体的类型实现trait时,通过`impl Trait for Type {}` 即对应代码中 `impl Animal for Cat{}` 和 `impl Animal for Dog{}`.

## 接口抽象

如同上面代码中`Animal` trait,就是一种接口抽象, Cat 和 Dog 实现了该 trait,但是具体的行为各不相同. 为不同的类型实现 trait, 属于一种函数重载.

### 关联类型

标准库中 Add trait的 定义:

```rust
pub trait Add<Rhs = Self> {
    type Output;

    // Required method
    fn add(self, rhs: Rhs) -> Self::Output;
}
```

代码中的`type Output;` 这种方式定义的类型叫做关联类型
`Add<Rhs = Self>` 表示类型参数Rhs 指定了默认值Self, Self是每个trait 都带的隐式类型参数,代表实现当前trait的具体类型.

我们自己可以为一些类型实现这个Add trait

```rust
pub trait Add<Rhs = Self> {
    type Output;

    // Required method
    fn add(self, rhs: Rhs) -> Self::Output;
}

// 未指定参数类型
impl Add for u32 {
    type Output = u32;

    fn add(self, rhs: Self) -> Self::Output {
        self + rhs
    }
}

// 指定参数类型为 &str
impl Add<&str> for String {
    type Output = String;

    fn add(mut self, rhs: &str) -> String {
        self.push_str(rhs);
        self
    }
}
```

实现Add trait时并未指明泛型参数的具体类型，则默认为Self类型，也就是u32类型
`impl Add＜&str＞`指明了泛型类型为`&str`，并没有使用Self默认类型参数,这表明对于String类型字符串来说，加号右侧的值类是`&str`类型，而非String类型

### trait 继承

Rust 支持 trait继承

```rust
trait Man {
    fn eat(&self, food: String) {
        println!("eating {}", food);
    }
}

trait Superman {
    fn fly(&self) {
        println!("flying");
    }
}

struct People;

impl Man for People {}
impl Superman for People {}

fn main() {
    let p = People;
    p.eat("apple".to_string());
    p.fly();
}
```

Man 和 Superman 两个trait 中分别有两个默认的方法,在此基础上,如果我们想要实现扩展,可以使用继承的方式,如:

```rust
trait Man {
    fn eat(&self, food: String) {
        println!("eating {}", food);
    }
}

trait Superman {
    fn fly(&self) {
        println!("flying");
    }
}

trait Coder: Man + Superman {
    fn write_code(&self, code: String) {
        println!("writing {} code ", code);
    }
}

struct People;

impl Man for People {}
impl Superman for People {}
impl<T: Man + Superman> Coder for T {}

fn main() {
    let p = People;
    p.eat("apple".to_string());
    p.fly();
    p.write_code("rust".to_string())
}
```

## 泛型约束

```rust
use std::ops::Add;

fn sum<T: Add<T, Output = T>>(a: T, b: T) -> T {
    a + b
}

fn main() {
    assert_eq!(sum(1u32, 2u32), 3);
    assert_eq!(sum(1u64, 2u64), 3);
}
```

通过使用 `<T: Add<T, Output = T>>` 对泛型进行约束,表示sum函数的参数必须实现Add trait,且加号两边的类型必须一致
在对T进行泛型约束时, `Add<T, Output = T>` 通过类型参数确定了关联类型Output 也是T,这里是可以省略直接写成 `Add＜Output=T＞`
这种通过trait 对泛型进行约束,叫做 `trait Bound`

rust 中 `trait Bound` 属于静态分发,在编译器通过单态化分别生成具体类型的实力,所以调用`trait Bound`中的方法也都是运行时零成本的.
如果我们在对泛型有比较多的 `trait Bound` 可以会导致看起来比较复杂,rust 提供了简单的方法:

`fn foo<T:A, K: B+C, R:D>(a:T, b:K, c:R){...}`

通过where 方式可以改为:

```rust
fn foo<T,K,R>(a:T, b:K, c:R)
    where T:A, K:B+C, R:D {

    }
```

## 抽象类型

```rust
#[derive(Debug)]
struct Foo;

trait Bar {
    fn baz(&self);
}

impl Bar for Foo {
    fn baz(&self) {
        println!("{:?}", self);
    }
}

fn static_dispatch<T>(t: &T)
where
    T: Bar,
{
    t.baz();
}

fn dynamic_dispatch(t: &dyn Bar) {
    t.baz();
}

fn main() {
    let foo = Foo;
    static_dispatch(&foo);
    dynamic_dispatch(&foo);
}
```

trait 本身也是一种类型, 但是它的类型在编译期是无法确定的,所以在`fn dynamic_dispatch(t: &dyn Bar)`, 使用的是 `&dyn Bar`
`dyn Bar` 表示是 `trait Bar`的 `trait Object`类型.由于 `trait Object` 大小不固定, 所以基本都是使用引用的方式 `&dyn Bar`.
`Trait Object`的引用保存在栈中,包含两份数据：`Trait Object`所指向数据的指针和指向一个虚表vtable的指针.vtable中保存了具体类型的实例对于可以调用的实现于trait的方法,及上述的`baz`方法.当调用方法时,直接从vtable中找到方法并调用.

简单来说,当类型B实现了`Trait A`时,类型B的实例对象b可以当作A的`Trait Object`类型来使用,b中保存了作为`Trait Object`对象的数据指针(指向B类型的实例数据)和行为指针(指向vtable).

## 相关链接

- <https://doc.rust-lang.org/std/ops/trait.Add.html>
