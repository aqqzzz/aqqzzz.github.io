---
title: javascript之原型和原型链
date: 2019-08-14 22:02:18
tags:
- javascript
- 原型
categories：
- javascript
---

```javascript
function Person() {
  
}
var person = new Person();
person.name = 'huachenyu'
```

Person 就是一个构造函数，new 来创建一个实例对象 person

### Prototype

prototype是**只有 函数 **才会有的一种属性

```javascript
function Person() {
  
}
Person.prototype.name = 'huachenyu';
var person1 = new Person();
var person2 = new Person();
```

函数的 prototype 属性指向一个对象，这个对象就是 调用这个构造函数创建的**实例的原型**，也就是 person1 和 person2 的原型

![](http://ww1.sinaimg.cn/large/8ac7964fly1g5zkh6y5g3j20te0aidgo.jpg)

### \_\_proto\_\_

**每个对象**都有一个 `__proto__` 属性，指向它的原型，也就是其构造函数的prototype属性

![](http://ww1.sinaimg.cn/large/8ac7964fly1g5zkjwxapuj20qk0ecjsn.jpg)

### constructor

每个原型对象都有一个指向其构造函数的 constructor 属性

```javascript
Person.prototype.constructor === Person
```

![](http://ww1.sinaimg.cn/large/8ac7964fly1g5zknaa2xnj20q60c4ta4.jpg)

构造函数、实例 和 原型对象的关系：

```javascript
Person.prototype === person.__proto__
Person.prototype.constructor === Person
Object.getPrototypeOf(person) = Person.prototype // 可以获得对象的原型
```



### 原型链

javascript 中继承的原理就是原型链

````javascript
function Person () {
}
Person.prototype.name = 'huachenyu';
var person = new Person();
person.name = 'et';
console.log(person.name); // et
delete person.name;
console.log(person.name); // huachenyu
````

当读取实例的属性时，javascript 首先会在实例上查找有没有这个属性，如果在实例身上没有找到，那么就会上翻一级，在 `person.__proto__` 上查找是否有对应的属性，上面的例子就是这样的一种情况

但是如果在实例的原型对象上还是没有找到对应的属性呢？这就涉及到了原型链的知识，如果在实例的原型对象上还是没有找到对应的属性，javascript 就会顺着原型链继续查找

原型链，说白了就是

```javascript
instance.__proto__.__proto__.xxxxx = Constructor.prototype
```

就是这样的一个链式结构，javascript会在这个原型链的原型对象上查找对应属性，直到找到了对应属性，或到了null

这个原型链是一定会有终点的，当javascript到达我们定义的继承链的最上层时，这个原型对象就会被看做一个普通的object，它是可以通过 new Object 来创建的，也就是说

```javascript
var object = new Object();
object.name = 'huachenyu';
```

Person.prototype 其实就是这样的一个对象，所以说，`Person.prototype.__proto__`，是指向 `Object.prototype` 的，而 `Object.prototype.__proto__`，就是 null

### javascript原生对象的原型链

```javascript
Array.prototype.__proto__ === Object.prototype
String.prototype.__proto__ === Object.prototype
...
Function.prototype.__proto__ === Object.prototype

```

构造函数作为实例，其原型对象

```javascript
// 构造函数的原型对象：
function Person(){}
Person.__proto__
```

我们之前讲过，所有对象都有 `__proto__` 属性，那么 函数对象自然也是有 `__proto__` 属性的，每个函数，都可以看做是 `new Function()` 构建出来的，所以

```javascript
Person.__proto__ === Function.prototype // true
Function.prototype.__proto__ = Object.prototype
// Function 可以看做是 new Object
```

一个比较特殊的例子

```javascript
Function.__proto__ === Function.prototype
Object.__proto__ === Function.prototype
```

可以理解为：就是先有的Function，然后实现上把原型指向了Function.prototype，但是我们不能倒过来推测因为Function.__proto__ === Function.prototype，所以Function调用了自己生成了自己。一种内建规则，不必深究

Object.prototype 并不是一个正统的对象，因为它的`__proto__` 为 null，可以说，对象都是实例，而实例不一定都是对象

【题外话】Function对象，所有的函数都是继承自 Function 对象，我们平时使用的 `bind apply call` 这些函数，都是在Function 的 prototype 上定义的属性—— [MDN-Function](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Function)

### 总结

![](http://ww1.sinaimg.cn/large/8ac7964fly1g5zm85ux1wj20u20qi41d.jpg)

![](http://ww1.sinaimg.cn/large/8ac7964fly1g5zm949p74j21440n4dke.jpg)

### 参考

[Javascript 深入系列之原型与原型链](https://github.com/mqyqingfeng/Blog/issues/2)

[理解 prototype、proto、constructor之间的关系](https://alexzhong22c.github.io/2017/08/08/js-proto/)