---
title: "rust库----serde_yaml"
date: 2022-10-23T20:31:42+08:00
tags: ["Rust", "serde_yaml"]
categories: ["Rust库使用"]
draft: true
---

## serde_yaml

今天写代码，需要将 yaml 文件中的数据序列化到一个结构体中，搜到了`Rust`的 `serde_yaml`库。

```rust
use std::collections::HashMap;

use serde::{Serialize, Deserialize};


#[derive(Debug,Serialize,Deserialize,PartialEq)]
struct User {
    name: String,
    age: i32,
}


fn main() {
    let yaml = "
        name: fan-tastic
        age: 18
    ";
    let result: User = serde_yaml::from_str(&yaml).unwrap();
    println!("{:?}", result);

    let map_result: HashMap<String, String> = serde_yaml::from_str(&yaml).unwrap();
    println!("{:?}", map_result);
}
```

## 复杂例子

而我需要的处理的数据如下：

```yaml
- name: 首页
  icon: dashboard
  router: "/dashboard"
  sequence: 9
- name: 系统管理
  icon: setting
  sequence: 7
  children:
    - name: 菜单管理
      icon: solution
      router: "/system/menu"
      sequence: 9
      actions:
        - code: add
          name: 新增
          resources:
            - method: POST
              path: "/api/v1/menus"
        - code: edit
          name: 编辑
          resources:
            - method: GET
              path: "/api/v1/menus/:id"
            - method: PUT
              path: "/api/v1/menus/:id"
        - code: del
          name: 删除
          resources:
            - method: DELETE
              path: "/api/v1/menus/:id"
        - code: query
          name: 查询
          resources:
            - method: GET
              path: "/api/v1/menus"
            - method: GET
              path: "/api/v1/menus.tree"
        - code: disable
          name: 禁用
          resources:
            - method: PATCH
              path: "/api/v1/menus/:id/disable"
        - code: enable
          name: 启用
          resources:
            - method: PATCH
              path: "/api/v1/menus/:id/enable"
    - name: 角色管理
      icon: audit
      router: "/system/role"
      sequence: 8
      actions:
        - code: add
          name: 新增
          resources:
            - method: GET
              path: "/api/v1/menus.tree"
            - method: POST
              path: "/api/v1/roles"
        - code: edit
          name: 编辑
          resources:
            - method: GET
              path: "/api/v1/menus.tree"
            - method: GET
              path: "/api/v1/roles/:id"
            - method: PUT
              path: "/api/v1/roles/:id"
        - code: del
          name: 删除
          resources:
            - method: DELETE
              path: "/api/v1/roles/:id"
        - code: query
          name: 查询
          resources:
            - method: GET
              path: "/api/v1/roles"
        - code: disable
          name: 禁用
          resources:
            - method: PATCH
              path: "/api/v1/roles/:id/disable"
        - code: enable
          name: 启用
          resources:
            - method: PATCH
              path: "/api/v1/roles/:id/enable"
    - name: 用户管理
      icon: user
      router: "/system/user"
      sequence: 7
      actions:
        - code: add
          name: 新增
          resources:
            - method: GET
              path: "/api/v1/roles.select"
            - method: POST
              path: "/api/v1/users"
        - code: edit
          name: 编辑
          resources:
            - method: GET
              path: "/api/v1/roles.select"
            - method: GET
              path: "/api/v1/users/:id"
            - method: PUT
              path: "/api/v1/users/:id"
        - code: del
          name: 删除
          resources:
            - method: DELETE
              path: "/api/v1/users/:id"
        - code: query
          name: 查询
          resources:
            - method: GET
              path: "/api/v1/users"
            - method: GET
              path: "/api/v1/roles.select"
        - code: disable
          name: 禁用
          resources:
            - method: PATCH
              path: "/api/v1/users/:id/disable"
        - code: enable
          name: 启用
          resources:
            - method: PATCH
              path: "/api/v1/users/:id/enable"
```

将上述的数据通过定义结构体，实现将数据序列化到定义的结构体中：

```rust
use std::fs::read_to_string;

use serde::{Serialize, Deserialize};


#[derive(Debug, PartialEq, Serialize, Deserialize)]
pub struct MenuActionResource {
    pub id: Option<String>,
    pub action_id: Option<String>,
    pub method: String,
    pub path: String,
}
#[derive(Debug, PartialEq, Serialize, Deserialize)]
pub struct MenuAction {
    pub id: Option<String>,
    pub menu_id: Option<String>,
    pub code: String,
    pub name: String,
    pub actions: Option<Vec<MenuActionResource>>,
}

#[derive(Debug, PartialEq, Serialize, Deserialize)]
pub struct MenuTree {
    pub id: Option<String>,
    pub name: String,
    pub icon: String,
    pub router: Option<String>,
    pub parent_id: Option<String>,
    pub parent_path: Option<String>,
    pub sequence: i32,
    pub is_show: Option<i32>,
    pub status: Option<i32>,
    pub menu_actions: Option<Vec<MenuAction>>,
    pub children: Option<Vec<Box<MenuTree>>>,
}

fn main() {
    let content = read_to_string("data.yml").unwrap();
    let result: Vec<MenuTree> = serde_yaml::from_str(&content).unwrap();
    println!("{:?}", result);
}
```

这个库还是挺方便的，即使是一些负载的数据结构体，也可以很方便的进行处理。

## 链接

- <https://docs.rs/serde_yaml/latest/serde_yaml/>
