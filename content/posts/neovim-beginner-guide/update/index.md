---
title: Neovim 入门教程 附录——版本更新带来的变化
description: neovim、插件更新带来的配置变化
date: 2025-06-13 13:30:00
lastmod: 2025-06-13 13:30:00
image: ./posts/neovim-beginner-guide/cover.jpg
categories:
  - Tutorials
tags:
  - neovim
---

在前面的教程中我们已经提到过，neovim 和它的插件都是在不断更新，我们难免会遇到一些配置上的变化。虽然我会在文字教程中进行修正，但是我也在 B 站发布了视频教程，视频修改起来比较麻烦，所以这是一篇汇总博文，用来记录视频教程发布后出现的配置变化。

## P9: mason / mason-lspconfig 的 api 和仓库地址变化 (2025.5.8)

- 在lsp那部分中，获取 mason 和 nvim-lspconfig 的映射的 api 发生了变化，现在是 `require("mason-lspconfig").get_mappings().package_to_lspconfig`
- mason 和 mason-lspconfig 的 github 地址发生了变化，不再是 williamboman，而是变成了 mason-org。这并不影响现有代码运行，因为 github 仍然会保留旧的地址，但是最好还是更换一下

## P12: nvim-treesitter 的 master 分支不再更新 (2025.5.26)

nvim-treesitter 进行了更新，默认分支不再是 master 而是 main，由于 main 分支和 master 分支不兼容，为了保持现有配置生效，我们在安装 nvim-treesitter 的时候需要在 table 中加上 `branch = "master"`

需要注意，这并不是一个很好的解决方案，因为 master 分支事实上已经停更了，所以上面的做法会导致我们停留在一个不再更新的版本上。但由于新版本的配置涉及到了 nvim-treesitter 这一讲前面还没有讲到的内容（autocmd，见最后一讲），所以如果你只看到了 P12 可能无法理解那些配置。

如果希望更新 nvim-treesitter 配置：

```lua
{
    -- 其余部分不变
    branch = "main",
    config = function()
        local nvim_treesitter = require "nvim-treesitter"
        nvim_treesitter.setup()

        local ensure_installed = { "lua", "toml" }
        local pattern = {}
        for _, parser in ipairs(ensure_installed) do
            -- neovim 自己的 api，找不到这个 parser 会报错
            local has_parser, _ = pcall(vim.treesitter.language.inspect, parser)

            if not has_parser then
                -- install 是 nvim-treesitter 的新 api，默认情况下无论是否安装 parser 都会执行，所以这里我们做一个判断
                nvim_treesitter.install(parser)
            else
                -- 新版本需要手动启动高亮，但没有安装相应 parser会导致报错
                pattern = vim.tbl_extend("keep", pattern, vim.treesitter.language.get_filetypes(parser))
            end
        end
        vim.api.nvim_create_autocmd("FileType", {
            pattern = pattern,
            callback = function()
                vim.treesitter.start()
            end,
        })
        -- VeryLazy 晚于 FileType，所以需要手动触发一下
        vim.api.nvim_exec_autocmds("FileType", {})
    end,
}
```

## P9: nvim-lspconfig 旧的配置 server 的 api 被废弃 (2025.6.13)

新版本的 nvim-lspconfig 的 api 发生了调整，比如 `require("lspconfig")["lua_ls"].setup(config)` 要改成 `vim.lsp.config("lua_ls", config)`。

## P12: nvim-treesitter 需要新的依赖

你可能需要通过系统的包管理器安装 tree-sitter-cli，否则安装 parser 的时候会卡住不动。
