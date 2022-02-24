---
title: "The Node.js Event Loop, Timers and process.nextTick()"
date: 2022-02-23T20:59:19+08:00
draft: false
---

### 为什么要写这篇文章

有一段时间 `nodejs` 工作经历，面试总是被问到 `Event Loop`,虽然看过官网上的文章，奈何总是记不住，面试也答不上来。因此，写这篇文章来加深印象。原文请👇[这里](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)

### 什么是 `Event Loop`

虽然 `Javascript` 是单线程的，但是 `Event Loop` 让 `Nodejs` 能够执行非阻塞的 I/O。

现代内核都是多线程的，它们可在后台执行多个操作。但其中一个操作完成后，内核通知 Nodejs 以便将相应的回调添加到轮询队列，最后回调会被执行。

### `Event Loop`简介

当 Nodejs 运行时，它会初始化事件循环，执行输入的脚本，脚本中可能会调用异步的 API，timer 函数或 `process.nextTick()`,然后开始事件循环。

下图（来源于[原文](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)）简单的描述了事件循环操作顺序。

   ┌───────────────────────────┐
┌─>│           timers          │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │
│  └─────────────┬─────────────┘      ┌───────────────┐
│  ┌─────────────┴─────────────┐      │   incoming:   │
│  │           poll            │<─────┤  connections, │
│  └─────────────┬─────────────┘      │   data, etc.  │
│  ┌─────────────┴─────────────┐      └───────────────┘
│  │           check           │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │
   └───────────────────────────┘
> 图中每个盒子代表事件循环中一个`阶段`

每个阶段都是一个队列，按照 FIFO 的方式执行回调函数。事件循环进入其中一个阶段后，首先会执行该阶段特定的操作。然后执行该阶段队列中的回调，知道队列中回调全部执行完或者达到最大的回调限制，然后事件循环进入下一个阶段。

由于这些操作可能会产生新的事件，而轮询阶段（poll phase）新的事件由内核加入到队列，因此处理轮询事件时也会有新的事件加入到队列中。因此，长时间运行的回调会让轮询阶段的时间远超过计时器的阈值。

### 阶段概览

* timers: 这个阶段执行 `setTimeout()` 和 `setInterval()` 的回调
* pending callbacks：执行推迟到下一个循环的 I/O 回调
* idle,prepare: 仅供内部使用
* poll（轮询）: 检索新的 I/O 事件；执行 I/O 相关的回调（几乎所有的回调，除了关闭回调，定时器调度的回调和`setImmediate()`）,节点会在适当的时候阻塞。
* check：`setImmediate()`的回调在这里执行。
* close callbacks: 关闭的回调，比如`socket.on('close',...)`

在事件循环之间，Nodejs 会检查它是否正在等待异步 I/O 或计时器。如果没有，则退出。（这里退出应该是指退出 nodejs 程序）。

### 阶段详情

#### timers

计时器可以指定一个阈值，到达阈值后执行回调，而不是决定回调执行的确切时间。计时器的回调会在指定的时间过后尽可能早的执行，但是回调的执行可能会延迟。

> 轮询阶段控制计时器回调的执行

举个例子，定时器在100ms后执行回调，然后一个异步读取文件会花 95ms

```nodejs
const fs = require('fs');

function someAsyncOperation(callback) {
  fs.readFile('/path/to/file', callback)
}

const timeoutScheduled = Date.now();

setTimeout(() => {
  const delay = Date.now() - timeoutScheduled
  
  console.log(`${delay}ms have passed since I was scheduled`)
}, 100)

// someAsyncOperation takes 95ms
someAsyncOperation(() => {
  const startCallback = Date.now()

  // do something will take 10ms
  while(Date.now() - startCallback < 10) {
    // do nothing
  }
})
```

当事件循环进入轮询阶段，此时队列是空的，所以会等待一直达到最近的计时器阈值。
95ms 后，读取文件的操作完成，回调进入轮询阶段的队列，然后花 10ms 执行回调。执行完成后，队列为空，这时候到达了 100ms 的阈值，因此会回到 timer 阶段执行定时器的回调。所以定时器的回调会在 105ms 后被执行，而不是 100ms。

> 为了防止轮询阶段过长使得事件循环陷入饥饿，libuv 对轮询阶段设置处理事件的最大值

#### pending callbacks

这个阶段执行系统操作的回调，比如 TCP error。比如一个 TCP 连接收到了 `ECONNREFUSED`,一些 *nix 的系统想要 report 这个错误。这些事件的回调会在这个阶段执行。

#### idle prepare

这个阶段仅供内部使用

#### poll(轮询)

轮询阶段主要有两个功能：

