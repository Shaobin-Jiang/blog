---
title: 解决Neovim中denols与tsserver冲突的问题
description: 针对不同项目进行细化的 lsp 配置
date: 2023-06-30 12:00:00
lastmod: 2023-06-30 12:00:00
image: cover.jpg
categories:
  - Tools
tags:
  - editor
  - neovim
---

最近在neovim中配置deno的开发环境的时候，遇到了一个问题，那就是在此之前我一直使用的JavaScript和TypeScript的LSP——tsserver，会和deno的LSP（denols）有冲突。

按照官方文档的说法，我们可以对LSP的`root_dir`进行任何配置：

```lua
nvim_lsp.denols.setup {
  on_attach = on_attach,
  root_dir = nvim_lsp.util.root_pattern("deno.json", "deno.jsonc"),
}

nvim_lsp.tsserver.setup {
  on_attach = on_attach,
  root_dir = nvim_lsp.util.root_pattern("package.json"),
  single_file_support = false
}
```

按说，这种情况下neovim的LSP会根据项目中不同的文件判断该启动tsserver还是denols，但是这样同样存在问题，那就是如果我们当前的文件夹中有一个`deno.json`，但是在上级文件夹中又存在`package.json`，就会导致tsserver和denols都被启动。例如，我的`~`文件夹中存在一个`package.json`文件，此时如果我想要在桌面（`~/Desktop`）新建一个deno项目，就会导致同一个js / ts文件的buffer中同时挂载tsserver和denols。

而这还不算完，此前我使用prettier对代码进行整理，而deno则使用自带的`deno fmt`进行格式化，如果同时启动两个server，可能会出现deno项目使用了prettier进行美化的问题。

因此，我最终的解决方案是，设置tsserver / denols在`on_attach`事件中对已经启动的server进行检测，如果在应该使用denols的情况下已经启动了tsserver，则停止tsserver。

tsserver：

```lua
local opts = {
	single_file_support = false,
	on_attach = function(client, bufnr)
        if #vim.lsp.get_active_clients({name = 'denols'}) > 0 then
            client.stop()
        else
            -- common是我的配置文件中自己定义的模块
            -- disableFormat是禁用neovim自带的格式化，使用null-ls
            -- keyAttach是快捷键配置，其中 <leader>fm 是使用null-ls进行格式化
            common.disableFormat(client)
            common.keyAttach(bufnr)
        end
	end,
}
```

denols：

```lua
local opts = {
	root_dir = require("lspconfig").util.root_pattern("deno.json", "deno.jsonc"),
    single_file_support = true,
	on_attach = function(client, bufnr)
        local active_client = vim.lsp.get_active_clients({name = 'tsserver'})
        if #active_client > 0 then
            active_client[1].stop()
        end
		common.disableFormat(client)
		common.keyAttach(bufnr)
        
        -- 不使用null-ls，而是使用 deno fmt
        -- 格式化的命令是 !deno fmt %，! 表示执行命令，%指代当前文件
		vim.keymap.del("n", "<leader>fm", { buffer = bufnr })
		vim.keymap.set(
			"n",
			"<leader>fm",
			"<cmd>w<CR><cmd>!deno fmt %<CR><CR>",
			{ noremap = true, silent = true, buffer = bufnr }
		)
	end,
}
```
