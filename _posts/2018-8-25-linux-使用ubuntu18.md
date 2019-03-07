---
layout: post
title:
modified:
categories: Tech
 
tags: [linux]

comments: true
---

 win10很强大，但wsl还是用起来不如纯linux,1个netstat命令不能用，让我觉得还是拥抱双系统...

 u盘安装即可，现在3个引导: win10, ubuntu18, 外界硬盘的ubuntu16(原工作内容全在里面了),需要用`boot-repair`神器修复下，最后还要修改下`/boot/grub/grub.cfg`,去掉多余的选项.

#### 软件配置

* 字体

[字体安装](https://blog.csdn.net/gatieme/article/details/51901396)

[windows字体挪到unbuntu上](https://www.cnblogs.com/Dylansuns/p/7648002.html)

另外需下一个gnome-trweak-tool来调整下系统
```
```


* vim

vim-gnome 默认python3+
```
sudo apt-get install vim-gnonme
```
什么vundle BundleInstalll都是老套路了.

核心自然是`youcompleteme`了,基于python3编译，先用`update-alternatives`管理下python版本:
```sh
sudo update-alternatives --install /usr/bin/python python /usr/bin/python3.6  36 --slave /usr/bin/python3m python3m /usr/bin/python3.6m --slave /usr/bin/pip pip /usr/bin/pip3
```
* shadowsocks

ss大杀器，前面配好了python3，这个才能顺利
```
pip install shadowsocks
```
用systemctl配置为开机Unit服务,添加到`shadowsocks.service`到`/lib/systemd/system`
```
[Unit]
Description=Shadowsocks Client Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/sslocal -qc /etc/shadowsocks/config.json

[Install]
WantedBy=multi-user.target
```
```
sudo systemctrl restart shadowsocks.service
```

就在刚配置这个玩意的时候，bwg被ddos攻击了..我这破vps还有人攻击...最后的解决方案是重装centos

* jekyll

blog必须的配置好
```
sudo apt-get install nodejs
sudo apt-get ruby ruby-dev
# not neccessary
gem sources --add https://ruby.taobao.org/ --remove https://rubygems.org/

sudo gem install jekyll --verbose
sudo gem install bundle
cd axx.github.io
bundle install
bundle exec jekyll server
```

* wechat
还真有Electron写的wechat，666.直接到ubuntu software中心下

* pycharm vscode
都纳入ubuntu software下了，很方便安装和升级,vscode直接官网下deb包更快.
ubuntu18还用到新的snap管理包的方式，中间下载pycharm出了点问题，通过下面的命令才解决:

[参考](https://askubuntu.com/questions/701618/pycharm-by-jetbrains-installation)

```
snap change  // 查看snap 状态
sudo snap abort no.x  //终止某个snap的过程
```
另vscode用`setgins sync`将自己的配置同步下,截屏+七牛相关的记得复制粘贴下.

* capslock提示

Thinkpad的老毛病了,[解决方案](https://blog.csdn.net/zhengkarl/article/details/12324879)

* email
尝试了hiri,很强大但是要收费,还是老实的thunderbird好了.
```
sudo apt-get install thunderbird
```

* cisco vpn any connect
sudo apt-get install network-manager-openconnect-gnome

