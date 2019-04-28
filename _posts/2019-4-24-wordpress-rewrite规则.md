---
layout: post
title:
modified:
categories: Tech
tags: [wordpress]
comments: true
---
<!-- TOC -->

- [需求](#需求)

<!-- /TOC -->

## 需求

增加新的评论详情页，如`www.xxx.com/comments/1048/`,可以打开以该评论为父评论的详情页.

访问wp的`www.xxx.com/hot/1234`时，到底发生了什么?

<https://wordpress.stackexchange.com/questions/5413/need-help-with-add-rewrite-rule>

<https://shibashake.com/wordpress-theme/wordpress-permalink-add>


原网址将转换为`www.xxx.com/index.php?cat=hot&postid=1234`.cat,postid就是query_var.




