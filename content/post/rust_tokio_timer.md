---
title: "Rust笔记----tokio中的timer"
date: 2023-08-22T01:14:13+08:00
tags: ["tokio"]
categories: ["Rust"]
keywords: ["tokio","tokio timer"]
draft: false
---

tokio 的timer模块中的几个概念:

- `Duration`类型：是对`std::time::Duration`的重新导出,两者等价
- `Instant`类型：从程序运行开始就单调递增的时间点,仅结合Duration一起使用
- `Sleep`类型：是一个`Future`, 通过调用`sleep()`或`sleep_until()`返回, 该`Future`本身不做任何事,它只在到达某个时间点(Instant)时完成
- `Interval`类型：是一个流式的间隔计时器, 通过调用`interval()`或`interval_at()`返回. `Interval`使用`Duration`来初始化, 表示每隔一段时间(即指定的Duration时长)后就产生一个值
- `Timeout`类型：封装异步任务, 并为异步任务设置超时时长,通过调用`timeout()`或`timeout_at()`返回. 如果异步任务在指定时长内仍未完成, 则异步任务被强制取消并返回Error

## 时长: tokio::time::Duration

`tokio::time::Duration`是对`std::time::Duration`的Re-exports,它两完全等价.

使用如下几种简便的方式构建各种单位的时长

- Duration::from_secs(3)：3秒时长
- Duration::from_millis(300)：300毫秒时长
- Duration::from_micros(300)：300微秒时长
- Duration::from_nanos(300)：300纳秒时长
- Duration::from_secs_f32(2.3)：2.3秒时长
- Duration::from_secs_f64(2.3)：2.3秒时长

对于构建好的Duration实例`dur = Duration::from_secs_f32(2.3)`,可以使用如下几种方法方便地提取、转换它的秒、毫秒、微秒、纳秒

- dur.as_secs()：转换为秒的表示方式, 2
- dur.as_millis(): 转换为毫秒表示方式,2300
- dur.as_micros(): 转换为微秒表示方式,2_300_000
- dur.as_nanos(): 转换为纳秒表示方式,2_300_000_000
- dur.as_secs_f32(): 小数秒表示方式,2.3
- dur.as_secs_f64(): 小数秒表示方式,2.3
- dur.subsec_millis(): 小数部分转换为毫秒精度的表示方式,300
- dur.subsec_micros(): 小数部分转换为微秒精度的表示方式,300_000
- dur.subsec_nanos(): 小数部分转换为纳秒精度的表示方式,300_000_000

Duration实例可以直接进行大小比较以及加减乘除运算

- checked_add(): 时长的加法运算, 超出Duration范围时返回None
- checked_sub(): 时长的减法运算, 超出Duration范围时返回None
- checked_mul(): 时长的乘法运算, 超出Duration范围时返回None
- checked_div(): 时长的除法运算, 超出Duration范围时(即分母为0)返回None
- saturating_add()：饱和式的加法运算, 超出范围时返回Duration支持的最大时长
- saturating_mul()：饱和式的乘法运算, 超出范围时返回Duration支持的最大时长
- saturating_sub()：饱和式的减法运算, 超出范围时返回0时长
- mul_f32()：时长乘以小数, 得到的结果如果超出范围或无效, 则panic
- mul_f64()：时长乘以小数, 得到的结果如果超出范围或无效, 则panic
- div_f32()：时长除以小数, 得到的结果如果超出范围或无效, 则panic
- div_f64()：时长除以小数, 得到的结果如果超出范围或无效, 则panic

## 时间点: tokio::time::Instant

`Instant`用于表示时间点, 主要用于两个时间点的比较和相关运算.

```rust
use tokio;
use tokio::time::Instant;
use tokio::time::Duration;

#[tokio::main]
async fn main() {
    // 创建代表此时此刻的时间点
    let now = Instant::now();
    
    // Instant 加一个Duration，得到另一个Instant
    let next_3_sec = now + Duration::from_secs(3);
    // Instant之间的大小比较
    println!("{}", now < next_3_sec);  // true
    
    // Instant减Duration，得到另一个Instant
    let new_instant = next_3_sec - Duration::from_secs(2);
    
    // Instant减另一个Instant，得到Duration
    // 注意，Duration有它的有效范围，因此必须是大的Instant减小的Instant，反之将panic
    let duration = next_3_sec - new_instant;
}
```

