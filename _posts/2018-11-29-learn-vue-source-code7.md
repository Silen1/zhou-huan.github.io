---
layout: post
title: "手抄Vue（七）—— 无题"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - Vue.js源码
---

学习Vue的笔记已经写到了第七篇，但其实直到现在我都一直是在浏览器控制台里写代码和测试的，因为原先只是想稍微了解一下原理记记笔记，但后来了解得越多愈发觉得自己知道的越少，深深感受到了自己学识的疏浅和视野的狭隘。所以想把前面写的整理一下，并在此基础上争取多写一些。

我在本地建了一个项目，把已经有的思路重新写了一遍，大体上还和之前差不多，只是在一些方法的实现和变量的命名上可能会有微小的差异。

项目暂时是这样：

![structure](/img/article/structure.png)

**core/index.js**

```js
import {observe, myWatch} from './observer'
import set from './set'

let data = {
  name: 'James Harden',
  team: 'Houston Rockets',
  teammate: {
    name: 'chris paul',
    team: {
      now: 'rockets',
      before: 'clippers'
    }
  },
  stars: ['kobe', 'jordan', 'wade']
}

function render() {
  document.body.innerText = `The last season's MVP is ${data.name}, he's from ${data.team}, he has a teammate ${data.teammate.name}, ${data.teammate.name} was played for ${data.teammate.team.before}, he is ${data.age} this year`
}

observe(data)

set(data, 'age', 40)

myWatch(render, render)
```

**core/observer.js**

```js
import { isPlainObj } from '../shared/util'

class Observer {
  constructor (value) {
    if (Array.isArray(value)) { // 数组
      value.__proto__ = arrayMethods
      this.observeArray(value)
    } else { // 对象
      this.walk(value)
    }
  }

  walk (obj) { // 逐一观察对象中的属性
    const keys = Object.keys(obj)
    keys.forEach((key) => {
      defineReactive(obj, key)
    })
  }

  observeArray (arr) { // 深度观测数组
    arr.forEach(item => {
      observe(item)
    })
  }
}

// 收集依赖的篮子
let basket

// 将某一属性定义为响应式属性
export function defineReactive (obj, key, val) {
  const deps = []

  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && !property.configurable) { // 如果指定的属性不在该对象上 会返回undefined
    return
  }

  const getter = property && property.get // 缓存原来的getter和setter
  const setter = property && property.set

  let value = obj[key]
  if (arguments.length === 3) {
    value = val
  }
  observe(value) // 实现深度观测

  Object.defineProperty(obj, key, {
    get () {
      if (getter) {
        value = getter.call(obj)
      }
      deps.push(basket)
      return value
    },
    set (newVal) {
      if (getter) {
        value = getter.call(obj)
      }
      if (newVal === value || (newVal !== newVal && value !== value)) { // 新旧值相等的话什么也不做
        return
      }
      if (setter) {
        setter.call(obj, newVal)
      } else {
        value = newVal
      }
      observe(newVal) // 新值可能是对象 继续观察新值
      deps.forEach((fn) => {
        fn()
      })
    }
  })
}

// 观察某一对象 将它转为响应式的
export function observe (target) {
  if (Array.isArray(target) || isPlainObj(target)) {
    return new Observer(target)
  }
}

// 要代理的变异方法集合
const methodsToPatch = [
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]
const arrayProto = Array.prototype
const arrayMethods = Object.create(arrayProto)
methodsToPatch.forEach((method) => {
  arrayMethods[method] = function (...args) {
    arrayProto[method].apply(this, args)
    console.log(`我捕获了数组的${method}方法`)
  }
})

export function myWatch (exp, depFn) {
  basket = depFn
  if (typeof exp === 'function') {
    exp()
    return
  }
  if (/\./.test(exp)) {
    let helpData = data
    exp.split('.').forEach((path) => {
      helpData = helpData[path]
    })
    return
  }
  data[exp]
}
```

**core/set.js**

```js
import { isUndef, isPrimitive, isValidArrayIndex } from '../shared/util'
import { defineReactive } from './observer'

export default function set (target, key, val) {
  if (isUndef(target)) {
    console.error('不能给 undefined 和 null 设置响应式属性！')
    return
  }
  if (isPrimitive(target)) {
    console.error('不能给原始类型的值设置响应式属性！')
    return
  }
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    target.length = Math.max(target.length, key)
    target.splice(key, 1, val)
    return val
  }
  if (key in target && !(key in Object.prototype)) {
    target[key] = val
    return
  }
  defineReactive(target, key, val)
  return val
}
```

**core/delete.js**

```js
import { isValidArrayIndex, hasOwn } from '../shared/util'

export default function del (obj, key) {
  if (Array.isArray(obj) && isValidArrayIndex(key)) {
    obj.splice(key, 1)
    return
  }
  if (!hasOwn(obj, key)) {
    return
  }
  delete obj[key]
  // TODO 删除完属性之后触发依赖
}
```

**shared/util.js**

```js
// 判断是否为纯对象
export function isPlainObj (value) {
  return Object.prototype.toString.call(value) === '[object Object]'
}

// 判断是否为undefined或null
export function isUndef (value) {
  return value === undefined || value === null
}

// 是否为原始值
export function isPrimitive (value) {
  return ['string', 'number', 'boolean', 'symbol'].indexOf(typeof value) !== -1
}

// 是否为有效的数组索引 >=0 有限值 整数
export function isValidArrayIndex (value) {
  return value >= 0 && isFinite(value) && Math.floor(value) == value
}

// 判断是否为对象自身属性
export function hasOwn (obj, key) {
  return Object.prototype.hasOwnProperty.call(obj, key)
}
```

