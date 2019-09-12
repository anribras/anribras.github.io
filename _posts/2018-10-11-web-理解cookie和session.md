---
layout: post
title:
modified:
categories: Tech

tags: [web]

comments: true
---

<!-- TOC -->

- [Cookie](#Cookie)
- [Session](#Session)
  - [实际操作](#实际操作)

<!-- /TOC -->

参考:

Cookie 和 Session 都是保存用户状态的机制,又可理解为一种数据结构.

Cookie 存在客户端(浏览器).

Session 存在服务端.

## Cookie

http 无状态，简单，减轻服务器压力。

但如用户登录后的访问，如果每个页面都让客户端带登录状态，也很烦。

Cookie 机制来了。客户端先第 1 次请求，服务端的在 respone header 里,`Set Cookie`字段带 1 个服务端给这次访问分配的某个标识，客户端保存好，在需要标识身份时，在 request header 里`Cookie`字段加上这个标识(jsessionid)，服务端就知道客户身份，实现登录。

同时服务器通过`Set-Cookie`字段的属性告诉客户端更多 Cookie 信息:

`expires=DATE`:Cookie 过期时间, 过期服务器就不认了;如果不设置，浏览器仅把 session id 放到内存，关掉浏览器就失效.

`path=PATH`:Cookie 适用的目录

`domain=name`:Cookie 适用的域名

`Secure`:https 才发送 Cookie

`HttpOnly`:Cookie 不能被 JavaScript 脚本访问

![2018-10-11-13-12-02](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-10-11-13-12-02.png)

在浏览器侧，有 Cookie 结构可存放上面的 sessionid,当然还包括其他的用户数据.

但有的浏览器被禁用 Cookie，这就需要`url重写`，即在 url 末端附加 sessionid 信息.

## Session

Session 可以记录用户的喜好与行为数据，即记录下用户目前访问服务器上的那些内容，状态是什么，而考虑到这些数据用户修改的随意性很大，并没有必要直接存储在数据库中。(上 redis,memcached)

从 session 角度讲,客户端的 Cookie 是 Session 机制里的一个参与者，它用于保存 sessionid

### 实际操作

python falcon 做后端，falcon 这方面还真不好用..

1. 用户名生成 sessionid，保存到 redis 作为 key;
2. setcookie，即将 sessionid 作为 Set-Cookie 的值,并设置好超时等参数
3. 浏览器一般已经实现了 cookie，下次发送的 cookie 会带上 sessionid
4. 服务端判断 sessionid 是否过期，如果过期，则要求用户重新登录

查找了就是 1 个 set_cookies 的 api:

```py
  sessionid = uuid.uuid3(uuid.NAMESPACE_URL, login['user'])
  LOGGER.info(sessionid)
  set_sessionid(str(sessionid) , login['user'])
  resp.set_cookie('jsessionid',str(sessionid),max_age=600)
```

这个就是实现了上面的步骤 2.

遇到了问题: post 方式提交的 usrname+passwd,尽管已经过期，但是重新刷新网页时，仍然会 post1 次，后台重新存 redis,相当于过期机制失效了.

问题还是比较普遍的。

其解决方式是 PRG.
<https://www.cnblogs.com/TonyYPZhang/p/5424201.html>

<https://en.wikipedia.org/wiki/Post/Redirect/Get>

server 端解决的思路就是，不要在 onPost 里，着急的返回提交数据后的访问页面,而是通过 redirect:xx.com 的 header (status code = 3xx)，让 client 重定 Get 向到新的网页,即 POST-REDIRECT-GET.

在 falcon 里如下处理:

```py

def onPost(...)
  # finally

  # with open('static/portal.html') as f:
  #     resp.body = f.read()
  # pass

  #PRG way
  raise falcon.HTTPMovedPermanently(req.url)
```

2 种超时的设计:

1. 在 Set-cookie 里，告诉 client expires,超时后，client(浏览器)将不再发送 sessionid;
2. 自己保存的 redis sessionid 同样设置 expired,超时后 sessionid 的 key 消失，自然服务器也可以提示超时

实际应用里，应该根据用户最后的访问时间，刷新 rediskey 的登录时间，(还剩 1s,重新访问了，再刷回 10s),这个用 client 的 expires 是做不到的

所以应该结合两个方式，用 expires 设置 1 个很长的过期时间，用 redis 设置 1 个较短的连续访问过期时间.

所以现在再来理解购物网站的购物车如果保持的，就很容易了

session 机制记住那个用户，server 维护 1 个 user:购物车清单就可以了
