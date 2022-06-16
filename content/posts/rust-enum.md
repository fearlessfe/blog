---
title: "Rust Enum"
date: 2022-06-16T19:12:50+08:00
draft: true
---

## 枚举和模式匹配

### 定义枚举

以 IP v4 和 v6 来定义枚举

```rust
enum IpAddrKind {
  V4,
  V6,
}

fn main() {
  let four = IpAddrKind::V4;
  let six = IpAddrKind::V6;
}

fn route(ip_kind: IpAddrKind) {

}
```

目前，我们只能知道类型，但是不知道具体的IP，所以我们可以定义以下类型。

```rust
enum IpAddrKind {
  V4,
  V6,
}

struct IpAddr {
  kind: IpAddrKind,
  address: String,
}

let home = IpAddr {
  kind: IpAddrKind::V4,
  address: String::from("127.0.0.1"),
}

let lookback = IpAddr {
  kind: IpAddrKind::V4,
  address: String::from("::1"),
}
```

上面，定义了 IpAddr 结构体，分别表示类型和值。但是，直接将值放到枚举中是一个更好的选择

```rust
enum IpAddrKind {
  V4(String),
  V6(String),
}

let home = IpAddr::V4(String::from("127.0.0.1"));

let loopback = IpAddr::V6(String::from("::1"));

enum IpAddrKind {
  V4(u8,u8,u8,u8),
  V6(String),
}

let home = IpAddr::V4(127, 0, 0, 1);

let loopback = IpAddr::V6(String::from("::1"));
```

枚举可以包含任意类型的值

```rust
enum Message {
  Quit,
  Move {x: i32, y: i32},
  Write(String),
  ChangeColr(i32,i32,i32),
}
// 将下面四种类型整合成一个枚举类型
struct QuitMessage; // unit struct
struct MoveMessage {
    x: i32,
    y: i32,
}
struct WriteMessage(String); // tuple struct
struct ChangeColorMessage(i32, i32, i32); // tuple struct
```

和结构体一样，枚举类型也可以实现方法

```rust
impl Message {
  fn call(&self) {}
}

let m = Message::Write(String::from("::1"));
m.call();
```

### Option 枚举

Option 是标准库中定义的枚举，代表一个值可能是something或者nothing。
标准库中 Option 的定义如下

```rust
enum Option<T> {
  None,
  Some(T),
}
```

Some,None,Option都是预置的，不需要引入。


```rust
let some_number = Some(1);
let some_string = Some("a string");

let absent_number: Option<i32> = None;
```

Option<T>, Some<T> 和 T 不是同一类型，需要转换。这时候需要使用 match 。

### match 控制流

match 是一个很强大的控制流结构，它可以匹配不同的类型来执行不同的逻辑。

将 match 想象为一个硬币分离机。

```rust
enum Coin {
  Penny,
  Nickel,
  Dime,
  Quarter,
}

fn value_in_cents(coin: Coin) -> u8 {
  match coin {
    Coin::Penny => 1,
    Coin::Nickel => 5,
    Coin::Dime => 10,
    Coin::Quarter => 25,
  }
}
```

match 的每个分支后面都是一个表达式，表达式的返回值就是match的返回值。

另一个很重要的功能是match分支可以绑定模式匹配的值，这样就可以从 enum 中取值。
还是上面的例子， Quarter 在不同的州有不同的设计，其余的硬币是统一的。因此可以定义如下的枚举。

```rust
#[derive(Debug)]
enum UsState {
  Alabama,
  Alaska,
}
enum Coin {
  Penny,
  Nickel,
  Dime,
  Quarter(UsState),
}

fn value_in_cents(coin: Coin) -> u8 {
  match coin {
    Coin::Penny => 1,
    Coin::Nickel => 5,
    Coin::Dime => 10,
    Coin::Quarter(state) => { // 可以取到枚举中的值 state
      println!("State quarter from {:?}!", state);
      25
    } 
  }
}

```

匹配 Option<T>

```rust
fn plus_one(x: Option<i32>) -> Option<i32> {
  match x {
    Node => Node,
    Some(i) => Some(i + 1),
  }
}

let five = Some(5);
let six = plus_one(five);
let none = plus_one(None);
```

rust 在编译阶段会检查 match 是否匹配了所有分支。match 还支持匹配剩余分支。下面代码中
other 代表其余的分支。如果不需要用到它的值，可用 `_` 代替。

```rust
let dice_roll = 9;
match dice_roll {
  3 => add_fancy_hat(),
  7 => remove_fancy_hat(),
  other => move_player(other),
  // _ => move_player(),
}

fn add_fancy_hat() {}
fn remove_fancy_hat() {}
fn move_player(num_spaces: u8) {}
```

#### if let

if let 语法可以只匹配某一个值，忽略其余的分支。

```rust
// match 必须匹配所有分支
let config_max = Some(3u8);
match config_max {
  Some(max) => println!("The maximum is configured to be {}", max),
  _ => (),
}

// 使用 if let 匹配特定的分支
let config_max = Some(3u8);
if let Some(max) = config_max {
  println!("The maximum is configured to be {}", max);
}
```

if let 后面还可以有 else 分支。以硬币的代码为例，可以改造成如下代码

```rust
let mut count = 0;
if let Coin::Quarter(state) = coin {
  println!("State quarter from {:?}!", state);
} else {
  count += 1;
}
```