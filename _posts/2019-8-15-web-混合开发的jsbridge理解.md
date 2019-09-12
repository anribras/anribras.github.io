---
layout: post
title:
modified:
categories: Tech
tags: [javascript, web]
comments: true
---

<!-- TOC -->

- [总览](#总览)
- [交互 jsbrige](#交互-jsbrige)

<!-- /TOC -->

## 总览

自己是从这个角度开始切入的前端开发.

目前阶段，web 页面是辅助 native.如一些 2 级页面，产品页等等.

技术栈

```sh
vue.js
webview
单页面混合hybrid app;(页面要么是h5,要么是native)
```

## 交互 jsbrige

看了这个<https://zhuanlan.zhihu.com/p/32899522>，封装的稍微好点.

我们这个很简单,不过大体也就这么用的了.

他是用一个 eventMap 保存在 jsBridge 对象里,最终也是把这个对象抛给了 native.

执行时，需要在 jsBrige.eventMap 里找对应的闭包并执行:

```js
eventMap: {
  'someMethod': function(data) {  callback && callback(data);}
}
```

我自己则是直接把 callback 方法挂在了 window 上，想回调,调你就调用.

```js
window.someMethod(...) {

}
```

web js 通过不同的函数名，传不同的 args 到 native.

```js
function inner(functionName, args) {
  console.log("jsb args:");
  console.log(args);
  args = typeof args === "object" && args ? JSON.stringify(args) : args;
  console.log("jsb: " + functionName);
  try {
    if (isAndroid) {
      if (args) window.android[functionName](args);
      else window.android[functionName]();
    }
    if (isIOS) window.webkit.messageHandlers[functionName].postMessage(args);
  } catch (e) {
    // alert(e);
    console.log("catch jsb: " + functionName);
  }
}
```

```js
startReservation: function (args = null) {
    inner('startReservation', args)
},
```

但是逻辑也是有 js 回调的设计的:

```sh
beNotifiedToken <-> setToken
beNotifiedComment <=> postComment
beNotifiedScroollToComment <=>scrollToComment
```

可以考虑优化
