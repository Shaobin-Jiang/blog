---
title: 在 neovim 中自动切换输入法
description: 切换模式的时候自动切换到相应的输入法
date: 2025-07-01 20:00:00
lastmod: 2025-07-01 20:00:00
image: cover.jpg
categories:
  - Tools
tags:
  - Editor
  - Neovim
---

Neovim 的入门一直是一件相对困难的事情，而对于中国用户来说，这一过程又额外多出来了一些困难，那就是中文输入。

对于只是轻度使用 neovim 的用户来说，这个问题似乎比较令人费解，但如果你使用 neovim 进行大量中文输入的话，就会明白我所说的这个困难具体是什么。我们可以设想一下，我们现在处于 insert mode 中正在输入中文，突然我们发现上面两行的内容有些问题，需要把那一行删掉，我们该怎么办呢？我们会依次按下 <kbd>esc</kbd> / <kbd>2</kbd> <kbd>k</kbd> / <kbd>d</kbd> <kbd>d</kbd>，对吧？

但是不要忘记了，我们现在在使用中文输入法，所以在你按下 <kbd>k</kbd> 的时候，输入法的候选框就会弹出来，此时我们就需要把这个候选框关掉。但还没完，等我们需要重新回到 insert mode 的时候，还要重新切换回中文输入法（因为我们在输入中文内容），这一来二去就多了至少两个按键；显然在进行编辑的时候，我们需要频繁切换模式，所以就多按了很多键，而如果是在使用 mac 原生的每次切换都慢的要死的那种输入法，这个过程就更加痛苦了。这也是为什么很多人说中文输入是 neovim 的一大痛点。

然而，这个问题解决起来也并没有那么困难，因为无论是什么系统——不管是 windows、mac 还是 linux——都存在着通过外部程序控制输入法切换的方法，而 neovim 又提供了调用外部命令的方式，所以我们完全可以将二者结合起来来解决上述问题。

## 1 基本思路

无论所使用的外部程序是什么，我们解决问题的基本思路都是一致的——在离开 insert mode 的时候切换到英文，进入 insert mode 的时候切换回去。这就需要我们使用到 autocmd 了：

```lua
local ime_autogroup = vim.api.nvim_create_augroup("ImeAutoGroup", { clear = true })

vim.api.nvim_create_autocmd("InsertLeave", {
    group = ime_autogroup,
    callback = function ()
        switch_to_english()
    end
})

vim.api.nvim_create_autocmd("InsertEnter", {
    group = ime_autogroup,
    callback = function ()
        switch_back()
    end
})
```

## 2 在 windows 上的实现

在理清楚基本的实现思路后，我们接下来就需要针对不同的系统寻找不同的工具来进行输入法的切换了。在 windows 上，我推荐的工具是 [im-select](https://github.com/daipeihust/im-select)，你可以在 [https://github.com/daipeihust/im-select/raw/master/win/out/x86/im-select.exe](https://github.com/daipeihust/im-select/raw/master/win/out/x86/im-select.exe) 下载。

在使用前，我们需要在设置的“语言和区域”中添加**英语**和**简体中文**这两种语言。其中，简体中文模式下我们可以选择是否启用拼音，例如根据你的设置的不同，可以用 <kbd>shift</kbd> 或 <kbd>ctrl</kbd> + <kbd>space</kbd> 切换中文和英文；而在英文模式下，我们无法使用中文输入法（所以我也比较推荐打游戏的时候开启这一语言选项）。默认情况下，我们使用按下 <kbd>win</kbd> + <kbd>enter</kbd> 就可以在这两种模式之间进行切换，且，如果你在简体中文模式下启用了英文输入，然后切换到英文再切换回简体中文，此时仍然会启用英文输入。

而使用 im-select 就可以通过程序控制这一过程。例如，`im-select.exe 1033` 就可以切换到英文，`im-select.exe 2052` 可以切换到简体中文。那么现在，我们就可以在 windows 下这样编写代码：

```lua
local ime_autogroup = vim.api.nvim_create_augroup("ImeAutoGroup", { clear = true })

vim.api.nvim_create_autocmd("InsertLeave", {
    group = ime_autogroup,
    callback = function ()
        vim.cmd(":silent :!<path-to-im-select> 1033")
    end
})

vim.api.nvim_create_autocmd("InsertEnter", {
    group = ime_autogroup,
    callback = function ()
        vim.cmd(":silent :!<path-to-im-select> 2052")
    end
})
```

## 3 在 mac 上的实现

在 mac 下的情况大致相似，不过虽然 im-select 也提供了 mac 版本，但是实测并不好用，所以我选用了另一个工具：[macism](https://github.com/laishulu/macism)（说来惭愧，知道这个工具还是通过 emacs-china），这个工具无论在切换速度还是稳定性上都要好太多了。

（不过整体切换的丝滑程度还是远不如 windows，这不得不骂一下果子的智障设计，为什么切换输入法一定要卡一下）

不过，mac 下面的情况和 windows 还是有一点点别的不同，如前文所说，windows 下面的简体中文模式包含了中文和英文两种输入方式，我们在回到 insert mode 的时候只需要进入简体中文模式就行了，无需关心具体是中文输入还是英文输入。但 mac 下，清晰地分成了中文和英文两种输入，所以我们在离开 insert mode 的时候需要先用一个变量存储之前的语言，在进入 insert mode 的时候再选择之前存储的语言。

所以，在 mac 下我们的代码可以这样写：

```lua
local ime_autogroup = vim.api.nvim_create_augroup("ImeAutoGroup", { clear = true })

vim.api.nvim_create_autocmd("InsertLeave", {
    group = ime_autogroup,
    callback = function ()
        vim.system({ "macism" }, { text = true }, function(out)
            -- 用一个全局变量存储之前的语言
            PREVIOUS_IM_CODE_MAC = string.gsub(out.stdout, "\n", "")
        end)
        vim.cmd ":silent :!macism com.apple.keylayout.ABC"
    end
})

vim.api.nvim_create_autocmd("InsertEnter", {
    group = ime_autogroup,
    callback = function ()
        if PREVIOUS_IM_CODE_MAC then
            vim.cmd(":silent :!macism " .. PREVIOUS_IM_CODE_MAC)
        end
        PREVIOUS_IM_CODE_MAC = nil
    end
})
```

## 4 总结

总体来说，虽然本文仅提供了最简单的解决方案，但是更多更复杂的情况的解决思路也大致相同。

至于 linux 上的实现，因为我手边没有装有 linux 的设备所以不是很方便实验，但是解决思路也是大致相同的，例如你可以在 Arch Wiki 中关于 fcitx5 的那部分中找到如何实现这一功能的答案。

在解决了这个问题之后，neovim 使用起来就变得顺手多了
