---
layout: post
title: "JavaScript柯里化 —— 实现lodash的curry方法[译]"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - JavaScript
---

> 当我读到 [Eric Elliott](https://medium.com/@_ericelliott) 在 [Medium](https://medium.com/javascript-scene/master-the-javascript-interview-what-is-function-composition-20dfb109a1a0) 上写的关于组合函数的文章时，我对于他 curry 函数的实现感到大惑不解，这看起来像是对  lodash.js 中 curry 方法的一个简单模仿，并且他是用 ES6 写的。

```js
const curry = fn => (…args) => fn.bind(null, …args);
```

为了帮助其他开发人员理解这行代码背后究竟发生了什么，我决定来写这篇文章，尽力使用一些非常简单和直观，甚至稍显稚拙的例子来一步步地阐明 curry 函数的基本实现。

让我们先看一下 lodash.js 的[文档](https://lodash.com/docs/4.17.11#curry)，看看一个真正的 curry 方法到底是做什么的。

```js
var abc = function(a, b, c) { return [a, b, c];};
var curried = _.curry(abc);
curried(1)(2)(3); // => [1, 2, 3]
curried(1, 2)(3); // => [1, 2, 3]
curried(1, 2, 3); // => [1, 2, 3]
// Curried with placeholders.
curried(1)(_, 3)(2); // => [1, 2, 3]
```

在我理解看来，curry 能够让我们：

1. 在多个函数调用中逐步收集参数，不用在一个函数调用中一次收集。

2. 当收集到足够的参数时，返回函数执行结果。

为了更好的理解它，我在网上找了多个实现示例。然而，我希望是有一个非常简单的教程从一个基本的例子开始，就像下面这个一样，而不是直接从最终的实现开始。

```js
var fn = function() {
  console.log(arguments);
  return fn.bind(null, ...arguments);
  // 如果没有es6的话我们可以这样写：
  // return Function.prototype.bind.apply(fn, [null].concat(
  //   Array.prototype.slice.call(arguments)
  // ));
}

fb = fn(1); //[1]
fb = fb(2); //[1, 2]
fb = fb(3); //[1, 2, 3]
fb = fb(4); //[1, 2, 3, 4]
```

理解 `fn` 函数是所有的起点。基本上，这个函数的作用就是一个“**参数收集器**”。每次调用该函数时，它都会返回一个自身的[绑定函数](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind)（`fb`），并且将该函数提供的“参数”绑定到返回函数上。该“参数”将位于之后调用返回的绑定函数时提供的任何参数之前。因此，每个调用中传的参数将被逐渐收集到一个数组当中。

当然，就像 curry 函数一样，我们不必一直收集下去。现在我们可以先写死一个终止点。

```js
var numOfRequiredArguments = 5;
var fn = function() {
  if (arguments.length < numOfRequiredArguments) {
    return fn.bind(null, ...arguments);
  } else {
    console.log('we already collect 5 arguments: ', [...arguments]);
    return null;
  }
}
```

为了让它表现得和 curry 方法一样，需要解决两个问题：

1. 我们希望将收集到的参数传递给需要它们的目标函数，而不是通过将它们传递给 `console.log` 在最后打印出来。

2. 变量 `numOfRequiredArguments` 不应该是写死的，它应该是目标函数所期望的参数个数。

幸运的是，JavaScript函数确实带有一个名为 “length” 的属性，它指定了函数所期望的参数个数。因此，我们就可以使用这个属性来确定所需要的参数个数，而不用再写死了。那么第二个问题就解决了。

那第一个问题呢：保持对目标函数的引用？

网上有几个例子可以解决这个问题。它们之间虽然略有不同，但是有着相同的思路：除去存储参数以外，我们还需要在某处存储对于目标函数的引用。

这里我把它们分为两种不同的方法，它们之间或多或少都有相似之处，理解它们能够帮助我们更好地理解背后的逻辑。顺便说一句，这里我将这个函数叫做 `magician`，以代替 curry。

### 方法1

```js
function magician(targetfn) {
  var numOfArgs = targetfn.length;
  return function fn() {
    if (arguments.length < numOfArgs) {
      return fn.bind(null, ...arguments);
    } else {
      return targetfn.apply(null, arguments);
    }
  }
}
```

`magician` 函数的作用是：它接收目标函数作为参数，然后返回‘参数收集器’函数，与上例中 `fn` 函数作用相同。唯一的不同点在于，当收集的参数数量与目标函数**所必需**的参数数量相等时，它将把收集到的参数通过 `apply` 方法给到该目标函数，并返回计算的结果。这个方法通过将其存储在 `magician` 创建的闭包当中来解决第一个问题（引用目标函数）。

### 方法2

这个方法更进一步，由于参数收集器函数只是一个普通函数，那为什么不使用 `magician` 函数本身作为参数收集器呢？

```js
function magician (targetfn) {
  var numOfArgs = targetfn.length;
  if (arguments.length - 1 < numOfArgs) {
    return magician.bind(null, ...arguments);
  } else {
    return targetfn.apply(null, Array.prototype.slice.call(arguments, 1));
  }
}
```

注意方法2中的一个不同。因为 `magician` 接收目标函数作为它的第一个参数，因此收集到的参数将始终包含该函数作为 `arguments[0]`。这就导致，我们在检查有效参数的总数时，需要减去第一个参数。

顺便说一句，因为目标函数是递归地传递给 `magician` 函数的，所以我们可以通过传入第一个参数显式地引用目标函数，以代替使用闭包来存储目标函数的引用。

正如你所见，[Eric Elliott](https://medium.com/@_ericelliott) 上面使用到的 “curry” 函数和方法1功能相似，但实际上它是一个[偏函数](https://medium.com/javascript-scene/curry-or-partial-application-8150044c78b8)（这又是另外一说了）。

```js
const curry = fn => (…args) => fn.bind(null, …args);
```

上面是一个 curry 函数，它返回“参数收集器”，该收集器只收集一次参数，并返回绑定的目标函数。

### 更进一步

上面的‘magician’函数仍然没有lodash.js中的‘curry’函数那样神奇。lodash的curry允许使用‘_’作为输入参数的占位符。

```js
curried(1)(_, 3)(2); // => [1, 2, 3], 注意占位符 '_'
```

为了实现占位符功能，有一个隐含的需求：我们需要知道哪些参数被预设给了绑定函数，以及哪些是在调用函数时显示提供的附加参数（这里我们称之为added参数）。

这个功能可以通过创建另外一个闭包来完成：

```js
function fn2() {
  var preset = Array.prototype.slice.call(arguments);
  /*
    原先是这样:
    return fn.bind(null, ...arguments);
  */
  return function helper() {
    var added = Array.prototype.slice.call(arguments);
    return fn2.apply(null, [...preset, ...added]); //简单起见，使用es6
  }
}
```

上面的 `fn2` 几乎和 `fn` 一样，功能就像‘参数收集器’一样。然而，`fn2` 不是直接返回绑定函数，而是返回一个中间辅助函数 `helper`。`helper` 函数是未绑定的，因此它可以用来分离预设的参数和后来提供的参数。

当然，我们需要在组合时进行一些修改，而不是通过 `[...preset, ...added]` 将预设的参数和后来提供的参数合并起来。我们需要在preset参数中找到占位符的位置，并用有效的added参数替换它。我没有看lodash是如何实现它的，但下面是一个完成类似功能的简单实现。

```js
// 定义占位符
var _ = '_';

function magician3 (targetfn, ...preset) {
  var numOfArgs = targetfn.length;
  var nextPos = 0; // 下一个有效输入位置的索引，可以是'_'，也可以是preset的结尾

  // 查看是否有足够的有效参数
  if (preset.filter(arg=> arg !== _).length === numOfArgs) {
    return targetfn.apply(null, preset);
  } else {
    // 返回'helper'函数
    return function (...added) {
      // 循环并将added参数添加到preset参数
      while(added.length > 0) {
        var a = added.shift();
        // 获取下一个占位符的位置，可以是'_'也可以是preset的末尾
        while (preset[nextPos] !== _ && nextPos < preset.length) {
          nextPos++
        }
        // 更新preset
        preset[nextPos] = a;
        nextPos++;
      }
      // 绑定更新后的preset
      return magician3.call(null, targetfn, ...preset);
    }
  }
}
```

第15到24行是用于将added参数放入preset数组中正确位置的逻辑：无论是占位符或是preset的结尾。该位置被标记为 `nextPos` 并初始化为索引0。

现在，函数 `magician3` 几乎已经和lodash的curry函数功能相当了。

译自：[Implementation of lodash ‘curry’ function](https://medium.com/@kj_huang/implementation-of-lodash-curry-function-8b1024d71e3b)（如有不准确之处，敬请指正，不胜感激）