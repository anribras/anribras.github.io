---
layout: post
title:
modified:
categories: Tech
 
tags: [wordpress]

comments: true
---


## 常规ajax评论

ajax js端:　提交局部更新的要求，server端更新dom结构，返回给js，js修改dom.

首先要有这个commentform的class,即ajax提交评论的数据 ，来自设计的表单.

```js
jQuery(document).on("submit", "#commentform", function() {
    // action=ajax_comment对应到wordpress的ajax处理cb
    error: function(request){
        ...}
    success: fuction(data) {...
     //data是服务端处理ajax返回的局部dom数据. 
    if (parent != '0') {
        jQuery('#comment-'+parent).append('<ul class="children mt-3 mt-md-4">' + data + '</ul>');
    } else if ( jQuery('.comment-list').length != '0') {
        if( globals.new_comment_position == 'desc' ){
            jQuery('.comment-list').prepend(data)
        }else{
            jQuery('.comment-list').append(data);
        }
    } else {
        jQuery('.comment-list').append(data);
    }
    //验证码机制.

    }
}
```

服务端处理都干嘛:
```php
add_action('wp_ajax_nopriv_ajax_comment', 'ajax_comment_callback');
add_action('wp_ajax_ajax_comment', 'ajax_comment_callback');
function ajax_comment_callback(){
    //判断文章是否可以被评论
    //是否登录
    //是否重复提交
    //是否回复太快?
    //重点来了 修改评论的结构,　并作为ajax的response返回.
}

```

## 自己的做法


server端,如果仅是dom:
```php
function ff_ajax_comment_callback(){ 
    //执行模板xx，并返回执行结果(h5).
    ?> <?php get_template_part('xxx'); ?> <?php die();
}
```
这个代码有意思:
```sh
在php里混编html;用`?><?php`; 也就是把h5代码生成好了
html里又需要混编php函数,用`<?php ?>`.
`get_template_part('xxx')`: 类似include require
```

因为同时还需要更新评论数，做了１个json内嵌html的混编来作为ajax的response，使用方式又不一样了:
```php
function ff_ajax_comment_callback(){ 
    wp_send_json(
            json_encode(
                array(
                        'comments_number'=>get_comments_number($comment->comment_post_ID),
                        'comment_dom'=> load_template_part('template-parts/comment-list')
                )
            )
);

//获得执行php模板后的结果，个人认为很重要的功能.不知道wordpress为啥没有api?
function load_template_part($template_name, $part_name=null) {
    ob_start();
    get_template_part($template_name, $part_name);
    $var = ob_get_contents();
    ob_end_clean();
    return $var;
}

}

```
js端将接收到的数据也json化，然后取字段就好了:
```js
js_function(data) {
    var data = JSON.parse(data);
    console.log(data['comment_dom']; // This is html source
    console.log(data['comments_number']; //This is json key-value
}
```


