---
layout: post
title:
modified:
categories: Tech

tags: [tools]

comments: true
---

<!-- TOC -->

- [.gitconfig 配置](#gitconfig-配置)
- [使用](#使用)

<!-- /TOC -->

某个项目的分支是这样的..

![2018-01-08-19-29-25](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-01-08-19-29-25.png)

分支间有的代码是要复用的。要管理这么多分支,经常用`bcompare`比较。用 git 命令可以方便的调出 bcompare 作为`difftool or mergetool`。

### .gitconfig 配置

添加如下配置

```
[diff]
    tool = bc3
[difftool "bc3"]
    cmd = /usr/bin/bcompare \"$LOCAL\" \"$REMOTE\"
[difftool]
    prompt = false
[merge]
    tool = bc3
[mergetool "bc3"]
    cmd = /usr/bin/bcompare \"$LOCAL\" \"$REMOTE\" \"$BASE\" \"$MERGED\"
    trustExitCode = true
```

### 使用

```
git difftool --dir-diff origin/streamer-5.0 origin/streamer-6.0
```
