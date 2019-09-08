title: react-redux 源码解读一
date: 2019-08-25 21:48:13
tags:

- React
- react-redux
categories:
- React

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
    // 也就是把 notify 当前组件的方法，作为 观察者注册到 parent Subscription 上，添加到 parentSubscription 的 观察者列表里
    // 保证了在父组件更新时，先调用父组件的观察者方法，再调用子组件的观察者方法
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

我理解 Subscription 类的作用，就是把 store 对象/ parentSub 和 当前对象的Subscription对象关联起来，用来构造一个顺序正确的观察者调用序列

在当前 Provider中，它其实可以起到的作用就是把 观察者注册到 store 上（Provider 通常用法其实就是包在整个应用的最外层），当store数据发生变化时通知最外层组件，现在我们的 react-redux 只是简单地通过 context 将 redux 的 store 挂在整个应用最外层，使得内部的 react 组件可以通过 **context API** 来获取 redux store 中的各种方法并进行调用。

同时，我们也可以看到，在 Subscription 代码中，除了调用 store.subscribe 直接向 redux store 注册观察者之外，也调用了 ParentSubscription 的观察者注册方法，这一段代码，就是为了让我们在 Provider 内部的 react 组件中，可以不用手动调用 context API 来获取 redux store，同时还可以帮我们关注我们需要关注的部分，而不需要将关注所有 redux store state 的变动，也就是说，这一部分同时也是设计来给 connect 调用的。

### connect.js

connect 其实只是一个包装器，其内部真正实现 connect 功能的，是 SelectFactory 和 connectAdvanced

connect 的功能，就是将 mapStateToProps 和 mapDispatchToProps 统一成接受相同参数的函数，然后将

- 此处的 initMapStatetoProps 和 initMapDispatchToProps，其实是一个接受（dispatch，Object）为参数的函数，

- 在 connectAdvanced 中初始化 SelectorFactory 时

  ```js
  const mapStateToProps = initMapStateToProps(dispatch, options)
  const mapDispatchToProps = initMapDispatchToProps(dispatch, options)
  const mergeProps = initMergeProps(dispatch, options)
  ```

  各个init 返回的是一个 proxy 函数，这个 proxy 函数包装了真正的 mapXXXToProps 函数

- 初始化 SelectorFactory 返回的是一个函数，接受 state 和 prop作为参数，计算获取下一次的 mergeProps

- 当我们真正调用 `mapXXXToProps(state/dispatch, ownProps)` 时，其实调用的是之前的 proxy 函数

  这个 proxy 函数 处理了一个情况，当 mapXXXToProps 返回的是一个 接受 （dispatch/state， ownProps）为参数的函数时，将返回的这个函数作为最后的 mapXXXToProps 真正调用的函数，但是如果这个返回函数也返回了一个函数，即 `dispatch => dispatch => dispatch => ({})` ，就会提示错误，mapDispatch 必须返回一个函数

  TODO 不清楚这里为什么要单独对 `dispatch => dispatch => ({})` 进行处理

connect代码：https://github.com/aqqzzz/react-redux/blob/master/src/connect/connect.js

wrapMapToProps：https://github.com/aqqzzz/react-redux/blob/master/src/connect/wrapMapToProps.js

应用到了哪些设计模式：

1. 简单工厂模式：根据传入参数不同，返回不同的 `initMapXXXToProps` 函数，而这些返回的函数都接受相同的参数进行调用

   ```js
   const initMapStateToProps = match(
     mapStateToProps,
     mapStateToPropsFactories,
     'mapStateToProps'
   )
   
   // match
   function match(arg, factories, name) {
     for (let i = factories.length - 1; i >= 0; i--) {
       // 依次调用 factory 方法，当返回回调函数时将其return
       const result = factories[i](arg)
       if (result) return result
     }
   
     return (dispatch, options) => {
       throw new Error(
         `Invalid value of type ${typeof arg} for ${name} argument when connecting component ${
           options.wrappedComponentName
         }.`
       )
     }
   }
   
   // mapStateToPropsFactories
   import { wrapMapToPropsConstant, wrapMapToPropsFunc } from './wrapMapToProps'
   
   // 参数 mapStateTOProps 是我们在 connect 中传入的第一个参数函数，这个函数对应 conenct mapstateToProps 传入函数的情况
   export function whenMapStateToPropsIsFunction(mapStateToProps) {
     return typeof mapStateToProps === 'function'
       ? wrapMapToPropsFunc(mapStateToProps, 'mapStateToProps')
       : undefined
   }
   
   export function whenMapStateToPropsIsMissing(mapStateToProps) {
     return !mapStateToProps ? wrapMapToPropsConstant(() => ({})) : undefined
   }
   
   export default [whenMapStateToPropsIsFunction, whenMapStateToPropsIsMissing]
   ```

