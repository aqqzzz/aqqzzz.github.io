---
title: React key和diff
date: 2019-07-27 16:16:04
tags:
- React
- 面试
- 前端100问
categories:
- React
---

写React代码时，我们经常会在遍历列表节点的时候给每个列表项添加一个key属性，总是说它可以加快节点更新，那么具体是怎么更新，又是如何加快的呢？

### React 更新节点的过程

React是通过更新其内部数据，驱动视图进行相应更新的一种模式，我们只需要关注想要关注的数据并对其进行操作。

但是想要反映到真实html上的话，这一过程必定会涉及 js 操作dom进行更新，我们都知道js操作dom是很耗费资源的一件事情，而React的虚拟DOM 和 diff算法就是为了加快这一过程而提出的核心思想。

js操作dom很耗费资源，而js操作js对象是很快的，React的虚拟DOM，说白了就是用javascript构造的一种dom树结构，用js来表达最终想要渲染的dom树结构，当每次数据更新发生时，React会根据render方法的内容先生成一个js的树结构，并把这个新生成的js树跟旧的js树进行比对，记录有区别的节点，最终每次只更新这些有区别的节点

React 虚拟DOM节点其实就是一个js对象，React有一个顶层方法叫 React.createElement，我理解这个方法就是生成一个可以代表一个dom节点的js对象，我们常常在写react时会写这样的代码

```javascript
const node = (<div class="item">hhh</div>)
```

其实这并不是合格的js代码，而是一种成为 jsx 的板式代码，react会在读到这样的代码时，使用React.createElement 代替它，生成对应的js代码，相当于

```javascript
const node = {
  tag: "div",
  attrs: {
    class: "item"
  },
  children: [{
    'hhh' // 也可以是一个跟node类似的(有tag、attrs、children的对象)
  }]
}
```

所以，虚拟DOM树最终就是这样的一个很大的js对象

但是，就算是一个简单的js对象表示的树结构，如果想比较两棵树是否完全相同的话，可以进行深度优先或广度优先遍历，不管是哪种，如果树结构很复杂或者树很深的话，即使是js对象，也会占用很长时间来进行比对

React又是如何快速比对两个很大的js对象是否相同的呢

### React diff

不能完全说React是在比对两个js对象是否完全相同，React想要比对的是两个代表dom树的js对象，而这样的对象又有一些特点，所以react 的diff算法是基于以下几个假设的

1. Web UI中跨层级的节点操作很少，可以忽略
2. 拥有相同类的两个组件会生成类似的树结构，而拥有不同类的两个组件会生成不同的树结构
3. 对于同一层级的子节点，可以通过id进行唯一区分

基于这三个策略，react分别对 tree diff、component diff 和 element diff进行了算法优化

#### tree diff

忽略跨层级的节点移动，只比较同层节点，即，对树进行分层比较。

