---
layout: post
title:
modified:
categories: Tech
tags: [vue, thinksns, web]
comments: true
---

<!-- TOC -->

- [登录](#登录)
  - [额外聊聊异步操作的封装](#额外聊聊异步操作的封装)
    - [Promise 解决 callback hell 问题](#Promise-解决-callback-hell-问题)

<!-- /TOC -->

## 登录

Login.vue 里，点击 button 登录:

```html
<form role="form" @submit.prevent="submit"></form>
```

```js
methods: {
    submit() {
       auth.login(access, password)
      .then(response => {
        this.$store.dispatch(USER_UPDATE, cb => {
          cb(response.data);
          this.$router.replace(this.$route.query.redirect || '/');
        });
      })
    }
}
```

- login api in auth.js

login 返回 Promise,后面才能 then 拿到.

```js
const login = (access, password) => request.post(...);
```

- 执行可以异步的 action

```js
//传入的参数是1个cb(cb1),闭包参数又是1个cb(cb2)
this.$store.dispatch(USER_UPDATE, cb => {
  //response.data来自前面login api的then返回
  cb(response.data);
  //router跳转，no history
  this.$router.replace(this.$route.query.redirect || "/");
});
```

action 的行为是:

```js
const actions = {
  // Create update auth user func.
  //context忽略.
  //执行cb,参数cb1,执行cb1,cb1的参数是cb2,
  //cb2就是user=>user => context.commit(USER_UPDATE, user)
  //神仙代码。。。并不直观.
  [USER_UPDATE]: (context, cb) =>
    cb(user => context.commit(USER_UPDATE, user), context)
};
```

action 里通过 commit 执行同步的 mutation,就是真正的动作:改变该 module 下 state 的 logged,user.

```js
const mutations = {
  // The func is update user state.
  [USER_UPDATE](state, user) {
    state.logged = !!user;
    state.user = {
      ...state.user,
      ...user
    };
    syncWindow(state.user);
  }
};
```

简单地说，就是异步拿数据，写到 store 里，然后 router 切路由...应该都是这个套路，不过一路哦组的艰辛.

对这种 cb 套 cb 的做法，还是绝的不够自然，如果用`async/await`改造下?

另外 getters 里的写法,用了 es2015 的参数解构:

```js
[USER_LOGGED]: ({ logged }) => logged,
```

- 常量函数

常量函数[USER_LOGGED], USER_LOGGED 应在一个 getter_type 的文件里

- 对象解构

```js
    ({ logged }) => logged：
    /_就是下面的简写:
    function(state) {
    return state.logged;
    }
```

### 额外聊聊异步操作的封装

#### Promise 解决 callback hell 问题

```js
function asyncAccess(url='xxx.com',cb)  {
  setTimeout(()=>{
      console.log('get Resource!')
      cb('Stuff');
    },1000);
}

//way 1
function getStuff(cb) {
 asyncAccess('nice.com', cb);
}

//way 2
function getStuff_p() {
  return new Promise( (resolve,reject) => {
    asyncAccess('nice.com', function(response){
      resolve(response);
        });
     });
}

//连着用的话 callback hell
let that = this;
getStuff( function(response1){
  console.log(response1);
  that.res = response1;
  getStuff(function(response2)) {
    console.log(response2);
    that.res = response2
    getStuff(function(response3)) {
      console.log(response3);
      that.res = response3;
    }
  }
  return that.res;
})

//Promise 链式调用 3次callback 当然Promise.all更好
let res = this.res;
getStuff_p((response1)=>{
  res += response1;
  return getStuff_p();
}).then((response2)=>{
  res += response2;
  return getStuff_p();
}).then((response3)=>{
  res += response3;
  return getStuff_p();
})

Promise.all( [getStuff_p(),getStuff_p(),getStuff_p() ]).then(response=>{
  //response是Array, 将3个结果并发的取到了.
})

```

1. axios

本身就是 Promise 实现的,比如 get 返回 Promise:

```js
axios.get = function(url) {
  return new Promise(resolve,reject) {
     http.accsess(url).callback(
       let ok;
       if(ok) (response)=>resove(response);
       else (response)=>reject(response)l
     )
  }
}
```

1. 项目里的方式

再封装下 axios,在 then 里增加 1 个回调，在 getSomtething 里定义即可.

```js
//定义api
getSomething(cb) {
  axios.get(url).then(response=>{ cb(respone)}).catch(...);
}

//使用api
getSomething((response)=>{
  //handle response;
})
```

更好的方式，自然是用 Promise 封装了

```js

getSomething( (response)=>{
  //p1
  return new Promise(resolve,reject){...}
})
```

内部 promise(p1)将决定外面返回的状态(reject or resolved)
