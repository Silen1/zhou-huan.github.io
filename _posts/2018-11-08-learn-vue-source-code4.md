---
layout: post
title: "手抄Vue（四）—— 封装Observer类"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - Vue.js源码
---

`Vue.js` 中，将数据对象转化为响应式数据的是 `Observer` 构造函数。我准备结合前面几篇已经整理出来的思路，实现一个自己的 `Observer`。

为了让代码结构更加清晰，同时考虑到可复用性，我先从前面几篇已有的实现中抽一些功能较为独立的代码出来：

* `defineReactive` 方法
```js
function defineReactive(obj, key) {
  const dep = []
  let value = obj[key]
  Object.defineProperty(obj, key, {
    get () {
      dep.push(target)
      return value
    },
    set (newVal) {
      if (newVal === value) return
      value = newVal
      dep.forEach(f => {
        f()
      })
    }
  })
}
```
该方法用来将数据对象 `obj` 上的数据属性 `key` 转化为响应式属性。

`dep` 是“依赖收集器”，属性 `key` 的 `getter setter` 都通过闭包引用着自己的 `dep`。`target` 仍然作为全局变量存在，中转依赖以帮助 `getter` 收集依赖。`setter` 会执行对应 `getter` 收集到的所有依赖，但如果发现设置的值与原值无异，则直接 `return`，什么也不做。

这是直接从 [手抄Vue（一）—— 简单实现数据响应](https://zhou-huan.github.io/2018/11/04/learn-vue-source-code1/) 里拿过来的代码，但如果要封装一个功能完善、可复用性高的方法的话，肯定还要考虑一些边界条件与异常场景，比如，如果传递进来的属性本来就是不可配置的？这时就得加个判断：
```js
const property = Object.getOwnPropertyDescriptor(obj, key)
if (property && !property.configurable) {
  return
}
```
首先获取到对象 `obj` 上属性 `key` 的属性描述符对象，然后进行判断，如果属性描述符对象存在，并且该属性本来就不可配置，那么直接 `return`。

再比如，如果传进来的属性本来就有 `getter setter` 函数对 ？那就要把原来的 `getter setter` 缓存起来，在新定义的 `getter` 里除却收集依赖这项工作以外，还要将缓存起来的 `getter` 执行并将结果返回。同样，在新定义的 `setter` 里，除去执行依赖的工作以外，还要将设置的新值 `newVal` 与缓存的 `getter` 执行之后得到的值比较，如果相等则直接 `return`，什么都不做。并且要将缓存起来的 `setter` 执行一遍，以替代原来的赋值操作 `value = newVal`。

反映至代码即：
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
      dep.forEach(f => {
        f()
      })
    }
  })
}
```
上面有这么一句：
```js
if (newVal === value || (newVal !== newVal && value !== value)) {
  return
}
```
其实本来是这样的：
```js
if (newVal === value) {
  return
}
```
但是考虑到 `NaN` 的情况：
```js
NaN === NaN // false
```
这会导致：
```js
newVal === value // false
```
所以应该在判断条件中加上：
```js
newVal !== newVal && value !== value
```
利用 `NaN` 与自身不相等的特性判断出 `NaN`，最后就成了：
```js
newVal === value || (newVal !== newVal && value !== value)
```
值得注意的是：
```js
Infinity === Infinity // true
-Infinity === -Infinity // true
1 / 0 === 2 / 0 // true
```
* `walk` 方法
```js
function walk(obj) {
  const keys = Object.keys(obj)
  for (let i = 0; i < keys.length; i++) {
    defineReactive(obj, keys[i])
  }
}
```
该方法用于遍历数据对象 `obj` 的每一个属性，同时调用之前定义的 `defineReactive` 方法，将遍历到的属性转化为响应式属性。

* `hasProto`
```js
const hasProto = '__proto__' in {}
```
该变量用于判断浏览器是否支持 `__proto__` 属性。

* `arrayMethods` 对象
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
const arrayProto = Array.prototype
const arrayMethods = Object.create(arrayProto)

mutationMethods.forEach(method => {
  arrayMethods[method] = function (...args) {
    const result = arrayProto[method].apply(this, args)
    console.log(`我截获了对数组的${method}操作`)
    return result
  }
})
```
该对象用于代理数组的变异方法以实现拦截。

* `def` 方法
```js
function def(obj, key, val, enumerable) {
  Object.defineProperty(obj, key, {
    value: val,
    enumerable: !!enumerable,
    writable: true,
    configurable: true
  })
}
```
该方法是 `Object.defineProperty` 的简单封装，用于定义一个属性，可以控制该属性是否可枚举。

* `protoAugment` 方法
```js
function protoAugment(target, src) {
  target.__proto__ = src
}
```
该方法用于在浏览器支持 `__proto__` 属性时，通过修改原型链，让 `__proto__` 指向 `src`，来增强目标对象或数组。

* `copyAugment` 方法
```js
function copyAugment(target, src, keys) {
  for (let i = 0, l = keys.length; i < l; i++) {
    const key = keys[i]
    def(target, key, src[key])
  }
}
```
该方法用来遍历 `keys`，并在目标对象 `target` 上定义不可枚举的属性，该属性的键为 `keys` 中的元素，值为该元素在 `src` 中对应的属性值。

* `isPlainObject` 方法
```js
function isPlainObject(obj) {
  return Object.prototype.toString.call(obj) === '[object Object]'
}
```
该方法用于判断给定的变量是否为纯对象。

