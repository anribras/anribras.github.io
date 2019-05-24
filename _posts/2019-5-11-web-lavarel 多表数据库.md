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
    - [pivot model](#pivot-model)
- [远程1对多](#远程1对多)
- [多态1对1](#多态1对1)
- [1对多多态](#1对多多态)
- [多对多多态](#多对多多态)

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

本质就是`条件查询`,封装起来了,确实作者经过了很好的思考，服.
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
这叫`渴求式加载`.
兜兜转转，打补丁...直接用sql不就完了...


## 1对多

基于上面的,1对多就很好理解了, user可发多个posts,每个posts表里的author_id肯定是一样的.

```php
//in user model
public function posts()
{
    return $this->hasMany(Post::class);
}
$user = User::findOrFail(1);
$posts = $user->posts;
//in post model, change user to author more readable
public function author()
{
    return $this->belongsTo(User::class, 'user_id', 'id', 'author');
}
``` 

## 多对多

tags表: id 
post表: id
post_tags是个`中间表`,核心就是它，也是它与`hasMany`的区别 
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

### pivot model
```php
//Post model
function tags(){
    return $this->belongsToMany('Tag','post_tags')->withPivot('like_counts')->withTimestamps();_
}

$post->tags; //返回中间表post_tags里tag_id和额外指定的like_counts,和时间戳字段.

$post->pivot//就是用来专门访问中间表的.


return $this->belongsToMany(Tag::class, 'post_tags')->as('taggable')->withTimestamps();
//as更可读 , $post->taggable->...
return $this->belongsToMany(Tag::class, 'post_tags')->wherePivot('user_id', 1);
//可以用where过滤


return $this->belongsToMany('App\User')->using('App\RoleUser');
//自定义pivot表，　RoleUser extends Pivot
```



## 远程1对多

1 country , 10 users , each users has 10 articles.
查出这个country的所有文章，远层的意思是中间隔了个users表.
users表要添加country_id.
看上去是个`联合查询`的问题.
```sql
select * from 'posts' where 'users_id' in (select * from 'users' where 'country_id'=xx);
```
```php
//in Country model
public function posts()
{
    return $this->hasManyThrough(Post::class, User::class);
}

//use it
$country->posts;
```

## 多态1对1
post 有唯一1缩略图
用户　有唯一1头像
按上面理解，2个1对1的表就行了.但是实际都是存的图像，这个表是可以共享的,只是类型不同，这就是所谓多态.Image表就可以是定义为多态性质的
```php
function up(){
       Schema::create('images', function (Blueprint $table) {
        $table->increments('id');
        $table->string('url')->comment('图片URL');
        $table->morphs('imageable');
        $table->timestamps();
    }); 
}

//in Image model
public function imageable()
{
    return $this->morphTo();//省略了'imageable'，就是关联到了表的字段.
}
$iamge->imageable()//直接返回模型实例
```
imageable实际是2个字段，1个type,1个id.


再看Post里怎么关联,就能返回自己的缩略图:
```php
//in User model
function image(){
    return $this->morphOne(Image:class,'imageable');
}
$user->image()//返回头像了
```

## 1对多多态

`评论`也是如此.共同资源都可以这样,有很多种不同的评论，但是可以放到1个表里.
![Screenshot from 2019-05-14 10-16-57-be3797f1-3065-4e8c-b505-8817f3e6e440](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202019-05-14%2010-16-57-be3797f1-3065-4e8c-b505-8817f3e6e440.png)

稍微不同的是,`comments`可有有多条！这就是1对多.

```php
//in Page model
public function comments()
{
    return $this->morphMany(Comment::class, 'commentable');
}
//返回page的所有评论.
$page->comments；

```

## 多对多多态

简单举个例子：

用户有头像，但头像不但可以用到做头像，也用到去作为用户发文的第1张图，这张图就有了多对多的形态，type不但是user,而且还是post.

post tag的例子，tag不仅属于post,也可以属于pages,视频,问答...最好用1个Taggable表.

返回某个tag的所有posts or pages:
```php
public function posts()
{
    return $this->morphedByMany(Post::class, 'taggable');
}

public function pages()
{
    return $this->morphedByMany(Page::class, 'taggable');
}
```
返回post or page的tags:
```php
public function tags()
{
    return $this->morphToMany(Tag::class, 'taggable');
}
```





