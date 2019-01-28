---
layout: post
title: "手抄Vue（六）—— 实现Vue.delete"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - Vue.js源码
---

`Vue.js` 源码中，`Vue.delete` 和 `vm.$delete` 指向的是同一个函数，两者作用完全相同，其中，`vm.$delete` 是 `Vue.delete` 的别名。

> `Vue.delete` 用来删除对象的属性。如果对象是响应式的，确保删除能触发更新视图。这个方法主要用于避开 Vue 不能检测到属性被删除的限制，但是你应该很少会使用它。需要注意的是，目标对象不能是一个 Vue 实例或 Vue 实例的根数据对象。

也可以使用这个方法来删除数组中的元素。

该方法接收两个参数：

* `{Object | Array} target` 目标对象/目标数组
* `{string | number} key/index` 要删除的对象键名/要删除的数组元素索引

那要实现一个功能相同的 `myDelete` 方法的话，其实只要将对象中的属性删除之后，通知视图更新，即执行收集的依赖即可。那么：
```js
function myDelete(obj, key) {
  delete obj[key]
  target()
}
```
这里通过调用 `target` 来执行依赖通知视图更新，`target` 是前面写的帮助中转依赖的一个全局变量。可以这么写是因为我到现在一直是使用之前写的 `render` 方法来做测试，没有别的依赖，而且只调用了一次 `myWatch` 方法，也没有收集别的依赖，所以这里也先通过调用 `target()` 来实现通知视图更新的功能。这样做肯定会有很多问题，但这些问题都需要依赖 `Dep` 类来解决，所以也放到 `Dep` 类之后再写。

这个方法实现了删除对象属性，但还不能删除数组元素，故增加以下代码：
```js
function myDelete(obj, key) {
  if (Array.isArray(obj) && isValidArrayIndex(key)) {
    obj.splice(key, 1)
    return
  }
  delete obj[key]
  target()
}
```
如果检测到传进来的 `obj` 是数组，并且 `key` 是个有效的数组索引，那么使用 `splice` 方法直接将 `key` 位置上的数组元素删除即可，变异方法 `splice` 本身已经可以触发响应，所以删除之后 `return` 即可。其中，`isValidArrayIndex` 方法是 [Vue数据响应原理（五）—— 实现Vue.set](https://zhou-huan.github.io/2018/11/12/learn-vue-source-code5/) 中已经封装好的方法。

将 `myDelete` 方法和前面文章中的代码放在一起，经过测试，功能是没有问题的。但同样，还要兼容异常与边界情况：

如果要删除的 `key` 不是 `obj` 自身的属性的话，那直接 `return` 就好了，什么也不用做。顺便封装一个 `hasOwn` 方法：
```js
const hasOwnProperty = Object.prototype.hasOwnProperty
function hasOwn(obj, key) {
  return hasOwnProperty.call(obj, key)
}
```
不直接使用 `obj.hasOwnProperty(key)` 而去调 `Object` 原型上的 `hasOwnProperty` 方法是因为传进来的 `obj` 是不受控制的，`obj` 上可能会有重新定义过的 `hasOwnProperty` 方法。

这时，增加以下判断：

```js
function myDelete(obj, key) {
  if (Array.isArray(obj) && isValidArrayIndex(key)) {
    obj.splice(key, 1)
    return
  }
  if (!hasOwn(obj, key)) {
    return
  }
  delete obj[key]
  target()
}
```
即最终代码。