---
layout: post
title:
modified:
categories: Tech
tags: [vue, thinksns, web,spa]
comments: true
---

<!-- TOC -->

- [app.js](#appjs)
  - [BOOTSTRAPPERS](#BOOTSTRAPPERS)
- [i18n.js](#i18njs)
- [bus.js](#busjs)
- [router](#router)
- [vuex](#vuex)
- [vuex-router-sync](#vuex-router-sync)

<!-- /TOC -->

## app.js

```sh
1 vue keep-alive缓存 router-view
2 watch $route ,取route.meta作为标题，标题又同步到document.title上
3 created里通过mapAction引入初始化：BOOTSTRAPPERS,
```

### BOOTSTRAPPERS

这是个 action:

```js
  async BOOTSTRAPPERS ({ commit, dispatch }) {
    bootApi.getBootstrappers().then(({ data: bootstrappers = {} }) => {
      commit('BOOTSTRAPPERS', bootstrappers)
      dispatch('currency/updateCurrencyUnit')
    })
  },
```

`bootApi.getBootstrappers()`是访问`/bootstrappers`的 api,拉取数据后,
commit 1 个 mutation 和 dipatch 1 个 action:

mutation 保存上面取到的 config:

```js
  BOOTSTRAPPERS (state, config) {
    state.CONFIG = config
    lstore.setData('BOOTSTRAPPERS', config)
  },
```

action 将`rootState.CONFIG.site.name`用`UPDATE_CURRENCY_UNIT`来再次提交,

```js
  updateCurrencyUnit ({ commit, rootState }) {
    const { currency_name: currency } = rootState.CONFIG.site
    commit(TYPES.UPDATE_CURRENCY_UNIT, { unit: currency.name })
  },

```

最终就是存到了 currency state 的 unit 里:

```js
  [TYPES.UPDATE_CURRENCY_UNIT] (state, payload) {
    const { unit } = payload
    unit && (state.unit = unit)
  },
```

最终是改变了类似`积分`or`xx币`or 的命名

```yml
currency:
  name: "@:(currency.unit)"
  unit: 积分
```

## i18n.js

locale 配置文件来自.yaml 而不是.json, 前者更灵活点

## bus.js

之前理解的 bus 是组建通信的.
很多处理 gif 的代码，可以单开个文章来分析.

## router

## vuex

store 模式

state,getter,mutation,action,module.

## vuex-router-sync

把 store 里的 state 按 module 组织后，再对应到响应的 router-->组件

这个组件是为了,在 store 里方便的取 router.

```js
this.$store.state.route;
```

<https://blog.csdn.net/vv_bug/article/details/84064708>
