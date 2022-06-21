---
title: "Rust Traits"
date: 2022-06-17T15:36:40+08:00
draft: false
---

## 范型，traits和生命周期

### 范型

范型可以减少重复代码。

以下例子是从 i32 和 char 类型的 vec 中找到最大值。

```rust
fn largest_i32(list: &[i32]) -> i32 {
  let mut largest = list[0];

  for &item in list {
    if item > largest {
      largest = item;
    }
  }
  largest
}

fn largest_i32(list: &[char]) -> char {
  let mut largest = list[0];

  for &item in list {
    if item > largest {
      largest = item;
    }
  }
  largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let result = largest_i32(&number_list);
    println!("The largest number is {}", result);

    let char_list = vec!['y', 'm', 'a', 'q'];

    let result = largest_char(&char_list);
    println!("The largest char is {}", result);
}
```

使用范型重新定义函数

```rust
fn largest<T>(list: &[T]) -> T {
  let mut largest = list[0];

  for &item in list {
    if item > largest {
      largest = item;
    }
  }
  largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let result = largest(&number_list);
    println!("The largest number is {}", result);

    let char_list = vec!['y', 'm', 'a', 'q'];

    let result = largest(&char_list);
    println!("The largest char is {}", result);
}
```

上面的代码编译会报错，因为编译器无法确定类型 T 是否是可比较的。这是 traits 中的内容，会在下面介绍。

范型的用法

```rust
// struct
struct Point<T> {
  x: T,
  y: T,
}

struct Point<T, U> {
    x: T,
    y: U,
}

// enum
enum Option<T> {
  Some(T)，
  None,
}

enum Result<T, E> {
  OK(T),
  Err(E),
}

// 方法

impl<T> Point<T> {
  fn x(&self) -> &T {
    &self.x
  }
}

```

范型不存在性能问题，rust在编译节点会收集具体的类型，然后生成对应的方法

### traits，定义行为

trait 为类型定义行为。不同的类型去实现

```rust
pub trait Summary {
  fn summarize(&self) -> String;
}

pub struct NewsArticle {
  pub headline: String,
  pub lication: String,
  pub author: String,
  pub content: String,
}

impl Summary for NewsArticle {
  fn summarize(&self) -> String {
    format!("{}, by {} ({})", self.headline, self.author, self.location)
  }
}

pub struct Tweet {
    pub username: String,
    pub content: String,
    pub reply: bool,
    pub retweet: bool,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("{}: {}", self.username, self.content)
    }
}
```

有些时候，可以提供默认实现，这样其余的类型就不需要实现了。

```rust
pub trait Summary {
    fn summarize(&self) -> String {
        String::from("(Read more...)")
    }
}

// 使用默认实现
impl Summary for NewsArticle {}
```

在 trait 中可以调用 trait 下面的其他方法

```rust
pub trait Summary {
    fn summarize_author(&self) -> String;

    fn summarize(&self) -> String {
        format!("(Read more from {}...)", self.summarize_author())
    }
}
```

trait 作为函数参数

```rust
pub fn notify(item: &impl Summary) {
  println!("Breaking news! {}", item.summarize());
}
```

上面的写法是语法糖，下面是另外一种写法

```rust
pub fn notify<T: Summary>(item: &T) {
  println!("Breaking news! {}", item.summarize());
}
```

实现多个 trait


```rust

pub fn notify(item: &(impl Summary + Display)) {
  println!("Breaking news! {}", item.summarize());
}

pub fn notify<T: Summary + Display>(item: &T) {
  println!("Breaking news! {}", item.summarize());
}
```

当函数参数有很多个 trait 的时候，函数签名可读性就会降低，这时候，可以使用 where

```rust
fn some_function<T: Display + Clone, U: Clone + Debug>(t: &T, u: &U) -> i32 {}

fn some_function<T, U>(t: &T, u: &U) -> i32
  where T: Display + Clone,
        U: Clone + Debug
```

trait 作为返回值

