---
layout: post
title:
modified:
categories: Tech

tags: [wordpress]

comments: true
---

<!-- TOC -->

- [FILE in PHP](#FILE-in-PHP)

<!-- /TOC -->

## FILE in PHP

本质上还是 http 的 post multipart/form-data 提交文件,到 PHP 的`$_FILE`变量取文件:

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