有了以上这些方法和属性之后，`Observer` 类也就应运而生了：
```js
class Observer {
  constructor (value) {
    if (Array.isArray(value)) {
      const augment = hasProto
        ? protoAugment
        : copyAugment
      augment(value, arrayMethods, mutationMethods)
    } else {
      this.walk(value)
    }
  }
  walk (obj) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }
}
```
但现在有两个问题，一个是这个类没有实现深度观测，再一个是没有对调用 `Observer` 时传进来的参数做检测，以防止传进来 `undefined null 100 'kobe'` 等等不能被观测的数据类型。并且我希望调用 `Observer` 的时候传进来的只能是数组或者纯对象。综合这些因素，再封装一层出来会比较好：
```js
function observe(value) {
  if (Array.isArray(value) || isPlainObject(value)) {
    return new Observer(value)
  }
}
```
`observe` 会判断给定的 `value` 如果是数组或者纯对象的话再去 `new` 出来 `Observer`，并将结果返回。

有了 `observe`，深度观测就可以这样来实现：在 `defineReactive` 方法中，对给定的 `obj[key]` 以及 `setter` 中的 `newVal` 调用 `observe` 方法进行观测，因为这两者都可能是数组或者纯对象，如果不是，`observe` 方法内部已经统一做了判断，外部调用时无需特殊处理。即：
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
  // 这里
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
      // 这里
      observe(newVal)
      dep.forEach(f => {
        f()
      })
    }
  })
}
```

但其实发现还有一个问题，现在数组、纯对象以及纯对象内嵌套数组、纯对象内嵌套纯对象这几种情形都已经实现了（深度）观测，但数组内嵌套纯对象以及数组内嵌套数组还没有实现，所以要再写这么一个方法：
```js
function observeArray(items) {
  for (let i = 0, l = items.length; i < l; i++) {
    observe(items[i])
  }
}
```
该方法用来遍历给定的数组，即 `items`，再分别对每一个元素 `items[i]` 执行 `observe` 方法，即可对数组里面的嵌套情形进行深度观测。同时 `Observer` 类要做以下改造：
```js
class Observer {
  constructor (value) {
    if (Array.isArray(value)) {
      const augment = hasProto
        ? protoAugment
        : copyAugment
      augment(value, arrayMethods, mutationMethods)
      // 二
      this.observeArray(value)
    } else {
      this.walk(value)
    }
  }

  walk (obj) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }

  // 一
  observeArray (items) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}
```
注释一的地方，给 `Observer` 类添加一个实例方法，也就是我刚写的 `observeArray`。

注释二的地方，调用 observeArray 方法，并将数组 value 作为参数传入。

那么最终，代码就是这个样子：
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

const hasProto = '__proto__' in {}
function isPlainObject(obj) {
  return Object.prototype.toString.call(obj) === '[object Object]'
}

function def(obj, key, val, enumerable) {
  Object.defineProperty(obj, key, {
    value: val,
    enumerable: !!enumerable,
    writable: true,
    configurable: true
  })
}

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

const arrayProto = Array.prototype
const arrayMethods = Object.create(arrayProto)
mutationMethods.forEach(method => {
  arrayMethods[method] = function (...args) {
    const result = arrayProto[method].apply(this, args)
    console.log(`我截获了对数组的${method}操作`)
    return result
  }
})

function observe(value) {
  if (Array.isArray(value) || isPlainObject(value)) {
    return new Observer(value)
  }
}

class Observer {
  constructor (value) {
    if (Array.isArray(value)) {
      const augment = hasProto
        ? protoAugment
        : copyAugment
      augment(value, arrayMethods, mutationMethods)
      this.observeArray(value)
    } else {
      this.walk(value)
    }
  }

  walk (obj) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }

  observeArray (items) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}

function protoAugment(target, src) {
  target.__proto__ = src
}
function copyAugment(target, src, keys) {
  for (let i = 0, l = keys.length; i < l; i++) {
    const key = keys[i]
    def(target, key, src[key])
  }
}

function myWatch(exp, fn) {
  target = fn
  if (typeof exp === 'function') {
    exp()
    return
  }
  let pathArr,
      obj = data
  if (/\./.test(exp)) {
    pathArr = exp.split('.')
    pathArr.forEach(p => {
      target = fn
      obj = obj[p]
    })
    return
  }
  data[exp]
}
```
添加以下测试代码：
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

function render() {
  document.body.innerText = `我最喜欢的NBA球员是${data.name}，他身高${data.otherInfo.height}cm，穿过${data.otherInfo.numbers.length}个球衣号码，${data.otherInfo.numbers[0]}和${data.otherInfo.numbers[1]}，他的队友有${data.teammates[0]}和${data.teammates[1].name}，其中，${data.teammates[1].name}在湖人时期穿的球衣号码为${data.teammates[1].numbers[1]}号`
}

observe(data)
myWatch(render, render)

data.name = 'michael'
data.otherInfo.height = 198.1
data.otherInfo.numbers.push(23)
data.teammates[1].name = 'scott pippen'
data.teammates[1].numbers.push(33)
```
执行以后发现，无论嵌套关系如何对属性的赋值操作均触发了 `render` 函数，对两个数组`data.otherInfo.numbers` 和 `data.teammates[1].numbers` 的 `push` 操作也执行了扩展的功能即打印 `'我截获了对数组的push操作'`这句信息。但是数组的 `push` 操作没有触发页面重新渲染，这是因为对数组变异方法的整个代理过程中没有收集依赖也没有触发依赖，这个问题先留下，等我写到 `Dep` 类的时候再回过头来写这个问题。但其实站在这篇博客的角度来看，`Observer` 类的封装就算是初步完成了。