---
title: Neovim 入门教程 05——自定义快捷键
description: 在 neovim 中定义自己的 keymap
date: 2025-01-15 20:00:00
lastmod: 2025-01-15 20:00:00
image: ./posts/neovim-beginner-guide/cover.jpg
categories:
  - Tutorials
tags:
  - neovim
---

Neovim 为我们提供了大量的快捷键。然而，再思虑周全的开发者也不可能为我们考虑到所有的使用情境，默认提供的快捷键必然不是足够的，这个时候拓展默认快捷键的功能就非常重要了。

Warning: 本讲内容略有难度，而且体量巨大，请酌情慢慢消化。

## 1 使用的 api

在 neovim 中，我们可以使用 `vim.keymap.set(mode, lhs, rhs, opts)` 来绑定一个快捷键。可以看到，该 api 接受 4 个参数，它们分别是：

- `mode`: 快捷键生效的模式，可以是一个字符串（仅对单一模式生效），也可以是一个 table（类似于 array 和 object 的混合，即，其中的元素可以没有键名也可以是键值对，此时快捷键对多个模式生效）；这些模式都由一个字母构成，例如 `"n"` (normal mode) / `"i"` (insert mode) / `"c"` (command mode)...
- `lhs`: 快捷键的按键，是一个字符串；其中，以 <kbd>Ctrl</kbd> 开头的按键表示为 `<C->`，例如 `<C-a>` 就表示 <kbd>Ctrl</kbd> + <kbd>a</kbd>；以 <kbd>Alt</kbd> 开头的按键表示为 `<A->`，例如 `<A-b>` 就表示 <kbd>Alt</kbd> + <kbd>b</kbd>
- `rhs`: 快捷键绑定的功能，可以是另外一组按键，也可以是一个 lua 函数
- `opts`: 又一个 table，包含了对这个快捷键的一些额外设置，这个我们在本讲后面单开一节进行说明

我们可以配置这样一个快捷键：

```lua
vim.keymap.set("n", "<C-a>b", ":lua print('hello world')<CR>", { silent = true })
```

为了方便起见，我们可以直接在 command mode 下运行这行命令（记得以 `:lua` 开头），反正我们后续也不需要保留这行命令，这样做还可以省去重启 neovim 的麻烦。

这里，第一个参数表示的这个快捷键在 normal mode 下生效。第二个参数表示我们绑定的快捷键是 <kbd>Ctrl</kbd> + <kbd>a</kbd> + <kbd>b</kbd>——注意，这里只要保证 <kbd>Ctrl</kbd> 和 <kbd>a</kbd> 同时按下即可。第三个参数是对应的按键，我们想要在 normal mode 下打印 "hello world"，就需要先按下冒号，然后输入命令，最后敲下回车——`<CR>` 就表示回车，如果你希望你的命令被执行，一定不要忘记 `<CR>`！！！关于第四个参数，我们这里还是不多说，只需要看一下 table 的形式即可——table 都是包裹在一对大括号里面，对于键值对，不是用冒号分隔，而是用等号连接。

现在，在 normal mode 下按下这组快捷键，你就会看到 "hello world" 被打印出来了。

## 2 `rhs`

现在，我们希望拓展一下这个快捷键，让它在 insert mode 下也生效，那么该怎么改呢？我们前面说了，`mode` 参数可以是一个 table，所以改成这样就可以了：

```lua
vim.keymap.set({ "n", "i" }, "<C-a>b", ":lua print('hello world')<CR>", { silent = true })
```

吗？我们说过，这里 `rhs` 是**按键**，那么在 insert mode 下按下这些按键，会执行命令吗？答案是不会，它们只会被作为文本输入到我们的文件中。所以，如果要确保我们后面的内容是在 command mode 下输入的，可以将 `:` 替换为 `<Cmd>`：

```lua
vim.keymap.set({ "n", "i" }, "<C-a>b", "<Cmd>lua print('hello world')<CR>", { silent = true })
```

