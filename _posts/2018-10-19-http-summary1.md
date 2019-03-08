---
layout: post
title: "关于HTTP的一点总结（一）—— GET和POST"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - 计算机网络
---

以前面试时多次被问到过GET和POST的区别，看到HTTP权威指南里这两个方法的内容时，我觉得自己的想法可能有点问题，所以做点总结如下。

关于这两个方法，[w3schools](https://www.w3schools.com/tags/ref_httpmethods.asp) 里是这么说的，这和我以前的想法非常吻合：

> ### The GET Method
> **GET is used to request data from a specified resource.**
>
> Note that the query string (name/value pairs) is sent in the URL of a GET request:
> ```
> /test/demo_form.php?name1=value1&name2=value2
> ```
> **Some other notes on GET requests:**
> * GET requests can be cached
> * GET requests remain in the browser history
> * GET requests can be bookmarked
> * GET requests should never be used when dealing with sensitive data
> * GET requests have length restrictions
> * GET requests is only used to request data (not modify)
> ### The POST Method
> **POST is used to send data to a server to create/update a resource.**
>
> The data sent to the server with POST is stored in the request body of the HTTP request:
> ```
> POST /test/demo_form.php HTTP/1.1
> Host: w3schools.com
> name1=value1&name2=value2
> ```
> Some other notes on POST requests:
> * POST requests are never cached
> * POST requests do not remain in the browser history
> * POST requests cannot be bookmarked
> * POST requests have no restrictions on data length
>

HTTP权威指南里是这么说的：

> GET是最常用的方法，通常用于请求服务器发送某个资源。HTTP/1.1要求服务器实现此方法。图3-7显示了一个例子，在这个例子中，客户端用GET方法发起了一次HTTP请求。
>
> ![get-example](/img/article/get-example.png)
>
> POST方法起初是用来向服务器输入数据的。实际上，通常会用它来支持HTML的表单。表单中填好的数据通常会发送给服务器，然后由服务将它发送到它要去的地方（比如，送到一个服务器网关程序，然后由这个程序对其进行处理）。图3-10显示了一个用POST方法发起HTTP请求 —— 向服务器发送表单数据 —— 的客户端。
>
> ![post-example](/img/article/post-example.png)
>

让我产生疑惑的是权威指南里并没有说w3schools里的那些区别，这会让我觉得它们只有语义上的区别 —— GET方法通常用于请求服务器发送某个资源而POST方法用来向服务器输入数据。除此之外别无二致。

那 [w3schools](https://www.w3schools.com/tags/ref_httpmethods.asp) 里说的几点区别又是怎么回事呢？

**GET requests have length restrictions / POST requests have no restrictions on data length**

关于这一点我查了之后发现，长度限制并不是HTTP协议规定的，而是浏览器和web服务器共同作用的结果。

每种浏览器都有自己的URL长度限制，这一点应该众所周知。但有一点我以前不知道的是，不同的Web服务器（Apache/ngnix/IIS）也会有不同的长度限制，并且根据服务器的处理能力以及设置的不同，也会有不同的作用结果。

**GET requests should never be used when dealing with sensitive data**

关于这一点，我有一些疑问，如果使用抓包工具或者打开浏览器控制台，POST请求中传递的数据同样可以一览无余。所以我觉得w3schools的这个说法不太严谨。硬要讨论安全性的话，GET请求应该只是看起来比POST请求安全一些。

**GET requests can be cached / POST requests are never cached**
**GET requests remain in the browser history / POST requests do not remain in the browser history**
**GET requests can be bookmarked / POST requests cannot be bookmarked**

关于这几点，应该说是不算区别的区别。因为GET请求的参数作为URL的一部分，在浏览器的作用下， 才有了这几点区别，也不关HTTP什么事。

**GET requests is only used to request data (not modify)**

我认为这一点也不是绝对的。如果使用GET方法来请求一个服务端接口，该接口也可以在接收到请求时对数据库进行相应修改，这取决于部署在应用服务器上的服务端代码功能。所以我觉得w3schools这么说，应该只是从语义上规范我们最好这么来做。
