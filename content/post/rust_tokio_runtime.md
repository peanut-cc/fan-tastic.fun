---
title: "Rust笔记----tokio:runtime"
date: 2023-08-11T10:54:37+08:00
tags: ["tokio"]
categories: ["Rust"]
keywords: ["tokio","runtime", "异步运行时环境"]
draft: false
---


使用tokio 需要先创建异步运行时环境(Runtime),然后在Runtime中执行异步任务.

```rust
use tokio;

fn main() {
    // 创建runtime
    let rt = tokio::runtime::Runtime::new().unwrap();

    // 创建带有线程池的runtime
    let rt2 = tokio::runtime::Builder::new_multi_thread()
        .worker_threads(8) // 8个工作线程
        .enable_io()       // 可在runtime中使用异步IO
        .enable_time()     // 可在runtime中使用异步计时器(timer)
        .build()
        .unwrap();
}
```

tokio提供两种工作模式的runtime:

- 单一线程的runtime
- 多线程的runtime

不过IO并发类任务较多时,由于多线程之间切换开销,多线程的runtime性能 不一定会比单一线程的runtime更快.
默认情况下创建出来的runtime都是多线程的runtime,且没有指定工作线程数量时,默认的工作线程数量将和CPU核数相同.

创建单一线程的runtime:

```rust
// 创建单一线程的runtime
let rt = tokio::runtime::Builder::new_current_thread().build().unwrap();
```

## async main

对于main函数,tokio提供了简化的创建方式:

```rust
use tokio;
#[tokio::main]
async fn main(){}
```

通过`#[tokio::main]`注解(annotation)，使得`async main`自身成为一个`async runtime`。
`#[tokio::main]`创建的是多线程runtime，还有以下几种方式创建多线程runtime：

```rust
#[tokio::main(flavor = "multi_thread")] // 等价于#[tokio::main]
#[tokio::main(flavor = "multi_thread", worker_threads = 10)]
#[tokio::main(worker_threads = 10)]
```

`#[tokio::main]`也可以创建单一线程的main runtime：

```rust
#[tokio::main(flavor = "current_thread")]
```

## 多个runtime共存

可手动创建线程,并在不同线程内创建互相独立的runtime.

```rust
use std::thread;
use std::time::Duration;
use tokio::runtime::Runtime;

fn main() {
    // 在第一个线程内创建一个多线程的runtime
    let t1 = thread::spawn(|| {
        let rt = Runtime::new().unwrap();
        thread::sleep(Duration::from_secs(10));
    });

    // 在第二个线程内创建一个多线程的runtime
    let t2 = thread::spawn(|| {
        let rt = Runtime::new().unwrap();
        thread::sleep(Duration::from_secs(10));
    });

    t1.join().unwrap();
    t2.join().unwrap();
}
```

## 在异步runtime中执行异步任务

```rust
use chrono::Local;
use tokio::runtime::Runtime;

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        println!("before sleep: {}", Local::now().format("%F %T.%3f"));
        tokio::time::sleep(tokio::time::Duration::from_secs(2)).await;
        println!("after sleep: {}", Local::now().format("%F %T.%3f"));
    });
}
```

`std::time`也提供了`sleep()`,但它会阻塞整个线程,而`tokio::time`中的`sleep()`则只是让它所在的任务放弃CPU并进入调度队列等待被唤醒,它不会阻塞任何线程,它所在的线程仍然可被用来执行其它异步任务.在`tokio runtime`中，应当使用`tokio::time中`的`sleep()`.

runtime的`block_on()` 方法要求一个Future 作为参数.每个Future都是一个已经定义好的但尚未执行的异步任务,每一个异步任务中可能会包含其它子任务.这些异步任务不会直接执行,需要先将它们放入到runtime环境,然后在合适的地方通过Future的await来执行它们.await可以将已经定义好的异步任务立即加入到runtime的任务队列中等待调度执行,await会等待该异步任务完成才返回.

```rust
rt.block_on(async {
    // 只是定义了Future，此时尚未执行
    let task = tokio::time::sleep(tokio::time::Duration::from_secs(2));
    // ...不会执行...
    // ...
    // 开始执行task任务，并等待它执行完成
    task.await;

    // 上面的任务完成之后，才会继续执行下面的代码
});
```

block_on会阻塞当前线程,直到其指定的**异步任务树(可能有子任务)**全部完成
block_on也有返回值，其返回值为其所执行异步任务的返回值

```rust
use tokio::{time, runtime::Runtime};

fn main() {
    let rt = Runtime::new().unwrap();
    let res: i32 = rt.block_on(async{
      time::sleep(time::Duration::from_secs(2)).await;
      3
    });
    println!("{}", res);  // 3
}
```

## spawn: 向runtime中添加新的异步任务

定义要执行的异步任务时, 并未身处runtime内部,此时可以使用`tokio::spawn()`来生成异步任务

