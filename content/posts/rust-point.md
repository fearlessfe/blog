---
title: "Rust Point"
date: 2022-06-21T23:31:05+08:00
draft: true
---

## 智能指针

智能指针式一类数据结构，表现类似于指针，也拥有额外的元数据和功能。

智能指针和常规结构体的显著区别就是智能指针实现了 `Deref` 和 `Drop` trait。

* Deref trait 允许智能指针结构体表现的像引用
* Drop trait 定义了智能指针离开作用域时运行的代码

本章会介绍标准库中最常用的智能指针

* Box<T>,用于堆上分配值
* Rc<T>,引用计数类型，数据有多个所有者
* Ref<T> 和 RefMut<T>，通过 RefCell<T> 访问 （RefCell<T> 是运行时执行借用规则的类型）

同时会涉及 内部可变性（不可变类型暴露出改变起内部值的API）和 引用循环。

### Box<T>

最简单的智能指针是 `box`，类型为 `Box<T>`。其值在堆上，栈上面的是指向堆数据的指针。

box 使用场景

* 编译期间无法知道该类型占用多少空间
* 大数据量转移所有权时，不希望数据被复制
* 当希望拥有一个值，只关心它实现了某个trait，不关心具体的类型

递归类型

```rust
// 编译器无法知道 List 的大小
enum List {
    Cons(i32, List),
    Nil,
}
// 使用 Box 类型
enum List {
  Cons(i32, Box<List>),
  Nil,
}
```

box 仅仅指向堆上的数据，没有其他特别的能力。

`Deref` Trait 让智能指针和常规的引用一样。

常规引用

```rust
fn main() {
  let x = 5;
  let y = &x;

  assert_eq!(5, x);
  assert_eq!(5, *y); // Deref
  // assert_eq!(5, y);  报错，y为指针类型
}
```

使用 Box

```rust
fn main() {
  let x = 5;
  let y = Box::new(x);

  assert_eq!(5, x);
  assert_eq!(5, *y); // Deref
  // assert_eq!(5, y);  报错，y为指针类型
}
```

#### 自定义智能指针

```rust
use std::ops::Deref;

struct MyBox<T>(T);

impl<T> MyBox<T> {
  fn new(x: T) -> MyBox<T> {
    MyBox(x)
  }
}

impl<T> Deref for MyBox<T> {
  type Target = T;

  fn deref(&self) -> &Self::Target {
    &self.0  // tuple struct,返回 T
  }
}
```

如果 deref 方法返回的值而不是值的引用，则值的所有权会被转移。

deref 强制转换时 Rust 在函数或方法传参上的一种便利


### Rc<T> 智能指针

当一个值有多个所有者时，使用 Rc，引用计数（reference counting）的缩写。Rc智能在单线程中使用。

```rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}

use crate::List::{Cons, Nil};

fn main() {
    let a = Cons(5, Box::new(Cons(10, Box::new(Nil))));
    let b = Cons(3, Box::new(a));
    let c = Cons(4, Box::new(a));
}
```

上面的代码， b,c都有a的所有权，会报错。

```rust
enum List{
  Cons(i32, Rc<List>),
  Nil,
}


use crate::List::{Cons, Nil};
use std::rc::Rc;

fn main() {
    let a = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil)))));
    let b = Cons(3, Rc::clone(&a)); // clone 至少增加引用数，不拷贝值
    let c = Cons(4, Rc::clone(&a));
}
```

Rc 只能是不可变的引用。

### RefCell<T>

内部可变性是 Rust 中的一个设计模式，允许在有不可变引用时也可以改变数据。