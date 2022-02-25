---
title: "Golang Mutex 源码解析"
date: 2022-01-22T13:12:08+08:00
draft: false
---

### Mutex 架构演进方向

golang 中的 Mutex 是互斥锁，它的实现不是一成不变的，主要经历了以下的变化

1. 初版：使用 flag 标记字段是否有锁
2. 给新人机会：新的 goroutine 也有机会竞争锁
3. 多给些机会：新来的和被唤醒的有更多机会竞争
4. 解决饥饿：解决竞争问题，不让 goroutine 一直等待

### 初版互斥锁

初版互斥锁使用 CAS 指令实现，以下代码为 Mutex 的初版实现。

> CAS 指令将给定的值和一个内存地址中的值进行比较，如果是同一个值，就用新的值替换内存中的值。

```'golang'
// CAS操作
func cas(val *int32, old, new int32) bool
func semacquire(*int32)
func semrelease(*int32)
// 互斥锁结构
type Mutex struct {
  key int32  // 锁是否被持有的flag
  sema int32  // 信号量，用来阻塞/唤醒goroutine
}

func xadd(val *int32, delta int32) (new int32) {
  for {
    v := val
    if cas(val, v, v + delta) {
      return v + delta
    }
  }
  panic("unreached")
}

func (m *Mutex) Lock() {
  if xadd(&m.key, 1) == 1 {
    return
  }
  semacquire(&m.sema)  // 进入队列
}

func (m *Mutex) Unlock() {
  if xadd(&m.key, -1) == 0 {
    return
  }
  semrelease(&m.sema)  // 唤醒队列中的goroutine
}
```

初版实现通过 Mutex 的 key 属性作为标识，key 为0或1来表示锁是否被占用。

Unlock 方法可以被任意的 goroutine 调用，即使是没有持有这个锁的 goroutine。所以使用 Mutex 时，一定要遵循“谁申请，谁释放”的原则。

这种实现方式，按照 FIFO 的方式来获取锁。但是在高并发的情况下，会频繁的切换上下文，性能会有损耗。如果能够将锁交给正在占用 CPU 时间片的 goroutine 的话，会有更好的性能。

### 给新人机会

这一版尝试给新来的 goroutine 获取锁的机会。

```'golang'
type Mutex struct {
  state int32
  sema uint32
}
// 32位 state 是复合字段。最小的一位表示锁是否占有，第二位表示是否有唤醒的 goroutine，剩下的代表等待该锁的 goroutine 数量
const (
  mutexLocked = 1 << iota  // 1
  mutexWoken  // 2
  mutexWaiterShift = iota // 2
)

func (m *Mutex) Lock() {
  // state 为 0,直接获取锁
  if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
    return
  }

  awoke := false

  for {
    old := m.state
    new := old | mutexLocked  // 设置加锁的状态
    // old 为有锁的状态，等待者加1
    if old & mutexLocked != 0 {
      new = old + 1 << mutexWaiterShift
    }

    if awoke {
      // goroutine 是被唤醒
      // 将 mutexWoken 标志位变为0
      new = new &^ mutexWoken
    }

    if atomic.CompareAndSwapInt32(&m.state, old, new) {
      // 无锁状态，直接获取锁
      if old & mutexLocked == 0 {
        break
      }
      runtime.Semacquire(&m.sema)  // 休眠
      awoke = true  // 唤醒后，awoke设置为true
    }
  }
}
```

请求锁的 goroutine 分为新来的和被唤醒的，两者都会尝试获取锁。两者的区别见下表

请求锁 | 锁被持有 | 锁未被持有
---|--- |---
新来的goroutine | waiter++ 休眠 | 获取锁
被唤醒的goroutine | 清除mutexWoken，加入等待队列 | 清除mutexWoken，获取锁


```'golang'
func (m *Mutex) Unlock() {
  new := atomic.AddInt32(&m.state, -mutexLocked) // 去掉锁标志
  if (new + mutexLocked) & mutexLocked == 0 {
    // unlock of unclocked mutex
    panic("....")
  }

  old := new

  for {
    // 没有等待者 或 (唤醒状态的goroutine或有锁)
    if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken) != 0 {
      return
    }
    // 等待者减1，唤醒标志位设为1
    new = (old - 1<<mutexWaiterShift) | mutexWoken
    if atomic.CompareAndSwapInt32(&m.state, old, new) {
      runtime.Semrelease(&m.sema)
      return
    }
    old = m.state
  }
}
```

释放锁时，首先会将去掉锁的标志位。如果已经是释放的状态，则直接panic。
进入 for 循环后，
如果没有等待者，return
如果现在有唤醒的 goroutine 或者持有锁的 goroutine，直接交给其余的 goroutine 处理，return

