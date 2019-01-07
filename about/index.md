---
layout: page
type: page
title: About me
---


{% assign count = 0 %}
{% assign articles = 0 %}
{% for post in site.posts %}
    {% assign single_count = post.content | strip_html | strip_newlines |size %}
    {% assign count = count | plus: single_count %}
    {% assign articles = articles | plus: 1 %}
{% endfor %}

今天是{{'now' | date: "%Y年%m月%d日"}},我已完成了{{articles}}篇文章.

Hope you enjoy my blog. :smile:




