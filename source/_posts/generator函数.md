---
title: generator函数
date: 2019-08-31 19:18:32
tags:
- javascript
- es6
categories:
- javascript
---

前几天学习 for...of 和 iterator 对象的时候接触到了 generator 函数，generator函数返回的是一个 iterator 对象，而 通过`[Symbol.iterator]` 接口返回的 iterator 对象，就是 for...of 遍历时的规则，所以，今天来学习一下这个 generator 函数

![image.png](https://i.loli.net/2019/08/31/5OFK1xebyfQGgMz.png)

## 复习：iterator对象

迭代器对象，结构为：

```js
{
   next() { 
     // 每次调用 iter.next() 时都要返回这样的对象
     // Required
     return { value: any, done: Boolean }
   },
   return() {
     // 当在循环中 break 或 throw error 的时候回默认调用 iter.return
     // not required
     // 一旦定义，返回必须为一个对象
     // 未定义时则默认返回 { done: true }
     return { done: true, value: any }
   },
   throw(value) {
     // 通常在 generator 函数中使用，return值为真正调用 throw 时的返回值
     // not required
   }
}
```

如果想要让某个对象能够使用 `for...of` 进行遍历

```js
var obj = {
  [Symbol.iterator]() {
    return {
      next() {...}
    }
  }
}
```

## generator 函数

generator 函数返回的就是一个 iterator 对象，也就是类似于

```js
{
  next: function,
  return: function,
  throw: function,
}
```

的一个对象

### 基本语法

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

#### yield*

指后面跟着的也是一个 generator 函数的执行结果（即一个 iterator 对象），这一步的执行相当于 把后面的generator函数平铺开来，对其进行遍历

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

#### next/return/throw

`next()`、`throw()`、`return()`这三个方法本质上是同一件事，可以放在一起理解。它们的作用都是让 Generator 函数恢复执行，并且使用不同的语句替换`yield`表达式。

`next()`是将`yield`表达式替换成一个值。

```javascript
const g = function* (x, y) {
  let result = yield x + y;
  return result;
};

const gen = g(1, 2);
gen.next(); // Object {value: 3, done: false}

gen.next(1); // Object {value: 1, done: true}
// 相当于将 let result = yield x + y
// 替换成 let result = 1;
```

上面代码中，第二个`next(1)`方法就相当于将`yield`表达式替换成一个值`1`。如果`next`方法没有参数，就相当于替换成`undefined`。

`throw()`是将`yield`表达式替换成一个`throw`语句。

```javascript
gen.throw(new Error('出错了')); // Uncaught Error: 出错了
// 相当于将 let result = yield x + y
// 替换成 let result = throw(new Error('出错了'));
```

`return()`是将`yield`表达式替换成一个`return`语句。

```javascript
gen.return(2); // Object {value: 2, done: true}
// 相当于将 let result = yield x + y
// 替换成 let result = return 2;
```

#### generator 函数的返回值

generator 函数的返回值是一个 指针对象（也就是一个 iterator 对象）

### generator 中的this

普通 generator 函数中的 this 是不生效的，即使使用 new 来创建，generator 返回的也是 iterator 对象，每次调用 iter.next() 时，generator函数中的this并不是指向 iter，而是指向其上级作用域链中的 this 对象

```js
var obj = {
    gen: function* () {
        this.a = 1
    },
    getA: function() {
        return this.a;
    },
    a: 0,
}

var iter = obj.gen();
iter.next();
console.log(obj.getA());
```

上面代码中，虽然执行 next 的对象是 iter，但是执行时其中的 this 指向的是 obj，也就是其上级作用域链中的 this

而且不能执行 new

```js
function* F() {
  yield this.x = 2;
  yield this.y = 3;
}

new F()
// TypeError: F is not a constructor
```

### generator 异步

```js
var fetch = require('node-fetch');

function* gen(){
  var url = 'https://api.github.com/users/github';
  var result = yield fetch(url);
  console.log(result.bio);
}

var g = gen();
var result = g.next(); // 返回值为 yield fetch(url) 的返回值 { done: false, value: Promise(fetchResult) }

result.value.then(function(data){
  return data.json();
}).then(function(data){
  g.next(data);
});
```

上面代码中，首先执行 Generator 函数，获取遍历器对象，然后使用`next`方法（第二行），执行异步任务的第一阶段。由于`Fetch`模块返回的是一个 Promise 对象，因此要用`then`方法调用下一个`next`方法。

那么，如何让函数自执行，也就是自己在上一次yield返回数据之后，调用 next 从而进行接下来代码的执行呢

> 既然 generator 函数的初衷是为了把执行权暂时出让，这里又为什么要探究自执行的机制呢？
>
> 在这里，generator 函数更大的作用是为了在 generator 函数内部，可以用 同步的方式来写异步的代码。
>
> 因为generator 函数能够出让执行权的时机都是每次 yield 执行结束，返回数据的时间点，而在yield 后面的函数执行过程中是没有办法被打断的，我们让 generator 函数自执行，那么在这个 generator 函数内部，我们可以保证，每次 yield 后的函数不会被打断，而且在这个函数执行的过程中这个generator函数会处于等待状态，而且 yield 一旦返回数据，函数就可以立即执行接下来的同步代码
>
> 协程的概念，在异步的时候把执行权交出给这个异步函数，异步函数一旦执行完毕，就把执行权收回来继续执行同步代码

#### 自执行函数的实现——thunk 函数

##### 什么是 thunk 函数

如果某个函数调用中包括一个回调函数，也就是说，在处理完这个函数的主流程之后，将调用这个回调函数进行一些本函数之外数据或状态的变更，那么这样的函数可以被包装成一个 thunk 函数

```js
function thunk(fn) {
  // fn 为 被包装的函数
  return function(...args) {
    // ...args 为调用fn时除了 callback 之外的所有参数
    return function(callback) {
      // 传入 callback
      fn.call(this, ...args, callback)
    }
  }
}
```

> call 和 apply 的区别，apply 的调用参数为数组，而 call 则直接传入即可

thunk 函数的作用，就是延迟 callback 的传入时机

##### generator 与 thunk 函数

那么，generator 函数又与 thunk 函数有什么关系呢？——使用 Thunk 函数，可以自执行 generator 函数

```js
function run(g) {
  var gen = g();
  
  function next(err, data) {
    const result = gen.next(data);
    if (result.done) return;
    result.value(next); 
    // result.value 应该是一个可以被包装的thunk 函数，也就是说， generator 函数中每个 yield 后面，都必须是一个 可以被执行的 thunk 函数
  }
  next()
}
```

调用：

```js
var g = function* (){
  // 这里的每个 readFileThunk，都是一个thunk函数
  // 执行 readFileThunk(string) 的返回值，是接受一个callback作为参数的函数
  var f1 = yield readFileThunk('fileA'); 
  var f2 = yield readFileThunk('fileB');
  // ...
  var fn = yield readFileThunk('fileN');
};

run(g);
```

#### 自执行函数的实现——Promise

使用Promise 实现，则需要 generator 函数中每个yield 后面跟着的都必须是一个 Promise 对象

```js
function run(g) {
  var gen = g();
  
  function next(data) {
    var result = gen.next();
    if (result.done) return;
    // 使用Promise 实现，则需要 generator 函数中每个yield 后面跟着的都必须是一个 Promise 对象
    result.value.then((data) => {
      next(data);
    })
  }
  
  next();
}
```





