---
title: "PostgresSQL 触发器"
date: 2022-10-15T16:21:08+08:00
draft: false
tags: ["PostgresSQL", "数据库","中间件"]
categories: ["PostgresSQL"]
---

## 触发器是什么

在说触发器是什么之前，很多人可能和我一样，在很多开发资料中，或者和DBA的沟通中，都会不建议使用触发器，在实际业务中使用触发器的也可能比较少，在最近了解其他项目业务逻辑时，发现了触发器的一个使用方案，觉得在某些情况下通过触发器实现一些简单的方案还是挺好的。加上之前自己在业务开发中也确实没有尝试通过触发器来做实现方案，所以了解一下触发器，以及触发器的使用，也正好是一个学习的过程。

触发器（Trigger）是由事件自动触发执行的一种特殊的存储过程，触发事件可以是对一个表进行INSERT、UPDATE、DELETE等操作。触发器主要也是用于加强数据的完整性约束和业务规则上的约束。

## 怎么创建触发器

创建触发器的步骤：

- 为触发器创建一个执行函数
- 创建一个触发器
  
这里通过一个简单的例子了解：

1. 创建一个student 表，字段id,user_name
2. 创建一个student_trigger 表，字段id,trigger_schema,trigger_table,trigger_action,trigger_id
3. 通过触发器实现，当student数据增删改的时候，在student_trigger表中增加一条数据，告诉是哪个scheme下，哪个表发生了什么操作，以及当前行的id

### 创建触发器执行函数

```sql
create or replace function student_insert_delete_update_trigger()
returns trigger as $$
    begin
        if (tg_op = 'INSERT') then
            insert into public.student_trigger ("trigger_schema", "trigger_table", "trigger_action", "trigger_id") values (tg_table_schema,tg_table_name,tg_op, NEW.id);
        elsif (tg_op = 'UPDATE') then
            insert into public.student_trigger ("trigger_schema", "trigger_table", "trigger_action", "trigger_id") values (tg_table_schema,tg_table_name,tg_op, NEW.id);
        elsif (tg_op = 'DELETE') then
            insert into public.student_trigger ("trigger_schema", "trigger_table", "trigger_action", "trigger_id") values (tg_table_schema,tg_table_name,tg_op, OLD.id);
        end if;
        return null;
    end
$$
language plpgsql;
```

### 创建触发器

为student 表创建触发器，触发执student_insert_delete_update_trigger函数

```sql
create trigger student_trigger
    after insert or delete or update on student
    for row execute procedure student_insert_delete_update_trigger();
```

当尝试在student 表进行增删改的时候，在student_trigger 表中即可以看到插入的数据如下：

| id  | trigger_schema | trigger_talbe | trigger_action | trigger_id |
| --- | -------------- | ------------- | -------------- | ---------- |
| 20  | public         | student       | INSERT         | 1          |
| 21  | public         | student       | INSERT         | 2          |
| 22  | public         | student       | DELETE         | 2          |
| 23  | public         | student       | UPDATE         | 1          |

## 触发器详解

创建触发器的语法：

```sql
CREATE [ CONSTRAINT ] TRIGGER name { BEFORE | AFTER | INSTEAD OF } { event [ OR ... ] } ON table_name [ FROM referenced_table_name ] { NOT DEFERRABLE | [ DEFERRABLE ] { INITIALLY IMMEDIATE | INITIALLY DEFERRED } } [ FOR [ EACH ] { ROW | STATEMENT } ] [ WHEN ( condition ) ] EXECUTE PROCEDURE function_name ( arguments )
```

### 触发器的分类

触发器分为：

- 语句触发器：执行每个SQL语句时只执行一次。
- 行级触发器：执行每行SQL语句时都会执行一次。

创建触发器的语法中 `[ FOR [ EACH ] { ROW | STATEMENT } ]` 就是表示你要使用哪种触发器，`ROW`即行级触发器，`STATEMENT`即语句触发器。

触发器也可以分为：

- BEFORE触发器
- AFTER触发器

语句级别的BEFORE触发器是在语句开始做任何事情之前就被触发了的，而语句级别的AFTER触发器是在语句结束时才触发的。行级别的BEFORE触发器是在对特定行进行操作之前触发的，而行级别的AFTER触发器是在语句结束时才触发的，但是它会在任何语句级别的AFTER触发器之前触发。

删除触发器语法

```sql
DROP TRIGGER [ IF EXISTS ] name ON table [ CASCADE | RESTRICT ];
```

- IF EXISTS：如果指定的触发器不存在，那么发出一个notice而不是抛出一个错误。
- CASCADE：级联删除依赖此触发器的对象。
- RESTRICT：默认值，有依赖对象存在就拒绝删除。

关于触发器返回值：

- 语句级触发器应该总是返回NULL，即必须显式地在触发器函数中写上“RETURN NULL”，如果没有写将导致报错。
- 对于BEFORE和INSTEAD OF这类行级触发器来说，如果返回的是NULL，则表示忽略对当前行的操作。如果是返回非NULL的行，对于INSERT和UPDATE操作来说，返回的行将成为被插入的行或者是将要更新的行。
- 对于AFTER这类行级触发器来说，其返回值会被忽略。

### 特殊变量

在前面的例子中我们使用了几个特殊的变量：new，old，tg_table_schema，tg_table_name，tg_op。其实还有一些其他常用变量,这里对一些常用的变量进行说明：

- NEW:该变量为INSERT/UPDATE操作触发的行级触发器中存储新的数据行，数据类型是“RECORD”。在语句级别的触发器中此变量未分配，DELETE操作触发的行级触发器中此变量也未分配。
- OLD: 该变量为UPDATE/DELETE操作触发的行级触发器中存储原有的数据行，数据类型是“RECORD”。在语句级别的触发器中此变量未分配，INSERT操作触发的行级触发器中此变量也未分配。
- TG_NAME：数据类型是name类型，该变量包含实际触发的触发器名。
- TG_WHEN：内容为“BEFORE”或“AFTER”字符串用于指定是BEFORE触发器还是AFTER触发器。
- TG_LEVEL：内容为“ROW”或“STATEMENT”字符串用于指定是语句级触发器还是行级触发器。
- TG_OP：内容为“INSERT”“UPDATE”“DELETE”“TRUNCATE”之一的字符串，用于指定DML语句的类型。
- TG_TABLE_NAME：触发器所在表的名称。
- TG_TABLE_SCHEMA：触发器所在表的模式。

## 触发器的一个应用场景

触发器在这个场景下的解决方案可能不是最佳的，这里只是分享一种使用方式
在租户场景下，假设我们每一个租户一个schema，并且每个租户schema下的表都是相同的，同时随着租户的增加，会不断增加新的租户schema。这个时候我们有一个需求，我们需要监控所有租户下某些表数据的增删改，这个时候随着租户数量的不断增加，可能这个监控的表越来越多。如何更好的解决这个问题？

触发器在这个场景下我们可以将监控的表的数量缩减到一个表上，我们针对所有的要监控的表创建触发器，将增删改的出发通知到我们创建的一个trigger_table，假设我们设置的字段为：
"trigger_schema", "trigger_table", "trigger_action", "trigger_id"
即会通知是哪个租户schema 下的哪个table 发生了什么操作，并且变更行的id是什么？
这样我们就只需要通过一个单独的程序监控这个trigger_table 表的变化，定时查询这个表的数据去做处理。

## 相关链接

- <https://www.postgresql.org/docs/current/plpgsql-trigger.html>
