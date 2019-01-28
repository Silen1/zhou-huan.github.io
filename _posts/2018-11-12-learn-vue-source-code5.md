---
layout: post
title: "手抄Vue（五）—— 实现Vue.set"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - Vue.js源码
---

Vue.js 中， `Vue.set` 和 `vm.$set` 是一回事，其中 `vm.$set` 是 `Vue.set` 的别名，二者在 `Vue.js` 源码当中指向的是同一个函数。

> `Vue.set` 可以向响应式对象中添加一个属性，并确保这个新属性同样是响应式的，且触发视图更新。它必须用于向响应式对象上添加新属性，因为 Vue 无法探测普通的新增属性 (比如 `this.myObject.newProperty = 'hi'`)。但对象不能是 Vue 实例，或者 Vue 实例的根数据对象。

这个方法接收三个参数：

* `{Object | Array} target` 目标对象
* `{string | number} key` 添加的键名
* `{any} value` 添加的值

返回值为开发者所设置的值。

那要实现一个功能相同的 `mySet` 方法的话，其实只要将新添加的属性转化为响应式属性即可：
```js
function mySet(target, key, val) {
  defineReactive(target, key)
  return val
}
```
首先使用 [Vue数据响应原理（四）—— 封装Observer类](https://zhou-huan.github.io/2018/11/08/learn-vue-source-code4/) 中定义好的方法 `defineReactive` 将属性转化为响应式属性，再将对应添加的值返回。

但很容易看出来这里存在一个问题，`defineReactive` 确实是将属性转化成了响应式属性，但是，它没有像期望的那样同时为属性设置一个初始值，这样的话，只有在属性值发生变化时才会借由 `defineReactive` 中的 `setter` 为属性设置上新的值，否则它一直都是 `undefined`。那么，可以将 `defineReactive` 方法改造一下以解决这个问题：

_这是原来的 `defineReactive` 方法：_
```js
function defineReactive(obj, key) {
  const dep = []

  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && !property.configurable) {
    return
  }

  const getter = property && property.get
  const setter = property && property.set

  let value = obj[key]
  observe(value)
  Object.defineProperty(obj, key, {
    get () {
      getter && (value = getter.call(obj))
      dep.push(target)
      return value
    },
    set (newVal) {
      getter && (value = getter.call(obj))
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      if (setter) {
        setter.call(obj, newVal)
      } else {
        value = newVal
      }
      observe(newVal)
      dep.forEach(f => {
        f()
      })
    }
  })
}
```
_这是改造后的 `defineReactive` 方法：_
```js
function defineReactive(obj, key, value) {
  const dep = []

  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && !property.configurable) {
    return
  }

  const getter = property && property.get
  const setter = property && property.set

  // 一
  if (arguments.length === 2) {
    value = obj[key]
  }

  observe(value)
  Object.defineProperty(obj, key, {
    get () {
      getter && (value = getter.call(obj))
      dep.push(target)
      return value
    },
    set (newVal) {
      getter && (value = getter.call(obj))
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      if (setter) {
        setter.call(obj, newVal)
      } else {
        value = newVal
      }
      observe(newVal)
      dep.forEach(f => {
        f()
      })
    }
  })
}
```
注释一的地方改造之前是这样：
```js
let value = obj[key]
```
现在增加一个判断：
```js
arguments.length === 2
```
如果只传进来了两个参数，也就是说没有第三个参数 `value`，那么再去给 `value` 取值：
```js
value = obj[key]
```
否则，`value` 就是传进来的那个值。

那么相应的，`mySet` 方法里调用 `defineReactive` 时也将属性值传入：
```js
function mySet(target, key, val) {
  defineReactive(target, key, val)
  return val
}
```
这样一来，新添加的属性已经是响应式属性并且拥有了初始值，只要在需要的时候再使用`myWatch` 方法为之添加相应的依赖即可。

将上述代码和上一篇的最终代码放在一起并添加如下测试代码：
```js
const data = {
  name: 'kobe bryant',
  otherInfo: {
    height: 198,
    numbers: [8, 24]
  },
  teammates: [
    'paul gasol',
    {
      name: 'shaq',
      numbers: [32, 34, 33]
    }
  ]
}

mySet(data, 'age', 40)

function render() {
  document.body.innerText = `我最喜欢的NBA球员是${data.name}，${data.age}岁，身高${data.otherInfo.height}cm，穿过${data.otherInfo.numbers.length}个球衣号码，${data.otherInfo.numbers[0]}和${data.otherInfo.numbers[1]}，他的队友有${data.teammates[0]}和${data.teammates[1].name}，其中，${data.teammates[1].name}在湖人时期穿的球衣号码为${data.teammates[1].numbers[1]}号`
}

observe(data)

myWatch(render, render)
```
测试结果和期望是一致的，通过 `mySet` 方法后来添加给 `data` 的属性 —— `age` 正确 `"render"` 到了页面并且可以响应对于它的修改操作。

实现 `mySet` 方法的思路大概就是这样。

和之前一样，方法实现了以后还要再去考虑对于边界情况与异常情况的兼容：

* 如果传进来的 `target` 对象不存在（`undefined` 或 `null`）？

顺便封装一个用来判断 `undefined` 和 `null` 的公共方法：
```js
function isUndef(value) {
  return value === undefined || value === null
}
```
那么只要在 `mySet` 中加句判断就好：
```js
function mySet(target, key, val) {
  if (isUndef(target)) {
    console.error('不能给 undefined 设置响应式属性！')
    return val
  }
  defineReactive(target, key, val)
  return val
}
```
如果目标对象 `target` 不存在，那么打印错误信息，并将 `val` 返回。

* 如果传进来的 `target` 对象是原始类型？

同样，封装一个用来判断原始类型值的公共方法：
```js
function isPrimitive(value) {
  return (
    typeof value === 'string' ||
    typeof value === 'number' ||
    typeof value === 'boolean' ||
    typeof value === 'symbol'
  )
}
```
那么，接着在 `mySet` 方法中添加判断：
```js
function mySet(target, key, val) {
  if (isUndef(target)) {
    console.error('不能给 undefined 和 null 设置响应式属性！')
    return val
  }
  if (isPrimitive(target)) {
    console.error('不能给原始类型的值设置响应式属性！')
    return val
  }
  defineReactive(target, key, val)
  return val
}
```
* 如果传进来的目标对象是一个数组？

如果目标对象是数组的话，参数 `key` 相应的就是数组索引了，那么有必要先封装一个验证数组索引有效性的方法：
```js
function isValidArrayIndex(index) {
  return index >= 0 && Math.floor(index) == index && isFinite(index)
}
```
只要 `index` 是一个大于等于零的整数，并且是有限的，那就可以断定它是一个有效的数组索引。

但，它其实还是有问题的：
```js
isValidArrayIndex([]) // true
```
这显然是不对的，出现错误是因为：
```js
Number([]) // 0
```
那么经过隐式类型转换：
```js
[] >= 0 // true
Math.floor([]) == [] // true
isFinite([]) // true
```
所以，传入空数组时 `isValidArrayIndex` 方法的返回值也就是 `true` 了。

Vue.js 里是这样写的：
```js
function isValidArrayIndex(index) {
  const n = parseFloat(String(index))
  return n >= 0 && Math.floor(n) === n && isFinite(index)
}
```
这就解决了我写出来的问题，`parseFloat(String(index))` 将 `index` 转为数值类型的同时，又不会像 `Number(index)` 一样将空数组转为 `0`：
```js
String([]) // ""
parseFloat("") // NaN
```
有了 `isValidArrayIndex` 方法之后，`mySet` 中接着添加以下代码：
```js
function mySet(target, key, val) {
  if (isUndef(target)) {
    console.error('不能给 undefined 和 null 设置响应式属性！')
    return val
  }
  if (isPrimitive(target)) {
    console.error('不能给原始类型的值设置响应式属性！')
    return val
  }
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    target.length = Math.max(target.length, key)
    target.splice(key, 1, val)
    return val
  }
  defineReactive(target, key, val)
  return val
}
```
判断如果 `target` 为数组，并且 `key` 是一个有效的数组索引，就进行以下操作：

首先将数组 `target` 的长度修改为 `target.length` 和 `key` 中较大的那一个，这是为了防止如果 `key` 大于原 `target.length` 的话，后面对 `target` 的 `splice` 操作（`target.splice(key, 1, val)`）会报错。

再然后对数组指定元素进行替换操作，即：
```js
target.splice(key, 1, val)
```
替换的同时数组的变异方法 `splice` 本身也是能够触发响应的。

最后将 `val` 返回。

* 如果新增的 `key` 本来就在目标对象 `target` 中？

这种情况兼容起来非常容易：
```js
function mySet(target, key, val) {
  if (isUndef(target)) {
    console.error('不能给 undefined 和 null 设置响应式属性！')
    return val
  }
  if (isPrimitive(target)) {
    console.error('不能给原始类型的值设置响应式属性！')
    return val
  }
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    target.length = Math.max(target.length, key)
    target.splice(key, 1, val)
    return val
  }
  if (target.hasOwnProperty(key)) {
    target[key] = val
    return val
  }
  defineReactive(target, key, val)
  return val
}
```
如果新增的 `key` 本来就在目标对象 `target` 中，那么直接修改该属性值即可：`target[key] = val`， `target` 中已有的属性本来就是响应式的。

但使用 `target.hasOwnProperty(key)` 来判断其实是有点问题的。

比如我有一个 `Model` 类如下：
```js
class Model {
  constructor() {
    this.foo = 1
    this._bar = 2
  }
  get bar() {
    return this._bar;
  }
  set bar(newvalue) {
    this._bar = newvalue;
  }
}
```
如果我的 `data` 是通过 `Model` 类初始化的：
```js
const data = new Model()
```
这时候 `data` 其实就是这个样子：
```js
{
  foo: 1,
  _bar: 2,
  __proto__: {
    get bar: function() {
      return this._bar
    },
    set bar: function(newvalue) {
      return this._bar = newvalue
    }
  }
}
```
`bar` 并不是实例的私有属性。

但使用 `target.hasOwnProperty(key)` 进行判断时条件就不成立了，这就会导致走到下面这段代码：
```js
defineReactive(target, key, val)
return val
```
进而，这段代码又会为 `data` 实例重新定义了一个私有属性 `bar`，但这显然不是我们期望的结果，开发者是把 `bar` 写在原型上的。

所以最好是使用 `in` 操作符进行判断：
```js
key in target && !(key in Object.prototype)
```
这个时候判断条件就是成立的了，也就不会走到最后 `defineReactive` 为 `data` 实例重新定义私有属性 `bar` 的地方去了。

那么最终，代码就成了这个样子：
```js
function mySet(target, key, val) {
  if (isUndef(target)) {
    console.error('不能给 undefined 和 null 设置响应式属性！')
    return val
  }
  if (isPrimitive(target)) {
    console.error('不能给原始类型的值设置响应式属性！')
    return val
  }
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    target.length = Math.max(target.length, key)
    target.splice(key, 1, val)
    return val
  }
  if (key in target && !(key in Object.prototype)) {
    target[key] = val
    return val
  }
  defineReactive(target, key, val)
  return val
}
```
