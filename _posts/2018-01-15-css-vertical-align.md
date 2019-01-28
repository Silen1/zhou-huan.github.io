---
layout: post
title: "CSS垂直居中的12种方式"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - CSS
---

## 使用绝对定位和负外边距对块级元素进行垂直居中

_HTML_

    <div id="box">
        <div id="child"></div>
    </div>

_CSS_

    #box {
        width: 300px;
        height: 300px;
        background: #ddd;
        position: relative;
    }
    #child {
        width: 150px;
        height: 100px;
        background: orange;
        position: absolute;
        top: 50%;
        margin: -50px 0 0 0;
    }

![demo01](/img/article/demo01.png)

这个方法兼容性不错，但是有一个小缺点：必须提前知道被居中块级元素的尺寸，否则无法准确实现垂直居中。

## 使用绝对定位和transform

_HTML_

    <div id="box">
        <div id="child">test vertical align</div>
    </div>

_CSS_

    #box {
        width: 300px;
        height: 300px;
        background: #ddd;
        position: relative;
    }
    #child {
        background: orange;
        position: absolute;
        top: 50%;
        transform: translate(0, -50%);
    }

![demo01](/img/article/demo02.png)

这种方法有一个明显的好处就是不必提前知道被居中元素的尺寸了，因为 `transform` 中 `translate` 偏移的百分比就是相对于元素自身的尺寸而言的。

## 另外一种使用绝对定位和负外边距进行垂直居中的方式

_HTML_

    <div id="box">
        <div id="child">test vertical align</div>
    </div>

_CSS_

    #box {
        width: 300px;
        height: 300px;
        background: #ddd;
        position: relative;
    }
    #child {
    　　width: 50%;
        height: 30%;
        background: orange;
        position: absolute;
        top: 50%;
        margin: -15% 0 0 0;
    }

![demo01](/img/article/demo03.png)

这种方式的原理实质上和前两种相同。补充的一点是：`margin` 的取值也可以是百分比，这时这个值规定了该元素基于父元素尺寸的百分比，可以根据实际的使用场景来决定是用具体的数值还是用百分比。

## 绝对定位结合margin: auto

_HTML_

    <div id="box">
        <div id="child">test vertical align</div>
    </div>

_CSS_

    #box {
        width: 300px;
        height: 300px;
        background: #ddd;
        position: relative;
    }
    #child {
        width: 200px;
        height: 100px;
        background: orange;
        position: absolute;
        top: 0;
        bottom: 0;
        margin: auto;
        line-height: 100px;
    }

![demo01](/img/article/demo04.png)

这种实现方式的两个核心是：把要垂直居中的元素相对于父元素绝对定位，top和bottom设为相等的值，我这里设成了0，当然也可以设为 99999px 或者 -99999px 无论什么，只要两者相等就行，这一步做完之后再将要居中元素的 `margin` 属性值设为 `auto`，这样便可以实现垂直居中了。

被居中元素的宽高也可以不设置，但不设置的话就必须是图片这种自身就包含尺寸的元素，否则无法实现。

## 使用padding实现子元素的垂直居中

_HTML_

    <div id="box">
        <div id="child"></div>
    </div>

_CSS_

    #box {
        width: 300px;
        background: #ddd;
        padding: 100px 0;
    }
    #child {
        width: 200px;
        height: 100px;
        background: orange;
    }

![demo01](/img/article/demo05.png)

这种实现方式非常简单，给父元素设置相等的上下内边距，子元素自然是垂直居中的，当然这时候父元素是不能设置高度的，要让它自动被填充起来，除非设置了一个正好等于上内边距+子元素高度+下内边距的值，否则无法精确垂直居中。

这种方式看似没有什么技术含量，但其实在某些场景下也是非常好用的。

## 设置第三方基准

_HTML_

    <div id="box">
        <div id="base"></div>
        <div id="child"></div>
    </div>

_CSS_

    #box {
        width: 300px;
        height: 300px;
        background: #ddd;
    }
    #base {
        height: 50%;
        background: orange;
    }
    #child {
        height: 100px;
        background: rgba(131, 224, 245, 0.6);
        margin-top: -50px;
    }

