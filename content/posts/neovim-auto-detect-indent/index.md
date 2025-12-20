---
title: 让你的 neovim 自动检测文件缩进
description: 不用外部插件，百行以内代码解决问题
date: 2025-12-20 20:00:00
lastmod: 2025-12-20 20:00:00
image: cover.jpg
categories:
  - Tools
tags:
  - editor
  - neovim
---

如果你使用 neovim 同时开发多个项目的话，可能会遇到一个很糟心的问题，那就是不同项目的缩进不太一样。譬如说，同样是 javascript 项目，我自己偏好 4 空格的缩进，而有的人就更喜欢 2 空格的缩进。为了应对不同的项目代码风格，我们也需要相应调节 neovim 的 `tabstop` 设置，虽然说这个工作量并不大，但每次新接手一个项目都需要手动编码调整一下 neovim 配置非常闹心也不优雅。

那么，我们可不可以通过一种方式来自动检测文件的缩进，来进行相应的调整呢？经过我的实践证明，是可行的。

## 1 用 python 检测代码缩进

首先是缩进检测这一功能本身的编写。为什么我没有选择直接用 neovim 内置的 lua 去进行编写呢？我们或多或少总会遇到一些大文件，这种时候即使用协程也有可能会卡住，更好的方式可能还是像 language server 那样做一个外置的进程，在解析完之后将结果直接返回给 neovim。另外，lua 直接编写可能还真没有 python 那么方便。综上，我选择用 python 来实现这个功能。

这里，我们直接 vibe coding 一下，让 GPT 5 直接帮我们生成：

```python
import os, re, sys
from collections import Counter
from math import gcd
from functools import reduce

INDENT_RE = re.compile(r"^(?P<indent>[ \t]+)")


def _gcd_of_list(nums):
    nums = [n for n in nums if n > 0]
    if not nums:
        return 0
    return reduce(gcd, nums)


def detect_indentation(file: str, skip_prefixes=None) -> str:
    if skip_prefixes is None:
        skip_prefixes = ["#", "//", "--"]  # common comment starts (can be extended)

    with open(file, mode="r") as f:
        lines = f.readlines()

    indent_strings = []
    for ln in lines:
        s = ln.strip()
        if not s:
            continue
        if any(s.startswith(p) for p in skip_prefixes):
            continue
        m = INDENT_RE.match(ln)
        if m:
            indent_strings.append(m.group("indent"))

    if not indent_strings:
        return "unknown"

    tabs = list(filter(lambda x: x.startswith("\t"), indent_strings))
    if len(tabs) / len(indent_strings) > 0.5:
        return "tabs"

    indent = 4
    lengths = sorted({len(s) for s in indent_strings if s})
    g = _gcd_of_list(lengths)
    if g > 0:
        indent = g
    else:
        indent = Counter(len(s) for s in indent_strings).most_common(1)[0][0]

    return str(indent)


if __name__ == "__main__":
    file = sys.argv[1]
    if file and os.path.exists(file):
        print(detect_indentation(file))
```

这样，当我们运行这个脚本 + 待检测的文件名的时候，脚本就会将文件的缩进打印出来。

## 2 在 neovim 中调用脚本、读取脚本结果并进行设置

这一步也是不难。neovim 自带一个 `vim.system` 命令，可以执行系统命令并在命令结束之后执行一个回调，那么我们只要在进入 buffer 的时候执行一个 autocmd 去调用这个功能即可。比如说我们把这个 python 脚本放在 neovim 配置文件夹下并命名为 `detect-indent.py`，那么我们就可以写这样一段代码：

```lua
-- 在 BufEnter 触发的时候，也就是进入一个 buffer 的时候，执行 callback
vim.api.nvim_create_autocmd("BufEnter", {
    callback = function(args)
        local buf = args.buf -- 获取 buffer id

        vim.system({
            "python3",
            "detect-indent.py", -- 为什么这里不写完整路径，因为下面设置了 cwd
            vim.fn.resolve(vim.fn.expand("%:p", true)), -- 获取当前 buffer 文件名
        }, {
            cwd = vim.fn.stdpath "config",
            text = true,
        }, function(out)
            if out.code == 0 then
                local indent = out.stdout -- python 脚本输出结果因为是打印出来的，所以可以通过 stdout 获取
                if indent ~= "unknown" and indent ~= "tabs" and indent ~= "" then
                    -- 使用之前获取的 buffer id 定位到目标 buffer，设置其 tabstop 属性
                    vim.api.nvim_set_option_value("tabstop", indent + 0, { scope = "local", buf = buf })
                end
            end
        end)
    end,
})
```

一切看起来都很棒对吗？但是当你打开一个新文件的时候，会发现报错了：`nvim_set_option_value must not be called in a fast event context`。事实上，你几乎不能在 `vim.system` 的回调中调用任何 `vim.api` 函数或者是 `vim.notify` 这样的函数，根本上来说 neovim 是单线程的，但是 `vim.system` 相当于开了另一条线程，二者是不互通的。

## 3 修改 `vim.system` 函数

那这咋办呢？这不是白忙活了吗？

诶，别急，我们可以另辟蹊径，虽然这个回调不能直接对 neovim 进行设置，但是简单的变量设置还是可以的。那么，我们只要让一个函数不断读取某个全局变量，直到它不为 `nil` 的时候则去执行我们本来想执行的回调，然后在原本的 `vim.system` 的回调中将这个全局变量设置为回调原本接受的 `out` 参数，不就可以了吗？

所以，我们这样修改代码：

```lua
-- 在 BufEnter 触发的时候，也就是进入一个 buffer 的时候，执行 callback
vim.api.nvim_create_autocmd("BufEnter", {
    callback = function(args)
        local buf = args.buf -- 获取 buffer id
        
        local function on_exit()
            -- global_var 是我们要检测的全局变量
            if global_var == nil then
                vim.schedule(on_exit)
                return
            end

            out = global_var
            if out.code == 0 then
                local indent = out.stdout -- python 脚本输出结果因为是打印出来的，所以可以通过 stdout 获取
                if indent ~= "unknown" and indent ~= "tabs" and indent ~= "" then
                    -- 使用之前获取的 buffer id 定位到目标 buffer，设置其 tabstop 属性
                    vim.api.nvim_set_option_value("tabstop", indent + 0, { scope = "local", buf = buf })
                end
            end
        end
        vim.schedule(on_exit)

        vim.system({
            "python3",
            "detect-indent.py", -- 为什么这里不写完整路径，因为下面设置了 cwd
            vim.fn.resolve(vim.fn.expand("%:p", true)), -- 获取当前 buffer 文件名
        }, {
            cwd = vim.fn.stdpath "config",
            text = true,
        }, function(out)
            global_var = out
        end)
    end,
})
```

注意，这里面我们使用的是 `vim.schedule` 而不是一个 while 循环，因为前者是异步的，不会阻碍主线程。

此时我们再去打开一个新的文件，可以发现 neovim 的 `tabstop` 已经自动被修改了。
