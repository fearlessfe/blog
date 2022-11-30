---
title: "Implementing a Regular Expression Engine with go"
date: 2022-09-25T20:10:52+08:00
draft: true
---

```javascript
// 创建一个 NFA state
function createState(isEnd) {
  return {
    isEnd,
    transition: {},
    epsilonTransitions: []
  }
}
```

有两种类型的转移， symbol 和 epsilon.
一个 state 最多只有一个 symbol 或 两个 epsilon 的转移，但不可能同时有 symbol 和 epsilon

* symbol: 接收一个字符使得状态转移
* epsilon: 空字符串使得状态转移

```javascript
// 在 state 中添加 epsilon transition
function addEpsilonTransition(from, to) {
  from.epsilonTransitions.push(to)
}

function addTransition(from, to, symbol) {
  from.transition[symbol] = to;
}
```

两种类型的 NFA

```javascript
function fromEpsilon() {
  const start = createState(false);
  const end = createState(true);
  addEpsilonTransition(start, end);

  return { start, end };
}

function fromSymbol(symbol) {
  const start = createState(false);
  const end = createState(true);
  addTrannsition(state, end, symbol);

  return { start, end };
}

```

将两种简单的状态机组合成复杂的状态机

```javascript
function concat(first, second) {
  addEpsilonTransition(first.end, second.start)
  first.end.isEnd = false
  return {start: first.start, end: second.end}
}

function union(first, second) {
  const start = createState(false);
  addEpsilonTransition(start, first.start)
  addEpsilonTransition(start, second.start)

  const end = createState(true);
  addEpsilonTransition(first.end, end);
  first.end.isEnd = false;
  addEpsilonTransition(second.end, end);
  second.end.isEnd = false;

  return { start, end };
}

function closure(nfa) {
  const start = createState(false);
  const end = createState(true);

  addEpsilonTransition(start, end);
  addEpsilonTransition(start, nfa.start);

  addEpsilonTransition(nfa.end, end);
  addEpsilonTransition(nfa.end, nfa.start);
  nfa.end.isEnd = false;

  return { start, end };
}
```