---
title: for...of 与 iterator对象
date: 2019-08-28 22:56:18
tags:
- javascript
categories:
- javascript
---

昨天下班回来太晚了，没来得及看代码，今天回来参加了阿里的笔试，有个 iterator 对象定义 for...of 的笔试题，自己写的时候只是模糊的有个印象，所以先把这部分学习一下

（dbq react-redux真的太难看了，明天我可能还得再看看，然鹅估计也只能看得懂一个大致逻辑）

### 遍历数组/对象

说起遍历，在js中有几种方式

1. for循环：正常使用，只是语法比较繁琐
2. forEach：Array内置，语法简洁，但是无法中途 return 或 break
3. for...in：
   - 遍历 obj 的键值，也就是说，对于对象而言，遍历对象key，对于数组而言，for...in 遍历的是数组下标
   - 遍历过程中，会遍历到原型链上的属性
   - 对于数组而言，如果在数组实例上添加了某个属性（虽然这个操作很sb），for...in 也会遍历到这个属性
4. for...of
   - 所有实现了 iterator 接口的对象都可以用 for...of 遍历，具体实现方式下文介绍
   - 原生实现了 iterator 的javascript对象有：
     - Array——value
     - Map——[key, value]
     - Set——value
     - String——char
     - TypedArray
     - 函数 arguments 对象——单个 arguments对象的值
     - NodeList（NodeList 和 普通的 Array 对象是不同的，NodeList不能使用 数组的方法，只能进行遍历和 length操作）——nodeItem的值
   - 只会按照 Symbol.iterator 中定义的规则进行遍历，对于用户添加的属性，或者 原型链上的属性，是不会遍历的
   - 可以中途 return 或 break

### iterator 接口

既然 for...of 这么好，那么它内部又是怎么实现的呢，说起 for...of 的内部实现，就绕不过 iterator 接口

Iterator 接口，就是为不同的数据结构提供统一的访问机制，任何数据结构只要部署了 Iterator 接口，就可以实现 遍历操作

Iterator 的作用：

1. 为遍历操作提供 **统一接口**
2. 使得数据结构的成员能够按照某种 **次序排列**
3. 为 for...of 提供 iterator 对象
4. 扩展运算符 `...` 默认调用 iterator 接口 // TODO

Iterator 对象本质上是一个 指针对象，通过不断调用自身对象的 next 方法更新指针指向，并返回该指向成员的信息。一个Iterator 的内部构成如下：

```js
function makeIterator() {
	let current = 0; // 指针对象，内部属性，外部无法访问
  return {
    next: function() { // required
      ...
      return { 
        // 返回值的格式必须是 { done：是否遍历完成, value: 当前遍历结果 }
        done: false,
        value: nextValue || value, 
      }
    },
    return() { // optional
      // 当迭代器中途 return 时的处理逻辑
      // 1. break
      // 2. throw error
      return { done: true } // 返回值必须为一个对象
    },
    throw() {
      // 主要配合 generator 函数使用
    }
  }
}
```

遍历器对象就是通过不断调用 next 方法来更新获取下一个值的，

return 则是在函数中途 break 或 throw error 时关闭迭代器对象

来看一下对于原生可以 for...of 的对象的 iterator 对象是什么样子的

```js
let arr = ['a', 'b', 'c'];
let iter = arr[Symbol.iterator]();

iter.next() // { value: 'a', done: false }
iter.next() // { value: 'b', done: false }
iter.next() // { value: 'c', done: false }
iter.next() // { value: undefined, done: true }
```

可以看到，直接访问实例上的 [Symbol.iterator] 属性，就可以得到这个迭代器对象

### for...of

那么 for...of 又是如何把 iterator 接口嵌入到 对象上的呢

for...of 会访问 Object上的`[Symbol.iterator]` 对象，从而获取 该Object 的 迭代器对象

接下来我们来简单实现一个 Class 的 for...of 

```js
class People {
  constructor(obj) {
    this.state = {
      obj,
      current: 0,
    }
  }
  // 或者 [Symbol.iterator]: function() {}
  [Symbol.iterator]() {
    return this;
    // { next: function }
  }
  next() {
    const { obj, current } = this.state;
    const keys = Object.keys(obj);
    if (current <= keys.length - 1) {
      this.state.current = current + 1;
      return {
        done: false,
        value: [keys[current], obj[keys[current]]]
      }
    } else {
      return {
        done: true, // 代表循环结束
        value: undefined,
      }
    }
  }
}
```

