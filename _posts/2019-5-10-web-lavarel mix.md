---
layout: post
title:
modified:
categories: Tech
tags: [lavarel]
comments: true
---

<!-- TOC -->

- [目录](#目录)
- [引用模块](#引用模块)
- [h5 使用](#h5-使用)
- [基本操作](#基本操作)

<!-- /TOC -->

## 目录

`resource/xx`　放预编译文件

生成希望是`public/dist`.

webpack.mix.js:

```js
let mix = require("laravel-mix");
let path = require("path");
mix.setPublicPath(path.join("public", "dist"));
mix.sourceMaps(!mix.inProduction());
mix.disableNotifications();
mix
  .sass("resources/xx/sass/xx-product.scss", "css")
  .options({})
  .version();

mix.js("resources/xx/js/xx-product.js", "public/dist/js").version();

//publicpath not working ..
//image直接输入到dist/images h5里直接引用
mix.copy("resources/xx/dist/images", "public/dist/images").version();
```

version()后，注意看生成的 manifest.json 文件:

```json
"/css/bootstrap.css": "/css/bootstrap.css?id=811d86ba49ce2d26552d",
```

## 引用模块

自己下好原始的文件,再到 webpack 里合并太 low 了点,版本也没随意控制.

文件来自`npm install` or 'yarn add'到 node_modules 下的包.

下面是在自己的 js 引用其他模块的方法:
js:

```js
try {
  // window.$ =  window.Jquery = require('jquery');
  window.$ = window.Jquery = require("jquery");
  window.Swiper = require("swiper");
} catch (e) {}
```

Q:`zepto`不能加载成功?

scss:

```scss
@import "~swiper/dist/css/swiper";
```

## h5 使用

用 lavarel 的 mix 和 asset 函数.
`asset`直接以 public 为根目录，不会加前面标记的 version.

```sh
<script type="text/javascript" src="{{asset('dist/js/zepto.min.js')}}"></script>
```

`mix`是我想要的，加 version，更新缓存,注意调用方式.

```sh
<script type="text/javascript" src="{{mix('/js/xx-product.js','dist')}}"></script>
```

## 基本操作

```sh
yarn run dev
yarn run prod
yarn add xxx
yarn install(可略)
yarn run watch
```
