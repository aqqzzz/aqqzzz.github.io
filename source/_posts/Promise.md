---
title: Promise
date: 2019-08-07 00:03:27
tags:
- Promise
- es6
- javascript
categories:
- javascript
---

关注点：

- Promise 的重点在于 状态转换（pending，fulfilled，rejected）
- then的回调存储在 fulfillQueue 和 rejectQueue，只有在状态由pending 转换为完成态的时候才会按照顺序调用 queue 中的回调
- 每次 then 都是返回一个全新的Promise对象
- then(onFulfilled, onRejected)：onFulfilled的几种情况
  - onFullfilled 不是函数：上一个then的处理结果透传到下一个then去
  - onFullfilled 是函数：
    - 返回值为Promise对象：返回这个 Promise.then 的结果（其实是一个递归的过程，每次then都会判断当前参数是否为Promise对象，如果是Promise的话会继续执行 then
    - 返回值为普通对象：将返回值作为参数执行下一个onFullfill

https://juejin.im/post/5b83cb5ae51d4538cc3ec354