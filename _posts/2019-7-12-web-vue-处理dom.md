---
layout: post
title:
modified:
categories: Tech
tags: [web, vue]
comments: true
---

<!-- TOC -->

- [直接处理 dom](#直接处理-dom)

<!-- /TOC -->

## 直接处理 dom

vue 不建议直接搞 dom.但现在`<vue-markdown>xxx<vue-marodown>`需要在里面的 img 标签做一些事情,比如,`占位图`,`点击回调`..

应该在`mounted()`的周期里处理吗？

错！是在 updated()里...但是注意修改了 data,updated 又会重新进..

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
