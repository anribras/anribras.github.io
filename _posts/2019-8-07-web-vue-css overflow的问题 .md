---
layout: post
title:
modified:
categories: Tech
tags: [web, vue]
comments: true
---

## 问题

1 个布局有 container ，通过 marin-left,right 确定了宽度

并且用 overflow-x: hidden 锁定横向不准滑动

但是现在用的元素(图加文字)超出 container 的空间,怎么办?

![Screenshot from 2019-08-07 10-27-12-d66d7a5c-4ed7-41f9-9720-81f0cc08c0f8](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202019-08-07%2010-27-12-d66d7a5c-4ed7-41f9-9720-81f0cc08c0f8.png)

## 解决

核心是加`postion: absolute`

<https://www.cnblogs.com/goloving/p/9275776.html>

```html
<div class="container">
  <div class="header">
    <div class="header-text"></div>
    <div class="poster">
      <div class="poster-text"></div>
    </div>
  </div>
</div>
```

如下即可.

```css
.container {
  margin-left: 0.24rem;
  margin-right: 0.24rem;
  overflow-x: hidden;
}
//需要解除overflow的限制 ,允许子元素绝对定位
.header {
  margin-left: -0.24rem;
  position: absolute;
}
//不需要解除overflow的限制 ,允许子元素绝对定位
.poster {
  margin-left: -0.24rem;
  position: relative;
}
//子元素绝对定位(文字悬浮)
.header-text {
  position: absolute;
}

//子元素绝对定位(文字悬浮)
.poster-text {
  position: absolute;
}
```
