---
title: es6之类与类继承
date: 2019-08-17 20:06:04
tags:
- javascript
- es6
categories:
- es6
---

趁着之前刚刚学过es5 的继承，今天来学习一下 es6 的类 以及 类继承

es6 的类其实只是一个语法糖

```javascript
class Parent_class {
  constructor(name) {
    this.name = name;
  }
}

function Parent_func(name) {
  this.name = name;
}
```

它与function最大的区别在于，class 只能通过 new 来创建，而function 可以new 也可以直接调用

es6 中 类本身，对应function.prototype，而类的constructor，则对应于function的构造函数（也就是function本身

接下来，我们会从 constructor、实例属性、静态方法、静态属性、getter 和 setter、new 几个方面来对 es6 的类进行解析，用es5的代码来实现对应的功能，并使用babel进行编译，从编译结果中查看babel的转换方式，并最后查看es6 类继承的实现

![](http://ww1.sinaimg.cn/large/8ac7964fly1g62yju6x7vj218e0ietcf.jpg)

先来看一些 babel 转义中的工具函数

```javascript
// instanceof 的polyfill（Symbol？）
function _instanceof(left, right) { 
  if (right != null && typeof Symbol !== "undefined" && right[Symbol.hasInstance]) { 
    return !!right[Symbol.hasInstance](left); 
  } else { 
    return left instanceof right; 
  } 
}
// 检查 Constructor 是否以 new 的方式调用
function _classCallCheck(instance, Constructor) { 
  if (!_instanceof(instance, Constructor)) { 
    throw new TypeError("Cannot call a class as a function"); 
  } 
}
// 在target对象上定义props
// 需要注意的是，class 中除实例属性之外所有的函数都是不可枚举的（Object.keys不能遍历）
function _defineProperties(target, props) { 
  for (var i = 0; i < props.length; i++) { 
    var descriptor = props[i];
    descriptor.enumerable = descriptor.enumerable || false; // enumerable 属性默认为false
    descriptor.configurable = true; 
    if ("value" in descriptor) 
      descriptor.writable = true; 
    Object.defineProperty(target, descriptor.key, descriptor); 
  } 
}
// 为Constructor添加proto属性，和static属性
function _createClass(Constructor, protoProps, staticProps) { 
  if (protoProps) // proto属性添加到 Constructor.prototype上
    _defineProperties(Constructor.prototype, protoProps); 
  if (staticProps) // static 属性添加到Constructor上
    _defineProperties(Constructor, staticProps); 
  return Constructor; 
}
// defineProperty 的polyfill
function _defineProperty(obj, key, value) { 
  if (key in obj) { 
    Object.defineProperty(obj, key, { 
      value: value, 
      enumerable: true, 
      configurable: true, 
      writable: true
    }); 
  } else { 
    obj[key] = value; 
  } 
  return obj; 
}
```

Object.defineProperty定义的属性，其descriptor属性如下

```javascript
// 数据描述符
Object.defineProperty(targetObject, key, {
  configurable: false,
  writable: false,
  enumerable: false,
  value: undefined;
})
// 存取描述符
Object.defineProperty(targetObject, key, {
  // 当访问targetObject 的 key 属性时，执行 get 方法
  get: undefined,
  // 当设置 targetObject 的 key 属性时，执行set方法
  set: undefined,
})
```

**数据描述符** 和 **存取描述符** 只能**二者取其一**

需要注意的是，存取描述符中的 get 和 set 都会传入一个 this 对象

### 实例属性

实例属性可以通过 constructor 和 this.xxx 两种方式来定义

Es6

```javascript
class Person {
  constructor(name) { // 构造函数
    this.name = name
  }
  state = 'state'; // 实例属性
  
  sayName() {
    console.log('my name is ' + this.name);
  }
}
```

es5

```javascript
function Person(name) {
  this.name = name;
  this.state = 'state';
}
Person.prototype.sayName = function() {
  console.log('my name is ' + this.name);
}
```

可以看出，es6中除实例属性和构造函数是挂在函数实例上之外，所有的构造函数都是直接挂到 prototype 上的

babel转义代码

```javascript
"use strict";

var Person =
/*#__PURE__*/
function () {
  function Person(name) {
    _classCallCheck(this, Person); // 是否通过new创建class

    _defineProperty(this, "state", 'state'); // 定义实例属性

    // 构造函数
    this.name = name;
  }

  _createClass(Person, [{
    key: "sayName",
    // proto属性
    value: function sayName() {
      console.log('my name is ' + this.name);
    }
  }]);

  return Person;
}();
```

我们可以看到 Babel 生成了一个 _createClass 辅助函数，该函数传入三个参数，第一个是构造函数，在这个例子中也就是 Person，第二个是要添加到原型上的函数数组，第三个是要添加到构造函数本身的函数数组，也就是所有添加 static 关键字的函数。该函数的作用就是将函数数组中的方法添加到构造函数或者构造函数的原型中，最后返回这个构造函数。

在其中，又生成了一个 defineProperties 辅助函数，使用 Object.defineProperty 方法添加属性。

默认 enumerable 为 false，configurable 为 true，这个在上面也有强调过，是为了防止 Object.keys() 之类的方法遍历到。然后通过判断 value 是否存在，来判断是否是 getter 和 setter。如果存在 value，就为 descriptor 添加 value 和 writable 属性，如果不存在，就直接使用 get 和 set 属性。

### static

es6代码

```javascript
class Person {
  static name = 'huachenyu';
	static clacName() {
    return 'hello';
  }
}
```

es5

```javascript
function Person(){}
Person.name = 'huachenyu';
Person.calcName = function() {
  return 'hello';
}
```

Babel 编译代码

```javascript
var Person =
/*#__PURE__*/
function () {
  function Person() {
    _classCallCheck(this, Person);
  }

  _createClass(Person, null, [{
    // static 属性
    key: "clacName",
    value: function clacName() {
      return 'hello';
    }
  }]);

  return Person;
}();

_defineProperty(Person, "name", 'huachenyu'); // 给Person上定义静态属性name
```

### Getter 和 setter

Es6

```javascript
class Person {
    get name() {
        return 'kevin';
    }
    set name(newName) {
        console.log('new name 为：' + newName)
    }
}

let person = new Person();

person.name = 'daisy';
// new name 为：daisy

console.log(person.name);
// kevin
```

es5

```javascript
function Person() {}
Object.defineProperty(Person.property, 'name', {
  get() {
    return 'kevin';
  },
  set(newName) { 
    console.log('new name 为：' + newName);
  }
})
```

babel

```javascript
var Person =
/*#__PURE__*/
function () {
  function Person() {
    _classCallCheck(this, Person);
  }

  _createClass(Person, [{
    // proto 属性
    key: "name",
    get: function get() {
      return 'kevin';
    },
    set: function set(newName) {
      console.log('new name 为：' + newName);
    }
  }]);

  return Person;
}();
```

### 继承

es6 extends

```javascript
class Parent {
  constructor(name) {
    this.name = name;
  }
  static classMethod() {
    return 'hello';
  }
}

class Child extends Parent {
  constructor(name, age) {
    super(name);
    this.age = age;
  }
}

var child1 = new Child('kevin', '18');

console.log(child1);
Child.classMethod(); // 'hello'
```

es6 的继承实现方式其实就是  [[javascript 之继承]([https://aqqzzz.github.io/2019/08/15/javascript之继承/#more](https://aqqzzz.github.io/2019/08/15/javascript之继承/#more)) 中讲到的**寄生组合式继承**

- super(name) 就是Parent.call(this)

  子类必须在 constructor 方法中调用 super 方法，否则新建实例时会报错。这是因为子类没有自己的 this 对象，而是继承父类的 this 对象，然后对其进行加工。如果不调用 super 方法，子类就得不到 this 对象。

  也正是因为这个原因，在子类的构造函数中，只有调用 super 之后，才可以使用 this 关键字，否则会报错。

- extends 关键字，则是

  ```javascript
  Child.prototype = Object.create(Parent.prototype, {
    constructor: {
      configurable: true,
      value: Child,
      enumerable: false,
      writable: true
    }
  })
  ```

  这里形成了一条 `Child.prototype.__proto__ = Parent.prototype` 的原型链

- Es6 中的 static 静态方法也是可以继承的，`Object.setPrototypeOf(Child, Parent)`

  `Child.__proto__ = Parent`

Class 作为构造函数的语法糖，同时有 prototype 属性和 __proto__ 属性，因此同时存在两条继承链。

（1）子类的 __proto__ 属性，表示构造函数的继承，总是指向父类。

（2）子类 prototype 属性的 __proto__ 属性，表示方法的继承，总是指向父类的 prototype 属性。

![](http://ww1.sinaimg.cn/large/8ac7964fly1g6344h97hkj20h00dtmxi.jpg)

babel 编译代码

```javascript
'use strict';

function _possibleConstructorReturn(self, call) {
    if (!self) {
        throw new ReferenceError("this hasn't been initialised - super() hasn't been called");
    }
    return call && (typeof call === "object" || typeof call === "function") ? call : self;
}

function _inherits(subClass, superClass) {
    if (typeof superClass !== "function" && superClass !== null) {
        throw new TypeError("Super expression must either be null or a function, not " + typeof superClass);
    }
    subClass.prototype = Object.create(superClass && superClass.prototype, { constructor: { value: subClass, enumerable: false, writable: true, configurable: true } });
    if (superClass) Object.setPrototypeOf ? Object.setPrototypeOf(subClass, superClass) : subClass.__proto__ = superClass;
}

function _classCallCheck(instance, Constructor) {
    if (!(instance instanceof Constructor)) {
        throw new TypeError("Cannot call a class as a function");
    }
}

var Parent = function Parent(name) {
    _classCallCheck(this, Parent);

    this.name = name;
};

var Child = function(_Parent) {
    _inherits(Child, _Parent);

    function Child(name, age) {
        _classCallCheck(this, Child);

        // 调用父类的 constructor(name)
        var _this = _possibleConstructorReturn(this, (Child.__proto__ || Object.getPrototypeOf(Child)).call(this, name));

        _this.age = age;
        return _this;
    }

    return Child;
}(Parent);

var child1 = new Child('kevin', '18');

console.log(child1);
```

#### inherit

```javascript
function _inherits(subClass, superClass) {
  	// extends 的对象可以是class 或者null
    if (typeof superClass !== "function" && superClass !== null) {
        throw new TypeError("Super expression must either be null or a function, not " + typeof superClass);
    }
    // Object.create 寄生组合式继承，constructor 要重新设置为子类构造函数
    subClass.prototype = Object.create(superClass && superClass.prototype, { constructor: { value: subClass, enumerable: false, writable: true, configurable: true } });
  	// 设置 子类的 __proto__ 指向父类构造函数（继承静态方法和属性）
    if (superClass) Object.setPrototypeOf ? Object.setPrototypeOf(subClass, superClass) : subClass.__proto__ = superClass;
}
```

#### _possibleConstructorReturn

```javascript
function _possibleConstructorReturn(self, call) {
    if (!self) {
        throw new ReferenceError("this hasn't been initialised - super() hasn't been called");
    }
    return call && (typeof call === "object" || typeof call === "function") ? call : self;
}
// 调用
_possibleConstructorReturn(this, (Child.__proto__ || Object.getPrototypeOf(Child)).call(this, name));
// 简化
 _possibleConstructorReturn(this, Parent.call(this, name));
```

也就是说，这个函数会判断 **父构造函数的返回值**，如果没有返回值的话，那么this就是 self，也就是子类自己的this，而如果父类有返回值并且返回值为object或function的话，this就会指向父类的这个返回值

对于这样一个 class：

```javascript
class Parent {
    constructor() {
        this.xxx = xxx;
    }
}
```

Parent.call(this, name) 的值肯定是 undefined。可是如果我们在 constructor 函数中 return 了呢？比如：

```javascript
class Parent {
    constructor() {
        return {
            name: 'kevin'
        }
    }
}
```

我们可以返回各种类型的值，甚至是 null：

```javascript
class Parent {
    constructor() {
        return null
    }
}
```

我们接着看这个判断：

```javascript
call && (typeof call === "object" || typeof call === "function") ? call : self;
```

注意，这句话的意思并不是判断 call 是否存在，如果存在，就执行 `(typeof call === "object" || typeof call === "function") ? call : self`

因为 `&&` 的运算符优先级高于 `? :`，所以这句话的意思应该是：

```javascript
(call && (typeof call === "object" || typeof call === "function")) ? call : self;
```

对于 Parent.call(this) 的值，如果是 object 类型或者是 function 类型，就返回 Parent.call(this)，如果是 null 或者基本类型的值或者是 undefined，都会返回 self 也就是子类的 this。

这也是为什么这个函数被命名为 `_possibleConstructorReturn`。

## 总结

和 es5 function 的区别：

1. 只能通过new 调用

2. 类内部所有定义的方法，都是不可枚举的（non-enumerable）

   也就是说，

   Object.keys(a_class); // []

   Object.keys(a_function); // [a,b]

3. function.prototype = class

   class.constructor = function

es6 继承的实现方式：

1. `inherits(Child, Parent)`，实现 原型链的继承关系
2. `Object.setPrototypeOf(Child, Parent)`， 实现静态方法的继承关系（也就是es5中的function函数对象上的属性与方法 `Person.staticProp`）
3. `Parent.call(this)` 实现实例对象的继承关系，根据 Parent 构造函数的返回值类型确定子类构造函数 this 的初始值 _this。
4. 最终，根据子类构造函数，修改 _this 的值，然后返回该值。

## 参考

[ES6 系列之 Babel 是如何编译 Class 的(上) #105](https://github.com/mqyqingfeng/Blog/issues/105)

[ES6 系列之 Babel 是如何编译 Class 的(下)](https://github.com/mqyqingfeng/Blog/issues/106#)

[MDN-Object.create](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/create)

