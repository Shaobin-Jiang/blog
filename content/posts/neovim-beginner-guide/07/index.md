---
title: Neovim 入门教程 07——Bufferline
description: 在多个文件之间丝滑切换
date: 2025-01-19 14:00:00
lastmod: 2025-01-19 14:00:00
image: ./posts/neovim-beginner-guide/cover.jpg
categories:
  - Tutorials
tags:
  - neovim
---

在前面几讲中，我们在修改配置文件的时候有一个非常不方便的地方，那就是在不同文件之间进行切换。当我们使用 `edit` 命令打开一个新的文件的时候，当前在编辑的文件就会隐藏不见，如果我们想要在这些文件之间来回切换，就麻烦了。幸运的是，我们可以通过一个插件解决这个痛点。不过在这之前，我们需要先了解 neovim 的一个基础知识。

## 1 Buffer / Window / Tab

Neovim 的官方文档中是这样描述这三个概念的：

> A buffer is the in-memory text of a file.
>
> A window is a viewport on a buffer.
>
> A tab page is a collection of windows.

简单来说，当我们打开一个文件，就会创建一个对应的 buffer，此时我们所谓对文件的编辑，其实都是对 buffer 的编辑，原始文件并没有改变，除非我们进行保存，才会将 buffer 内容写入文件。说白了，buffer 就是文件内容在内存当中的表示。当我们使用 `edit` 打开一个新的文件的时候，旧的文件 buffer 并没有消失，而是转为了 `hidden` 状态。那么，我们怎么看到这些在内存中的内容呢？neovim 的 window 就是用来呈现一个 buffer 的。到目前为止，我们还只会同时显示一个 window，不过马上我们就会看到如何在 neovim 中同屏显示多个 window，这些 window 组成的集合就叫做 tab——你可以简单地认为，我们现在看到的整个页面，就是一个 tab。

我们可以使用 `:buffers` 查看当前所有的 buffer，类似：

```
1 %a + "content/posts/neovim-beginner-guide/07/index.md" line 27
9  h   "~/Documents/orgfiles/neovim.org" line 95
```

这里，开头的数字代表着 buffer 的 id，我们可以用 `buffer` 命令加上这个 id 切换到指定的 buffer——例如 `:buffer 9` 就会切换到上面的 neovim.org 文件。后面的 `a` 表示 active，`h` 表示 hidden，`+` 表示 buffer 被修改了但没有写入文件。当然我们暂时没有必要了解这些东西是什么意思，因为我们本讲后半部分就是关于使用插件呈现这些信息。现在讲解这些都只是为了演示以便更好了解 buffer 等概念。

我们还可以将同一个 buffer 在不同 window 中显示。neovim 中，我们可以使用 `vsplit` 命令进行左右分屏，也可以用 `split` 进行上下分屏，还可以用这两个命令接上文件名直接在新的 window 打开文件；切换窗口后，也会将光标移动到新的窗口。在 neovim 中，窗口管理有一系列的快捷键，这里挑一些重要的说说：

- `<C-w>h`：切换到左侧窗口
- `<C-w>l`：切换到右侧窗口
- `<C-w>j`：切换到下方窗口
- `<C-w>k`：切换到上方窗口
- `<C-w>c`：关闭窗口
- `<C-w>o`：关闭其他所有窗口

> 现在讲了 window 的操作之后，我们也可以学习一下如何查阅文档了。在 neovim 中，我们可以使用 `:h` 命令查看帮助文档，例如 `:h :buffers` 就会查看 `buffers` 这个命令的用法。这个功能现在还有一点点难用，但会在我们配置补全功能后变得更加方便。
>
> 帮助文档会在一个新的窗口打开，我们可以使用刚才讲的快捷键对窗口进行操作。
>
> 顺便，帮助文档属于特殊的 buffer，不会被 `:buffers` 命令显示出来，我们可以使用 `:buffers!` 查看。

特别地，neovim 默认是向左侧和上方进行分屏的，但是我觉得这样很奇怪，譬如这样打开帮助文档会在上方打开新窗口，所以我会在我的配置中（在 `core/basic.lua` 中）加上这两行代码：

