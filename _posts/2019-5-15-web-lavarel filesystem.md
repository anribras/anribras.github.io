---
layout: post
title:
modified:
categories: Tech
tags: [lavarel]
comments: true
---

<!-- TOC -->

- [tp无法上传图片](#tp无法上传图片)

<!-- /TOC -->

## tp无法上传图片

admin下发布资讯，无法上传图片. 图片在storage/public/app下

<https://laravel.com/docs/5.8/filesystem>

symbol link来解决
```sh
cd  public
ln -s ../storage/app/public storage
```
or artisan command:
```sh
php artisan storage:link
```
