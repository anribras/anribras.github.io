---
layout: post
title:
modified:
categories: Tech
 
tags: [wordpress]

comments: true
---

ajax传来的数据.

之前理解在wordpress哪都可以用`$_GET,$_POST`等,实际不是这样的，再template里调用get竟然没参数了.在ajax callback还是有的.

被rewrite了?到template那url被rewrite了,所以url的参数没了.

那应该在ajax_callback里　先保存下.

但是lcc并没有...

