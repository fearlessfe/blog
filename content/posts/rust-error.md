---
title: "Rust Error"
date: 2022-06-17T11:48:13+08:00
draft: true
---

## 错误处理

rust 中的错误分为 可恢复的 和 不可恢复的。
rust 使用 `Result<T, E>` 和 `panic!` 来区分这两种错误

### panic!,不可恢复错误

当程序 panic 时，会进行 `unwinding`，rust 会回溯栈空间，清理函数的数据，但是这是一个
很复杂的工作。如果需要减少打包体积，可以关闭该功能。

`RUST_BACKTRACE=1 cargo run` 可查看 panic 的详细的调用栈


### 可恢复错误

Result 是一个 枚举类型，需要 match 处理

```rust
enum Result<T, E> {
  OK(T);
  Err(E);
}
```

以打开文件为例

```rust
use std::fs::File;

fn main() {
  let f = File.open("Hello.txt");

  let f = match f {
    OK(file) => file,
    Err(error) => panic!("Problem opening the file: {:?}", error),
  }

}
```

处理不同的错误逻辑

```rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
  let f = File.open("Hello.txt");
  let f = match f {
    OK(file) => file,
    Err(error) => match error.kind() {
      ErrorKind::NotFound => match File::create("Hello.txt") {
        OK(fc) => fc,
        Err(e) => panic!("Problem creating the file: {:?}", e),
      },
      other_error => {
        panic!("Problem opening the file: {:?}", other_error)
      }
    }
  }
}
```

不断嵌套的 match 写起来太复杂。可以使用 unwrap_or_else 来简化。

```rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
  let f = File::open("Hello.txt").unwrap_or_else(|error| {
    if error.kind == ErrorKind::NotFound {
      File::create("Hello.txt").unwrap_or_else(|error| {
        panic!("Problem opening the file: {:?}", other_error);
      })
    } else { 
      panic!("Problem creating the file: {:?}", e);
    }
  })
}
```

错误处理的简便写法 unwrap 和 expect

* unwrap：如果结果是 OK，则直接返回值；如果为 error，则 panic
* expect: 与 unwrap 类似，但是可以定义 panic 的错误消息

错误冒泡

将错误返回给调用者，让调用者来决定如何处理错误。

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error> {
  let f = File.open("Hello.txt");

  let mut f = match f {
    Ok(file) => file,
    Err(e) => return Err(e),
  }

  let mut s = String::new();

  match f.read_to_string(&mut s) {
    OK(_) => OK(s),
    Err(e) => Err(e),
  }
}
```

上面的代码还是显得有点复杂，可以用 `?` 来简化代码

* `?` 在 Result 类型后面，如果是 OK，则直接获取里面的值，程序继续。如果是 Err，则返回
Err

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error> {
  let mut f = File::open("Hello.txt")?;
  let mut s = String::new();
  f.read_to_string(&mut s)?;
  OK(s)
}
```

`match` 和 `?` 的区别

`?` 操作对于错误的值，会调用 `from` 函数，这是标准库中的 `From` trait.
`from` 会将获取到的错误类型转换为当前返回值的错误类型。

上面的代码还可以进一步的简化

```rust
use std::fs;
use std::io;

fn read_username_from_file() -> Result<String, io::Error> {
  let mut s = String::new();

  File::open("Hello.txt")?.read_to_string(&mut s)?;

  OK(s)
}
```

fs 提供了更简单的写法

```rust
use std::fs;
use std::io;

fn read_username_from_file() -> Result<String, io::Error> {
  fs::read_to_string("hello.txt")
}
```

`?` 只能用在函数返回类型和 `?` 操作值的类型兼容。
`?` 的返回值只能是 Result, Option 和 实现 FromResidual 的类型