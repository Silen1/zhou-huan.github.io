---
layout: post
title: "了解 Fetch API [译]"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - JavaScript
---

### 介绍

自从IE5于1998年发布以来，我们可以选择使用XMLHttpRequest（XHR）在浏览器中进行异步网络调用。

几年后，GMail和其它几款应用大量使用了它，使得该方法大受欢迎，以至于它必须拥有一个名字，这就是后来我们所知道的 XMLHttpRequest （XHR）。

直接使用 XMLHttpRequest 一直是很痛苦的一件事情，所以它总是被一些库抽象化之后再被使用，特别是 jQuery 有自己基于 XHR 的辅助函数：

* `jQuery.ajax()`
* `jQuery.get()`
* `jQuery.post()`
* ...

它们在易用性和兼容性上都给开发者提供了巨大的便利。

**Fetch API** 已被标准化为异步网络请求的现代方法，并使用 **Promises** 作为构建块。

fetch 在各大主流浏览器上都有着非常高的支持度，但除了 IE。

GitHub 发布的 polyfill [fetch](https://github.com/github/fetch) 让我们可以在任何浏览器上使用fetch。

### 使用

写个 GET 请求是非常简单的：

```js
fetch('/file.json')
```

fetch 将发出 HTTP 请求以获取同一域上的 file.json 资源。

如你所见，fetch 方法在全局作用域可用。

现在让我们来写点更有用的东西，我们来看一下请求到的文件内容是什么：

```js
fetch('./file.json')
  .then(response => response.json())
  .then(data => console.log(data))
```

调用 `fetch()` 会返回一个 promise，我们可以给 promise 的 then 方法传递一个处理函数，然后坐等 promise resolve。

该处理程序会接收到 promise 的返回值，即 Response 对象。

下一节会详细介绍这个对象。

### 捕获错误

由于fetch（）返回一个 promise，我们可以使用 promise 的 catch 方法来拦截执行请求期间发生的任何错误，以及在 then 回调中完成了的处理：

```js
fetch('./file.json')
.then(response => {
  //...
})
.catch(err => console.error(err))
```

### Response 对象

fetch() 调用返回的 Response 对象包含网络请求中请求与响应的所有信息。

#### HEADERS

访问响应对象上的 headers 属性使您能够查看请求返回的 HTTP 头：

```js
fetch('./file.json').then(response => {
  console.log(response.headers.get('Content-Type'))
  console.log(response.headers.get('Date'))
})
```

![request-headers](/img/article/request-headers.png)

#### STATUS

该属性是表示 HTTP 响应状态的一个整数。

* 101, 204, 205, or 304 是没有响应体的状态
* 200 到 299 是成功装填
* 301, 302, 303, 307, 308 是重定向

```js
fetch('./file.json').then(response => console.log(response.status))
```

#### STATUSTEXT

statusText 是表示响应状态消息的属性，如果请求成功，则状态为 OK。

```js
fetch('./file.json').then(response => console.log(response.statusText))
```

#### URL

就是我们请求的完整的 URL。

```js
fetch('./file.json').then(response => console.log(response.url))
```

#### BODY CONTENT

每一个响应都有一个 body，可以使用 text() 或 json() 方法来进行访问，这两个方法都返回一个 promise。

```js
fetch('./file.json')
  .then(response => response.text())
  .then(body => console.log(body))
```

```js
fetch('./file.json')
  .then(response => response.json())
  .then(body => console.log(body))
```

![text-json](/img/article/text-json.png)

也可以使用 ES2017 async 函数这么写：

```js
;(async () => {
  const response = await fetch('./file.json')
  const data = await response.json()
  console.log(data)
})()
```

### Request 对象

Request 对象表示资源请求，它通常使用 `new Request()` 来创建。

举个栗子：

```js
const req = new Request('/api/todos')
```

Request 对象提供了几个只读属性来检查资源请求的详细信息，包括：

* method
* url
* headers
* referrer
* cache 表示请求的缓存模式，例如：default, reload, no-cache

Request 对象还暴露出来了几个方法以供处理请求的：json(), text() 和 formData()。

完整 API 请移步 [MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/Request)。

#### REQUEST HEADERS

设置 HTTP 请求头是必不可少的，fetch 为我们提供了这样的能力，可以使用 Headers 对象来进行相关设置。

```js
const headers = new Headers()
headers.append('Content-Type', 'application/json')
```

或者更简单些这么写：

```js
const headers = new Headers({
  'Content-Type': 'application/json'
})
```

要将 headers 附加到请求上，我们要使用 Request 对象，并将它传递给 fetch() 方法，而不是像之前一样只传一个 URL了。

```js
const request = new Request('./file.json', {
  headers: new Headers({
    'Content-Type': 'application/json'
  })
})
fetch(request)
```

Headers 对象不只限于设置值，我们也可以进行查询：

```js
headers.has('Content-Type')
headers.get('Content-Type')
```

我们也可以删除之前设置的 header：

```js
headers.delete('X-My-Custom-Header')
```

### POST REQUESTS

fetch 当然也可以使用其它 HTTP 方法，POST, PUT, DELETE 或 OPTIONS。

在请求的 method 属性中指定方法，并在请求体（body）和 请求头（headers）中传递相关的附加参数。

举个栗子：

```js
const options = {
  method: 'post',
  headers: {
    'Content-type': 'application/x-www-form-urlencoded; charset=UTF-8'
  },
  body: 'foo=bar&test=1'
}

fetch(url, options).catch(err => {
  console.error('Request failed', err)
})
```

### Fetch 的缺点

虽然它和 XHR 相比有了很大的改进，但是 fetch 目前无法在请求完成后中止请求， 使用 Fetch，也很难测量上传进度。

如果在你的应用中又恰好需要这些东西，那 [Axios](https://github.com/axios/axios) 库可能非常适合你。

### 如何取消一个 Fetch 请求

在引入 fetch 后的几年，一旦打开了请求就无法中止。

但现在我们可以了，`AbortController` 和 `AbortSignal` 两个通用 API 可以帮到我们。

我们可以通过传递一个 `signal` 作为 fetch 的参数来集成此 API。

```js
const controller = new AbortController()
const signal = controller.signal

fetch('./file.json', { signal })
```

你可以设置一个 timeout 函数，在请求开始后 5 秒钟触发中止事件来取消该请求：

```js
setTimeout(() => controller.abort(), 5 * 1000)
```

同时你也不用担心，如果 fetch() 已经返回了结果，调用 abort() 不会导致任何错误。

当中止信号被触发时，fetch 将使用名为 AbortError 的 DOMException 来拒绝promise。

那我们的程序就可以这么写：

```js
fetch('./file.json', { signal })
  .then(response => response.text())
  .then(text => console.log(text))
  .catch(err => {
    if (err.name === 'AbortError') {
      console.error('我是中止信号触发时产生的错误^_^')
    } else {
      console.error('Another error', err)
    }
  })
```

—— 译自[UNDERSTANDING THE FETCH API](https://flaviocopes.com/fetch-api/)，不当之处，敬请指正！