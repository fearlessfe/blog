---
title: "Rust Oop"
date: 2022-07-02T09:13:16+08:00
draft: true
---

## OOP

```rust
pub struct AveragedCollection {
  list: Vec<i32>,
  average: f64,
}

impl AveragedCollection {
  pub fn add(&mut self, value: i32) {
    self.list.push(value);
    self.update_average();
  }

  pub fn remove(&mut self) -> Option<i32> {
    let result = self.list.pop();
    match result {
      Some(value) => {
        self.update_average();
        Some(value)
      }
      None => None,
    }
  }

  pub fn average(&self) -> f64 {
    self.average
  }

  fn update_average(&mut self) {
    let total: i32 = self.list.iter().sum();
    self.average = total as f64 / self.list.len() as f64;
  }
}
```

rust 利用 trait 来实现多态


### 利用 trait object 实现不同类型的值

实现一个 GUI 程序

```rust
pub trait Draw {
  fn draw(&self);
}

pub struct Screen {
  pub components: Vec<Box<dyn Draw>>
}

impl Screen {
  pub fn run(&self) {
    for component in self.components.iter() {
      component.draw();
    }
  }
}

pub struct Button {
  pub width: u32,
  pub height: u32,
  pub label: String,
}

impl Draw for Button {
  fn draw(&self) {}
}
```

### 实现 OOP 设计模式

状态模式。实现一个博文的工作流，最终功能如下

1. 博文从空的草稿开始
2. 草稿完成后，需要被审核
3. 审核后会被发布

```rust
use blog::Post;

fn main() {
  let mut post = Post::new();

  post.add_text("I ate a salad for lunch today");
  assert_eq!("", post.content());

  post.request_review();
  assert_eq!("", post.content());

  post.approve();
  assert_eq!("I ate a salad for lunch today", post.content());
}

```

实现 POST

```rust
pub struct Post {
  state: Option<Box<dyn State>>,
  content: String,
}

impl Post {
  pub fn new() -> Post {
    Post {
      state: Some(Box::new(Draft {})),
      content: String::new(),
    }
  }

  pub fn add_text(&mut self, text: &str) {
    self.content.push_str(text);
  }

  pub fn content(&self) -> &str {
    ""
  }

  pub fn request_review(&mut self) {
    if let Some(s) = self.state.take() {
      self.state = Some(s.request_review());
    }
  }
}

trait State {
  fn request_review(self: Box<Self>) -> Box<dyn State>;
}

struct Draft {}

impl State from Draft {}
```