`tokio::time::Instant`有如下常用方法:

- `from_std()`: 将`std::time::Instant`转换为`tokio::time::Instant`
- `into_std()`: 将`tokio::time::Instant`转换为`std::time::Instant`
- `elapsed()`: 指定的时间点实例,距离此时此刻的时间点,已经过去了多久(返回Duration)
- `duration_since()`: 两个Instant实例之间相差的时长,要求`B.duration_since(A)`中的B必须晚于A,否则panic
- `checked_duration_since()`: 两个时间点之间的时长差,如果计算返回的Duration无效,则返回None
- `saturating_duration_since()`: 两个时间点之间的时长差,如果计算返回的Duration无效, 则返回0时长的Duration实例
- `checked_add()`: 为时间点加上某个时长,如果加上时长后是无效的Instant,则返回None
- `checked_sub()`: 为时间点减去某个时长,如果减去时长后是无效的Instant,则返回None

## 睡眠: tokio::time::Sleep

tokio的睡眠在进入睡眠后不做任何事, 仅仅只是立即放弃CPU,并进入任务轮询队列,等待睡眠时间终点到了之后被Reactor唤醒,然后进入就绪队列等待被调度

```rust
use tokio::{self, runtime::Runtime, time};

fn main(){
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        // 睡眠2秒
        time::sleep(time::Duration::from_secs(2)).await;

        // 一直睡眠，睡到2秒后醒来
        time::sleep_until(time::Instant::now() + time::Duration::from_secs(2)).await;
    });
}
```

`sleep()`或`sleep_until()`都返回`time::Sleep`类型, 它有3个方法可调用:

- `deadline()`: 返回Instant, 表示该睡眠任务的睡眠终点
- `is_elapsed()`: 可判断此时此刻是否已经超过了该sleep任务的睡眠终点
- `reset()`：可用于重置睡眠任务. 如果睡眠任务未完成,则直接修改睡眠终点,如果睡眠任务已经完成,则再次创建睡眠任务,等待新的终点

## 任务超时: tokio::time::Timeout

`tokio::time::timeout()`或`tokio::time::timeout_at()`可设置一个异步任务的完成超时时间, 前者接收一个Duration和一个Future作为参数,后者接收一个Instant和一个Future作为参数. 这两个函数封装异步任务之后,返回`time::Timeout`, 它也是一个Future.

如果在指定的超时时间内该异步任务已完成,则返回该异步任务的返回值,如果未完成,则异步任务被撤销并返回Err.

```rust
use chrono::Local;
use tokio::{self, runtime::Runtime, time};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        let res = time::timeout(time::Duration::from_secs(5), async {
            println!("sleeping: {}", now());
            time::sleep(time::Duration::from_secs(6)).await;
            33
        });

        match res.await {
            Err(_) => println!("task timeout: {}", now()),
            Ok(data) => println!("get the res '{}': {}", data, now()),
        };
    });
}
```

得到`time::Timeout`实例res后, 可以通过`res.get_ref()`或者`res.get_mut()`获得Timeout所封装的Future的可变和不可变引用,使用`res.into_inner()`获得所封装的Future, 这会消费掉该Future

如果要取消Timeout的计时等待,直接删除掉Timeout实例即可

## 间隔任务: tokio::time::Interval

`tokio::time::interval()`和`tokio::time::interval_at()`用于设置间隔性的任务

- `interval_at()`: 接收一个Instant参数和一个Duration参数, Instant参数表示间隔计时器的开始计时点,Duration参数表示间隔的时长
- `interval()`: 接收一个Duration参数,它在第一次被调用的时候立即开始计时

这两个函数只是定义了间隔计时器的起始计时点和间隔的时长,要真正开始让它开始计时,还需要调用它的`tick()`方法生成一个Future任务, 并调用await来执行并等待该任务的完成

