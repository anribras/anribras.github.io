---
layout: post
title:
modified:
categories: Tech
 
tags: [python]

  
comments: true
---

[参考](https://farer.org/2018/01/16/pipenv-notes/)

之前已经习惯conda了,但是新的项目工程里并没有用到conda的导出那么多的包，同时考虑conda对其他可能用到的同事也不熟，于是考虑用pipenv,并用install单独安装需要的包

之前的conda是先有环境再有项目,而pipenv我理解是基于项目去构建环境，依赖性各方面应该要隔离的更好，特别是不同项目环境区别很大时,另外更适合生产部署。

来演示下我的用法.
有一个工程pylink,已经是在conda python的pylink(3.6)环境下能够work的;
```
source activate pylink
```
新建pipenv 的项目环境:
```
cd pylink
pipenv --python 3.6
```
看到生成了空的Pipfile,地位类似于requirement.txt不过更具体
```
cat Pipfile
[[source]]
url = "https://pypi.org/simple"
verify_ssl = true
name = "pypi"

[packages]

[dev-packages]

[requires]
python_version = "3.6"

```

这里没有任何依赖文件，如果文件夹里有requirement.txt则会从中导入相应的包.

接下来手动添加安装包
```
pipenv install wxpython
```

报错！原因是默认build的wxpython需要gtk-3支持，本机仅有gtk-2
要继续，要么安装gtk3，要么修改pipenv install 时build的参数为gtk2的support.

再换成用下面的pip通过网址安装gtk2的version:
```
pipenv run pip install -f -U https://extras.wxpython.org/wxPython4/extras/linux/gtk2/ubuntu-16.04/wxPython-4.0.3-cp36-cp36m-linux_x86_64.whl
```

安装倒是好了，运行时缺乏核心库文件！

看来,pipenv也不是万能的

不过安装常用的一些库倒是很顺畅的
```
pipenv install requests
```


