---
title: javascript之继承
date: 2019-08-15 22:13:22
tags:
- javascript
categories:
- javascript
---

今天复习一下 javascript 的继承

javascript 的继承有：原型继承、经典继承、组合继承、寄生组合式继承几种，我们今天就来介绍一些这些继承的方式

```javascript
function Parent(age) {
  this.name = 'huachenyu';
  this.names = ['huachenyu', 'et'];
  this.age = age;
  this.setName = (name) => {
    this.name = name;
  }
}

Parent.prototype.getName = function() {
	console.log(this.name);
}

function Child() {}
```

我们接下来的所有继承方式，都是为了让Child可以继承到**Parent构造函数**以及**Parent原型链**上的属性和方法，并且没有副作用（比如修改Child的某个属性结果影响到了父元素）

### 原型链继承

顾名思义，用原型链形成的继承方式

```javascript
Child.prototype = new Parent();
```

![](http://ww1.sinaimg.cn/large/8ac7964fly1g60qzi5vvpj20vg0c275c.jpg)

缺点：

1. 引用类型的属性被所有实例共享

   ```javascript
   var child1 = new Child();
   child1.names.push('judy');
   var child2 = new Child();
   console.log(child1.names); // ['huachenyu', 'et', 'judy']
   console.log(child2.names); // ['huachenyu', 'et', 'judy']
   ```

   因为 names 属性，是存储在 Parent 类上的一个 **引用类型的变量**，那么当对 child1的names 属性进行操作时，child 实例上没有names属性，那么就会上翻找到 **作为父亲的 Parent 实例** （所有child实例的原型链上都是这个Parent实例）上的 names 属性，push操作也就是对这个所有child的父亲元素进行操作，此时就会发生 属性共享的问题

2. 创建 child 实例的时候不能向 Parent 传参

### 借用构造函数（经典继承）

```javascript
function Child() {
  Parent.call(this); // 这里是可以将参数传递给 Parent 的
}
```

![](http://ww1.sinaimg.cn/large/8ac7964fly1g60rj1wb2rj216g0kkgoo.jpg)

每个child实例，实际上都是通过指定 this 执行了一遍 Parent 的构造函数，所以每个 child 持有的都是完全不同的 parent 实例，相当于，每个 child 都拷贝了一份 **parent 实例**的属性和方法到自己身上

优点：

1. 避免了引用类型的属性共享的问题
2. 创建 child 实例时可以根据不同情况给parent传参

缺点：

1. 没有形成原型链上的继承关系，child instanceof Parent 返回为false，child 也没有办法继承 **Parent 原型链** 上的属性和方法
2. 可以共享的函数定义没有共享，而是创建了两个一模一样的函数（如我们例子中的 setName

### 组合继承

原型继承 + 构造函数

```javascript
function Child () {
	Parent.call(this);
}
Child.prototype = new Parent();
Child.prototype.constructor = Child;
```

优点：可以使用 instanceof / isPrototypeOf() 判断原型链上的对象属性，而且不会共享不想被共享的对象的和方法（想被共享的放到 Parent.prototype上，不想被共享的放在 Parent 构造函数内部）

缺点：父原型对象的构造函数会被调用两次，

- 第一次是在 构建原型链的时候，Child的prototype（原型对象）上 创建了一个Parent 实例，那么Child.prototype 会有一个 name 的属性，
- 而在初始化 child 对象时，new Child 在Child 的构造函数中 又调用了一次 Parent的构造函数，在Child对象上添加了一个name  的属性

### 寄生组合式继承

组合式继承最大的缺点在于会调用两次 父类的构造函数，一次在

```javascript
Child.prototype = new Parent()
```

另一次在

```javascript
var child = new Child();
```

这个时候会发现，在 child 和 Child.prototype 上，会存在一套相同的 names、name、以及 setName 属性

那么如何避免这次多余的调用呢

也就是说 如何优化`Child.prototype = new Parent()` ，使得 Child.prototype 上不会存在 这套多余的属性

只要将 Child.prototype 直接连接到 Parent.prototype上即可，Parent实例的属性由 Parent.call 提供，而Parent原型链的属性则由 Parent.prototype 提供

```javascript
function inherit(Child, Parent) {
  function F() {};
  F.prototype = Parent.prototype;
  Child.prototype = new F();
  Child.prototype.constructor = Child;
}
或
function inherit(Child, Parent) {
	Child.prototype = Object.create(Parent.prototype); 
  // Object.create(proto, [protoProp])，以proto为原型创建一个新的对象
  Child.prototype.constructor = Child;
}
```

中间的 function F，切断了 Child.prototype 和 Parent 构造函数之间的联系，进而直接跟 Parent.prototype 建立了联系

引用《JavaScript高级程序设计》中对寄生组合式继承的夸赞就是：

这种方式的高效率体现它只调用了一次 Parent 构造函数，并且因此避免了在 Parent.prototype 上面创建不必要的、多余的属性。与此同时，原型链还能保持不变；因此，还能够正常使用 instanceof 和 isPrototypeOf。开发人员普遍认为寄生组合式继承是引用类型最理想的继承范式。

### 参考

[Javascript深入之继承](https://github.com/mqyqingfeng/Blog/issues/16)