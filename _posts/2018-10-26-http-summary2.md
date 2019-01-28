---
layout: post
title: "关于HTTP的一点总结（二）—— URI、URL和URN"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - 计算机网络
---

* URI（Universal Resource Identifier）统一资源标识符
* URL（Universal Resource Locator）统一资源定位符
* URN（Universal Resource Name）统一资源名称

URI在世界范围内唯一标识并定位信息资源。URI有两种形式，分别称为URL和URN。

URL我们都很熟悉。它描述了一台特定服务器上某资源的特定位置。它们可以明确地说明如何从一个精确，固定的位置获取资源。

URN作为特定内容的唯一名称使用，与目前的资源所在地无关。通过URN，还可以用同一个名字通过多种网络协议来访问资源。

URN现在仍处于试验阶段，还未大范围使用。为了更有效的工作，URN需要一个支撑架构来解析资源的位置，此类架构的缺乏也延缓了它被采用的进度。

**举例说明**

一个URL的例子：https://www.jianshu.com/u/99dadf57ad5e

URL包含三个部分：

* scheme（协议）对应到上例为 https://
* host（服务器的因特网地址）对应上例 www.jianshu.com
* path（Web服务器上某个资源的路径）对应上例 /u/99dadf57ad5e

URN的两个例子：

| URN | 相当于 |
| ------ | ------ |
| urn:isbn:0451450523 | 1968年出版的 《The Last Unicorn》 一书，由其书号确定 |
| urn:ietf:rfc:2141 | 因特网标准文档 RFC 2141 |

第一个例子中的书号是 International Standard Book Number，国际标准书号。同一个版本的《The Last Unicorn》都有着相同的ISBN（平装本、精装本、电子书各自具有不同的ISBN）。那么就可以这么说，同一个版本的《The Last Unicorn》不管身在何处，都可以用相同的URN来表示。

同样的，第二个例子中，不管因特网标准文档 RFC 2141的具体地址是什么，都可以通过urn:ietf:rfc:2141来命名它。

**从这些例子和概念能够看出来，这三者的区别是这样子的：**

* URI标识并定位了世界范围内的唯一信息资源。具体来说，是通过URL和URN这两种形式来标识和定位的。

* URL是通过一个地址来标识定位信息资源的，只要能够定位到一个资源，那它就叫URL。

* URN是通过一个特定格式的名称来标识定位信息资源的，但它不指定该资源的具体位置。

关于这三者的关系，看到网上很多示意图是这样的：

![relationship_from_other](/img/article/relationship_from_other.jpg)

但我觉得这个不太合理。根据我们上面的推断，URL/URN都和URI没有父子集的关系。看到这张图时我会以为一个URI可以对应多个URL，一个URI也可以对应多个URN。但事实又不是这样，URL/URN分别是URI的两种表现而已。

所以我重新画了一张：

![relationship](/img/article/relationship.png)

体现了URL和URN分别是URI的两种形式。

关于这三者的关系，又可以这么说。

假如有个坏人警察要抓他，那URI就相当于警察要找的这个人。警察想抓到他，要么得知道这个人的地址，相当于URL。要么得知道这个人的姓名和身份证号，相当于URN。通过URL和URN都可以定位到想找的这个URI。

关于这个问题，我还有一点补充。

假设已经有了一个URL1，定位到了Internet上的一个资源。现在，如果我又将另外一个域名和该资源所在服务器的外网IP绑定，并且和原来设置相同的端口，那又会产生另外一个URL2，URL2也能够定位到这个资源。也就是说该资源可以通过两个不同的URL来定位。URL只是URI的一种表现形式，这时候，该资源也就有着两个URI。

URL和URN也会存在一些交集，比如[SMTP](https://en.wikipedia.org/wiki/Simple_Mail_Transfer_Protocol)，简单邮箱传输协议。