1. 计算应该阻塞和轮询 I/O 的时间
2. 处理轮询队列的事件

当事件循环进入 轮询阶段并且没有定时器，将发生以下两件事之一：

* 如果轮询队列不是空的，那事件循环会按顺序同步的执行队列中的事件，知道队列为空或者达到最大的事件限制
* 如果队列为空，则可能发生下列两件事之一：
  * 如果有 `setImmediate()` 事件，事件循环会结束轮询阶段，进入 check 阶段，执行该阶段的回调。
  * 如果没有 `setImmediate()` 事件，事件循环会等待回调被加入到队列，然后立即执行回调

如果轮询队列为空，事件循环会检查是否达到定时器的阈值。如果有，事件循环会回到 timer 阶段，执行定时器的回调。

#### check

这个阶段是在轮询阶段结束后执行该阶段的回调。

`setImmediate()`是一个特别的定时器，在单独的阶段执行。

一般来说，代码执行后，事件循环会最终达到轮询阶段，等待新来的链接，请求等。如果有 `setImmediate()` 而且轮询阶段处于空闲状态，轮询阶段会结束，然后进入 check 阶段执行 `setImmediate()` 的回调。

#### close callback

如果套接字或句柄被突然关闭（例如 socket.destroy()）,此阶段会 emit 'close' 事件。否则会通过 process.nextTick()


### setImmediate() VS setTimeout()

setImmediate() 和 setTimeout() 比较类似，但是根据调用时间不同而有不同的表现。

* setImmediate() 设计为 轮询阶段结束后调用
* setTimeout() 达到设定的阈值后调用

如果两者都在主模块中调用，那么两者的顺序是不固定的，两者顺序取决于处理器的性能。

```nodejs
setTimeout(() => {
  console.log('timeout)
}, 0)

setImmediate(() => {
  console.log('immediate)
}, 0)
```

如果在 I/O 事件中调用，则 setImmediate 总是先执行

```nodejs
const fs = require('fs');

fs.readFile("", () => {
  setTimeout(() => {
    console.log('timeout)
  }, 0)

  setImmediate(() => {
    console.log('immediate)
  }, 0)
})

```

### process.nextTick()

#### 理解 nextTick

process.nextTick() 没有在图中展示，即使这是一个异步的 API。因为 nextTick 不属于事件循环。

在某个阶段的任意时刻调用 process.nextTick()，它的回调会在事件循环继续之前执行。这会出现一些坏的情况，比如在 process.nextTick() 递归调用 process.nextTick()，这会使得事件循环无法继续。

#### 为什么需要 nextTick

nextTick 一部分的设计理念是 API 应该始终是异步的，即使它不是。以下面的代码为例

```nodejs
function apiCall(arg, callback) {
  if (typeof arg != 'string') {
    return process.nextTick(callback, new TypeError('args should be string'))
  }
}
```

这样回调会在所有用户的 code 执行完毕后，在进入事件循环之前才执行。

下面是一个真正的例子

```nodejs
const server = net.createServer(() => {}).listen(8080)
server.on('listening', () => {})
```

只有绑定端口后， 'listening' 回调才会执行。但问题是那时候 'listening' 回调还没设置。
所以 'listening' 回调放在 nextTick() 中。

### process.nextTick() VS setImmediate()

这两者比较类似，但是名字有点让人困惑

* process.nextTick() 在当前阶段执行
* setImmediate() 在时间循环中执行

> 建议开发者在所有情况下都使用 setImmediate()

### 为什么需要 process.nextTick()

主要有以下两个原因

1. 允许用户处理错误，清理不需要的资源或在事件循环之前再次尝试请求
2. 有时，需要允许回调在调用栈解除后但在事件循环之前调用

一个符合用户期望的例子如下

```nodejs
const server = net.createServer();
server.on('connection', (conn) => {})

server.listen(8000)
server.on('listening', () => {})
```

listen() 在时间循环开始时允许，如果 'listening' 函数放在 setImmediate 中，那么有一定概率在轮询阶段收到新的请求的时候，'listening' 回调还没有注册。

另一个例子是在构造函数中 emit 事件

```nodejs
const EventEmitter =  require('events')
const util = require('util')

function MyEmitter() {
  EventEmitter.call(this)
  this.emit('event)
}

util.inherits(MyEmitter, EventEmitter)

const myEmitter = new MyEmitter();
myEmitter.on('event', () => {
  console.log('event occurred!')
})
```

上诉代码中的事件不会触发，因为脚本不会处理到为事件分配回调的地方。如果将 this.emit('event) 放入 process.nextTick() 中执行，则会触发 'event' 事件。