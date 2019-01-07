---
layout: post
title:
modified:
categories: Tech
 
tags: [web]

  
comments: true
---

<!-- TOC -->

- [ab](#ab)
- [locust](#locust)
- [两者比较](#两者比较)

<!-- /TOC -->

测试的内容是以`ContentType: multipart/form-data`post方式提交的某文件

## ab

为了提交命名都费了很大劲:

```sh
ab -n 100 -c 100 -p ab-test.mp3 -T "multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW"  http://localhost/vo
```

这还没完 需要在ab-test.mp3原始文件里，加上额外信息，即编辑原文件为:

```
------WebKitFormBoundary7MA4YWxkTrZu0gW 
Content-Disposition: form-data; name="test"; filename="ab-test.mp3" 
Content-Type: audio/mp3

/..../ //orginnal content

------WebKitFormBoundary7MA4YWxkTrZu0gW--

```

## locust

```sh
locust -f xxx.py --no-web -c 100 -r 1000 -t 5 HttpApiUser  
```
locust是python的工具，测试前需要准备1个lucst.py的脚本

## 两者比较

locust不能设置总请求数，但是可以讲最大并发数和测试时间设置成和ab一致，并且-r 要超过-c，保证尽快达到最大并发数

ab命令:
```sh
ab -t 5 -n 20000 -c 30 -p ab-test.mp3 -T "multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW"  http://localhost/vo
```

性能:
```
Concurrency Level:      30                                                                                                                                                                                                                                    
Time taken for tests:   5.001 seconds                                                                                                                                                                                                                         
Complete requests:      3164                                                                                                                                                                                                                                  
Failed requests:        0                                                                                                                                                                                                                                     
Write errors:           0                                                                                                                                                                                                                                     
Total transferred:      626472 bytes                                                                                                                                                                                                                          
Total body sent:        97945275                                                                                                                                                                                                                              
HTML transferred:       155036 bytes                                                                                                                                                                                                                          
Requests per second:    632.61 [#/sec] (mean)                                                                                                                                                                                                                 
Time per request:       47.422 [ms] (mean)                                                                                                                                                                                                                    
Time per request:       1.581 [ms] (mean, across all concurrent requests)                                                                                                                                                                                     
Transfer rate:          122.32 [Kbytes/sec] received
                        19124.26 kb/s sent

```

locust命令:
```sh
locust -f xxx_server/tests/perfermance_test.py --no-web -c 30 -r 100 -t 5 HttpApiUser
```

结果:
```
 Name                                                          # reqs      # fails     Avg     Min     Max  |  Median   req/s
--------------------------------------------------------------------------------------------------------------------------------------------
 POST /voice                                                     1966     0(0.00%)      60      11     162  |      57  370.33
--------------------------------------------------------------------------------------------------------------------------------------------
 Total                                                           1966     0(0.00%)                                     370.33

```

ab比locust快了30%...





