---
title: Neovim 入门教程 13——一些实用插件
description: 安装这些插件，进一步提升你的 neovim 使用体验
date: 2025-02-18 16:30:00
lastmod: 2025-02-18 16:30:00
image: ./posts/neovim-beginner-guide/cover.jpg
categories:
  - Tutorials
tags:
  - neovim
---

到目前为止，我觉得对于一份最简单的 neovim 所必要的且安装起来比较困难或者是值得单独拎出来说道说道的插件，我们都已经讲完了。本讲，我们最后再把剩下一些安装起来比较简单但是会大幅度提升 neovim 使用体验的插件简单讲一讲。

## 1 查看文件：nvim-tree

到目前为止，我们一直在通过 `:e` 的方式去打开新文件，这种方式需要我们去记忆文件的路径并一点一点手动输入，即便有了自动补全的帮助也非常不方便。不过，我们可以借助 nvim-tree 这个插件来简化这一过程。该插件的作用就是打开一个边栏，显示文件：

```lua
return {
    "nvim-tree/nvim-tree.lua",
    dependencies = { "nvim-tree/nvim-web-devicons" },
    opts = {
        actions = {
            open_file = {
                quit_on_open = true,
            },
        },
    },
    keys = {
        { "<leader>uf", ":NvimTreeToggle<CR>" },
    },
}
```

默认情况下，nvim-tree 为我们配置好了很多快捷键，例如：

- <kbd>a</kbd>：新建文件
- <kbd>d</kbd>：删除文件
- <kbd>y</kbd>：复制文件
- <kbd>p</kbd>：粘贴
- <kbd>x</kbd>：剪切文件
- <kbd>r</kbd>：重命名
- <kbd>Enter</kbd>：打开文件

说起来，nvim-tree 的配置项实在是太多了，我在上面的代码里面也只是打了个样，配置了一个用 nvim-tree 打开文件后自动关闭 nvim-tree 的功能。如果你想要了解 nvim-tree 的全部功能，可以运行 `:h nvim-tree-opts` 查看

说起来，nvim-tree 的配置项实在是太多了，我在上面的代码里面也只是打了个样，配置了一个用 nvim-tree 打开文件后自动关闭 nvim-tree 的功能。如果你想要了解 nvim-tree 的全部功能，可以运行 `:h nvim-tree-opts` 查看。

## 2 Lualine

你会看到很多 neovim 配置，它们在页面最下面有一个状态栏，里面显示了诸如当前 neovim 的 mode、git 所在的 branch 等各种各样信息。其实，这些信息 neovim 自己也会自动显示（比如说 mode）或者可以通过命令查看，但是有这样一个状态栏会更加直观且方便。

![](lualine.png)

这种状态栏有很多插件去实现，我比较推荐 lualine：

```lua
return {
    "nvim-lualine/lualine.nvim",
    dependencies = {
        "nvim-tree/nvim-web-devicons",
    },
    event = "VeryLazy",
    opts = {
        options = {
            theme = "auto",
            component_separators = { left = "", right = "" },
            section_separators = { left = "", right = "" },
        },
        extensions = { "nvim-tree" },
        sections = {
            lualine_b = { "branch", "diff" },
            lualine_x = {
                "filesize",
                "encoding",
                "filetype",
            },
        },
    },
}
```

上面的这些选项是我觉得比较必要的。其中，`options.theme` 设置为 `"auto"` 是为了让 lualine 根据当前的主题配色自动切换自己的配色。至于 `component_separators` 和 `section_separators`，我们有必要说一下 lualine 的构成。lualine 分为 6 个 section：

```
+-------------------------------------------------+
| A | B | C                             X | Y | Z |
+-------------------------------------------------+
```

其中，左侧 3 个用 ABC 指代，右侧 3 个用 XYZ 指代。`section_separators` 设置的就是 section 之间的分隔符。每一个 section 中可以显示多个内容，每一种内容我们称之为一个 component，所以 `component_separators` 规定的就是 component 之间的分隔符。

那么怎么控制 lualine 具体显示什么呢？默认情况下，lualine 具有如下的配置：

```lua
{
    lualine_a = {'mode'}, -- 当前的 mode
    lualine_b = {'branch', 'diff', 'diagnostics'}, -- 所在的 git branch、git diff 信息（多少修改、多少增添等）、诊断信息数量
    lualine_c = {'filename'}, -- 文件名
    lualine_x = {'encoding', 'fileformat', 'filetype'}, -- 文件编码、文件的 <EOL>（可以通过 :h file-formats 查看）、文件类型
    lualine_y = {'progress'}, -- 当前所在行数占总行数的百分比
    lualine_z = {'location'} -- 当前所在的行数和列数
}
```

