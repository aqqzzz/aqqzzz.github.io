---
title: react-redux 源码解读一
date: 2019-08-25 21:48:13
tags:
- React
- react-redux
categories:
- React
---

要分几篇来写，今天先写一下比较简单的部分

对于 react-redux，只针对 Provider 和 connect 的代码以及具体逻辑进行解读，今天先把边角的代码解读一下，明天了解 核心的 connectAdvanced 是怎么实现的

这次不从入口文件开始看了，因为入口文件中其实export了很多其他的函数，但是react-redux中我们常用的也就 Provider 和 connect 两个，那么就直接从这两个函数入口开始看

### Provider

用法：

```js
<Provider store={store}>
  <App />
</Provider>
```

其实猜测也能知道，这里的重点就是把 redux 的store 作为context 传递给 应用，那么再应用内部就可以随拿随用了

看一下具体实现

Provider 中render的具体代码为

```javascript
render() {
    const Context = this.props.context || ReactReduxContext

    return (
      <Context.Provider value={this.state}>
        {this.props.children}
      </Context.Provider>
    )
  }
```

其 state 具体值为

```javascript
this.state = {
      store, // redux store
      subscription // 负责 观察者相关的方法
}
```

### Subscription

subscription 是 react redux 中专门抽出来处理 观察者逻辑的相关代码，在Provider 和 connect时都会使用到，

其中首先定义了一个 createListenerCollection，用来处理观察者相关的任务

```javascript
function createListenerCollection() {
 ...
  return {
    // 清空观察者
    clear() {
      ...
    },

    // 通知观察者
    notify() {
      ...
    },

    // 获取观察者列表
    get() {
     ...
    },

    // 注册观察者
    subscribe(listener) {
      ...
      // 返回一个解注册当前观察者的方法
      return function unsubscribe() {
        ...
      }
    }
  }
}
```

再声明一个 Subscription 类，它接受 store 和 parent Subscription 作为参数，处理 跟 Provider、connect以及 redux store 相关的观察者注册、notify以及解注册逻辑

```javascript
export default class Subscription {
  constructor(store, parentSub) {
    this.store = store // redux store
    this.parentSub = parentSub // 父 subscription 对象
    this.unsubscribe = null // 解注册的方法
    this.listeners = nullListeners // 观察者集合，非数组，而是几个函数的集合（就是createListenersCollection 返回的函数集合）

    this.handleChangeWrapper = this.handleChangeWrapper.bind(this)
  }

  // 添加一个观察者
  addNestedSub(listener) {
    this.trySubscribe() // 添加第一个观察者的时候生成观察者列表
    return this.listeners.subscribe(listener) // 向观察者列表中添加观察者
  }

  // 通知观察者
  notifyNestedSubs() {
    this.listeners.notify()
  }

  // onStateChange：通过 instance.onStateChange 定义，
  // 在 Provider 中，onStateChange 指向的调用方法为 subscription.notifyNestedSubs
  // TODO 为什么要把这个逻辑包在Provider中，反正肯定是调用Subscription自身的方法
  handleChangeWrapper() {
    if (this.onStateChange) {
      this.onStateChange()
    }
  }

  // 是否已被注册过了
  isSubscribed() {
    return Boolean(this.unsubscribe)
  }

  trySubscribe() {
    // 初次注册，如果存在parentSub，则将 handleChangeWrapper 注册到 parentSub 中去，否则注册到当前组件对应的store中去
    if (!this.unsubscribe) {
      this.unsubscribe = this.parentSub
        ? this.parentSub.addNestedSub(this.handleChangeWrapper)
        : this.store.subscribe(this.handleChangeWrapper)

      // 生成观察者集合
      this.listeners = createListenerCollection()
    }
  }

  // 解注册
  tryUnsubscribe() {
    if (this.unsubscribe) {
      this.unsubscribe()
      this.unsubscribe = null
      this.listeners.clear()
      this.listeners = nullListeners
    }
  }
}

```

TODO 代码了解还不够，之后在看完 connectAdvanced 之后再回来看这里是不是有新的发现

很多地方都看不懂我擦

决定明天看完整个代码之后再写读源码文档

### SelectorFactory

