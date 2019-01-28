---
layout: post
title: "手抄Vue（三）—— 数组响应的补充"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - Vue.js源码
---

上篇 [Vue数据响应思路（二）—— 数组的响应](https://zhou-huan.github.io/2018/11/05/learn-vue-source-code2/) 中，没有考虑兼容性问题，`__proto__` 属性在 IE10 以及更低版本 IE 中是不支持的，需实现兼容方案。

思路就是直接在数组实例上面定义新的同名变异方法作为“拦截器”：
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
    console.log(`我截获了对数组的${method}操作`)
    return result
  }
})

const arr = ['kobe', 'jordan']

mutationMethods.forEach(method => {
  arr[method] = arrayMethods[method]
})

arr.push('wade') // '我截获了对数组的push操作'
console.log(JSON.stringify(arr)) // '["kobe","jordan","wade"]'
```
数组变异方法扩展功能以及原功能均正常。

但仍然有一个问题：
```js
console.log(JSON.stringify(Object.keys(arr))) // ["0","1","2","push","pop","shift","unshift","splice","sort","reverse"]
```
直接添加到数组实例上的这些方法都是可枚举的，按理说，数组本身的变异方法不应该被枚举出来。

那么，使用 `Object.defineProperty` 将其定义为不可枚举：
```js
mutationMethods.forEach(method => {
  Object.defineProperty(arr, method, {
    enumerable: false,
    writable: true,
    configurable: true,
    value: arrayMethods[method]
  })
})

arr.push('wade') // '我截获了对数组的push操作'
console.log(JSON.stringify(arr)) // '["kobe","jordan","wade"]'

console.log(JSON.stringify(Object.keys(arr))) // '["0","1","2"]'
```
那么低版本 IE 的兼容代码就成了这样：
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
    console.log(`我截获了对数组的${method}操作`)
    return result
  }
})

const arr = ['kobe', 'jordan']

mutationMethods.forEach(method => {
  Object.defineProperty(arr, method, {
    enumerable: false,
    writable: true,
    configurable: true,
    value: arrayMethods[method]
  })
})
```
结合上一篇，便可实现最终兼容方案：
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
    console.log(`我截获了对数组的${method}操作`)
    return result
  }
})

const arr = ['kobe', 'jordan']

if ('__proto__' in {}) {
  arr.__proto__ = arrayMethods
} else {
  mutationMethods.forEach(method => {
    Object.defineProperty(arr, method, {
      enumerable: false,
      writable: true,
      configurable: true,
      value: arrayMethods[method]
    })
  })
}
```
