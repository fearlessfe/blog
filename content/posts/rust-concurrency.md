---
title: "Rust Concurrency"
date: 2022-07-01T21:25:51+08:00
draft: true
---

## 无畏并发

### spawn 创建并发

使用 `thread::spawn` 创建新的线程，传入一个闭包

主线程退出后，spawn 的线程也会退出

```rust
use std::thread;
use std::time::Duration;

fn main() {
 thread::spawn(|| {
  for i in 1..10 {
   println!("number {} from the spawned thread!", i);
   thread::sleep(Duration::from_millis(1));
  }
 })
 for i in 1..10 {
  println!("number {} from the main thread!", i);
  thread::sleep(Duration::from_millis(1));
 }
}
```

spawn 方法会返回一个 JoinHandler, 调用 join 方法可以让 spawn 线程执行完后退出

```rust
let handle := thread::spawn(|| {
  for i in 1..10 {
   println!("number {} from the spawned thread!", i);
   thread::sleep(Duration::from_millis(1));
  }
 })
 for i in 1..10 {
  println!("number {} from the main thread!", i);
  thread::sleep(Duration::from_millis(1));
 }
 handle.join().unwrap();
```

如果将 `handle.join().unwrap()` 放到主线程 for 循环前面。
则主线程会在 spawn 线程后执行

如果主线程的值传到 spawn 的新线程中，通过 move 关键字将值的所有权
转移到新的线程中

### 线程之间通过消息传递数据

rust 标准库实现了 channel，来实现线程之间的消息传递。

channel 有两个部分，发送者和接收者。接收者或发送者被 drop 后，channel 会被关闭

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
 let (tx, rx) = mspc::channel();

 thread::spawn(move ||{  // move tx to thread
  let val = String::from("hi");
  tx.send(val).unwrap();
 })

 let received = rx.recv().unwrap();
 println!("Get: {}", received);
}
```

channel 的 send 方法也会获取值的所有权。

receiver 方可以通过 for 循环来获取值

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
 let (tx, rx) = mpsc::channel();

 thread::spawn(move || {
  let vals = vec![
            String::from("hi"),
            String::from("from"),
            String::from("the"),
            String::from("thread"),
        ];

        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration::from_secs(1));
        }
 })
 // 程序一直循环，不回退出
 for received in rx {
  println!("Got: {}", received);
 }
}
```

复制 tx 来实现多个生产者

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
 let (tx, rx) = mpsc::channel();

 let tx1 = tx.clone();

 thread::spawn(move || {
  let vals = vec![
            String::from("hi"),
            String::from("from"),
            String::from("the"),
            String::from("thread"),
        ];

        for val in vals {
            tx1.send(val).unwrap();
            thread::sleep(Duration::from_secs(1));
        }
 })

 thread::spawn(move || {
  let vals = vec![
            String::from("hi"),
            String::from("from"),
            String::from("the"),
            String::from("thread"),
        ];

        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration::from_secs(1));
        }
 })
 // 程序一直循环，不回退出
 for received in rx {
  println!("Got: {}", received);
 }
}
```

### 共享状态 Mutex

```rust
use std::sync::Mutex;

fn main() {
 let m = Mutex::new(5);

 {
	// 调用 lock 方法获取 mutex 中的值
  let mut num = m.lock().unwrap();
  *num = 6;
 }
 println!("m = {:?}", m);
}
```

通过 Mutex 在多个线程共享数据

```rust
use std::sync::Mutex;
use std::thread;

fn main() {
	let counter = Mutex.new(0);
	let mut handles = vec![];

	for _ i 0..10 {
		let handle = thread::spawn(move ||{
			let mut num = counter.lock().unwrap();

			*num += 1;
		})
		handles.push(handle);
	}

	for handle in handles {
		handle.join().unwrap();
	}

	println!("Result: {}", *counter.lock().unwrap());
}
```

上面的代码会报错，因为 counter 的所有权已经被转移了。

对 mutex 用 RC 来管理也会报错。因为 RC 计算引用不是线程安全的。
可以使用 ARC 来管理引用

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
	let counter == Arc::new(Mutex::new(0));
	let mut handles = vec![];

	for _ i 0..10 {
		let counter = Arc::clonnne(&counter);
		let handle = thread::spawn(move ||{
			let mut num = counter.lock().unwrap();

			*num += 1;
		})
		handles.push(handle);
	}

	for handle in handles {
		handle.join().unwrap();
	}
}
```

counter 是不可变的，但是却可以更改它的值。是因为 Mutex 提供了内部可变性。

### Sync 和 Send traits

`Send` 表明值所有权的转移。几乎所有的类型都实现了 `Send`。但是 Rc 类型却没有。 