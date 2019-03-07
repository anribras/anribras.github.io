---
layout: post
title:
modified:
categories: Tech
 
tags: [wordpress]

comments: true
---

更换域名port后，访问wordpress站点出现了问题.

各种重定向到原来的port,自然是报错.

但是wp-admin还是能进的,解决办法:在function.php添加:

```php
remove_filter('template_redirect','redirect_canonical');
```

重启nginx, apache.