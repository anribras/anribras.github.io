---
layout: post
title:
modified:
categories: Tech

tags: [wordpress]

comments: true
---

## 分析

button click 导致 callback,传递到 callback 的参数怎么来的?

```php
    <?php if( get_comment_pages_count() > 1 ){ ?>
    <div class="comment-nav text-center pt-3 pt-md-4">
        <div class="post-loading"><span></span><span></span><span></span><span></span><span></span></div>
        <button
            data-page="comments"
            data-query="<?php the_ID(); ?>"
            data-action="ajax_load_posts"
            data-paged="<?php echo get_next_page_number(); ?>"
            data-commentcount="<?php echo get_comment_pages_count(); ?>"
            data-commentspage="<?php echo get_option( 'default_comments_page' );?>"
            data-append="comment-list"
            class="btn btn-secondary dposts-ajax-load">加载更多</button>
    </div>
    <?php } ?>
```

在 h5 里元素想自定义属性都可以用`data-*`:

```html
<div
  id="testDiv"
  data-cname="张三"
  data-e-name="zhangsan"
  data-myName="my name is zs."
>
  测试在元素上存储一个key-value
</div>
```

jquery 可以取到这些 data,也可以改变:

```js
var name = "#testDiv".data("cname");
```

再来分析传递到 js 的参数:

```sh
papge:comments表示是评论;
query: 表示当前文章的id
paged: 表示下1页的index;
commentcount: 表示评论有多少页;
commentspage:表示１页多少个评论，这是个系统参数，后台设置
append:表示具体到１条评论的模板class, 也就是comment-list.php定义的那个;
```

`comment1s_per_page`和`default_comments_page`:(newest)对应的设置:
![Screenshot from 2019-03-11 20-43-25-ab8c6d3d-eb60-4c0d-9c12-8cc2a3b443b4](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202019-03-11%2020-43-25-ab8c6d3d-eb60-4c0d-9c12-8cc2a3b443b4.png)

实际最有用的就是 paged.为实现这个功能.先看看涉及的 api:

```php
//根据per_page设置的page大小，返回总页数,默认由系统的comments_per_page决定.
function get_comment_pages_count( $comments = null, $per_page = null, $threaded){}
//获取当前评论所在的pgage
function get_page_of_comment( $comment_ID, $args = array() )
```

## 实现逻辑

```sh
1 评论数目<10; 直接全部加载;
2 评论>10; 先加载１页,记current page index =0;
3 滑动底时,js拿到最后1条评论的class value,类似"comment-id-123"
  ajax把这个值发送，后台通过get_comment_pages_count,得到paged number; 计算下１page的评论数据，返回dom data;
  ajax respond append数据到末尾;

```