![demo01](/img/article/demo06.png)

这种方式也非常简单，首先设置一个高度等于父元素高度一半的第三方基准元素，这时该基准元素的底边线就是父元素纵向上的中分线，做完这些之后再给要垂直居中的元素设置一个 `margin-top` 属性，值的大小是它自身高度的一半取负，则实现垂直居中。

## 使用flex布局

_HTML_

    <div id="box">test vertical align</div>

_CSS_

    #box {
        width: 300px;
        height: 300px;
        background: #ddd;
        display: flex;
        align-items: center;
    }

![demo01](/img/article/demo07.png)

这种方式同样适用于块级元素：

_HTML_

    <div id="box">
        <div id="child"></div>
    </div>

_CSS_

    #box {
        width: 300px;
        height: 300px;
        background: #ddd;
        display: flex;
        align-items: center;
    }
    #child {
        width: 300px;
        height: 100px;
        background: orange;
    }

![demo01](/img/article/demo08.png)

flex布局请参考阮一峰[《felx布局教程》](http://www.ruanyifeng.com/blog/2015/07/flex-grammar.html)

flex也就是flexible，意为灵活的、柔韧的、易弯曲的。

元素可以通过设置 `display:flex;` 将其指定为 flex 布局的容器，指定好了容器之后再为其添加 `align-items` 属性，该属性定义项目在交叉轴（这里是纵向轴）上的对齐方式，可能的取值有五个，分别如下：
　　
- flex-start:：交叉轴的起点对齐
- flex-end：交叉轴的终点对齐
- center：交叉轴的中点对齐
- baseline：项目第一行文字的基线对齐
- stretch（该值是默认值）：如果项目没有设置高度或者设为了auto，那么将占满整个容器的高度

## 第二种使用弹性布局的方式

_HTML_

    <div id="box">
        <div id="child"></div>
    </div>

_CSS_

    #box {
        width: 300px;
        height: 300px;
        background: #ddd;
        display: flex;
        flex-direction: column;
        justify-content: center;
    }
    #child {
        width: 300px;
        height: 100px;
        background: orange;
    }

![demo01](/img/article/demo08.png)

这种方式也是首先给父元素设置 `display:flex;` 设置好之后改变主轴的方向 `flex-direction: column;` 该属性可能的取值有四个，分别如下：
　　
- row（该值为默认值）：主轴为水平方向，起点在左端
- row-reverse：主轴为水平方向，起点在右端
- column：主轴为垂直方向，起点在上沿
- column-reverse：主轴为垂直方向，起点在下沿

`justify-content` 属性定义了项目在主轴上的对齐方式，可能的取值有五个，分别如下（具体的对齐方式与主轴的方向有关，以下假定主轴方向为默认的从左到右）：

- flex-start（该值是默认值）：左对齐
- flex-end：右对齐
- center：居中对齐
- space-between：两端对齐，各个项目之间的间隔均相等
- space-around：各个项目两侧的间隔相等

## 使用 `line-height` 对单行文本进行垂直居中

_HTML_

    <div id="box">test vertical align</div>

_CSS_

    #box{
        width: 300px;
        height: 300px;
        background: #ddd;
        line-height: 300px;
    }

![demo01](/img/article/demo07.png)

要注意的是，`line-height` (行高) 的值不能设为 `100%`，我们来看看官方文档中给出的关于 `line-height` 取值为百分比时候的描述：“基于当前字体尺寸的百分比行间距”。也就是说，这里的百分比并不是相对于容器元素尺寸而言的，而是相对于字体尺寸。

## 使用 `line-height` 和 `vertical-align` 对图片进行垂直居中

_HTML_

    <div id="box">
        <img src="smallpotato.jpg">
    </div>

_CSS_

    #box{
        width: 300px;
        height: 300px;
        background: #ddd;
        line-height: 300px;
    }
    #box img {
        width: 200px;
        height: 200px;
        vertical-align: middle;
    }

![demo01](/img/article/demo09.png)

