---
title: Neovim 入门教程 12——使用 nvim-treesitter
description: Treesitter 的作用是什么？它和 nvim-treesitter 的关系是什么？
date: 2025-02-17 15:00:00
lastmod: 2025-02-17 15:00:00
image: ./posts/neovim-beginner-guide/cover.jpg
categories:
  - Tutorials
tags:
  - neovim
---

到目前为止，我推荐的插件都是根据我自己的爱好来的。插件管理器有人会使用 packer 或者 rocks 而非 lazy，跳转有人会使用 flash 而非 hop，补全有人会使用 nvim-cmp 而非 blink 甚至是直接使用 coc。归根结底，这都体现了 neovim 高度自定义的优势——你可以随意选择自己喜欢的插件。但是我可以确定地说，本讲要讲的插件，99% 以上的 neovim 用户会在他们的配置里进行安装。

这个插件就是 [nvim-treesitter](https://github.com/nvim-treesitter/nvim-treesitter)。很多教程在讲到这个插件的时候，会说它的作用是给 neovim 带来更好的高亮。某种程度上，这并没有错误，但是这并不是它全部的功能，因为你会看到很多看起来和高亮毫不相关的插件也会要求你安装 nvim-treesitter，比如说上一讲我们讲的 lspsaga，nvim-treesitter 就是它的可选依赖之一。

我此前写过一篇[博客](/blog/posts/nvim-treesitter-query)，其中提到了 nvim-treesitter，不过那篇博客更多是关于如何编写 treesitter 的 query，这里我重新对 treesitter 本身做一个简单的梳理。

## 1 Treesitter 是什么

我们需要区分一下 nvim-treesitter 和 treesitter——前者是方便我们在 neovim 中使用 treesitter 功能的插件，而 treesitter 的真正功能是把代码内容解析为抽象语法树（abstract syntax tree, AST），用树上的节点表示源代码中的一些结构。例如，这样一段 lua 代码：

```lua
local function foo(target)
    print("hello " .. target)
end

foo("world")
```

解析出的 AST 就是这样的：

```scheme
(chunk ; [0, 0] - [5, 0]
  local_declaration: (function_declaration ; [0, 0] - [2, 3]
    name: (identifier) ; [0, 15] - [0, 18]
    parameters: (parameters ; [0, 18] - [0, 26]
      name: (identifier)) ; [0, 19] - [0, 25]
    body: (block ; [1, 4] - [1, 29]
      (function_call ; [1, 4] - [1, 29]
        name: (identifier) ; [1, 4] - [1, 9]
        arguments: (arguments ; [1, 9] - [1, 29]
          (binary_expression ; [1, 10] - [1, 28]
            left: (string ; [1, 10] - [1, 18]
              content: (string_content)) ; [1, 11] - [1, 17]
            right: (identifier)))))) ; [1, 22] - [1, 28]
  (function_call ; [4, 0] - [4, 12]
    name: (identifier) ; [4, 0] - [4, 3]
    arguments: (arguments ; [4, 3] - [4, 12]
      (string ; [4, 4] - [4, 11]
        content: (string_content))))) ; [4, 5] - [4, 10]
```

默认情况下，neovim 是基于正则来对代码进行高亮的，效率低下且并不准确。而 treesitter 将代码解析为 ast 之后就可以让我们更准确地为节点进行高亮，所以会有教程告诉你 nvim-treesitter 是一个高亮插件。但解析为 ast 之后我们可不止可以用它来进行高亮，还可以进行更好的缩进、折叠等功能。

## 2 安装 nvim-treesitter

注意，在开始之前，安装 nvim-treesitter 需要 C 的编译器。

我们创建 `lua/plugins/nvim-treesitter.lua`：

```lua
return {
    "nvim-treesitter/nvim-treesitter",
    main = "nvim-treesitter.configs",
    event = "VeryLazy",
    opts = {
        ensure_installed = { "lua" },
        highlight = { enable = true }
    },
}
```

这里，`main = "nvim-treesitter.configs"` 是因为 nvim-treesitter 用来 setup 的模块名称比较奇怪，lazy 无法自动识别，需要我们手动设置；`ensure_installed` 参数是告诉 nvim-treesitter 自动安装哪些 treesitter parser——treesitter 本身只是一个解析 ast 的工具，具体针对不同语言进行解析还是需要具体的规则，neovim 自己提供了部分规则，但是不一定足够，此时我们就可以通过 nvim-treesitter 进行安装；我们也可以通过 `TSInstall <parser>` 来进行安装。`highlight` 属性则是启用了基于 treesitter 的高亮。

当然，对于配置文件这种简单的代码来说，nvim-treesitter 带来的高亮提升还是不太明显。我们来看另一个例子。

我们在对代码进行注释的时候，会根据当前 buffer 类型判断应该进行何种注释。但是，如果是在内嵌了其他语言代码块的 markdown 文档中，这种方式就不合适了——markdown 的注释和 html 一样是 `<!-- -->`，但如果我们想要对其中的 一段 lua 代码进行注释应该是使用 `-- `。此时如果我们使用 treesitter 对 markdown 进行解析，就可以识别出这是一段 lua 代码，从而进行正确的注释。

> Neovim 中注释的快捷键是 `gcc`，我们可以假设有这样一个 markdown 文件，此时在 lua 代码块中按下 `gcc`：
>
> ````markdown
> Some text
> 
> ```lua
> print("hello world")
> ```
> ````
>
> 就可以正确进行注释切换。
