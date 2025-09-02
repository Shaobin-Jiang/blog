---
title: 通过禁用 shada 提高 neovim 启动速度
description: 推迟 shada 的读取，换取启动阶段的速度
date: 2024-04-17 14:00:00
lastmod: 2024-04-17 14:00:00
image: cover.jpg
categories:
  - Tools
tags:
  - editor
  - neovim
---

我曾经一直有一个问题：为什么我在重新启动 neovim 之后，此前我在 command line 输入过的命令仍然存在，上一次启动时使用过的 register 的内容仍然得以保留？不难猜到，neovim 一定是将这些内容保存在了某个文件里，在下一次启动后读取这个文件，以实现上述效果。然而，我既不知道这个文件到底是什么，也不知道它到底是何时被如何读取的。

机缘巧合，我在优化 neovim 启动速度的过程中，发现了一件事情：在 lazy profile 中，从 LazyDone 到 UIEnter，中间还是隔了很长一段时间。在 WSL 上，这段时间还是在一个可以接受的范围内的，然而在 Windows 上，这却是一段相当长的时间。鉴于 lazy profile 并没有很详细地给出这段时间内发生了什么的描述，我在启动 neovim 的时候使用了 `--startuptime` 这个参数。然后，我就在记录下来的事件列表里看到了这样的内容：

```
...
051.553  000.404: inits 3
051.555  000.002: reading ShaDa
051.636  000.082: opening buffers
052.016  000.380: BufEnter autocommands
052.021  000.005: editing files in windows
```

其中，`reading ShaDa` 这一行命令引起了我的注意。这段 log 发生的时间位于 neovim 启动的末期，那为什么 neovim 要在这个时候去读取这个叫做 ShaDa 的东西呢？于是，我在 neovim 的文档中查找了一下：

> If you exit Vim and later start it again, you would normally lose a lot of
> information.  The ShaDa file can be used to remember that information, which
> enables you to continue where you left off.  Its name is the abbreviation of
> SHAred DAta because it is used for sharing data between Neovim sessions.
> 
> This is introduced in section |21.3| of the user manual.
> 
> The ShaDa file is used to store:
> - The command line history.
> - The search string history.
> - The input-line history.
> - Contents of non-empty registers.
> - Marks for several files.
> - File marks, pointing to locations in files.
> - Last search/substitute pattern (for 'n' and '&').
> - The buffer list.
> - Global variables.

这段文字解释了我在本文一开始提出的问题：neovim 将每一次启动中的一些信息——比如命令行历史、寄存器——存在一个叫做 shada (shared data) 文件中；而根据 log 显示，neovim 会在启动末期读取这个文件。

然而，需要注意，上述的 log 是我在完成了优化之后截取的；在我进一步针对 shada 进行优化之前，读取 shada 占用了相当多的时间。而实际上，shada 中的大部分信息在启动阶段都用不到，譬如命令行历史，只有在进入命令行之后才会有用。因而，我想做的是，在启动阶段禁用掉 shada，然后在 `CmdlineEnter` 事件触发后再读取 shada。

于是，我添加了如下代码：

```lua
vim.opt.shadafile = "NONE"
vim.api.nvim_create_autocmd("CmdlineEnter", {
    once = true,
    callback = function()
        local shada = vim.fn.stdpath("state") .. "/shada/main.shada"
        vim.o.shadafile = shada
        vim.api.nvim_command("rshada! " .. shada)
    end,
})
```

其中，将 `shadafile` 设置为 `NONE` 可以禁用 shada 的读取；而在 `CmdlineEnter` 触发后，则可以通过重新将 `vim.o.shadafile` 指向 shada 的默认存储位置并使用 `rshada` 命令读取 shada。shada 默认位置的获取见文档（`:h shada-file-name`）：

> - Default name of the |shada| file is:
>       Unix:     "$XDG_STATE_HOME/nvim/shada/main.shada"
>       Windows:  "$XDG_STATE_HOME/nvim-data/shada/main.shada"

而 `$XDG_STATE_HOME` 的获取则可以通过 `vim.fn.stdpath("state")` 进行。

如上，在启动阶段禁用 shada 后，neovim 的启动速度又得到了大幅的提升。
