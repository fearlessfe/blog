---
title: "Rust Test"
date: 2022-06-17T17:19:56+08:00
draft: true
---

## 测试

### 如何编写测试

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn exploration() {
        assert_eq!(2 + 2, 4);
    }

    #[test]
    fn another() {
        panic!("Make this test fail");
    }
}
```

使用 assert! 来检查结果，如果是true，则通过。否则调用 panic!

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    fn can_hold(&self, other: &Rectangle) -> bool {
        self.width > other.width && self.height > other.height
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn larger_can_hold_smaller() {
        let larger = Rectangle {
            width: 8,
            height: 7,
        };
        let smaller = Rectangle {
            width: 5,
            height: 1,
        };

        assert!(larger.can_hold(&smaller));
    }
}
```

assert 宏

* assert! ： 参数是否为 true
* assert_eq!：参数相等
* assert_ne！：参数不等

自定义错误消息

```rust
assert!(true, "show {}", "message")

// should_panic
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic]  // 应该
    fn greater_than_100() {
        Guess::new(200);
    }
}
// should_panic with 错误信息
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic(expected = "Guess value must be less than or equal to 100")]
    fn greater_than_100() {
        Guess::new(200);
    }
}
// Result
#[cfg(test)]
mod tests {
    #[test]
    fn it_works() -> Result<(), String> {
        if 2 + 2 == 4 {
            Ok(())
        } else {
            Err(String::from("two plus two does not equal four"))
        }
    }
}
```

### 控制如何测试

