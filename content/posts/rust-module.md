---
title: "Rust Module"
date: 2022-06-16T21:57:05+08:00
draft: true
---

## rust 模块系统

* Packages：Cargo特性，可以编译，测试和分享 crates
* Crates：模块树，生成 lib 或者 bin
* Mudoles and use: 管理组织，范围和私有路径
* Paths: 命名方式

### Packages 和 Crate

一个 Package 包含一个或多个 crate。通过 Cargo.toml 来描述怎么编译 crate。

crate 可以分为 binary crate 和 library crate。binary crate 可以编译为二进制执行文件。library crate 没有 main 方法，不能编译为执行文件。

一个 package 至少包含一个 crate，可以有多个 binary crate，但是最多只能有一个 library crate。

使用一下命令可以创建一个 package

```shell
cargo new my-project
```

`src/main.rs` 是 binary crate 的入口
`src/lib.rs` 是 library crate 的入口

package 可以包含多个 binary crate。 `src/bin` 下的每个文件都是一个 binary crate。

### module and scope

模块加载流程

* 从 crate root 开始（src/lib.rs 或 src/main.rs）
* 声明模块： 在 crate root 文件中声明了一个模块，比如 `mod garden`,rustc会在一下路径加载这个模块
  * 当前文件，直接在模块声明后，用大括号包裹
  * 文件 `src/garden.rs`
  * 文件 `src/garden/mod.rs`
* 声明子模块：比如在 garden 模块中声明 vegetables 模块
  * 当前文件，直接在模块声明后，用大括号包裹
  * 文件 `src/garden/vegetables.rs`
  * 文件 `src/garden/vegetables/mod.rs`
* 模块中的代码：当模块编译成crate的一部分后，可以使用一下路径引用其中的方法 `crate::garden::vegetables::Asparagus`
* 只有声明为 pub 的属性或方法才可以被引用

### Paths

当需要在模块树中查找item时，我们用到path，这个和文件系统中的path类似。

path 有一下两种形式

* 绝对路径：从crate开始。内部从 crate 开始，外部从真正的 crate 名称开始
* 相对路径：从当前模块开始，使用 self， super 或当前模块中的标识
// the book 7.3