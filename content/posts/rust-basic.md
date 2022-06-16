---
title: "Rust Basic"
date: 2022-06-15T20:21:04+08:00
draft: false
---

## 通用概念

### 变量和可变性

#### 变量

rust 中变量默认是不可变的。下面的代码片段会报错，因为 x 是不可变的，不能重新赋值。
如果需要声明可变变量，需要加上关键字 mut

```rust
fn main() {
  let x = 5;
  // let mut x = 5;
  println!("x is: {}", x )
  x = 6;
  println!("x is: {}", x )
}
```

#### 常量

rust 中用 const 来声明常量，常量是不可变的。
常量的值在编译阶段就需要确定，因此不能是运行阶段计算出来的值。

常量的值在整个程序运行期间都是有效的。

#### 变量遮蔽

用 let 声明相同的变量，会屏蔽前面的同名变量。
下面的代码用num类型的变量遮蔽了前面的字符串类型变量。

```rust
fn main() {
  let x = "    ";
  let x = x.len();
  println!("x is: {}", x )
}
```

### 数据类型

rust 中每个值都有一个确定的类型。接下来我们将了解一下两种数据类型子集：标量和复合变量。

#### 标量类型

标量类型代表一个单一的值。rust 有四种标量类型： 整数，浮点数，布尔和字符。

整数类型：i是有符号，u是无符号。长度 8-16-32-64-128. isize 和 usize 类型和系统有关
32 位系统代表 i32或u32，64位系统代表 i64或u64.

```rust
fn main() {
  let a: i32 = 98_222;
  let b: i32 = 0xff;
  let c: i32 = 0x77;
  let d: i32 = 0b1111_0000;
  let e: u8 = b'A'; // 代表 Byte，与u8一致
}
```

#### 复合类型

将多个值组合成一个类型。rust原生类型有元组和数组。

##### 元组类型

元组将多个不同类型的值组合起来，元组的长度是固定的。

```rust
fn main() {
  let tup: (i32, f64, u8) = (500, 6.4, 1);
  // 通过解构赋值
  let (x,y,z) = tup;
  // 通过下标
  let a = tup.0;
  let b = tup.1;
}
```

##### 数组类型

数组中只能有一个类型，而且长度也是固定的。越界访问会报错。

```rust
fn main() {
  let a: [i32; 5] = [1, 2, 3, 4, 5];

  // 通过下标
  let first = a[0];
  let second = a[1];
}
```

### 函数

rust 用 fn 声明函数， main 函数是程序的入口。函数命名风格为 snake_case。

函数参数必须要明确类型

```rust
fn main() {
    println!("Hello, world!");

    another_function(5, 'h');
}

fn another_function(x: i32, unit_label: char) {
    println!("Another function.");
}
```

#### 语句和表达式

语句没有返回值，表达式有返回值。

```rust
fn main() {
    let y = 6; // 声明语句
    // 大括号中是一个表达式，最后一个没有逗号，代表返回值
    // y 的值是4
    let y = {
        let x = 3;
        x + 1
    };
}
```

#### 带返回值的函数

```rust
fn five() -> i32 {
    5
}

fn main() {
    let x = five();

    println!("The value of x is: {}", x);
}
```

### 控制流

#### if 表达式

if 是表达式，也会有返回值。

```rust
fn main() {
    let number = 3;

    if number < 5 {
        println!("condition was true");
    } else {
        println!("condition was false");
    }

    let condition = true;
    let number = if condition { 5 } else { 6 };
}
```

#### loop 循环

break 跳出当前循环。
loop 也是表达式，也有可以有返回值。

```rust
fn main() {
  let mut count = 0;
  'counting_up: loop {
    println!("count = {}", count);
    let mut remaining = 10;

    loop {
      println!("remaining = {}", remaining);
      if remaining == 9 {
        break;
      }
      if count == 2 {
        break 'counting_up;
      }
      remaining -= 1;
    }
    count += 1;
  }
  println!("End count = {}", count);
}
```

while 循环

```rust
fn main() {
    let a = [10, 20, 30, 40, 50];
    let mut index = 0;

    while index < 5 {
        println!("the value is: {}", a[index]);

        index += 1;
    }
}
```

for..in循环数组

```rust
fn main() {
    let a = [10, 20, 30, 40, 50];

    for element in a {
        println!("the value is: {}", element);
    }

    for number in (1..4).rev() {
        println!("{}!", number);
    }
    println!("LIFTOFF!!!");
}
```