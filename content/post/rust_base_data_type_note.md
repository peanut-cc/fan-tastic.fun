---
title: "Rust笔记----基本数据类型"
date: 2022-11-01T13:59:09+08:00
tags: ["Rust"]
categories: ["Rust"]
draft: true
---

Rust 是`静态类型`的语言,即每个值都有确切的数据类型

## 数值类型

数值类型: 有符号整数 (i8, i16, i32, i64, isize)、 无符号整数 (u8, u16, u32, u64, usize) 、浮点数 (f32, f64)、以及有理数、复数

Rust 中数值类型 的使用和其他语言差别不大，可能定义上更加灵活，下面三种方式是等效的，Rust默认整数类型时`i32`,浮点类型默认时`f64`

```rust
let a = 18i32;
let b = 18_i32;
let c = 18;
```

Rust 允许使用 `0b, 0o, Ox` 来表示二进制，八进制，和十六进制的整数。

```rust
let d = 0b101_i32;
let f = 0o17;
let g = 0xac;
```

整数之间默认不会隐式转换，如果需要转换，可以手动使用`as` 进行转换。不过需要注意从宽类型数值转换为窄类型数值时，如果溢出会从高位截断。

需要注意Rust中将字节字面量存储为`u8`类型，字节字面量的表示形式为`b'A'`。如： `let aa = b'A';`

## 布尔类型

Ture 和 False
和其他语言不同，在类似 if, while 语句中进行逻辑运算符 `||` 或者 `&&` 进行条件判断时，Rust只允许在条件判断处使用布尔类型。

下面这种写法时错误的

```rust
let a = 10;
if a {
    println!("True");
} else {
    println!("False");
}
```

这种写法时正确的：

```rust
let a = 10;
if a == 10 {
    ...
}
```

Rust的布尔值可以使用as操作符转换为数值类型，false对应0，true对应1

```rust
fn main() {
    println!("{}", true as i32);
    println!("{}", true as u8);
    println!("{}", false as u8);
    println!("{}", false as i32);
}
```

## char类型

char类型时Rust的一种基本数据类型，用于存放单个`unicode`字符，占4字节(32bit)。char字面量是**单引号**引起来的任意单个字符

```rust
let a = 'A';
```

数据转换:

- 可以使用`as`将char 转换为各种整数类型，目标类型小于4字节时，将从高位截断。
- 可以用`as`将u8类型转换为char
- 可以使用 `std::char::from_u32`将 u32 转换为 char, 返回值：`Option<char>`
- 可以使用`std::char::from_digit` 将 u32 根据第二个参数`BASE` 转换为对应进制的char, 返回值：`Option<char>`

## 字符串

字符串：字符串字面量和字符串切片 `&str`

### 字符串字面量

```rust
fn main() {
    let name = "fan-tastic";
    println!("{}", name);
}
```

代码中的 `"fan-tastic"` 就是字符串字面量。在 `let name = "fan-tastic";` 进行赋值变量时进行类型推导，`name` 变量的数据类型是`&str`, `&` 表示该类型的引用，即一个指针。`&str` 表示一个指向内存中的`str`类型数据的指针，该指针指向的内存位置处存了字符串数据`fan-tastic`。

可以理解为：

- 字符串字面量(string literal)的数据类型是`&str`
- 字符串字面量(string literal)是字符串切片类型的引用类型

### String

```rust
fn main() {
    let name = String::from("fan-tastic");
    println!("{}", name);
}
```

Rust 中可以通过 `String::from("fan-tastic")`来创建 `String` 类型的字符串

### str 和 String 的关系

`str` 字符串是`String`类型字符串的切片类型。

```rust
fn main() {
    let name = String::from("fan-tastic");
    let s_str = &name[..3];
    println!("{}", s_str);
}
```

`name[..3]`的类型是 `str`, 即 `&name[..3]`类型是 `&str`

### 字符串字面量如何存储

上面代码中中的字符串字面量 `fan-tastic` 是怎么存储的？
这里的处理比较特殊，编译器编译的时候直接将字符串字面量以硬编码的方式写入程序二进制文件中，存在`.RODATA`段(Linux)，只读。在Linux上可以通过`objdump`命令 或者 `readelf` 进行查看。

```shell
objdump -s -j .rodata rust_example_code | more

rust_example_code:     file format elf64-x86-64

Contents of section .rodata:
 3d000 66616e2d 74617369 74630a00 00000000  fan-tasitc......
 3d010 01000000 00000000 00000000 00000000  ................
 3d020 00040000 00000000 00000000 00000000  ................
 3d030 00000000 00000000 01000000 00000000  ................
 3d040 01000000 00000000 01000000 00000000  ................
 3d050 00000000 00000000 00000000 00000000  ................
 3d060 01000000 00000000 13000000 00000000  ................
 3d070 01000000 00000000 02000000 00000000  ................
 3d080 80000000 00000000 00000000 00000000  ................
 3d090 14000000 00000000 00000000 00000000  ................
```

```shell
readelf -x .rodata rust_example_code | more

Hex dump of section '.rodata':
  0x0003d000 66616e2d 74617369 74630a00 00000000 fan-tasitc......
  0x0003d010 01000000 00000000 00000000 00000000 ................
  0x0003d020 00040000 00000000 00000000 00000000 ................
  0x0003d030 00000000 00000000 01000000 00000000 ................
  0x0003d040 01000000 00000000 01000000 00000000 ................
  0x0003d050 00000000 00000000 00000000 00000000 ................
  0x0003d060 01000000 00000000 13000000 00000000 ................
  0x0003d070 01000000 00000000 02000000 00000000 ................
  0x0003d080 80000000 00000000 00000000 00000000 ................
  0x0003d090 14000000 00000000 00000000 00000000 ................
```

