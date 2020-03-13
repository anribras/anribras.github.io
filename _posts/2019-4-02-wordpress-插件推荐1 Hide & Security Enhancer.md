---
layout: post
title:
modified:
categories: Tech

tags: [wordpress]

comments: true
---

<!-- TOC -->

- [插件 WP Hide & Security Enhancer - Rewrite](#插件-WP-Hide--Security-Enhancer---Rewrite)
- [Hide for Wappalzer](#Hide-for-Wappalzer)

<!-- /TOC -->

## 插件 WP Hide & Security Enhancer - Rewrite

原理其实就是很简单的 rewrite.不过它写到了.htaccess 里，用 bt 在站点设置的`伪静态`一栏，转换一下就好.<https://www.bt.cn/Tools/apache_to_nginx>

转换后为:

```sh
    setenv HTTP_MOD_REWRITE:On;
#ignored: "-" thing used or unknown variable in regex/rew
if (-f "$document_root/wp-content/cache/wph/$http_host$uri"){
	set $rule_1 1$rule_1;
}
if ($rule_1 = "1"){
	rewrite /.* /"/wp-content/cache/wph/$http_host$uri" last;
}
	rewrite ^/v1/template/v1/template/custom-style.css /wp-content/plugins/wp-hide-security-enhancer/router/file-process.php?action=style-clean&file_path=/wp-content/themes/wp-bootstrap-4/style.css&replacement_path=/v1/template/v1/template/custom-style.css last;
	rewrite ^/v1/template/v1/template/custom-style.css /wp-content/themes/wp-bootstrap-4/style.css last;
	rewrite ^/v1/template/(.+) /wp-content/themes/wp-bootstrap-4/$1 last;
	rewrite ^/v1/ext/(.+) /wp-content/plugins/$1 last;
	rewrite ^/v1/inc/(.+) /wp-includes/$1 last;
	rewrite ^/v1/static/(.+) /wp-content/uploads/$1 last;
	rewrite ^/v1/comment.php /wp-comments-post.php last;
	rewrite ^/v1/res/(.+) /wp-content/$1 last;
```

所以要自己写也是没什么问题的

## Hide for Wappalzer

要能不被他发现....还是不简单的