这里，我们插播又一条 neovim 编辑小技巧——如何快速在一行内定位到冒号呢？我们可以使用 <kbd>f</kbd> 快捷键，其作用是在当前行内向右找到第一个目标字符，例如 `f:` 就会向右找到第一个冒号并将光标移动到冒号上；当然，如果你的光标现在就已经在目标字符右面了，可以使用 <kbd>F</kbd> 快捷键，它会向左查找——这两个快捷键前面可以加上数字，例如 `2f:` 就是找到第二个冒号。如果目标不存在，那么光标**不会移动**。

> 这里也有必要补充一下 `<Cmd>` 的作用。前面我们说需要从 normal mode 进入 command mode，但这并不准确，我们也可以在其他模式中直接进入 command mode（相应地，执行完命令也会回到原来的模式），例如从 insert mode 进入 command mode 的快捷键就是 `<C-o>:`。`<Cmd>` 的作用就是可以让我们直接无视不同模式进入 command mode 的不同快捷键，直接使用一个统一的描述方式。

现在，再在 insert mode 中按下这个快捷键——你会发现它没有被输出到文本中了，但是好像也没有被打印出来。其实，它被打印出来了，只是因为打印的那个区域被用来显示当前的模式了（“-- 插入 --”）所以看不到打印的内容。不信的话，我们可以禁用这个模式显示：`:lua vim.opt.showmode=false`，再在 insert mode 中按下快捷键，现在我们想要的内容就被打印出来了。

不过，上面的功能还有一种解法：我们要绑定的功能实际上就是一段 lua 代码，那么完全可以直接用一个函数代替：

```lua
vim.keymap.set({ "n", "i" }, "<C-a>b", function ()
    print("hello world")
end, { silent = true })
```

在 lua 中，函数用 `function` 关键字声明，结尾处需要加上一个 `end`。当 `rhs` 是一个函数的时候，按下快捷键时该函数就会被执行。

我们再来看一个更实用一些的例子。在 neovim 中是有撤销和还原的操作，分别是 `u` 和 `<C-r>`。在我刚学习 neovim 的时候，我很快就习惯了后者，但是用 <kbd>u</kbd> 来进行撤销我无论如何都无法习惯，我总会不自觉去使用 <kbd>Ctrl</kbd> + <kbd>z</kbd>，而这个快捷键会默认将 neovim 挂起——顺便，如果你遇到了这个问题，可以在终端中用 `fg` 回到 neovim——所以，我们不妨试一下将 `<C-z>` 绑定到撤销上。

<span style="color: red;">（注意，这里覆盖了默认快捷键并不意味着这个原本的快捷键就是没有意义的。一来，这主要是出于演示的目的；二来，由于我自己的需求，我并不是很需要这个挂起的功能——我曾经很长一段时间都在 powershell 中使用 neovim，挂起功能会直接把 neovim 卡死。所以，这还是一个见仁见智的问题，并不意味着我推荐你将默认的快捷键覆盖掉。）</span>

这一功能的实现和刚才大同小异，唯一需要知道的知识点是，撤销命令是 `undo`：

```lua
vim.keymap.set({ "n", "i" }, "<C-z>", "<Cmd>undo<CR>", { silent = true })
```

这里，我们再回顾本讲前面提到的 <kbd>f</kbd> / <kbd>F</kbd> 快捷键。这两个快捷键还可以搭配 <kbd>y</kbd> / <kbd>d</kbd> / <kbd>c</kbd> 进行使用，例如 `df:` 就会向右删除至冒号处（冒号也会被删除）。然而，有些时候我们并不想删除最后那个目标字符，此时还可以使用 <kbd>t</kbd> / <kbd>T</kbd> 快捷键，它们会移动到目标字符的左侧（<kbd>t</kbd>）/ 右侧（<kbd>T</kbd>）一位，所以 `dt:` 就会删除到冒号之前那个字符。

因而，假设我们在书写上面这段代码的时候，想要复制一下前面写过的内容进行简单的更改的话，就可以使用上面这个新的小技巧进行编辑：例如，用 <kbd>f</kbd> 定位到 `<Cmd>` 最后面的 `>`，然后 `ct<` 删除 `<CR>` 之前的内容，再进行更改。

如果你想要保留这个快捷键，让它持久生效，可以把它写进配置文件中。我一般会将快捷键放在 `lua/core/keymap.lua` 文件中（顺便，复习一下上一讲的配置文件结构）。不要忘记在 `init.lua` 中引入这个新文件哦！

