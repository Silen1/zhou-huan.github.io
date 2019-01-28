---
layout: post
title: "关于金额格式化"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - JavaScript
---

金额格式化，也就是数字加逗号，有这么几个方法：

#### 第一招 朴实无华

每隔三位加个逗号，纯手动操作，稍显驽钝。

```js
const price = 399999

const priceArr = String(price).split('').reverse()
let res = ''
priceArr.forEach((item, index) => {
  if (index % 3 === 0 && index) {
    res = item + ',' + res
  } else {
    res = item + res
  }
})
console.log(res) // 399,999
```

#### 第二招 锐气初现

可以用 `Number` 的 `toLocaleString` 方法替换手动加逗号的过程。

```js
const price = 399999

const res = price.toLocaleString('en')
console.log(res) // 399,999
```

`toLocaleString` 方法也可以处理小数：

```js
const price = 399999.99

const res = price.toLocaleString('en')
console.log(res) // 399,999.99
```

这个方法虽然简洁的多，但是如果传递参数的话，PC端IE浏览器需要11以上，兼容性略显不足（所有现代浏览器都是支持传递参数的，所以移动端没有问题的）。

#### 第三招 气贯长虹

可以写个正则来完成格式化，相较于前两个方法，正则兼具了简洁性与良好的兼容性。

```js
const price = 399999

const res = String(price).replace(/(?=(\d{3})+$)(?!^)/g, ',')
console.log(res) // 399,999
```

#### 最后

我们可以据此写一个更通用的方法，不管是不是小数，不管是数字还是字符串，如果是字符串的话也不管除了数字有没有其它字符，都可以一并进行处理。

```js
function formatPrice (value) {
  if (!value) {
    return 0
  }
  value = String(value)
  const intPrice = value.match(/\d+/)[0]
  const HOLDER = 'HOLDER'
  const holder = value.replace(intPrice, HOLDER)
  value = holder.replace(HOLDER, intPrice.replace(/(?=(\d{3})+$)(?!^)/g, ','))
  return value
}
```

| input | output |
| ------ | ------ |
| 2689999 | 2,689,999 |
| '2689999' | 2,689,999 |
| '2689999元' | 2,689,999元 |
| '裸车价：￥2689999' | 裸车价：￥2,689,999 |
| '优惠：2689999元' | 优惠：2,689,999元 |
| '到手价：2689999.99元' | 到手价：2,689,999.99元 |
| '' | 0 |
|  | 0 |

#### 关于第二招的一点说明

第二个方法中，`toLocaleString` 如果不传参数的话，会使用宿主环境的系统语言进行分位处理。但各语言的处理形式可能是不一样的，比如印度是这样的：9,00,00,000.00，小数点左侧首先是三位分隔，然后继续向左都是两位分隔。

所以如果宿主环境的系统语言为印地语，那 `price` 为 `90000000` 时， `price.toLocaleString()` 就会输出 `"9,00,00,000"`。

可以使用可选参数 `locales` 来进行模拟：

```js
var price = 90000000
price.toLocaleString('en-IN') // "9,00,00,000"
```

那么为了保证在任何地区输出的格式一致，需要加上 `locales` 参数：

```js
price.toLocaleString('en')
```

`toLocaleString` 方法还有第二个可选参数 `options` 可以用来配置更复杂的格式化方式，具体可以查看 [MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Number/toLocaleString)。

#### 关于第三招的一点说明

代码的正则中，`(?=(\d{3})+$)` 这部分为正向先行断言，`(?!^)` 这部分为负向先行断言。

正则表达式中先行断言和后行断言一共有4种形式：

1. `(?=pattern)` 零宽正向先行断言(zero-width positive lookahead assertion)，用来匹配字符串中的一个位置，该位置之后紧接着的的字符能够匹配到pattern。

2. `(?!pattern)` 零宽负向先行断言(zero-width negative lookahead assertion)，用来匹配字符串中的一个位置，该位置之后紧接着的的字符不能匹配到pattern。

3. `(?<=pattern)` 零宽正向后行断言(zero-width positive lookbehind assertion)，用来匹配字符串中的一个位置，该位置之前紧接着的的字符能够匹配到pattern。

4. `(?<!pattern)` 零宽负向后行断言(zero-width negative lookbehind assertion)，用来匹配字符串中的一个位置，该位置之前紧接着的的字符不能匹配到pattern。

零宽可以理解为不占宽度，只是用来匹配一个位置。

举个例子：

```js
const me = 'i was born in 1994, 24 years old, my height is 181.5cm'
```

要匹配出所有的数字，即 `1994 24 181 5`，可以这么写正则：

```js
const reg = /\d+/g
me.match(reg) // ["1994", "24", "181", "5"]
```

但如果要匹配出 `1994 24 181`，也就是所有的整数，以及小数的整数部分，就要使用零宽负向后行断言来过滤掉 `5`：

```js
const reg = /(?<!\.)\d+/g
me.match(reg) // ["1994", "24", "181"]
```

如果要匹配出 `1994 24`，也就是所有的整数，过滤小数，可以使用零宽负向后行断言和零宽负向先行断言来过滤掉 `181` 和 `5`：

```js
const reg = /(?<!\.)\d+(?!\d*\.)/g
me.match(reg) // ["1994", "24"]
```
