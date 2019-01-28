---
layout: post
title: "JS执行上下文和变量提升"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - JavaScript
---

执行上下文（execution context）是 JavaScript 中的一个基本部分，可以大致理解为当前被执行代码的环境或者作用域。变量提升（hoisting）也是一个基本概念，简单地说就是，我可以在变量和函数声明之前就使用它们，其中具体的原因又和执行上下文有着密切的关系。

首先，请看一个例子：
```js
(function() {
  console.log(typeof team);
  console.log(typeof player);
  var team = 'lakers',
      player = function() {
        return 'kobe bryant';
      };
  function team() {
    return 'bulls';
  }
  console.log(typeof team);
  console.log(typeof player);
}());
```
试想一下这几个 `console.log` 会分别打印出来什么。

这里的答案是：
```js
// function
// undefined
// string
// function
```
如果和你的答案不一致，那么，请看下文。

要弄清楚原因，首先要明白以下几点内容：
### 第一点
JavaScript 代码执行时的执行环境有以下几种：

* 全局环境
* 函数环境
* eval

其中，全局环境为 JavaScript 代码执行时首次进入的默认环境；函数环境为每当一个函数被调用时，会进入该函数内部的一个环境；`eval` 内部的文本被执行时也会相应地产生一个执行环境（`eval` 不被推荐使用）。

### 第二点
浏览器里的 JavaScript 解释器是单线程的，同一时间里只能做一件事情。代码执行时，会产生并进入不同的执行上下文，这些执行上下文会构成一个执行上下文栈（execution context stack），这个栈的栈底永远是全局上下文，栈顶是当前正在执行的上下文。代码执行过程中，每生成一个执行上下文，都会压入执行上下文栈中，当栈顶的上下文执行完毕后，会自动出栈，控制权给到当前的栈。

一个例子：
```js
function foo() {
  console.log('我是foo里面的代码')
  function bar() {
    console.log('我是bar里面的代码')
    function baz() {
      console.log('我是baz里面的代码')
    }
    baz()
  }
  function box() {
    console.log('我是box里面的代码')
  }
  bar()
  box()
}
foo()
```
上下文的出入栈过程在该例中的表现为：
1. 全局执行上下文入栈。

2. 执行 `foo()`，创建 `foo` 对应的执行上下文，入栈。

3. 接着执行 `foo` 内部的代码，到了 `bar()`，创建 `bar` 的上下文，入栈。

4. 继续执行 `bar` 内部的代码，到了 `baz()`，创建 `baz` 的上下文，入栈。

5. `baz` 里没有再创建新的上下文，代码执行完毕之后，`baz` 的执行上下文出栈。

6. `baz` 的执行上下文出栈后，`bar` 里面也没有其它可执行代码，`bar` 的执行上下文出栈。

7. `bar` 的执行上下文出栈后，继续执行 `foo` 内部剩余可执行代码，即 `box()`，随即创建 `box` 的执行上下文，入栈。

8. `box` 内部没有再创建新的上下文，代码执行完毕之后，出栈。

这时，就只剩全局执行上下文了。

可以用 `console.trace()` 来验证一下：
```js
function foo() {
  console.log('我是foo里面的代码')
  console.trace()
  function bar() {
    console.log('我是bar里面的代码')
    console.trace()
    function baz() {
      console.log('我是baz里面的代码')
      console.trace()
    }
    baz()
  }
  function box() {
    console.log('我是box里面的代码')
    console.trace()
  }
  bar()
  box()
}
console.trace()
foo()
console.trace()
```
结果和预期是一致的：

![stack](/img/article/stack.png)

总结，关于调用栈，有几个关键点：
* 单线程。

* 同步执行，所有的执行上下文都要等到栈顶的上下文出栈以后才能依次执行。

* 唯一的全局上下文。

* 无限个数的函数上下文。

* 每一次函数被调用时都会创建新的执行上下文，包括调用自己。

### 第三点

