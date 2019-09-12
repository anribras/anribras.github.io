---
layout: post
title:
modified:
categories: Tech
tags: [lavarel]
comments: true
---

<!-- TOC -->

- [tp 无法上传图片](#tp-无法上传图片)

<!-- /TOC -->

## tp 无法上传图片

admin 下发布资讯，无法上传图片. 图片在 storage/public/app 下

<https://laravel.com/docs/5.8/filesystem>

symbol link 来解决

```sh
cd  public
ln -s ../storage/app/public storage
```

or artisan command:

```sh
php artisan storage:link
```
