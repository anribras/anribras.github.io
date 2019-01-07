---
layout: post
title:
modified:
categories: Tech
 
tags: [web]

  
comments: true
---

<!-- TOC -->

- [Cookie](#cookie)
- [Session](#session)
    - [实际操作](#实际操作)

<!-- /TOC -->

参考:

Cookie和Session都是保存用户状态的机制,又可理解为一种数据结构.

Cookie存在客户端(浏览器).

Session存在服务端.


## Cookie

http无状态，简单，减轻服务器压力。

但如用户登录后的访问，如果每个页面都让客户端带登录状态，也很烦。

Cookie机制来了。客户端先第1次请求，服务端的在respone header里,`Set Cookie`字段带1个服务端给这次访问分配的某个标识，客户端保存好，在需要标识身份时，在request header里`Cookie`字段加上这个标识(jsessionid)，服务端就知道客户身份，实现登录。

同时服务器通过`Set-Cookie`字段的属性告诉客户端更多Cookie信息:

`expires=DATE`:Cookie过期时间, 过期服务器就不认了;如果不设置，浏览器仅把session id放到内存，关掉浏览器就失效.

`path=PATH`:Cookie适用的目录

`domain=name`:Cookie适用的域名

`Secure`:https才发送Cookie

`HttpOnly`:Cookie不能被JavaScript脚本访问


![2018-10-11-13-12-02](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-10-11-13-12-02.png)

在浏览器侧，有Cookie结构可存放上面的sessionid,当然还包括其他的用户数据.

但有的浏览器被禁用Cookie，这就需要`url重写`，即在url末端附加sessionid信息.


## Session

Session可以记录用户的喜好与行为数据，即记录下用户目前访问服务器上的那些内容，状态是什么，而考虑到这些数据用户修改的随意性很大，并没有必要直接存储在数据库中。(上redis,memcached)

从session角度讲,客户端的Cookie是Session机制里的一个参与者，它用于保存sessionid





### 实际操作

python falcon做后端，falcon这方面还真不好用..

1. 用户名生成sessionid，保存到redis作为key;
2. setcookie，即将sessionid作为Set-Cookie的值,并设置好超时等参数
3. 浏览器一般已经实现了cookie，下次发送的cookie会带上sessionid
4. 服务端判断sessionid是否过期，如果过期，则要求用户重新登录


查找了就是1个set_cookies的api:

```py
  sessionid = uuid.uuid3(uuid.NAMESPACE_URL, login['user'])
  LOGGER.info(sessionid)
  set_sessionid(str(sessionid) , login['user'])
  resp.set_cookie('jsessionid',str(sessionid),max_age=600)
```
这个就是实现了上面的步骤2.

遇到了问题: post方式提交的usrname+passwd,尽管已经过期，但是重新刷新网页时，仍然会post1次，后台重新存redis,相当于过期机制失效了.

问题还是比较普遍的。 

其解决方式是PRG.

https://www.cnblogs.com/TonyYPZhang/p/5424201.html
https://en.wikipedia.org/wiki/Post/Redirect/Get

server端解决的思路就是，不要在onPost里，着急的返回提交数据后的访问页面,而是通过redirect:xx.com的header (status code = 3xx)，让client重定Get向到新的网页,即POST-REDIRECT-GET.

在falcon里如下处理:
```py

def onPost(...)
  # finally

  # with open('static/portal.html') as f:
  #     resp.body = f.read()
  # pass

  #PRG way
  raise falcon.HTTPMovedPermanently(req.url)
```

2种超时的设计:

1. 在Set-cookie里，告诉client expires,超时后，client(浏览器)将不再发送sessionid;
2. 自己保存的redis sessionid同样设置expired,超时后sessionid的key消失，自然服务器也可以提示超时 

实际应用里，应该根据用户最后的访问时间，刷新rediskey的登录时间，(还剩1s,重新访问了，再刷回10s),这个用client的expires是做不到的

所以应该结合两个方式，用expires设置1个很长的过期时间，用redis设置1个较短的连续访问过期时间.

所以现在再来理解购物网站的购物车如果保持的，就很容易了

session机制记住那个用户，server维护1个 user:购物车清单就可以了