2. 代理模式：使用 proxy 代理真正 mapXXXToProps 的执行，在其中统一处理了一些异常情况

   ```js
   export function wrapMapToPropsFunc(mapToProps, methodName) {
     return function initProxySelector(dispatch, { displayName }) {
       const proxy = function mapToPropsProxy(stateOrDispatch, ownProps) {
         return proxy.dependsOnOwnProps
           ? proxy.mapToProps(stateOrDispatch, ownProps) // 【proxy.mapToProps】 是会不断更新的
           : proxy.mapToProps(stateOrDispatch)
       }
   
       // allow detectFactoryAndVerify to get ownProps
       proxy.dependsOnOwnProps = true
   
       proxy.mapToProps = function detectFactoryAndVerify(
         stateOrDispatch,
         ownProps
       ) {
         proxy.mapToProps = mapToProps // 【proxy.mapToProps】初始化为传入的 mapToProps(mapStateToProps 或 mapDispatchToProps)
         proxy.dependsOnOwnProps = getDependsOnOwnProps(mapToProps)
         let props = proxy(stateOrDispatch, ownProps)
   
         // 如果 mapToProps的返回值为一个 function，那么将这个返回的新function 作为之后调用中的 mapToProps 函数
         // 例如 当mapDispatchToProps 的返回值为 bindActionCreators 函数时就会走到这个流程
         if (typeof props === 'function') {
           proxy.mapToProps = props // 【proxy.mapToProps】二次更新，为 初始化 mapToProps时返回的 函数
           proxy.dependsOnOwnProps = getDependsOnOwnProps(props)
           props = proxy(stateOrDispatch, ownProps) // 使用新的 mapToProps，调用这个函数获取返回值
         }
   
         if (process.env.NODE_ENV !== 'production')
           verifyPlainObject(props, displayName, methodName)
   
         return props
       }
   
       // 最终返回的对象为这个 proxy 对象，当真正调用 mapStateToProps(state, ownProps) 时，其实调用的是 proxy(state, ownProps)
       return proxy
     }
   }
   ```

   

TODO 代码了解还不够，之后在看完 connectAdvanced 之后再回来看这里是不是有新的发现

很多地方都看不懂我擦

决定明天看完整个代码之后再写读源码文档

### SelectorFactory

代码：https://github.com/aqqzzz/react-redux/blob/master/src/connect/selectorFactory.js

不贴代码了，这个文件的作用就是，调用 被代理的 mapXXXToProps，获得 map 之后的关注对象

 一个小优化

当 options.pure 为 true 时，当传入 react-redux 的 state 和 props 发生变更时，这里会根据对比结果做一个优化

- store.state 【全等判断】为false && props 【浅比较】结果为false ---->  更新 mapStateToProps 和 mapDispatchToProps 的结果
- store.state 全等判断为false ------> 更新 mapStateToProps
- props 浅比较结果为false -----> 更新 mapStateToProps 和 mapDispatchToProps

然后把这个merge结果返回

这也就是为什么，我们在 mapXXXToProps 时，只有在必要的时候才会传入 第二个参数 ownProps，以减少 map 的计算次数

### connectAdvanced

react-redux 已经用 钩子函数重构这个文件了，有很多地方我还是看不懂，先挑着主流程走通

```js
function WrapWithConnect(WrappedComponent) {
  function createChildSelector(store) {
    return new selectorFactory(...)
  }
  // props 是Connect组件上传入的 props
  function ConnectFunction(props) {
    ...
  }
  const Connect = pure ? React.memo(ConnectFunction) : ConnectFunction

  Connect.WrappedComponent = WrappedComponent
  Connect.displayName = displayName
  if (forwardedRef) {
    // 返回forwardRef 的 Connect 对象，并把当前 props一并传入
    const forwarded = React.forwardRef(function forwardConnectRef(
        props,
        ref
      ) {
        return <Connect {...props} forwardedRef={ref} />
      })

      forwarded.displayName = displayName
      forwarded.WrappedComponent = WrappedComponent
      return hoistStatics(forwarded, WrappedComponent)
  }
  // hoistStatics(targetComponent, sourceComponent)，将 sourceComponent上 非React 的静态方法copy到targetComponent上
  return hoistStatics(Connect, WrappedComponent)
}
```

这一部分就是声明了一些函数，并对 WrappedComponent 做了一些操作，之后将包装过的 Connect 和 WrappedComponent 返回

关键函数在 ConnectFunction 中，这是一个使用了很多React 16 钩子函数的地方，通过 React.Memo、React.useLayoutEffect等钩子函数，将关注的props以正确的顺序注册到了 WrappedComponent 上

#### 将包装过的 WrappedComponent（props） 和 contextValue（context.Provider）传递下去—— renderedChild

