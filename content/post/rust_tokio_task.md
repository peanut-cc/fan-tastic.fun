---
title: "Rust笔记----tokio中的task"
date: 2023-08-20T18:01:26+08:00
tags: ["tokio"]
categories: ["Rust"]
keywords: ["tokio","rust task"]
draft: false
---

task: Asynchronous green-threads. Rust中的原生线程(std::thread)是OS线程,每一个原生线程,都对应一个操作系统的线程.green thread则是用户空间的线程,由程序自身提供的调度器负责调度.同一个OS线程内的多个绿色线程之间的上下文切换的开销非常小.

每定义一个Future(async语句块), 就是定义一个静止的尚未执行的task,当它在runtime中开始运行的时候,就是真正的task,一个真正的异步任务.

`tokio::task` 模块提供了几个常用函数:

- spawn：向runtime中添加新异步任务
- spawn_blocking：生成一个blocking thread并执行指定的任务
- block_in_place：在某个worker thread中执行同步任务,但是会将同线程中的其它异步任务转移走,使得异步任务不会被同步任务饥饿
- yield_now: 立即放弃CPU,将线程交还给调度器,自己则进入就绪队列等待下一轮的调度
- unconstrained: 将指定的异步任务声明未不受限的异步任务,它将不受tokio的协作式调度,它将一直霸占当前线程直到任务完成,不会受到tokio调度器的管理
- spawn_local: 生成一个在当前线程内运行，一定不会被偷到其它线程中运行的异步任务

这里的三个spawn类的方法都返回JoinHandle类型,JoinHandle类型可以通过`await`来等待异步任务的完成,也可以通过`abort()`来中断异步任务,异步任务被中断后返回JoinError类型.

## task::spawn()

直接在当前的runtime中生成一个异步任务

```rust
use chrono::Local;
use std::thread;
use tokio::{self, task, runtime::Runtime, time};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

fn main() {
    let rt = Runtime::new().unwrap();
    let _guard = rt.enter();
    task::spawn(async {
        time::sleep(time::Duration::from_secs(3)).await;
        println!("task over: {}", now());
    });

    thread::sleep(time::Duration::from_secs(4));
}
```

## task::spawn_blocking()

生成一个blocking thread来执行指定的任务

```rust
let join = task::spawn_blocking(|| {
    // do some compute-heavy work or call synchronous code
    "blocking completed"
});

let result = join.await?;
assert_eq!(result, "blocking completed");
```

## task::block_in_place()

`block_in_place()`是在当前worker thread中执行指定的可能会长时间运行或长时间阻塞线程的任务,但是它会先将该worker thread中已经存在的异步任务转移到其它worker thread,使得这些异步任务不会被饥饿.
`block_in_place()`只应该在多线程runtime环境中运行,如果是单线程runtime,block_in_place会阻塞唯一的那个worker thread
在`block_in_place`内部, 可以使用`block_on()`或`enter()`重新进入runtime环境

```rust
use tokio::task;
use tokio::runtime::Handle;

task::block_in_place(move || {
    Handle::current().block_on(async move {
        // do something async
    });
});
```

## task::yield_now

让当前任务立即放弃CPU,将worker thread交还给调度器.任务自身则进入调度器的就绪队列等待下次被轮询调度
需注意 调用`yield_now()`后还需`await`才立即放弃CPU，因为`yield_now`本身是一个异步任务

```rust
use tokio::task;

async {
    task::spawn(async {
        // ...
        println!("spawned task done!")
    });

    // Yield, allowing the newly-spawned task to execute first.
    task::yield_now().await;
    println!("main task done!");
}
```

## task::unconstrained()

`task::unconstrained()`创建一个不受限制不受调度器管理的异步任务,它将不会参与调度器的协作式调度,可以认为是将这个异步任务暂时脱离了调度管理. 这样一来,即便该任务中遇到了本该阻塞而放弃线程的操作,也不会去放弃,而是直接阻塞该线程.

## 取消任务abort()

正在执行的异步任务可以随时被`abort()`取消,取消之后的任务返回`JoinError`类型

