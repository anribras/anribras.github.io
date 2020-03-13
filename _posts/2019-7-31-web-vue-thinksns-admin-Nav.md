---
layout: post
title:
modified:
categories: Tech
tags: [vue, thinksns, web]
comments: true
---

<!-- TOC -->

- [不同 package 对应不同的 admin iframe 页面](#不同-package-对应不同的-admin-iframe-页面)

<!-- /TOC -->

## 不同 package 对应不同的 admin iframe 页面

vue 文件里:

```html
<iframe :class="$style.appIframe" :src="uri"></iframe>
```

```js
  computed: {
    ...mapGetters({
      manages: MANAGES_GET
    }),
    uri() {
      //拿到url的参数,比如key=0;
      const { key } = this.$route.params;
      //知识点1 ES6解构
      //知识点2 字面量标记,也即是key这样的变量可以当成obj的key用.
      //{uri} = {} , uri可以为undefined
      const { [key]: { uri } = {} } = this.manages;

      //把manages里的key的value取出来,就是这个uri了..
      return uri;
    }
  }
```

manages 的哪里来的? manager 的 store module 里定义的:

```js
const getters = {
  [MANAGES_GET]: state => state.manages
};
```

自然地，肯定是通过 mutation 设置的:

```js
const mutations = {
  [MANAGES_SET](state, manages) {
    state.manages = manages;
  }
};
```

按前面的套路，mutation 通过 action:

```js
const actions = {
  [MANAGES_SET]: (context, cb) =>
    cb(manages => context.commit(MANAGES_SET, manages), context)
};
```

最终是在 Nav.vue 的 created 里获取的...

```js
  created() {
    this.$store.dispatch(MANAGES_SET, cb => request.get(
        ///axios GET api/manages...
      createRequestURI('manages'),
      { validateStatus: status => status === 200 }
    ).then(({ data = [] }) => {
      cb(data);
    })
  }
```

很喜欢在这个 action 的参数里搞事情...

增加 package 的 nav 扩展在这:

```html
<router-link
  class="list-group-item __button"
  v-for="(item, index) in manages"
  :key="item['uri']"
  :to="`/package/${index}`"
  active-class="active"
  exact
></router-link>
```

也就是点击后，直接 router 到`xxx/admin#/package/0`下, 回到刚开始具体的 iframe 内容了

再扣下...iframe 的 src 到底是什么内容? 是`http://localhost:8000/xx/issue-feedback`

这个自然是 TP package 开发的 admin route 里定义的:

```php

Route::group(['prefix' => 'xx/issue-feedback'], function (RouteRegisterContract $route) {
    // Home router.
    $route->get('/', Admin\HomeController::class.'@index')->name('issue-feedback:admin-home');
});
```

到底怎么绑定上去的呢..telescope 查看下 request：`HomeController@showManages`.
就是它绑定的了:

```php
    public function getManages(): array
    {
        $manages = [];
        foreach (static::$manages as $item) {
            $name = $item['name'];
            $uri = $item['uri'];
            $option = $item['option'];

            $isRoute = $option['route'] ?? false;
            $parameters = (array) ($option['parameters'] ?? []);
            $absolute = $option['absolute'] ?? true;
            $icon = $option['icon'] ?? null;

            $manages[] = [
                'name' => $name,
                'icon' => $icon,
                'uri' => ! $isRoute ? $uri : route($uri, $parameters, $absolute),
            ];
        }

        return $manages;
    }
```

`static::$manages`是 package 创建时 RouteServiceProvider.php 里提供的:

```php
{
    // Publish admin menu.
    $this->app->make(ManageRepository::class)->loadManageFrom('issue-feedback', 'issue-feedback:admin-home', [
        'route' => true,
        'icon' => '📦',
    ]);
}
```

uri 要么来自 option 里的设置，要么来自 route 函数返回.
route 函数返会 1 个命名路由的 uri:

```php
route("issue-feedback:admin-home",xx,xx);
//命名路由的定义:
Route::group(['prefix' => 'ff/issue-feedback'], function (RouteRegisterContract $route) {

    // Home router.
    $route->get('/', Admin\HomeController::class.'@index')->name('issue-feedback:admin-home');
});

```
