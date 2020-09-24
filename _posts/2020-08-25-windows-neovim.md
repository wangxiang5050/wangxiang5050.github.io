---
layout: post
title:  "install neovim on windows"
date:   2020-08-25 11:19:00 +0800
categories: neovim
tags:
    - neovim
---

## 在windows上安装neovim

### 安装neovim

[下载页面](https://github.com/neovim/neovim/releases)

### 查看配置文件路径

打开neovim。

```
:help config
	The Nvim config file is named "init.vim", located at:
		Unix		~/.config/nvim/init.vim
		Windows		~/AppData/Local/nvim/init.vim
```

### 复制粘贴

在init.vim中增加以下片段。

```
set mouse=a

" Paste with middle mouse click
vmap <LeftRelease> "*ygv

" Paste with <Shift> + <Insert>
imap <S-Insert> <C-R>*

" Paste with <shift> + <Insert> in command mode
cmap <S-Insert> <C-R>*

" Yank to system clipboard
set clipboard+=unnamedplus
```

