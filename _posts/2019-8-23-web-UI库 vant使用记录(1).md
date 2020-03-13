---
layout: post
title:
modified:
categories: Tech
tags: [vant,web]
comments: true
---


<!-- TOC -->

- [postcss](#postcss)
    - [已有设计](#已有设计)
    - [postcss-plugin-pxtorem](#postcss-plugin-pxtorem)

<!-- /TOC -->

看了mint,vant,vux,vutify等流行的,感觉vant的比较清爽,拿来试试


## postcss

第1个问题是字体适配.按官方的是将px转rem,那一开始就得用px. 由于我已经采用了rem布局, 
所以我的做法是反过来,将vant的px转rem,rem的自适应方式仍然按自己的来:

### 已有设计
```html
html: {font-size: 100px}
body: {font-size: 1rem}
```
相当于1rem=100px; 1px=0.01rem

js里根据不同width调整html的font-size,我们的设计稿是375:
```js
  fontSizeAuto : function() {
    let viewportWidth = window.outerWidth || document.body.clientWidth || document.documentElement.clientWidth;
    //set html font-size. 3.75 from design
    let res =  viewportWidth / 3.75
    if(res > 200) {
          res = 200
    }
    document.documentElement.style.fontSize = res + 'px !important';
  }
fontSizeAuto();
window.addEventListener('resize', fontSizeAuto);
```

### postcss-plugin-pxtorem

安装<https://www.npmjs.com/package/postcss-plugin-px2rem>
 
使用看lm4的说明: <https://laravel-mix.com/docs/4.1/css-preprocessors>

```js
mix.i18n().js(['src/app.js'],'js')
  .options({
    postCss: [
      require('postcss-plugin-px2rem')({
        selectorBlackList: ['van-circle__layer'],
        rootValue: 100
        // propList: ["*"]
      }),
      require("autoprefixer")
    ]
  }).version()
```