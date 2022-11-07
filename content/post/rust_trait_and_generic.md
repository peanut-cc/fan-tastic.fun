---
title: "Rust trait和generic"
date: 2022-11-06T20:25:17+08:00
tags: ["Rust"]
categories: ["Rust"]
keywords: ["rust", "trait", "trait object", "generic", "泛型"]
---

## Trait

从多种数据类型中抽取出这些类型之间可通用的方法或属性，并将它们放进另一个相对更抽象的类型中，是一种很好的代码复用方式，也是多态的一种体现方式。

在面向对象语言中，这种功能一般通过接口(interface)实现，在 Rust 中，这种功能通过`Trait`实现，所以`Trait` 是可以被其他具体的类型实现（implement）,也可以在 `Trait` 中定义一些方法，实现该 `Trait` 的类型都必须实现这些方法。

Rust 中 `Trait` 的作用：
`Trait` 类型：用于定义抽象行为，抽取那些共性的属性，主要表现是作为泛型的数据类型(对泛型进行限制)
`Trait` 对象：即`Trait Object`，能用于多态

```rust
trait Animal {
    fn eat(&self);
    fn sleep(&self);
    fn say(&self) {
        println!("hello");
    }
}

struct People {
    name: String,
    age: i32,
}

// 为 People 实现 Animal trait
impl Animal for People {
    fn eat(&self) {
        println!("{} is eating....", self.name);
    }

    fn sleep(&self) {
        println!("{} sleeping....", self.name);
    }
}

impl People {
    fn get_age(&self) -> i32 {
        self.age
    }
    fn new(name: String, age: i32) -> Self {
        People {
            name,
            age,
        }
    }
}

struct Cat {
    name: String,
}

// 为 Cat 实现 Animal trait
impl Animal for Cat {
    fn eat(&self) {
        println!("{} is eating....", self.name);
    }

    fn sleep(&self) {
        println!("{} sleeping....", self.name);
    }

    fn say(&self) {
        println!("maio miao....");
    }
}

impl Cat {
    fn new(name: String) -> Self {
        Cat {
            name,
        }
    }
}

fn main(){
    let user = People::new("fan-tastic".to_string(), 18);
    user.eat();
    user.sleep();
    user.say();
    let age = user.get_age();
    println!("age is {}", age);

    let miao = Cat::new("xiao bai".to_string());
    miao.eat();
    miao.sleep();
    miao.say();
}
```

`trait Animal` 中 `eat` 和 `sleep` 方法都仅仅只规范了它们的方法签名，并没有定义方法体。`say` 方法则指定了函数签名并定义了方法体，如果实现 `trait Animal` 的对象没有 `say` 方法，则会使用`trait Animal`的方法，也可以自己在对象里覆盖，如 `impl Animal for People` 中的 say 方法。
`impl Animal for Cat` 中只能写和 `trait Animal` 定义的方法，和 trait 无关的方法需要 在单独的 `impl Cat` 中写。 例如上面 new 方法 和  `get_age` 方法。

总结：Trait 描述了一种通用的功能，这种通用功能要求具有某些行为，这种通用功能可以被很多中类型实现，每个实现了这种通用功能的类型都可以称为： “具有该功能的类型”。

一个类型可以实现很多种Trait，那么这个类型就会具有很多公共能，可以调用这些 `Trait` 的方法。

对于开发经常使用的 `Struct` 和 `Enum` 类型需要自己手动去实现需要的`Trait`，有些常用的可以通过 `#[derive()]`实现：

```rust
#[derive(Debug, Copy, Clone)]
struct A {
    num:i32,
    f: f64,
}
```

### Trait继承

Rust 支持 Trait 之间的继承

```rust
trait A {
    fn fun_in_a(&self);
}

trait B: A {
    fn fun_in_b(&self);
}

struct S {}

impl B for S {
    fn fun_in_b(&self) {
        println!("fun in B");
    }
}

impl A for S {
    fn fun_in_a(&self) {
        println!("fun in A");
    }
}

fn main(){
    let s = S{};
    s.fun_in_a();
    s.fun_in_b();
}
```

这种情况需要注意，因为 `trait B: A` 所以 我们需要为 S 实现 `trait B` 和 `trait A` ,这两个需要分开写，并不能将`fn fun_in_a(&self)` 写在 `impl B for S` 中。
个人觉得这种对于阅读还是非常方便，可以很直观的看到当前对象实现哪些 `trait`

### Trait Object

`Trait Object` 是什么，通过一段代码来理解更清楚：

```rust
trait Animal {
    fn eat(&self);
    fn sleep(&self);
    fn say(&self) {
        println!("hello");
    }
}

struct People {
    name: String,
    age: i32,
}

// 为 People 实现 Animal trait
impl Animal for People {
    fn eat(&self) {
        println!("{} is eating....", self.name);
    }

    fn sleep(&self) {
        println!("{} sleeping....", self.name);
    }
}

impl People {
    fn new(name: String, age: i32) -> Self {
        People {
            name,
            age,
        }
    }
    fn get_age(&self) -> i32 {
        self.age
    }
}

struct Cat {
    name: String,
}

// 为 Cat 实现 Animal trait
impl Animal for Cat {
    fn eat(&self) {
        println!("{} is eating....", self.name);
    }

    fn sleep(&self) {
        println!("{} sleeping....", self.name);
    }

    fn say(&self) {
        println!("maio miao....");
    }
}

impl Cat {
    fn new(name: String) -> Self {
        Cat {
            name,
        }
    }
}

fn animal_say(animal: &dyn Animal) {
    animal.say();
}

fn animal_say2(animal: impl Animal) {
    animal.say();
}

fn main(){
    let user = People::new("fan-tastic".to_string(), 18);
    let miao = Cat::new("xiao bai".to_string());
    animal_say(&user);
    animal_say(&miao);
    animal_say2(user);
    animal_say2(miao);
}
```

