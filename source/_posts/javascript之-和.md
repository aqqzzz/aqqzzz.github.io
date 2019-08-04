---
title: javascript之==和===
date: 2019-08-04 20:08:57
tags:
- javascript
categories:
- javascript
---

javascript 中， == 和 === 是有所不同的，== 号两侧的标记，只要经过类型转换之后可以相等，那么返回结果就会是true，而 === 则需要等号两侧完全相同，类型也必须相同，我们在开发过程中，通常会要求使用全等运算符（===），这样有助于我们写出更加可以预测行为的代码，但是我们也需要了解 == 的转换逻辑

== 转换逻辑可以用下图概括

![img](https://user-gold-cdn.xitu.io/2018/12/19/167c4a2627fe55f1?imageslim)

==状态下判断结果的流程图，可以总结为以下四步

1. undefined == null，结果为true，且他俩和所有其他值比较的结果都是 false
2. String == Boolean，需要两个操作数同时转为 Number
3. String/Boolean == Number，需要String/Boolean 转为 Number
4. Object == Primitive，需要Object转为Primitive（通过valueOf 和 toString 方法）
   - 如果Object为Date，则先调用toString，若toString的结果不是基本类型，再调用valueOf
   - 如果Object为其他类型（obj、array、func)，则先调用 valueOf，若结果不为**基本类型**，再调用 toString
   - 基本类型：指 null、undefined、string、number、boolean



那么为什么 [] == ![] 结果为true呢

1. 运算符优先级： ! > ==，所以先计算！（非运算符）

2. 非运算符的运算规则，!toBoolean(oldValue)，需要将[]先转为boolean值

   - boolean转换规则：

     - Number：0、NaN为false
     - undefined、null：false
     - string：空字符串为false
     - object：true

     由于[]为object，所以toBoolean([])为true，取反，![] 结果为false

3. 问题转换为：[] == false

   - Object 和 Primitive 类型的比较，首先要把Object类型转换为Primitive类型
   - []为普通object类型（不是date），所以先调用valueOf()，结果为[]，仍然不是Primitive类型，无法比较
   - 再调用[]的toString方法，[].toString() =''，为字符串类型，结束

4. 问题转换为：'' == false

   - string 和 boolean 进行比较，需要将两个值都转换为 number类型
   - toNumber('') = 0
   - toNumber(false) = 0
   - 所以结果为true

#### [] == [] 结果是什么

答：false，因为 array为引用类型，两个 [] 指向不同的内存地址

#### ![] == {} 结果是什么

![]结果为false，{}在valueOf和toString后结果为"[Object Object]"，转换为Number为NaN，NaN与任何值比较的结果都是false

#### {} == ![] 结果是什么

报错：==（因为{}被认为是空代码块 而不是 对象Object）

js 语句优先，{} 有三个语义

- 复合语句块
- 对象直接量
- 声明函数块

理解顺序如上，所以{}首先被理解为 复合语句块

#### {} == !{} 结果是什么

false，【？？？？】为什么没有语句优先

### + 运算符

https://segmentfault.com/a/1190000007184573

#### 转换规则

```
operand + operand = result
```

1. 使用 ToPrimitive 运算符转换 两个操作符为原始数据类型
2. 转换结束之后，如果有运算元出现原始数据类型为**“字符串”**时，则另一运算元 强转为string，再进行**字符串连接运算**
3. 其他情况，所有运算元都会转换为 **数字** 类型，然后做数字的相加运算

ToPrimitive 运算符

`ToPrimitive(input, PreferredType?)`

- preferredType ：Number | String
  - Number
    - 若input为原始数据类型，直接返回input
    - 当input是一个对象时，调用对象的valueOf() 方法，若能够得到原始类型的值，则返回这个值
    - 否则，如果input调用valueOf之后仍然为一个对象，再调用input的toString() 方法，如果能得到原始数据类型的值，则返回这个值
    - 否则，抛出 TypeError 错误（可能用户覆盖了object的toString 方法，导致这个方法 返回了一个对象而不是string）
  - String（即Number类型的2、3步骤对调）
    - input（原始数据类型直接输出）
    - input为一个对象时，先调用toString，再调用valueOf（如Date对象就是这样的）
    - 否则，抛出TypeError错误
  - default：
    - Date对象和Symbol对象的预设类型为 String
    - 其他对象为 Number
- 一些特殊情况
  - Object的toString，会输出该 object 的type "[object type]"，而Array的toString，会返回将数组用 `join(',')` 操作后返回的字符串
    - 如果想用toString来判断对象类型，可以通过 Object.prototype.toString.call(obj)来判断，因为Array的toString表现不同只是因为Array对象的prototype上覆写了toString方法
    - Function 的 toString，返回 将整个函数转换为string的结果
  - 一元正号（+），会让这个PreferedType 设置为 Number（如：+[] 的输出其实为 Number([])，输出0）

#### 实例

【注】 undefined 转换为数字为 NaN，而 null 转换为数字为 0

[] + []

解析：

1. toPrimitive([]) => 
   - [].valueOf() = [] 不是原始类型
   - [].toString() = ''为原始类型
2. '' + []
3. 第二个 [] 同理，结果为 ''

{} + {}

解析：两种情况

- 第一个 {} 被当做对象处理（Chrome浏览器）
  - toPromitive({})，valueof为{}，toString为"[object object]"
  - "[object object]" + {}
  - 第二个 {} 直接toString
- 第一个 {} 被当做代码块处理（Firefox）
  - 第一个 {} 代码块直接被忽略，原问题转化为 +{}
  - Number({})为NaN

{} + []

不同浏览器表现相同，第一个 {} 都被当作 代码块处理，则结果为 Number([])，结果为0

【注】所有 {} + anyType 的运算，第一个{} 都会当作代码块而忽略，直接运行 Number(anyType)

### 几个神奇的转换

![img](https://pic3.zhimg.com/v2-c6abab0936a5e136e306123d6d7036d3_b.jpg)

(! + [] + [] + ![]).length

!(+[]) + [] + ![]

!Number([]) + [] + ![]

true + [].valueOf() + ![]

true + [].toString() + ![]

true + '' + false = 'truefalse'.length = 9