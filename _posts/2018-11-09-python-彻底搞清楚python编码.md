---
layout: post
title:
modified:
categories: Tech
 
tags: [python]

  
comments: true
---

<!-- TOC -->

- [基本概念](#基本概念)
- [python2](#python2)
    - [读写unicode数据](#读写unicode数据)
    - [类型识别](#类型识别)
    - [==比较](#比较)
- [python3](#python3)

<!-- /TOC -->

[python官方的 How to unicode](https://docs.python.org/2/howto/unicode.html)

[阮一峰的ascii_unicode_and_utf-8](http://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html)


[why-declare-unicode-by-string-in-python](https://stackoverflow.com/questions/3170211/why-declare-unicode-by-string-in-python)

## 基本概念

字符(串)是让人理解的符号,但存储到计算机，只能用二进制(字节)序列,从字符到字节称为encode,反之为decode.

`ascii`:最基本的英文符号0-9,a-Z，+-*/%这些的编码，范围0-0x7F，比如`0`用0x30表示.但全球其他语言的符号无法表现了.

`lattin-1`: 类似ascii,不过范围扩充了0x80-0xFF,加入了拉丁字符,仍是1个字节存储

`unicode`:是符号的规范,全球的语言符号都有独一的code point(数值),由2-4字节表示.Unicode规定了符号的二进制表示，却没有规定它应该如何存储.，如字符`0`的code point值为`U+0030`,那么可以存成`0x00 0x30`，但需要2个字节，挺浪费的,实际很少人这么干.
```
Unicode contains the alphabets for every single human language, and describes how characters are represented by code points
```

`utf-8`:实现了unicode的存储方式之一,节省了空间，又考虑了ascii的兼容.对code point小于128的符号来说,utf-8和ascii的编码一致,仅用1个字节,这对英文符号就很友好了.utf-8对unicode编码后的字节从1个到4个不等, 汉字一般是3个,如'中'的utf-8编码为`'\xe4\xb8\xad'`

`utf-16`:类似utf-8，规则有些不同，字符用至少2字节表示

`gb2312`:中国汉字特有的编码, 2个字节表示1个汉字，范围0-65536,英文符号也同ascii.

## python2

首先py文件按系统orIDE设置的encoding存储为文件,如linux一般是utf-8,windows为gbk

之后py interpretor读取文件,按正常的文本处理，将读到的utf-8转换成unicode.

为何interpretor解码成unicode? 自然是为了语法分析、显示等的方便，其实文本编辑器也是这么干的

默认interpretor使用的是ascii来解码字节流.比如遇到字节`\xe7`,那它不是ascii字符，理解不了，解释器报错.

```py
# 这一行的中文注释也会报错

# SyntaxError: Non-ASCII character '\xe7' in file xxx.py on line 4, but no encoding declared,see
# http://www.python.org/peps/pep-0263.html for details

```
[pep-0263](https://www.python.org/dev/peps/pep-0263/)

可以通过py文件的的开头加上声明:`# -*- coding: utf-8 -*-`

这样interpretor便按照utf-8去解码字节流变成unicode字符，自然不会出错.

python2定义了几个type和字符有关:basestring、str、bytes、unicode.[实际有7种，看这里](https://docs.python.org/2/library/stdtypes.html#typesseq)

>basestring是所有字符表示的抽象类，不管是哪种string，都属于basestring

>str or bytes type, 表示的都是字符串的encoding后的字节形式

当需要存储,传输,或者从其他字节流中读取时,就是使用这种字符串.

可以通过decode 转变为unicode string,反过来unicode string可以通过encode变成bytes string.

```
'中国人'.decode('utf8')
Out[150]: u'\u4e2d\u56fd\u4eba'
u'中国人'.encode('utf-8')
Out[151]: '\xe4\xb8\xad\xe5\x9b\xbd\xe4\xba\xba'

print(u'中国人') #print时，终端总会让字符准确的输出到屏幕
中国人
```

>unicode type表示字符串的unicode形式

unicode(some_encoded_string,[encoding=],[error=])函数可以将some_encoded_string转换为unicode string.

unicode string在python里，是按照unicode的方式存储的:
```py
a = u'中' # u'\u4e2d'
```
a根据系统不同，可能存储为`0x4e,0x2d`,`0x2d,0x4d`,`0x00,0x00,0x4e,0x2d`,`0x4e,0x2d,0x00,0x00`等形式

[how-is-unicode-represented-internally-in-python](https://stackoverflow.com/questions/26079392/how-is-unicode-represented-internally-in-python)

为何定义unicode字符串形式?应该是为了人们对自己熟悉的字符集李字符的各种操作的方便，如切换，比较，find等等..
```sh
u'中'
Out[143]: u'\u4e2d'
u'中' in u'中国'
Out[144]: True
u'中国'.find(u'中')
Out[145]: 0
```
反过来，如果用byte string的方式，不但不方便，还容易出问题:
```
'中国'.find('国')
Out[147]: 3
Out[147]: 3 #明显不对，其实是查找到了'国'的utf-8的位置
'中国'
Out[148]: '\xe4\xb8\xad\xe5\x9b\xbd'
'中国'.find('\xe5') #不方便
Out[149]: 3

```
因此很多人建议，对所有字符串都+u的前缀，目的就是为了使用方便.

###  读写unicode数据

read/write数据时，一般都是字节流，也就是encode后的数据,但是有时直接操作unicode string更方便，如某些数据库.手动去decode/encode当然是可以，但是不太方便，可以考虑使用`codecs`模块:

```py
# the way to write unicode
import codecs

with open('11.txt') as f:
    a=f.read()
print('Read bytes from file ,which is encoded by %s' %chardet.detect(a)['encoding'])
unicode_b = a.decode('utf-8')

with open('22.txt','w+') as f:
    f.write(a)
print('Writing utf-8 encoded data into file')

unicode_b = a.decode('utf-8')
f = codecs.open('33.txt','a',encoding='utf-8')
f.write(unicode_b)
f.close()
print('Using codec.open to write unicode string directly')
# the way to read in a unicode
f = codecs.open('33.txt','rb',encoding='utf-8')
unicode_c = f.read()
f.close()
print('Using codec.open read sa unicode string directly %s ' % type(unicode_c))
```

### 类型识别

```
4 types in python2: basestring str bytes unicode 
isinstance('abc',str) ? True
isinstance('abc',bytes) ? True
isinstance('abc',basestring) ? True
isinstance('abc',unicode) ? False
-------
isinstance(u'abc'.encode('utf-8'),str) ? True
isinstance(u'abc'.encode('utf-8'),bytes) ? True
isinstance(u'abc'.encode('utf-8'),basestring) ? True
isinstance(u'abc'.encode('utf-8'),unicode) ? False
-------
isinstance(u'abc',str) ? False
isinstance(u'abc',bytes) ? False
isinstance(u'abc',basestring) ? True
isinstance(u'abc',unicode) ? True
```

### ==比较

```py
# -*- coding: utf-8 -*-
print('python2 set coding=utf-8')
import chardet

print(r"'abc' format: %s" % chardet.detect('abc')['encoding'] )
print(r"b'abc' format: %s" % chardet.detect(b'abc')['encoding'] )

print(r" 'abc'==u'abc' ? {}".format('abc'==u'abc') )
print(r" 'abc'==b'abc' ? {}".format('abc'==b'abc') )
print(r" 'abc'==u'abc'.encode('utf-8') ? {}".format('abc'==u'abc'.encode('utf-8')) )

print(r"'中文' format: %s" % chardet.detect('中文')['encoding'] )
print(r"b'中文' format: %s" % chardet.detect(b'中文')['encoding'] )
print(r" '中文'==u'中文' ? {}".format('中文'==u'中文') )
print(r" '中文'==b'中文' ? {}".format('中文'==b'中文') )
print(r" '中文'==u'中文'.encode('utf-8') ? {}".format('中文'==u'中文'.encode('utf-8')) )
print(r" '中文'==u'中文'.encode('gbk') ? {}".format('中文'==u'中文'.encode('gbk')) )
print(r" u'中文'=='中文'.decode('utf-8') ? {}".format(u'中文'=='中文'.decode('utf-8')) )
```
输出:
```
python2 set coding=utf-8
'abc' format: ascii
b'abc' format: ascii
 'abc'==u'abc' ? True
 'abc'==b'abc' ? True
 'abc'==u'abc'.encode('utf-8') ? True
'中文' format: utf-8
b'中文' format: utf-8
 '中文'==u'中文' ? False
 '中文'==b'中文' ? True
 '中文'==u'中文'.encode('utf-8') ? True
 '中文'==u'中文'.encode('gbk') ? False
 u'中文'=='中文'.decode('utf-8') ? True
```

比较奇怪的地方是:
```
 'abc'==u'abc' ? True
 '中文'==u'中文' ? False
```
这涉及到==的比较机制了，并不建议unicode type和str type不要做比较，两者是不同的概念.

非要比较的话，可能会有一些自动转换的操作,产生了`'abc'==u'abc' ? True`的结果

['abc'==u'abc'的一些解释](https://stackoverflow.com/questions/16471332/how-can-i-compare-a-unicode-type-to-a-string-in-python)

## python3


python2 编码混乱的根源，就在'abc中国人'这样的字符串竟然是字节串.

python2 引入unicode来专门表示unicode字符串，甚至为了让某些操作顺滑，隐蔽的处理了unicode字符串到字节串的转换问题.

python3解决了2里面混乱，就一个道理：让字符串归字符串，让字节串归字节串.str就真正存str,bytes就真正存bytes.

并且坚决划清2者界限，不存在还能任意隐式转换一说.

> 1 没有unicode type了，只保留str type, 所有'',"",""""""括起来的字符串都是unicode形式，也没有+u前缀的概念了,也不需要`coding=utf-8` 声明了(默认就这个，有额外的编码，还可以再声明).

> 2 字节串用新的二进制tpye bytes,字符串前加b的前缀

str再不用decode了,而是变成了bytes的decode, 配合str的encode, 实现字节串与字符串的互转, 

