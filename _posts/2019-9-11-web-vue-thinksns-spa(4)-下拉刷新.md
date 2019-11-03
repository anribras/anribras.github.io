---
layout: post
title:
modified:
categories: Tech
tags: [vue, thinksns, spa, web]
comments: true
---

<!-- TOC -->

- [分析JoLoadMore.vue](#分析JoLoadMorevue)

<!-- /TOC -->

## 分析JoLoadMore.vue

下拉刷新很常见，核心是拦截默认的scroll/moving动作，用translateY来代替:

```js
      if (this.dragging && isPull && window.scrollY <= 0) {
        // 阻止 原生滚动 事件
        e.preventDefault()
        //dy就是translateY的距离
        this.dY = offsetY
        this.pulling = true
        this.$emit('onPull', this.dY)
      }
```

第2个，下拉刷新不是一个独立的组件，需要配合1个可以scroll的元素:用于计算scrollTop的值.

```js
function getScrollTarget (el) {
  while (
    el &&
    el.nodeType === 1 &&
    el.tagName !== 'HTML' &&
    el.tagName !== 'BODY'
  ) {
    const overflowY = document.defaultView.getComputedStyle(el).overflowY
    if (overflowY === 'scroll' || overflowY === 'auto') {
      return el
    }
    el = el.parentNode
  }
  return document.documentElement
}
  mounted () {
    this.scrollTarget = getScrollTarget(this.$el)o
    }
```
