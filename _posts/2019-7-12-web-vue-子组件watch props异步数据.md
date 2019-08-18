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

## 场景2

prop把父亲的info对象给了子组件，但是子组件如何监控其变化？能否修改?
```js
computed: {
    aID: function() {
        return this.info['id'];
    }
}
```
上面的aID将报错，无`id`属性, 因为info是异步的,computed计算前不能保证info有值了.
这个和template直接使用有区别:
```html
<vue-markdown :source="post_info['feed_content']" :postrender="postRender"></vue-markdown>
```

<https://www.cnblogs.com/goloving/p/9114389.html>

前面提到过了思路，其实就是watch.

直接watch对象的item,看教程有这么做的，但是行不通?
```js
watch: {
    "me_info.follower" : function (val) {
        console.log('follower val:'+ val)
    },

    "me_info.id" : function (val) {
        console.log(' val:'+ id)
    }
}
```
那只能watch整个prop对象了:
```js
"post_info": function (newVal) {
    this.comment_count =
        this.url.post_type === 'news' ?
        newVal['comment_count']: newVal['feeds_comment_count']
}
```


