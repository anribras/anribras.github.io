---
layout: post
title:
modified:
categories: Tech
 
tags: [tools]

  
comments: true
---
<!-- TOC -->

- [说明](#说明)
- [vim增加python支持](#vim增加python支持)
- [vimrc](#vimrc)
- [.vim文件夹及bundle](#vim文件夹及bundle)
- [BundleInstall](#bundleinstall)
- [颜色主题](#颜色主题)
- [youcompleteme 安装](#youcompleteme-安装)

<!-- /TOC -->

### 说明

自己的[配置文件。](https://github.com/anribras/gvim)

youcompleteme真乃神器中的神器，把vim的补全提高到了目前无人能及的高度。(后面用了vscode + vim，才觉得补全的超越之辈存在，但是vim仍然是不可取代)

ycm本身的配置文件为.ycm_extra_conf。

网上有很多配置，我这里主要添加了一个`recursive`添加根目录的方式，最原始的并不能递归添加，其余需求可自行添加。

添加的方式:
```
#qt5
'-ISUB','/usr/include/x86_64-linux-gnu/qt5/',
```

要按下面步骤才能顺利使用youcompleme。

###  vim增加python支持

apt-get install vim也行，注意是否有python支持
```
vim --version |grep python
```

没有从[官网](www.vim.org)下,重新编译
```
./configure --with-features=huge --enable-pythoninterp=yes --with-python-config-dir=/usr/lib/python2.7/config

...

make && sudo make install
```

### vimrc

将git里的.vimrc拷贝到home下:

```
cp .vimrc ~
```

### .vim文件夹及bundle

```
mkdir -p ~/.vim/bundle
cd ~/.vim/bundle
git clone http://github.com/gmarik/vundle.git

```

### BundleInstall

打开任意文件，　在vim cmd里输入`BundleInstall`即可，接下来是按vimrc指定的bundle的安装过程，
其中`youcompleteme`略久，耐心等待下。



### 颜色主题

不知为何bundle下载的solorazied.vim没有生效，需要将文件拷贝到安装目录`/usr/local/share/vim/vim80/colors/`去。


### youcompleteme 安装

在下好的vundle里,执行 
```
.vim/bundle/YouCompleteMe/install.py

```
应该是下载clang相关。

成功后，将.extra_ycm_conf.py拷贝到home即可:

```
cp .extra_ycm_conf ~
```

大功告成。