[Symbol.iterator] 写在 class 里，相当于是把它加到了 实例对象的 原型对象上，是一种比较符合逻辑的做法，这样就相当于多个实例对象用的是一个 内存地址里的函数

另一种更简单的实现方式是 使用 generator 函数

### generator 函数

#### 基本语法

```js
function *gen() {
  yield 'abfc';
  yield* gen();
}
```

在函数 function 之后使用 `*` 声明这是一个 generator 函数，其实就是一种可以暂停执行的函数，函数内部的 yield 就是暂停点，每次都等到 yield 后面的部分返回值之后，函数才会继续向下执行

Generator 函数返回的就是一个 遍历器对象，它的暂停执行，也是通过 遍历对象的next() 实现的，遍历器对象的`next`方法的运行逻辑如下。

（1）遇到`yield`表达式，就暂停执行后面的操作，并将紧跟在`yield`后面的那个表达式的值，作为返回的对象的`value`属性值。

（2）下一次调用`next`方法时，再继续往下执行，直到遇到下一个`yield`表达式。

（3）如果没有再遇到新的`yield`表达式，就一直运行到函数结束，直到`return`语句为止，并将`return`语句后面的表达式的值，作为返回的对象的`value`属性值。

（4）如果该函数没有`return`语句，则返回的对象的`value`属性值为`undefined`。

yield 后面的代码，只有在调用 next ，执行到这一行的时候，才会开始执行（手动的惰性求值）

##### yield* 

语法，是指后面跟着的是一个 Generator 函数，这一步的执行相当于 把后面的generator函数平铺开来，对其进行遍历

```js
function* concat(iter1, iter2) {
  yield* iter1;
  yield* iter2;
}

// 相当于

function* concat(iter1, iter2) {
  for (var value of iter1) {
    yield value;
  }
  for (var value of iter2) {
    yield value;
  }
}
```

##### next 的传参

```js
function* foo(x) {
  var y = 2 * (yield (x + 1));
  var z = yield (y / 3);
  return (x + y + z);
}

var a = foo(5);
a.next() // Object{value:6, done:false}
a.next() // Object{value:NaN, done:false}
a.next() // Object{value:NaN, done:true}

var b = foo(5);
b.next() // { value:6, done:false }
b.next(12) // { value:8, done:false }
b.next(13) // { value:42, done:true }
```

yield 表达式是没有返回值的，如上例中的 a， yield (x + 1) 的返回值为 undefined，所以第二次 next 的计算value 为 NaN

但是我们可以通过向next 函数传参，将其作为 yield 表达式的返回值，如上例中的 b，在第二次b.next(12) 的调用过程中，将 12  传入作为 `yield (x + 1) ` 的返回值，也就是说，此时 y 为 24，而 24 / 3 为8，所以第二次返回值中的 value 为 8 。其余同理

注意点：

1. yield 是惰性计算的，执行一次 next ，执行一段代码，比如例子中 首次 next，其实只执行了 yield (x + 1)，在第二次 next 的时候才进行 `y = 2 * yield ; yield (y / 3)`  的计算
2. next 中的参数直接作为 上次执行 yield 的整个表达式的值 ，也就是对于第二次执行b.next(12) 而言，12替代的是 `yield (x + 1)` 这个部分的值

#### 和 iterator 的关系

上文说过，generator 函数生成的是 iterator 对象（next），所以 for...of 的时候我们可以直接遍历这个对象，那么for...of 的 generator 实现

```js
class Person1 {
    constructor(obj) {
        this.obj = obj;
        this.keys = Object.keys(obj);
    }
    *[Symbol.iterator]() {
        const { keys, obj } = this;
        for (let key of keys) {
            yield [key, obj[key]];
        }
    }
}
```

其实任何想要使用 for...of 的地方，都可以使用 `[Symbol.iterator]` 和 `generator 函数` 来实现，更加简洁

### 总结

1. for...in 遍历对象，for...of 遍历任何实现了 iterator 接口的对象
2. iterator 对象提供 `{ next, return, throw }` 三个函数，只有 next 必须，for...of 的暂停点就在 执行 next 的时候
3. iterator 对象的 next函数返回值为 `{ done: boolean, value: any }`， done代表循环是否结束，返回true时结束
4. generator 函数返回的是一个 iterator 对象，所以天然可以支持 for...of 
5. ...扩展运算符也是调用 iterator 接口的 // TODO



// TODO

1. 扩展运算符
2. generator 函数





