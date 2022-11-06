---
title: "Rust的Struct和Enum"
date: 2022-11-06T16:45:51+08:00
tags: ["Rust"]
categories: ["Rust"]
keywords: ["rust","struct", "enum"]
draft: false
---

## Struct类型

```rust
struct Person {
    name:String,
    age: u32,
    email: String,
}


fn main() {
    let user = Person {
        name:"fan-tasitc".to_string(),
        age:18,
        email: "fan-tasitc@xx.com".to_string(),
    };

    println!("name: {} age: {}, email: {}", user.name, user.age, user.email);
}
```

构造 struct 的时候时可以有一些简写方法：

```rust
struct Person {
    name:String,
    age: u32,
    email: String,
}


fn main() {
    let name = "fan-tastic".to_string();
    let age = 18;
    let user = Person {
        name,
        age,
        email: "fan-tasitc@xx.com".to_string(),
    };
    let user2 = Person {
        name: "peanut".to_string(),
        email: "peanut@xx.com".to_string(),
        ..user
    };
    println!("name: {} age: {}, email: {}", user.name, user.age, user.email);
    println!("name: {} age: {}, email: {}", user2.name, user2.age, user2.email);
}
```

当变量的名称是struct 的属性名一致时，构造时可以省略写属性名称。
当一个结构体的数据某些属性和另外一个实例一样时，可以通过类似上面`..user`的方式构造结构体，表示user2实例的剩余字段使用user 实例中的。

不过上面这种内用另外一个结构体实例构造结构体时需要注意：如果借用的那个字段时可Copy 的，那么在借用时会自动Copy,这种情况时不会影响被借用的结构体的，但是如果不是可Copy的，那么将在借用时将被借用的结构体中的相关字段的所有权转移走，使得被借用的结构体的相关字段不可用。这种情况可以使用clone() 来避免，如下代码：

```rust
struct Person {
    name:String,
    age: u32,
    email: String,
}


fn main() {
    let name = "fan-tastic".to_string();
    let age = 18;
    let user = Person {
        name,
        age,
        email: "fan-tasitc@xx.com".to_string(),
    };
    let user2 = Person {
        name: "peanut".to_string(),
        ..user.clone() // 使用clone 避免user 中的email 在之后无法使用
    };
    println!("name: {} age: {}, email: {}", user.name, user.age, user.email);
    println!("name: {} age: {}, email: {}", user2.name, user2.age, user2.email);

}
```

元组结构体

```rust
struct Point(i32,i32,i32)
let p = Point(1,2,3);
```

元组结构体实例类似于元组：可以将其解构，也可以使用`.`后跟索引来访问单独的值，等等。

类单元结构体

类单元结构体(unit-like struct)是没有任何字段的空struct。

```rust
struct a;
```

### struct 方法

struct类似面向对象的类一样，Rust 允许为 struct 定义实例方法和关联方法，实例方法是struct实例化对象调用的方法，关联方法类似于其他语言中的类方法或者静态方法。

```rust

struct Person {
    name:String,
}

impl Person {
    // 实例方法
    fn write_code(&self, language: &str){
        println!("name {} can write {} code", self.name, language);
    }

    // 关联方法
    fn new(name: String) -> Self {
        Person { name}
    }
}

fn main() {
    let user = Person::new("fan-tastic".to_string());
    user.write_code("rust")
}
```

Struct的实例方法的第一个参数都是self(的不同形式):

- `fn f(self)`：当`obj.f()`时，转移`obj`的所有权，调用f方法之后，`obj`将无效
- `fn f(&self)`：当`obj.f(`)时，借用而非转移`obj`的只读权，方法内部不可修改`obj`的属性，调用f方法之后，`obj`依然可用
- `fn f(&mut self)`：当`obj.f()`时，借用`obj`的可写权，方法内部可修改`obj`的属性，调用f方法之后，`obj`依然可用

Rust不会自动引用或自动解除引用，但有例外：当使用`.`运算符和比较操作符(如= > >=)时，Rust会自动创建引用和解引用，并且会尽可能地解除多层引用：

- 方法调用`v.f()`会自动解除引用或创建引用
- 属性访问`p.name`或`p.0`会自动解除引用
- 比较操作符的两端如果都是引用类型，则自动解除引用
- 能自动解除的引用包括普通引用`&x`、`Box<T>`、`Rc<T>`等

关联函数，就像上面的例子，调用关联方法的语法·StructName::func()·。

## Enum类型

Enum枚举类型，Rust 的枚举类型逼其他语言的枚举类型更加强大。

```rust
enum Gender {
    Mal,
    Female,
}

fn main() {
    let g1 = Gender::Mal;
    let g2 = Gender::Female;
    println!("{}", g1 as i32); // 0
    println!("{}", g2 as i32); // 1
}
```

定义的枚举类型，其每个成员都有对应的数值。默认第一个成员对应的数值为0，第二个成员的对应的数值为1，后一个成员的数值总是比其前一个数值大1。并且，可以使用=为成员指定数值，但指定值时需注意，不同成员对应的数值不能相同。

可在enum定义枚举类型的前面使用`#[repr]`来指定枚举成员的数值范围，超出范围后将编译错误。

```rust
#[repr(u8)]
enum E  {
    A,
    B,
    C = 2222,
}
```

不过Rust 的 Enum 会更加灵活，你可以如下定义：

```rust
enum E {
    a,
    b(i32, i32),
    c(x: i32, y:f64),
}
```

所以对于Rust 如果实现一个Json 解析工具，定义一个枚举类型去枚举Json允许的数据类型：

```rust
enum Json {
  Null,
  Boolean(bool),
  Number(f64),
  String(String),
  Array(Vec<Json>),
  Object(Box<HashMap<String, Json>>),
}
```

枚举类型还可以和结构体一样定义方法：

```rust
#[derive(Copy, Clone)]
enum Week {
  Monday = 1,
  Tuesday,
  Wednesday,
  Thursday,
  Friday,
  Saturday,
  Sunday,
}

impl Week {
  fn is_weekend(&self) -> bool {
    if (*self as u8) > 5 {
      return true;
    }
    false
  }
}

fn main(){
  let d = Week::Thursday;
  println!("{}", d.is_weekend());
}
```