```rust
use std::thread;

use chrono::Local;
use tokio::{self, runtime::Runtime, time};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

// 在runtime外部定义一个异步任务，且该函数返回值不是Future类型
fn async_task() {
  println!("create an async task: {}", now());
  tokio::spawn(async {
    time::sleep(time::Duration::from_secs(10)).await;
    println!("async task over: {}", now());
  });
}

fn main() {
    let rt1 = Runtime::new().unwrap();
    rt1.block_on(async {
      // 调用函数，该函数内创建了一个异步任务，将在当前runtime内执行
      async_task();
    });
}
```

除了`tokio::spawn()`, runtime自身也能spawn，可以传递runtime(注意要传递runtime的引用),然后使用runtime的`spawn()`

```rust
use tokio::{Runtime, time}
fn async_task(rt: &Runtime) {
  rt.spawn(async {
    time::sleep(time::Duration::from_secs(10)).await;
  });
}

fn main(){
  let rt = Runtime::new().unwrap();
  rt.block_on(async {
    async_task(&rt);
  });
}
```

## 进入runtime: 非阻塞的enter()

`block_on()`进入runtime时,会阻塞当前线程,`enter()`进入runtime时,不会阻塞当前线程,它会返回一个`EnterGuard`. `EnterGuard`没有其它作用,它仅仅只是声明从它开始的所有异步任务都将在runtime上下文中执行,直到删除该`EnterGuard`.

```rust
use tokio::{self, runtime::Runtime, time};
use chrono::Local;
use std::thread;

fn now() -> String {
  Local::now().format("%F %T").to_string()
}

fn main() {
    let rt = Runtime::new().unwrap();

    // 进入runtime，但不阻塞当前线程
    let guard1 = rt.enter();

    // 生成的异步任务将放入当前的runtime上下文中执行
    tokio::spawn(async {
      time::sleep(time::Duration::from_secs(5)).await;
      println!("task1 sleep over: {}", now());
    });

    // 释放runtime上下文，这并不会删除runtime
    drop(guard1);

    // 可以再次进入runtime
    let guard2 = rt.enter();
    tokio::spawn(async {
      time::sleep(time::Duration::from_secs(4)).await;
      println!("task2 sleep over: {}", now());
    });

    drop(guard2);

    // 阻塞当前线程，等待异步任务的完成
    thread::sleep(std::time::Duration::from_secs(10));
}
```

## tokio的两种线程：worker thread和blocking thread

tokio提供了两种功能的线程:

- 用于异步任务的工作线程(worker thread)
- 用于同步任务的阻塞线程(blocking thread)

有些必要的任务可能会长时间计算而占用线程,甚至任务可能是同步的,它会直接阻塞整个线程(比如`thread::time::sleep()`)
这类任务如果计算时间或阻塞时间较短,勉强可以考虑留在异步队列中,但如果任务计算时间或阻塞时间可能会较长,它们将不适合放在异步队列中,因为它们会破坏异步调度,使得同线程中的其它异步任务处于长时间等待状态,也就是说,这些异步任务可能会被饿很长一段时间.

例如,直接在runtime中执行阻塞线程的操作,由于这类阻塞操作不在tokio系统内,tokio无法识别这类线程阻塞的操作,tokio只能等待该线程阻塞操作的结束,才能重新获得那个线程的管理权.换句话说,worker thread被线程阻塞的时候,它已经脱离了tokio的控制,在一定程度上破坏了tokio的调度系统.

```rust
rt.block_on(async{
  // 在runtime中，让整个线程进入睡眠，注意不是tokio::time::sleep()
  std::thread::sleep(std::time::Duration::from_secs(10));
});
```

worker thread只用于执行那些异步任务, 异步任务指的是不会阻塞线程的任务
而一旦遇到本该阻塞但却不会阻塞的操作(如使用`tokio::time::sleep()`而不是`std::thread::sleep()`),会直接放弃CPU,将线程交还给调度器,使该线程能够再次被调度器分配到其它异步任务. blocking thread则用于那些长时间计算的或阻塞整个线程的任务.

`blocking thread`默认是不存在的,只有在调用了`spawn_blocking()`时才会创建一个对应的`blocking thread`
`blocking thread`不用于执行异步任务,因此runtime不会去调度管理这类线程,它们在本质上相当于一个独立的`thread::spawn()`创建的线程,它也不会像`block_on()`一样会阻塞当前线程,它和独立线程的唯一区别,是`blocking thread`是在runtime内的,可以在runtime内对它们使用一些异步操作, 例如await.

