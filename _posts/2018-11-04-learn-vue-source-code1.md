---
layout: post
title: "手抄Vue（一）—— 简单实现数据响应"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - Vue.js源码
---

Vue 中可以用 [`$watch`](https://cn.vuejs.org/v2/api/#vm-watch) 实例方法观察一个字段，当该字段的值发生变化时，会执行指定的回调函数（即观察者），实际上和 `watch` 选项作用相同。如下：
```js
vm.$watch('box', () => {
    console.log('box变了')
})
vm.box = 'newValue' // 'box变了'
```

以上例切入，我想实现一个功能类似的方法 `myWatch`。

### 如何知道我观察的属性被修改了？

—— [`Object.defineProperty`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty) 方法

该方法可以为指定对象的指定属性设置 `getter-setter` 函数对，通过这对 `getter-setter` 可以捕获到对属性的读取和修改操作。示例如下：
```js
const data = {
  box: 1
}
Object.defineProperty(data, 'box', {
  set () {
    console.log('修改了 box')
  },
  get () {
    console.log('读取了 box')
  }
})

console.log(data.box) // '读取了 box'
                      // undefined
data.box = 2   // '修改了 box'
console.log(data.box) // '读取了 box'
                      // undefined
```
如此，便拦截到了对 `box` 属性的修改和读取操作。

但 `res` 为 `undefined`，`data.box = 2` 的修改操作也无效。
### `get` 与 `set` 函数功能不健全
故修改如下：
```js
const data = {
  box: 1
}
let value = data.box
Object.defineProperty(data, 'box', {
  set (newVal) {
    if (newVal === value) return
    value = newVal
    console.log('修改了 box')
  },
  get () {
    console.log('读取了 box')
    return value
  }
})

console.log(data.box) // '读取了 box'
                      // 1

data.box = 2 // '修改了 box'
console.log(data.box) // '读取了 box'
                      // 2
```
有了这些， `myWatch` 方法便可实现如下：
```js
const data = {
  box: 1
}
function myWatch(key, fn) {
  let value = data[key]
  Object.defineProperty(data, key, {
    set (newVal) {
      if (newVal === value) return
      value = newVal
      fn()
    },
    get () {
      return value
    }
  })
}
myWatch('box', () => {
    console.log('box变了')
})

data.box = 2 // 'box变了'
```
但存在一个问题，不能给同一属性添加多个依赖（观察者）：
```js
myWatch('box', () => {
  console.log('我是观察者')
})
myWatch('box', () => {
  console.log('我是另一个观察者')
})

data.box = 2 // '我是另一个观察者'
```
后面的依赖（观察者）会将前者覆盖掉。
### 如何能够添加多个依赖（观察者）？
—— 定义一个数组，作为依赖收集器：
```js
const data = {
  box: 1
}
const dep = []
function myWatch(key, fn) {
  dep.push(fn)
  let value = data[key]
  Object.defineProperty(data, key, {
    set (newVal) {
      if (newVal === value) return
      value = newVal
      dep.forEach((f) => {
        f()
      })
    },
    get () {
      return value
    }
  })
}

myWatch('box', () => {
  console.log('我是观察者')
})
myWatch('box', () => {
  console.log('我是另一个观察者')
})

data.box = 2 // '我是观察者'
             // '我是另一个观察者'
```
修改 `data.box` 后，两个依赖（观察者）都执行了。

若上例 `data` 对象需新增两个能够响应数据变化的属性 `foo bar`：
```js
const data = {
  box: 1,
  foo: 1,
  bar: 1
}
```
只需执行以下代码即可：
```js
myWatch('foo', () => {
  console.log('我是foo的观察者')
})
myWatch('bar', () => {
  console.log('我是bar的观察者')
})
```
但问题是，不同属性的依赖（观察者）都被收集进了同一个 `dep`，修改任何一个属性，都会触发所有的依赖（观察者）：
```js
data.box = 2 // '我是观察者'
             // '我是另一个观察者'
             // '我是foo的观察者'
             // '我是bar的观察者'
```
我想可以这样解决：
```js
const data = {
  box: 1,
  foo: 1,
  bar: 1
}
const dep = {}
function myWatch(key, fn) {
  if (!dep[key]) {
    dep[key] = [fn]
  } else {
    dep[key].push(fn)
  }
  let value = data[key]
  Object.defineProperty(data, key, {
    set (newVal) {
      if (newVal === value) return
      value = newVal
      dep[key].forEach((f) => {
        f()
      })
    },
    get () {
      return value
    }
  })
}

myWatch('box', () => {
  console.log('我是box的观察者')
})
myWatch('box', () => {
  console.log('我是box的另一个观察者')
})
myWatch('foo', () => {
  console.log('我是foo的观察者')
})
myWatch('bar', () => {
  console.log('我是bar的观察者')
})

data.box = 2 // '我是box的观察者'
             // '我是box的另一个观察者'
data.foo = 2 // '我是foo的观察者'
data.bar = 2 // '我是bar的观察者'
```
但实际上这样更好些：
```js
const data = {
  box: 1,
  foo: 1,
  bar: 1
}
let target = null
for (let key in data) {
  const dep = []
  let value = data[key]
  Object.defineProperty(data, key, {
    set (newVal) {
      if (newVal === value) return
      value = newVal
      dep.forEach(f => {
        f()
      })
    },
    get () {
      dep.push(target)
      return value
    }
  })
}
function myWatch(key, fn) {
  target = fn
  data[key]
}
myWatch('box', () => {
  console.log('我是box的观察者')
})
myWatch('box', () => {
  console.log('我是box的另一个观察者')
})
myWatch('foo', () => {
  console.log('我是foo的观察者')
})
myWatch('bar', () => {
  console.log('我是bar的观察者')
})

data.box = 2 // '我是box的观察者'
             // '我是box的另一个观察者'
data.foo = 2 // '我是foo的观察者'
data.bar = 2 // '我是bar的观察者'
```
声明 `target` 全局变量作为依赖（观察者）的中转站，`myWatch` 函数执行时用 `target` 缓存依赖，然后调用 `data[key]` 触发对应的 `get` 函数以收集依赖，`set` 函数被触发时会将 `dep` 里的依赖（观察者）都执行一遍。这里的 `get set` 函数形成闭包引用了上面的 `dep` 常量，这样一来，`data` 对象的每个属性都有了对应的依赖收集器。

且这一实现方式不需要通过 `myWatch` 函数显式地将 `data` 里的属性一一转为访问器属性。

但运行以下代码，会发现仍有问题：
```js
console.log(data.box)
data.box = 2 // '我是box的观察者'
             // '我是box的另一个观察者'
             // '我是bar的观察者'
```
四个 `myWatch` 执行完之后 `target` 缓存的值变成了最后一个 `myWatch` 方法调用时所传递的依赖（观察者），故执行 `console.log(data.box)` 读取 `box` 属性的值时，会将最后缓存的依赖存入 `box` 属性所对应的依赖收集器，故而再修改 `box` 的值时，会打印出 `'我是bar的观察者'`。

我想可以在每次收集完依赖之后，将全局变量 `target` 设置为空函数来解决这问题：
```js
const data = {
  box: 1,
  foo: 1,
  bar: 1
}
let target = null
for (let key in data) {
  const dep = []
  let value = data[key]
  Object.defineProperty(data, key, {
    set (newVal) {
      if (newVal === value) return
      value = newVal
      dep.forEach(f => {
        f()
      })
    },
    get () {
      dep.push(target)
      target = () => {}
      return value
    }
  })
}
function myWatch(key, fn) {
  target = fn
  data[key]
}
myWatch('box', () => {
  console.log('我是box的观察者')
})
myWatch('box', () => {
  console.log('我是box的另一个观察者')
})
myWatch('foo', () => {
  console.log('我是foo的观察者')
})
myWatch('bar', () => {
  console.log('我是bar的观察者')
})
```
经测无误。

但开发过程中，还常碰到需观测嵌套对象的情形：
```js
const data = {
  box: {
    gift: 'book'
  }
}
```
这时，上述实现未能观测到 `gift` 的修改，显出不足。
### 如何进行深度观测？
——递归

通过递归将各级属性均转为响应式属性即可：
```js
const data = {
  box: {
    gift: 'book'
  }
}
let target = null
function walk(data) {
  for (let key in data) {
    const dep = []
    let value = data[key]
    if (Object.prototype.toString.call(value) === '[object Object]') {
      walk(value)
    }
    Object.defineProperty(data, key, {
      set (newVal) {
        if (newVal === value) return
        value = newVal
        dep.forEach(f => {
          f()
        })
      },
      get () {
        dep.push(target)
        target = () => {}
        return value
      }
    })
  }
}
walk(data)
function myWatch(key, fn) {
  target = fn
  data[key]
}

myWatch('box', () => {
  console.log('我是box的观察者')
})
myWatch('box.gift', () => {
  console.log('我是gift的观察者')
})

data.box = {gift: 'basketball'} // '我是box的观察者'
data.box.gift = 'guitar'
```
这时 `gift` 虽已是访问器属性，但 `myWatch` 方法执行时 `data[box.gift]` 未能触发相应 `getter` 以收集依赖， `data[box.gift]` 访问不到 `gift` 属性，`data[box][gift]` 才可以，故 `myWatch` 须改写如下：
```js
function myWatch(exp, fn) {
  target = fn
  let pathArr,
      obj = data
  if (/\./.test(exp)) {
    pathArr = exp.split('.')
    pathArr.forEach(p => {
      obj = obj[p]
    })
    return
  }
  data[exp]
}
```
如果要读取的字段包括 `.` ，那么按照 `.` 将其分为数组，然后使用循环读取嵌套对象的属性值。

这时执行代码后发现，`data.box.gift = 'guitar'` 还是未能触发相应的依赖，即打印出 `'我是gift的观察者'` 这句信息。调试之后找到问题：
```js
myWatch('box.gift', () => {
  console.log('我是gift的观察者')
})
```
执行以上代码时，`pathArr` 即 `['box', 'gift']`，循环内 `obj = obj[p]` 实际上就是 `obj = data[box]`，读取了一次 `box`，触发了 `box` 对应的 `getter`，收集了依赖：
```js
() => {
  console.log('我是gift的观察者')
}
```
收集完将全局变量 `target` 置为空函数，而后，循环继续执行，又读取了 `gift` 的值，但这时，`target`  已是空函数，导致属性 `gift` 对应的 `getter` 收集了一个“空依赖”，故，`data.box.gift = 'guitar'` 的操作不能触发期望的依赖。

以上代码有两个问题：

* 修改 `box` 会触发“我是gift的观察者”这一依赖
* 修改 `gift` 未能触发“我是gift的观察者”的依赖

第一个问题，读取 `gift` 时，必然经历读取 `box` 的过程，故触发 `box` 对应的 `getter` 无可避免，那么，`box` 对应 `getter` 收集 `gift` 的依赖也就无可避免。但想想也算合理，因为 `box` 修改时，隶属于 `box` 的 `gift` 也算作修改，从这一点看，问题一也不算作问题，划去。

第二个问题，我想可以这样解决：
```js
function myWatch(exp, fn) {
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
  target = fn
  data[exp]
}

data.box.gift = 'guitar' // '我是gift的观察者'
data.box = {gift: 'basketball'} // '我是box的观察者'
                                // '我是gift的观察者'
```
保证属性读取时 `target = fn` 即可。

那么：
```js
const data = {
  box: {
    gift: 'book'
  }
}
let target = null
function walk(data) {
  for (let key in data) {
    const dep = []
    let value = data[key]
    if (Object.prototype.toString.call(value) === '[object Object]') {
      walk(value)
    }
    Object.defineProperty(data, key, {
      set (newVal) {
        if (newVal === value) return
        value = newVal
        dep.forEach(f => {
          f()
        })
      },
      get () {
        dep.push(target)
        target = () => {}
        return value
      }
    })
  }
}
walk(data)
function myWatch(exp, fn) {
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
  target = fn
  data[exp]
}

myWatch('box', () => {
  console.log('我是box的观察者')
})
myWatch('box.gift', () => {
  console.log('我是gift的观察者')
})
```
现在我想，假如我有以下数据：
```js
const data = {
  player: 'James Harden',
  team: 'Houston Rockets'
}
```
执行以下代码：
```js
function render() {
  document.body.innerText = `The last season's MVP is ${data.player}, he's from ${data.team}`
}
render()
myWatch('player', render)
myWatch('team', render)

data.player = 'Kobe Bryant'
data.team = 'Los Angeles Lakers'
```
是不是就可以将数据映射到页面，并响应数据的变化？

执行代码发现，`data.player = 'Kobe Bryant'` 报错，究其原因，`render` 方法执行时，会去获取 `data.player` 和 `data.team` 的值，但此时，`target` 为 `null`，那么读取 `player` 时对应的依赖收集器 `dep` 便收集了 `null`，导致 `player` 的 `setter` 调用依赖时报错。

那么我想，在 `render` 执行时便主动去收集依赖，就不会导致 `dep` 里收集了 `null`。

细看 `myWatch`，这方法做的事情其实就是帮助 `getter` 收集依赖，它的第一个参数就是要访问的属性，要触发谁的 `getter`，第二个参数是相应要收集的依赖。

这么看来，`render` 方法既可以帮助 `getter` 收集依赖（`render` 执行时会读取 `player team`），而且它本身就是要收集的依赖。那么，我能不能修改一下 `myWatch` 的实现，以支持这样的写法：
```js
myWatch(render, render)
```
第一个参数作为函数执行一下便有了之前第一个参数的作用，第二个参数还是需要被收集的依赖，嗯，想来合理。

那么，`myWatch` 改写如下：
```js
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
但，对 `team` 的修改未能触发页面更新，想来因为 `render` 执行读取 `player` 收集依赖后 `target` 变为空函数，导致读取 `team` 收集依赖时收集到了空函数。这里大家的依赖都是 `render`，故可将 `target = () => {}` 这句删去。

`myWatch` 这样实现还有个好处，假如 `data` 中有许多属性都需要通过 `render` 渲染至页面，一句 `myWatch(render, render)` 便可，无须如此这般繁复：
```js
myWatch('player', render)
myWatch('team', render)
myWatch('number', render)
myWatch('height', render)
...
```
那么最终：
```js
const data = {
  player: 'James Harden',
  team: 'Houston Rockets'
}
let target = null
function walk(data) {
  for (let key in data) {
    const dep = []
    let value = data[key]
    if (Object.prototype.toString.call(value) === '[object Object]') {
      walk(value)
    }
    Object.defineProperty(data, key, {
      set (newVal) {
        if (newVal === value) return
        value = newVal
        dep.forEach(f => {
          f()
        })
      },
      get () {
        dep.push(target)
        return value
      }
    })
  }
}
walk(data)
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
function render() {
  document.body.innerText = `The last season's MVP is ${data.player}, he's from ${data.team}`
}

myWatch(render, render)
```
