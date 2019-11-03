---
layout: post
title:
modified:
categories: Tech
tags: [vue, thinksns, spa, web]
comments: true
---

<!-- TOC -->

- [分析FeedList](#分析FeedList)
  - [数据拉取](#数据拉取)
  - [数据使用](#数据使用)
  - [总结](#总结)
- [NewsList](#NewsList)

<!-- /TOC -->

## 分析FeedList


以Feedlist.vue为例,看看vuex的state,getter,mutation,action到底怎么配合用的:

### 数据拉取

数据拉取下拉刷新，上拉刷新的地方:

```js
    <JoLoadMore
    ref="loadmore"
    @onRefresh="onRefresh"
    @onLoadMore="onLoadMore"
    >
```

onRefresh就是上拉刷新,Feed有3个tab:new,hot,follow

```js
    async onRefresh () {
      const type = this.feedType.replace(/^\S/, s => s.toUpperCase())
      const action = `feed/get${type}Feeds`
      //获取最新的不需要after
      const data = await this.$store.dispatch(action, { refresh: true })
      this.$refs.loadmore.afterRefresh(data.length < limit)
    },
    async onLoadMore () {
      const type = this.feedType.replace(/^\S/, s => s.toUpperCase())
      const action = `feed/get${type}Feeds`
      //根据after的id分页拉取
      const data = await this.$store.dispatch(action, { after: this.after })
      this.$refs.loadmore.afterLoadMore(data.length < limit)
    },
  },
```

action异步拉数据。哪里定义?肯定是vuex module的feed.js:

```js
const actions = {
      async getNewFeeds ({ commit }, payload) {
    //上面传来的refresh为true
    const { after, refresh = false } = payload
    //真正的api在这里
    const { data } = await api.getFeeds({ type: 'new', after })
    const { feeds = [], pinned = [] } = data
    //pinned和普通的处理稍微不同,
    commit(TYPES.SAVE_PINNED_LIST, { list: pinned })
    //普通列表 保存
    commit(TYPES.SAVE_FEED_LIST, { type: 'new', data: feeds, refresh })
    return feeds
  },
}
```

`async/await`异步转同步;

拉取数据到state里,并且第一时间缓存,缓存则是通过mutation，同步的.

```js
const state = {
  list: {
    hot: lstore.getData('FEED_LIST_HOT') || [], // 热门动态列表
    new: lstore.getData('FEED_LIST_NEW') || [], // 最新动态
    follow: lstore.getData('FEED_LIST_FOLLOW') || [], // 关注列表
    pinned: lstore.getData('FEED_LIST_PINNED') || [], // 置顶列表
  },
}
```

`refresh`参数决定要不要更新，

```js
const mutations = {
  [TYPES.SAVE_FEED_LIST] (state, payload) {
    const { type, data, refresh = false } = payload
    //refresh=true，就缓存最新的， 否则，旧的在list前面，data在后面?
    const list = refresh ? data : [...state.list[type], ...data]
    state.list[type] = list
    //更新缓存
    lstore.setData(`FEED_LIST_${type.toUpperCase()}`, list)
  },
```

- 逻辑分析

假设1次拉10个，已有10(1-10)个feed文章， 已缓存到FEED_LIST_NEWS里.

拉取时发现有最新的,11-15,那么直接保存15-6，剩余的靠底部的Loadmore重新拉取(5-1)

感觉是有点浪费的,一旦有新的，缓存要整体的更新.

- 如果改进

刷新时，记录下当前最新的，localstorage是否可以写map? obj: true 表示缓存:

[obj,true][obj,false]

### 数据使用

自然是通过getter:

```js
const getters = {
  pinned (state) {
    return state.list.pinned
  },
  hot (state) {
    return state.list.hot
  },
  new (state) {
    return state.list.new
  },
  follow (state) {
    return state.list.follow
  },
}
```

getter用`mapGetters`,合并到了组件的computed属性里:

可以看到Feedlist.vue里是不存任何数据的，全部在vuex里.

再通过query确定feedType，得到1个新的计算属性feeds, 在template里就查询feeds就好.

```js
  computed: {
      //来自feed module
    ...mapGetters('feed', ['hot', 'new', 'follow', 'pinned']),
        feeds () {
      return this[this.feedType]
    },
    feedType: {
      get () {
        return this.$route.query.type || 'hot'
      },
      set (val) {
        const { query } = this.$route
        this.$router.replace({ query: { ...query, type: val } })
      },
    },
```

```js
    <li
    v-for="(card, index) in feeds"
    :key="`feed-${feedType}-${card.id}-${index}`"
    :data-feed-id="card.id"
    >
    <FeedCard v-if="card.user_id" :feed="card" />
    <FeedAdCard v-if="card.space_id" :ad="card" />
    </li>
```

### 总结

```sh
1 数据获取与使用;
2 不同的分类又如何统一处理; news/hot/follow
3 置顶和普通的处理.
```

## NewsList

这个数据又是直接放在组件里了.

news不是每个用户能投，涉及到`投稿认证`,这个逻辑暂时不分析，准备另开一篇来做.

![20190912181812.png](https://images-1257933000.cos.ap-chengdu.myqcloud.com/undefined20190912181812.png)