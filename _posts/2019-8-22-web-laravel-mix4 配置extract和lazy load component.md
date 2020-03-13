---
layout: post
title:
modified:
categories: Tech
tags: [laravel,web]
comments: true
---

<!-- TOC -->

- [gogo](#gogo)
- [Updated](#updated)

<!-- /TOC -->

## gogo

因为laravel-mix 4有些奇怪的问题,这里记录一下

想用以下2个功能:

* extract

extract()在lm4引入的.是把dependencies的包打包为vendor.js, 应用部分为app.js ,vendor不常变, 可缓存起来,优化访问.

* lazy load import

动态加载组件的js,不把组件打包到1个文件里.这是spa的组件化的一个基础.


```js
//设置manifest.json的输出目录,也就是dist的根目录
mix.setPublicPath('../../public/spa');

mix.config.webpackConfig.output = {
  // chunkFilename: 'chunk/[name].[chunkhash].js',
  chunkFilename: '[name].chunk.js',
  //为lazy load的componnets js文件设置url目录
  //比较重要
  publicPath: '/spa/'
};

mix.i18n()
  //第2个参数为输出目录,相对上面设置的setPublicPath
  .js('src/app.js','js').version()
  .sass('sass/post.scss','css').version()
//将原来的app.js分为manifest.js , vendor.js 和app.js,在blade里分别引用就好.
mix.extract();
```
生成的目录如下:
```sh
├── css
│   └── app.css
├── js
│   ├── app.chunk.js #改动较大的chunk,
│   ├── manifest.js
│   └── vendor.chunk.js # 提取vendor的,额外打包,方便缓存.
├── mix-manifest.json
└── sharebar.chunk.js #提取自组件,动态加载用
```

之前有点蒙圈,不知道怎么用了,其实配合好mix函数就好,不然为啥用larave-mix呢?mix函数,其实在往manifest.json里写内容:

参数1是告诉blade相对public目录,去找文件;

参数2是告诉相对public到哪里找manifest.json,也就是`public/spa`.

```sh
{
    "/js/app.chunk.js": "/js/app.chunk.js?id=2e268187bbacc5b29dc2",
    "/css/app.css": "/css/app.css?id=d41d8cd98f00b204e980",
    "/js/manifest.js": "/js/manifest.js?id=3a2343d5990c000f3bea",
    "/js/vendor.chunk.js": "/js/vendor.chunk.js?id=4eac5c3b3e61171821dc",
    "/sharebar.chunk.js": "/sharebar.chunk.js?id=a5f7808c006d9e0835cb"
}
```

spa.blade.php,写法1
```html
<script type="text/javascript" src="{{mix('spa/js/manifest.js','spa')}}" ></script>
<script type="text/javascript" src="{{mix('spa/js/app.chunk.js','spa')}}" ></script>
<script type="text/javascript" src="{{mix('spa/js/vendor.chunk.js','spa')}}" ></script>
```
按上面的写法ok,但是manifest.json没有真正work.
![Screenshot from 2019-08-22 11-01-08-ec1e23c7-a794-488c-956c-a4a244c20bfc](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202019-08-22%2011-01-08-ec1e23c7-a794-488c-956c-a4a244c20bfc.png)

正确写法:
```js
<script type="text/javascript" src="{{mix('js/manifest.js','spa')}}" ></script>
<script type="text/javascript" src="{{mix('js/app.chunk.js','spa')}}" ></script>
<script type="text/javascript" src="{{mix('js/vendor.chunk.js','spa')}}" ></script>
```

## Updated

遇到天坑 ,lazy load无法工作:
<https://github.com/JeffreyWay/laravel-mix/issues/2064>

生成的app.css为空:

爬坑姿势:
<https://github.com/JeffreyWay/laravel-mix/issues/1914>

最终是将webpack.mix.js分为2份,并且通过1个插件合并manifest.json...
<https://github.com/KABBOUCHI/laravel-mix-merge-manifest>