`vertical-align` 并不像看起来那样天真无邪，深入研究请参考张鑫旭 [我对CSS vertical-align的一些理解与认识](http://www.zhangxinxu.com/wordpress/2010/05/%E6%88%91%E5%AF%B9css-vertical-align%E7%9A%84%E4%B8%80%E4%BA%9B%E7%90%86%E8%A7%A3%E4%B8%8E%E8%AE%A4%E8%AF%86%EF%BC%88%E4%B8%80%EF%BC%89/)

本例具体的实现原理请参考张鑫旭 [CSS深入理解vertical-align和line-height的基友关系](http://www.zhangxinxu.com/wordpress/2015/08/css-deep-understand-vertical-align-and-line-height/)

## 使用 `display: table;` 和 `vertical-align: middle;` 对容器里的文字进行垂直居中

_HTML_

    <div id="box">
        <div id="child">test vertical align</div>
    </div>

_CSS_

    #box {
        width: 300px;
        height: 300px;
        background: #ddd;
        display: table;
    }
    #child {
        display: table-cell;
        vertical-align: middle;
    }

![demo01](/img/article/demo07.png)

`vertical-align` 属性只对拥有 `valign` 特性的 html 元素起作用，例如表格元素中的 `<td> <th>` 等等，而像 `<div> <span>` 这样的元素是不行的。

`valign` 属性规定单元格中内容的垂直排列方式，语法：`<td valign="value">`，value的可能取值有以下四种：

- top：对内容进行上对齐
- middle：对内容进行居中对齐
- bottom：对内容进行下对齐
- baseline：基线对齐

关于 `baseline`：基线是一条虚构的线。在一行文本中，大多数字母以基线为基准。`baseline` 值设置行中的所有表格数据都分享相同的基线。该值的效果在文本的字号各不相同时效果会更好，请看下例：

_HTML_

    <div id="box">
        <div class="child">glory</div>
        <div class="child">glad</div>
        <div class="child">align</div>
    </div>

_CSS_

    #box {
        width: 300px;
        height: 300px;
        background: #ddd;
        display: table;
    }
    .child {
        display: table-cell;
        vertical-align: top;
        border-right: 1px solid orange;
    }
    .child:first-child {
        font-size: 30px;
    }
    .child:last-child {
        font-size: 50px;
    }

![demo01](/img/article/demo13.png)

如果将 `vertical-align` 属性的值修改为 `baseline`，效果更好：

![demo01](/img/article/demo14.png)

## 使用 CSS Grid

_HTML_

    <div id="box">
        <div class="one"></div>
        <div class="two">target item</div>
        <div class="three"></div>
    </div>

_CSS_

    #box {
        width: 300px;
        height: 300px;
        display: grid;
    }
    .two {
        background: orange;
    }
    .one, .three {
        background: skyblue;
    }

![demo01](/img/article/demo10.png)

这种场景下使用 `Grid Layout` 非常方便，只需要设置 `.one .three` 两个辅助元素即可，只是 Grid 布局现在[浏览器支持度](https://caniuse.com/#search=grid)还比较低。

_使用 CSS Grid 设置水平居中_

_HTML_

    <div id="box">
        <div></div>
        <div class="two">target item</div>
        <div></div>
    </div>

_CSS_

    #box {
        width: 300px;
        height: 300px;
        background: #ddd;
        display: grid;
        grid-template-columns: 1fr 1fr 1fr;
    }
    .two {
        background: orange;
    }

![demo01](/img/article/demo11.png)

同样的添加两个辅助元素，然后将 `grid-template-columns` 属性值设置为 `1fr 1fr 1fr`，意为三列子元素等分全部可用宽度。

也会有这样的场景，需要被居中的元素宽度已知，则：

_HTML_

    <div id="box">
        <div></div>
        <div class="two">target item</div>
        <div></div>
    </div>

_CSS_

    #box {
        width: 300px;
        height: 300px;
        background: #ddd;
        display: grid;
        grid-template-columns: 1fr 211px 1fr;
    }
    .two {
        background: orange;
    }

![demo01](/img/article/demo12.png)

#box 里的第一个 div 和最后一个 div 会平分全部剩余可用宽度，作为自身的宽度，即 (300px - 211px) / 2，各自 44.5px。