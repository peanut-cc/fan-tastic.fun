---
title: "Rust笔记----所有权"
date: 2023-07-24T08:47:03+08:00
tags: ["Rust", "ownership", "所有权"]
categories: ["Rust"]
keywords: ["rust","ownership", "rust所有权", "作用域", "Move", "可变引用与不可变引用"]
draft: false
---

Rust所有权系统是保证Rust内存安装的最关键手段之一。

## Rust 变量作用域

Rust 中大括号就表示一个单独的作用域。常见的大括号作用域有：

- if，while 等流程控制语句的大括号
- match 模式匹配的大括号
- 单独的大括号
- 函数定义的大括号
- mod 定义模块的大括号

```rust
fn main() {
    let name = "fan-tastic";
    println!("name is {} address is {:p}", name, name);
    {
        let name = "peanut";
        println!("name is {} address is {:p}", name, name);
    }
    println!("name is {} address is {:p}", name, name);
}
```

运行结果为：

```bash
name is fan-tastic address is 0x55bd3842b05b
name is peanut address is 0x55bd3842b07a
name is fan-tastic address is 0x55bd3842b05b
```

在看另外两种种情况：

```rust

/// 情况一
fn main() {
    {
        let name = "peanut";
        println!("name is {} address is {:p}", name, name);
    }
    let name = "fan-tastic";
    println!("name is {} address is {:p}", name, name);
}

// 打印的结果为：
name is peanut address is 0x55a184fa805b
name is fan-tastic address is 0x55a184fa8076


/// 情况二
fn main() {
    {
        let name = "peanut";
        println!("name is {} address is {:p}", name, name);
    }
    let name = "peanut";
    println!("name is {} address is {:p}", name, name);
}

// 打印的结果：
name is peanut address is 0x55f81906405b
name is peanut address is 0x55f81906405b
```

字符串字面量是放在二进制文件中的`.RODATA`段中，所以即使是不同作用域，如果他们的字面量是相同的的，这个时候编译器会优化，让在不同作用域中的变量使用的都是同一个字面量的引用。
字面量是编译时写入到二进制文件中的，会在程序启动到程序终止期间一直存在。

### 函数作用域

函数作用域内，无法访问函数外部的变量，而大括号的作用域，可以访问大括号外部的变量。

```rust
fn main() {
    let mut a = 18;
    {
        a += 1;
    }
    println!("{}", a); // 19
}
```

再看一个例子：

```rust
fn main() {
    let mut a = 10;
    {
        a+=1;
        println!("{}", a); // 11
        let mut a = 3;
        a += 10;
        println!("{}", a);// 13
    }
    println!("{}", a); // 11
}
```

- `a+=1;`访问并修改了 外部变量 a
- `let mut a = 3;` 在当前大括号内重新声明了变量a,这就是变量的遮盖(shadow)

### 悬垂引用

在Rust中，编译器保证不会出现悬垂引用，即引用必须总是有效。

```rust
fn foo() -> &String {
    let s = String::from("fan-tastic");
    &s
}

fn main() {
    foo();
}
```

这段代码不能编译通过。函数foo 返回的是 s 的引用，而s 是在函数内声明的，当离开函数后，s 就会被销毁，这使得返回值`&s`变成了一个无效的引用。

## 位置和值

首先理解一下当声明一个变量的时候，数据在堆栈中的存储情况

如`let a = 3;`

![变量位置和值](/images/rust_ownership/变量位置和值.png)

a 称为变量名，可以理解为就是编程语言上提供了一个别名，就是对栈上位置的一个代号，可以理解为房间号和房间的关系
变量 a 对应栈中的一个位置，这个位置保存了值
栈上的位置是有自己的内存地址的，假设就是上图中的 `0x123`
这里可以简单粗暴的理解为 a 就是 栈上的那个位置，即地址`0x123`
每个位置(变量)都是它所存放的值的所有者