在 JS 解释器内部，每次调用执行上下文分为两个阶段：

1. 创建阶段

这个阶段实际上处于函数被调用，但还未真正执行其内部的代码。

该阶段主要做了三件事情：

* 创建作用域链。

* 创建包含变量、函数和参数的变量对象。

  具体来说，首先，根据函数的参数创建 `arguments` 对象，初始化参数名称和值。然后，扫描上下文的函数声明，将函数名称作为属性名存入变量对象中，并且该属性指向函数在内存中的引用地址，如果函数名称已经存在，那么引用的指针将被重写。最后，扫描上下文的变量声明，将变量名称作为属性名存入变量对象中，并且将属性值初始化为 `undefined`，但是，如果变量名称已经存在于变量对象中，则对当前变量不进行任何操作，继续扫描。

  值得注意的是，解释器先扫描的是函数声明，然后才是变量声明，这和代码的书写顺序无关。

* 设置当前上下文 `this` 的值。

那么，执行上下文便可以抽象为一个对象：
```js
executionContextObj = {
  scopeChain: {...},
  variableObject: {...},
  this: {...}
}
```

2. 激活/代码执行阶段

这个阶段会逐行运行代码，过程中，设置变量的值和函数的引用，解释/执行代码。

为了进一步了解这两个阶段，可以举个例子：
```js
function player(playerTeam) {
  var player = 'kobe bryant'
  var getPlayer = function getYourPlayer() {
    return `You got ${player}!`
  }
  function getTeam() {
    return playerTeam
  }
}
player('lakers')
```
当调用 `player('lakers')` 时，进入创建阶段，创建的执行上下文抽象成对象以后应该是这样子：
```js
playerExecutionContext = {
  scopeChain: { ... },
  variableObject: {
    arguments: {
      0: 'lakers',
      length: 1
    },
    playerTeam: 'lakers',
    getTeam: pointer to function getTeam() {...}, /* 保存着一个指向函数 getTeam 的指针*/
    player: undefined,
    getPlayer: undefined
  },
  this: { ... }
}
```
之后进入激活/代码执行阶段，执行完之后，抽象出来的对象会变成这个样子：
```js
playerExecutionContext = {
  scopeChain: { ... },
  variableObject: {
    arguments: {
      0: 'lakers',
      length: 1
    },
    playerTeam: 'lakers',
    getTeam: pointer to function getTeam() {...}, // 保存着一个指向函数 getTeam 的指针
    player: 'kobe bryant',
    getPlayer: pointer to function getYourPlayer() {...} // 保存着一个指向函数 getYourPlayer 的指针
  },
  this: { ... }
}
```
了解了这些以后，文章一开始的那个例子就明朗了起来：
```js
(function() {
  console.log(typeof team); // function
  console.log(typeof player); // undefined
  var team = 'lakers',
      player = function() {
        return 'kobe bryant';
      };
  function team() {
    return 'bulls';
  }
  console.log(typeof team); // string
  console.log(typeof player); // function
}());
```
这是一个立即执行函数，执行上下文会被马上创建出来，代码中，前两个 `console.log` 执行时，其它代码还没有真正执行，那其实可以理解为前两个 `console.log` 处于创建阶段之后，以及激活/代码执行阶段之前，而后两个 `console.log` 处于激活/代码执行阶段之后。

创建阶段，解释器先扫描函数声明，变量对象中就有了 `team`，指向 `team` 函数在内存中的引用地址。然后扫描变量声明，遇到变量 `team`，发现同名属性已经存在于变量对象中，跳过，继续扫描，遇到变量 `player`，将其存入变量对象中，并赋值 `undefined`。

如此一来，第一个 `log` 打印出来的就是 `function`，第二个 `log` 打印出来的是 `undefined`。

激活/代码执行阶段，将 `team` 赋值为 `'lakers'`，将 `player` 赋值为函数表达式。

那么，第三个 `log` 打印出来的就是 `string`，第四个 `log` 打印出来的为 `function`。
