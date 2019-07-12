---
layout: post
title:
modified:
categories: Tech
tags: [web,vue]
comments: true
---

<!-- TOC -->

- [场景1](#场景1)

<!-- /TOC -->

## 场景1

首先1个无关的,即vue class绑定的方式: 

`isFollowing`决定following, unfollow的类是否添加到button里.
```html
        <button type="button" class="user-follow-btn col-sm-1" :class="{following: isFollowing, unFollow: !isFollowing}">
            Follow
        </button>
```

重点:

`me_info`来自props父给子传来的数据,注意是`异步`的数据;

因为要改写改变me_info的1个key，(知识点:props最好只读),需要`isFollowing`保存并改写这个状态

这里采用了`watch`.(因为异步的原因)

```js
data()  {
    return {
        followText: "",
        isFollowing: false,
        id: 0
    }
},
watch: {
    "me_info" : function (newVal, oldVal) {
        this.id = newVal.id;
        this.isFollowing = (typeof newVal.follower ==="undefined") ? false : newVal.follower  ;
        if(this.isFollowing === true) {
            //followText联动.用data or computed attr都行.
            this.followText = "Following";
        }
    },
}
```