如 `let aa = vec![1,2,3];`

![变量位置和值2](/images/rust_ownership/变量位置和值2.png)

这里有两个位置，一个是栈中的位置，一个是堆中的位置
在栈中的位置存的是胖指针，这个是在声明变量的时候显示创建的位置，这个位置存放的是Vec类型，而堆中的位置是隐式产生的，这个位置和变量没有关系，而是栈中位置上的胖指针中有一个指针指向这个堆中的位置。
变量 aa 的值是栈上的胖指针，而不是实际的数据
变量 aa 是栈上的胖指针的所有者，而不是实际数据的的所有者

再看

```rust
let n = 18;
let nn = &n;
```

![变量位置和值3](/images/rust_ownership/变量位置和值3.png)

变量 n 对应栈中的一个位置，这个位置保存了18,这个位置的地址是 0x777
变量 nn 也是栈中中的一个位置，这个位置保存了一个地址值，这个地址值为 0x777 即指向变量n的位置

同理如下代码的：

```rust
let v = vec![1,2,3];
let vv = &v;
```

变量 vv 在栈上位置中存的是 变量 v 在栈上位置地址，Rust的宗旨之一就是保证安全，不允许存在对堆中同一个内存的多个指向
只有最初创建这块堆内存的v变量才指向堆中这块数据。

变量一旦被初始化，无论它之后保存的数据发生什么变化，它的地址都是固定不变的。

```rust
fn main() {
    let mut name = "fan-tasitc".to_string();
    println!("{:p}", &name);

    let name2 = name;
    println!("{:p}", &name2);

    name = "peanut".to_string();
    println!("{:p}", &name);
}
```

运行结果：

```bash
0x7fff590900d0
0x7fff59090130
0x7fff590900d0
```

从结果也可以看出，不管怎么改变的变量name的值，name 在栈上的位置是始终不变的

## 所有权规则

所有权规则：

- 每一个值都被一个变量所拥有，该变量被称为值的所有者
- 一个值同时只能被一个变量所拥有，或者说一个值只能有一个所有者。(值在任一时刻有且只有一个所有者)
- 当所有者(变量)离开作用域范围时，这个值将被销毁(drop)

关于第三条需要注意：如果类似字符串字面量这种保存在全局内存区域，那么数据其实并没有销毁，只是删除了栈上的引用。

### 所有者

如何理解Rust 中每个值都有一个所有者？
`let s = String::from("fan-tastic");` String字符串的实际数据在堆中，但是String大小不确定，所以在栈中使用一个胖指针结构来表示这个String 类型的数据，这个胖指针中的指针指向堆中的String 实际数据。即变量s 的值是那个胖指针，而不是堆中的实际数据。如下图：

![string内存结构](/images/rust_ownership/string内存结构.png)

所以准确的说的 变量 s 是那个胖指针的所有者，而不是堆中实际数据的所有者。

## move/copy/clone

### move

```rust
fn main() {
    let s = String::from("fan-tasitc");
    let s2 = s;
    println!("{},{}", s,s2);
}
```

会看到如下错误：

```rust
--> src/main.rs:6:23
  |
4 |     let s = String::from("fan-tasitc");
  |         - move occurs because `s` has type `String`, which does not implement the `Copy` trait
5 |     let s2 = s;
  |              - value moved here
6 |     println!("{},{}", s,s2);
  |                       ^ value borrowed here after move
  |
```

s 是 String 字符串，变量 s 是指向 栈上的胖指针，在执行 `let s2 = s;` 的时, 拷贝 s 变量在栈中的旁指针给s2,s变量重置为未初始化状态,即将s移动到s2, 也成为值的所有权从s转移给s2. 这里需要切记这个过程并不会拷贝堆中的数据.

![move](/images/rust_ownership/move.png)

move不仅发生在上面这种变量赋值的过程中,在函数传递参,函数返回数据也同样会发生move.