```rust
use chrono::Local;
use tokio::{self, runtime::Runtime, time::{self, Duration, Instant}};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        println!("before: {}", now());

        // 计时器的起始计时点：此时此刻之后的5秒后
        let start = Instant::now() + Duration::from_secs(5);
        let interval = Duration::from_secs(1);
        let mut intv = time::interval_at(start, interval);

        // 该计时任务"阻塞"，直到5秒后被唤醒
        intv.tick().await;
        println!("task 1: {}", now());

        // 该计时任务"阻塞"，直到1秒后被唤醒
        intv.tick().await;
        println!("task 2: {}", now());

        // 该计时任务"阻塞"，直到1秒后被唤醒
        intv.tick().await;
        println!("task 3: {}", now());
    });
}
```

再看一个例子:

```rust
use chrono::Local;
use tokio::{
    self,
    runtime::Runtime,
    time::{self, Duration, Instant},
};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        println!("before: {}", now());

        let start = Instant::now() + Duration::from_secs(5);
        let interval = Duration::from_secs(1);
        let mut intv = time::interval_at(start, interval);

        time::sleep(Duration::from_secs(10)).await;
        intv.tick().await;
        println!("task 1: {}", now());
        intv.tick().await;
        println!("task 2: {}", now());
    });
}
```

打印结果如下:

```bash
before: 2023-08-21 19:18:26
task 1: 2023-08-21 19:18:36
task 2: 2023-08-21 19:18:36
```

输出结果中的task 1和task 2的时间点是相同的,说明第一次tick之后,并没有等待1秒之后再执行紧跟着的tick,而是立即执行
上面代码的运行逻辑:

- 定义5秒后开始的计时器intv, 该计时器内部有一个字段记录者下一次开始tick的时间点,其值为 `19:18:31`
- 睡眠10秒后,时间点到了`19:18:36`,此时第一次执行`intv.tick()`,它将生成一个异步任务,执行器执行时发现此时此刻的时间点已经超过该计时器内部记录的值,于是该异步任务立即完成并进入就绪队列等待调度,同时修改计时器内部的值为`19:18:32`
- 下一次执行tick的时候,此时此刻仍然是`19:18:36`, 已经超过了该计时器内部的`19:18:32`,因此计时任务立即完成

这是tokio Interval在遇到计时延迟时的默认计时策略，叫做`Burst`,tokio支持三种延迟后的计时策略.

Burst策略,冲刺型的计时策略,当出现延迟后,将尽量快地完成接下来的tick,直到某个tick赶上它正常的计时时间点
Delay策略,延迟性的计时策略,当出现延迟后,仍然按部就班地每隔指定的时长计时.

```rust
use chrono::Local;
use tokio::{self, runtime::Runtime};
use tokio::time::{self, Duration, Instant, MissedTickBehavior};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        println!("before: {}", now());

        let mut intv = time::interval_at(
            Instant::now() + Duration::from_secs(5),
            Duration::from_secs(2),
        );
        intv.set_missed_tick_behavior(MissedTickBehavior::Delay);

        time::sleep(Duration::from_secs(10)).await;

        println!("start: {}", now());
        intv.tick().await;
        println!("tick 1: {}", now());
        intv.tick().await;
        println!("tick 2: {}", now());
        intv.tick().await;
        println!("tick 3: {}", now());
    });
}
```

Skip策略,忽略型的计时策略,当出现延迟后,仍然所有已经被延迟的计时任务

```rust
use chrono::Local;
use tokio::{self, runtime::Runtime};
use tokio::time::{self, Duration, Instant, MissedTickBehavior};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        println!("before: {}", now());

        let mut intv = time::interval_at(
            Instant::now() + Duration::from_secs(5),
            Duration::from_secs(2),
        );
        intv.set_missed_tick_behavior(MissedTickBehavior::Skip);

        time::sleep(Duration::from_secs(10)).await;

        println!("start: {}", now());
        intv.tick().await;
        println!("tick 1: {}", now());
        intv.tick().await;
        println!("tick 2: {}", now());
        intv.tick().await;
        println!("tick 3: {}", now());
    });
}
```
