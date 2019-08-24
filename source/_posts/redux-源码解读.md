---

title: redux 源码解读
date: 2019-08-24 17:24:23
tags:
- React
- Redux
categories:
- React
---

这周也不知道在忙什么，愧疚

leader说之前有个面试的同学每天写一篇总结，结果啥都问不倒，那我也努力一下，先来每两天一篇，努力尝试自己产出。

今天来读一下 Redux 的源码，其实之前也读过，算是复习

### Redux 简介

![](http://ww1.sinaimg.cn/large/8ac7964fly1g6b08rfm9yj21aq0jqdli.jpg)

目的：使得状态的变化变得可预测

三个特性：

1. 单一数据源：**整个应用的 state 被储存在一棵 object tree 中，并且这个 object tree 只存在于唯一一个 store 中**
2. State 是只读的：只能通过 派发 action 对象来触发 state的更新
3. 使用纯函数来进行修改：描述 action 如何改变 state 的纯函数

核心实现原理：

1. 观察者模式：

   通过 store.subscribe 注册观察者，并在dispatch 之后逐一调用注册的观察者方法

2. combineReducer：将多个单独的数据源合并为一个，并返回一个接受一个 state 和 action 作为参数的函数

   之后作为总的reducer函数，在 createStore 时作为参数被传入

3. applyMiddleware：核心代码

   ```javascript
   const middlewareAPI = {
     getState: store.getState,
     dispatch: (...args) => dispatch(...args),
   }
   const chains = middlewares.map((middleware) => middleware(middlewareAPI));
   dispatch = compose(...chains)(store.dispatch);
   ```

   增强dispatch，当最终增强之后的dispatch收到 action 时，真正执行dispatch函数，也就是action的派发

目录结构：

![](http://ww1.sinaimg.cn/large/8ac7964fly1g6b0sl1ywqj20as0digmm.jpg)

Utils 里为一些工具文件

- actionTypes：定义了 INIT、REPLACE 等redux内置的action类型，由 redux 自行调用
- isPlainObject：判断当前元素是否是一个纯对象（Object）
- Warning：在 console 提示错误的地方

### 入口文件——index.js

![](http://ww1.sinaimg.cn/large/8ac7964fly1g6b16bdue1j20ng08mgn1.jpg)

就是export了一些 redux 可以用的东西（虽然 compose 真的从来没有用过）

ok，那么我们就从这几个文件开始看代码

### createStore——创建应用的 store 对象

应用的 store 其实并不是指 整个应用的 state树，而是一个包含了很多方法的对象

createStore 的返回值就是这个对象

```javascript
return {
    dispatch, // 派发action & 实际调用 注册的listeners的地方
    subscribe, // 注册观察者
    getState, // 获取当前应用state树
    replaceReducer, // 替换 createStore时传入的 根reducer 方法
    [$$observable]: observable // [$$observable] 并没有看懂
  }
```

没有明显的调用顺序

#### subscribe

代码并不复杂，功能也不复杂，只是为了注册一个观察者，但是这里有一个值得注意的点是，因为在dispatch过程中也是有可能会注册到观察者的，但是却无法保证这个观察者是否能在本次 dispatch 过程中 被调用，所以 redux 在注册时做了一个处理。

```javascript
// listenrs 浅复制，是一个防止 在某次dispatch过程中添加新的 观察者 的方法
// 只有在 listener副本 和 真实listener 指向同一个内存地址的数组时才会执行copy方法
function ensureCanMutateNextListeners() {
  if (nextListeners === currentListeners) {
    nextListeners = currentListeners.slice()
  }
}
```

Store 内的listeners对象是一个数组，但是在其内部却维护了 `currentListeners` 和 `nextListeners` 两个listeners对象，

- currentListeners：本次 dispatch 中调用的 listners 数组
- nextListeners： 如果在 dispatch 调用listeners的过程中又注册了 观察者，那么这个观察者会注册到 nextListeners 数组中去，在下次dispatch的时候才会生效

听着有点抽象，其实就是每次注册新的观察者的时候，都会确保 currentListeners 和 nextListeners 是指向不同内存地址的两个数组，然后把 新注册的观察者注册到 nextListeners 中去，然后再dispatch结束，逐一调用 listeners 数组时，再将这个 nextListeners 指向的数组重新赋值给 currentListeners(`currentListener = nextListener`)

这么做就是为了避免，在某次 调用观察者的过程中，又注册或解绑了观察者，那么此时这个 观察者是否会被执行是不可预料的，这个机制就是为了保证 新注册或新解绑的观察者变动，在下次dispatch 的时候再体现出来

看一下 subscribe函数（返回值为解绑函数）

```javascript
 function subscribe(listener) {
    // listener必须为函数
    // 在一次subscribe过程中，是不能进行注册的
   	...

    let isSubscribed = true // 标志位，标识当前listener被注册为一个观察者了

    ensureCanMutateNextListeners()  // 将currentListeners 浅复制到 nextListeners（保存了一个观察者副本）
    nextListeners.push(listener) // 将当前 listener 注册到 观察者副本中去

    return function unsubscribe() { // return 一个取消注册的方法
      if (!isSubscribed) { // 如果当前 listener 已经不是一个观察者了（可能会有多次解绑的情况），直接return
        return
      }

      if (isDispatching) { // 如果当前正好处于一次 dispatch 的过程，是不允许解绑的
        ...
      }

      isSubscribed = false // 标志当前 listener 已经被解绑

      ensureCanMutateNextListeners() // 真正解绑之前 复制一份解绑前的 listeners 副本
      const index = nextListeners.indexOf(listener)
      nextListeners.splice(index, 1) // 在副本中删除当前 listener
    }
  }
```

#### getState——返回应用 state状态树

没什么好讲的，就是返回 currentStore，唯一需要注意的一点是，如果正好处于一次 dispatch的reducer执行 过程中，是不能调用这个方法的

#### replaceReducer——在创建应用store 之后替换 rootReducer

```javascript
function replaceReducer(nextReducer) {
    if (typeof nextReducer !== 'function') {
      throw new Error('Expected the nextReducer to be a function.')
    }
    currentReducer = nextReducer
    // 用旧的state树的某些数据 和一些其他的数据信息 构建新的state树
    dispatch({ type: ActionTypes.REPLACE })
  }
```

replace之后要使用 当前的currentState + 新的rootReducer（nextReducer），来构建新的 state 树

#### dispatch——核心方法，派发action+通知观察者

```javascript
/**
   * // dispatch 是触发改变 store 的唯一方法
   *
   * // reducer 定义了根据 旧的store树和具体action，返回新的store 树的逻辑
   * 同时会触发所有的观察者
   *
   * // 基本方法只支持 dispatch 一个对象，
   * 对于 Promise、Observable、thunk函数或者其他对象，需要使用 middleware 处理
   *
   * @param {Object} action 代表改变的部分，对象
   *
   * @returns {Object} 返回值为 dispatch 的action 对象
   *
   * // 自定义中间件 需要包装 dispatch 方法来返回一些其他的东西
   */
  function dispatch(action) {
    ... 一堆检查

    try {
      isDispatching = true // 将派发状态位设置为 true
      currentState = currentReducer(currentState, action) // 用传入的reducer对store 状态树进行更新
    } finally {
      isDispatching = false // 不管更新是否成功都要把派发状态位 设置为 false
    }

    const listeners = (currentListeners = nextListeners) // 将当前listeners监听数组设置为 操作过的副本listeners数组
    // 在dispatch 的时候才将更新的 listener 数组赋值给当前listener 数组
    // 循环过程中新注册的 观察者方法 不会影响本次循环（CurrentListener）
    for (let i = 0; i < listeners.length; i++) {
      const listener = listeners[i]
      listener() // notify 每个观察者方法
    }

    return action // 返回dispatch 的原 action
  }
```

可以看出，其实 `isDispatching` 字段代表的，是当前是否处于 reducer 执行过程中（而不是dispatch 过程中）

#### others

createStore 方法除了这些方法之外，也在函数体内部执行了一些代码

```javascript
export default function createStore(reducer, preloadState, enhancer) {
  // 略去一些检查参数类型以及个数的方法
  if (typeof enhancer !== 'undefined') {
    ... // enhancer 必须为函数
    // enhancer为增强函数（中间件）
    // 其实是 applyMiddlewares 的返回值函数
    return enhancer(createStore)(reducer, preloadedState)
  }
  ...
  let currentReducer = reducer // 当前 store 中的 reducer
  let currentState = preloadedState // 当前 store 中存储的状态
  let currentListeners = [] // 当前 store 中放置的 监听函数
  let nextListeners = currentListeners // 下一次dispatch notifyListeners 时的监听函数
  // 注意，当我们新添加一个监听函数时，这个监听函数只会在下一次 dispatch 的时候生效
  let isDispatching = false // 当前是否处于一次dispatch过程中
  
  ... // 定义函数们
  
  dispatch({ type: ActionTypes.INIT });
  return {
    dispatch,
    subscribe,
    getState,
    replaceReducer,
    [$$observable]: observable // [$$observable] 是什么意思
  }
  
}
```

createStore 可以 传 reducer+enhancer 或 reducer+preloadState+enhancer

##### 初始化的过程

另外，可以看到在return之前，派发了一个 init 的action，也就是说，在createStore 的时候，同时会初始化 initialState，刚开始的 currentState 为 空，那么在dispatch 调用 reducer 时，传入的 state 为空，而action为redux内置的redux

在使用 reducer 时

```javascript
const initialState = {
  ...
}

export default function todos(state = initialState, action) {
	switch (action) {
      ...
      default: 
      	return state;
  }
}
```

在我们自定义的 reducer 中当然不会匹配到 任何action，所以会返回 defaultState，也就是 initialState

### combineReducer——合并 reducer 对象

传入一个

```javascript
{
  key1: reducer1,
  key2: reducer2,
  ...
  keyN: reducerN
}
```

对象

返回一个 函数，这个函数接受一个 state 和 action，遍历这个 rootReducer，执行其中的每个 reducer 方法并更新 对应的state，具体代码解读见 [redux-combineReducer](https://github.com/aqqzzz/redux/blob/master/src/combineReducers.js)

### bindActionCreators 

生成一个 键值为函数的对象，调用对应的函数时触发 dispatch 函数

### compose——组合传入的函数

组合传入的一系列函数，中间件时会用到

```javascript
return funcs.reduce((a, b) => (...args) => a(b(...args)))
```

只有最后一个函数可以接受多个参数

返回值为： `(...args) => f(g(h(...args))).` 的一个函数

### applyMiddlewares——增强 dispatch

在dispatch之前为 dispatch添加附加功能，返回 Store对象，并且用包装后的dispatch替代原来的dispatch

```javascript
export default function applyMiddleware(...middlewares) {
 	// 返回值为一个多层嵌套的函数
  return createStore => (...args) => {
    const store = createStore(...args)
    let dispatch = () => {
      throw new Error(
        'Dispatching while constructing your middleware is not allowed. ' +
          'Other middleware would not be applied to this dispatch.'
      )
    }

    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args) // 这里为什么要用匿名函数包裹返回 dispatch，而不是直接用我们定义的dispatch
      /**
       * js函数传参是按值传递的，如果我们直接返回 用自定义的dispatch 去调用的话，middleware(API)执行时的dispatch是我们传入dispatch时
       * 那个throw Error 的内存地址，
       * 而我们之后会对这个dispatch进行增强，并重新给它赋值，这时js会在堆内存中分配一块新的内存来存放这个新的dispatch 函数实体，
       * 并把栈中dispatch变量的值修改为这个堆内存地址，
       * 这个时候，当我们对middlerware传入 action进行调用时，它对应的dispatch 是我们更新前的 dispatch 函数实体
       * 匿名函数的作用就是，把这个传递的值变为 这个匿名函数的内存地址，而当它被真正调用的时候再去调用真正的dispatch
       * 其实就是把dispatch 包装了一层，在真正 dispatch action 的时候再去对应这个dispatch 真正的函数体（也就是增强之后的函数实体）
       */
    }
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    dispatch = compose(...chain)(store.dispatch) 
    // 这里的compose其实就是一系列的增强函数，传入store.dispatch是最终触发compose执行的参数
    // f(g(h(store.dispatch))) 相当于 next(g(h(store.dispatch)))
    // 因为我们需要返回一个可以替换原先dispatch 的函数，所以这个返回值其实也应该是一个可以接受 一个action作为参数 的函数
    // 只有在真正传入action 的时候，dispatch 才会被调用

    return {
      ...store,
      dispatch
    }
  }
}
```

由此的话我们可以推出中间件的写法：因为中间件是要多个首尾相连的，需要一层层的“加工”，所以要有个next方法来独立一层确保串联执行，另外dispatch增强后也是个dispatch方法，也要接收action参数，所以最后一层肯定是action。

```javascript
// redux-thunk 源码
// 中间件代码真正执行的时机是 传入 action之后，所以此时需要保证 dispatch 为更新过后的函数地址
function createThunkMiddleware(extraArgument) {
    return ({getState, dispatch}) => next => action => {
        if (typeof action === 'function') {
            return action(dispatch, getState, extraArgument)
        }
        return next(action)
    }
}
```

理解：传入action时真正触发 dispatch 执行，若action是一个函数，则调用这个函数并将增强后的dispatch传进去（由applimiddlerware源码可知，dispatch是函数内声明的一个自定义变量，而最后增强store.dispatch时我们又将dispatch重新赋值，所以这里在真正传入action执行的时候，dispatch是增强后的dispatch）

若action不是一个函数，则调用 next(action)，就是一层层调用 next，其实就是增强dispatch之后最后调用 store.dispatch

redux-thunk 的作用：普通的dispatch只能dispatch一个对象，redux-thunk可以让我们dispatch 一个函数，这个函数的参数是（dispatch, getState, payload），dispatch函数之后，我们就可以在这个函数里进行异步处理或调用接口

thunk 的含义就是 延迟执行，这里其实是延迟了真正的dispatch，只有在dispatch action 的时候才会触发reducer更新，所以这里实际上是延迟了 真正的reducer更新 

#### Q1：middleware为什么要嵌套函数？为何不在一层函数中传递三个参数，而要在一层函数中传递一个参数，一共传递三层？

因为中间件是要多个首尾相连的，对next进行一层层的“加工”，所以next必须独立一层。那么Store和action呢？Store的话，我们要在中间件顶层放上Store，因为我们要用Store的dispatch和getState两个方法。action的话，是因为我们封装了这么多层，其实就是为了作出更高级的dispatch方法，是dispatch，就得接受action这个参数。

函数柯里化，提前传入一些参数，构造好最后执行的函数，最后就可以只传入action进行调用了，也是对dispatch 的一个封装和增强

#### Q2：middlewareAPI中的dispatch为什么要用匿名函数包裹呢？

我们用applyMiddleware是为了改造dispatch的，所以applyMiddleware执行完后，dispatch是变化了的，而middlewareAPI是applyMiddleware执行中分发到各个middleware，所以必须用匿名函数包裹dispatch，这样只要dispatch更新了，middlewareAPI中的dispatch应用也会发生变化。

也就是说，匿名函数包裹之后，只有在真正执行dispatch的时候（传入action之后），系统才会去查找其真正指向的函数进行调用

如果不包裹的话，传入middleware中的 dispatch 其实是增强前的dispatch 地址（结合函数按值传参的特性理解）可以同时查看源码中我的注释

#### Q3: 在middleware里调用dispatch跟调用next一样吗？

因为我们的dispatch是用匿名函数包裹，所以在中间件里执行dispatch跟其它地方没有任何差别，而执行next相当于调用下个中间件。

### 总结

单纯redux 的使用流程为：

1. combineReducer
2. applyMiddlewares( ...midlewares )
3. createStore( rootReducer, preloadState, applyMiddlewares(...midlewares))

### 参考

[Redux从设计到源码](https://tech.meituan.com/2017/07/14/redux-design-code.html)



### TODO

- react-redux
- react-router
- 函数柯里化