```rust
use tokio::{self, runtime::Runtime, time};

fn main() {
    let rt = Runtime::new().unwrap();

    rt.block_on(async {
        let task = tokio::task::spawn(async {
            time::sleep(time::Duration::from_secs(10)).await;
        });

        // 让上面的异步任务跑起来
        time::sleep(time::Duration::from_millis(1)).await;
        task.abort();  // 取消任务
        // 取消任务之后，可以取得JoinError
        let abort_err: JoinError = task.await.unwrap_err();
        println!("{}", abort_err.is_cancelled());
    })
}
```

如果异步任务已经完成,再对该任务执行`abort()`操作将没有任何效果,也就是说,没有JoinError.
`task.await.unwrap_err()`将报错,而`task.await.unwrap()`则正常

## tokio::join!宏和tokio::try_join!宏

可以使用`await`去等待某个异步任务的完成,无论这个异步任务是正常完成还是被取消.
tokio提供了两个宏`tokio::join!`和`tokio::try_join!`,可以用于等待多个异步任务全部完成

- `join!`必须等待所有任务完成
- `try_join!`要么等待所有异步任务正常完成,要么等待第一个返回`Result Err`的任务出现

```rust
use chrono::Local;
use tokio::{self, runtime::Runtime, time};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

async fn do_one() {
  println!("doing one: {}", now());
  time::sleep(time::Duration::from_secs(2)).await;
  println!("do one done: {}", now());
}

async fn do_two() {
  println!("doing two: {}", now());
  time::sleep(time::Duration::from_secs(1)).await;
  println!("do two done: {}", now());
}

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
      tokio::join!(do_one(), do_two());// 等待两个任务均完成，才继续向下执行代码
      println!("all done: {}", now());
    });
}
```

## 固定在线程内的本地异步任务: tokio::task::LocalSet

当使用多线程runtime时,tokio会协作式调度它管理的所有worker thread上的所有异步任务. 例如某个worker thread空闲后可能会从其它worker thread中偷取一些异步任务来执行, 或者tokio会主动将某些异步任务转移到不同的线程上执行. 这意味着, 异步任务可能会不受预料地被跨线程执行.

有时候并不想要跨线程执行,例如那些没有实现Send的异步任务, 它们不能跨线程,只能在一个固定的线程上执行.

要使用`tokio::task::LocalSet`, 需使用`LocalSet::new()`先创建好一个`LocalSet`实例, 它将生成一个独立的任务队列用来存放本地异步任务.之后便可以使用LocalSet的`spawn_local()`向该队列中添加异步任务. 但是添加的异步任务不会直接执行,只有对`LocalSet`调用`await`或调用`LocalSet::run_until()`或`LocalSet::block_on()`的时候,才会开始运行本地队列中的异步任务.调用后两个方法会进入`LocalSet`的上下文环境

```rust
use chrono::Local;
use tokio::{self, runtime::Runtime, time};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

fn main() {
    let rt = Runtime::new().unwrap();
    let local_tasks = tokio::task::LocalSet::new();

    // 向本地任务队列中添加新的异步任务，但现在不会执行
    local_tasks.spawn_local(async {
        println!("local task1");
        time::sleep(time::Duration::from_secs(5)).await;
        println!("local task1 done");
    });

    local_tasks.spawn_local(async {
        println!("local task2");
        time::sleep(time::Duration::from_secs(5)).await;
        println!("local task2 done");
    });

    println!("before local tasks running: {}", now());
    rt.block_on(async {
        // 开始执行本地任务队列中的所有异步任务，并等待它们全部完成
        local_tasks.await;
    });
}
```

除了`LocalSet::spawn_local()`可以生成新的本地异步任务, `tokio::task::spawn_local()`也可以生成新的本地异步任务,但是它的使用有个限制, 必须在LocalSet上下文内部才能调用

