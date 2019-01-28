---
layout: post
title: "JavaScript事件循环机制[译]"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - JavaScript
---

如果要学习并理解JavaScript，事件循环可以说是最重要的方面之一。这篇文章简单地对事件循环机制进行了解释。

### 介绍

**事件循环**是理解JavaScript的最重要方面之一。

> 我已经用JavaScript编程了多年，但是，我从来没有完全真正理解它是怎么工作的。没有详细了解这个概念也是完全没问题的，但通常情况下，了解它的工作方式会很有帮助，而且在这一点上你可能也会有些好奇。

这篇文章旨在解释JavaScript单线程工作的一些内部细节，以及，它是如何处理异步函数的。

JavaScript代码单线程运行，一次只会发生一件事情。

这是一个实际上非常有用的限制，因为它很大程度上简化了你的编程方式，实际编写程序时不必担心并发问题。

你只需要注意编写代码的方式，避免任何可能阻塞线程的东西，比如同步网络请求或者无限循环。

通常情况下，在大多数浏览器中，每个标签页都有一个事件循环，以使每个进程隔离并避免具有无限循环或繁重处理任务的页面阻塞整个浏览器。

浏览器环境管理着多个并发的事件循环，比如API调用。Web Workers也在自己的事件循环当中运行。

你主要需要担心你的代码将在单个的事件循环上运行，并在编写代码时要考虑到这一点，以避免阻塞它。

### 阻塞事件循环

任何花费过长时间将控制权交回给事件循环的JavaScript代码都将阻塞页面中其余所有JavaScript代码的执行，甚至阻塞UI线程，用户也就无法进行点击或滚动页面等操作。

JavaScript中几乎所有的I/O原语都是非阻塞的，网络请求，Node.js文件系统操作等。阻塞是例外，这就是为什么JavaScript很大程度上基于回调，还有最近的promises和async/await。

### 调用栈

调用栈是一个LIFO队列（Last In, First Out）。

事件循环不断地检查**调用栈**，以查看是否存在需要运行的函数。

在执行此操作时，它会将它找到的所有函数调用添加到调用栈，并按顺序一一执行。

你可以在调试器或浏览器控制台中了解你可能已经熟悉的错误栈追踪，浏览器在调用栈中查找函数名称，以通知你是哪个函数发起的当前调用。

![exception-call-stack](/img/article/exception-call-stack.png)

### 一个简单的事件循环说明

我们来举一个例子：
```js
const bar = () => console.log('bar')

const baz = () => console.log('baz')

const foo = () => {
  console.log('foo')
  bar()
  baz()
}

foo()
```
这段代码打印出了：
```
foo
bar
baz
```
和期望结果一致。

当这段代码运行时，首先调用`foo()`。在`foo()`里面，我们先调用了`bar()`，然后调用了`baz()`。

那么这时，调用栈如下所示：

![call-stack-first-example](/img/article/call-stack-first-example.png)

每次重复的事件循环都会去查看调用栈中是否有内容，有的话就去执行它：

![execution-order-first-example](/img/article/execution-order-first-example.png)

直到调用栈为空。

### 让*函数执行*排队

上面的例子看起来很正常，没有什么特别之处；JavaScript找到要执行的东西，然后按顺序运行他们。

让我们来看看如何将函数推迟至直到栈被清空。

`setTimeout(() => {}), 0)` 的使用场景是调用一个函数，该函数的调用时机是一旦代码中的其它函数都执行完，马上调用该函数。

举个例子：
```js
const bar = () => console.log('bar')

const baz = () => console.log('baz')

const foo = () => {
  console.log('foo')
  setTimeout(bar, 0)
  baz()
}

foo()
```
这段代码也许会令人惊讶的打印出来：
```
foo
baz
bar
```
这段代码运行时，首先调用了`foo()`，在`foo()`里面我们首先调用`setTimeout`，将`bar`作为一个参数传递进去，并且我们指示它尽可能快地运行，将0作为计时器的时间传递进去。然后我们调用了`baz()`。

那么这时，调用栈看起来就是这个样子：

![call-stack-second-example](/img/article/call-stack-second-example.png)

以下是我们程序中所有函数的执行顺序：

![execution-order-second-example](/img/article/execution-order-second-example.png)

为什么会这样？

### 消息队列

当我们调用`setTimeout()`时，浏览器或Node.js立即启动计时器。一旦计时器结束（在该例中我们传递了0作为计时器的到期时间，也就意味着“立即”），回调函数就会被立即放入**消息队列**中。

消息队列也可以是用户启动的事件，比如点击操作、键盘事件或者获取响应，在代码有机会对其作出反应之前它们统统都会被放在队列里面。还有像onLoad这样的DOM事件。

**循环的优先权在调用栈，它会首先处理在调用栈中找到的所有内容，一旦都处理完了，它就会去事件队列中拾取内容。**

我们不必等待像setTimeout，fetch或其它的一些函数来做它们自己的工作，因为它们是由浏览器提供的，并且它们存在于自己的线程当中。举例来说，如果你将setTimeout的超时时间设置为2秒，你不必真的等待2秒 —— 等待发生在其它地方。

### ES6 JOB QUEUE

ECMAScript 2015引入了Job Queue的概念，Promises使用了它。这是一种尽快执行异步函数结果的方法，而不是放在调用栈的末尾。

在当前函数结束之前resolve了的Promises将会在当前函数之后立即执行。

我觉得在游乐园里坐过山车是一个很好的类比：消息队列将你放在队列的最后面，在所有其他人的后面，那么你将不得不等待转弯，而job queue是一个快速通过的票，它可以让你在完成前一次之后马上再坐一次。

一个例子：
```js
const bar = () => console.log('bar')

const baz = () => console.log('baz')

const foo = () => {
  console.log('foo')
  setTimeout(bar, 0)
  new Promise((resolve, reject) =>
    resolve('should be right after baz, before bar')
  ).then(resolve => console.log(resolve))
  baz()
}

foo()
```
这段代码打印出：
```
foo
baz
should be right after baz, before bar
bar
```
这是Promises（以及建立在Promises上的Async/await）与通过setTimeout()或其它平台API建立的普通旧异步函数之间的一个巨大差异。

译自：[THE JAVASCRIPT EVENT LOOP](https://flaviocopes.com/javascript-event-loop/)（如有翻译得不准确之处，敬请指正，不胜感激）