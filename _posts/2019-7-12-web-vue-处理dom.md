---
layout: post
title:
modified:
categories: Tech
tags: [web,vue]
comments: true
---

<!-- TOC -->

- [直接处理dom](#直接处理dom)

<!-- /TOC -->
## 直接处理dom

vue不建议直接搞dom.但现在`<vue-markdown>xxx<vue-marodown>`需要在里面的img标签做一些事情,比如,`占位图`,`点击回调`..

应该在`mounted()`的周期里处理吗？

错！是在updated()里...但是注意修改了data,updated又会重新进..

保证不修改的前提下，可以这样处理:
```js
updated() {
    if(!this.updated_flag ) {

        this.$nextTick(function () {
            let m = this.$refs.mk;
            let images = m.$el.getElementsByTagName("img");
            console.log(images)
            this.imageHandler()
        });

        this.updated_flag = true;
    }
},
```