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
    - [再次理解](#再次理解)
    - [渴求式加载with](#渴求式加载with)
- [1对多](#1对多)
    - [实例的例子](#实例的例子)
    - [1对多(包括1对1)的sql本质](#1对多包括1对1的sql本质)
- [多对多](#多对多)
    - [pivot model](#pivot-model)
    - [多对多的的本质](#多对多的的本质)
- [远程1对多](#远程1对多)
    - [和多对多的中间表的区别?](#和多对多的中间表的区别)
- [1对1 多态](#1对1-多态)
- [1对多 多态](#1对多-多态)
- [多对多 多态](#多对多-多态)
    - [总结](#总结)
- [其他](#其他)
    - [软删除](#软删除)
    - [关联查询+过滤](#关联查询过滤)
    - [关联查询的更新](#关联查询的更新)

<!-- /TOC -->

还是英文的讲清楚多了:

<https://laravel.com/docs/5.8/eloquent-relationships>

## 1对1

`user`,`userprofile`表.对应2个model.

userprofile表有user_id, user表是userprofile的扩展表.

user 主键是id

userprofile: 主键是自己的id 有1个`user_id`字段

需要理解belongsTo和hasOne的关系:

```sh
`user` hasOne `userprofile`
`userprofile` belongTo `user`:
    为啥叫`belongsto`?
    扩展表试图将主表分类，本例每个profile都 belongsto 1个user. 扩展表好像level更搞，
    所以主表belongto扩展表.
```



对user:
```php
public function profile(){
    return $this->hasOne(UserProfile::class);
}
$user->profile;
```

对profile:
```php
public function user(){
    return $this->belongsTo(User::class);
}
$profile->user;
```

### 再次理解

今天发现下面`hasOne` 和`belongsTo`是等价的:
```php
    //in issue model
    public function issue_cate() {
        return $this->belongsTo(IssueCate::class,'issue_cate_id','id'); //ok
        //return $this->hasOne(IssueCate::class,'id', 'issue_cate_id'); ok
    }
```

belongsTo就按之前的理解，但是hasOne why?仔细看看定义:
```php
public function belongsTo($related, $foreignKey = null, $ownerKey = null, $relation = null)

//其中第一个参数是关联模型的类名。第二个参数是关联模型(cate)在当前模型(issue)所属表的外键字段,第三个参数是关联模型类(cate)所属表的主键：

public function hasOne($related, $foreignKey = null, $localKey = null);

//其中，第一个参数是关联模型(cate)的类名，第二个参数是关联模型(cate)类所属表的外键，第三个参数是关联表的外键关联到当前模型(issue)所属表的哪个字段
```

为啥上面的hasOne可以代替belongsTo?
```sh
1. issues 关联到模型 cate;
2. 指定关联表的外键，这里直接把cate表的id作为外键，完全是可以的
3. 指定关联到当前模型的的字段,自然是issue_cate_id
```

总之:
```sh

表结构的角度来讲: 
A:  id, b_id:
B:  id;

A hasOne B: A是当前模型，B是关联模型，先指定B的里用的字段(id)，再指定A里的字段(b_id);
A belongsTo B: 先指定A里的字段(b_id),再指定A里的字段(id).

```

还是belongsTo更加好理解.



### 渴求式加载with

* n+1性能问题


根据某些条件查询出符合条件的10个profile:
```sql
select * from profiles where xxx and xxx;
```
需要给每个profile增加user信息. profile信息里有user_id
```sql
select * from users where  user.id = some_id_1;
select * from users where  user.id = some_id_2;
...
```
查询10个profile+user, 先1次select出所有的profile,然后针对每个profile需要做1次select user,
如果不加优化需要10+1次,这就是`n+1`的性能问题:

<https://segmentfault.com/a/1190000019515106?utm_source=tag-newest>

laravel提供with，很方便的解决该问题:

<https://blog.csdn.net/u013032345/article/details/82772938>

优化的本质是:
```sh
1 查询1次profile,得到10行;
2.把所有的author id,组成集合 sets(1,2,3,4,...)
3.再用1次`select * from users where user.id in sets`;得到所有的user.
4.把profile的结果和user的结果合并.
```

最终只用到了2次查询.但是疑问在于`In`的效率.

当然,上面的优化还可以用join表实现.join的缺点在于不能跨服务器:
```sql
select p.* u.* from profiles as p inner join users as u where p.user_id = u.id
```
Q: `In`和`join查询`的SQL效率问题? 这涉及到了SQL自身的查询优化.

* with in laravel

laravel的with就是上面的优化实现:

```php
$profile->with('user')
```

`with`默认会带上user里的所有字段，可指定仅需要的字段:
```php
$issue->->with( ['issue_cate:id,category','issue_status:id,status','issue_level:id,level' ])
```

`with`条件约束:
```php
$profile->with('user'=>function($query){
    $query->where('id','<',100);
})
```
* with and load
`load`: lazy渴求式加载，需要时才调用,更灵活;
```php
if($flag) {
    $profile->load('user');//前面没调用过with
}else {
    //xxx
}
```

## 1对多

基于上面的,1对多就很好理解了, user可发多个posts,每个posts表里的author_id肯定是一样的.

```php
//in user model, user has many posts
public function posts() {
    return $this->hasMany(Post::class);
}
$user = User::findOrFail(1);
$posts = $user->posts();
//in post model post belongto user. 
public function author() {
    return $this->belongsTo(User::class);
}
``` 


### 实例的例子

issue_back表是主表: 字段 id, issue_cate_id; 关联表是cate表。
```php
//in cate model:
public function issues()
{
    //选择关联id
    return $this->hasMany(IssueFeedback::class,'issue_cate_id','id');
}

// in issues model:
public function issue_cate() {
    //这里用的默认,issue_cate_id
    return $this->belongsTo(IssueCate::class);
}
```

### 1对多(包括1对1)的sql本质

```sh
posts: id, cate_id, title,content...
cates: id, name...
```

1. 查posts 带cate的信息:

对应`belongsTo`，不过laravel的model里默认返回的了整个
cate的对象而不是cate.name这个字段.
```sql
select n.*,c.name from news as n inner join news_cates as c on(n.cate_id=c.id)
```
2. 查某个cate的所有文章:

对应的hasMany关系,cate.name='News'为例:
```sql
select n.*,c.name from news as n inner join news_cates as c on(c.name='News');
```

当然join表的效率可能是个问题，可能的解决办法就是前面说的n+1优化思路.


## 多对多

tags表: id 
post表: id
post_tags是个`中间表`,核心就是它，也是它与`belongsTo`的区别 
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

相互belongsToMany.

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

### 多对多的的本质

<https://stackoverflow.com/questions/5158950/mysql-join-many-to-many-single-row>

就是两个join查询.

多对多的核心是`中间表`,除了简单的`A--A_B---A`,还有:

1. A_B可以与其他表(C)发生`1对1`，`1对多`的关系
```sh
A -- A_B ---B
    A belongsToMany B
    B belongstoMany A
C -- A_B
    C hasMany A_B
    A_B belongs to C
```
2. A和中间表A_B的关系，实际也是可以用1对多来描述:
```sh
// post_tags的某条记录属于某个tag or 某个post.
A_B belongsto A
A_B belongsto B
```

3. A_B表可能不止是A,B的中间表，也是C,D,E的中间表
```sh
A -- A_B_C_D --- B
A -- A_B_C_D --- C
A -- A_B_C_D --- D
```

理解上面的3种情况，对于设计更复杂嵌套的表关系,我认为至关重要。


## 远程1对多

1 country , 10 users , each users has 10 articles.
查出这个country的所有文章，远层的意思是中间隔了个users表.
users表要添加country_id.

### 和多对多的中间表的区别?

<https://stackoverflow.com/questions/21699050/many-to-many-relationships-in-laravel-belongstomany-vs-hasmanythrough>

<https://laravel.io/forum/03-04-2014-hasmanythrough-with-many-to-many>

多对多belongs-to-many关系中，使用时相互`belongsToMany`:

借助了中间表(pivot):

```sh
post belongToMany tags;
tag belongsToMany posts;

post表: id
post_tag表: id,post_id,tag_id.
tag表: tag_id
```

而has-many-through,也是3张表,看下id的区别:
```sh
country-->user->post

country表: id
user表: id, country_id
post表: id, user_id

country hasManyThrough('post','user').
    此时准确的说法是:country hasMany posts through user.
```
看上去是个`子查询`的问题.
```sql
select * from 'posts' where 'users_id' in (select * from 'users' where 'country_id'=xx);
```

```php
//in Country model
public function posts(){
    return $this->hasManyThrough(Post::class, User::class);
}
//use it
$country->posts;
```

## 1对1 多态
post 有唯一缩略图
用户　有唯一头像
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
$iamge->imageable//直接返回模型实例
```
imageable表: id，imageable_type,imageable_id.(来自post_id or author_id)

再看Post里怎么关联,就能返回自己的缩略图:
```php
//in User model
function image(){
    return $this->morphOne(Image:class,'imageable');
}
$user->image//返回头像
```




## 1对多 多态

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


## 多对多 多态

多对多的多态，也有`中间表`的概念.

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

### 总结

morphOne对应hasOne, morphTo对应 belongsTo

morphMany对应hasMany, morphTo对应 belongsTo

morphToMany对应belongsToMany morphByMany 对应belongToMany..



## 其他

### 软删除

<https://laravelacademy.org/post/138.html>

### 关联查询+过滤

注意不是join查询。我的理解就是做嵌套查询.(or 子查询)

以`一对多`为例子:
```sh
author hasMany posts;
```
```php

//直接返回某作者的所有文章的实例
$author->posts;

//使用方法时，返回的是关联关系实例，该实例注入了查询构建器
$author->posts()->get();

//查询方法，该怎么查就怎么查
$author->posts()->query()->when(...)->get();

//结果过滤,是根据关联查询的结果来过滤, 底层是用了sql的EXSIT
$users = User::has('posts')->get();
//仅需要发表文章数>1的用户.
$users = User::has('posts', '>', 1)->get();

$users = User::doesntHave('posts')->get();

//仅需要有评论or有tag的文章
$posts = Post::has('comments')->orHas('tags')->get();

//用whereHas orWhereHas基于闭包，查询更复杂的.
$users = User::whereHas('posts', function ($query) {
    $query->where('title', 'like', 'Laravel学院%');
})->get();


```

### 关联查询的更新

hasOne hasMany关系,用save create
```php
//author has many posts;
$post=  Post::findOrFail(1);
//添加文章到该author
$author->posts()->save($posts);
```
belongsTo关系 用associate/dissociate
```php
//posts belongs to user;
$author= User::findOrFail(1);

//更换(添加)文章的作者
$posts->author()->associate($authoer);
//解除绑定,post对应的user_id为null
$posts->author()->dissociate();
```

belongToMany(many-to-many)关系，用attach/detach，本质是往中间添加/删除记录.
```php
//post belongs-to-many tags;
$tag = Tag::findOrFail(1);
//添加文章的标签
$post->tags()->attach($tag)

//删除文章指定的标签
$post->tags()->detach($tag)

//删除文章的所有标签
$post->tags()->detach()
```
还有个`sync`方法,接受tag里的id为参数,其他id则都detach掉
```php
//为post
$post->tag()->sync([1, 2, 3]);
```

还有个triggle, 相当于attach->detach->attach的repeat
```php
$user->roles()->sync([1, 2, 3]);
```
