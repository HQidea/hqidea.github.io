---
title: 浅析postMessage
date: 2024-12-27 21:31:12
tags:
---

## 前言

在前端开发中，有时候我们需要在两个页面间进行数据交换。如果两个页面属于同一个源，我们有很多办法可以实现这一目标。但是，当两个页面属于不同的源时，办法就很有限了。本文将介绍其中的一种方法，即使用`postMessage`在不同源的两个页面间进行数据交换，当然该方法也适用于同源的页面间。

很多接口(interface)都提供了`postMessage`方法，比如`Window`，`MessagPort`，`Worker`，`Client`，`ServiceWorker`，`BroadcastChannel`等。<s>首先要做的事情是分清楚不同`postMessage`的作用和应用场景，</s>本文将结合在不同源的两个页面间进行数据交换的场景，介绍`Window: postMessage()`和`MessagePort: postMessage()`方法。

## `Window: postMessage()`

`window.postMessage()`方法可以安全地在不同源的`window`对象间进行通信。当然这有两个前提，

1. 消息发送方需要拿到接收方的`window`对象，
2. 消息接收方需要监听`message`事件

有两种方式可以满足前提1，分别是调用`window.open()`和使用`<iframe>`标签，因此也限制了`window.postMessage()`方法只能在这两种场景下使用。

> `window.open()`返回被打开页面的`window`对象，被打开的页面则可以通过`window.opener`访问调用了`window.open()`方法的页面`window`对象。
>
> 主frame可以通过`iframe.contentWindow`访问`iframe`内部的`window`对象，`iframe`内部则可以通过`window.parent`访问主frame的`window`对象。
>
> 消息接收方可以通过收到消息中的`source`属性获得发送方的`window`对象。

### API
#### 发送方API
```
postMessage(message)
postMessage(message, targetOrigin)
postMessage(message, targetOrigin, transfer)

postMessage(message, options)
```

+ `message`参数很好理解，就是发送的消息，可以传递能够被结构化克隆<sup>[3]</sup>的对象，比如js中的大多数类型。
+ `targetOrigin`参数指定接收方的`origin`，只有和实际的接收方`origin`（包括协议、域名和端口）相同时，接收方才能收到该消息。默认取当前页面的`origin`，可以设置为`*`，表示所有源都可以收到该消息。
+ `transfer`参数表示将一组可转移对象（Transferable objects）<sup>[4]</sup>的控制权交给接收方，用于共享内存或者交换`MessagePort`对象。
+ `options`参数是一个对象，包含`targetOrigin`和`transfer`属性。

#### 接收方收到的参数

接收方可以通过监听`message`事件来接收消息。

```
window.addEventListener(
  "message",
  (event) => {
    if (event.origin !== "http://example.org:8080") return;

    // …
  },
  false,
);
```

`event`包含下列属性：

+ `data`：接收到的消息，对应发送方的`message`参数。
+ `origin`：消息发送方的`origin`，这里的目的其实是和发送方做相互校验。
+ `source`：消息发送方的`window`对象，可以用该对象建立双向通信通道。

### 🌰

+ 发送消息
```
var o = document.getElementsByTagName('iframe')[0];
o.contentWindow.postMessage('Hello world', 'https://b.example.org/');
```

+ 接收消息
```
window.addEventListener('message', receiver, false);
function receiver(e) {
  if (e.origin == 'https://example.com') {
    if (e.data == 'Hello world') {
      e.source.postMessage('Hello', e.origin);
    } else {
      alert(e.data);
    }
  }
}

```

## `MessagePort：postMessage()`

Channel Messaging API目的是让两个不同上下文的脚本可以直接建立通道，进行双向通信。`MessagePort`则是使用该API建立通道后的两侧端点。

### API

#### 建立通道（MessageChannel）

```
const channel = new MessageChannel();
```

`channel`有两个属性，`port1`和`port2`，分别是通道的两侧端点即`MessagePort`对象。具体的收发消息需要使用这两个端点来完成。

#### 收发消息（MessagePort）

```
// 发送消息
postMessage(message)
postMessage(message, transfer)
postMessage(message, options)

// 接收消息
addEventListener("message", (event) => {});

onmessage = (event) => {};

addEventListener("messageerror", (event) => {});

onmessageerror = (event) => {};

// 开始
start()

// 关闭
close()
```

