---
title: IFrame跨域通信方法
date: 2019-06-07 23:30:09
categories:
- 前端
tags:
- 跨域
- 同源策略
- CRO IFrame
---

最近遇到业务方希望能iframe嵌入我们的页面，直接复用我们的产品功能，我们欣然应允。但是发现不是简单iframe就能完成，嵌入我们页面的页面（暂且称之为父页面）需要与我们的页面有一些交互，比如页面在保存的时候，需要告诉父页面返回的数据id。父页面与我们的页面域名不同，由于浏览器同源策略的限制，导致普通的页面通信方法被浏览器阻断，但是最终我们基于`Window​.post​Message()`实现了跨域的iframe通信，成功服务了客户。本文分享一下解决iframe跨域问题的过程和方法。

# 同源策略
同源策略是web安全的基石，同源策略通过限制不同源之间的交互，避免恶意代码访问未授权的信息。例如在服务端未授权跨域访问的情况下，ajax请求无法获得非同源服务端的返回数据。在iframe的场景，父页面无法直接访问嵌入的iframe页面的数据（非同源情况下），反过来被嵌入的iframe页面也无法访问父页面的数据。

那么同**源**策略的**源**是怎么定义的，什么样的源是同源呢？根据MDN的定义，**源**其实是一个三元组：
```text
(协议, 主机（http[s]协议场景就是域名）, 端口)
```
例如：https://jerryxgh.xyz/about/ 对应的源三元组就是
```text
(https, jerryxgh.xyz, 80)
```
只有在两个源的三元组全部相同的时候，才是同源的。那么在不同源的场景，iframe嵌入页面与外层页面怎么通信呢，使用`window.postMessage`。
> MDN: To communicate between documents from different origins, use `window.postMessage`.

# iframe跨域通信-发消息
iframe能够帮助整体复用产品，在实际业务中经常被使用，而且非同源（大部分是跨域）的场景占大多数，这是因为同源的场景往往不需要iframe，直接复用组件效果更好，因此解决iframe跨域场景下的通信问题尤为重要。根据MDN文档，跨域iframe最好使用`window.postMessage`。

先来看看`window.postMessage`的函数定义：
> targetWindow.postMessage(message, targetOrigin, [transfer]);

我们逐个来看函数各部分的含义。
## `targetWindow`
`targetWindow`是消息发送的目标窗口，那么如何获得目标窗口呢，有以下方法
* `window.open` 打开一个新窗口并引用它
* `window.opener` 如果当前窗口是另一个窗口（父窗口）用`window.open`函数打开的，那么当前窗口的`window.opener`指向父窗口，否则为`null`
* `HTMLIFrameElement.contentWindow` 父窗口引用被嵌入的iframe窗口，例如我们为iframe元素设置id，之后用`document.getElementById('iframeElementId').contentWindow`就可以获得iframe页面的窗口对象
* `window.parent` 被嵌入的iframe子页面，引用嵌入自己的父页面窗口对象
* `window.frames` 加上索引值（可以是从零开始的数字索引或者iframe的名称）引入当前页面嵌入的iframe页面。但其实`window.frames === window`，因此这其实是一个语法糖，可以直接使用`window[0]`代替`window.frames[0]`

## `message`参数
`message`参数是发送到目标窗口的数据，为了保证数据安全性，发送的数据是结构化算法(The structured clone algorithm**克隆之后的结果，由于克隆算法不支持`Error**、`Function`和DOM节点，因此不能直接发送此类对象。

## `targetOrigin`参数
`targetOrigin`指定允许接收参数的源，之前讲过源是一个三元组，因此`targetOrigin`只需要指定**协议**+**域名**+**端口号**，如果希望所有源都能接收到消息，`targetOrigin`可以写成"*"。一般不建议用"*"，尤其是传输的是类似密码等敏感数据的时候，尽量明确指定可以接收消息的源，避免敏感数据被恶意代码劫持。此外`targetOrigin`只能指定一个明确的源或者所有源，无法指定多个源。

## `transfer`参数
`transfer`参数是可选的，是`Transferable`类型对象数组，与消息一起传递到目标窗口，不同的是这些对象的所有权也将转移到接收方。`transfer`参数使用场景比较少，而且兼容性不太好。

## 兼容性
`window.postMessage`的基本功能在现代浏览器上已经全部支持，对于`transfer`参数的支持度一般，尤其是移动端浏览器。因此可以放心使用`window.postMessage`的基本功能。

# iframe跨域通信-收消息
`window.postMessage`用于发送消息，接收方使用`window.addEventListener`接收消息：
```javascript
window.addEventListener("message", receiveMessage, false);

function receiveMessage(event) {
    console.log(event);
    // ...
}
```
其中回调函数收到的event对象，包含的主要属性如下：
* `data`
发送方发出的数据
* `origin`
发送方窗口的源
* `source`
发送方的窗口引用，接收方可以通过该窗口引用向发送发发送数据，行程通信通道。

# 实战样例
首先是父页面，也就是嵌入iframe的页面，域名是iframe.com，域名解析可以使用本地host绑定的方式实现，父页面代码如下：
```html
<!doctype html>

<html lang="zh-CN">
    <head>
        <title>Test CRO IFrame Inner Page</title>

        <meta charset="utf-8">
        <meta name="description" content="Test CRO IFrame">
        <meta name="author" content="jerryxgh">
        <script type="text/javascript">
         window.addEventListener("message", receiveMessage, false);

         function receiveMessage(event) {
             console.log(event);
         }
        </script>
    </head>

    <body>
        你好，我是嵌入的iframe页面
        <br/>
        <button onclick="window.parent.postMessage('message from iframe inner', 'http://iframe.com:8080')">按我给父页面发消息</button>
    </body>
</html>
```

被iframe嵌入的页面，域名是iframeinner.com，页面代码如下：
```html
<!doctype html>

<html lang="zh-CN">
  <head>
    <title>Test CRO IFrame</title>

    <meta charset="utf-8">
    <meta name="description" content="Test CRO IFrame">
    <meta name="author" content="jerryxgh">
    <script type="text/javascript">
     window.addEventListener("message", receiveMessage, false);

     function receiveMessage(event) {
         console.log(event);
         event.source.postMessage("got your message", 'http://iframeinner.com:8080')
     }
    </script>
  </head>

  <body>
    <iframe id="inline-frame-id"
            title="Inline Frame Example"
            width="300"
            height="200"
            name="inner-frame"
            src="http://iframeinner.com:8080/_iframe_inner.html">
    </iframe>
  </body>
</html>
```
点击iframe嵌入页面的按钮，效果如下图：
![IFrame实验截图](iframe_test_snapshot.jpg "IFrame实验截图")
子页面的按钮点击之后，向父页面发送了一条消息，内容是"message from iframe inner"，父页面收到消息之后，向子页面回复了一条消息"got your message"，从图中看出消息的确发送和接收成功了。

# 参考资料
1. [同源策略](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy)
2. [iframe](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/iframe)
3. [Window​.post​Message()](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage)
4. [Event​Target​.add​Event​Listener()](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener)
5. [结构化克隆算法](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm)
