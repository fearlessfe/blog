---
title: "Rust Struct"
date: 2022-06-16T17:19:16+08:00
draft: false
---

## 结构体

### 定义,初始化和赋值

定义结构体使用 struct 关键字

```rust
struct User {
  active: bool,
  username: String,
  email: String,
  sign_in_count: u64,
}

fn main() {
  let user1 = User {
    email: String::from("someone@example.com"),
    username: String::from("someusername123"),
    active: true,
    sign_in_count: 1,
  };
  user1.email = String::from("someone@example.com");
}
```

整个结构体都是可以修改的，不能将某个字段标记为不可修改。

结构体赋值和更新

```rust
struct User {
  active: bool,
  username: String,
  email: String,
  sign_in_count: u64,
}

fn main() {
  let user1 = User {
    email: String::from("someone@example.com"),
    username: String::from("someusername123"),
    active: true,
    sign_in_count: 1,
  };
  user1.email = String::from("someone@example.com");
  // 类似于 js 的解构赋值
  user2 = User {
    email: String::from("someone@example.com"),
    ..user1
  };
}
```

tuple struct 和 unit struct

```rust
struct Color(i32,i32,i32)

struct Unit;

fn main() {
  let black = Color(0,0,0);
  let unit = Unit;
}
```

### 结构体数据的所有权

定义 User 结构体时，email 数据类型是 String，这样每个结构体都有自己的数据。
结构体的字段也可以是引用，但是这样就需要标识声明周期。

### 方法

方法定义在结构体的上下文中，第一个参数总是 self

```rust
#[derive(Debug)]
struct Rectangle {
  width: u32,
  height: u32,
}
// 在结构体上下文中，Self 就是当前结构体（Rectangle）的别名
// &self 是 self: &Self 的简写
// 如果需要拿到 Self 的所有权，则传 self,等效于 self: Self
// 如果需要可变引用，则传 &mut self, 等效于 self: &mut Self
impl Rectangle {
  fn area(&self) {
    self.width * self.height
  }
}

fn main() {
    let rect1 = Rectangle {
        width: 30,
        height: 50,
    };

    println!(
        "The area of the rectangle is {} square pixels.",
        rect1.area()
    );
}
```

### 关联函数

在结构体的 impl 块中，第一个参数不为 self 的函数称为关联方法。如 String::from 就是 String 的关联方法。

```rust
impl Reactangle {
  fn square(size: u32) -> Reactangle {
    Rectangle {
      width: size,
      height: size,
    }
  }
}

fn main() {
  Reactangle::square(20);
}
```

一个结构体可以有多个 impl 块，这在语法上是正确的。多个 impl 块在范型和traits中很有用。