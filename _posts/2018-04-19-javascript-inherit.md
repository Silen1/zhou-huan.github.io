---
layout: post
title: "JavaScript继承"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - JavaScript
---

## 1. 使用原型链实现继承
```js
function Box () {
    this.type = 'box'
}
Box.prototype.getType = function () {
    return this.type
}
function RedBox () {
    this.color = 'red'
}
// 让RedBox继承Box
RedBox.prototype = new Box()
RedBox.prototype.getColor = function () {
    return this.color
}
var redBox = new RedBox()
console.log(redBox.getColor())           // 'red'
console.log(redBox.getType())            // 'box' - 实现了继承
```
### 这种继承方式有两个缺点：

* 包含引用类型值的原型会被所有实例共享，请看下例：

```js
function Box () {
    this.inside = ['apple', 'peach', 'orange']
}
function RedBox () {
}
// 让RedBox继承Box
RedBox.prototype = new Box
var redBox1 = new RedBox
redBox1.inside.push('banana')
console.log(redBox1.inside)         // ["apple", "peach", "orange", "banana"]
var redBox2 = new RedBox()
console.log(redBox2.inside)         // ["apple", "peach", "orange", "banana"]
// 对redBox1.inside进行修改，redBox2.inside也反映了这一修改
```

* 在创建子类型时，不能向超类型的构造函数中传递参数

## 2. 借用构造函数实现继承

```js
function Box () {
    this.inside = ['apple', 'peach', 'orange']
}
function RedBox () {
    // 继承Box
    Box.call(this)
}
var redBox1 = new RedBox()
redBox1.inside.push('banana')
console.log(redBox1.inside)         // ["apple", "peach", "orange", "banana"]
var redBox2 = new RedBox()
console.log(redBox2.inside)         // ["apple", "peach", "orange"]
```

### 这种继承方式的优点：

* RedBox的实例之间共享Box的inside属性，且互不影响

* 可以在RedBox（子类型）构造函数中向Box（超类型）构造函数传递参数，请看下例：

```js
function Box (height) {
    this.height = height
}
function RedBox () {
    // 继承了Box，同时还传递了参数
    Box.call(this, '10cm')
    // 实例属性
    this.color = 'red'
}
var redBox = new RedBox()
console.log(redBox.color)           // 'red'
console.log(redBox.height)          // '10cm'

// 在RedBox构造函数内部调用Box构造函数时，实际上是为RedBox的实例设置了height属性
// 为了确保Box构造函数不会重写子类型RedBox的属性，最好在调用了Box构造函数之后，再添加应在子类型中定义的属性
```

### 这种继承方式的缺点：

* 由于方法都在构造函数中定义，则无法实现函数复用

* 超类型的原型中定义的方法，对于子类型不可见

## 3. 组合继承

```js
function Person (name) {
    this.name = name
    this.hobbies = ['basketball', 'music', 'reading']
}
Person.prototype.getName = function () {
    console.log(this.name)
}
function Boy (name, age) {
    // 继承Person的属性
    Person.call(this, name)

    this.age = age
}
// 继承Person的方法
Boy.prototype = new Person()
// 让constructor再指回Boy
Boy.prototype.constructor = Boy
Boy.prototype.getAge = function () {
    console.log(this.age)
}

var  boy1 = new Boy('paul', 23)
boy1.hobbies.push('bodybuilding')
console.log(boy1.hobbies)           // ["basketball", "music", "reading", "bodybuilding"]
boy1.getAge()           // 23
boy1.getName()          // 'paul'

var boy2 = new Boy('james', 22)
console.log(boy2.hobbies)           // ["basketball", "music", "reading"]
boy2.getAge()           // 22
boy2.getName()          // 'james'
```

### 优点：

* 不同的Boy实例可以分别拥有自己的属性，同时还可以使用相同的方法。这种方式避免了原型链和借用构造函数的缺陷，融合了它们两者个优点

## 4. 原型式继承

```js
function object (o) {
    function Tmp () {}
    Tmp.prototype = o
    return new Tmp()
}

var person = {
    name: 'wade',
    friends: ['kobe', 'james']
}

var person1 = object(person)
person1.name = 'harden'
person1.friends.push('green')

var person2 = object(person)
person2.name = 'jordan'
person2.friends.push('durant')

console.log(person.friends)         // ["kobe", "james", "green", "durant"]
```