## 3 关于 `lhs`——设计你的快捷键

新手使用 neovim 的时候往往会有一个问题，那就是设计快捷键的时候非常不合理。当然，我并不认为我在这方面就是什么专家了，我自己设计的快捷键也只是能保证自己用得舒服，而且我也经常会因为一些不合理之处而不得不修改快捷键配置，这也是为什么我在第一讲就提到学习 neovim 的时候一定不要害怕修改自己的使用习惯。

不过，作为比屏幕前的你多使用了一段时间 neovim 的人，我还是可以给出一些些建议的。

一个建议就是，不要试图以字母开始你的快捷键。你可能会说，啊有些高手人家的快捷键就是以字母开头的，怎么就你事多？那是因为高手们知道自己在干嘛，他们不会一不小心就把默认的快捷键给覆盖掉。事实上，26 个字母在 neovim 中都对应了快捷键，以它们开头设计快捷键会影响这些原本快捷键的使用。

为什么会影响呢？我们不妨看这个例子：

```lua
vim.keymap.set("n", "jk", ":lua print(123)<CR>", {})
```

现在，打开一个文件，按下 `jk`，会打印出 `123`。看起来没错对吧？那么，现在按下 <kbd>j</kbd> 试试。有没有发现卡了将近 1 秒才会向下移动？这是因为 neovim 默认设置了 `timeoutlen` 为 1000，即，如果有一长串按键构成的快捷键，按下一个键后，如果等 1000 ms 还没按下下一个键，则视为快捷键没有完成。这意味着，因为 `jk` 这个快捷键的存在，在按下 <kbd>j</kbd> 后 neovim 需要等待 1000 ms 才能判断你要按的快捷键不是 `jk`，才会执行 `j` 本来的功能，所以才出现了我们现在看到的短暂卡住的现象。这也是我所说的以已经存在的快捷键开头设计别的快捷键会影响本来快捷键使用。

唯一一个被相对较多人用来开始快捷键的字母是 `g`，因为它自己不构成一个单独的快捷键。但是，仍然有很多以 `g` 开头的默认快捷键，你仍然有可能不小心覆盖其中一些，比如你想要将 `git` 作为启动 neogit 插件的快捷键，就一不小心覆盖了默认的 `gi` 快捷键。所以，我的建议是，干脆就不要设计以字母开头的快捷键——除非你知道自己在干嘛。

那么，该如何设计快捷键呢？一种方案是设计以 <kbd>Alt</kbd> 开头的快捷键，因为几乎没有以这个键开头的快捷键；相比之下 <kbd>Ctrl</kbd> 没有那么推荐，因为 neovim 自己也绑了很多以 <kbd>Ctrl</kbd> 开头的快捷键，比如前面的 `<C-z>`。不过，无论是 <kbd>Alt</kbd> 还是 <kbd>Ctrl</kbd>，都比以字母开头设计快捷键好得多。

此外，如果你只是想设计一些仅在 normal mode 下生效的快捷键，也可以以 <kbd>SPC</kbd> / <kbd>,</kbd> / <kbd>\\</kbd> / <kbd>.</kbd> 等开头设计快捷键。为什么说仅针对 normal mode 呢？因为在 insert mode 下这些按键也是正常的输入内容。其实，我们选取快捷键开头的思路之一就是，它最好在当前情景下本身没有任何功能。

经过良好设计的快捷键大多数前缀是统一的，此时我们就管这些共同的前缀为 leader key，它们在绑定快捷键的时候可以用 `<leader>` 表示。例如，我们可以进行如下的设置：

```lua
vim.g.mapleader = " "
vim.keymap.set("n", "<leader>aa", ":lua print(123)<CR>", {})
```

此时，我们就创建了一个 <kbd>SPC</kbd> + <kbd>a</kbd> + <kbd>a</kbd> 的快捷键。

你会发现，添加了这样一个前缀后，我们可以使用的语义清晰的快捷键数量一下子就大大丰富了。不过，还是要建议大家在一开始不要忙着过度设计你的快捷键，完全可以先绑定用着，后面用多了发现不合理之处再改也来得及。

## 4 写给 mac 用户——如何绑定 <kbd>Option</kbd> 键

