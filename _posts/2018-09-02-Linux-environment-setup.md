---
layout:     post
title:      "Linux配置记录"
subtitle:   "Hello World, Hello Blog"
date:       2017-12-31
author:     "TXXT"
catalog:    true
tags:
    - 环境配置
    - Linux
    - Vim
---

## 1.改变SSH配色

在`.zshrc`中加上`export TERM=xterm-256color`

```shell
tput colors	//查看色彩支持
echo $TERM	//查看终端类型
```

## 2.常用软件

```shell
sudo apt install build-essential git vim cmake zsh
```

启用universe软件源：sudo add-apt-repository universe

安装oh-my-zsh:

```shell
chsh -s $(which zsh)

sh -c "$(wget https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh -O -)"
```

## 3.VIM

### 安装universal-ctags

`https://github.com/universal-ctags/ctags.git`

Like most Autotools-based projects, you need to do:

```
$ ./autogen.sh
$ ./configure --prefix=/where/you/want # defaults to /usr/local
$ make
$ sudo make install # may require extra privileges depending on where to install
```

autogen.sh runs autoreconf internally. If you use a (binary oriented) GNU/Linux distribution, autoreconf may be part of the autoconf package. In addition you may have to install automake and/or pkg-config, too.

自动索引ctags: https://github.com/ludovicchabant/vim-gutentags

### 安装插件

https://github.com/VundleVim/Vundle.vim

```shell
Plugin 'scrooloose/nerdtree'
Plugin 'Yggdroot/LeaderF'
Plugin 'vim-airline/vim-airline'
Plugin 'ludovicchabant/vim-gutentags'
```

### 配置VIM

1. ctags索引

```shell
set tags=./.tags;,.tags
```
