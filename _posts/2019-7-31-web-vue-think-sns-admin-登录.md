---
layout: post
title:
modified:
categories: Tech
tags: [vue,thinksns,web]
comments: true
---

<!-- TOC -->

- [登录](#登录)
    - [额外聊聊异步操作的封装](#额外聊聊异步操作的封装)
        - [Promise 解决callback hell问题](#promise-解决callback-hell问题)

<!-- /TOC -->


## 登录

Login.vue里，点击button登录:

```html
<form role="form" @submit.prevent="submit">
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

1. login api in auth.js

login返回Promise,后面才能then拿到.

```js
const login = (access, password) => request.post(...);
```

2. 执行可以异步的action，
```js
    //传入的参数是1个cb(cb1),闭包参数又是1个cb(cb2)
    this.$store.dispatch(USER_UPDATE, cb => {
        //response.data来自前面login api的then返回
        cb(response.data);
        //router跳转，no history
        this.$router.replace(this.$route.query.redirect || '/');
    });
```
action的行为是:
```js
const actions = {
  // Create update auth user func.
  //context忽略.
  //执行cb,参数cb1,执行cb1,cb1的参数是cb2,
  //cb2就是user=>user => context.commit(USER_UPDATE, user)
  //神仙代码。。。并不直观.
  [USER_UPDATE]: (context, cb) => cb(
    user => context.commit(USER_UPDATE, user),
    context
  ),
}
```
action里通过commit执行同步的mutation,就是真正的动作:改变该module下state的logged,user.
```js
const mutations = {
  // The func is update user state.
  [USER_UPDATE] (state, user) {
    state.logged = !!user;
    state.user = {
      ...state.user,
      ...user
    };
    syncWindow(state.user);
  },
}
```

简单地说，就是异步拿数据，写到store里，然后router切路由...应该都是这个套路，不过一路哦组的艰辛.

对这种cb套cb的做法，还是绝的不够自然，如果用`async/await`改造下?

另外getters里的写法,用了es2015的参数解构:
```js
[USER_LOGGED]: ({ logged }) => logged,
```
1.  常量函数
常量函数[USER_LOGGED], USER_LOGGED应在一个getter_type的文件里
2. 对象解构
 ({ logged }) => logged：
/*就是下面的简写:
    function(state) {
        return state.logged;
    }
*/

### 额外聊聊异步操作的封装

#### Promise 解决callback hell问题
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

0. axios

本身就是Promise实现的,比如get返回Promise:

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

再封装下axios,在then里增加1个回调，在getSomtething里定义即可.

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
更好的方式，自然是用Promise封装了
```js

getSomething( (response)=>{
  //p1
  return new Promise(resolve,reject){...}
})
```
内部promise(p1)将决定外面返回的状态(reject or resolved)