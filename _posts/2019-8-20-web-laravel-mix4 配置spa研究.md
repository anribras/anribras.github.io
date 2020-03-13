---
layout: post
title:
modified:
categories: Tech
tags: [laravel,web]
comments: true
---

<!-- TOC -->

- [前面](#前面)
- [laravel-mix 4](#laravel-mix-4)
    - [extract](#extract)
- [babel](#babel)

<!-- /TOC -->


## 前面

一直不太愿意碰构建这块,在别人项目基础上用laravel-mix已经够省心了

但还是决定要深入了解下...

## laravel-mix 4

<https://laravel-mix.com/docs/4.1/upgrade>

```sh
Upgraded to webpack 4.
Upgraded to vue-loader 15.
Upgraded to Babel 7.
Babel config merging strategy:
    还是提供.babelrc,不是7应该用babel.config.js了?
```

因为走的laravel,所以就laravel-mix了,vue全家桶的vue cli3也是干这个的...都是based on webpack

### extract

extract()是把dependencies的里包打包为vendor.js, 应用为app.js ,vendor不常便,缓存起来,优化缓存策略

```js
//设置manifest.json的输出目录,也就是dist的根目录
mix.setPublicPath('../../public/spa');
mix.config.webpackConfig.output = {
  // chunkFilename: 'chunk/[name].[chunkhash].js',
  chunkFilename: '[name].chunk.js',
};
mix.i18n()
  //这里的第2个参数,js就是相对上面设置的publicPath
  .js('src/app.js','js').version()
  .sass('sass/post.scss','css').version()
//将原来的app.js分为manifest.js , vendor.js 和app.js,在blade里分别引用就好.
mix.extract();
```
生成的目录如下:
```sh
├── css
│   └── post.css
├── js
│   ├── app.chunk.js
│   ├── manifest.js
│   └── vendor.chunk.js
└── mix-manifest.json

```
之前有点蒙圈,好像不知道怎么用了,其实配合好mix函数,就好,不然为啥用larave-mix呢?

spa.blade.php:
```html
<script type="text/javascript" src="{{mix('spa/js/manifest.js','spa')}}" ></script>
<script type="text/javascript" src="{{mix('spa/js/app.chunk.js','spa')}}" ></script>
<script type="text/javascript" src="{{mix('spa/js/vendor.chunk.js','spa')}}" ></script>
```
按上面的写法
mix函数:
```
参数1是告诉blade相对public目录,去找文件 
参数2是告诉相对public到哪里找manifest.json,也就是`public/spa`.
```

但是,如果写的相对目录,它会自动根据setPublicPath来推断

```js
<script type="text/javascript" src="{{mix('spa/js/manifest.js','spa')}}" ></script>
<script type="text/javascript" src="{{mix('spa/js/app.chunk.js','spa')}}" ></script>
<script type="text/javascript" src="{{mix('spa/js/vendor.chunk.js','spa')}}" ></script>
```



## babel

知识点:

* babel6->babel7

* babelrc->babel.config.js

monorepo工程怎么区分两者的:

<https://blog.csdn.net/weixin_34195546/article/details/88567067>

* stage0-stage1

* preset

```
@vue/babel-preset-app: 
    它会把 useBuiltIns: 'usage' 传递给 @babel/preset-env，这样它会根据源代码中出现的语言特性自动检测需要的 polyfill。这确保了最终包里 polyfill 数量的最小化。然而，这也意味着如果其中一个依赖需要特殊的 polyfill，默认情况下 Babel 无法将其检测出来。
```

* plugin