```lua
vim.opt.splitbelow = true
vim.opt.splitright = true
```

现在我们这一个页面上有多个 window，这其实就是一个 tab。tab 的作用主要是管理窗口布局，例如我可以在第一个 tab 左右分屏查看代码和帮助文档，在第二个 tab 只呈现代码。我们可以使用 `:tabnew` 创建新的 tab，使用 `:tabclose` 关闭 tab，使用 `:tabs` 列出所有 tab，使用 `:tabnext` 和 `:tabprevious` 前后切换 tab。

## 2 更好地管理 buffer——使用 bufferline 插件

以我自己的 workflow 来说，我几乎不会用到 tab，window 的管理有方便的快捷键而且打开的窗口一目了然，唯一不方便的就是管理 buffer，因为没有方便的快捷键也没有办法直观看到所有 buffer 的情况，因而我急需一个插件来助力我管理 buffer。这个插件就是 [bufferline.nvim](https://github.com/akinsho/bufferline.nvim)，它的作用是将当前所有的 buffer 以类似于浏览器标签页的方式呈现在页面最上方，同时提供了一系列操作 buffer 的命令。

经过上一讲的学习，安装这个插件应该不是什么难事。我们新建一个 `plugins/bufferline.lua` 文件：

```lua
return {
    "akinsho/bufferline.nvim",
}
```

不过，重启 neovim 后，我们会发现除了 lazy 执行了安装以外，似乎没有发生什么界面的变化。这是因为，lazy 只会对设置了 `opts` / `config` 的插件执行 `require("<plugin-name>").setup()` 操作。

（<span style="color: red;">注意，不是说只有设置了这两个属性中的一个才会**加载**插件。现在打开 lazy 面板可以看到，bufferline 事实上也被加载了。事实上，bufferline 这一类插件是需要既被加载又 setup 之后会才能发挥作用，但也有一些插件只需要加载就可以发挥作用，比如上一讲讲到的主题插件，仅需要被加载就可以通过 `colorscheme` 命令使用。我们马上就会看到，除非明确指出，插件都会被加载的</span>）

Bufferline 有着很多配置项，按照 README 的指示，我们可以通过 `:h bufferline-configuration` 查看这些配置项。不过事实上，默认的配置已经足够我们使用了。所以，我们只需要将 `opts` 设置为一个空的 table 即可：

```lua
return {
    "akinsho/bufferline.nvim",
    opts = {},
}
```


## 3 绑定快捷键

现在，我们的 bufferline 已经安装完成，我们可以用 `:BufferLineCyclePrev` 和 `:BufferLineCycleNext` 来回切换 buffer 了。但显然，这种方式太麻烦了，切换 buffer 应该是一个很快完成的操作，而不是通过输入一长串命令，所以我们应该把一些快捷键绑定到上面。

我们前面已经学习了如何绑定快捷键，但是 lazy 为我们提供了一个简单一些的方式，那就是添加 `keys` 属性。该属性是一个 table，该 table 又由多个 table 构成，每个 table 对应一个快捷键。最内层的 table 第一个值是 `lhs`，第二个值是 `rhs`（同样可以是按键或者 lua 函数），后面则是键值对，包括 `mode`（默认为 `"n"`）、`ft`（生效的文件类型）、以及其他在 `vim.keymap.set` 中使用的属性。

例如，我们可以这样进一步配置 bufferline：

```lua
return {
    "akinsho/bufferline.nvim",
    opts = {},
    keys = {
        { "<leader>bh", ":BufferLineCyclePrev<CR>", silent = true },
        { "<leader>bl", ":BufferLineCycleNext<CR>", silent = true },
        { "<leader>bd", ":bdelete<CR>", silent = true },
        { "<leader>bo", ":BufferLineCloseOthers<CR>", silent = true },
        { "<leader>bp", ":BufferLinePick<CR>", silent = true },
        { "<leader>bc", ":BufferLinePickClose<CR>", silent = true },
    },
}
```

这里，因为我们都是在 normal mode 下使用这些快捷键，所以没有设置 `mode`；此外，因为无所谓对某些特定文件类型生效，所以也没有设置 `ft`。按照次序，这 6 个快捷键的作用是：

- `<leader>bh`：切换到左侧的 buffer
- `<leader>bl`：切换到右侧的 buffer
- `<leader>bd`：关闭当前 buffer（这里的 `bdelete` 是 neovim 自带的命令）
- `<leader>bo`：关闭其他 buffer
- `<leader>bp`：选中特定 buffer（按下后，buffer 旁边会出现字母，按下对应字母跳转到该 buffer）
- `<leader>bc`：关闭特定 buffer

## 4 按需加载插件——lazy 的高级用法

如果你重启了 neovim，你会发现，这个快捷键似乎有点问题——为什么设置了快捷键之后，bufferline 干脆消失了呢？

这就涉及到插件加载的一种优化技巧：**懒加载** (lazy loading)，即仅在需要的时候加载插件。默认情况下，lazy 会在启动 neovim 的时候就加载所有插件，加载完成后才会进入 neovim 的界面，那么随着我们的插件不断增加，启动时间会越来越长。虽然这可能只是 20 ms 和 200 ms 的区别，但是我们使用 neovim 的一个原因就是它的速度优势。然而，我们并没有必要在启动的一开始就一股脑加载所有插件。例如，可以把不涉及到初始 ui 的插件放在进入 neovim 界面之后加载。再比如，我们后面可能会用到一个在边栏展示文件树的插件，那么我们可以在需要展开边栏的时候再加载这个插件。又比如，一个针对 flutter 开发的插件，我们可以让它仅在 dart 文件中加载。

在 lazy 中实现这个功能，可以通过以下几个属性：

- `event`：在某个事件触发的时候加载插件（后面的教程会对事件展开更详细的讲解）
- `cmd`：在某个命令被执行的时候加载插件
- `ft`：当前 buffer 为特定文件类型的时候加载插件
- `keys`：当触发快捷键时加载插件，如果快捷键不存在则创建快捷键

所以，我们现在可以理解为什么 bufferline 消失了——`keys` 固然可以用来创建快捷键，但它也是在告诉 lazy 只有在按下这些快捷键的时候才去加载插件，所以我们这样写代码，相当于懒加载了 bufferline。在 lazy 中，如果要明确不需要懒加载一个插件，可以加上 `lazy = false`：

```lua
return {
    "akinsho/bufferline.nvim",
    opts = {},
    keys = {
        { "<leader>bh", ":BufferLineCyclePrev<CR>", silent = true },
        { "<leader>bl", ":BufferLineCycleNext<CR>", silent = true },
        { "<leader>bd", ":bdelete<CR>", silent = true },
        { "<leader>bo", ":BufferLineCloseOthers<CR>", silent = true },
        { "<leader>bp", ":BufferLinePick<CR>", silent = true },
        { "<leader>bc", ":BufferLinePickClose<CR>", silent = true },
    },
    lazy = false,
}
```

## 5 声明依赖项

我们在编写代码的时候一个重要的原则就是不要重复造轮子。显然，neovim 的插件编写者也遵循这个原则，我们经常会遇到某个插件需要依赖另外一些插件。lazy 允许我们在安装插件的同时自动安装其依赖项。例如，bufferline 有一个可选的依赖项：[nvim-web-devicons](https://github.com/nvim-tree/nvim-web-devicons)。该插件可以显示一些漂亮的图标，与 bufferline 配合使用就可以在 buffer 名称旁边显示文件类型对应的图标。

声明依赖项写在 `dependencies` 属性中：

```lua
return {
    "akinsho/bufferline.nvim",
    dependencies = {
        "nvim-tree/nvim-web-devicons",
    },
    opts = {},
    keys = {
        { "<leader>bh", ":BufferLineCyclePrev<CR>", silent = true },
        { "<leader>bl", ":BufferLineCycleNext<CR>", silent = true },
        { "<leader>bd", ":bdelete<CR>", silent = true },
        { "<leader>bo", ":BufferLineCloseOthers<CR>", silent = true },
        { "<leader>bp", ":BufferLinePick<CR>", silent = true },
        { "<leader>bc", ":BufferLinePickClose<CR>", silent = true },
    },
    lazy = false,
}
```

重启 neovim，此时我们就能看到更加美观的 bufferline 了。