Lualine 内置的 component 可以在[这里](https://github.com/nvim-lualine/lualine.nvim?tab=readme-ov-file#available-components)查看；此外，lualine 也支持自定义的 component，设置起来也非常简单，这里就不做过多讲解。

另外，我们上面还设置了 `extensions`，这里的 nvim-tree 是 lualine 针对 nvim-tree 做的特别优化，因为 lualine 显示的是当前 buffer 的信息，但打开 nvim-tree 也需要一个新的 buffer 去呈现它的内容，而我们显然不需要 nvim-tree 所在的 buffer 的信息，所以此时启用这个扩展就可以在我们打开 nvim-tree 的时候不会更新 lualine 信息（你可以自己尝试一下不启用这个扩展，会呈现一些很奇怪的东西）。

最后，因为我们有了 lualine 来显示 mode，就不需要 neovim 自己在左下角为我们显示的 mode 了。我们可以通过下面的代码将其禁用（按照我们目前的设计，是在 `lua/core/basic.lua` 中编写）：

```lua
vim.opt.showmode = false
```

## 3 Indent-Blankline

在写一些需要游标卡尺的语言（i.e., python）的时候，我们需要快速看清楚缩进。一些编辑器会显示一些竖线，把当前所在的层级表示出来。在 neovim 中，我们可以通过 indent-blankline 来实现这个功能：

```lua
return {
    "lukas-reineke/indent-blankline.nvim",
    event = "VeryLazy",
    main = "ibl",
    opts = {},
}
```

## 4 Telescope

这个插件我很难完整描述它的功能，简单来说的话它是一个查找器，比如说你想要在当前的工作路径下搜索某段文字，或者从 git commit 历史中搜索某个条目，都可以使用这个插件完成。上一讲我们说过 nvim-treesitter 是大多数 neovim 配置会安装的插件，但数据显示 telescope 的安装量甚至还要更高，足以体现其流行程度。

在安装这个插件前，我们需要安装 [ripgrep](https://github.com/BurntSushi/ripgrep) 和 cmake。插件的安装如下：

```lua
return {
    "nvim-telescope/telescope.nvim",
    dependencies = {
        "nvim-lua/plenary.nvim",
        {
            "nvim-telescope/telescope-fzf-native.nvim",
            build = "cmake -S. -Bbuild -DCMAKE_BUILD_TYPE=Release && "
                .. "cmake --build build --config Release && "
                .. "cmake --install build --prefix build",
        },
    },
    cmd = "Telescope",
    opts = {
        extensions = {
            fzf = {
                fuzzy = true,
                override_generic_sorter = true,
                override_file_sorter = true,
                case_mode = "smart_case",
            },
        },
    },
    config = function(_, opts)
        local telescope = require "telescope"
        telescope.setup(opts)
        telescope.load_extension("fzf")
    end,
}
```

这里，依赖项之一是 telescope-fzf-native，这个工具可以大幅提高 telescope 进行模糊搜索的性能。但是，它在安装之后需要使用 cmake 进行构建，所以我们添加了 `build` 里面的命令。后面，我们也在 `opts` 里对其进行了配置，并在 `config` 中手动加载了这个扩展。

在安装之后，我们就可以使用 telescope 进行各种查找了。完整的命令可以在[官方 repo](https://github.com/nvim-telescope/telescope.nvim?tab=readme-ov-file#pickers) 找到，例如：

- `Telescope live_grep`：查找内容
- `Telescope colorscheme`：查找并切换配色主题
- `Telescope git_commits`：查找 git commit

你可以从这些命令中挑选出你需要用到的，然后按照前面的教程所讲的那样自己绑定快捷键。

此外，Telescope 默认提供了一套[快捷键](https://github.com/nvim-telescope/telescope.nvim?tab=readme-ov-file#default-mappings)，其中最主要的是可以通过 `<C-n>` 和 `<C-p>` 切换下一个 / 上一个条目。

## 5 Grug-Far

目前，我们在编辑的时候还缺少一个功能，那就是全局的替换——telescop 的 live grep 只提供了查找，但是如果要替换的话，我们还需要别的手段实现。Neovim 自己是提供了替换功能的，但是那个功能太难用了，所以这里我们还是借用插件实现，这个插件就是 grug-far。


```lua
return {
    "MagicDuck/grug-far.nvim",
    cmd = "GrugFar",
    opts = {},
}
```

安装后，我们就可以通过 `GrugFar` 开始进行替换了。如果需要切换光标位置，可以按 <kbd>Esc</kbd> 进入 normal mode 然后移动光标。

不过需要说一下，如果只是在当前 buffer 中进行替换，倒也没有必要用这个插件。我们可以用 `:<range>s/<from>/<to>` 进行替换，其中 range 可以是行号，如 `:1,5s/<from>/<to>` 替换第 1 到 5 行的内容，也可以用 `:%s/<from>/<to>` 在全篇进行替换。默认情况下，这个替换只会应用于每一行第一个匹配的条目，如果要把一行中出现的所有符合条件的内容都进行替换，可以在后面加上 `/g`，即 `:<range>s/<from>/<to>/g`。
