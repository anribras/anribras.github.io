---
layout: post
title:
modified:
categories: Tech
tags: [vue, web]
comments: true
---

<!-- TOC -->

- [laravel-mix 5](#laravel-mix-5)
  - [webpack.mix.js](#webpackmixjs)
- [babel 7 支持](#babel-7-支持)
- [elemet-ui](#elemet-ui)
  - [1个有意思的问题](#1个有意思的问题)
- [vue 全家桶](#vue-全家桶)

<!-- /TOC -->

基于tp的package

## laravel-mix 5

先删掉原来的laravel-mix 2，用最新的laravel-mix 5

### webpack.mix.js

配置1个browser sync:

```js
//browser sync
mix.browserSync({
  notify: false,
  proxy: 'http://localhost:8000',
  files: [
    './resources/views/**/*.blade.php',
    './resources/assets/**/*.js',
    './resources/assets/**/*.scss',
    './resources/assets/**/*.vue',
  ],
  reloadOnRestart: true,
  watchOptions: {
    usePolling: false,
  },
});
```

## babel 7 支持

<https://juejin.im/post/5c03a4d0f265da615e053612>

## elemet-ui

yarn add即可

### 1个有意思的问题

el-select的change函数只能接受一个参数，即改变的value,

现在我传入1个额外的数据进来，怎么办?
<https://www.cnblogs.com/mmzuo-798/p/10438071.html>

确实就是闭包...这个厉害了.

## vue 全家桶

vue vuex vue-router
