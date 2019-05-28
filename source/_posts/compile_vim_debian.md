---
title: Debian源码安装Vim
date: 2019-05-28 21:55:14
tags: 
- Debian
- Vim
- +lua
categories: 
- Vim
---

由于想使用`SpaceVim`的一套配置， 但很多Vim的插件需要启用`+lua`或`+python`配置， 服务器上自带的没有这些配置，所以从源码来自行编译Vim启用这些配置。 

<!-- more -->

## 卸载原有的vim

```shell
sudo apt-get remove --purge vim vim-runtime vim-gnome vim-tiny vim-gui-common
sudo rm -rf /usr/local/share/vim /usr/bin/vim
```

## 下载源码包

```shell
git clone https://github.com/vim/vim
```

## 安装依赖

```shell
sudo apt-get install lua5.1 liblua5.1-dev \
                     luajit libluajit-5.1 \
                     python-dev python3-dev ruby-dev \
                     libperl-dev libncurses5-dev \
                     libatk1.0-dev libx11-dev \
                     libxpm-dev libxt-dev
```

## 编译Vim

### 配置

```
./configure --with-features=huge \
            --enable-multibyte \
	    --enable-rubyinterp=yes \
	    --enable-pythoninterp=yes \
	    --with-python-config-dir=/usr/lib/python2.7/config \
	    --enable-python3interp=yes \
	    --with-python3-config-dir=/usr/lib/python3.5/config \
	    --enable-perlinterp=yes \
	    --enable-luainterp=yes \
            --enable-gui=gtk2 \
            --enable-cscope \
	   --prefix=/usr/local
```

> 这里要注意：`--with-python-config-dir` 和 `--with-python3-config-dir` 两项， 要选择正确的config目录

### 编译

```
sudo make install
```

## 设置为默认编辑器

```
sudo update-alternatives --install /usr/bin/editor editor /usr/local/bin/vim 1
sudo update-alternatives --set editor /usr/local/bin/vim
sudo update-alternatives --install /usr/bin/vi vi /usr/local/bin/vim 1
sudo update-alternatives --set vi /usr/local/bin/vim
```

## 确认

```
vim --version
```