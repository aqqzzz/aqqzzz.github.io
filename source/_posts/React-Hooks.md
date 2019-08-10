---
title: React Hooks
date: 2019-08-09 18:24:27
tags:
- React
- React hooks
categories:
- React
---

这两天在公司写需求的时候，有一个项目react版本在 16.8，所以想顺便学习一下 react hooks

react-hooks是 react 加在 函数式组件里的一些增强方法，简单说来，就是让 函数式组件，也具备了类组件维持自己内部状态的能力

### 为什么要设计 hooks

而为什么需要用hook来代替类组件的这些功能呢，按照 Dan 的说法，设计 hooks 主要是为了解决 classComponent 的以下几个问题：

- 很难复用逻辑，只能用 HOC，或者 renderProps，导致组件层级嵌套很深
- 会产生巨大的组件（指很多方法会被写到类里）
- 类组件不好理解，比如方法需要bind，this指向不明确

HOC嵌套确实是一个问题，如果是多层嵌套的话，组件层级嵌套深，也会不好理解

然后还有在`componentDidMount`和`componentDidUpdate`中订阅内容，还需要在`componentWillUnmount`中取消订阅的代码，里面会存在很多重复性工作。最重要的是，在一个`ClassComponent`中的生命周期方法中的代码，是很难在其他组件中复用的，这就导致了代码复用率低的问题。

而我们都很清楚，react 中原有的 函数式组件，只能根据prop进行render，是没有办法保存自己的状态的，所以，hooks的出现，就是为了让我们在避免 class component 冗杂的代码的同时，还可以在 function component 里维护并使用自己的状态

那么，react 的hooks 是怎么在函数式组件内部存储自己的状态的呢

### hooks实现原理

react 16 提出了 Fiber 架构的概念，通过把之前无法被打断的循环diff过程拆分成一个一个的小任务，以划分的时间片为执行时间单位，在这个时间片单位里，要执行以下的几个任务

![](http://ww1.sinaimg.cn/large/8ac7964fly1g5klvalzkjj21jk0i3gp4.jpg)

这些任务执行完毕之后，如果还有空余时间，就会执行 diff 过程划分的小任务，而这些小任务，就是以 fiber node 为基本单位的

fiber node 基本结构如下：

```javascript
{
  ...
  // 浏览器环境下指 DOM 节点
  stateNode: any,

  // 形成列表结构
  return: Fiber | null,
  child: Fiber | null,
  sibling: Fiber | null,

  // 更新相关
  pendingProps: any,  // 新的 props
  memoizedProps: any,  // 旧的 props
  // 存储 setState 中的第一个参数
  updateQueue: UpdateQueue<any> | null,
  memoizedState: any, // 旧的 state

  // 调度相关
  expirationTime: ExpirationTime,  // 任务过期时间

  // 大部分情况下每个 fiber 都有一个替身 fiber
  // 在更新过程中，所有的操作都会在替身上完成，当渲染完成后，
  // 替身会代替本身
  alternate: Fiber | null,

  // 先简单认为是更新 DOM 相关的内容
  effectTag: SideEffectTag, // 指这个节点需要进行的 DOM 操作
  // 以下三个属性也会形成一个链表
  nextEffect: Fiber | null, // 下一个需要进行 DOM 操作的节点
  firstEffect: Fiber | null, // 第一个需要进行 DOM 操作的节点
  lastEffect: Fiber | null, // 最后一个需要进行 DOM 操作的节点，同时也可用于恢复任务
  ....
}
```

这些fiber node，就是通过 return、child、和 sibling 串成一个链表结构的。

这些属性中，hook里需要特别关注的，就是 memoizedState

在普通的class component中，这里的memoizedState就是一个存储在上次渲染过程中最终获得的节点的 state，每次执行`render`方法之前，React会计算出当前组件最新的`state`然后赋值给`class`的实例，再调用`render`

而在 function component 中其实也是这个字段来存储 组件状态的，但是它的机制又有所不同，hook 组件的 memoizedState，不是一个直接代表state整体的对象，而是一个特殊的数据结构，我们称其为一个 hook 对象

```javascript
{
  baseState,
  next,
  baseUpdate,
  queue,
  memoizedState
}
```

重点关注 memoizedState、 next 和 queue

为什么要用这种数据格式呢？主要是因为 function component 中，React 不知道我们调用了几次 useState，所以就索性使用这种类似链表的结构， memoizedState 代表 这次useState 设置的state，next指向下一个 useState代表的hook对象，这是一个顺序结构

在首次 render 的时候，function component 就会构建一个这样的 Fiber hook 链表

在之后的更新过程中，每次setState都会派发一个update 对象

```javascript
var update = {
  expirationTime: _expirationTime, // 过期时间(根据优先级计算)
  action: action, // setState 更新的值
  callback: callback !== undefined ? callback : null,
  next: null
}
```

这其中的 action 就是我们调用 setState 传入的值，react 把这个update 对象加到 hook 对象的queue中，在之后 batchUpdate 的过程中，从 Hook 对象的queue中拿出所有的 update依次执行，更新 fiber node 中的 memoizedState，从而达到 存储内部state 的目的

**为什么React Hook 必须在函数组件的最外层调用**：

其实看了上面 hook 的内部实现，我们应该也可以理解，Hook 对象这种类似链表的存储方式，对hook次序的要求是很高的，如果我们某一次 的hook在 条件判断里，那么我们并不能保证这次useState是一定会执行的，假设 第一次render 的时候条件判断为 true，这个 hook 被加到了我们之前说的 那个链表结构里去，而在更新的时候条件判断变成了false，这个hook没有执行，那么其前一位hook结构的 next 虽然在更新，但是用问题hook的更新方法去更新了这个问题hook的下一个state，eg：

```javascript
if (something) {
	const [state1] = useState(1)
}
const [state2] = useState(2)
```

本来的对应关系是 hook1 -> state1， hook2 -> state2，结果某次render 导致 state1 没有执行，那么链表结构里就会拿 state1的更新方法去更新 state2 的对应数据，这一逻辑显然是错误的，所以才会有这样的限制

更简单的理解模型：

[React hook: not magic, just arrays](https://medium.com/@ryardley/react-hooks-not-magic-just-arrays-cd4f1857236e)

### 参考

[react源码解读](https://yuchengkai.cn/react/2019-04-24.html#文章相关资料)

[阅读源码后，来讲讲React Hooks是怎么实现的](https://juejin.im/post/5bdfc1c4e51d4539f4178e1f#comment)

[React hook: not magic, just arrays](https://medium.com/@ryardley/react-hooks-not-magic-just-arrays-cd4f1857236e)