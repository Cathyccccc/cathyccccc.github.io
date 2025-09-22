---
layout: default
tags: "Js"
---

# Worker 之 SharedWorker 初探 —— 多个页面间共享资源

**实现多个 tab 页面间的通信方式**有很多种，比如：

- websocket
- localStorage
- BroadcastChannel
- SharedWorker
等等

-----

## Web Worker

特点：
（1）worker 会在一个单独的线程中执行，不会阻塞主线程，因此可以将包含大量计算的代码交给 worker 执行。
（2）worker 有自己的上下文，因此无法访问 Window、DOM 等。
（3）消息机制：通过自身的 `postMessage()` 方法和 `onmessage` 事件处理程序，来处理和主线程之间的数据传递。也可以通过 MessageChannel 与主线程进行通信。（注：**传递的数据包含在 message 事件的 data 属性中，这个过程数据是被复制而不是被共享的。**）

通常所说的 Web Worker 指的是 Dedicated Worker，其他还有 Shared Worker（共享），Service Worker（缓存）、Audio Worker（处理音频）。

------------------------------------

这里主要对 **Shared Worker** 进行一个简单的探索，示例一个小 Demo，因为官方的 Demo（运行共享型 worker）没什么卵用。

相关链接：

- [MDN SharedWorker](https://developer.mozilla.org/zh-CN/docs/Web/API/SharedWorker)

**SharedWorker 是一种在多个浏览器上下文中共享脚本的机制。**

### 示例

这里我们有两个页面 `index1.html` 和 `index2.html`，它们都**使用**了同一个 worker.js 文件。
在 index1 页面中输入两个数字，然后进行乘法运算，之后将运算结果发送给 index2 页面，将计算的结果呈现在页面上。（计算过程可以移到 worker.js 中）

```html
<-- index1.html -->
<label for="number1">number1</label>
<input type="number" id="number1" />
<label for="number2">number2</label>
<input type="number" id="number2" />
<button>Send</button>
```

```js
let number1 = 0, number2 = 0;
const input1 = document.getElementById("number1");
const input2 = document.getElementById("number2");
const button = document.querySelector("button");
input1.addEventListener("change", (e) => {
  number1 = e.target.value;
});
input2.addEventListener("change", (e) => {
  number2 = e.target.value;
});

var myWorker = new SharedWorker("worker.js");
myWorker.port.start();
button.addEventListener("click", () => {
  myWorker.port.postMessage([number1, number2]);
});
```

![Description](/images//090237-74282542.png)

```html
<-- index2.html -->
<span id="shared"></span>
```

```js
const oSpan = document.querySelector("#shared");
var myWorker = new SharedWorker("worker.js");
myWorker.port.start();
myWorker.port.onmessage = (e) => {
  oSpan.textContent = e.data
}
```
这时，我们可以看到 index2 页面上显示计算结果为 12 。