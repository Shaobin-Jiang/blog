---
title: Neovim 入门教程 11——LSP (第三部分)
description: 代码格式化及其他零碎的功能
date: 2025-02-16 17:00:00
lastmod: 2025-06-13 13:30:00
image: ./posts/neovim-beginner-guide/cover.jpg
categories:
  - Tutorials
tags:
  - neovim
---

除了进行代码补全以外，我们在日常开发中还需要用到其他的一些功能，例如代码的格式化、跳转、帮助文档等。

## 1 代码格式化

如果我们翻阅 neovim 自己的文档，会发现有这样一个函数：`vim.lsp.buf.format()`。是的，lsp 协议中已经包含了格式化的功能，但是一般来说我们不会选择使用这种方式进行格式化，因为普通的 lsp 的格式化功能太弱了。更好的选择是使用专门的 formatter——在前面使用 mason 的时候，我们可以看到上面有专门的一栏用来安装 formatter。

当然和之前一样，我们只安装 formatter 是不够的，还需要其他插件来将这些工具使用起来。格式化相关的插件有很多很多种，各有各的好处，这里我们选择的插件是 [none-ls](https://github.com/nvimtools/none-ls.nvim)。这个插件的前身叫做 null-ls，但是它的作者在 23 年的时候宣布终止了对这个插件的维护，于是社区里一群一直在使用 null-ls 的开发者们 fork 了原仓库并更名为 none-ls 继续进行维护。

none-ls 的功能其实并不只是格式化，它也提供了包括 hover doc、diagnostics 等功能，但这些功能多少算是附加项，我们之后会选择其他插件来负责，只使用 none-ls 负责格式化。none-ls 的工作原理大概是，为需要的 buffer 启动一个 lsp，然后使用相应的 formatter 来负责这个 lsp 的格式化功能，这样我们仍然只需要通过 `vim.lsp.buf.format()` 函数就可以进行格式化。

下面我们来安装 none-ls。我们创建 `lua/plugins/none-ls.lua`：

```lua
return {
    "nvimtools/none-ls.nvim",
    dependencies = { "nvim-lua/plenary.nvim" },
    event = "VeryLazy",
}
```

这里我们安装了一个依赖项 plenary，这个插件主要是用来处理一些 lua 标准库中不存在或使用起来比较麻烦的功能，比如更方便的异步操作、文件读写等。

接下来，我们来对插件本身进行配置。我们以 lua 的格式化为例，使用 stylua 工具：

```lua
return {
    "nvimtools/none-ls.nvim",
    dependencies = { "nvim-lua/plenary.nvim" },
    event = "VeryLazy",
    config = function()
        local null_ls = require("null-ls")
        null_ls.setup({
            sources = {
                null_ls.builtins.formatting.stylua,
            },
        })
    end,
}
```

这部分代码很简单，为了保持和原始的 null-ls 的兼容性，none-ls 的名称也使用了 null-ls。这里我们简单地对插件进行 setup，并在其中设置了一个 source 为 stylua。不过，我们这里思考一个问题，为什么我不把这段配置用 `opts` 来写呢？

这个问题在我们配置 hop 插件的时候讲到过。因为我们设置的 source 值，也来自 null-ls，如果写在 `opts` 里，那么在 lazy 加载到这个文件的时候，由于只是读取了配置而还没有把插件所在的路径添加到 runtimepath，这个时候对于 neovim 来说这个插件是不存在的，会导致错误。

当然，这段配置显然是不够的，因为我们并没有安装 stylua。就这一点，我们可以像之前安装 lsp 一样，通过代码进行控制：

```lua
function()
    local registry = require("mason-registry")

    local function install(name)
        local success, package = pcall(registry.get_package, name)
        if success and not package:is_installed() then
            package:install()
        end
    end

    install("stylua")

    local null_ls = require("null-ls")
    null_ls.setup({
        sources = {
            null_ls.builtins.formatting.stylua,
        },
    }
end
```

然后，我们为格式化功能配备一个快捷键。我的设计是将所有与 lsp 相关的快捷键以 `<leader>l` 开头，所以我将格式化的快捷键设置为 `<leader>lf`。至于具体如何实现格式化，我们也说过，可以使用 `vim.lsp.buf.format()` 来实现：

```lua
return {
    "nvimtools/none-ls.nvim",
    dependencies = { "nvim-lua/plenary.nvim" },
    event = "VeryLazy",
    config = function()
        local registry = require("mason-registry")

        local function install(name)
            local success, package = pcall(registry.get_package, name)
            if success and not package:is_installed() then
                package:install()
            end
        end

        install("stylua")

        local null_ls = require("null-ls")
        null_ls.setup({
            sources = {
                null_ls.builtins.formatting.stylua,
            },
        })
    end,
    keys = {
        {
            "<leader>lf",
            function()
                vim.lsp.buf.format()
            end,
        }
    },
}
```

但是，这还没有完，我们说过 none-ls 是以一个专门用来格式化的 lsp 的形式存在的，而前面我们还为 lua 配置了 lua-language-server，那么此时在同一个 buffer 中就存在了两个 lsp，都可以进行格式化，这不就冲突了吗？所以，我们现在要回到 mason 的配置中，告诉那里配置的 lsp 不要进行格式化——所有的格式化工作都应该由 none-ls 负责。

我们这样修改 mason 配置中的 `setup` 函数：

```lua
local function setup(name, config)
    local success, package = pcall(registry.get_package, name)
    if success and not package:is_installed() then
        package:install()
    end

    local lsp = require("mason-lspconfig").get_mappings().package_to_lspconfig[name]
    config.capabilities = require("blink.cmp").get_lsp_capabilities(config.capabilities)

    -- 添加在这里
    config.on_attach = function (client)
        client.server_capabilities.documentFormattingProvider = false
        client.server_capabilities.documentRangeFormattingProvider = false
    end
    vim.lsp.config(lsp, config)
end
```

可以看到我们在 setup lsp 的时候，在 config 中添加了一个 `on_attach` 函数。顾名思义，这个函数会在 lsp 被附加到 buffer 上的时候执行。因为这里面我们只 setup 了 lsp，none-ls 的启动并不在这里处理，所以 `on_attach` 不会对 none-ls 生效。函数中，我们针对 `server_capabilities` 进行了处理，这个值和我们前面说的 `capabilities` 差不多，只不过它规定的是语言服务器应该支持什么功能。这里我们直接禁用掉其中和格式化相关的功能就可以——你无需知道具体有哪些选项（当然你也可以直接在 `on_attach` 函数中打印查看一下），这两行新增的内容可以直接复制下来。

这样，我们就把 lua 的格式化配置好了。不过需要说明的是，none-ls 其实主要是调用了外部的格式化工具，所以它本身不对格式化的具体工作做规定。如果想要更细致地控制如何进行格式化，则需要根据格式化工具的不同添加配置文件，比如 stylua 用得就是 [stylua.toml](https://github.com/JohnnyMorganz/StyLua?tab=readme-ov-file#configuring-runtime-syntax-selection)。此外，我们熟悉的 `.prettierrc` 也可以搭配 none-ls 进行使用。

## 2 其他增强 lsp 体验的功能

现在，我们的功能还是不太够。如果我们想要查看帮助文档、进行跳转、对变量重命名，似乎都无法实现。当然，这些功能在 neovim 的 lsp 中都已经实现好了，但说实话，它们并不算好用，至少和我们接下来要讲的插件相比并不好用。

这个插件是 [lspsaga](https://github.com/nvimdev/lspsaga.nvim)，它为我们提供了超级多的功能。你可以参考一下其[文档](https://nvimdev.github.io/lspsaga/)，就可以看到这下面包含了多少特性。

我们来安装并配置这个插件。创建 `lua/plugins/lspsaga.lua` 文件：

```lua
return {
    "nvimdev/lspsaga.nvim",
    cmd = "Lspsaga",
    opts = {
        finder = {
            keys = {
                toggle_or_open = "<CR>",
            },
        },
    },
    keys = {
        { "<leader>lr", ":Lspsaga rename<CR>" },
        { "<leader>lc", ":Lspsaga code_action<CR>" },
        { "<leader>ld", ":Lspsaga definition<CR>" },
        { "<leader>lh", ":Lspsaga hover_doc<CR>" },
        { "<leader>lR", ":Lspsaga finder<CR>" },
        { "<leader>ln", ":Lspsaga diagnostic_jump_next<CR>" },
        { "<leader>lp", ":Lspsaga diagnostic_jump_prev<CR>" },
    }
}
```

这里我们设置了这个插件仅在运行 `Lspsaga` 命令的时候加载，因为该插件安装后会添加这个命令，我们在这个命令后面接上各种参数就可以实现相应的功能；其他时候，我们就用不上这个插件了，所以可以通过这种方式进行懒加载。至于 `opts`，这里的设置更多是我自己的习惯，`finder.keys.toggle_or_open` 是在查看一个变量的 reference 的情况的时候，按哪个键跳转到引用处，默认是 <kbd>o</kbd>，我不太喜欢所以换成了回车。

下面的快捷键设计也是根据我自己的偏好来的，可以看到我们这里使用到了重命名、code action、跳转到定义处、查看文档、查看变量引用、跳转到下一个诊断、跳转到上一个诊断这些功能，而实际上 lspsaga 还提供了更多功能我用不到，你可以查看官方文档去进行挑选。

---

以上就是 lsp 配置的全部内容了。可能会有朋友好奇为什么我不配置调试器。Neovim 是支持断点调试的，你可以在 mason 中看到 dap，GitHub 上也可以找到一个很火的插件 nvim-dap，配置好之后就可以实现断点调试。不过，我个人向来是不喜欢断点调试，所以也实在没有办法讲解这部分内容，请见谅。
