---
title: React 新旧生命周期与 Fiber 架构
date: 2019-08-01 22:13:32
tags:
- React
- React 16
- Fiber
categories: 
- React
---

昨天回来莫名颓在沙发上开始刷手机，刷着刷着就十点了，洗完澡都十点半了，唉我怎么能这么不上进呢

8月了！新的月份要有新的学习样貌！

今天本来打算复习一下 React 的新旧生命周期，在看旧的笔记的时候想起来之前在网上看到过一份 Fiber 架构的详解，但是今天也没找到那一份，不过找到了另一份也同样优秀的Fiber架构讲解，果然世界上就是 大佬层出不穷呀。

所以今天就学习一下 React 新旧生命周期以及16版本中用到的Fiber架构。

## React 新旧生命周期的对比

### React 15

在 React 15以前的版本，React 生命周期可以按照 创建期、存在期 和 销毁期 大致分为三个区间段

- 创建期：
  - componentWillMount
  - render
  - componentDidMount
- 存在期：
  - prop改变触发：componentWillReceiveProps
  - prop改变或state改变之后触发：shouldComponentUpdate
  - 当shouldComponentUpdate 返回true时：componentWillUpdate
  - render
  - componentDidUpdate
- 销毁期：
  - componentWillUnmount

可以通过下面这个图来记忆和理解