```rust
use chrono::Local;
use tokio::{self, runtime::Runtime, time};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

fn main() {
    let rt = Runtime::new().unwrap();
    let local_tasks = tokio::task::LocalSet::new();

    local_tasks.spawn_local(async {
        println!("local task1");
        time::sleep(time::Duration::from_secs(2)).await;
        println!("local task1 done");
    });

    local_tasks.spawn_local(async {
        println!("local task2");
        time::sleep(time::Duration::from_secs(3)).await;
        println!("local task2 done");
    });

    println!("before local tasks running: {}", now());
    // LocalSet::block_on进入LocalSet上下文
    local_tasks.block_on(&rt, async {
        tokio::task::spawn_local(async {
            println!("local task3");
            time::sleep(time::Duration::from_secs(4)).await;
            println!("local task3 done");
        }).await.unwrap();
    });
    println!("all local tasks done: {}", now());
}
```

需要注意的是, 调用`LocalSet::block_on()`和`LocalSet::run_until()`时均需指定一个异步任务(Future)作为其参数,它们都会立即开始执行该异步任务以及本地任务队列中已存在的任务, 但是这两个函数均只等待其参数对应的异步任务执行完成就返回. 这意味着,它们返回的时候,可能还有正在执行中的本地异步任务,它们会继续保留在本地任务队列中.当再次进入`LocalSet上`下文或`await LocalSet`的时候,它们会等待调度并运行.

```rust
use chrono::Local;
use tokio::{self, runtime::Runtime, time};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

fn main() {
    let rt = Runtime::new().unwrap();
    let local_tasks = tokio::task::LocalSet::new();

    local_tasks.spawn_local(async {
        println!("local task1");
        time::sleep(time::Duration::from_secs(2)).await;
        println!("local task1 done {}", now());
    });

    // task2要睡眠10秒，它将被第一次local_tasks.block_on在3秒后中断
    local_tasks.spawn_local(async {
        println!("local task2");
        time::sleep(time::Duration::from_secs(10)).await;
        println!("local task2 done, {}", now());
    });

    println!("before local tasks running: {}", now());
    local_tasks.block_on(&rt, async {
        tokio::task::spawn_local(async {
            println!("local task3");
            time::sleep(time::Duration::from_secs(3)).await;
            println!("local task3 done: {}", now());
        }).await.unwrap();
    });
    
    // 线程阻塞15秒，此时task2睡眠10秒的时间已经过去了，
    // 当再次进入LocalSet时，task2将可以直接被唤醒
    thread::sleep(std::time::Duration::from_secs(15));

    // 再次进入LocalSet
    local_tasks.block_on(&rt, async {
        // 先执行该任务，当遇到睡眠1秒的任务时，将出现任务切换，
        // 此时，调度器将调度task2，而此时task2已经睡眠完成
        println!("re enter localset context: {}", now());
        time::sleep(time::Duration::from_secs(1)).await;
        println!("re enter localset context done: {}", now());
    });
    println!("all local tasks done: {}", now());
}
```

需要注意的是, 再次运行本地异步任务时, 之前被中断的异步任务所等待的事件可能已经出现了, 因此它们可能会被直接唤醒重新进入就绪队列等待下次轮询调度. 正如上面需要睡眠10秒的task2, 它会被第一次block_on中断, 虽然task2已经不再执行, 但是15秒之后它的睡眠完成事件已经出现, 它可以在下次调度本地任务时直接被唤醒. 但注意, 唤醒的任务不是直接就可以被执行的, 而是放入就绪队列等待调度。

这意味着, 再次进入上下文时, 所指定的Future中必须至少存在一个会引起调度切换的任务, 否则该Future以同步的方式运行直到结束都不会给已经被唤醒的任务任何执行的机会.

下面是使用`run_until()`两次进入`LocalSet`上下文的示例, 和`block_on()`类似, 区别仅在于它只能在`Runtime::block_on()`内或`[tokio::main]`注解的main函数内部被调用.