- Cat 和 People 都实现了 `trait Animal`, 所以 都可以作为 `trait Animal` 的 `trait Object` 来使用，即`trait Object`是Rust 支持的一种数据类型。
- 对于 `trait Animal`， `dyn Animal` 表示 `trait Animal` 的  `trait Object`
- `trait Object` 的大小不固定，所以使用时基本都是引用的方式 `&dyn Animal`
- `trait Object` 的引用保存在栈中，包含两份数据：`trait Object` 所指向的数据的指针 和一个虚表 `vtable`的指针
- 虽然 `trait Object`没有固定大小，但它的引用类型的大小是固定的，它由两个指针组成，因此占用两个指针大小，即两个机器字长
- `vtable` 中保存了实现`trait Animal`对象也可以调用的方法。当调用方法时，直接从vtable中找到方法并调用。之所以要使用一个`vtable`来保存各实例的方法，是因为实现了`trait Animal`的类型有多种，这些类型拥有的方法各不相同，当将这些类型的实例都当作`trait Animal`来使用时(此时，它们全都看作是`trait Animal`类型的实例)，有必要区分这些实例各自有哪些方法可调用
- `Trait Object`的引用方式有多种。例如对于`trait Animal`，其`Trait Object`类型的引用可以是`&dyn Animal`、`Box<dyn Animal>`、`Rc<dyn Animal>`等

这里需要注意，对象一旦被当作 `Trait Object` 使用，如代码中的 `animal_say(&user);` 那么 `user` 将不在是 `People`的实例对象，`user` 中保存了作为 `Trait Object` 对象的数据指针(指向`People`的实例对象)和行为指针（指向vtable）。
同时 `fn animal_say2(animal: impl Animal)` 是会在编译时根据根据实现 `trait Animal` 的对象进行展开

同时是可以在这样写：

```rust
let user2: &dyn Animal = &People::new("fan-tastic".to_string(), 18);
```

但是这样写之后，user2 就只能调用`trait Animal` 定义的方法，无法调用和 ``trait Animal` 无关的实例方法，如代码中的 `get_age` 方法

## Generic

泛型，对于Rust来说，在编译阶段，泛型会被替换为它代表的数据类型, 所以一个函数可能会变成多个具体数据类型的函数，这种膨胀会导致编译后的程序文件变大。不过好好处就是运行时不用消消耗额外的资源计算泛型所代表的具体类型。

泛型 可以用于解决代码冗余的问题：

```rust
use std::ops::Add;

fn double<T>(i:T) -> T
    where T: Add<Output=T> + Clone + Copy
{
     i+i
}

fn double2<T: Add<Output=T> + Clone + Copy>(i:T) -> T {
    i+i
}

fn main(){
    println!("{}", double(3_i32));
    println!("{}", double(3_f64));
    println!("{}", double2(3_f64));

}
```

- `T` 就是泛型，你可以使用你喜欢的字母，这个没有限制，只是用来代表各种可能的数据类型。
- `fn double<T>(i:T) -> T`:
  - 函数名称后面的`<T>`表示在函数作用域内定义一个泛型`T`,这个泛型只能在函数签名和函数体内使用
  - 参数部分`i: T`表示参数i的类型是泛型`T`。
  - 返回值部分`-> T`表示该函数的返回值类型是泛型`T`
- `where T: Add<Output=T> + Clone + Copy` 通过`where` 对泛型`T`添加限制条件，这里表示泛型`T` 必须实现 `Add`, `Copy`, `Clone` 三种`trait`
- double2 和 double 是两种写法，效果是一样的，不过 在比较复杂的的情况时，第一种写法看着更加直观。

对于泛型 做限制的原因：

- 一方面的原因是函数体内需要某种`Trait`提供的功能(比如函数体内要对i执行加法操作，需要的是`std::ops::Add`的功能)
- 另一方面的原因是要让泛型T所能代表的数据类型足够精确化(如果不做任何限制，泛型将能代表任意数据类型)。

泛型的引用：`&T`或`&mut T`。

### 泛型的使用

泛型 可以在很多地方使用，例如  `Struct`、`Enum`、`impl`、`Trait` :

`Struct` 和 `impl`实现中：

```rust
struct S<T> {
    a: T
}

impl<T> S<T> {
    fn new(item:T) -> Self {
        S {a:item}
    }
}
```

`Enum` 中：

```rust
enum Option<T> {
    Some(T),
    None,
}
```

`Trait`使用泛型：

```rust
// 表示将某种类型T转换为当前类型
trait From<T> {
  fn from(T) -> Self;
}
```

## Trait Object 和 Generic 对比

- `Trait Object`可以被看作一种数据类型，它总是以引用的方式被使用，在运行期间，它在栈中保存了具体类型的实例数据和实现自该`Trait`的方法。
- 泛型不是一种数据类型，它可被看作是数据类型的参数形式或抽象形式，在编译期间会被替换为具体的数据类型。
- `Trait Object` 也被称为动态分发(dynamic dispatch)，在程序运行期间动态的决定具体的类型
- 泛型是静态分派，在编译期间代码膨胀，将反省参数转换为每种具体的类型。
