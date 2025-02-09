---
title: 增强 neovim 0.10 的注释功能
description: 如何使用 neovim 自带的注释实现此前 Comment 插件的功能
date: 2025-02-09 14:30:00
lastmod: 2025-02-09 14:30:00
image: cover.jpg
categories:
  - Tools
tags:
  - editor
  - neovim
---

在很长一段时间内，neovim 的使用者在使用注释功能的时候会依赖一个叫做 [Comment.nvim](https://github.com/numToStr/Comment.nvim) 插件。这个插件功能非常全面，不但提供了注释的切换，还提供了向当前行的上方、下方、末尾添加注释的功能。然而，很不幸的是，这个插件目前似乎停止维护了——它的最后一次 commit 时间是 2024 年 8 月 20 日（也许作者跑去玩黑猴了），issue 也在以一个缓慢的速度逐渐增加。虽然我在它停更之后继续用了小半年，从没遇到任何 bug，但是对于不维护的插件，只要存在替代品，我都倾向于不再使用。

那么这个替代品是什么呢？2024 年 5 月 16 日，neovim 0.10.0 版本正式发布，正式提供了注释功能。我们可以通过 `:h commenting` 查看，大致功能包括：

- 使用 `gcc` 切换注释
- 使用 `gc` + motion 的方式进行多行注释，如 `gc5j`、`gc3{`
- 在 visual mode 下使用 `gc` 进行注释
- 将注释作为一个 textobject，可以进行诸如 `dgc` 这样的操作

如果我没记错的话，前三个功能在 Comment.nvim 插件中都提供了，完全可以无缝切换过来；textobject 是一个非常酷的新功能，毕竟 neovim 编辑用着很爽的原因之一就是有着各种各样的 textobject。

但是很不幸，注释功能似乎就这么多了。我在 Comment.nvim 中非常喜欢的在行尾、上一行、下一行插入注释的功能完全没有提供。而且，如果你仔细看了帮助文档，你会发现这句话：

> Acting on a single line behaves as follows:
> - If the line matches 'commentstring', the comment markers are removed (e.g.
>   `/*foo*/` is transformed to `foo`).
> - Otherwise the comment markers are added to the current line (e.g. `foo` is
>   transformed to `/*foo*/`). **Blank lines are ignored.**

什么？空白的行竟然不能通过 `gcc` 添加注释？您甭管我说这些功能有没有用，我之前用 Comment.nvim 的时候反正经常这么干，所以既然你不提供，那我就自己写好了。

## 1 在行尾 / 上一行 / 下一行添加注释

在 neovim 中有一个 buffer-local 的值：`commentstring`。比如说你在 lua 文件中运行 `= vim.bo.commentstring`，就会得到 `-- %s`。不难看到，这个值规定的是注释的格式，其中的 `%s` 会被替换为注释内容。所以如果我们要实现在行尾 / 上一行 / 下一行添加注释的功能，就可以利用这个值。我们要做的事情包括：

- 对指定行的内容进行替换
- 将光标移动到正确位置
- 进入 insert mode（因为之前 Comment.nvim 有这一步，为了保持体验一致我们也这样做）

### 1.1 在行尾添加注释

我们先来看最简单的功能——至于为什么最简单我们之后会看到。

在 neovim 中，通过 lua 代码控制 buffer 内容是通过 `vim.api.nvim_buf_set_lines()` 实现的：

> nvim_buf_set_lines({buffer}, {start}, {end}, {strict_indexing}, {replacement})
>     Sets (replaces) a line-range in the buffer.
> 
>     Indexing is zero-based, end-exclusive. Negative indices are interpreted as
>     length+1+index: -1 refers to the index past the end. So to change or
>     delete the last element use start=-2 and end=-1.
> 
>     To insert lines at a given index, set `start` and `end` to the same index.
>     To delete a range of lines, set `replacement` to an empty array.
> 
>     Out-of-bounds indices are clamped to the nearest valid value, unless
>     `strict_indexing` is set.
> 
>     Attributes: ~
>         not allowed when |textlock| is active
> 
>     Parameters: ~
>       • {buffer}           Buffer handle, or 0 for current buffer
>       • {start}            First line index
>       • {end}              Last line index, exclusive
>       • {strict_indexing}  Whether out-of-bounds should be an error.
>       • {replacement}      Array of lines to use as replacement

好，那么我们就开始实现功能。首先我们需要获取一些内容：

```lua
local line = vim.api.nvim_get_current_line() -- 当前行的内容
local row = vim.api.nvim_win_get_cursor(0)[1] -- 当前所在的行数，从 1 开始

local commentstring = vim.bo.commentstring -- 获取 commentstring
local comment = commentstring:gsub("%%s", "") -- 将 commentstring 中的 %s 替换掉; %% 是对 % 转义
local index = commentstring:find "%%s" -- 获取光标插入处相对于注释的位置（从 1 开始），这个值相当于 %s 前面的字符数 + 1
```

现在，我们要在行尾添加注释内容。我们看到 `vim.api.nvim_buf_set_lines()` 并不支持在行尾添加内容，所以我们需要手动拼接当前行的内容和注释内容，然后去修改当前整行的内容。

不过在此之前，我们还有多余的一步要做：一般来说，在行尾添加注释会在注释前留一个空格；但是，如果这一行是空行，此时在注释前添加空格则显得多余，所以我们这里对 `comment` 进行修改：

```lua
if line:find "%S" then -- 查找第一个非空字符
    comment = " " .. comment
    index = index + 1 -- 相当于 commentstring 开头被添加了一位
end
```

OK，现在我们可以对当前行的内容进行修改了：

```lua
-- 0 代表当前 buffer
-- 这里的行数是从 0 开始的
-- 将第 [start, end - 1] 行的内容替换掉
vim.api.nvim_buf_set_lines(0, row - 1, row, false, { line .. comment })
```

然后，我们来修改光标的位置：

```lua
-- 0 代表当前 window
-- 这里的第二个参数是一个 table，由 row 和 col 组成，其中 row 从 1 开始，col 从 0 开始
--
-- 当我们的光标放在一行的第一位，col 为 0，所以 col 的值等于光标前面字符的数量
-- 我们现在要把光标放在 %s 再往前一位，例如如果是 // %s，则把光标放在 //█%s 的位置
-- 这样我们后续进入 insert mode 直接按 a 键就可以
-- 至于为什么不是放在 %s 处然后按 i 键，别忘了我们把 %s 替换掉了，最终的字符出在 %s 前面就结束了
-- 所以现在光标前面的字符包括：line 的全部内容 + %s 前面的字符数 - 1 = #line + index - 1 - 1
vim.api.nvim_win_set_cursor(0, { row, #line + index - 2 })
```

最后，调用 `vim.api.nvim_feedkeys()` 按下 <kbd>a</kbd> 键：

```lua
vim.api.nvim_feedkeys("a", "n", false)
```

完整代码：

```lua
local function comment_end()
    local line = vim.api.nvim_get_current_line()
    local row = vim.api.nvim_win_get_cursor(0)[1]

    local commentstring = vim.bo.commentstring
    local comment = commentstring:gsub("%%s", "")
    local index = commentstring:find "%%s"

    if line:find "%S" then
        comment = " " .. comment
        index = index + 1
    end

    vim.api.nvim_buf_set_lines(0, row - 1, row, false, { line .. comment })
    vim.api.nvim_win_set_cursor(0, { row, #line + index - 2 })

    vim.api.nvim_feedkeys("a", "n", false)
end

vim.keymap.set("n", "gcA", comment_end)
```

### 1.2 在上一行 / 下一行添加注释

相比于在行尾添加注释，另起一行添加注释略有一点麻烦，因为我们还要给注释应用缩进。

我们先来考虑在上一行添加注释，此时我们希望让注释的缩进和当前行一致。前面的操作和在行尾添加注释一致：

```lua
local line = vim.api.nvim_get_current_line()
local row = vim.api.nvim_win_get_cursor(0)[1]

local commentstring = vim.bo.commentstring
local comment = commentstring:gsub("%%s", "")
local index = commentstring:find "%%s"
```

接着，我们来获取当前行的缩进：

```lua
-- 查找第一个非空字符，减 1 就是空白字符数
-- 如果没有非空字符，则空白字符数等于当前行的字符数
local blank_chars = (line:find "%S" or #line + 1) - 1

local blank = line:sub(1, blank_chars)
```

然后就是修改内容、放置光标、进入 insert mode：

```lua
-- 复习一下，我们可以理解是将第 [start, end - 1] 行的内容替换掉；如果 start 和 end 相等，则向 start 上面添加一行
-- 也就是在当前行前插入换行符，当前行变成第 row 行（行数从 0 开始），要写入的行变为第 row - 1 行
vim.api.nvim_buf_set_lines(0, row - 1, row - 1, true, { blank .. comment })

vim.api.nvim_win_set_cursor(0, { row, #blank + index - 2 })
vim.api.nvim_feedkeys("a", "n", false)
```

完整代码：

```lua
local function comment_above()
    local line = vim.api.nvim_get_current_line()
    local row = vim.api.nvim_win_get_cursor(0)[1]

    local commentstring = vim.bo.commentstring
    local comment = commentstring:gsub("%%s", "")
    local index = commentstring:find "%%s"

    local blank_chars = (line:find "%S" or #line + 1) - 1
    local blank = line:sub(1, blank_chars)

    vim.api.nvim_buf_set_lines(0, row - 1, row - 1, true, { blank .. comment })
    vim.api.nvim_win_set_cursor(0, { row, #blank + index - 2 })

    vim.api.nvim_feedkeys("a", "n", false)
end

vim.keymap.set("n", "gcO", comment_above)
```

在下一行添加注释的整体逻辑差不多，但是此时我们的缩进应该按照下一行的缩进来。例如，我们在 python 中向 `def` 的下一行添加注释，那么这个注释的缩进应该和函数体内部缩进一致。代码如下：

```lua
local function comment_below()
    local row = vim.api.nvim_win_get_cursor(0)[1]

    -- 如果当前行为最后一行，则仍然取用当前行的缩进
    local total_lines = vim.api.nvim_buf_line_count(0)
    local line
    if row == total_lines then
        line = vim.api.nvim_buf_get_lines(0, row - 1, row, true)[1]
    else
        line = vim.api.nvim_buf_get_lines(0, row, row + 1, true)[1]
    end

    local commentstring = vim.bo.commentstring
    local comment = commentstring:gsub("%%s", "")
    local index = commentstring:find "%%s"

    local blank_chars = (line:find "%S" or #line + 1) - 1
    local blank = line:sub(1, blank_chars)

    vim.api.nvim_buf_set_lines(0, row, row, true, { blank .. comment })
    vim.api.nvim_win_set_cursor(0, { row + 1, #blank + index - 2 })

    vim.api.nvim_feedkeys("a", "n", false)
end

vim.keymap.set("n", "gco", comment_below)
```

## 2 覆盖 `gcc`：针对空白行的注释添加

在经历了前面一系列麻烦的要死的操作之后，针对空白行添加注释似乎不是什么很困难的事情。然而……

前面我们是在凭空创造快捷键，而现在我们是要拓展 `gcc` 快捷键的功能——我们需要在当前行为空白的时候手动添加注释，在其他时候使用 `gcc` 原本的功能。你可能觉得这不是什么很麻烦的事情，比如我们都知道设置快捷键有一个选项叫 `remap`（比如，vim 中的 `nmap` 和 `nnoremap`），我们只要不启用这个选项，就可以使用快捷键原本的功能。例如，我们可以这样：

```lua
vim.keymap.set("n", "j", "jA")
```

此时，<kbd>j</kbd> 本身的功能并没有被改变。但是：

```lua
vim.keymap.set("n", "gcc", "gcc")
```

按说此时这个快捷键应该不会发生任何改变，但是实际试验就会发现，这个快捷键直接失效了。至于为什么不能这样做我们姑且按下不表，我们暂且先假定不能通过按下 `gcc` 来实现注释功能，那直接去调用注释功能相关的 api 函数不就行了吗？ 但是，更加令人抓狂的事情是，整个文档中，关于注释功能本身的内容只有 `:h commenting` 中提供的那么多。你以为它会提供一个 api 供我们调用吗？没有，至少在 0.10.4 中，`vim.api` 提供的 180 个 api 函数中并没有包含任何一个和注释相关的。

这就太烦人了。难道堂堂 neovim 还不允许我们在这上面做自定义了？我们不妨回来思考一下为什么 `gcc` 这个绑定这么特殊。前面 <kbd>j</kbd> 可以进行二次的绑定是因为这个快捷键是内置的快捷键，那 `gcc` 不能二次绑定难道是因为……这也是通过 lua 进行绑定的快捷键？

于是我去到 neovim 的 github 仓库中搜索了 `gcc`，发现了这段代码：

```lua
vim.keymap.set('n', 'gcc', line_rhs, { expr = true, desc = 'Toggle comment line' })
```

果然，这个快捷键也是用 lua 进行绑定的，所以我们再次使用 `vim.keymap.set` 的时候它就被覆盖掉了。而除了定位到原因之外，我们也找到了进行注释的 lua 函数：`require("vim._comment).toggle_line()`。这个函数接受三个参数，分别是注释开始的行（从 0 开始）、结束的行、以及用来判定当前注释状态的字符位置（大概就是根据这个位置的上下文判定是要添加注释还是取消注释）。 所以，我们只需要在当前行为空白的时候手动添加注释，其他时候手动调用这个函数即可：

```lua
local function comment_line()
    local line = vim.api.nvim_get_current_line()

    local row = vim.api.nvim_win_get_cursor(0)[1]
    local commentstring = vim.bo.commentstring
    local comment = commentstring:gsub("%%s", "")
    local index = vim.bo.commentstring:find "%%s"

    if not line:find "%S" then
        vim.api.nvim_buf_set_lines(0, row - 1, row, false, { line .. comment })
        vim.api.nvim_win_set_cursor(0, { row, #line + index - 1 })
    else
        require("vim._comment").toggle_lines(row, row, { row, 0 })
    end
end

vim.keymap.set("n", "gcc", comment_line)
```


当然，我们现在这样做肯定不算是最完美的解法。neovim 没有把这个函数作为 api 暴露出来也许是有自己的考虑，可能在未来的版本中会有所改变。另外，我们用来判定缩进的方法可能也不是最理想的，譬如如果当前行是最后一行但是一个 python 的 `def` 语句，那么下一行的注释就应该自动增加缩进。此外，我们现在是通过 buffer 获取 commentstring，但有些时候我们在一个 buffer 中可能出现另一种语言的代码（例如 markdown 中出现其他语言的代码块），此时 commentstring 就应该通过 treesitter 的功能去获取。但是至少，现在我们把最基础的问题解决好了。
