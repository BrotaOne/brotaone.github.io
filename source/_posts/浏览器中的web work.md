---
title: 浏览器中的web work
date: 2023-11-23 15:02:17
tags:
    - 编程
    - 前端
---

参考：
-   [一文彻底学会使用web worker](https://juejin.cn/post/7139718200177983524)
-   [基于 Service Worker 进行多页面通信 | 项目复盘](https://juejin.cn/post/6941680632921587719)

# Web Worker
## Web Worker 是什么

Web Worker 是 HTML5 标准的一部分，这一规范定义了一套 API，允许我们在 js 主线程之外开辟新的 Worker 线程，并将一段 js 脚本运行其中，它赋予了开发者利用 js 操作多线程的能力。
因为是独立的线程，Worker 线程与 js 主线程能够同时运行，互不阻塞。所以，在我们有大量运算任务时，可以把运算任务交给 Worker 线程去处理，当 Worker 线程计算完成，再把结果返回给 js 主线程。这样，js 主线程只用专注处理业务逻辑，不用耗费过多时间去处理大量复杂计算，从而减少了阻塞时间，也提高了运行效率，页面流畅度和用户体验自然而然也提高了。

## Web Worker 能干些什么
虽然 Worker 线程是在浏览器环境中被唤起，但是它与当前页面窗口运行在不同的全局上下文中，我们常用的顶层对象 window，以及 parent 对象在 Worker 线程上下文中是不可用的。另外，在 Worker 线程上下文中，操作 DOM 的行为也是不可行的，document对象也不存在。但是，location和navigator对象可以以可读方式访问。除此之外，绝大多数 Window 对象上的方法和属性，都被共享到 Worker 上下文全局对象 WorkerGlobalScope 中。同样，Worker 线程上下文也存在一个顶级对象 self。

详细信息请参考：[Functions and classes available to Web Workers](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.mozilla.org%2Fen-US%2Fdocs%2FWeb%2FAPI%2FWeb_Workers_API%2FFunctions_and_classes_available_to_workers)

## Web Worker 分类
除了new Worker外，还有 SharedWorker 和 ServiceWorker


# 注意
* Web Worker 疑似不同页面都是共享的一个线程运行的，在一个页面断点暂停后，其他页面也会暂停
* Web Worker 的访问方式和其他文件不同，需要能在根目录访问，而且文件名需要固定，不能被混淆，通常在 webpack 或 vite 中直接放在 public 目录下即可
* 由于访问方式的限制，只能引用可以同样方式访问的文件，或者使用 url 路径（不能使用项目中的 npm 包）
* SharedWorker 需要在 edge://inspect/#workers 中才能看到 console.log 的输出，而 ServiceWorker 两个页面都能看到，Woker 只能在当前页面看到


# 使用
## Web Worker
Web Wokrer 相比另外两种比较简单，需要注意传递的是值而不是引用，而且只能传可以是由结构化克隆算法处理的任何值或 JavaScript 对象，包括循环引用
```js
// main.js
const worker = new Worker('./worker.js')
worker.postMessage('hello')
worker.onmessage = (e) => {
  console.log(e.data)
}
worker.addEventListener('message', e => { // 接收消息
    console.log(e.data); // Greeting from Worker.js，worker线程发送的消息
});
```

```js
// worker.js
self.postMessage('Greeting from Worker.js'); // 发送消息
self.addEventListener('message', e => { // 接收消息
    console.log(e.data); // hello，主线程发送的消息
});
```

## SharedWorker
如果采用 onmessage 方法，则默认开启端口，不需要再手动调用SharedWorker.port.start()方法

```js
// main.js
const worker = new SharedWorker('./sharedWorker.js')
worker.port.start(); // 开启端口
worker.port.postMessage('hello')
worker.port.addEventListener('message', e => { // 接收消息
    console.log(e.data); // Greeting from Worker.js，worker线程发送的消息
});

```

```js
// sharedWorker.js
let num = 0;
const workerList = [];

self.addEventListener('connect', e => {
    const port = e.ports[0];
    port.addEventListener('message', e => {
        num += e.data === 'add' ? 1 : -1;
        workerList.forEach(port => { // 遍历所有已连接的part，发送消息
            port.postMessage(num);
        })
    });
    port.start();
    workerList.push(port); // 存储已连接的part
    port.postMessage(num); // 初始化
});
```

但是如果要用这种方式来处理，多页面共享的话，似乎不是很方便，甚至不如用 ServiceWorker

## ServiceWorker
之前我以为 ServiceWorker 不能从 Worker postMessage给主线程，需要借助其他方式，比如在 connect 时用 MessageChannel 传递 port，但是实际上可以用 self。clients，已经维护好了所有的 页面

```js
// main.js
if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('./serviceWorker.js').then(registration => {
        console.log('service worker 注册成功')
    }).catch(err => {
        console.log('servcie worker 注册失败')
    })
}

const useNotification = () => {
  const [data, setData] = useState({});
  useEffect(() => {
    if ("serviceWorker" in navigator) {
      let listener = (event) => {
        const clientId = event.data.client;
        console.log(`receive message from ${clientId}`);
        setData(event.data);
      };
      navigator.serviceWorker.addEventListener("message", listener);
      return () => {
        navigator.serviceWorker.removeEventListener("message", listener);
      };
    }
  }, []);
  const postMessage = useCallback((message) => {
    if ("serviceWorker" in navigator) {
      navigator.serviceWorker.controller.postMessage(message);
    }
  }, []);
  return [data, postMessage];
};
```

```js
// serviceWorker.js
/* eslint-disable no-restricted-globals */
import { clientsClaim } from "workbox-core";
import { precacheAndRoute } from "workbox-precaching";

clientsClaim();
precacheAndRoute(self.__WB_MANIFEST);

// 监听来自网页客户端的消息
self.addEventListener("message", (event) => {
  if (event.data && event.data.type === "SKIP_WAITING") {
    self.skipWaiting();
  }
  var promise = self.clients.matchAll().then(function (clientList) {
    var senderID = event.source.id;
    // 消息不传递给发送者本身
    clientList.forEach(function (client) {
      if (client.id === senderID) {
        return;
      }
      client.postMessage({
        client: senderID,
        message: event.data,
      });
    });
  });
  if (event.waitUntil) {
    event.waitUntil(promise);
  }
});
```

ServiceWorker 可以拦截请求，可以做缓存和 PWA(Progressive Web App)
