---
layout: post
date: 2025-08-01
tags: Js
---

<!-- # Cookie，sessionStorage，localStorage，IndexedDB 介绍 -->

参考链接：

MDN：
- [Cookie](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Reference/Headers/Cookie)
- [HTTP Cookie](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Guides/Cookies)
- [Document.cookie](Document.cookie)
- [Window.sessionStorage](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/sessionStorage)
- [Window.localStorage](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/localStorage)

其他：
- [一文读懂浏览器本地存储：Web Storage](https://zhuanlan.zhihu.com/p/647359656)
- [Express example —— cookie](https://github.com/expressjs/express/blob/master/examples/cookies/index.js)

---

## Cookie

### 定义

HTTP 请求标头，可通过服务器设置 `Set-Cookie` 表头或 js 通过 `document.cookie` 进行设置。

### 大小

4KB

### 形式

```txt
Cookie: key1=value1; key2=value2 ...
```

键值对以“分号+空格”进行分隔。

### 分类

根据是否设置过期时间（`Expires`）和有效期（`Max-Age`）分为会话期 Cookie 和 持久性 Cookie。

### 存储位置

会话期 Cookie 存储在内存中，持久性 Cookie 存储在硬盘中。

### 特点

- 1.可设置过期时间。但无法直接删除 Cookie，需要将过期时间设置为当前时间之前。
- 2.Cookie 在跨站点请求中通常不会发送，但有例外情况——首先，`Secure` 未设置或值为 `Lax` ，然后对于“安全”的顶级导航（**`GET` 请求，且是顶级导航**，比如用户点击链接或重定向），Cookie 仍会被发送。

### 使用场景

1. 存储 Refresh Token（需要携带 `HttpOnly=true; Secure=true; SameSite=Strict`）
2. 会话状态管理（如用户登录状态、购物车、游戏分数或其他需要记录的信息）
3. 个性化设置（如用户自定义设置、主题等）
4. 浏览器行为跟踪（如跟踪分析用户行为）

### 如何使用

通常，服务器在收到首个 HTTP 请求后，在响应头例添加一个或多个 `Set-Cookie` 选项，浏览器收到响应后通常会保存 Cookie，并将其放在 HTTP Cookie 标头内，在向同一服务器发出请求时一起发送。

```js
response.setHeader('Set-Cookie', ['type=ninja', 'language=javascript']);
```

响应体：

```txt
HTTP/1.0 200 OK
Content-type: text/html
Set-Cookie: type=ninja
Set-Cookie: language=javascript
```

后续请求头：

```txt
Cookie: type=ninja; language=javascript
```

前端获取 Cookie 值的方式：

- document.cookie（标记了 `HttpOnly` 的 Cookie 除外）
- 第三方包（例如 js-cookie）

> 🧐通过 `document.cookie` 获取到 cookie 字符串后如何处理，方便后续使用？

## sessionStorage

### 特点

- 1.会话结束（关闭 Tab/浏览器）时数据被清除。
- 2.重新加载页面数据并不会被清除（刷新）。
- 3.“新标签或窗口打开一个页面时会复制定级浏览会话上下文作为新会话的上下文，这点和 session cookie 的运行方式不同”。
- 4.打开多个相同 URL 的 Tab 页面，会创建各自的 sessionStorage，不会共享。
- 5.存储的数据是特定于页面的协议的，也就是**受同源策略限制**。

### 形式

存储数据为字符串，因此对象格式的数据存储需要转换成 JSON 格式。

### 大小

5MB-10MB 不等，一般现代浏览器至少会分配 5MB 空间，实际大小受设备实际存储空间用量动态分配。

### 存储位置

通常是内存中。

### sessionStorage 会共享的情况（受同源影响）

- `<a target="_blank" href="url_or_path" rel="opener"></a>`
- `window.location.href = url`
- `window.open(url, ...)`
- 在当前地址栏输入 url
- 服务端重定向

> 如果 a 标签没有添加 `rel="opener"` 属性，跳转时是不会复制顶级页面的 sessionStorage 的。这是因为 Chrome 89 版本更新后，通过 a 标签 `target="_blank"` 跳转会导致 sessionStorage 丢失，原因是默认添加了 `noopener` 属性。

##  localStorage

### 特点

- 1.相对于 sessionStorage，localStorage 的数据可以**长期保留**，即使浏览器关闭（**建议程序中手动封装设置过期时间，否则数据会一直存在**）。
- 2.和 sessionStorage 一样，存储的数据是特定于页面的协议的，也就是**受同源策略限制**，不同源的页面间无法共享数据。
- 3.**不能频繁存储大量数据，因为写入数据时会阻塞页面**。

### 使用场景

- 页面中基本不会发生变化的数据，存储下来减少请求次数。
- Tab 间通信、共享数据（同源）
- 用户导引界面的数据
- 部分表单数据

### 封装 localStorage

```js
const myStorage = {
  set(key, value) {
    const time = Date.now();
    localStorage.setItem(key, JSON.stringify({ time, value }));
  },
  get(key) {
    const valueJson = localStorage.getItem(key);
    if (!valueJson) return null;
    const { time, value } = JSON.parse(valueJson);
    const expiresTime = 60;
    if (Date.now() - time > expiresTime) {
      localStorage.removeItem(key);
      return null;
    } else {
      return value;
    }
  }
}
```

> 🧐如何用代码控制 localStorage 用量？

❓Safari 的无痕浏览模式

## IndexedDB

### 定义

一个事务型数据库系统，是一个基于 JavaScript 的面向对象数据库。

### 特点

- 1.异步执行操作
- 2.用于大量数据的本地存储
- 3.和 worker 结合，存储数据时不会对页面造成阻塞
- 4.使用索引实现对数据的高性能搜索

### 大小

多达几个GB，受设备实际存储空间用量动态分配。

### 存储位置

本地磁盘

---

待探索：
- [cookie-based sessions](https://github.com/expressjs/express/blob/master/examples/cookie-sessions/index.js)