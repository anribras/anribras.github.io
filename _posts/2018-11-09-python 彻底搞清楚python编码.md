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
  - [读写 unicode 数据](#读写-unicode-数据)
  - [类型识别](#类型识别)
  - [==比较](#比较)
- [python3](#python3)

<!-- /TOC -->

[python 官方的 How to unicode](https://docs.python.org/2/howto/unicode.html)

[阮一峰的 ascii_unicode_and_utf-8](http://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html)

[why-declare-unicode-by-string-in-python](https://stackoverflow.com/questions/3170211/why-declare-unicode-by-string-in-python)

## 基本概念

字符(串)是让人理解的符号,但存储到计算机，只能用二进制(字节)序列,从字符到字节称为 encode,反之为 decode.

`ascii`:最基本的英文符号 0-9,a-Z，+-\*/%这些的编码，范围 0-0x7F，比如`0`用 0x30 表示.但全球其他语言的符号无法表现了.

`lattin-1`: 类似 ascii,不过范围扩充了 0x80-0xFF,加入了拉丁字符,仍是 1 个字节存储

`unicode`:是符号的规范,全球的语言符号都有独一的 code point(数值),由 2-4 字节表示.Unicode 规定了符号的二进制表示，却没有规定它应该如何存储.，如字符`0`的 code point 值为`U+0030`,那么可以存成`0x00 0x30`，但需要 2 个字节，挺浪费的,实际很少人这么干.

```sh
Unicode contains the alphabets for every single human language, and describes how characters are represented by code points
```

`utf-8`:实现了 unicode 的存储方式之一,节省了空间，又考虑了 ascii 的兼容.对 code point 小于 128 的符号来说,utf-8 和 ascii 的编码一致,仅用 1 个字节,这对英文符号就很友好了.utf-8 对 unicode 编码后的字节从 1 个到 4 个不等, 汉字一般是 3 个,如'中'的 utf-8 编码为`'\xe4\xb8\xad'`

`utf-16`:类似 utf-8，规则有些不同，字符用至少 2 字节表示

`gb2312`:中国汉字特有的编码, 2 个字节表示 1 个汉字，范围 0-65536,英文符号也同 ascii.

## python2

首先 py 文件按系统 orIDE 设置的 encoding 存储为文件,如 linux 一般是 utf-8,windows 为 gbk

之后 py interpretor 读取文件,按正常的文本处理，将读到的 utf-8 转换成 unicode.

为何 interpretor 解码成 unicode? 自然是为了语法分析、显示等的方便，其实文本编辑器也是这么干的

默认 interpretor 使用的是 ascii 来解码字节流.比如遇到字节`\xe7`,那它不是 ascii 字符，理解不了，解释器报错.

```py
# 这一行的中文注释也会报错

# SyntaxError: Non-ASCII character '\xe7' in file xxx.py on line 4, but no encoding declared,see
# http://www.python.org/peps/pep-0263.html for details

```

[pep-0263](https://www.python.org/dev/peps/pep-0263/)

可以通过 py 文件的的开头加上声明:`# -*- coding: utf-8 -*-`

这样 interpretor 便按照 utf-8 去解码字节流变成 unicode 字符，自然不会出错.

python2 定义了几个 type 和字符有关:basestring、str、bytes、unicode.[实际有 7 种，看这里](https://docs.python.org/2/library/stdtypes.html#typesseq)

> basestring 是所有字符表示的抽象类，不管是哪种 string，都属于 basestring
>
> str or bytes type, 表示的都是字符串的 encoding 后的字节形式

当需要存储,传输,或者从其他字节流中读取时,就是使用这种字符串.

可以通过 decode 转变为 unicode string,反过来 unicode string 可以通过 encode 变成 bytes string.

```sh
'中国人'.decode('utf8')
Out[150]: u'\u4e2d\u56fd\u4eba'
u'中国人'.encode('utf-8')
Out[151]: '\xe4\xb8\xad\xe5\x9b\xbd\xe4\xba\xba'

print(u'中国人') #print时，终端总会让字符准确的输出到屏幕
中国人
```

> unicode type 表示字符串的 unicode 形式

unicode(some_encoded_string,[encoding=],[error=])函数可以将 some_encoded_string 转换为 unicode string.

unicode string 在 python 里，是按照 unicode 的方式存储的:

```py
a = u'中' # u'\u4e2d'
```

a 根据系统不同，可能存储为`0x4e,0x2d`,`0x2d,0x4d`,`0x00,0x00,0x4e,0x2d`,`0x4e,0x2d,0x00,0x00`等形式

[how-is-unicode-represented-internally-in-python](https://stackoverflow.com/questions/26079392/how-is-unicode-represented-internally-in-python)

为何定义 unicode 字符串形式?应该是为了人们对自己熟悉的字符集李字符的各种操作的方便，如切换，比较，find 等等..

```sh
u'中'
Out[143]: u'\u4e2d'
u'中' in u'中国'
Out[144]: True
u'中国'.find(u'中')
Out[145]: 0
```

反过来，如果用 byte string 的方式，不但不方便，还容易出问题:

```sh
'中国'.find('国')
Out[147]: 3
Out[147]: 3 #明显不对，其实是查找到了'国'的utf-8的位置
'中国'
Out[148]: '\xe4\xb8\xad\xe5\x9b\xbd'
'中国'.find('\xe5') #不方便
Out[149]: 3

```

因此很多人建议，对所有字符串都+u 的前缀，目的就是为了使用方便.

### 读写 unicode 数据

read/write 数据时，一般都是字节流，也就是 encode 后的数据,但是有时直接操作 unicode string 更方便，如某些数据库.手动去 decode/encode 当然是可以，但是不太方便，可以考虑使用`codecs`模块:

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

```sh
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

```sh
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

```sh
 'abc'==u'abc' ? True
 '中文'==u'中文' ? False
```

这涉及到==的比较机制了，并不建议 unicode type 和 str type 不要做比较，两者是不同的概念.

非要比较的话，可能会有一些自动转换的操作,产生了`'abc'==u'abc' ? True`的结果

['abc'==u'abc'的一些解释](https://stackoverflow.com/questions/16471332/how-can-i-compare-a-unicode-type-to-a-string-in-python)

## python3

python2 编码混乱的根源，就在'abc 中国人'这样的字符串竟然是字节串.

python2 引入 unicode 来专门表示 unicode 字符串，甚至为了让某些操作顺滑，隐蔽的处理了 unicode 字符串到字节串的转换问题.

python3 解决了 2 里面混乱，就一个道理：让字符串归字符串，让字节串归字节串.str 就真正存 str,bytes 就真正存 bytes.

并且坚决划清 2 者界限，不存在还能任意隐式转换一说.

> 1 没有 unicode type 了，只保留 str type, 所有'',"",""""""括起来的字符串都是 unicode 形式，也没有+u 前缀的概念了,也不需要`coding=utf-8` 声明了(默认就这个，有额外的编码，还可以再声明).
>
> 2 字节串用新的二进制 tpye bytes,字符串前加 b 的前缀

str 再不用 decode 了,而是变成了 bytes 的 decode, 配合 str 的 encode, 实现字节串与字符串的互转,