![img](https://user-gold-cdn.xitu.io/2017/11/11/88e11709488aeea3f9c6595ee4083bf3?imageslim)

### React 16

![img](https://pic4.zhimg.com/v2-8c9f2b2eebc3449da805e8bd0deced47_r.jpg)

React 16看起来也是分了三个周期，但是其更核心的概念是它把React的更新过程分成了三个阶段

- render 阶段（或reconcile阶段）：在这个阶段计算虚拟dom的更新节点，在React 16 中，这个过程是可以被打断的（Fiber架构，下文详细介绍），React 可能会随时打断，或丢弃在这个Fiber时间片内所做的工作，所以在这个阶段的生命周期不能有函数副作用（后端请求或DOM操作等）
  - **static** getDerivedStateFromProps(nextProps, prevState)：
    1. 在 render 之前被调用
    2. 如何实现 this.props 和 nextProps 的对比——在prevState里存一个想要对比的字段，跟 nextProps 的对应字段进行对比
    3. 返回值应该为更新之后的 state（整个替换）
    4. **只能执行纯函数操作**：输出完全依赖于输入，不能执行副作用的过程
  - shouldComponentUpdate( nextProps, nextState )：
    1. 当 prop 或 state 更新时调用，默认返回 true
    2. 首次 render  和 forceUpdate 不会调用这个生命周期
    3. return false 只能阻止当前组件不进行diff，不能阻止子孙组件
  - render：其实就是构建虚拟dom树的过程（是可以被打断的）
- Pre-commit 阶段（预提交阶段）：可以读取到更新后dom节点的阶段（其实很少用）
  - getSnapshotBeforeUpdate( prevProps, prevState)：
    1. 调用时机：最近一次 render 的结果即将提交给 真实 DOM 之前调用，可以获取更新前的一些DOM信息
    2. 其返回值会被当做第三个参数传递给 componentDidUpdate
    3. 需要返回 null 或一个默认值
- commit 阶段：可以进行有副作用的操作，这个阶段的工作是不能被打断的，React必须一口气更新完
  - componentDidMount
  - componentDidUpdate
  - componentWillUnmount

### React 新旧生命周期的对比

要说的话其实React 新旧生命周期的最大区别，就在于 16 出了 Fiber 架构，这使得我们可以在 reconcile 调谐阶段打断React的生命周期，而如果用户在这个阶段执行了有函数副作用的函数，那么它可能会被调用多次，可能会造成无法预计的后果，所以说，React 16 用新的**静态生命周期函数** getDerivedStateFromProps 来替代 componentWillMount / componentWillReceiveProps / componentWillUpdate，就是为了保证用户没有办法在调谐阶段获取组件实例并进行有函数副作用的操作。

所以说，React 16 的重点，在于 Fiber 架构的提出，生命周期的更改，只是为了适应新的 Fiber 架构。

那么React 官方又为什么要提出这一改动这么大的Fiber架构呢

## Fiber架构

### 一、目标

核心目标为以下几个：

- 把可中断的工作分割成几个小任务
- 对正在做的工作调整优先次序、重做、复用上一次（做了一半的）成果
- 在父子任务之间从容切换，以支持React执行过程中的布局刷新

其实就是不希望**JS不受控制地长时间执行**，涉及到 浏览器主线程的执行，JS是在浏览器主线程上执行的，而浏览器主线程同时又要负责 样式计算、布局、动画等，如果JS长时间运行的话，就会阻塞这些工作的进行。

### 二、fiber 与 fiber tree

React 运行时存在三种实例

- DOM：真实DOM节点
- instance层：React 维护的 virtual DOM 树
- Elements：描述UI长什么样子（type，props）

instance 层其实就是上一篇 React 系列中将diff操作时的 虚拟dom树，在 render 时构造出 虚拟dom 树，将其与旧的dom树进行比对，然后把改动的部分 patch 到真实的DOM 节点上

Fiber 之前的调谐阶段被称为 Stack Reconcile，是自顶向下递归进行的，而且不能被打断，这样主线程上的布局、动画等周期性任务以及交互响应就无法立即得到处理，影响体验

Fiber 的 <font color="red">解决思路</font>：把调谐阶段拆分成一系列小任务，每次检查树上的一小部分，做完之后看还有没有时间继续做下一个任务，有的话继续，没有的话就把自己挂起，主线程不忙的时候再继续

增量更新需要保留自己被挂起时的上下文信息，所以之前简单的节点不够用了，扩展除了 fiber tree（其实主要是扩展了 instance 层），更新过程就是根据输入数据以及现有的fiber tree构造出新的fiber tree（workInProgress tree）

Fiber instance 层的结构：

- effect：每个 workInProgress tree 上都有一个 effect list 用来存放 diff 结果，当前节点更新完毕之后会向上 merge effect list
- workInProgress： workInProgress tree是reconcile过程中从fiber tree建立的当前进度快照，用于断点恢复
- fiber：fiber tree与vDOM tree类似，用来描述增量更新所需的上下文信息

effect 和 workInProgress 只在调谐阶段维护

fiber 节点的数据结构如下

```javascript
// fiber tree节点结构
{
    stateNode,
    child, // 孩子节点
    return, // 处理完成之后应该向谁提交自己的effect list
    sibling, // 兄弟节点
    ...
}
```

fiber 树是用单链表结构表示的

![img](http://www.ayqy.net/cms/wordpress/wp-content/uploads/2018/01/fiber-tree.png)

上层为 Stack Reconcile 的旧的dom树，下层为 Fiber 树结构

### 三、Fiber reconcile

终于要讲到最核心的地方了，Fiber reconcile 包括两个阶段：

1. 可中断（render / reconcilation），通过构造 workInProgress tree 得出 change
2. 不可中断（commit），应用这些 change

#### render / reconcilatoin 阶段

以fiber tree为蓝本，把每个fiber作为一个工作单元，自顶向下逐节点构造*workInProgress tree*（构建中的新fiber tree）

具体过程如下（以组件节点为例）：

1. 如果当前节点不需要更新，直接把子节点clone过来，跳到5；要更新的话打个tag
2. 更新当前节点状态（`props, state, context`等）
3. 调用`shouldComponentUpdate()`，`false`的话，跳到5
4. 调用`render()`获得新的子节点，并为子节点创建fiber（创建过程会尽量复用现有fiber，子节点增删也发生在这里）
5. 如果没有产生child fiber，该工作单元结束，把effect list归并到return，并把当前节点的sibling作为下一个工作单元；否则把child作为下一个工作单元
6. 如果没有剩余可用时间了，等到下一次主线程空闲时才开始下一个工作单元；否则，立即开始做
7. 如果没有下一个工作单元了（回到了workInProgress tree的根节点），第1阶段结束，进入pendingCommit状态

实际上是1-6的<font color="red">工作循环</font>，7是出口，工作循环每次只做一件事，做完看要不要喘口气。工作循环结束时，workInProgress tree的根节点身上的effect list就是收集到的所有side effect（因为每做完一个都向上归并）

所以，构建workInProgress tree的过程就是diff的过程，通过`requestIdleCallback`来调度执行一组任务，每完成一个任务后回来看看有没有插队的（更紧急的），每完成一组任务，把时间控制权交还给主线程，直到下一次`requestIdleCallback`回调再继续构建workInProgress tree

P.S.Fiber之前的reconciler被称为Stack reconciler，就是因为这些调度上下文信息是由系统栈来保存的。虽然之前一次性做完，强调栈没什么意义，起个名字只是为了便于区分Fiber reconciler

##### requestIdleCallback

歪题（只是介绍这个api的用法），如果只是想要了解 Fiber 的话直到它大概是在浏览器空闲时间片执行即可，可以跳过

浏览器在空闲时间片执行的其回调的一个函数

![](http://ww1.sinaimg.cn/large/8ac7964fly1g5klvalzkjj21jk0i3gp4.jpg)

图中一帧包含了用户的交互、js的执行、以及requestAnimationFrame的调用，布局计算以及页面的重绘等工作。 假如某一帧里面要执行的任务不多，在不到16ms（1000/60)的时间内就完成了上述任务的话，那么这一帧就会有一定的空闲时间，这段时间就恰好可以用来执行requestIdleCallback的回调，如下图所示：

![](http://ww1.sinaimg.cn/large/8ac7964fly1g5klvysghjj20jx08mt9o.jpg)

由于requestIdleCallback利用的是帧的空闲时间，所以就有可能出现浏览器一直处于繁忙状态，导致回调一直无法执行，这其实也并不是我们期望的结果（如上报丢失），那么这种情况我们就需要在调用requestIdleCallback的时候传入第二个配置参数timeout了？

```javascript
requestIdleCallback(myNonEssentialWork, { timeout: 2000 });

function myNonEssentialWork (deadline) {
  // 当回调函数是由于超时才得以执行的话，deadline.didTimeout为true
  while ((deadline.timeRemaining() > 0 || deadline.didTimeout) &&
         tasks.length > 0) {
       doWorkIfNeeded();
    }
  if (tasks.length > 0) {
    requestIdleCallback(myNonEssentialWork);
  }
}
```

#### commit 阶段

第2阶段直接一口气做完：

1. 处理effect list（包括3种处理：更新DOM树、调用组件生命周期函数（指render之后的commit阶段的生命周期函数）以及更新ref等内部状态）
2. 出对结束，第2阶段结束，所有更新都commit到DOM树上了

注意，真的是*一口气做完*（同步执行，不能喊停）的，这个阶段的实际工作量是比较大的，所以尽量不要在后3个生命周期函数里干重活儿

### 四、总结

#### 已知

React在一些响应体验要求较高的场景不适用，比如动画，布局和手势

根本原因是渲染/更新过程一旦开始无法中断，持续占用主线程，主线程忙于执行JS，无暇他顾（布局、动画），造成掉帧、延迟响应（甚至无响应）等不佳体验

#### 求

一种能够彻底解决主线程长时间占用问题的机制，不仅能够应对眼前的问题，还要有长远意义

> The “fiber” reconciler is a new effort aiming to resolve the problems inherent in the stack reconciler and fix a few long-standing issues.

#### 解

把渲染/更新过程拆分为小块任务，通过合理的调度机制来控制时间（更细粒度、更强的控制力）

那么，面临5个子问题：

#### 1.拆什么？什么不能拆？——两个阶段：diff + patch

把渲染/更新过程分为2个阶段（diff + patch）：

```
1.diff ~ render/reconciliation
2.patch ~ commit
```

diff的实际工作是对比`prevInstance`和`nextInstance`的状态，找出差异及其对应的DOM change。diff本质上是一些计算（遍历、比较），是可拆分的（算一半待会儿接着算）

patch阶段把本次更新中的所有DOM change应用到DOM树，是一连串的DOM操作。这些DOM操作虽然看起来也可以拆分（按照change list一段一段做），但这样做一方面可能造成DOM实际状态与维护的内部状态不一致，另外还会影响体验。而且，一般场景下，DOM更新的耗时比起diff及生命周期函数耗时不算什么，拆分的意义不很大

所以，render/reconciliation阶段的工作（diff）可以拆分，commit阶段的工作（patch）不可拆分

P.S.diff与reconciliation只是对应关系，并不等价，如果非要区分的话，reconciliation包括diff：

> This is a part of the process that React calls reconciliation which starts when you call ReactDOM.render() or setState(). By the end of the reconciliation, React knows the result DOM tree, and a renderer like react-dom or react-native applies the minimal set of changes necessary to update the DOM nodes (or the platform-specific views in case of React Native).

（引自[Top-Down Reconciliation](https://reactjs.org/blog/2015/12/18/react-components-elements-and-instances.html#top-down-reconciliation)）

#### 2.怎么拆？—— 以 fiber 节点作为拆分单位

先凭空乱来几种diff工作拆分方案：

- 按组件结构拆。不好分，无法预估各组件更新的工作量
- 按实际工序拆。比如分为`getNextState(), shouldUpdate(), updateState(), checkChildren()`再穿插一些生命周期函数

按组件拆太粗，显然对大组件不太公平。按工序拆太细，任务太多，频繁调度不划算。那么有没有合适的拆分单位？

有。Fiber的拆分单位是fiber（fiber tree上的一个节点），实际上就是*按虚拟DOM节点拆*，因为fiber tree是根据vDOM tree构造出来的，树结构一模一样，只是节点携带的信息有差异

所以，实际上是vDOM node粒度的拆分（以fiber为工作单元），每个组件实例和每个DOM节点抽象表示的实例都是一个工作单元。工作循环中，每次处理一个fiber，处理完可以中断/挂起整个工作循环

#### 3.如何调度任务？——工作循环+优先级

分2部分：

- 工作循环
- 优先级机制

工作循环是*基本的任务调度机制*，工作循环中每次处理一个任务（工作单元），处理完毕有一次喘息的机会：

```javascript
// Flush asynchronous work until the deadline runs out of time.
while (nextUnitOfWork !== null && !shouldYield()) {
  nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
}
```

`shouldYield`就是看时间用完了没（`idleDeadline.timeRemaining()`），没用完的话继续处理下一个任务，用完了就结束，把时间控制权还给主线程，等下一次`requestIdleCallback`回调再接着做：

```javascript
// If there's work left over, schedule a new callback.
if (nextFlushedExpirationTime !== NoWork) {
  scheduleCallbackWithExpiration(nextFlushedExpirationTime);
}
```

也就是说，（不考虑突发事件的）正常调度是由工作循环来完成的，基本*规则*是：每个工作单元结束检查是否还有时间做下一个，没时间了就先“挂起”

优先级机制用来处理突发事件与优化次序，例如：

- 到commit阶段了，提高优先级
- 高优任务做一半出错了，给降一下优先级
- 抽空关注一下低优任务，别给饿死了
- 如果对应DOM节点此刻不可见，给降到最低优先级

这些策略用来动态调整任务调度，是工作循环的*辅助机制*，最先做最重要的事情

#### 4.如何中断/断点恢复？——workInProgress tree + tag

中断：检查当前正在处理的工作单元，保存当前成果（`firstEffect, lastEffect`），修改tag标记一下，迅速收尾并再开一个`requestIdleCallback`，下次有机会再做

断点恢复：下次再处理到该工作单元时，看tag是被打断的任务，接着做未完成的部分或者重做

P.S.无论是时间用尽“自然”中断，还是被高优任务粗暴打断，对中断机制来说都一样

#### 5.如何收集任务结果？——effect list

Fiber reconciliation的工作循环具体如下：

1. 找到根节点优先级最高的workInProgress tree，取其待处理的节点（代表组件或DOM节点）
2. 检查当前节点是否需要更新，不需要的话，直接到4
3. 标记一下（打个tag），更新自己（组件更新`props`，`context`等，DOM节点记下DOM change），并为孩子生成workInProgress node
4. 如果没有产生子节点，归并effect list（包含DOM change）到父级
5. 把孩子或兄弟作为待处理节点，准备进入下一个工作循环。如果没有待处理节点（回到了workInProgress tree的根节点），工作循环结束

通过每个节点更新结束时*向上归并effect list*来收集任务结果，reconciliation结束后，根节点的effect list里记录了包括DOM change在内的所有side effect

#### 举一反三

既然任务可拆分（只要最终得到完整effect list就行），那就允许*并行执行*（多个Fiber reconciler + 多个worker），首屏也更容易分块加载/渲染（vDOM森林）

并行渲染的话，据说Firefox测试结果显示，130ms的页面，只需要30ms就能搞定，所以在这方面是值得期待的，而React已经做好准备了，这也就是在React Fiber上下文经常听到的待*unlock*的更多特性之一

## 参考

[完全理解 React Fiber 架构](http://www.ayqy.net/blog/dive-into-react-fiber/#articleHeader4)