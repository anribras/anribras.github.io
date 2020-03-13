---
layout: post
title:
modified:
categories: Tech
tags: [javascript,web]
comments: true
---

<!-- TOC -->

- [finally实现](#finally实现)

<!-- /TOC -->

## finally实现

```js
Promise.prototype.finally = function (callback) {
  let P = this.constructor;
  return this.then(
    value  => P.resolve(callback()).then(() => value),
    reason => P.resolve(callback()).then(() => { throw reason })
  );
};
```

```sh
1. finally返回1个Promise:
2. P.resolve(callback());调用finnally的回调，创建1个新Promise, 该Promise返回value
```
