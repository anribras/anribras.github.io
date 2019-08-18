---
layout: post
title:
modified:
categories: Tech
tags: [web,mysql,laravel]
comments: true
---

<!-- TOC -->

- [问题](#问题)

<!-- /TOC -->

## 问题

issue morphmany to comments:
```php
    public function comments()
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}
```
但是comment字段里的commentable_type的在表里填充的名称太长太丑陋了，可以通过morphMap来定义，
放在ModelServiceProvider的register里:
```php
    protected function registerMorphMap()
    {
         $this->morphMap([
             'issue-feedbacks' => \xx\IssueFeedback\Models\issue-feedback::class,
         ]);
    }
```