---
layout: post
title:
modified:
categories: Tech
 
tags: [wordpress]

comments: true
---

<!-- TOC -->

- [_FILE in PHP](#_file-in-php)

<!-- /TOC -->


## _FILE in PHP
本质上还是http的post multipart/form-data提交文件,到PHP的`$_FILE`变量取文件:

```sh
    [attachment] => Array
        (
            [name] => Screenshot from 2019-03-09 23-29-01.png
            [type] => image/png
            [tmp_name] => /tmp/phpdGwZGU
            [error] => 0
            [size] => 11005
        )
```