再看如下代码：

```rust
fn main() {
    let x = "fan-tasitc";
    let y = x;
    println!("{} {}",x,y);
}
```

这里因为字符串字面量 "fan-tasitc" 是在 二进制文件的 `.RODATA`段中 中，所以 x 只是引用了存储在二进制文件的 `.RODATA`段的字符串，并未持有所有权。所以`let y = x;` 只是对该引用进行了拷贝，此时 y 和 x 都引用了同一个字符串。

```rust
fn main() {
    let mut x = String::from("hello");
    let y = x;
    // y.push_str(" rust")
    let a = String::from("hello");
    let mut b = a;
    b.push_str(" rust");
    println!("{}", b);
}
```

x将所有权转移给y,但y无法修改字符串.
虽然a无法修改字符串,但转移所有权后,b可修改字符串.

### copy

Rust copy语义 和 move的区别是在copy后，原始变量仍然可以用。Rust 默认是使用Move语义，如果需要使用Copy语义，需要被拷贝的数据类型实现了Copy Trait。

Rust 中常见的默认实现了Copy Trait 的类型：

- 所有整数类型
- 所有浮点数类型
- 布尔类型
- 字符类型
- 元组 只有其包含的类型都是Copy 的时候
- 共享指针类型或共享引用类型
- ....

```rust
fn main() {
    let n = 100;
    let n2 = n;
    println!("{} {}", n, n2); // 100 100
}
```

### clone

clone 可以拷贝变量的数据，同时不会影响变量.不过只有实现了 Clone Trait的类型才可以进行Clone。

Copy 和 Clone 区别：

- Copy 时，只拷贝变量本身的值，如果这个变量指向了其他数据，则不会拷贝其他指向的数据
- Clone 时，拷贝变量本身的值，如果这个变量指向了其他数据，则也会拷贝其指向的数据

所以可以理解,Copy 是浅拷贝,Clone是深拷贝,rust 会对每个字段每个元素递归调用clone().

```rust
fn main() {
    let v = vec!["fan-tasitc".to_string()];
    let v2 = vec![v];
    println!("{:p}",&v2[0][0]); //0x556a5c5e0ba0

    let v3 = v2.clone();
    println!("{:p}", &v3[0][0]); //0x556a5c5e0c20
}
```

### 函数参数和返回值

函数的参数类似于变量赋值,在调用函数时,会将所有权移动给函数参数.
函数返回时,返回值的所有权从函数内部移动到函数外部变量.

```rust
fn f1(s: String) -> String {
    let a = String::from("peanut");
    println!("args is {} and a is {}", s, a);
    a
}

fn f2(i: i32){
    println!("{}", i);
}

fn main() {
    let s = String::from("fan-tasitc");
    let s2 = f1(s);
    println!("{}", s2);
    // println!("{}", s); // error: value borrowed here after move

    let n = 10;
    f2(n);
}
```

- `let s2 = f1(s);` 所有权从s 移动到了f1的参数，f1 函数返回值的所有权移动给了s2。
- `f2(n);` 没有所有权的移动，只是拷贝了一份给f2 的参数。

自己在代码中应该根据需要确定是否需要将所有权转移给函数参数。如果不想失去所有权，可以将变量的引用传递给函数参数。

## 引用和所有权借用

```rust
fn main() {
    let s = String::from("fan-tasitc");
    let s1 = &s;
    let s2 = &s;
    println!("{} {:p} {:p}", s, s1, s2); // fan-tasitc 0x7ffd9cc1a6b0 0x7ffd9cc1a6b0
}
```

![ownership1](/images/rust_ownership/ownership1.png)

(不可变)引用实现了`Copy Trait`, 所以上面的代码也可以这样写:

```rust
fn main() {
    let s = String::from("fan-tasitc");
    let s1 = &s;
    let s2 = s1;
    println!("{} {:p} {:p}", s, s1, s2);
}
```

