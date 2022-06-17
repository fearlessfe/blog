---
title: "Rust Collections"
date: 2022-06-16T23:09:25+08:00
draft: false
---

## 常用集合

* vector： 向量
* string: 字符串
* hash map

### vector

初始化向量

```rust
// 未赋值，必须显示的制定类型
let v: Vec<i32> = Vec::new();

// 如果需要初始化并赋值，可以使用 vec! 宏
// rust会自动推断类型，默认为 i32
let v = vec![1,2,3]
```

更新向量

```rust
// 未赋值，必须显示的制定类型
let v: Vec<i32> = Vec::new();

v.push(2)
// 当 v 的作用域结束后，vec会自动释放
```

读取向量中的元素

```rust
let v = vec![1,2,3,4,5];

let third: &i32 = &v[2];
println!("The third element is {}", third);
// get 方法返回的是 Option<&T>
match v.get(2) {
  Some(third) => println!("The third element is {}", third),
  None => println!("There is no third element."),
}
```

向量所有权.下面的代码会报错。
first 是第一个元素的不可变引用，当向 vec 中push 元素时，可能会引起扩容，导致
first 引用错误的内存地址。所以下面的代码无法编译通过。

```rust
    let mut v = vec![1, 2, 3, 4, 5];

    let first = &v[0];

    v.push(6);

    println!("The first element is: {}", first);
```

遍历 vec

```rust

let v = vec![100, 20,33];
// 不可变引用
for i in &v {
  println!("{}", i);
}
// 可变引用
for i in &mut v {
  *i += 50;
}
```

使用 Enum 存储多个类型

```rust
enum SpreadsheetCell {
  Int(i32),
  Float(f64),
  Text(String),
}

let row = vec![
  SpreadsheetCell::Int(3),
  SpreadsheetCell::Text(String::from("blue")),
  SpreadsheetCell::Float(10.12),
];
```

### String

创建 String。String 是 utf-8 编码。

```rust
let mut s = String::new();

let data =  "initial contents";

let s1 = data.to_string();
```

更新 string

```rust
let mut s = String::new();
// push_str 
s.push_str("hello");

s.push('l');

let s1 = String::from("Hello ");
let s2 = String::from("world!");

let s3 = s1 + &s2; // s1 的所有权移到了 s3
```

在标准库中， `+` 操作的签名如下。第一个参数是 self,有所有权的转移。
直接将 s 的内容追加到 self 中。

```rust
fn add(self, s: &str) -> String {}
```

多个字符串拼接可以用 format! 宏.
format! 不会拿字符串的所有权

```rust
    let s1 = String::from("tic");
    let s2 = String::from("tac");
    let s3 = String::from("toe");

    let s = format!("{}-{}-{}", s1, s2, s3);
```

字符串索引。我们不能直接通过下标获取字符串中的某个字符
以下语句是非法的。

```rust
let s1 = String::from("hello");
let h = s1[0];
let h = &s1[..][0];
```

遍历字符串. chars() 和 bytes().

```rust
    let s1 = String::from("hello");
    for c in s1.chars() {
        println!("{}",c );
    }

    for b in s1.bytes() {
        println!("{}",b );
    }
```

### Hash Map

新建 Hash Map

```rust
use std::collections::HashMap;

let mut scores = HashMap.new();

scores.insert(String::from("blue"),10);

scores.insert(String::from("yellow"),50);

// keys 和 values 构建 map

let keys = vec![String::from("blue"), String::from("yellow")];
let values = vec![10, 50];

let mut map: HashMap<_, _> = keys.into_iter().zip(values.into_iter()).collect();
```

插入的key和value会根据语义转移所有权

map 的值也可以是引用，这样值的所有权不回转移。但是要注意生命周期，在map有限期间都能访问
到引用的值。

```rust
use std::collections::HashMap;

let mut map = HashMap.new();

let field_name = String::from("Favorite color");
let field_value = 32;
// field_name 所有权转移，field_value是 copy
map.insert(field_name, field_value);

let value = map.get(&field_name); // value 是 Some(&T) 类型

for (key, value) in &map {
  println!("{}: {}", key, value);
}
```

更新hash map

```rust
use std::collections::HashMap;

let mut map = HashMap.new();

let field_name = String::from("Favorite color");
let field_value = 32;
// field_name 所有权转移，field_value是 copy
map.insert(field_name, field_value);
// 覆盖原来的值
map.insert(String::from("Favorite color"), 10);
map.insert(String::from("Blue"), 25);

// 判断key是否存在，不存在则设置新的值,存在则忽略
// entry 返回的是值的可变引用
map.entry(String::from("Favorite color")).or_insert(50);

// 在原值的基础上操作

let value = map.entry(String::from("Favorite color")).or_insert(50);
*value += 1;
```