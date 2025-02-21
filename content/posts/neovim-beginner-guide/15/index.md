---
title: Neovim 入门教程 15——使用 autocmd
description: 使用事件功能，强化我们的配置
date: 2025-02-21 22:00:00
lastmod: 2025-02-21 22:00:00
image: ./posts/neovim-beginner-guide/cover.jpg
categories:
  - Tutorials
tags:
  - neovim
---

在前面使用 lazy 的时候，我们懒加载的方法之一就是设置 `event` 属性，告诉 lazy 在某个事件触发的时候加载插件。这种事件机制我相信各位读者都应该十分熟悉，比如说前端开发中我们就会经常使用 `addEventListener` 去添加事件监听。在 neovim 中，这种在事件触发的时候执行某个操作的功能叫做 autocmd。

## 1 使用 api 创建 autocmd

关于 neovim 中具体涉及哪些事件，你可以通过 `:h events` 查看。这里我们主要来看一下怎么使用 neovim 的 api 去创建 autocmd。这个 api 就是 `vim.api.nvim_create_autocmd(event, opts)`。其中，`event` 可以是一个字符串或者是 table（多个事件），而 `opts` 则是一个用来设置当前 autocmd 的 table，它的常用可选属性包括：

- `group`: 使用 `vim.api.nvim_create_augroup` 创建的组，这样我们可以批量处理拥有相同 group 的 autocmd
- `pattern`: 事件的模式，取决于具体的事件类型，例如 `BufRead` 事件可以是 `BufRead *.txt`，后面的 `*.txt` 就是模式，即仅对 txt 格式的文件触发 `BufRead`
- `buffer`: 触发事件的 buffer id，如果不设置则为所有 buffer 添加 autocmd；不可以和 `pattern` 一起使用
- `desc`: 对 autocmd 的描述
- `callback`: 事件触发的时候执行的函数，该函数接受一个 table 作为传入参数，具体参数想见 `:h nvim_create_autocmd` 的 event-args 部分
- `once`: 是否只触发一次，默认为 `false`

下面我们来实操一下。比如说，我想要在进入 insert mode 的时候在 cmdline 打印文字，就可以这样做：

```lua
vim.api.nvim_create_autocmd("InsertEnter", {
    callback = function ()
        print("Insert Mode!!!")
    end
})
```

## 2 FileType 事件

那么 autocmd 可以怎么让我们的配置变得更加强大呢？其中一种用法就是针对不同的文件类型做不同的配置。

> 也许会有部分读者知道我们可以直接设置 ftplugins，但是 ftplugin 只能用 vimscript 写，我个人是非常反感大量使用 vimscript 的。

Neovim 在打开文件的时候，会检测文件类型，此时就会触发 `FileType` 事件。例如，我们想要在打开 python 文件的时候讲缩进设置为 2 个字符，将 `colorcolumn` 设置在 120 个字符处，就可以这样做：

```lua
vim.api.nvim_create_autocmd("FileType", {
    pattern = "python", -- 对于 FileType 事件，pattern 就是文件类型
    callback = function()
        vim.bo.tabstop = 2
        vim.bo.shiftwidth = 0
        vim.wo.colorcolumn = "120"
    end,
})
```

值得一提的是，这里我们没有使用 `vim.opt`，而是 `vim.wo` / `vim.bo`，这两个属性和前者类似，但是只针对当前的窗口 / buffer 生效。

> 另外需要说一嘴的是这里我们额外设置了 `shiftwidth`。其实一般来说我们不需要设置这个值，因为此前我们已经在 basic.lua 中将它设置为 0 了。但是如果你不加上这一行，会发现在 python 文件中这个值变成了 4，这是因为 neovim 自己就默认对一些文件类型做了特定的设置。你可以在 python 文件中运行 `:verbose set shiftwidth` 查看是哪里的文件修改了这个值。

## 3 自定义事件

有时候，我们会发现 neovim 自带的事件不够用了，这个时候我们就需要创建自己的事件了。我们在 lazy 中常使用的 `VeryLazy` 就是一个自定义的事件。

所有的自定义事件其实都属于 `User` 事件的不同 pattern，比如我们想要为 `VeryLazy` 事件添加 autocmd，需要这样做：

```lua
vim.api.nvim_create_autocmd("User", {
    pattern = "VeryLazy",
    callback = function()
        print("hello world")
    end
})
```

如果直接将事件设置为 `VeryLazy` 会导致报错，因为 autocmd 只认识 neovim 内置的事件。至于为什么我们在 lazy 中可以不加上前面的 `User`，那是因为 lazy 针对自己创建的这个事件做了特殊的优化。

那么我们怎么自己创建自定义事件呢？很简单，在特定的时间点发出 event + pattern 就行，这个命令就是 `vim.api.nvim_exec_autocmds`。譬如说，我想要确保 mason 加载完之后再加载 none-ls，就可以这样做：

```lua
{
    "williamboman/mason.nvim",
    event = "VeryLazy",
    config = function (_, opts)
        -- 此处省去之前的配置代码
        vim.api.nvim_exec_autocmds("User", { pattern = "MasonLoaded" })
    end,
}

{
    "nvimtools/none-ls.nvim",
    event = "User MasonLoaded",
    -- 此处略缺剩下的配置代码
}
```

---

到此为止，本系列教程就暂时告一段落。编写这段教程期间，我自己也在不断学习，因为我需要去确定一些此前我感到模棱两可的知识点。我所讲的这些内容远远算不上全面，我甚至算不上倾囊相授，因为我自己掌握的很多技巧都没办法有条理地整理出来。但是我相信，我所讲的这些内容，足够为你开始使用 neovim 打下一个相对牢固的基础。

其实，我讲得再多，也永远不可能覆盖所有的使用情境，总有一刻，你需要自己踏出走向未知的那一步。但这正是使用 neovim 的乐趣所在，不是么？只要你坚持使用 neovim，敢于不断更新自己的配置、自己的使用习惯，面对看起来困难的需求敢于去想办法解决而不是向其妥协，你一定会发现 neovim 越来越顺手。

感谢您愿意阅读我的教程。

（完）
