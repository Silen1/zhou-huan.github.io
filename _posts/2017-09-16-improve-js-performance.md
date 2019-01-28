---
layout: post
title: "改进JavaScript代码性能的几种方式"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - JavaScript
---

## 1. 减少作用域链查找时间

随着作用域链中作用域数量的增加，访问当前作用域以外的变量的时间也在增加。只要能减少花费在作用域链上的时间，就能增加脚本的整体性能。

使用全局变量和函数的开销肯定要比局部的大，因为涉及到作用域链上的查找。请看下例：

```js
function updatePage () {
    var images = document.querySelectorAll('img')
    for (var i=0, len=images.length; i < len; i++) {
        images[i].title = 'image' + i + 'of' + document.title
    }
    var msg = document.querySelector('#msg')
    msg.innerHTML = 'The page has been updated...'
}
```

该函数里包含了三个对全局document对象的引用，如果页面上的图片非常多的话，那么循环中的document引用就会被执行很多很多次，每次的引用都要进行一次作用域链查找，性能开销非常大。

解决方法：将对document对象的引用保存到一个局部变量，就可以通过限制一次全局查找来改进该函数的性能。如下：

```js
function updatePage () {
    var doc = document
    var images = doc.querySelectorAll('img')
    for (var i=0, len=images.length; i < len; i++) {
        images[i].title = 'image' + i + 'of' + doc.title
    }
    var msg = doc.querySelector('#msg')
    msg.innerHTML = 'The page has been updated...'
}
```

将在一个函数中会多次用到的上层作用域的对象存储为局部变量总是没错的。

## 2. 避免不必要的属性查找

常量值查找以及访问数组元素的算法复杂度都是O(1)，而访问对象上属性的复杂度为O(n)。

对象上的任何属性查找都要比访问变量或者数组花费的时间更长，因为必须在原型链中对拥有该名称的属性进行一次搜索，属性查找越多，执行的时间就越长。请看下例：

```js
var query = window.location.href.substring(window.location.href.indexOf('?'))
```

改进如下：

```js
var link = window.location.href
var query = link.substring(link.indexOf('?'))
```

改进之后，6次属性查找变为了4次。基数小的话并不会导致显著的性能问题，但在更大的脚本中进行这种优化的话，将会获得非常明显的性能收益。

一旦多次用到对象属性，最好就将其存储在局部变量中，第一次访问该值是会是一次属性查找（O(n)操作），但后续的访问都会是值查找（O(1)操作）。

## 3. 优化循环

### (1) 将加值迭代更换为减值迭代

请看下例：

```js
for (var i=0; i < box.length; i++) {
    // do something with box[i]
}
```

改进如下：

```js
for (var i=box.length-1; i >= 0; i--) {
    // do something with box[i]
}
```

改进之后，将终止条件从box.length的O(n)调用简化成了0的O(1)调用。由于循环每进行一次都要去判定终止条件成立与否，故循环进行的次数越多性能改进就越明显。

### (2) 简化终止条件

由于每次的循环过程都会计算终止条件，所以有必要保证计算尽可能快，也就是要尽量避免属性查找或其它的O(n)操作。

### (3) 简化循环体

循环体是执行最多的，所以要确保其最大限度的被优化，确保没有某些可以被很容易移出循环的密集计算。

### (4) 使用后测试循环

最常用的for循环和while循环都是前测试循环，而像do-while这种后测试循环，可以避免最初终止条件的计算，因而会运行的更快。

## 4. 避免双重解释

当JavaScript代码想解析JavaScript的时候就会存在双重解释惩罚。比如以下几个例子：

```js
eval('alert("Hi summer!")')

var sayHi = new Function('alert("Hi summer!")')

setTimeout('alert("Hi summer!")', 500)
```

这几个例子中的代码都要解析包含了JavaScript代码的字符串，可是这个操作是不能在初始的解析过程中完成的，因为代码是包含在字符串中的，这也就是说，JavaScript代码在运行的同时必须新启动一个解析器来解释包含在字符串中的代码，实例化一个新的解析器有着不容忽视的开销，所以这种代码要比直接解析慢的多。

分别修改如下：

```js
alert('Hi summer!')

var sayHi = function () {
    alert('Hi summer!')
}

setTimeout(function() {
    alert('Hi summer!')
}, 500);
```

只有极少的情况下eval()是绝对必须的，所以要尽量避免使用。

## 5. 最小化语句数

