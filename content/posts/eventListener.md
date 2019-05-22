---
title: "事件监听函数"
date: 2019-05-19T10:36:08+08:00
draft: false
---

毕业后转行为一名前端开发，一共好像就面试过6次。其中就有两次面试的考过手写事件监听函数，第一次不会写，第二次写出来了，但也不太不太完美。在Vue中，每个实例上都有$on,$once,$emit,$off这四个方法，接下来就来看看Vue是怎么实现事件监听函数的。

1. vm.$on
根据[官方文档][1],该方法传入两个参数：事件名和回调函数；其中事件名可以是字符串或字符串数组。自己实现on方法时只会考虑字符串的情况，从没有考虑过数组的情况。废话不多说，接下来看看Vue是如何实现的。
```
Vue.prototype.$on = function(event, fn) {
  const vm = this;
  if(Array.isArray(event)) {
    for(let i = 0, l = event.length; i < l; i++ ) {
      this.$on(event[i], fn)
    }
  } else {
    (vm._events[event] || (vm._events[event] = [])).push(fn)
  }
  return vm
}
```
$on方法直接挂载在Vue上，方法中vm是Vue的实例，首先判断event是否为数组，如果是数组，则遍历数组，调用$on。如果不是数组，则直接添加到_events对象中。_events是vm上的一个熟悉，初始化时vm._events = {}。首先判断_events中有没有当前事件名，如果没有，则初始化；如果有，则添加到数组中。添加的过程用if语句改写会更清楚一点。
```
if(!vm._events[event]) {
  vm._events[event] = []
}
vm._events[event].push(fn)
// 但是Vue的实现更简洁，学到了
```

2. vm.$off
说完添加事件的方法，接下来看看移除事件监听器的方法。$off方法和$on的参数一样。但如果没有参数，则移除所有事件；只提供一个参数，则移除该事件下的所有监听器；如果提供两个参数，则移除对应的监听器。

```
Vue.prototype.$off = function(event, fn) {
  const vm = this;
  // 没有参数，直接清空_events对象
  // Object.create(null)也是生成一个空对象，但是没有__proto__熟悉
  if(!arguments.length) {
    vm._events = Object.create(null);
    return vm;
  }
  // 判断是否为数组
  if(Array.isArray(event)) {
    for(let i = 0, l = event.length; i < l; i++ ) {
      this.$off(event[i], fn)
    }
  }
  // cbs为空说明没有对应的事件，直接返回
  const cbs = vm._events[event]
  if(cbs) {
    return vm
  }
  // 只有一个参数时，将对应的event属性变为null
  if(arguments.length === 1) {
    vm._events[event] = null
    return vm
  }
  // 两个参数的情况
  if(fn) {
    const cbs = vm._events[event]
    let cb
    let i = cbs.length
    while(i--) {
      cb = cbs[i]
      if(cb === fn || cb.fn === fn) {
        cbs.splice(i, 1)
        break
      }
    }
  }
  return vm;
}
```
两个参数的情况时，cb.fn === fn，这个判断有点莫名。不要急，看到后面就知道这段代码的作用了。

3. vm.$once
参数和用法与$on方法一致，但是该方法监听的事件只会执行一次，下面就看看Vue的实现逻辑
```
Vue.prototype.$once = function(event, fn) {
  const vm = this;
  function on () {
    vm.$off(event, on);
    fn.apply(vm, arguments);
  }
  on.fn = fn;
  vm.$on(event, on);
  return vm;
}
```
$once方法绑定的是内部的on函数，执行on函数时，会先执行$off方法，然后执行输入的fn函数，达到只执行一次的目的。
这里将on.fn = fn，把fn当中on方法的一个属性，所以在执行$off方法时，会有cb.fn === fn的判断。

4. vm.$emit
接收多个参数，第一个是事件名，只支持字符串类型，后面的参数都会传给对应的回调函数。
```
Vue.prototype.$emit = function () {
  const vm = this;
  const _event = Array.prototype.shift(arguments)
  let cbs = vm._events[_event]
  if(cbs) {
    const args = Array.from(arguments);
    for(let i = 0; l = args.length; i < l; i++) {
      try{
        cbs[i].apply(vm,args)
      } catch (e) {
        handleError(e, vm, `event handler for ${event}`)
      }
    }
  }
  return vm
}
```
上面并不是完整的源码，在参数处理上，本人做了一点修改，arguments第一个参数为event,剩下的都是回调函数的参数，所以用shift方法处理。

一个简单的事件监听函数，框架的源码实现的很完整，各种边界情况都做了处理，值得学习。同时我还注意到，源码中遍历的实现都是用的for循环，并没有用高级一点forEach,map等方法，这是为什么呢？这个我也不是太明白，可能是为了更好的性能吧，毕竟高级的方法底层也是用的for循环。

[1]: https://cn.vuejs.org/v2/api/#vm-on