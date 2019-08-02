---
title: lodash源码防抖
date: 2019-08-02 20:56:00
tags:
- lodash
- javascript
categories:
- javascript
---

什么是防抖？

在前端开发中，我们常常会遇到一些频繁的事件触发，比如：

- window 的 resize、scroll
- mousedown、mouseover，mousemove
- keyup、keydown

这些事件触发是很频繁的，

```javascript
var count = 1;
var container = document.getElementById('container');

function getUserAction() {
    container.innerHTML = count++;
};

container.onmousemove = getUserAction;
```

如上代码，响应用户鼠标移动的操作，鼠标从界面左边移动到右边，

![](https://github.com/mqyqingfeng/Blog/raw/master/Images/debounce/debounce.gif)

一次移动就触发了这么多次操作，这个js代码只是简单地修改html内容，如果我们鼠标移动的相应函数要执行很复杂的DOM操作，或者要跟后端请求一些数据的话，假设 1 秒触发了 60 次，每个回调就必须在 1000 / 60 = 16.67ms 内完成，否则就会有卡顿出现。

解决这个问题有两种方式：

1. debounce——防抖
2. throttle——节流

## 防抖

防抖的原理就是，我一定要等待一个固定的时间段之后再执行回调函数，如果在我等待的这段时间里又一次被调用到了，那么我的计时器会清零，从此刻开始再等一个固定时间段之后再执行回调函数

自己实现版本

```javascript
function debounce(fn, timeout) {
  let timer = null;
  return function(...args) {
    clearTimeout(timer);
    timer = setTimeout(() => {
      fn.apply(this, args); // 绑定回调函数的this，并将想要传给回调函数的参数也一起传出来
		}, timeout)
  }
}
```

lodash 的 debounce 实现

```javascript
function debounce(func, wait, options) {
    let lastArgs,
        lastThis,
        maxWait,
        result,
        timerId,
        lastCallTime

    // 参数初始化
    let lastInvokeTime = 0 // func 上一次执行的时间
    let leading = false
    let maxing = false
    let trailing = true

    // 基本的类型判断和处理
    if (typeof func != 'function') {
        throw new TypeError('Expected a function')
    }
    wait = +wait || 0
    if (isObject(options)) {
        // 对配置的一些初始化
    }

    function invokeFunc(time) {
        const args = lastArgs
        const thisArg = lastThis

        lastArgs = lastThis = undefined
        lastInvokeTime = time
        result = func.apply(thisArg, args)
        return result
    }

    function leadingEdge(time) {
        // Reset any `maxWait` timer.
        lastInvokeTime = time
        // 为 trailing edge 触发函数调用设定定时器
        timerId = setTimeout(timerExpired, wait)
        // leading = true 执行函数
        return leading ? invokeFunc(time) : result
    }

   function remainingWait(time) {
        const timeSinceLastCall = time - lastCallTime // 距离上次debounced函数被调用的时间
        const timeSinceLastInvoke = time - lastInvokeTime // 距离上次函数被执行的时间
        const timeWaiting = wait - timeSinceLastCall // 用 wait 减去 timeSinceLastCall 计算出下一次trailing的位置

        // 两种情况
        // 有maxing:比较出下一次maxing和下一次trailing的最小值，作为下一次函数要执行的时间
        // 无maxing：在下一次trailing时执行 timerExpired
        return maxing
            ? Math.min(timeWaiting, maxWait - timeSinceLastInvoke)
            : timeWaiting
    }

    // 根据时间判断 func 能否被执行
    function shouldInvoke(time) {
        const timeSinceLastCall = time - lastCallTime
        const timeSinceLastInvoke = time - lastInvokeTime

        // 几种满足条件的情况
        return (lastCallTime === undefined //首次
            || (timeSinceLastCall >= wait) // 距离上次被调用已经超过 wait
            || (timeSinceLastCall < 0) //系统时间倒退
            || (maxing && timeSinceLastInvoke >= maxWait)) //超过最大等待时间
    }

    function timerExpired() {
        const time = Date.now()
        // 在 trailing edge 且时间符合条件时，调用 trailingEdge函数，否则重启定时器
        if (shouldInvoke(time)) {
            return trailingEdge(time)
        }
        // 重启定时器，保证下一次时延的末尾触发
        timerId = setTimeout(timerExpired, remainingWait(time))
    }

    function trailingEdge(time) {
        timerId = undefined

        // 有lastArgs才执行，意味着只有 func 已经被 debounced 过一次以后才会在 trailing edge 执行
        if (trailing && lastArgs) {
            return invokeFunc(time)
        }
        // 每次 trailingEdge 都会清除 lastArgs 和 lastThis，目的是避免最后一次函数被执行了两次
        // 举个例子：最后一次函数执行的时候，可能恰巧是前一次的 trailing edge，函数被调用，而这个函数又需要在自己时延的 trailing edge 触发，导致触发多次
        lastArgs = lastThis = undefined
        return result
    }

    function cancel() {}

    function flush() {}

    function pending() {}

    function debounced(...args) {
        const time = Date.now()
        const isInvoking = shouldInvoke(time) //是否满足时间条件

        lastArgs = args
        lastThis = this
        lastCallTime = time  //函数被调用的时间

        if (isInvoking) {
            if (timerId === undefined) { // 无timerId的情况有两种：1.首次调用 2.trailingEdge执行过函数
                return leadingEdge(lastCallTime)
            }
            if (maxing) {
                // Handle invocations in a tight loop.
                timerId = setTimeout(timerExpired, wait)
                return invokeFunc(lastCallTime)
            }
        }
        // 负责一种case：trailing 为 true 的情况下，在前一个 wait 的 trailingEdge 已经执行了函数；
        // 而这次函数被调用时 shouldInvoke 不满足条件，因此要设置定时器，在本次的 trailingEdge 保证函数被执行
        if (timerId === undefined) {
            timerId = setTimeout(timerExpired, wait)
        }
        return result
    }
    debounced.cancel = cancel
    debounced.flush = flush
    debounced.pending = pending
    return debounced
}

作者：zhe.zhang
链接：https://juejin.im/post/5a142de15188251c11404085
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```

这里我用文字来简单描述一下流程：

首次进入函数时因为 lastCallTime === undefined 并且 timerId === undefined，所以会执行 leadingEdge，如果此时 leading 为 true 的话，就会执行 func。同时，这里会设置一个定时器，在等待 wait(s) 后会执行 timerExpired，timerExpired 的主要作用就是触发 trailing。

如果在还未到 wait 的时候就再次调用了函数的话,会更新 lastCallTime，并且因为此时 isInvoking 不满足条件，所以这次什么也不会执行。

时间到达 wait 时，就会执行我们一开始设定的定时器timerExpired，此时因为time-lastCallTime < wait，所以不会执行 trailingEdge。

这时又会新增一个定时器，下一次执行的时间是 remainingWait，这里会根据是否有 maxwait 来作区分：

- 如果没有 maxwait，定时器的时间是 wait - timeSinceLastCall，保证下一次 trailing 的执行。
- 如果有 maxing，会比较出下一次 maxing 和下一次 trailing 的最小值，作为下一次函数要执行的时间。

最后，如果不再有函数调用，就会在定时器结束时执行 trailingEdge。