```js
// 渲染connect子组件的关键代码，同时也使用 useMemo 对其做了一次优化
const renderedWrappedComponent = useMemo(
		() => <WrappedComponent {...actualChildProps} ref={forwardedRef} />, // 将计算得到的 actualChildProps 添加到真正的组件上
    [forwardedRef, WrappedComponent, actualChildProps] // 当 forwradedRef、wrappedCompoennt 或 计算得到的 actualChildProps更新时触发更新
)
const renderedChild = useMemo(() => {
    if (shouldHandleStateChanges) {
      // 作为 Provider 的目的是把 当前的subscription 对象作为 context 传给子组件，保证 按照组件父子顺序 调用观察者函数
      return (
        <ContextToUse.Provider value={overriddenContextValue}>
        	{renderedWrappedComponent}
   			</ContextToUse.Provider>
    	)
  	}

  	return renderedWrappedComponent
}, [ContextToUse, renderedWrappedComponent, overriddenContextValue])

return renderedChild
```

renderedChild：将当前使用的 overriddenContextValue 作为 Context 的 Provider 挂载到 renderedWrappedComponent 上

renderedWrappedComponent 其实就是 渲染 connect 子组件的关键代码，将处理之后的 props 挂到真正的 component 上去

#### 保证观察者被调用的顺序——overriddenContextValue（subscription）

```js
// 当 store、didstorcomfromprops、context值发生变化时重新计算 subscription，notifyNestedSubs
const [subscription, notifyNestedSubs] = useMemo(() => {
  if (!shouldHandleStateChanges) return NO_SUBSCRIPTION_ARRAY

  // This Subscription's source should match where store came from: props vs. context. A component
  // connected to the store via props shouldn't use subscription from context, or vice versa.
  const subscription = new Subscription(
    store,
    didStoreComeFromProps ? null : contextValue.subscription // 如果 store 是从 context 中获取的，那么就要把 context中的 subscription作为 parent subscription 传入 当前观察类中去
  )

  // `notifyNestedSubs` is duplicated to handle the case where the component is unmounted in
  // the middle of the notification loop, where `subscription` will then be null. This can
  // probably be avoided if Subscription's listeners logic is changed to not call listeners
  // that have been unsubscribed in the  middle of the notification loop.
  const notifyNestedSubs = subscription.notifyNestedSubs.bind(
    subscription
  )

  return [subscription, notifyNestedSubs]
}, [store, didStoreComeFromProps, contextValue])

// 重写 Context 对象
const overriddenContextValue = useMemo(() => {
   if (didStoreComeFromProps) {
     // This component is directly subscribed to a store from props.
     // We don't want descendants reading from this store - pass down whatever
     // the existing context value is from the nearest connected ancestor.
     return contextValue
   }

   // Otherwise, put this component's subscription instance into context, so that
   // connected descendants won't update until after this component is done
   // 为了保证 更新顺序，保证触发更新时，父组件先更新，其后代组件再更新
   return {
     ...contextValue,
     subscription
   }
 }, [didStoreComeFromProps, contextValue, subscription])
```

主要作用就是当 store 不是来自 props（也就是说，当前组件不是 Provider 直接子组件时），将当前组件的 subscription 对象作为 context 的一部分，传递给其包装的子组件，

当前组件的 subscription，则是通过前一个 useMemo，使用

```js
 const subscription = new Subscription(
    store,
    didStoreComeFromProps ? null : contextValue.subscription // 如果 store 是从 context 中获取的，那么就要把 context中的 subscription作为 parent subscription 传入 当前观察类中去
  )
```

新建的，通过这种方式，我们确保了 connect 时，父组件的 connect 函数比子组件更早触发

#### 计算connect真正关注的childProps——actualChildProps

```js
// 返回 根据connect提供的三个方法（state、dispatch、merge）计算得到的，与本组件相关的 props
const actualChildProps = usePureOnlyMemo(() => {
  // 通过 提供的 mapSTateToProps、mapDispatchToProps、mergeProps 计算新的 mergeProps
  return childPropsSelector(store.getState(), wrapperProps)
}, [store, previousStateUpdateResult, wrapperProps])
```

childPropsSelector：store 发生变化时重新计算 childSelector， 生成新的函数（selectorFactory），这个函数接受一个 state 和 prop 参数，通过 mapStateToPRops、mapDispatchToProps、mergeProps 计算新的 merge结果

这里的wrapperProps 其实就是 参数里传进来的 `initMapStateToProps, initMapDispatchToProps, initMapMergeToProps`

#### 总结

connectAdvanced 的主流程可以概括为下图

![image.png](https://i.loli.net/2019/09/08/JCplkdPin3hmjAT.png)

### 总结

react-redux 的源码有很多难读的地方，我现在也没有完全读懂，只能看得懂大概流程，大致概括，react-redux做的事情就是，

1. 使用 React 的 context 语法，通过 Provider 将 redux 的 store 对象注册到 最外层的 context 上
2. 使用 connect ，将 component 关注的 props 和对应的更新方法从 context 里取出来，作为 props 放到被connect 包装的组件上去，同时将 subscription 对象作为 父subscription 通过 context 传递给子组件，以保证 父子监听调用的顺序



react-redux 看了好久，怎么说，还是觉得自己总结的不够好，之后可能还会再重新搞一下

多画图是一件很有用的事情