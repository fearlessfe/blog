---
title: "Rust Ownership"
date: 2022-06-16T14:35:42+08:00
draft: false
---

## 所有权系统

所有权是rust管理内存的一系列的规则。

* rust 的每个值都有一个被称为 'owner' 的变量
* 每个时刻只能有一个 owner
* 当 owner 出作用域后，值会被丢弃

### String 类型

字符串字面量是不可变的，如果需要可变字符串，可以使用 String 类型，它的内存会分配在堆上

```rust
fn main() {
  let mut s = String.from("Hello")
  s.push_str(", world!");
}
```

### 变量与值的交互

下面的代码是简单的复制语句。首先我们需要了解 String 类型在栈上存的是一个简单的结构体
存储着字符串的长度，容量和指向实际内容的指针。

```rust
fn main() {
  let x = 5;
  let y = x;

  let s1 = String.from("hello");
  let s2 = s1;
}
```

#### move 语义

上面代码中执行 s2 = s1 后，rust 只会复制栈上的内容，堆上的内容没有变化。
同时 s1 将失效，可以理解为数据的所有权从 s1 转移到了 s2。这就是 move 语义，
代表所有权的转移。

#### clone 语义

如果需要 s1 和 s2 同时有效，需要使用 clone 方式，这会深度拷贝，不仅拷贝栈上的值，
也会拷贝堆上的值。代码如下

```rust
fn main() {

  let s1 = String.from("hello");
  let s2 = s1.clone();
}
```

#### copy 语义

最开始的代码中，执行 y = x 后，x 和 y 都是有效的。这是因为rust为栈上的数据类型默认
实现了 Copy 语义，直接复制栈上的数据。

需要注意的是，这个只有栈上数据类型才有copy语义。一下数据类型都自动实现了copy语义

* 整数和浮点数，如 u32，f64
* 布尔类型
* 字符类型
* 元组，其中的每个类型都实现了 Copy，(i32,u32)可以。(i32,String)不行
* 数组，和元组类似

#### 所有权和函数

函数传参和变量赋值类似，要么是 move，要么是 copy。

```rust
fn main() {
    let s = String::from("hello");  // s comes into scope

    takes_ownership(s);             // s's value moves into the function...
                                    // ... and so is no longer valid here

    let x = 5;                      // x comes into scope

    makes_copy(x);                  // x would move into the function,
                                    // but i32 is Copy, so it's okay to still
                                    // use x afterward

} // Here, x goes out of scope, then s. But because s's value was moved, nothing
  // special happens.

fn takes_ownership(some_string: String) { // some_string comes into scope
    println!("{}", some_string);
} // Here, some_string goes out of scope and `drop` is called. The backing
  // memory is freed.

fn makes_copy(some_integer: i32) { // some_integer comes into scope
    println!("{}", some_integer);
} // Here, some_integer goes out of scope. Nothing special happens.
```

#### 函数返回值

函数返回值也会有所有权的转移

```rust
fn main() {
    let s1 = gives_ownership();         // gives_ownership moves its return
                                        // value into s1

    let s2 = String::from("hello");     // s2 comes into scope

    let s3 = takes_and_gives_back(s2);  // s2 is moved into
                                        // takes_and_gives_back, which also
                                        // moves its return value into s3
} // Here, s3 goes out of scope and is dropped. s2 was moved, so nothing
  // happens. s1 goes out of scope and is dropped.

fn gives_ownership() -> String {             // gives_ownership will move its
                                             // return value into the function
                                             // that calls it

    let some_string = String::from("yours"); // some_string comes into scope

    some_string                              // some_string is returned and
                                             // moves out to the calling
                                             // function
}

// This function takes a String and returns one
fn takes_and_gives_back(a_string: String) -> String { // a_string comes into
                                                      // scope

    a_string  // a_string is returned and moves out to the calling function
}
```

### 引用和借用

如果我们只想将值传递给函数，但是不影响所有权，我们需要用到引用。

引用就像一个指针，它指向的地址就是实际的内容；但和指针不同的是，引用可以保证指向的有效的值。

```rust
fn main() {
  let s1 = String.from("Hello");

  let len = calculate_length(&s1);

  println!("The length of '{}' is {}.", s1, len);
}
// 参数 s 为一个指针，指针指向的是变量 s1
fn calculate_length(s: &String) -> usize {
  s.len()
} // & 表明是一个引用，没有所有权。所以函数结束后，所有权没有变化
```

我们将创建引用的行为称为借用。引用默认也是不可变的，如果在函数中修改引用就会报错。

如果需要修改引用，应创建可变引用。

```rust
fn main() {
  let mut s = String.from("hello");

  change(&mut s);
}

fn change(s: &mut String) {
  some_string.push_str(", world");
}

```

需要注意的是，在某一时刻只能有一个可变引用。

同一代码块中，需要注意 不可变引用和可变引用的生命周期。一下代码是正常的
```rust
fn main() {
  let mut s = String::from("hello");

    let r1 = &s; // no problem
    let r2 = &s; // no problem
    println!("{} and {}", r1, r2);
    // variables r1 and r2 will not be used after this point

    let r3 = &mut s; // no problem
    println!("{}", r3);
}
```

引用的准则

* 任意时刻，你可有一个可变引用或任意多的不可变引用。
* 引用一定是有效的

### 切片类型

切片让你引用一块连续的区间，而不是所有的集合。切片是引用，所以没有所有权。

现在有一个需求，需要找到字符串中的第一个单词。我们需要考虑该函数返回什么？可以考虑返回
第一个单词的位置。

```rust
fn main() {
  let mut s = String::from("hello world");

  let word = first_word(&s); // word will get the value 5

  s.clear(); // this empties the String, making it equal to ""

  // word still has the value 5 here, but there's no more string that
}

fn first_world(s: &String) -> usize {
  let bytes = s.as_bytes();

  for (i, &item) in bytes.iter().enumerate() {
    if item = b' ' {
      return i;
    }
  }
  s.len()
}
```

上述代码的问题就是返回的位置是没有意义的，即使字符串清空了，位置还是存在。
所以可以考虑切片。

```rust
fn main() {
  let s = String::from("hello");

  let slice = &s[0..2];  // index [0, 2)，不包含索引2
  let slice = &s[..2];
}
```

切片的位置必须是有效的 UTF-8 的分界，否则会出错。

string slice 的类型是 &str

```rust
fn first_word(s: &String) -> &str {
  let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}
```

前面说到字符串字面量直接存储在二进制文件中，所以字符串字面量就是字符串切片类型，&str.
除了字符串有切片，数组也有切片

```rust
let a = [1,2,3,4,5];

let slice = &a[1..3];

assert_eq!(slice, &[2, 3]);
```