当程序运行时，可执行文件中的代码和数据从磁盘复制到内存中，通过下面这个图就可以知道代码中字面量 `fan-tastic` 的位置：

![Linux内存映像](/images/rust_base_data_type/linux内存映像.png)

所以当执行到 `let name = "fan-tastic";` 将 `fan-tasitc` 赋值给变量name(name是存在栈上)时，是将 `fan-tastic` 字面量的的内存地址保存到name中。
`let name = String::from("fan-tastic");`,当执行到这里的时候，会将在 `.RODATA`中存储的字符串字面量拷贝到`堆`上。并将该数据在堆内存中的地址赋值给变量 name。

## tuple 类型

Rust 中的 tuple 可以存放 0个，1个或多个任意类型的数据。访问时通过索引进行访问：tuple.N
当不存任何数据的tuple表示为`()`,这种情况比较特殊，这个时候叫做：unit(单元类型)。
当只存一个元素的时候，写法也比较特殊`let a =("fan-tastic",);` 这个元素后面的逗号不能省略。

```rust
fn main() {
    let tup = ("fan-tasitc", 18);
    let (name, age) = tup;
    println!("{}-{}", name,age);
    println!("{}-{}", tup.0,tup.1);
}
```

总结：

- 元组中的元素可以时不同数据类型
- 访问的索引必须编译时确定不能是变量
- 只存在一个元素时，元素后面的逗号不能省略
- 当不存在元素时，是unit(单元类型)

## Array 类型

Rust 中的数组**长度固定，元素类型相同。**

数组的表示形式`[Type;N]`:

- Type表示数组要存储的数据类型
- N时数组的长度,必须编译期确定，不能是一个变量

```rust
fn main() {
    let _arr = [1,2,3,4]; // 自定推导类型为[i32;4]
    let _arr2 = ["rust","fan-tastic"]; // 自动推到类型为[&str;2]
    let _arr3:[&str;3] = ["fan-tasitc", "peanut", "hhh"];
}
```

## 引用类型

引用类型是一种数据类型，它表示其所保存值的一个引用。Rust 中使用 `&T` 表示类型T的引用类型。

```rust
fn main() {
    let n = 18;
    let n_ref1 = &n;
    let n_ref2 = &n;
    // std::ptr::eq 判断两个引用是否指向同一个地址。
    println!("{}", std::ptr::eq(n_ref1,n_ref2)); // true
}
```

### 可变引用

默认我们直接使用`&T` 创建的是不可变引用，即只读。
`&mut T` 可以创建可变引用。

```rust
let mut n = 10;
let n_ref = &mut n;
```

注意：如果本身变量不是可变的的，那就无法创建可变引用。

### 解引用

解引用，通过引用获取该引用只想的原始值。`*T` 表示解引用。

```rust
fn main() {
    let mut n = 18;
    let n_ref = &mut n;
    n = *n_ref + 1;
    assert_eq!(n, 19);
}
```

Rust 在很多情况下会替我们自动解引用：

- 使用`.`操作符时，如获取属性，方法调用等，会隐式地尽可能解除或创建多层引用。
- 使用比较操作符时，如果比较的两边时相同类型的引用，则会自动解除引用，再进行比较。

## Slice 类型

Slice 通常被叫做切片类型。Rust 中常见的数据类型中，有三种类型已支持Slice 操作： String类型，Array类型，Vec类型。

### Slice 操作

- `s[n1..n2]`: 获取 s 中 `n1<=index<n2` 之间的元素
- `s[n1..]`: 获取 s 中 index=n1 到 最后的元素
- `s[..n1]`: 获取 s 中 从 开头到 n1(但不包含n1) 之间的元素
- `s[..=n1]`: 获取 s 中 从 开头到 n1(包含n1) 之间的元素

注意： 切片操作允许使用usize 类型的变量作为切片的边界。

### 数据类型

Slice 类型是一个胖指针即包含两分元数据：

- 指向源数据中切片起点元素的指针
- 元素的数量，即切片的长度

```rust
fn main() {
    let arr = [1,2,3,4,5];
    let n:usize = 3;
    // let arr_s = arr[..n]; // 报错
    let arr_s = &arr[..n];
    println!("{:?}", arr_s);
}
```

Rust中几乎总是使用切片数据的引用。切片数据的引用对应的数据类型描述为`&[T]`或`&mut [T]`，前者不可通过Slice引用来修改源数据，后者可修改源数据。

注意和数组的描述区分：
数组类型表示为`[T; N]`，数组的引用类型表示为`&[T; N]`，Slice类型表示为`[T]`，Slice的引用类型表示为`&[T]`。

### str切片类型

String的切片类型是`str`，而非`[String]`，String切片的引用是&str而非`&[String]`。

Rust为了保证字符串总是有效的Unicode字符，它不允许用户直接修改字符串中的字符，所以也无法通过切片引用来修改源字符串。

### Array 类型自动转换 Slice 类型

```rust
fn main() {
    let names = ["peanut","fan-tasitc"];
    let slice = &names; //&arr 将自动转换为slice类型
    println!("{}", slice.first().unwrap())// 这里调用的是slice类型的first()方法
}
```

可以直接将数组的引用当成slice来使用。即`&arr`和`&mut arr`当作不可变slice和可变slice来使用。
在调用方法的时候，由于`.`操作符会自动创建引用或解除引用，因此Array可以直接调用Slice的所有方法。
