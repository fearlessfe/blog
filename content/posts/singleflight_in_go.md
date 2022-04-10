---
title: "SingleFlight in go"
date: 2022-03-13T18:25:54+08:00
draft: true
---

### singleFlight 作用

在缓存的场景中，对于缓存中不存在的 key，需要去后面的服务去请求，如果同时有大量相同的 key 的请求，会对后面的服务造成很大的压力。SingleFlight 的作用是将同一个 key 的请求合并成一个，来减轻对下层服务的影响。

### singleFlight 方法

golang singleFlight 暴露了一下三个方法

```golang
type Group
  func(g *Group)Do(key string, fn func()(interface{}, error)) (v interface{}, error, shared bool)
  func(g *Group)DoChan(key string, fn func()(interface{}, error)) <-chan Result
  func(g *Group)Forget(key string)
  ```

  Do：这个方法执行一个函数，并返回函数的结果。对于相同的 key，同一时间只有一个函数在执行。shared 指示 v 是否返回给多个请求。

  DoChan：与 Do 方法类似，只不过返回的是一个 channel。

  Forget：告诉 Group 忘记这个 key，这样后面的请求会执行 fn，而不是等待前一个未完成 fn 的结果.

### singleFlight 源码

Group 代表 SingleFlight，call 是一个辅助对象，代表正在执行的 fn 或者已经完成的请求。Result 是对 Do 方法返回值的一个封装，传到 channel 中。

```golang
type Group struct {
  mu sync.Mutex
  m map[string]*call
}

type call struct {
  wg sync.WaitGroup
  // WaitGroup 执行结束时会赋值，以后就是只读。
  val interface{}
  err error
  // 处理时是否忘记 key
  forgotten bool
  dups int
  chans []chan<- Result
}

type Result struct {
	Val    interface{}
	Err    error
	Shared bool
}
```

下面查看 Do 方法源码

```golang
func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool) {
  g.mu.Lock()
  // lazily initialized
  if g.m == nil {
		g.m = make(map[string]*call)
	}
  // 如果 key 存在，则等待第一个返回的结果
  if c, ok := g.m[key]; ok{
    c.dups++
    g.mu.Unlock()
    c.wg.Wait()
  }
  // 第一次请求，创建一个 call
  c := new(call)
  c.wg.Add(1)
  g.m[key] = c
  g.mu.Unlock()

  g.doCall(c, key, fn)
  return c.val, c.err, c.dups > 0
}
```
