---
layout: post
title:
modified:
categories: Tech
tags: [vue,thinksns,web]
comments: true
---

<!-- TOC -->

- [app.js](#appjs)
    - [BOOTSTRAPPERS](#bootstrappers)

<!-- /TOC -->


## app.js

```
1 vue keep-alive缓存 router-view
2 watch $route ,取route.meta作为标题，标题又同步到document.title上
3 created里通过mapAction引入初始化：BOOTSTRAPPERS,
```

### BOOTSTRAPPERS
这是个action:
```js
  async BOOTSTRAPPERS ({ commit, dispatch }) {
    bootApi.getBootstrappers().then(({ data: bootstrappers = {} }) => {
      commit('BOOTSTRAPPERS', bootstrappers)
      dispatch('currency/updateCurrencyUnit')
    })
  },
```
`bootApi.getBootstrappers()`是访问`/bootstrappers`的api,拉取数据后,
commit 1个mutation和dipatch 1个action:

mutation保存上面取到的config:
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
最终就是存到了currency state的unit里:
```js
  [TYPES.UPDATE_CURRENCY_UNIT] (state, payload) {
    const { unit } = payload
    unit && (state.unit = unit)
  },
```
最终是改变了类似`积分`or`xx币`or的命名
```yml
currency:
  name: '@:(currency.unit)'
  unit: 积分
 ```

## i18n.js
locale配置文件来自.yaml 而不是.json, 前者更灵活点

## bus.js

之前理解的bus是组建通信的.
很多处理gif的代码，可以单开个文章来分析.

## router

## vuex

store模式

state,getter,mutation,action,module.

## vuex-router-sync

把store里的state按module组织后，再对应到响应的router-->组件

这个组件是为了,在store里方便的取router.
```js
this.$store.state.route
```
<https://blog.csdn.net/vv_bug/article/details/84064708>


