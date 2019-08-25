---
title: react-redux（一）——如何使用
date: 2019-08-25 18:59:28
tags:
- React
- react-redux
categories:
- React
---

昨天读了 redux 的源码，今天来看一下在React全家桶里经常和它配对使用的 react-redux

这个源码有点复杂，感觉按我现在的能力完全读懂还是有点难，就先挑着 重点看一下

### React-redux 的使用

React-redux 其实就是一个在 react 和 redux 之间架起桥梁的角色，将组件关心的 store state 和 对应的更新方法 作为props绑定到对应的组件上，实现 react 和 redux 之间的隔离

其中起到重要作用的主要有两个部分：Provider 和 connect

#### connect

```ts
function connect(
	mapStateToProps?: (state, props) => Object,
  mapDispatchToProps?: (dispatch, ownProps) => Object,
  mergeProps?: (stateProps, dispatchProps, ownProps) => Object,
  options?: Object
): Function {}
```

##### mapStateToProps

```javascript
function mapStateToProps(state, ownProps) {
  return plainObject
}
```

1. 参数中的 state 为 store.state， props 则为组件的props

   - 只提供 state 参数——只有state发生变化的时候会重新计算 mapStateToProps
   - state 和 props同时提供——不管props是否用到，只要state发生变化或者 props有变动，都会重新计算 mapStateToProps

   所以，如果返回对象中不需要 props 进行计算，那么就不要添加这个参数，否则会多很多次mapStateToProps 的计算

   与此同时，如果在只提供state的情况下，state没有发生变化，那么是不会重新计算 mapStateToProps 函数的

2. 可以在 mapStateToProps 函数中做一些简单的处理，但是不能太多，因为 mapStateToProps 的计算频率很高，如果很复杂的话会耗费资源

3. 尽可能在需要的时候再返回新的对象引用

   react-redux会对 mapStateToProps 的结果做一次浅比较来查看结果是否有改动，然而某些方法很容易在每次都返回新的对象引用，即使真实依赖的对象或数据并没有改动，这样就会导致——即使数据实际上没有改动，但是组件还是会重新渲染。这些方法例如：

   - 使用 `someArray.map()` 或 `someArray.filter()` 生成新的 array
   - `array.concat()` 合并数组
   - 使用 `array.slice()` 获取array 的一部分
   - 使用 `Object.assign()` 合并对象
   - 使用扩展运算符合并对象：`{...oldState, ...newState}`

   使用这些方法时要尽量小心，尽可能把他们转移到 redux 的 action 或 reducer中计算，或者在对应组件中根据prop进行计算，尽量减少这种无畏的计算与更新

   或者，如果这个计算一定要在 mapStateToProps 中计算的话，建议使用 memoized select Function(?????) 来确保，只有输入真正发生变动的时候才会重新计算mapStateToProps

##### mapDispatchToProps

建立UI组件参数到 store.dispatch 方法的映射，它定义了哪些用户的操作应该当做 Action 传给 Store。可以是一个函数，也可以是一个对象。

函数形式：参数 dispatch + 组件props

```javascript
const mapDispatchToProps = (
  dispatch,
  ownProps
) => {
  return {
    onClick: () => {
      dispatch({
        type: 'SET_VISIBILITY_FILTER',
        filter: ownProps.filter
      });
    }
  };
}
```

对象形式：

```javascript
const mapDispatchToProps = {
  onClick: (filter) => {
    type: 'SET_VISIBILITY_FILTER',
    filter: filter
  };
}
```

##### mergeProps

```js
function mergeProps (stateProps, dispatchProps, ownProps) {
  return {
    ...ownProps,
    ...stateProps,
    ...dispatchProps,
  }
}
```

- stateProps为 `mapStateToProps` 的返回值
- dispatchProps 为 `mapDispatchToProps` 的返回值
- ownProps 为 组件自身的props

以上代码片段中写出的是 默认的 mergeProps 的实现，我们可以覆盖这个函数的返回值，对其进行一些特殊处理（TODO 不大知道实际应用场景是什么）

##### Options

```typescript
{
  context?: Object, 
  pure?: boolean, // default: true
  areStatesEqual?: Function, // strictEqual: (prev, cur) => prev === cur
  areOwnPropsEqual?: Function, // shallowEqual: (objA, objB) => boolean ( 当两个对象每个 field 的值都相同的时候返回true )
  areStatePropsEqual?: Function, // shallowEqual
  areMergedPropsEqual?: Function, // shallowEqual
  forwardRef?: boolean,
}
```

- context：

  基本没有用过，仿佛是可以用来替换整体 context 的一个东西，但是想要替换的话就必须把 Provider 中和connect中的 context 都换掉

- pure：默认为 true

  true 则代表，将当前组件看做是一个 pure component，它只依赖于 自身的props 和 connect 中从 Redux.state 中选择的 state，与其它无关

  当这个值为 true 的时候，connect 会做一些 额外的检查来判断，是否需要重新执行 `mapStateToProps`、`mapDispatchToProps`、`mergeProps`，并最终决定是否需要执行组件的 `render` 方法	

  这些方法为：`areStatesEqual`、`areOwnPropsEqual`、`areStatePropsEqual`、`areMergedPropsEqual`

  这些方法大多数情况下都是会被省略的，但是也可以覆盖来提高效率（并提升看源码的效率）

- areStatesEqual：pure为true的情况下，覆盖 state 变动的判断逻辑

- areOwnPropsEqual：pure为true的情况下，覆盖 props 变动的判断逻辑（当我们重写了这个方法的时候，通常需要在 mapStateToProps、mapDispatchToProps、mergeProps 中忽略对应的prop属性）

- areStatePropsEqual：pure为true的情况下，对 `mapStateToProps` 的结果和其 之前的计算结果进行比对

- areMergedPropsEqual：pure为true的情况下，对 mergeProps 的结果与其之前的计算结果进行比对

- forwardRef：true时，给 connectWrapComponent 添加一个 ref 属性

#### Provider

`connect`方法生成容器组件以后，需要让容器组件拿到`state`对象，才能生成 UI 组件的参数。

一种解决方法是将`state`对象作为参数，传入容器组件。但是，这样做比较麻烦，尤其是容器组件可能在很深的层级，一级级将`state`传下去就很麻烦。

React-Redux 提供`Provider`组件，可以让容器组件拿到`state`。

> ```javascript
> import { Provider } from 'react-redux'
> import { createStore } from 'redux'
> import todoApp from './reducers'
> import App from './components/App'
> 
> let store = createStore(todoApp);
> 
> render(
>   <Provider store={store}>
>     <App />
>   </Provider>,
>   document.getElementById('root')
> )
> 
> ```

上面代码中，`Provider`在根组件外面包了一层，这样一来，`App`的所有子组件就默认都可以拿到`state`了。

它的原理是`React`组件的[`context`](https://facebook.github.io/react/docs/context.html)属性，请看源码。

> ```javascript
> class Provider extends Component {
>   getChildContext() {
>     return {
>       store: this.props.store
>     };
>   }
>   render() {
>     return this.props.children;
>   }
> }
> 
> Provider.childContextTypes = {
>   store: React.PropTypes.object
> }
> ```



