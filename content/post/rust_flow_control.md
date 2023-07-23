---
title: "Rust笔记----流程控制"
date: 2023-07-23T17:49:29+08:00
tags: ["Rust"]
categories: ["Rust"]
draft: false
---

rust 中流程控制结构都是表达式,表达式就一定会有值,默认的返回值是单元值`()`.

## 条件表达式

if语句的语法如下：

```rust
if COND1 {
  ...
} else if COND2 {
  ...
} else {
  ...
}
```

需要注意: COND 只能是布尔类型

表达式就一定会有值,if 表达式的分支必须返回同一个类型的值才可以.

```rust
fn is_available(a: i32) -> bool {
    if a > 10 {
        true
    } else {
        false
    }
}
```

必须要有else分支,否则如果所有的条件盘大u能都不通过返回默认值`()`.

```rust
fn main() {
    let x = 20;
    let a = if x < 10 {
        x + 10
    } else if x < 30 {
        x + 30
    } else {
        x
    }; // 这是let的分号
    println!("{}", a);
}
```

但是下面这种情况就会导致错误:

```rust
fn main() {
    let x = 20;
    let a = if x < 10 {
        x + 10
    }; // 这是let的分号
    println!("{}", a);
}
```

if分支返回i32类型的值, 但如果没有执行if分支，则返回默认值`()` 这使得a的类型不是确定的，因此报错.

## 循环表达式

Rust 中的循环表达式有: `while`, `loop`, `for...in`.

### while

while循环的语法：

```rust
while COND {
  ...
}
```

条件表达式COND和if结构的条件表达式规则完全一致,必须是布尔类型.
和其他语言一样,退出循环使用`break` 关键字, 进入下次循环 使用 `continue` 关键字

```rust
fn main() {
    let mut x = 10;
    while x > 0 {
        println!("{}", x);
        x -= 1;
    }
}
```

### loop

`loop` 表达式是一个无限循环,只有在`loop`循环体内使用`break`才能终止循环.

```rust
fn main() {
    let mut x = 0;
    loop {
        x += 1;
        if x == 5 {
            break;
        }
        if x % 2 == 0 {
            continue;
        }
        println!("{}", x);
    }
}
```

loop循环中使用`break` 和其他循环中不同,只有在loop 中 break才能指定返回值.

```rust
fn main() {
    let mut x = 0;
    let a = loop {
        x += 1;
        if x == 5 {
            break x; // 返回跳出循环时的x，并赋值给变量a
        }
        if x % 2 == 0 {
            continue;
        }
        println!("{}", x);
    };
    println!("var a: {:?}", a); // 输出 var a: 5
}
```

### for

`for...in` 本质上是一个迭代器.迭代是一种特殊的循环，每次从数据的集合中取出一个元素是一次迭代过程，直到取完所有元素，才终止迭代.

```rust
fn main() {
    for i in 1..5 {
        println!("{}", i);
    }

    let arr = ["fan-tastic", "fan", "cc"];
    for i in arr {
        println!("{}", i);
    }
}
```

## match 表达式与模式匹配

```rust
fn main() {
    let number = 30;
    match number {
        0 => println!("this is 0"),
        1..=9 => println!("this is in 1-9"),
        11 | 15 | 17 => println!("this is in 11, 15,17"),
        n @ 30 => println!("this is {}", n), // @ 可以将模式中的值绑定给一个变量,这种叫绑定模式
        _ => println!("other"),
    }
}
```

match 分支的左边就是模式, 右边就是执行代码. 和if 表达式类似, 所有分支必须返回同一个类型.
match 分支必须穷尽每一种可能,所以一般会使用 `_` 来处理剩余的所有情况.

## if let 和 while let表达式

if let 表达式只是对只有一个模式的 match 的简写.

```rust
if let pattern = expr {
    block1
} else {
    block2
}
```

这里的 expr 要么匹配 pattern,运行 block1,要么不匹配 pattern, 运行 block2, else不是必须的.

```rust
fn main() {
    let b = true;
    let mut number = 0;
    if let true = b {
        number = 1;
    }
    assert_eq!(number, 1);
}
```

while let 语法:

```rust
while let pattern = expr {
    block
}
```

在每次循环迭代开始时, expr 的值要么匹配给定的 pattern,运行后面的块,要么不匹配给定的 pattern,退出循环.

```rust
fn main() {
    let mut v = vec![1, 2, 3, 4];
    while let Some(x) = v.pop() {
        println!("{}", x);
    }
}
```