在 Mac 上并没有 <kbd>Alt</kbd> 键。很多用户会想要将 <kbd>Option</kbd> 键作为替代。那么该怎么实现这个功能呢？

一种策略是，通过终端的配置将 <kbd>Option</kbd> 映射为 <kbd>Alt</kbd> 然后一切照旧，例如在 Kitty 终端中就可以设置 `macos_option_as_alt yes` 来进行映射。

另一种做法就是，直接将 <kbd>Option</kbd> + key 对应的字符作为 `lhs`，例如 <kbd>Option</kbd> + <kbd>a</kbd> 对应的是 `å`，那么你在快捷键中的 `lhs` 就是这个。

## 5 `opts`

最后，我们来说说快捷键的一些属性。可以使用的属性很多很多，这里我们也只挑一些重要的来说说。

### 5.1 `remap`

布尔值，如果为 `false` 则禁用递归映射。默认为 `false`。

设想一下，如果你突发奇想想要交换 <kbd>j</kbd> 和 <kbd>k</kbd> 该怎么办呢？像这样？

```lua
vim.keymap.set("n", "j", "k", { remap = true })
vim.keymap.set("n", "k", "j", { remap = true })
```

加入我们启用了 remap，那么就会导致 <kbd>j</kbd> 被映射到 <kbd>k</kbd> 上，<kbd>k</kbd> 再被映射到 <kbd>j</kbd> 上，无穷递归。所以，我们需要禁用 remap。

实际上，我们绝大多数情况下都没必要启用这个属性。之所以在这里讲一下只是因为一些历史遗留原因，你可能经常会看到这个属性。但实际上，我们根本用不到它。

### 5.2 `silent`

有些时候，我们的快捷键对应的功能本身并不会有什么输出。这个时候，你会发现，如果我们的 `rhs` 是一条命令，那么这条命令会在 command line 被显示出来。经过这段时间的观察，你应该也发现了，command mode 下输入的命令在执行后不会被清空。例如：

```lua
vim.keymap.set("n", ",a", ":lua a=1<CR>", {})
```

此时你会看到，按下快捷键后，`:lua a=1` 被显示出来了。这挺烦人的，所以我们可以添加 `silent = true` 来解决问题：

```lua
vim.keymap.set("n", ",a", ":lua a=1<CR>", { silent = true })
```

### 5.3 `nowait`

这个属性一般是在你设计的快捷键是另一个快捷键的开头的时候使用。例如：

```lua
vim.keymap.set("n", ",ab", ":lua print(123)<CR>", {})
vim.keymap.set("n", ",a", ":lua print(456)<CR>", {})
```

我们前面也讲到了这个，当按下 `,a` 的时候，neovim 会等待一段时间，因为它要判断你是不是想要按 `,ab`。为了不等待，我们可以加上 `nowait = true`。

```lua
vim.keymap.set("n", ",ab", ":lua print(123)<CR>", {})
vim.keymap.set("n", ",a", ":lua print(456)<CR>", { nowait = true })
```

不过，这样做存在一个问题，那就是更长的那个快捷键会失效。所以，我们一般只在想要更短的快捷键暂时生效、且确定当前情境下不需要更长的那个快捷键的情况下使用这个属性。

### 5.4 `desc`

这个属性类似于一个注释，是对快捷键功能的描述，例如：

```lua
vim.keymap.set("n", ",ab", ":lua print(123)<CR>", { desc = "Print 123 in the cmdline" })
```

那么它有什么用呢？几乎没有，因为你也可以直接写注释。据我所知，这个属性唯一的作用，是有一些插件会基于你绑定的快捷键动态给出提示（比如 which-key），此时你设置的 `desc` 属性就会被作为描述显示在插件的页面中。

![](which-key.jpg)

<center class="caption">which-key 插件会在我们按键的同时对可能匹配的快捷键进行提示，旁边的文本描述就是我们设置的 `desc`</center>

以上就是关于快捷键配置的一些基本内容。本讲的内容量很大，大家可以好好消化一下。此外，本讲也提到了一些新的操作：<kbd>f</kbd> / <kbd>F</kbd> / <kbd>t</kbd> / <kbd>T</kbd>，在今后的编辑中，也可以尝试着将这些技巧应用起来。