但s1是引用，是可Copy的,因此在`let s2 = s1;`sf1仍然有效,即仍然指向数据,并不会失效.
不止是在赋值的时候,在函数参数传递时,也可以通过使用引用来避免在调用函数时,丢失所有权.

```rust
fn main() {
    let s = String::from("fan-tastic");
    let s1 = s.clone();
    f1(s1);

    let l = f2(&s);
    println!("{} len is {}", s, l);
}

fn f1(s: String) {
    println!("{}", s);
}
```

上述代码中,s1 在执行`f1(s1)`的时候,丢失了所有权,s1 重置为未初始化状态.
`let l = f2(&s);` 传递的是s的引用(borrow),借用s所有权

### 可变引用和不可变引用的所有权规则

`&mut T` 表示可变引用， `&T` 表示不可变引用。站在所有权借用的角度来看，可变引用表示的是可变借用，不可变引用表示的是不可变借用。

- 不可变借用：借用只读权，不允许修改其引用的数据
- 可变引用：借用可写权，允许修改其引用的数据
- 可以同时存在多个不可变引用
- 在有可变引用时，不允许存在其他可变引用和不可变引用

```rust
fn main() {
    let mut x = String::from("fan-tasitc");

    let x_ref = &mut x;
    // let x_ref_read = &x; // 如果放在这里将会编译失败
    x_ref.push_str(".fun");
    println!("{}", x);
    let x_ref_read = &x;
    println!("{}",x_ref_read);

    let mut s = String::from("hello");
    f1(&mut s);
    println!("{}", s);
}

fn f1(s: &mut String) {
    s.push_str(" world");
}
```

思考为什么仅仅因为顺序的不同编译失败？

个人理解 "在有可变引用时，不允许存在其他可变引用和不可变引用" 的意思是：

- 不可变引用是可以和可变引用同时创建。
- 如果存在可变引用，这个时候就会具有排他性，但这个排他性在于你是否同时使用。
- 如果存在可变引用，原始变量的使用和可变引用的使用也会存在排他性。

```rust
fn main() {
    let mut a = String::from("fan-tasitc");
    let a_ref1 = &a;
    let a_ref2 = &a;
    // 上面只存在不可变引用，这个不存在冲突，是可以创建多个的
    // 下面开始创建了可变引用，在这个时候，上面创建的两个引用都将无法使用
    let a_ref3 = &mut a;

    // 再次创建不可变引用，上面的都将不可以使用
    let a_ref4 = &a;
    let a_ref5 = &mut a;

    println!("{}", a_ref5);
    println!("{}", a);
    println!("{}",a_ref5); //报错
}
```

再看一段代码：

```rust
fn main() {
    let mut x = 10;
    let y = &mut x;
    x = *y + 1;
    println!("{}", x);
    println!("{}", y); // 报错
}
```

运行错误如下：

```rust
 --> src/main.rs:6:5
  |
5 |     let y = &mut x;
  |             ------ borrow of `x` occurs here
6 |     x = *y + 1;
  |     ^^^^^^^^^^ assignment to borrowed `x` occurs here
7 |     println!("{}", x);
8 |     println!("{}", y); // 报错
  |                    - borrow later used here
```

在`x = *y + 1;` 这里已经借用了y,而在借用获得所有权之后，将无法在使用，即y将失效。

总结：如果是创建的不可变引用，如果再次创建可变引用，

### 容器集合类型所有权规则

容器中可能包含栈中的数据，也可能包含堆中的数据。

`let tup = (10, String::from("fan-tasitc));`

容器变量拥有容器中所有元素值的的所有权。但是需要注意：
容器中的元素如果发生了所有权的转移，容器将不在拥有其所有权，容器本身也不能使用，但是容器中的其他元素依然可以使用