这种原型式继承，要求必须有一个对象可以作为另一个对象的基础，我们把这个对象传递给object()函数，返回的对象就是以该对象作为原型的，再然后根据具体需求对得到的对象加以修改即可。

### 缺点

* person.friends不仅为person所有，而且也会被person1和person2共享

ECMAScript5通过新增的Object.create()方法规范化了原型式继承。该方法接受两个参数：一个是用作新对象原型的对象，一个是为新对象定义额外属性的对象。第二个参数可选，在传入一个参数的情况下，该方法和前面的object()方法的行为相同。请看下例：

```js
var person = {
    name: 'wade',
    friends: ['kobe', 'james']
}

var person1 = Object.create(person)
person1.name = 'jordan'
person1.friends.push('durant')

var person2 = Object.create(person)
person2.name = 'paul'
person2.friends.push('green')

console.log(person.friends)         // ["kobe", "james", "durant", "green"]

Object.create()方法与Object.defineProperties()方法的第二个参数格式相同：每个属性都是通过自己的描述符定义的。以这种方式指定的任何属性都会覆盖原型对象上的同名属性。请看下例：

var person = {
    name: 'wade',
    friends: ['kobe', 'james']
}

var person1 = Object.create(person, {
    name: {
        value: 'jordan'
    }
})

console.log(person1.name)           // 'jordan'
```

在没有必要兴师动众地创建构造函数，而只是想让一个对象与另一个对象保持类似的情况下，原型式继承是完全可以胜任的。


## 5. 寄生式继承

```js
function object (o) {
    function Tmp () {}
    Tmp.prototype = o
    return new Tmp()
}

function createPerson (original) {
    // 通过调用object创建一个新对象
    var clone = object(original)
    // 增强这个对象
    clone.saySth = function () {
        console.log('hello summer!')
    }
    // 返回该对象
    return clone
}

var person = {
    name: 'wade',
    friends: ['kobe', 'james']
}

var anotherPerson = createPerson(person)
anotherPerson.saySth()          // 'hello summer!'
```

anotherPerson不仅拥有person的所有属性和方法，而且还有自己的方法。在主要考虑对象而不是自定义类型和构造函数的情况下，寄生式继承也是一种很有用的模式。

示例中的object()函数不是必需的，任何能够反悔新对象的函数都能够适用于此模式。

### 缺点

* 使用寄生式继承为对象添加函数会由于不能做到函数复用而降低效率。

## 6. 寄生组合式继承

前面说到的组合式继承是js中最常用的继承模式，不过它也有一定的不足，即会调用两次超类型构造函数，一次是在创建子类型原型时，一次是在子类型构造函数内部。

寄生组合式继承，就是通过借用构造函数来继承属性，通过原型链的混成形式来继承方法。

背后的基本思路是：不用为了指定子类型的原型而去调用超类型的构造函数，我们需要的无非就是超类型原型的一个副本，本质上，就是使用寄生式继承来继承超类型的原型，然后再将结果指定给子类型。

寄生组合式继承的一个基本模式如下：

```js
function inheritPrototype (subType, superType) {
    var prototype = object(superType.prototype)         // 创建对象
    prototype.constructor = subType
    subType.prototype = prototype           // 指定对象
}
```

使用如下：

```js
function object (o) {
    function Tmp () {}
    Tmp.prototype = o
    return new Tmp()
}

function inheritPrototype (subType, superType) {
    var prototype = object(superType.prototype)         // 创建对象
    prototype.constructor = subType
    subType.prototype = prototype           // 指定对象
}

function SuperType (name) {
    this.name = name
    this.friends = ['kobe', 'james']
}
SuperType.prototype.getName = function () {
    console.log(this.name)
}

function SubType (name, age) {
    SuperType.call(this, name)
    this.age = age
}

inheritPrototype(SubType, SuperType)

SubType.prototype.getAge = function () {
    console.log(this.age)
}

var person = new SubType('kobe', 39)
person.getName()            // kobe
person.getAge()         // 39
```

### 优点

* 这种方式的高效体现在只调用一次超类型构造函数，并且避免了在子类型的原型对象上面创建不必要的多余属性。与此同时，原型链还能保持不变。寄生组合式继承是引用类型最理想的继承方式。
