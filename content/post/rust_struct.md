---
title: "Rust笔记----结构体"
date: 2023-07-26T17:44:10+08:00
tags: ["Rust"]
categories: ["Rust"]
keywords: ["rust","struct", "结构体"]
draft: false
---

Rust中有3种结构体类型: named-field结构体,tuple-like结构体,unit-like结构体.

结构体中规范:

- 结构体的名称需要使用驼峰拼写法(CameCase)
- 结构体的字段或者属性值需要使用蛇形拼写法(snake_case)

Rust结构体默认是私有的,只在声明它的模块中可见, 如果需要对模块外可见,需要加上`pub`关键字,结构体中的字段同样默认也是私有的.如果需要模块外可见同样使用`pub`关键字

## 三种结构体

### named-field结构体

```rust
struct Person {
    name: String,
    age: u32,
    email: String,
}

fn main() {
    let user = Person {
        name: "fan-tasitc".to_string(),
        age: 18,
        email: "fan-tasitc@xx.com".to_string(),
    };

    println!(
        "name: {} age: {}, email: {}",
        user.name, user.age, user.email
    );

    let name = "fan".to_string();
    let age = 11;
    // 当变量名和结构体字段名相同时可以使用下面简写的方式
    let u: Person = Person {
        name,
        age,
        email: "fan@xx.com".to_string(),
    };
    println!("name: {} age: {}, email: {}", u.name, u.age, u.email);
    // ..u 表示让 p 借用或者拷贝 u 的某些字段,如下面这种方式
    let p: Person = Person {
        name: "cc".to_string(),
        email: "cc@xx.com".to_string(),
        ..u
    };
    println!("name: {} age: {}, email: {}", p.name, p.age, p.email);
}
```

上面通过`..base`时,如果借用 base的字段是可Copy的,那么在借用时会自动Copy,否则,在借用时会将base中字段的所有权转移走,使得base中的该字段无效.

### tuple-like 结构体

tuple-lik组结构体如同名字一样,和元组非常类似.

```rust
struct Color(i32, i32, i32); 
struct Point(i32, i32, i32); 

let black = Color(0, 0, 0); 
let origin = Point(0, 0, 0);
```

tuple-lik结构体中个别的元素可以是公有的，也可以不是: `pub struct Color(pub i32,pub i32, i32);`

### unit-like 结构体

unit-like struct 是没有任何字段的空struct.
这种类型的值不占内存，非常像基元类型 `()`

## impl 定义方法

`impl` 块只是 `fn` 定义的集合, 这些函数都是如下例子中Student 这个结构体类型的方法.
这里的方法也叫做关联函数 (associated funtion),即是与特定类型关联.
下面代码中new 函数,不将`self` 作为参数的方法,这样的方法与结构体类型本身而非该类型的值关联的函数,在rust中叫做静态方法.通常用于定义构造结构体函数.

```rust
pub struct Student {
    name: String,
    age: u32,
}

impl Student {
    pub fn new(name: String, age: u32) -> Student {
        Student { name, age }
    }
    pub fn say(&self) {
        println!("say hello");
    }

    pub fn hello(&self) {
        println!("my name is {} my age is {}", self.name, self.age);
    }
}
```

Struct的实例方法的第一个参数都是`self`(的不同形式):

- `fn f(self)`：当`obj.f()`时，转移 obj的所有权，调用f方法之后，obj 将无效
- `fn f(&self)`：当`obj.f()`时，借用而非转移obj的只读权，方法内部不可修改obj的属性，调用f方法之后，obj依然可用
- `fn f(&mut self)`：当`obj.f()`时，借用obj的可写权，方法内部可修改obj的属性，调用f方法之后，obj依然可用
