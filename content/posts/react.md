---
title: "React 笔记"
date: 2022-05-22T22:09:49+08:00
draft: false
---

## 类组件与函数组件

函数组件能更好的复用逻辑

高阶组件缺点：组件静态方法，ref 传递

OOP思想与函数式思想，组合优于继承

## setState 是否为异步

在 React 生命周期和合成事件中，是批量更新，表现为异步。在 addEventListener, setTimeout, setInterval 中，不受 React 控制，是同步的

为什么设置为异步更新？保持内部一致性，props 无法同步；为后面的 React 异步设计打下基础。

## React 跨层级通信

确定主题，列举场景

* 父子: props
* 子父: 回调函数
* 兄弟: 公共父组件
* 无关系: Context，状态管理工具

## 状态管理

FLUX 单项数据流

Redux 如何处理副作用：Dispatch 中间件处理副作用 或者 在 Reducer 中处理副作用

分形架构：如果子组件能够以同样的结果，作为一个应用使用

Redux：单一数据源，reducer纯函数， store 只读

Mobx：监听方式，Mobx5 前，采用 Object.defineProperty, 5 之后采用 Proxy 方案

## Virtual DOM 工作原理

XHP 防止 XSS，但是状态更新需要重新渲染全部页面，影响用户体验。

React.createElement 返回的是一个 js 的对象。

diff 函数，计算状态变更前后虚拟 DOM 树的差异
渲染函数，渲染整个虚拟 DOM 树或者处理差异点
所以 React 和 ReactDOM 是两个库

Virtual DOM 的优势

大量直接操作 DOM 容易引起网页性能下降，这样的情况下，React 基于虚拟DOM的diff处理与皮处理操作可降低 DOM 的操作范围和频次，提升页面性能

防止 XSS

跨平台成本低

Virtual DOM 的劣势

内存，高性能场景难以优化

## React Diff 算法与其他框架的不同

* 更新时机：setState, forceUpdate, hooks调用等
* 遍历算法：深度优先遍历，保证组件生命周期的一致性
* 优化策略：树的遍历，复杂度为O(n^3)。React 使用分治的思想，将复杂度将为O(n)

从 树，组件，元素三个方向来优化

1. 忽略节点跨层级操作场景，对树进行分层比较，两棵树只对同一层次节点进行比较
2. 组件的 Class 是同一类型，则进行树比对
3. 同一层级的节点，通过 key 的方式直接移动

Fiber 机制下节点与树分别采用 FiberNode 和 FiberTree 进行重构

current 树与 workInProgress 两株树双缓冲，直接更新 current 树的指针

Vue2.0 使用 snabbdom,整体思路与 React 相同。

## React 渲染流程

* 核心
* 阶段
* 调度
* 事务

协调，Reconciler

Stack Reconciler 是 React 15 及以前版本的渲染方案，核心是以递归的方式逐级调度栈中子节点到父节点的渲染

Fiber Reconciler 是 React 16 及以后版本的渲染方案，核心设计是增量渲染，将渲染工作分隔为多区块，将其分散到多个帧中执行

React 渲染的整体策略是递归，并通过事务建立 React 与虚拟 DOM 的联系并完成调度

渲染过程大概一致，但协调不同

## React 渲染异常

是什么？怎么解决？

预防： ?. 操作符
兜底： 高阶组件，hooks

## 如何分析和调优性能瓶颈

性能指标，统计数据

## 避免重复渲染

## React Hooks 使用限制

问题
组件难以复用状态逻辑
类难以编译优化
类组件逻辑杂乱

限制
不要嵌套调用hook
只能在函数式组件中调用hook

## useEffect 与 useLayoutEffect 

共同点

都是用于处理副作用，如DOM事件

不同点

直接操作 DOM 样式，或者引起页面闪烁，使用 useLayoutEffect

useLayoutEffect：会在所有 DOM 变更之后同步调用，所以可以读取 DOM 布局并同步触发重渲染，但计算量较大时，会造成卡顿

## React-Router 实现原理及工作方式

实现原理

hash 路由
history

## React 工具库

初始化： create-react-app, umi
路由： react-router
css: css in js: styled-component, classnames
基础组件： antd
状态管理：redux， mobx
构建： webpack， rollup， esbuild
检查： ESLint， prettier
测试： jest, react-testing-library, react-hooks-testing-library
