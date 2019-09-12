---
layout: post
title:
modified:
categories: Tech
tags: [vue, thinksns, spa, web]
comments: true
---

<!-- TOC -->

- [组件挂载](#组件挂载)
    - [指定 el](#指定-el)

<!-- /TOC -->

2 个 plugin 实现，不同 Vue 里直接挂组件，是通过\$mount 手动挂载

## 组件挂载

### 指定 el

全局 Vue 实例化后,挂载到 id=app 的元素，并替换它:

```js
new Vue {
  el: '#app'
  ...
}
```

```html
<div id="app">
  <author-header
    :author-info="authorInfo"
    :me-id="meId"
    :me-follow="meFollow"
    :post-info="postInfo"
  >
  </author-header>
  <post-content-vue :post-info="postInfo">
    <comment-pagination
      :post-info="postInfo"
      ref="cmt_page_r"
    ></comment-pagination>
  </post-content-vue>
</div>
```

2. mount 指定 HtmlElement

```js
new Vue {
  ...
}.mount('#app')
```

3. 手动 mount

```js
const MessageInstance = new Vue({
  data: _props,
  render(h) {
    return h(Message, {
      props: _props
    });
  }
});

const component = MessageInstance.$mount();
document.body.appendChild(component.$el);
```