```rust
fn main() {
    let tup = (1, String::from("fan-tastic"));

    let (x, y ) = tup;
    println!("{}, {}", x, y);
    println!("{}", tup.0);
    println!("{}", tup.1); // 错误
    print!("{:?}", tup); // 错误
}
```

`let (x, y ) = tup;` 这里 tup 的第二个元素发生了所有权转移。所以后续无法再使用`tup.1` 和 `tup`

如果想让原容器中的变量依然可以使用，可以有三种方法：

```rust
// 方式一：忽略
let (x, _) = tup;
println!("{}", tup.1);  //  正确

// 方式二：clone
let (x, y) = tup.clone();
println!("{}", tup.1);  //  正确

// 方式三：引用
let (x, ref y) = tup;
println!("{}", tup.1);  //  正确
```

总结： 一旦对变量进行了可变引用，这个位置将只能存在单一使用者，使用者可以是原始变量，也可以是新的可变引用或者不可变引用，使用者可以随时切换，但保证任意时刻只能有一个使用者。

## 再次理解Move

```rust
fn main() {
    let name = &"fan-tasitc".to_string();
    let name2 = *name;
}
```

这代码编译错误：

```rust
cannot move out of `*name` which is behind a shared reference
```

当产生一个位置，并且需要向位置中放入值，就会发生移动。这个值可能来自某个变量，可能来自计算结果，值的类型可能实现了 `Copy Trait` 。需要注意的是解引用操作也是需要转移所有权。

这里 name 是一个引用，并不是 String 类型的所有者，当使用 `*name` 解引用时，需要转移所有权，但是发现 name 只是引用，并不是所有者，导致无法转移值的所有权，出现了错误。

下面代码时正确的使用方式：

```rust
fn main() {
    let name = &"fan-tasitc".to_string();
    let name2 = &*name;
    let name3 = (*name).clone();
    let num = &3;
    let num2 = *num;
}
```

个人总结：
`let name2 = *name;` 的时候`*name`需要获取所有权，但是name并没有String 的所有权，加上，String 类型并没有实现Copy 所以这里报错。
`let num2 = *num;` 这里虽然`*num`也会获取所有权，但是因为int32类型实现了Copy，所以这里不会报错

再看一段代码：

```rust
fn main() {
    let name = "fan-tasitc".to_string();
    name;
    println!("{}", name);
}
```

编译错误：

```rust
--> src/main.rs:4:20
  |
2 |     let name = "fan-tasitc".to_string();
  |         ---- move occurs because `name` has type `String`, which does not implement the `Copy` trait
3 |     name;
  |     ---- value moved here
4 |     println!("{}", name);
  |                    ^^^^ value borrowed here after move
  |
```

这里单独的 `name;` 其实等价于 `let _tmp = name;` 即将值移动给了一个临时变量。

## 引用类型的Copy和Clone

引用类型是可Copy的，所以引用类型在Move的时候都会Copy 一个引用的副本，Copy前后的引用都指向同一个目标值。

```rust
fn main() {
    let name = "fan-tastic".to_string();

    // name2 和 name3 都是 name 的引用
    let name2 = &name;
    let name3 = name2; // Copy 引用
}
```

引用类型也是可Clone的(实现Copy的时候要求也必须实现Clone，所以可Copy的类型也是可Clone的)，但是引用类型的clone()需注意: 对引用进行clone() 时，将拷贝引用类型本身，而不是拷贝引用所指向的数据本身。

```rust

#[derive(Clone)]
struct Person;

struct Person2;

fn main() {
    let a = Person;
    let b = &a;
    let _c = b.clone(); // _c的类型时 Persion

    let aa = Person2;
    let bb = &aa;
    let _cc = bb.clone(); // _cc的类型时 &Person2
}

```

- 没有实现Clone时，引用类型的clone()将等价于Copy
- 实现了Clone时，引用类型的clone()将克隆并得到引用所指向的类型
- 方法调用的符号`.`会自动解引用
