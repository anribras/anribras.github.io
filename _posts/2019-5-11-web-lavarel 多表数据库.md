---
layout: post
title:
modified:
categories: Tech
tags: [lavarel]
comments: true
---

<!-- TOC -->

- [1对1](#1对1)
    - [问题](#问题)
- [1对多](#1对多)
    - [多对多](#多对多)

<!-- /TOC -->

## 1对1

`user`,`userprofile`表.对应2个model.

user 主键: id
userprofile: 主键:自己的id 有1个`user_id`字段

对user:
```php
public function profile()
{
    return $this->hasOne(UserProfile::class);
}

$user->profile();
```

对profile:
```php
public function user()
{
    return $this->belongsTo(User::class);
}
$profile->user();
```

本质就是`关联查询`,封装起来了,确实作者经过了很好的思考，服.
```sql
select hobby,nickname from profile where `user_id`=xxx
```

### 问题

10个post, 做文章列表，需要用到post-user-profile,　按上面的1对1，至少要10次查询.
用`with`解决. 
```php
$posts = Post::with('author')->where('views', '>', 0)->offset(1)->limit(10)->get();
```
让查询只有2次.第1次查文章,第2次查author:
```sql
select * from `users` where `users`.`id` in (?, ?, ?, ?, ?, ?)
```

兜兜转转，打补丁...直接用sql不就完了...



## 1对多
基于上面的,1对多就很好理解了,  相当与把该用户的所有post都返回了.
```php
public function posts()
{
    return $this->hasMany(Post::class);
}
$user = User::findOrFail(1);
$posts = $user->posts;
``` 

### 多对多

tags表: id 
post表: id
post_tags是个中间表: 
```php
public function up()
{
    Schema::create('post_tags', function (Blueprint $table) {
        $table->increments('id');
        $table->integer('post_id')->unsigned()->default(0);
        $table->integer('tag_id')->unsigned()->default(0);
        //联合索引，加速selete * from table where a=XX and b=XX的查找
        $table->unique(['post_id', 'tag_id']);
        $table->timestamps();
    });
}
```

```php
//in Post model
public function tags()
{
    return $this->belongsToMany(Tag::class, 'post_tags');
}
//in Controller login
$post = Post::with('tags')->find(1);
//通过post查询它关联的tag
$tags = $post->tags;


//in Tag model
public function posts()
{
    return $this->belongsToMany(Post::class, 'post_tags');
}

//
$tag = Tag::with('posts')->where('name', 'ab')->first();
//通过tag查询它关联的文章
$posts = $tag->posts;
```
还是太不直观了，也没省多少事.

