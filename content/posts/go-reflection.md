---
title: "Go Reflection"
date: 2022-08-25T17:39:10+08:00
draft: true
---

### 简介

计算机中的反射是程序检查自身结构的能力，特别是类型。这是一种元编程的能力，
也会引起很多麻烦

### 类型和接口

因为反射是建立在类型系统上，所以先复习一下 go 中的类型

Go 是静态类型，所有的变量都有一个类型，在编译阶段就会确定类型

一个重要的类型类别就是接口，它代表一系列方法的集合。接口类型的变量可以存储
任何实现接口方法的具体类型的值

interface{} 是一个特殊的类型，可以是任意的类型

有人说 Go 中的接口是动态类型，但这是错误的。接口也是静态类型：一个接口类型的变量总是有
同样的静态类型。即使在运行时，接口变量的值发生变化，但新的值也满足接口的类型。

我们需要对上述内容很了解，因为反射和接口紧密相关。

### 接口的表现形式

一个接口类型的变量存储一对值：变量的具体值和值的类型描述。更准确的说，值是实现底层接口的具体数据项，类型描述了数据项的完整类型。以 io.Reader 为例

```golang
var i io.Reader
tty, err := os.OpenFile("/dev/tty", os.O_RDWR, 0)
if err != nil {
  return nil, err
}
r = tty
```

上诉代码中，r 为接口类型，存储 (value, type)对，真实值为 (tty, *os.File)。其中 *os.File 实现了 Reader 接口。接口的静态类型决定了接口类型的变量可以调用哪些方法，即使具体的值有更多的方法。


### 反射定律1：反射只能由接口值转为反射体

反射是一种机制，用来检查接口变量的类型和值。在 go `reflect` 包中的 `Type`和 `Value` 代表接口中的类型和值。其中 `TypeOf` 和 `ValueOf` 这两个方法获取接口的`Type`和 `Value`

`Value` 类型通过 `Type` 方法可以获取到 `Type`，两者都有 `Kind` 方法获取具体的类型，该方法获取的下层的类型，比如 `type MyInt int` ，`Kind`方法返回的是 `int`.

### 反射定律2:反射由反射体转为接口值

调用 `Value` 类型的 `Interface` 方法将反射转为接口，然后通过类型断言转换为具体的类型。

### 反射定律3:可修改的值才能被反射改变

这条定律时最微妙且令人困惑的，如果我们从第一条定律出发，就会很容易理解。下面的代码会panic，但是值得学习

```golang
var x float64 = 3.4
v := reflect.ValueOf(x)
v.SetFloat(7.1) // Error

// panic: reflect.Value.SetFloat using unaddressable value
```

报错的根本原因是该值是不可设置的。`Value`的`CanSet`方法可以知道是否可以赋值。

x 不能被更改的原因和函数传参的方式有关系。 reflect.ValueOf(x) 实际上获取的是x的副本，更改副本的值对原值没有影响。如果我们传指针，情况会怎么样？

```golang
var x float64 = 3.4
p := reflect.ValueOf(&x) // Note: take the address of x.
fmt.Println("type of p:", p.Type())
fmt.Println("settability of p:", p.CanSet())

// type of p: *float64
// settability of p: false
```

虽然传的是指针，但是 p 还是不可赋值的

为什么传指针也不能赋值？原因还是一样，传过来的是指针的副本，不能对指针做修改。但是指针存的地址指向真实的值，所以通过更改指针指向的值来达到修改值的目的。`Value`的`Elem()`方法可以获取指针指向的元素，更改指向元素的值来达到修改值的目的。

```golang
v := p.Elem()
fmt.Println("settability of v:", v.CanSet())
// settability of v: true
```