---
title: "《Rust Atomics and Locks Low-Level 》阅读笔记1"
date: 2023-04-18T14:38:08+08:00
tags: ["Rust"]
categories: ["Rust"]
keywords: ["rust","thread", "concurrency"]
draft: false
---

## Threads in Rust

在 Rust 中 可以通过标准库中 `std::thread::spawn` 创建线程。

```rust
use std::thread;

fn main() {
    thread::spawn(f);
    thread::spawn(f);
    println!("Hello from the main thread.");
}

fn f() {
    println!("hello from another thread");

    let id = thread::current().id();
    println!("This is my thread id:{id:?}");
}

```

这段代码需要注意,`main` 线程一旦结束，程序就会立刻退出，上面的代码中，通过`thread::spawn(f);`创建了两个线程，但是多次运行程序你会发现，他们可能并没有机会运行。

如果想要保证所有的线程执行完成，可以使用 `join` 方法：

```rust
use std::thread;

fn main() {
    let t1 = thread::spawn(f);
    let t2 = thread::spawn(f);

    println!("Hello from the main thread.");

    t1.join().unwrap();
    t2.join().unwrap();
}

fn f() {
    println!("hello from another thread");

    let id = thread::current().id();
    println!("This is my thread id:{id:?}");
}
```

`join` 方法会等待线程执行完成并切返回 `std::thread::Result`,如果线程执行失败，这个返回的数据中将会包含`panic`的信息。

在创建线程的时候还有一种使用更多的方式是通过闭包。

```rust
use std::thread;

fn main() {
    thread::spawn(move || {
        for n in &numbers {
            println!("{n}");
        }
    })
    .join()
    .unwrap();
}
```

因为在闭包的时候使用的`move`, `numbers`的所有权会传入到新的开启的线程中。
如果我们没有使用`move`, 就不是发生所有权的转移，而是通过引用的方式在新的线程中使用`numbers`，但是这个时候编译代码就会提示如下错误：

```rust
  |
5 |     thread::spawn(|| {
  |                   ^^ may outlive borrowed value `numbers`
6 |         for n in &numbers {
  |                   ------- `numbers` is borrowed here
  |
note: function requires argument type to outlive `'static`
 --> ch1-03-spawn-closure.rs:5:5
  |
5 | /     thread::spawn(|| {
6 | |         for n in &numbers {
7 | |             println!("{n}");
8 | |         }
9 | |     })
  | |______^
help: to force the closure to take ownership of `numbers` (and any other referenced variables), use the `move` keyword
  |