![](http://ww1.sinaimg.cn/large/8ac7964fly1g5ej4cz4ntj20g608c0tp.jpg)

React 只对相同颜色的node节点进行比较，如果在某一层，发现新旧dom树的某个节点不一样了，那么不管它们的子树是否相同，react都会对其进行替换（或删除，这里的具体策略下文再讲）

那么，如果dom树真的进行了跨层级的移动，React会怎样比对节点树呢？

![](http://ww1.sinaimg.cn/large/8ac7964fly1g5ej84sbtsj20es08laao.jpg)

如图，旧的DOM树中A子树是R的左节点，而在新的DOM树中这个子树被移动到了D子树的下一层级。

这种情况，React会在比对R的左节点时，发现新的DOM树中没有这个节点，React会把这个子树整个删除，再按顺序进行D的比对，D比对完成后发现新的DOM树还有下一层，那么再进行下一层遍历，由于旧子树中D没有左节点，React会直接创建一个新的A节点并添加到D的左子树上去

【建议】在React开发中，保持稳定的DOM树结构有利于性能的提升

#### component diff

React是基于组件构建应用的，对于组件的比较

- 对于同类型的组件，使用原策略进行比对
- 对于不同类型的组件，不管新旧组件的子树结构是否相同，都会直接删除旧的节点树，并创建新的节点树（替换其所有子节点）
- 对于同类型的组件，可能virtual dom 没有任何变化，如果能够确切的知道这点那可以节省大量的 diff 运算时间，因此 React 允许用户通过shouldComponentUpdate() 来判断该组件是否需要进行 diff。

#### element diff 和 key

同一层级节点的比较有 新增、移动和删除 三种操作

没有用key的时候，React只会简单的比较同一个未知的元素是否相同

![](http://ww1.sinaimg.cn/large/8ac7964fly1g5ejmisds7j20e507f3zg.jpg)

对于这个过程，虽然只是相邻元素交换位置，但是React只会比对相同位置的元素，它会认为每个元素都是不同的，所以每次比对都会删除旧节点，创建新节点，这个过程很浪费，所以React提供了key的策略来让我们手动标识，这两个元素是相同的

![](http://ww1.sinaimg.cn/large/8ac7964fly1g5ejop7yclj20ef08i3zp.jpg)

首先对新集合的节点进行循环遍历，for (name in nextChildren)，通过唯一 key 可以判断新老集合中是否存在相同的节点，if (prevChild === nextChild)，如果存在相同节点，则进行移动操作，但在移动前需要将当前节点在老集合中的位置与 lastIndex 进行比较，if (child._mountIndex < lastIndex)，则进行节点移动操作，否则不执行该操作。这是一种<font color="red">顺序优化</font>手段，lastIndex 一直在更新，表示访问过的节点在老集合中最右的位置（即最大的位置），**如果新集合中当前访问的节点比 lastIndex 大，说明当前访问节点在老集合中就比上一个节点位置靠后，则该节点不会影响其他节点的位置，因此不用添加到差异队列中，即不执行移动操作，只有当访问的节点比 lastIndex 小时，才需要进行移动操作**。

以上图为例，可以更为清晰直观的描述 diff 的差异对比过程：

- 从新集合中取得 B，判断老集合中存在相同节点 B，通过对比节点位置判断是否进行移动操作，B 在老集合中的位置 B._mountIndex = 1，此时 lastIndex = 0，不满足 child._mountIndex < lastIndex 的条件，因此不对 B 进行移动操作；更新 lastIndex = Math.max(prevChild._mountIndex, lastIndex)，其中 prevChild._mountIndex 表示 B 在老集合中的位置，则 lastIndex ＝ 1，并将 B 的位置更新为新集合中的位置prevChild._mountIndex = nextIndex，此时新集合中 B._mountIndex = 0，nextIndex++ 进入下一个节点的判断。
- 从新集合中取得 A，判断老集合中存在相同节点 A，通过对比节点位置判断是否进行移动操作，A 在老集合中的位置 A._mountIndex = 0，此时 lastIndex = 1，满足 child._mountIndex < lastIndex的条件，因此对 A 进行移动操作enqueueMove(this, child._mountIndex, toIndex)，其中 toIndex 其实就是 nextIndex，表示 A 需要移动到的位置；更新 lastIndex = Math.max(prevChild._mountIndex, lastIndex)，则 lastIndex ＝ 1，并将 A 的位置更新为新集合中的位置 prevChild._mountIndex = nextIndex，此时新集合中A._mountIndex = 1，nextIndex++ 进入下一个节点的判断。
- 从新集合中取得 D，判断老集合中存在相同节点 D，通过对比节点位置判断是否进行移动操作，D 在老集合中的位置 D._mountIndex = 3，此时 lastIndex = 1，不满足 child._mountIndex < lastIndex的条件，因此不对 D 进行移动操作；更新 lastIndex = Math.max(prevChild._mountIndex, lastIndex)，则 lastIndex ＝ 3，并将 D 的位置更新为新集合中的位置 prevChild._mountIndex = nextIndex，此时新集合中D._mountIndex = 2，nextIndex++ 进入下一个节点的判断。
- 从新集合中取得 C，判断老集合中存在相同节点 C，通过对比节点位置判断是否进行移动操作，C 在老集合中的位置 C._mountIndex = 2，此时 lastIndex = 3，满足 child._mountIndex < lastIndex 的条件，因此对 C 进行移动操作 enqueueMove(this, child._mountIndex, toIndex)；更新 lastIndex = Math.max(prevChild._mountIndex, lastIndex)，则 lastIndex ＝ 3，并将 C 的位置更新为新集合中的位置 prevChild._mountIndex = nextIndex，此时新集合中 C._mountIndex = 3，nextIndex++ 进入下一个节点的判断，由于 C 已经是最后一个节点，因此 diff 到此完成

···

```javascript
_updateChildren: function(nextNestedChildrenElements, transaction, context) {
  var prevChildren = this._renderedChildren;
  var nextChildren = this._reconcilerUpdateChildren(
    prevChildren, nextNestedChildrenElements, transaction, context
  );
  if (!nextChildren && !prevChildren) {
    return;
  }
  var name;
  var lastIndex = 0;
  var nextIndex = 0;
  for (name in nextChildren) {
    if (!nextChildren.hasOwnProperty(name)) {
      continue;
    }
    var prevChild = prevChildren && prevChildren[name];
    var nextChild = nextChildren[name];
    if (prevChild === nextChild) {
      // 移动节点
      this.moveChild(prevChild, nextIndex, lastIndex);
      lastIndex = Math.max(prevChild._mountIndex, lastIndex);
      prevChild._mountIndex = nextIndex;
    } else {
      if (prevChild) {
        lastIndex = Math.max(prevChild._mountIndex, lastIndex);
        // 删除节点
        this._unmountChild(prevChild);
      }
      // 初始化并创建节点
      this._mountChildAtIndex(
        nextChild, nextIndex, transaction, context
      );
    }
    nextIndex++;
  }
  for (name in prevChildren) {
    if (prevChildren.hasOwnProperty(name) &&
        !(nextChildren && nextChildren.hasOwnProperty(name))) {
      this._unmountChild(prevChildren[name]);
    }
  }
  this._renderedChildren = nextChildren;
},
// 移动节点
moveChild: function(child, toIndex, lastIndex) {
  if (child._mountIndex < lastIndex) {
    this.prepareToManageChildren();
    enqueueMove(this, child._mountIndex, toIndex);
  }
},
// 创建节点
createChild: function(child, mountImage) {
  this.prepareToManageChildren();
  enqueueInsertMarkup(this, mountImage, child._mountIndex);
},
// 删除节点
removeChild: function(child) {
  this.prepareToManageChildren();
  enqueueRemove(this, child._mountIndex);
},

_unmountChild: function(child) {
  this.removeChild(child);
  child._mountIndex = null;
},

_mountChildAtIndex: function(
  child,
  index,
  transaction,
  context) {
  var mountImage = ReactReconciler.mountComponent(
    child,
    transaction,
    this,
    this._nativeContainerInfo,
    context
  );
  child._mountIndex = index;
  this.createChild(child, mountImage);
},
```

这种比对策略其实还是有一些缺陷的，比如，如果我们只是把列表的最后一个元素移动到列表开头，那么React在比较时，

![](http://ww1.sinaimg.cn/large/8ac7964fly1g5ejtpvr2xj20f208gdh6.jpg)

理论上 diff 应该只需对 D 执行移动操作，然而由于 D 在老集合的位置是最大的，导致其他节点的 _mountIndex < lastIndex，造成 D 没有执行移动操作，而是 A、B、C 全部移动到 D 节点后面的现象。

【建议】开发过程中，尽量减少类似将一个节点从最后移动到最开头的操作，当节点数量过大或更新操作过于频繁时，在一定程度上会影响 React 的渲染性能。

### 总结

| 假设策略                                       | 优化策略                                                     |
| ---------------------------------------------- | ------------------------------------------------------------ |
| dom操作中跨层级的节点操作可以忽略              | tree diff：对树进行分层比对                                  |
| 相同类生成类似的树结构，不同类生成不同的树结构 | Component diff：<br />相同类根据原比对规则（或shouldComponent可直接省略diff过程）<br />不同类则直接认为是不同的子节点，删除整个旧子树并创建新子树 |
| 同一层级的节点可以根据id进行区分               | element diff:<br/>提供了key，可以让react在旧子树中更快地找到可以被复用的节点（是一个mapping的过程） |

建议：

- 保持稳定的dom树结构
- 在开发过程中，尽量减少类似将最后一个节点移动到列表首部的操作，当节点数量过大或更新操作过于频繁时，在一定程度上会影响 React 的渲染性能。



### 参考：

https://github.com/Advanced-Frontend/Daily-Interview-Question/issues/1

https://zhuanlan.zhihu.com/p/20346379