---
layout: post
title: "Vue内部怎样处理props选项的多种写法"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - Vue.js源码
---

开发过程中，`props` 的使用有两种写法：
```js
// 字符串数组写法
const subComponent = {
  props: ['name']
}
```
```js
// 对象写法
const subComponent = {
  props: {
    name: {
      type: String,
      default: 'Kobe Bryant'
    }
  }
}
```
Vue在内部会对 `props` 选项进行处理，无论开发时使用了哪种语法，Vue都会将其规范化为对象的形式。具体规范方式见[Vue源码](https://github.com/vuejs/vue/blob/dev/src/core/util/options.js) `src/core/util/options.js` 文件中的 `normalizeProps` 函数：
```js
/**
 * Ensure all props option syntax are normalized into the
 * Object-based format.（确保将所有props选项语法规范为基于对象的格式）
 */
 // 参数的写法为 flow(https://flow.org/) 语法
function normalizeProps (options: Object, vm: ?Component) {
  const props = options.props
  // 如果选项中没有props，那么直接return
  if (!props) return
  // 如果有，开始对其规范化
  // 声明res，用于保存规范化后的结果
  const res = {}
  let i, val, name
  if (Array.isArray(props)) {
    // 使用字符串数组的情况
    i = props.length
    // 使用while循环遍历该字符串数组
    while (i--) {
      val = props[i]
      if (typeof val === 'string') {
        // props数组中的元素为字符串的情况
        // camelize方法位于 src/shared/util.js 文件中，用于将中横线转为驼峰
        name = camelize(val)
        res[name] = { type: null }
      } else if (process.env.NODE_ENV !== 'production') {
        // props数组中的元素不为字符串的情况，在非生产环境下给予警告
        // warn方法位于 src/core/util/debug.js 文件中
        warn('props must be strings when using array syntax.')
      }
    }
  } else if (isPlainObject(props)) {
    // 使用对象的情况（注）
    // isPlainObject方法位于 src/shared/util.js 文件中，用于判断是否为普通对象
    for (const key in props) {
      val = props[key]
      name = camelize(key)
      // 使用for in循环对props每一个键的值进行判断，如果是普通对象就直接使用，否则将其作为type的值
      res[name] = isPlainObject(val)
        ? val
        : { type: val }
    }
  } else if (process.env.NODE_ENV !== 'production') {
    // 使用了props选项，但它的值既不是字符串数组，又不是对象的情况
    // toRawType方法位于 src/shared/util.js 文件中，用于判断真实的数据类型
    warn(
      `Invalid value for option "props": expected an Array or an Object, ` +
      `but got ${toRawType(props)}.`,
      vm
    )
  }
  options.props = res
}
```
如此一来，假如我的 `props` 是一个字符串数组：
```js
props: ["team"]
```
经过这个函数之后，`props` 将被规范为：
```js
props: {
  team:{
    type: null
  }
}
```
假如我的 `props` 是一个对象：
```js
props: {
  name: String,
  height: {
    type: Number,
    default: 198
  }
}
```
经过这个函数之后，将被规范化为：
```js
props: {
  name: {
    type: String
  },
  height: {
    type: Number,
    default: 198
  }
}
```
***
注：对象的写法也分为以下两种，故仍需进行规范化
```js
props: {
  // 第一种写法，直接写类型
  name: String,
  // 第二种写法，写对象
  name: {
    type: String,
    default: 'Kobe Bryant'
  }
}
```
最终会被规范为第二种写法。