```rust

use std::thread;
use chrono::Local;
use tokio::{self, runtime::Runtime, time};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

fn main() {
    let rt1 = Runtime::new().unwrap();
    // 创建一个blocking thread，可立即执行(由操作系统调度系统决定何时执行)
    // 注意，不阻塞当前线程
    let task = rt1.spawn_blocking(|| {
      println!("in task: {}", now());
      // 注意，是线程的睡眠，不是tokio的睡眠，因此会阻塞整个线程
      thread::sleep(std::time::Duration::from_secs(10))
    });

    // 小睡1毫秒，让上面的blocking thread先运行起来
    std::thread::sleep(std::time::Duration::from_millis(1));
    println!("not blocking: {}", now());

    // 可在runtime内等待blocking_thread的完成
    rt1.block_on(async {
      task.await.unwrap();
      println!("after blocking task: {}", now());
    });
}
```

注意: `blocking thread`生成的任务虽然绑定了runtime. 但是它不是异步任务,不受tokio调度系统控制.因此如果在`block_on()`中生成了`blocking thread`或普通的线程,`block_on()`不会等待这些线程的完成.

## 关闭Runtime

由于异步任务完全依赖于Runtime,而Runtime又是程序的一部分,它可以轻易地被删除(drop),这时Runtime会被关闭(shutdown).

```rust
let rt = Runtime::new().unwrap();
...
drop(rt);
```

注意: 这种删除runtime句柄的方式只会立即关闭未被阻塞的`worker thread`, 那些已经运行起来的`blocking thread`以及已经阻塞整个线程的`worker thread`仍然会执行. 但是删除runtime又要等待runtime中的所有异步和非异步任务(会阻塞线程的任务)都完成,因此删除操作会阻塞当前线程.

```rust
use std::thread;
use chrono::Local;
use tokio::{self, runtime::Runtime, time};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

fn main() {
    let rt = Runtime::new().unwrap();
    // 一个运行5秒的blocking thread
    // 删除rt时，该任务将继续运行，直到自己终止
    rt.spawn_blocking(|| {
      thread::sleep(std::time::Duration::from_secs(5));
      println!("blocking thread task over: {}", now());
    });
    
    // 进入runtime，并生成一个运行3秒的异步任务，
    // 删除rt时，该任务直接被终止
    let _guard = rt.enter();
    rt.spawn(async {
      time::sleep(time::Duration::from_secs(3)).await;
      println!("worker thread task over 1: {}", now());
    });

    // 进入runtime，并生成一个运行4秒的阻塞整个线程的任务
    // 删除rt时，该任务继续运行，直到自己终止
    rt.spawn(async {
      std::thread::sleep(std::time::Duration::from_secs(4));
      println!("worker thread task over 2: {}", now());
    });
    
    // 先让所有任务运行起来
    std::thread::sleep(std::time::Duration::from_millis(3));

    // 删除runtime句柄，将直接移除那个3秒的异步任务，
    // 且阻塞5秒，直到所有已经阻塞的thread完成
    drop(rt);
    println!("runtime droped: {}", now());
}
```

关闭runtime可能会被阻塞,因此如果是在某个runtime中关闭另一个runtime,将会导致当前的runtime的某个worker thread被阻塞,甚至可能会阻塞很长时间,这是异步环境不允许的.

tokio提供了另外两个关闭runtime的方式：`shutdown_timeout()`和`shutdown_background()`. 前者会等待指定的时间,如果正在超时时间内还未完成关闭,将强行终止runtime中的所有线程.后者是立即强行关闭runtime.

```rust
use std::thread;
use chrono::Local;
use tokio::{self, runtime::Runtime, time};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

fn main() {
    let rt = Runtime::new().unwrap();

    rt.spawn_blocking(|| {
      thread::sleep(std::time::Duration::from_secs(5));
      println!("blocking thread task over: {}", now());
    });
    
    let _guard = rt.enter();
    rt.spawn(async {
      time::sleep(time::Duration::from_secs(3)).await;
      println!("worker thread task over 1: {}", now());
    });

    rt.spawn(async {
      std::thread::sleep(std::time::Duration::from_secs(4));
      println!("worker thread task over 2: {}", now());
    });
    
    // 先让所有任务运行起来
    std::thread::sleep(std::time::Duration::from_millis(3));

    // 1秒后强行关闭Runtime
    rt.shutdown_timeout(std::time::Duration::from_secs(1));
    println!("runtime droped: {}", now());
}
```

需要注意的是强行关闭Runtime,可能会使得尚未完成的任务的资源泄露.因此应小心使用强行关闭Runtime的操作.

## runtime Handle

tokio提供了一个称为`runtime Handle`的东西, 它实际上是runtime的一个引用,可以随意被clone.它可以`spawn()`生成异步任务,这些异步任务将绑定在其所引用的runtime中,还可以`block_on()`或`enter()`进入其所引用的`runtime`,此外还可以生成`blocking thread`.

```rust
let rt = Runtime::new().unwrap();
let handle = rt.handle();
handle.spawn(...)
handle.spawn_blocking(...)
handle.block_on(...)
handle.enter()
```
