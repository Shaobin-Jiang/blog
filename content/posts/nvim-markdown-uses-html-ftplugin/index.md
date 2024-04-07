---
title: Neovim 中 markdown 文件中为什么 colorcolumn 不生效？
description: 为什么只在 ftplugin/markdown.vim 中设置是不够的
date: 2023-10-27 11:00:00
lastmod: 2023-10-27 11:00:00
image: cover.jpg
categories:
  - Tools
tags:
  - editor
  - linux
  - neovim
---

我用 neovim 写 markdown 很长时间了，在使用 `im-select` 解决了 Windows 和 WSL 上的中文切换问题之后，整体体验已经非常不错了。然而，不知道从什么时候开始，我用 neovim 打开 markdown 的时候遇到了一个奇怪的问题：我的 `colorcolumn` 始终是 `120`，而我明明设置了其值为 `80`。

按说，这个真的是一个不怎么影响体验的 bug，但它出现的原因真的很奇怪，于是我决定排查一下。

首先，我需要做的是确定在我打开一个 markdown 文档的时候，到底是哪里修正了我的 `colorcolumn`。于是，我在打开 markdown 文档之后，运行了一个 `verbose` 命令：

```
:verbose set colorcolumn
```

然后得到了如下的输出：

```
colorcolumn=120
    Last set from ~/.config/nvim/ftplugin/html.vim line 4
```

哦……这的确是合理了，我确实是为了方便我的 html 开发而将这个值设置的大了一些，但问题是，我开的是一个 markdown 文档，为什么会触发 html 的 ftplugin 呀？

但仔细想一想，这样倒也有道理，毕竟 markdown 中支持 html，所以 markdown 中启用了 html 的 ftplugin 也是合情合理。至于解决方案，倒也简单，只需要在我自己的 html ftplugin 中判定一下文件类型即可：

```vim
if (&filetype == 'html')
    setlocal colorcolumn=120
endif
```
