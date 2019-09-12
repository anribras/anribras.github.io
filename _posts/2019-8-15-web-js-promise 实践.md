---
layout: post
title:
modified:
categories: Tech
tags: [javascript,web]
comments: true
---
<!-- TOC -->

- [Promise实践](#promise实践)

<!-- /TOC -->

## Promise实践

需求就是常见的缓存，如果有缓存使用缓存，没有api拉.

1. 链式，逻辑清晰
P.then().then().catch()
2. then chain如果中间不想返回了怎么办
Promise.reject in then
3. Promisify,既支持回调，又支持Promise
就是function还是有callback但是整体作为1个Promise返回.
这里用async包装，和new Promise一个效果.
async函数就是返回Promise.

```js
fetchImgUrl: async function (url, fn, cached = true) {
    if (cached) {
      //Best practice for Promise then chain with async/await ....
      return await cache.store.getItem(url).then(value => {
        if (!value) {
          console.log('1st time');
          return api.get(url + '?json=true', {})
        } else {
          fn(value);
          // 中断then链条,
          // throw error to stop then chain
          // throw new Error('Already cached')
          //or reject , better
          return Promise.reject('Already cached');
        }
      }).then((response) => {
          console.log(response)
          fn(response.data.url);
          return response.data.url;
      }).then(response => {
          cache.store.setItem(url, response)
          console.log(`cache: ${response} ok`);
      }).catch(e => {
        console.log(e);
      })
    } else {
      return axios.get(url + '?json=true', {}).then((response) => {
          fn(response.data.url);
        }
      ).catch(e=>{
        console.log(e);
      });
    }
```

Uny1vDY0cTROJ9vbBgAUlzCScwATp6LY

![20190912110134.png](https://images-1257933000.cos.ap-chengdu.myqcloud.com/undefined20190912110134.png)


![20190912110204.png](https://images-1257933000.cos.ap-chengdu.myqcloud.com/undefined20190912110204.png)
