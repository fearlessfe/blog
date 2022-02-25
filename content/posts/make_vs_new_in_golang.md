---
title: "make VS new in golang"
date: 2022-02-25T12:50:34+08:00
draft: true
---

在 golang 里面，make 和 new 是两个内存分配的原语，两者区别如下：

* [make](https://go.dev/doc/effective_go#allocation_make): make(T, args),只能用来创建 slice, map 和 channel,返回的是初始化后的 T（这三种类型必须初始化后才能使用）。
* [new](https://go.dev/doc/effective_go#allocation_new): new(T) 为类型 T 分配内存地址，设置为 T 的零值，然后返回 *T

### new

上面提到，new 函数分配地址，然后为该地址复制为零值，返回一个 *T。 golang 里面不同类型的零值是不同的，对于值类型，每个类型都有对应的零值；对于引用类型，如切片，channel等，零值均为nil。接下来我们验证一下 new 函数的功能。

