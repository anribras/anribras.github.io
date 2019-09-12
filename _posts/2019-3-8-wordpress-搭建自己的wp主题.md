---
layout: post
title:
modified:
categories: Tech

tags: [wordpress]

comments: true
---

<!-- TOC -->

- [function.php](#functionphp)
	- [自定义设置](#自定义设置)
	- [小工具设置](#小工具设置)
	- [加载脚本](#加载脚本)
	- [模板 tag](#模板-tag)
	- [templdate-functions](#templdate-functions)

<!-- /TOC -->

准确的讲不是０开始,而是从１个合适的 clean theme 开始.

找了 2 个,
<http://html5blank.com/>
<https://underscores.me/>

就是\_s 这个开始，够简单的.

## function.php

### 自定义设置

都是调用`add_theme_support`.

<https://developer.wordpress.org/reference/functions/add_theme_support/>

![Screenshot from 2019-03-09 23-23-21-73396899-5915-4b42-88e5-318cba4c8b57](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202019-03-09%2023-23-21-73396899-5915-4b42-88e5-318cba4c8b57.png)

不要小看了这个东西，除了根据需求定义，在 jetpack 里也定义丰富的设置选项，可以根据需要添加进来:

```php
if ( defined( 'JETPACK__VERSION' ) ) {
	require get_template_directory() . '/inc/jetpack.php';
}
```

具体可以看<https://jetpack.com/support/>,太多 support,哎,只要找到合适，拿来爽爽的用就好。

### 小工具设置

控制是否允许小工具的设置,主要是设置侧边栏的

![Screenshot from 2019-03-09 23-29-04-8fa5dadb-3ce0-4e68-893c-2fdeaf2d4f07](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202019-03-09%2023-29-04-8fa5dadb-3ce0-4e68-893c-2fdeaf2d4f07.png)

```php
function mytheme_widgets_init() {
	register_sidebar( array(
		'name'          => esc_html__( 'Sidebar', 'mytheme' ),
		'id'            => 'sidebar-1',
		'description'   => esc_html__( 'Add widgets here.', 'mytheme' ),
		'before_widget' => '<section id="%1$s" class="widget %2$s">',
		'after_widget'  => '</section>',
		'before_title'  => '<h2 class="widget-title">',
		'after_title'   => '</h2>',
	) );
}
add_action( 'widgets_init', 'mytheme_widgets_init' );
```

### 加载脚本

加载所需的 js.css 等等..

### 模板 tag

```php
require get_template_directory() . '/inc/template-tags.php';
```

类似的,用`get_template_part`就好.

就是文章出现在前后的 tag，要调用的函数:

![Screenshot from 2019-03-09 23-36-55-dd2a5e57-d0f9-44ff-b0c9-a44c8ef82360](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202019-03-09%2023-36-55-dd2a5e57-d0f9-44ff-b0c9-a44c8ef82360.png)

时间，作者，分类，再详细点，比如浏览数，点赞数，评论数，等.

### templdate-functions

为啥分的类，不清楚，hook 了`body_class`<https://www.cnblogs.com/xcxc/p/3658903.html>

作用是,根据不同的页面类型为 body 标签生成 class 选择器.不同的 body class,用不同的 body css 定义?

您站点近期的数篇文章
