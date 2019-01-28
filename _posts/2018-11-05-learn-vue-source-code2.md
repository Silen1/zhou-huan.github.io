---
layout: post
title: "手抄Vue（二）—— 数组的响应"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - Vue.js源码
---

之前梳理 [手抄Vue（一）—— 简单实现数据响应](https://zhou-huan.github.io/2018/11/04/learn-vue-source-code1/) 时没有考虑数组的情况。

js 中数组有很多实例方法，其中有一部分会改变数组本身的值，比如 `push pop shift unshift` 等，这些方法被称为变异方法，这些变异方法也是 Vue 开发中常用的数组操作方法。那么要实现对数组的观测，首先要考虑的就是如何截获这些变异方法的调用。

简单来说，Vue 是通过保持这些数组变异方法原有功能不变的前提下，对其功能进行扩展来实现拦截的。具体怎么操作，可以先看一下例子：
```js
function add10(num) {
    return num + 10
}
console.log(add10(5)) // 15

const originalAdd10 = add10
add10 = function(num) {
    console.log('截获了add10操作')
    return originalAdd10(num)
}
console.log(add10(5)) // '截获了add10操作'
                      // 15
```
该例中，首先使用变量 `originalAdd10` 缓存 `add10` 函数，再重新定义 `add10` 函数，在重新定义的函数体里就可以执行额外增加的功能，比如上例中的 `console.log('截获了add10操作')`，然后执行缓存的 `add10` 函数即 `originalAdd10`，并将结果返回，原理大抵如此。

那么，具体可实现如下：
```js
const mutationMethods = [
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]
const arrayMethods = Object.create(Array.prototype)
const arrayProto = Array.prototype

mutationMethods.forEach(method => {
  arrayMethods[method] = function (...args) {
    const result = arrayProto[method].apply(this, args)
    console.log(`我截获了对数组的${method}操作`)
    return result
  }
})

const arr = ['kobe', 'jordan']
arr.__proto__ = arrayMethods

arr.push('harden') // '我截获了对数组的push操作'
console.log(JSON.stringify(arr)) // '["kobe","jordan","harden"]'
```
以上，`mutationMethods` 是所有要拦截的数组变异方法的集合。

整体思路就是通过设置数组对象的 `__proto__` 属性的值为一个新对象 `arrayMethods`，以代理数组 `mutationMethods` 中的变异方法，并将 `arrayMethods` 的原型设置为数组构造函数本来的原型，这样方能保证除却代理的方法以外，不影响数组本身的其它方法和属性。

其中：
```js
const arrayMethods = Object.create(Array.prototype)
```
以上实现了 `arrayMethods` 的原型是数组构造函数本来的原型，即 `arrayMethods.__proto__ === Array.prototype`。

紧接着：
```js
const arrayProto = Array.prototype
```
这句使用 `arrayProto` 变量缓存了 `Array.prototype`。

再然后：
```js
mutationMethods.forEach(method => {
  arrayMethods[method] = function (...args) {
    const result = arrayProto[method].apply(this, args)
    console.log(`我截获了对数组的${method}操作`)
    return result
  }
})
```
将 `mutationMethods` 进行循环，在 `arrayMethods` 对象上以 `mutationMethods` 中各元素为 `key`，即方法名，定义作为拦截器的同名变异方法。

具体：
```js
const result = arrayProto[method].apply(this, args)
```
执行缓存的 `Array.prototype`，即 `arrayProto` 中对应的变异方法，并传入 `this` 以及 `args`，也就是将来调用该方法的数组对象，和调用该方法时传入的参数（或参数列表）转化成的参数数组，并将结果给到变量 `result`。

这里使用了解构赋值的方式将参数（或参数列表）转化成了参数数组，这么做是因为不能确定参数的个数，所以只能使用 `apply`（不能用 `call`），并传入参数数组。

之后：
```js
console.log(`我截获了对数组的${method}操作`)
```
也就是拦截之后要额外执行的操作了。

最后：
```js
return result
```
将数组原变异方法执行的结果返回，保证原有功能不受影响。

`forEach` 执行完之后：
```js
const arr = ['kobe', 'jordan']
arr.__proto__ = arrayMethods
```
声明并初始化 `arr`，并将 `arr` 的 `__proto__` 指向 `arrayMethods`，这样便代理了 `mutationMethods` 中的变异方法。

最终：
```js
arr.push('harden') // '我截获了对数组的push操作'
console.log(JSON.stringify(arr)) // '["kobe","jordan","harden"]'
```
数组对象手动扩展的功能以及原功能均正常，实现了数组变异方法的拦截。