将等待者减1，设置唤醒标志位，然后唤醒等待的 goroutine.

这一版本的实现，让新来的 goroutine 和唤醒的 goroutine 竞争，使得新来的 goroutine 可以获取锁，这样可以减少上下文的切换，提升性能。

### 多给些机会

新来的 goroutine 和唤醒的 goroutine 如果没有获取到锁，会通过自旋的方式，尝试检查锁是否被释放。经过一定次数的尝试后，再执行原来的操作。改动后的 Lock 实现如下

```'golang'
func (m *Mutex) Lock() {
  // state 为 0,直接获取锁
  if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
    return
  }
  
  awoke := false
  iter := 0

  for {
    old := m.state
    new := old | mutexLocked

    if old & mutexLocked != 0 {
      if runtime_canSpin(iter) {
        // 新的goroutine && 没有唤醒的goroutine && 有等待者 && 设置mutexWoken标志位
        if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 && atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
          awoke = true
          // 设置 mutexWoken 标志位后，Unlock 方法不会唤醒新的 goroutine，自选期间获取锁的概率变大
        }
        runtime_doSpin()
        iter++
        continue
      }
      // 自旋结束，等待列表加1
      new = old + 1<<mutexWaiterShift
    }
    if awoke {
      if new&mutexWoken == 0 {
        panic("sync: inconsistent mutex state")
      }
      new &^ = mutexWoken // 新状态清除唤醒标记
    }
        
    if atomic.CompareAndSwapInt32(&m.state, old, new) {
      if old&mutexLocked == 0 {
        break
    }
      runtime_Semacquire(&m.sema)
      awoke = true
      iter = 0
    }
  }
}

```

上面自旋时，会设置唤醒标志位，导致等待队列的goroutine可能一直处于休眠状态，这样会产生饥饿的问题。

### 解决饥饿

```'golang'
type Mutex struct {
  state int32
  sema uint32
}

const (
  mutexLocked = 1 << iota  // 1
  mutexWoken  // 2
  mutexStarving  // 4
  mutexWaiterShift = iota  // 3
    
  starvationThresholdNs = 1e6  // 1ms
)

func (m *Mutex) Lock() {
  if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
    return
  }
  // Slow path
  m.lockSlow()
}

func (m *Mutex) lockSlow() {
  var waitStartTime int64
  starving := false
  awoke := false
  iter := 0
  old := m.state
  for {
    // 锁未释放，且不处于饥饿状态。 可以自xuan
    if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
      if runtime_canSpin(iter) {
        if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 && atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
          // 新的goroutine && 没有唤醒的goroutine && 有等待 && 将当前giroutine设为唤醒状态
          awoke = true
        }
        runtime_doSpin()
        iter++
        old = m.state // 获取锁的状态，之后会检查锁是否释放
        continue
      }

      new := old
      // 非饥饿模式，加锁
      if old&mutexStarving == 0 {
        new |= mutexLocked
      }
      // 饥饿模式或者有锁
      if old&(mutexLocked|mutexStarving) != 0 {
        new += 1<<mutexWaiterShift
      }
      // 有锁，设置饥饿模式
      if starving && old&mutexLocked != 0 {
        new |= mutexStarving
      }
      if awoke {
        if new&mutexWoken == 0 {
          throw("sync: inconsistent mutex state")
        }
        new &^ = mutexWoken
      }

      if atomic.CompareAndSwapInt32(&m.state, old, new) {
        // 无锁，非饥饿模式
        if old&(mutexLocked|mutexStarving) == 0 {
          break
        }
        // 处理饥饿状态
        // 如果以前就在队列里，加入到队列头
            
        queueLifo := waitStartTime != 0
        if waitStartTime == 0 {
          waitStartTime = runtime_nanotime()
        }
        // 阻塞等待
        runtime_Semacquire(&m.sema, queueLifo, 1)
                
        starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
        old = m.state
        // 如果锁已经处于饥饿状态，直接抢到锁，返回
        if old&mutexStarving != 0 {
          if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
            throw("sync: inconsistent mutex state")
          }
                    
          // 加锁且将waiter减1
          delta := int32(mutexLocked - 1<<mutexWaiterShift)
                    
          if !starving || old>>mutexWaiterShift == 1 {
            delta -= mutexStarving // 最后一个waiter或者已经不饥饿，清除标记
          }
          atmoc.AddInt32(&m.state, delta)
          break
        }
        awoke = true
        iter = 0
      }
    } else {
      old = m.state
    }
  }
}
```