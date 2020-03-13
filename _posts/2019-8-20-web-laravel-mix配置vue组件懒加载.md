---
layout: post
title:
modified:
categories: Tech
tags: [laravel,web]
comments: true
---

<!-- TOC -->

- [安装依赖](#安装依赖)
- [.baberc](#baberc)
- [webpack.config.js](#webpackconfigjs)
- [引入组件改为:](#引入组件改为)

<!-- /TOC -->

larave-mix version: v4.1.2

### 安装依赖
```sh
yarn add babel-plugin-syntax-dynamic-import --save-dev
yarn add  babel-plugin-dynamic-import-webpack --save-dev
```

### .baberc

增加:
```sh
    "plugins": [
        ["syntax-dynamic-import"]
    ]
```

### webpack.config.js

增加:
```sh
mix.config.webpackConfig.output = {
  chunkFilename: 'js/[name].bundle.js',
  publicPath: 'public/dist/js',
};
```

### 引入组件改为:
```js
//Vue.component('post-content-vue', require('../components/PostContent'));

PostContent = ()=>import(/* webpackChunkName: "post-content" */'../components/PostContent');
Vue.component('post-content-vue', PostContent);
```

Enjoy.