`MessagePort`是一个可转移对象。

+ `postMessage`的参数含义和`window.postMessage`一样，在此不再赘述，不一样的地方是不需要指定`targetOrigin`参数了。

+ 接收消息的方式也和接收`window.postMessage`消息的方式一样，只是调用方由`window`对象变成了`MessagePort`对象。有一点不一样的是，如果使用`addEventListener("message", (event) => {});`方式来监听消息的话，需要手动执行`start()`方法。

  + `event`包含下列属性：
    + `data`：接收到的消息，对应发送方的`message`参数。
    + `origin`：消息发送方的`origin`，实践中不同源场景下两侧端点收到的消息`origin`都为空。
    + `source`：消息发送方，可以是WindowProxy, MessagePort或者ServiceWorker，实践中不同源场景下两侧端点收到的消息`source`都为空。
    + `lastEventId`：消息的序号，实践中不同源场景下两侧端点收到的消息`lastEventId`都为空。
    + `ports`：与消息一起发送的所有`MessagePort`对象<sup>8</sup>，实践中不同源场景下两侧端点收到的消息`ports`都为空数组。

+ `start()`: 开始接收消息，在调用该方法之前收到的消息并不会丢弃，只是不执行`message`的回调函数。等`start()`执行后，之前收到的消息会依次调用已注册的回调函数。

+ `close()`: 关闭接收消息。

到目前为止，创建的`channel`和对应的`port1`和`port2`都还是在同一个上下文中。要实现在不同上下文的脚本中直接建立通道，需要借助可以转移对象控制权的方法，比如前文所述的`Window: postMessage()`。

### 🌰

+ 发送消息
```
var channel = new MessageChannel();
otherWindow.postMessage('hello', 'https://example.com', [channel.port2]);
channel.port1.postMessage('hello');
```

+ 接收消息
```
channel.port2.onmessage = handleMessage;
function handleMessage(event) {
  // message is in event.data
  // ...
}
```

## refences

1. https://www.google.com/search?q=postmessage+mdn&newwindow=1&sca_esv=d79442c60ff0c8fa&sxsrf=ADLYWIK6h__HzcQYgHwwAuYeqfWZyZftLw%3A1735269517122&source=hp&ei=jRxuZ8OrBajk2roP646MsQY&iflsig=AL9hbdgAAAAAZ24qnZBmMSYyd9SKfIRjjIjC4h2jXTxI&ved=0ahUKEwiDpp_Z_saKAxUoslYBHWsHI2YQ4dUDCBg&uact=5&oq=postmessage+mdn&gs_lp=Egdnd3Mtd2l6Ig9wb3N0bWVzc2FnZSBtZG4yChAjGIAEGCcYigUyBhAAGBYYHjIGEAAYFhgeMgsQABiABBiGAxiKBTILEAAYgAQYhgMYigUyCxAAGIAEGIYDGIoFSIAhUABY9R9wBXgAkAEAmAGCAaAB7AmqAQQxLjEwuAEDyAEA-AEBmAIQoAKNCsICBBAjGCfCAgsQABiABBiRAhiKBcICCxAuGIAEGLEDGIMBwgIIEAAYgAQYsQPCAgsQABiABBixAxiDAcICDhAAGIAEGLEDGIMBGIoFwgIKEAAYgAQYQxiKBcICDRAAGIAEGLEDGEMYigXCAgsQABiABBixAxjJA8ICDhAuGIAEGNEDGJIDGMcBwgIFEC4YgATCAgUQABiABMICCxAuGIAEGLEDGNQCwgIIEAAYgAQYywHCAgkQABgWGMcDGB6YAwCSBwQ1LjExoAelUw&sclient=gws-wiz
2. https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage
3. https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm
4. https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Transferable_objects
5. https://developer.mozilla.org/en-US/docs/Web/API/Channel_Messaging_API
6. https://developer.mozilla.org/en-US/docs/Web/API/Channel_Messaging_API/Using_channel_messaging
7. https://developer.mozilla.org/en-US/docs/Web/API/MessagePort
8. https://developer.mozilla.org/en-US/docs/Web/API/MessagePort/message_event
9. https://html.spec.whatwg.org/multipage/web-messaging.html