5 |     thread::spawn(move || {
  |                   ++++

error: aborting due to previous error
```

`spawn`函数对其参数类型有一个 "静态寿命" 的约束，一个通过引用捕获局部变量的闭包不可能一直存在，当局部变量不再存在时，就会导致引用无效。

Rust 标准库 `std::thread::scope`可以创建 `scoped threads`, 允许创建的线程安全的借用局部变量。

```rust
use std::thread;

fn main() {
    let numbers = vec![1, 2, 3, 4];

    thread::scope(|s| {
        s.spawn(|| {
            println!("{}", numbers.len());
        });
        s.spawn(|| {
            for n in &numbers {
                println!("{n}");
            }
        });
    });
}
```

在`scope` 不用在使用 `join` 方法等待所有线程执行完成，而是变成自动 `join`

## 共享所有权 和 引用计数

在两个线程之间共享数据时，无法保证一个线程比另外一个线程“活得更久”，它们都不能成为该数据的所有者。

`static` 静态变量会在整个程序运行的时期中存在。每一个线程都可以借用它。
`Box` 使用`Box::leak`，可以释放一个`Box`的所有权，并保证永远不放弃它。

```rust
use std::thread;

fn main() {
    let x: &'static [i32; 3] = Box::leak(Box::new([1, 2, 3]));
    thread::spawn(move || dbg!(x));
    thread::spawn(move || dbg!(x));
}
```

Rust 标准库 `std::rc::Rc` , 引用计数，clone 它并不会分配任何新的东西，而是增加一个存储的值的一个引用计数。需要注意：`Rc` 不是线程安全的。

```rust
use std::rc::Rc;

fn main() {
    let a = Rc::new([1, 2, 3]);
    let b = a.clone();
    assert_eq!(a.as_ptr(), b.as_ptr());
}
```

代码中 变量a 和 变量b 共享所有权。

标准库`std::sync::Arc`，标识原子引用计数，保证了对引用的计数器的修改是`原子操作`,因此可以安全的用于多线程。

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let a = Arc::new([1, 2, 3]);
    let b = a.clone();

    thread::spawn(move || dbg!(a));
    thread::spawn(move || dbg!(b));
}
```

> Rust allows (and encourages) you to shadow variables by defining a new variable with the same name

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let a = Arc::new([1, 2, 3]);
    thread::spawn({
        let a = a.clone();
        move || {
            dbg!(a);
        }
    });
    dbg!(a);
}
```

> Because ownership is shared, reference counting pointers (`Rc<T>` and `Arc<T>`) have the same restrictions as shared references (`&T`). They do not give you mutable access to their contained value, since the value might be borrowed by other code at the same time.

## 内部可变性 Interior Mutability

共享引用（`&T`）可以被复制并与他人共享，而独占引用（`&mut T`）则保证它是该`T`的唯一独占借用。

使用标准库 `std::cell:Cell<T>` 允许通过共享引用进行突变，允许把值复制出来（如果T是可Copy的），或者用另外一个值替换。
注意：Cell 只能在单个线程中使用。

```rust
fn f(v: &Cell<Vec<i32>>) {
    let mut v2 = v.take(); // Replaces the contents of the Cell with an empty Vec
    v2.push(1);
    v.set(v2); // Put the modified Vec back
}
```

a `std::cell::RefCell` does allow you to borrow its contents, at a small runtime cost.

一个`RefCell<T>`不仅持有一个`T`，而且还持有一个`counter`，用于跟踪任何未完成的`borrow`。如果已经被`mutably borrow`了，再进行借用，就会Panic。
注意：RefCell 只能在单个线程中使用。

```rust
use std::cell::RefCell;

fn f(v: &RefCell<Vec<i32>>) {
    v.borrow_mut().push(1); // We can modify the `Vec` directly.
}
```

## RwLock and Mutex

`RwLock` 是`RefCell`的并发版本，不同的是，这个不会在冲突的`borrow` 发生panic, 而是阻塞当前的线程，使其进入睡眠状态，同时等待冲突的`borrow` 消失。

`Mutex` 不会像 RwLock那样记录共享和独占的借用的数量，而只允许独占`borrow`。

`atomic`是 `Cell`的并发版本，不过不同的是不能有任意的大小，所以没有通用的`Atomic<T>`类型，只有特定类型的`Atomic`类型。

为了保证被Lock的的`Mutex`只能被`Lock` 的线程解锁，这里并没有提供一个`unlock()`方法，而是在`lock()`方法返回一个特殊的类型`MutexGuard`, 这个就标识了我们已经锁定了`Mutex`, 它的行为类似于通过`DerefMut`特性的独占引用，让我们独占访问`Mutex`所保护的数据。解除对`Mutex`的锁定是通过 Drop guard。

When we drop the guard, we give up our ability to access the data, and the Drop implementation of the guard will unlock the mutex.

```rust
use std::sync::Mutex;
use std::thread;

fn main() {
    let n = Mutex::new(0);
    thread::scope(|s| {
        for _ in 0..10 {
            s.spawn(|| {
                let mut guard = n.lock().unwrap();
                for _ in 0..100 {
                    *guard += 1;
                }
            });
        }
    });
    assert_eq!(n.into_inner().unwrap(), 1000);
}
```

After the threads are done, we can safely remove the protection from the integer through into_inner(). The into_inner method takes ownership of the mutex, which guarantees that nothing else can have a reference to the mutex anymore, making locking unnecessary.

在Rust 中，当一个线程在持有锁的时候出现的panic, `Mutext`会被标记为 `poisoned`, 这种情况下`Mutex`将不再被锁定，但调用它的锁定方法将会导致一个Err来标识已经被 `poisoned`。
在一个已经被标记为`poisoned`的`Mutext`调用`lock()`仍然会锁定该`Mutext`, `lock()` 的 Err 包含 `MutextGuard`,允许我们在必要时纠正一个不一致的状态。

`RwLock` 有三种状态： 解锁，被单个写锁定（用于独占访问），被任何数量的读锁定（用于共享访问），通畅用于经常被多个线程读取的数据。
提供了`read()`和`write()`方法,用于读锁或者写锁，所以这里会有两个`guard`类型 `RwLockReadGuard`和 `RwLockWriteGuard`。
`Mutext<T>` 和 `RwLock<T>`都要求`T`实现 `Send`,因为他们可以用来发送一个`T`给另外一个线程 `RwLock<T>` 还要求T实现`Sync` 它允许多个线程对受保护数据的共享引用（`&T`）

一个线程可以 `park`自己，进入睡眠状态，停止消耗任何CPU, 另外一个线程可以`unpark` 被 `park`状态的线程，把它从睡眠状态唤醒。

```rust
use std::collections::VecDeque;
use std::sync::Mutex;
use std::thread;
use std::time::Duration;

fn main() {
    let queue = Mutex::new(VecDeque::new());

    thread::scope(|s| {
        let t = s.spawn(|| loop {
            let item = queue.lock().unwrap().pop_front();
            if let Some(item) = item {
                dbg!(item);
            } else {
                thread::park();
            }
        });

        for i in 0.. {
            queue.lock().unwrap().push_back(i);
            t.thread().unpark();
            thread::sleep(Duration::from_secs(1));
        }
    });
}
```

## Condition Variables

Condition variables are a more commonly used option for waiting for something to happen to data protected by a mutex. They have two basic operations: wait and notify.
Threads can wait on a condition variable, after which they can be woken up when another thread notifies that same condition variable. Multiple threads can wait on the same condition variable, and notifications can either be sent to one waiting thread, or to all of them.
This means that we can create a condition variable for specific events or conditions we’re interested in, such as the queue being non-empty, and wait on that condition.
Any thread that causes that event or condition to happen then notifies the condition variable, without having to know which or how many threads are interested in that notification.
The Rust standard library provides a condition variable as std::sync::Condvar. Its wait method takes a MutexGuard that proves we’ve locked the mutex. It first unlocks the mutex and goes to sleep. Later, when woken up, it relocks the mutex and returns a new MutexGuard (which proves that the mutex is locked again).

```rust
use std::collections::VecDeque;
use std::sync::Condvar;
use std::sync::Mutex;
use std::thread;
use std::time::Duration;

fn main() {
    let queue = Mutex::new(VecDeque::new());
    let not_empty = Condvar::new();

    thread::scope(|s| {
        s.spawn(|| loop {
            let mut q = queue.lock().unwrap();
            let item = loop {
                if let Some(item) = q.pop_front() {
                    break item;
                } else {
                    q = not_empty.wait(q).unwrap();
                }
            };
            drop(q);
            dbg!(item);
        });
        for i in 0.. {
            queue.lock().unwrap().push_back(i);
            not_empty.notify_one();
            thread::sleep(Duration::from_secs(1));
        }
    })
}
```

Normally, a Condvar is only ever used together with a single Mutex. If two threads try to concurrently wait on a condition variable using two different mutexes, it might cause a panic.

## Thread Safety

> All primitive types such as i32, bool, and str are both Send and Sync.
> A struct with fields that are all Send and Sync, is itself also Send and Sync.

**Send**
A type is Send if it can be sent to another thread. In other words, if ownership of a value of that type can >    be transferred to another thread. For example, `Arc<i32>` is Send, but `Rc<i32>` is not.
**Sync**
A type is Sync if it can be shared with another thread. In other words, a type T is Sync if and only if a > shared reference to that type, &T, is Send. For example, an i32 is Sync, but a `Cell<i32>` is not. (A `Cell<i32`> is Send, however.)
