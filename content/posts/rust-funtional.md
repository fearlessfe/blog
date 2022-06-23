---
title: "Rust Funtional"
date: 2022-06-21T22:40:36+08:00
draft: true
---

## 函数式编程

rust 具有一些函数式编程的特性，如闭包和迭代器。这两个特性的性能也很好。

### 闭包

使用闭包的特性缓存值，或者惰性求值

```rust
struct Cacher<T>
where
  T: Fn(u32) -> u32,
{
  calculation: T,
  value: Option<u32>,
}

impl<T> Cacher<T>
where
  T: Fn(u32) -> u32,
{
  fn new(calculation: T) -> Cacher<T> {
    Cacher {
      calculation,
      value: None,
    }
  }

  fn value(&mut self, arg: u32) -> u32 {
    match self.value {
      Some(v) => v,
      None => {
        let v = (self.calculation)(arg);
        self.value = Some(v);
        v
      }
    }
  }
}
```

只有闭包可以捕获环境变量，普通的函数定义不能捕获环境中的变量。

下面代码中，equal 函数执行回报错；equal2 函数不回报错。

```rust
fn main() {
  let x = 4;
  fn equal(z: i32) -> bool {
    z == x
  }
  let equal2 = |z| -> z == x

  let y = 4;

  assert!(equal(y));
  assert!(equal2(y));
}
```

闭包可以通过三种方式捕获环境中的变量：获取所有权，可变借用和不可变借用。这三种方式对应如下的三种 trait

* FnOnce: 获取变量的所有权，不能被多次调用
* FnMut: 可变的借用值
* Fn: 不可变借用

当你创建一个闭包后，rust 会根据值如何被使用来推断具体的 trait。使用 move 关键字可以强制转移所有权。

```rust
fn main() {
  let x = vec![1,2,3,4];
  let equal_to_x = move |z| z == x; // move 关键字，x所有权被转移

  println!("can't use x here: {:?}", x);  // 无法访问x，报错

  let y = vec![1, 2, 3];

assert!(equal_to_x(y));
}
```

使用 move 关键字后，闭包还是会实现 Fn 或 FnMut，因为实现什么 trait 是由闭包怎么使用这些值决定的。

### 迭代器

rust 中定义了 Iterator 的 trait

```rust
pub trait Iterator {
  type item;

  fn next(&mut self) -> Option<Self::item>;
  // 省略默认方法
}
```

不同的迭代器方法，所有权是不一样的。

```rust
fn main() {
  let v1 = vec![1, 2, 3];

  // v1.iter() 不可变引用的迭代器
  // v1.into_iter() 获取v1的所有权
  // v1.iter_mut() 可变引用
}
```