```rust
fn returns_summarizable() -> impl Summary {
  Tweet {

  }
}
```

目前只能返回特定的类型,下面的代码根据 switch 的值返回不同的类型，编译会报错

```rust
fn returns_summarizable(switch: bool) -> impl Summary {
    if switch {
        NewsArticle {
            headline: String::from(
                "Penguins win the Stanley Cup Championship!",
            ),
            location: String::from("Pittsburgh, PA, USA"),
            author: String::from("Iceburgh"),
            content: String::from(
                "The Pittsburgh Penguins once again are the best \
                 hockey team in the NHL.",
            ),
        }
    } else {
        Tweet {
            username: String::from("horse_ebooks"),
            content: String::from(
                "of course, as you probably already know, people",
            ),
            reply: false,
            retweet: false,
        }
    }
}

```

修复 largest 函数

```rust
fn largest<T: PartialOrd + Copy>(list: &[T]) -> T {
    let mut largest = list[0];  // Copy trait

    for &item in list {
        if item > largest { // PartialOrd trait
            largest = item;
        }
    }

    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let result = largest(&number_list);
    println!("The largest number is {}", result);

    let char_list = vec!['y', 'm', 'a', 'q'];

    let result = largest(&char_list);
    println!("The largest char is {}", result);
}
```

利用 trait 控制特定方法的实现

```rust
use std::fmt::Display;

struct Pair<T> {
  x: T,
  y: T,
}

impl<T> Pair<T> {
  fn new(x: T, y: T) -> Self {
    Self{x, y}
  }
}
//  只有实现 Display + PartialOrd trait 的类型才可以调用 cmp_display 方法
impl<T: Display + PartialOrd> Pair<T> {
  fn cmp_display(&self) {
        if self.x >= self.y {
            println!("The largest member is x = {}", self.x);
        } else {
            println!("The largest member is y = {}", self.y);
        }
    }
}
```

同理，也可以选择性的实现其余 trait 如果当前type 实现了某个 trait
比如标准库中，为 T: Display 的类型实现了 ToString trait

```rust
impl<T: Display> ToString for T {

}
```

所以只要实现了 Display trait 的类型都可以调用 to_string() 方法

### 生命周期

使用生命周期避免 悬垂指针

```rust
{
  let r;

  {
    let x = 5;
    r = &x;
  }
  println!("r: {}", r);
}
```

上面的代码编译会报错。因为 x 会被清理，但是 r 是 x 的引用。println 时，x 已经释放了。

```rust
fn longest(x: &str, y:&str) -> &str {
  if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

上诉的 longest 编译会报错。因为无法确定到底返回的是 x 还是 y，需要加上 声明周期 标识。

```rust
&i32 // 引用
&‘a i32 // 显示的生命周期
&‘a mut i32 // 

fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

接下来longest函数是如何限制生命周期的

```rust
fn main() {
    let string1 = String::from("long string is long");
    // string1 声明周期大于 string2，代码正常运行
    {
        let string2 = String::from("xyz");
        let result = longest(string1.as_str(), string2.as_str());
        println!("The longest string is {}", result);
    }
}
// result 可能是 string2 的引用，print时， string2已经回收了
fn main() {
    let string1 = String::from("long string is long");
    let result;
    {
        let string2 = String::from("xyz");
        result = longest(string1.as_str(), string2.as_str());
    }
    println!("The longest string is {}", result);
}
```

声明周期注解的其他用法

```rust
// 结构体
struct ImportantExcerpt<'a> {
  part: &'a str;
}
// 方法
impl<'a> ImportantExcerpt<'a> {
    fn announce_and_return_part(&self, announcement: &str) -> &str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

静态生命周期 `'static`，在程序运行期间都是有效的。比如字符串字面量

范型，trait 和 生命周期

```rust
use std::fmt::Display;

fn longest_with_an_anno<'a, T>(
  x: &'a str,
  y: &'a str,
  ann: T
) -> &'a str
where
  T: Display,
{
  println!("Announcement! {}", ann);
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```