```rust
use chrono::Local;
use tokio::{self, runtime::Runtime, time};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

fn main() {
    let rt = Runtime::new().unwrap();
    let local_tasks = tokio::task::LocalSet::new();

    local_tasks.spawn_local(async {
        println!("local task1");
        time::sleep(time::Duration::from_secs(5)).await;
        println!("local task1 done {}", now());
    });

    println!("before local tasks running: {}", now());
    rt.block_on(async {
        local_tasks
            .run_until(async {
                println!("local task2");
                time::sleep(time::Duration::from_secs(3)).await;
                println!("local task2 done: {}", now());
            })
            .await;
    });

    thread::sleep(std::time::Duration::from_secs(10));
    rt.block_on(async {
        local_tasks
            .run_until(async {
                println!("local task3");
                tokio::task::yield_now().await;
                println!("local task3 done: {}", now());
            })
            .await;
    });
    println!("all local tasks done: {}", now());
}
```

## tokio::select!宏

在Golang中有一个select关键字,tokio中则类似地提供了一个名为`select!`的宏.

`select!`宏的作用是轮询指定的多个异步任务,每个异步任务都是select!的一个分支,当某个分支已完成,则执行该分支对应的代码,同时取消其它分支. 简单来说, `select!`的作用是等待第一个完成的异步任务并执行对应任务完成后的操作.

```rust
use tokio::{self, runtime::Runtime, time::{self, Duration}};

async fn sleep(n: u64) -> u64 {
    time::sleep(Duration::from_secs(n)).await;
    n
}

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        tokio::select! {
          v = sleep(5) => println!("sleep 5 secs, branch 1 done: {}", v),
          v = sleep(3) => println!("sleep 3 secs, branch 2 done: {}", v),
        };

        println!("select! done");
    });
}
```

每个分支都有一个异步任务, 并对异步任务完成后的返回结果进行模式匹配,如果匹配成功,则执行对应的handler

`select!`本身是阻塞的, 只有`select!`执行完,它后面的代码才会继续执行.

默认情况下, `select!`会伪随机公平地轮询每一个分支, 如果确实需要让`select!`按照任务书写顺序去轮询,可以在`select!`中使用`biased`.

```rust
#[tokio::main]
async fn main() {
    let mut count = 0u8;
    loop {
        tokio::select! {
            // 如果取消biased，挑选的任务顺序将随机，可能会导致分支中的断言失败
            biased;
            _ = async {}, if count < 1 => { count += 1; assert_eq!(count, 1); }
            _ = async {}, if count < 2 => { count += 1; assert_eq!(count, 2); }
            _ = async {}, if count < 3 => { count += 1; assert_eq!(count, 3); }
            _ = async {}, if count < 4 => { count += 1; assert_eq!(count, 4); }
            else => { break; }
        };
    }
}
```

## JoinHandle::is_finished()

可使用`JoinHandle`的`is_finished()`方法来判断任务是否已终止,它是非阻塞的.

```rust
let task = tokio::spawn(async {
    tokio::time::sleep(tokio::time::Duration::from_secs(5)).await;
});

// 立即输出 false
println!("1 {}", task.is_finished());
tokio::time::sleep(tokio::time::Duration::from_secs(10)).await;
// 输出 true
println!("2 {}", task.is_finished());
```

`is_finished()`常用于在多个任务中轮询直到其中一个任务终止

## tokio task JoinSet

`tokio::task::JoinSet`用于收集一系列异步任务,并判断它们是否终止.
注意, 使用JoinSet的`spawn()`方法创建的异步任务才会被收集.

```rust
use tokio::task::JoinSet;

#[tokio::main]
async fn main() {
  let mut set = JoinSet::new();

  // 创建10个异步任务并收集
  for i in 0..10 {
    // 使用JoinSet的spawn()方法创建异步任务
    set.spawn(async move { i });
  }

  // join_next()阻塞直到其中一个任务完成
  set.join_next().await();
  set.abort_all();
}
```

如果要等待多个或所有任务完成, 则循环`join_next()`即可. 如果`JoinSet`为空,则该方法返回None.

```rust
while let Some(_) = set.join_next().await {}
```

使用JoinSet的`abort_all()`或直接`Drop JoinSet`, 都会对所有异步任务进行`abort()`操作.
使用JoinSet的`shutdown()`方法, 将先`abort_all()`,然后`join_next()`所有任务,直到任务集合为空.
使用JoinSet的`detach_all()`将使得集合中的所有任务都被detach, 即使JoinSet被丢弃,被detach的任务也依然会在后台运行
