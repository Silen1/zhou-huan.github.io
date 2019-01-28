---
layout: post
title: "手抄Vue（八）—— 封装Watcher和Dep类"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - Vue.js源码
---

当我回看自己的博客时，发现有些写得很不好。拿写个更好的柯里化函数那篇来说，原本简简单单的几行代码，被我一写，成了巨长一篇，实不相瞒，连我自己都懒得看。这和杨洋大佬的博客形成了鲜明对比，也让我想起了 Linus 大神说的 —— Talk is cheap. Show me the code. 所以我想换个简洁一点的方式来记我的笔记，虽然不知道会不会矫枉过正，但，就先这么着吧。

**watcher.js**

```js
import { pushTarget } from './dep'

export default class Watcher {
  getter
  cb

  constructor (expOrFn, depFn) {
    this.cb = depFn
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    } else {
      this.getter = () => {
        if (/\./.test(expOrFn)) {
          let helpData = data
          expOrFn.split('.').forEach((path) => {
            helpData = helpData[path]
          })
          return helpData
        }
        return data[expOrFn]
      }
    }
    this.get()
  }

  get () {
    pushTarget(this)
    this.getter()
  }

  addDep (dep) {
    dep.addSub(this)
  }

  update () {
    this.getter()
    this.cb && this.cb()
  }
}
```

Watcher 的主要功能是从原来的 myWatch 方法移植过来的，即对被观察对象求值，达到触发属性 getter 并辅助收集依赖的作用。

这时调用的方式也从 `myWatch(render, render)` 变成了 `new Watcher(render)`，实例化 Watcher 时如果传递了第二个参数作为回调函数，依赖执行时会执行传入的回调函数，如果没传，实际上只执行了传入的渲染函数。

**dep.js**

```js
export default class Dep {
  subs = []

  constructor () {
  }

  depend () {
    Dep.target.addDep(this)
  }

  addSub (sub) {
    this.subs.push(sub)
  }

  notify () {
    this.subs.forEach(sub => {
      sub.update()
    })
  }
}

Dep.target = null

export function pushTarget (_target) {
  Dep.target = _target
}
```

depend 实例方法用来收集依赖，notify 实例方法用来触发依赖的执行。经过了 `depend => watcher.addDep => addSub` （watcher 表示 Watcher 的一个实例）之后，subs 中收集的依赖实际上都是 Watcher 实例，再经过 `notify => watcher.update` 之后就可以触发实例化 Watcher 时的渲染函数和回调函数（如果有）的执行了。

**observer.js**

```js
// ......
import Dep from './dep'

// ......

export function defineReactive (obj, key, val) {
  const deps = new Dep();

  // ......

  Object.defineProperty(obj, key, {
    get () {
      // ......
      // 替代原来收集依赖的方式
      deps.depend()
      // ......
    },
    set (newVal) {
      // ......
      // 替代原来触发依赖执行的方式
      deps.notify()
    }
  })
}
```
相应地，在Object.defineProperty 方法定义的 getter 和 setter 中使用 Dep 类来修改一下收集和触发依赖的方式。
