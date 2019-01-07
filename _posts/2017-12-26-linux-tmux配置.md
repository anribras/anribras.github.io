---
layout: post
title:
modified:
categories: Tech
 
tags: [tools]

  
comments: true
---

<!-- TOC -->

- [.tmux.conf](#tmuxconf)

<!-- /TOC -->

其实自己也是个工具党，vim+tmux 已经用的非常6了。现在把自己的配置记录下。也可以到我的[github-config](https://github.com/anribras/gvim)上下载。

tmux带来的方便性一张图就可以说明:

![2017-12-26-10-55-07](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2017-12-26-10-55-07.png)

项目多，开的窗口多，随时要切换，不同窗口可能开了不同的环境脚本，不用tmux用什么...

.tmux.conf放在`~`目录下就好。

### .tmux.conf

* solarized dark背景
* 自定义底部状态栏

加载了一个右下角的显示时间，cpu内存等的信息栏的脚本`tmux-mem-cpu-load`。
来自[这位大神](https://github.com/thewtex/tmux-mem-cpu-load)。
另外x260灭有capslock按键，用脚本写了监控按键的显示

* 复制到系统剪切板

tmux有个要命的缺陷就是无法复制到系统剪切板，借助`xclip`和快捷键绑定完成。

* tmux插件

下载tpm后，可以方便的管理tmux插件,详细可以见配置里。


```
set -g default-terminal "screen-256color"
#
set-option -g history-limit 8000
#将r 设置为加载配置文件，并显示"reloaded!"信息
bind r source-file ~/.tmux.conf \; display "Reloaded!"

# 设置提示信息的前景及背景色
set -g message-fg colour232
set -g message-bg colour23
set -g message-attr bright
#设置状态栏颜色
set -g status-bg default
#set -g status-fg red
# 关闭状态栏窗口占位的自动命名
setw -g automatic-rename on
set-option -g allow-rename off
setw -g utf8 on
set -g status-utf8 on
# 设置状态栏左部宽度
set -g status-left-length 40
# 设置状态栏显示内容和内容颜色  显示session name
set -g status-left "#[fg=yellow,bold]>>>><#S> "
# 设置状态栏右部宽度
set -g status-right-length 80
# 设置状态栏右边内容，依次为capslock状态 , 内存, cpu , 时间
set -g status-right "#[fg=red,bold]Caps<#(xset q |grep Caps |awk '{print $4}')>#[fg=green,bold] Sys<#(tmux-mem-cpu-load -i 5 -g 0 -m 0 -t 0 -a 0)> #[fg=##4E9AB3,bold]%Y-%m-%d %H:%M:%S"
# 窗口信息左对齐
#set -g status-justify center
# 设置自动刷新的时间间隔
set -g status-interval 1 
# 监视窗口信息，如有内容变动，进行提示
setw -g monitor-activity on
set -g visual-activity on

# 窗口号和窗口分割号都以1开始（默认从0开始）
set -g base-index 1
setw -g pane-base-index 1

#copy-mode 将快捷键设置为vi 模式
setw -g mode-keys vi
#复制到系统剪切板o
bind-key -t vi-copy 'v' begin-selection
bind-key -t vi-copy y copy-pipe 'xclip -selection clipboard >/dev/null'

#preifx +  jkhl jump tmux window
#up
bind-key k select-pane -U
#down 
bind-key j select-pane -D
#left
bind-key h select-pane -L
#right
bind-key l select-pane -R

#重新调整窗格的大小
bind ^k resizep -U 3 # 跟选择窗格的设置相同，只是多加 Ctrl（Ctrl-k）
bind ^j resizep -D 3 # 同上
bind ^h resizep -L 3 # ...
bind ^l resizep -R 3 # ...

# zoom pane <-> window
#http://tmux.svn.sourceforge.net/viewvc/tmux/trunk/examples/tmux-zoom.sh
#bind ^z run "tmux-zoom"
 
#鼠标配置 复制要加shift
set -g mouse off

#tpm tmux 插件管理 
# examples:
# set -g @plugin 'github_username/plugin_name'
# set -g @plugin 'git@github.com/user/plugin'
# set -g @plugin 'git@bitbucket.com/user/plugin'
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-sensible'
set -g @plugin 'tmux-plugins/tmux-resurrect'
set -g @plugin 'tmux-plugins/tmux-continuum'
#solarized
set -g @plugin 'seebi/tmux-colors-solarized'


# Enable automatic restore
set -g @continuum-restore 'on'
set -g @continuum-save-interval '30'
#Set solarized
set -g @colors-solarized '256'
#set -g @colors-solarized 'dark'
#set -g @colors-solarized 'light'
#set -g @colors-solarized 'base16'

# Initialize TMUX plugin manager (keep this line at the very bottom of tmux.conf)
run '~/.tmux/plugins/tpm/tpm'
```