JavaScript代码中，完成多个操作的单个语句要比完成单个操作的多个语句快，所以要找出可以组合在一起的多个语句，以减少脚本执行的整体时间。以下有几个可以参考的模式：

### (1) 多个变量声明使用单个语句

```js
var firstName = 'kobe'
var lastName = 'bryant'
var team = 'Lakers'
var number = 24
var friends = ['james', 'wade', 'paul']
```

改写如下：

```js
var firstName = 'kobe',
    lastName = 'bryant',
    team = 'Lakers',
    number = 24,
    friends = ['james', 'wade', 'paul']
```

### (2) 插入迭代值

在不同的位置进行增加或减少值的时候，尽可能地合并语句。

```js
var name = players[i]
i++
```

改写如下：

```js
var name = players[i++]
```

出现类似情况的话，就可以考虑将迭代值插入到最后使用它的语句中去。

### (3) 使用数组和对象字面量

```js
var players = new Array()
players[0] = 'kobe'
players[1] = 'jordan'
players[2] = 'wade'
```

改写如下：

```js
var players = ['kobe', 'jordan', 'wade']
```

改写之前使用了四条语句，第一句调用构造函数，其它三句用于分配数据，改写之后只使用一条语句来创建并初始化该数组。虽然这里只减少了三条语句数，可是在包含成千上万行JavaScript代码的页面中，这些优化的价值会非常大。

下例同理：

```js
var player = new Object()
player.name = 'jordan'
player.number = 23
player.height = 198
```

改写如下：

```js
var player = {
    name = 'jordan',
    number = 23,
    height = 198
}
```

## 6. 优化DOM操作

在JavaScript中，DOM操作无疑是最慢的一部分，因为它们往往需要重新渲染整个页面或者其中某一部分。看似细微的操作可能也要花很长时间才能执行的完，因为DOM要处理非常多的信息。

### (1) 使用事件代理

页面中事件处理程序的数量和页面响应用户交互的速度有一个负相关，为了减轻这种惩罚，最好使用事件代理。

### (2) 最小化现场更新

一旦你需要访问的DOM部分是已经显示的页面的一部分，那么你就是在进行一次现场更新。每一个更改，不管是小到插入单个字符，还是大到移除整个片段，都有一个性能惩罚，因为浏览器需要重新计算无数尺寸以进行更新。请看下例：

```html
<div id="list"></div>
```
```js
<script>
    var list = document.querySelector('#list'),
        item,
        i
    for (i = 0; i < 10; i++) {
        item = document.createElement('li')
        list.appendChild(item)
        item.appendChild(document.createTextNode('item-' + i))
    }
</script>
```

这段代码里每个循环中都有两个现场更新：一个是添加\<li\>元素，另外一个是给它添加文本节点，这样下来，一共进行了20个现场更新。要修正这个性能瓶颈，一般有两种方法：

* 先将列表从页面上移除，然后更新列表，最后再将列表插回到原来的位置。这个方法不是非常理想，因为在每次页面更新的时候会出现不必要的闪烁。

* 使用文档片段来构建DOM结构，接着再将其添加到list元素中。这个方式是最理想的，因为它同时避免了频繁的现场更新和页面的闪烁问题。

修改如下：

```html
<div id="list"></div>
```
```js
<script>
    var list = document.querySelector('#list'),
        box = document.createDocumentFragment(),
        item,
        i
    for (i = 0; i < 10; i++) {
        item = document.createElement('li')
        box.appendChild(item)
        item.appendChild(document.createTextNode('item-' + i))
    }
    list.appendChild(box)
</script>
```

修改之后只有一次现场更新，发生在所有项目都创建好之后，将所有项目添加到列表中的时候。

文档片段用作一个临时的占位符，放置新创建的项目，当给appendChild()方法传入文档片段时，只有片段中的子节点会被添加到目标元素中，片段本身不会被添加。一旦需要更新DOM，就可以考虑使用文档片段来构建DOM结构，然后再将其添加到现存文档中。

### (3) 使用innerHTML

一般情况下，有两种在页面上创建DOM节点的方法：一种是使用createElement()和appendChild()这种DOM方法；另外一种是使用innerHTML。

对于小的DOM更改来说，两种方法的效率都差不多。然而对于大的DOM更改，使用innerHTML要比使用标准的DOM方法快得多。

当把innerHTML设置为某个值的时候，后台会创建一个HTML解析器，然后使用内部的DOM调用来创建DOM结构，而非基于JavaScript的DOM调用。由于内部方法是编译好的而非解释执行的，所以执行起来要快得多。
