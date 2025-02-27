---
title: Neovim 入门教程 01——初识 Neovim
description: 了解 neovim / 一些前言
date: 2024-12-20 16:00:00
lastmod: 2024-12-20 16:00:00
image: ./posts/neovim-beginner-guide/cover.jpg
categories:
  - Tutorials
tags:
  - neovim
---

## 1 引言

我想，点进这篇教程的朋友应该或多或少知道 neovim 是什么。至少，我相信你应该听说过 vim，知道这是一个非常神奇、可以不依赖鼠标只靠键盘进行编辑的编辑器。江湖上流传着关于这个编辑器的诸多传说，众多博主逢人便吹嘘 vim 高手如何神乎其技，毫不吝惜地夸赞 vim 对工作效率的提升，让无数编程的爱好者们心驰神往。

不过，在介绍 neovim 和 vim 的关系是什么、neovim 到底是不是真的像传说中那样神奇高效这些内容之前，我想先抛出一个问题：如果要你设计一个可以脱离鼠标的编辑器，你会怎么做呢？事实上，在这个过程中，一个很重要的需要解决的问题就是如何区分快捷键和普通的文本输入。你可能会想，我们一来可以依靠一些非编辑区的按键例如上下左右键、<kbd>Home</kbd>、<kbd>End</kbd> 等移动光标，二来可以设计一些以功能键开头（例如 <kbd>Ctrl</kbd>、<kbd>Alt</kbd>）的快捷键。没错，许多现代的编辑器设计思路就是这样。

然而，这些现代编辑器或多或少还是大量使用了鼠标，而如果我们要把这些操作都用键盘替代，在非编辑区按键有限的情况下，我们势必需要大量使用以功能区按键为前缀的快捷键，这样不但会导致快捷键几乎无限制地增长、使得记忆负担大大增加，还会由于需要长时间按压 <kbd>Ctrl</kbd>、<kbd>Alt</kbd> 而造成小拇指的伤病——这不是开玩笑，一款大量使用这类快捷键的编辑器 Emacs 的很多用户就有这种问题。甚至，我个人是弹钢琴的，手指比较有力，所以长时间按这种边边角角的键并不是什么费劲的事情，但是我多少也觉得 <kbd>Ctrl</kbd> 按多了不太舒服。

于是，我们决定进一步改进这个设计——既然长时间按压功能键让人不适，我们干脆绑定一个按键，使得这个按键按下的时候自动进入到快捷键输入模式，此时输入的内容不会被作为文字内容添加进去，而是被当作快捷键的一部分。好了，如果你想到了这一步，那么你也就理解了 vim 的基本思路——模式编辑。

为什么先要抛出这个问题？因为我希望很多对于全键盘编辑完全没有任何概念的小伙伴大概了解一下 vim 到底是怎样的原理。说白了，这个东西并没有那么神奇，全键盘编辑也不是什么魔法黑科技。所以，我们也就回到了一开始的一个问题：vim 真的那么高效吗？关于这一点，我曾在[另一篇博客](/blog/posts/using-neovim/)里写过——与一些主流的观点不同，我并不认为使用 vim 会提高我们的效率：

> 需要明确，我们是在写代码而不是在聊天，这也就意味着我们大多数时间是在思考而不是在劈里啪啦地敲键盘；而即使到了敲键盘的时候，其效率也要看我们的打字速度——vim千好万好，也不会让我们打字的速度提高……
>
> 某种角度来说，vim和现代通用的编辑模式的区别，就类似于用手柄打游戏和用键鼠打游戏……很多人会觉得手柄打游戏比键鼠打游戏舒服，但一定不会有人觉得手柄打游戏会比键鼠更快让我们通关。

事实上，我认为 vim 真正的优势在于以下几点：

- 只用键盘确实很舒服——当你习惯键鼠协同操作的时候可能感受不到这一点，但一旦你习惯了纯键盘操作，就会意识到频繁操作鼠标有多么麻烦
- 高度的可定制性——这个编辑器，哪怕有一点点让你不满意的，都可以非常方便地去修改；e.g., 添加快捷键
- 一些可以在某些时候大幅度提高效率的功能——比如，宏操作
- 轻便而统一的开发环境

你现在可能并不是很能理解这后三点，没有关系，我们会在后面的教程中逐步覆盖这些内容。

此外，你可能会注意到虽然前文我们一直在说 vim，但是教程系列的名称里用的是 neovim。neovim 是 vim 的一个 fork 版本，二者基本操作基本相同，但是 neovim 对默认设置做了一些更加合理的调整、将 vim 的配置语言从 vimscript 换成了 lua 等一系列调整。总而言之就是，neovim 是一个更新、用起来（我个人认为）更加方便的 vim。需要注意，这篇教程大部分内容和 vim 并不互通，如果你只是想要学习 vim 的话，那么这一系列教程对你的帮助可能并不会很大。

## 2 为什么你可能不需要 neovim

是的，即使我在前面也是大肆夸奖了 neovim，但在正式开始教程之前，我是要劝退一波的，因为我见到了太多人用 neovim 只是因为脑子一热，但到头来不但没学会 neovim 还耽误了手头的工作。所以：

- 如果你只是编程的初学者，请不要碰 vim / neovim / emacs 等任何需要大量自定义的编辑器
  - 一来，你不应该把宝贵的学习时间花费在配置编辑器上；先把那门编程语言学明白
  - 二来，neovim 的配置也是需要编程能力的，我并不认为一个初学者有能力解决配置过程中可能出现的问题
- 如果你没有时间进行 DIY 或者没有 DIY 的意愿，VsCode / VS / JetBrains 系等工具更适合你
  - 一些插件会有 api 更新，如果你长时间不更新插件后哪一天突然想起来更新一下，可能会出现旧配置无效的情况（类似于 Arch 的滚挂）
  - 多数插件是社区维护的，所以作者可能会停止维护（比如说 null-ls），你可能需要自己找替代品
  - 当然，你可以选择一些现成的配置比如说 LazyVim，但我个人不喜欢这么做，具体原因后续教程会讲

很多人会说使用 neovim 的都是高手，也有人说使用的工具并不是判断一个人是不是高手的依据。那么实际上呢？我认为，熟练使用 neovim 的人一定以高手居多——不一定是那种顶尖的大牛，但技术一定不算差；然而，需要明白的是，这些人是因为熟悉技术所以能够熟练使用 neovim，而不是因为使用了 neovim 才成为了高手。所以，学习 neovim 之前，至少打好基础，弄明白一些基础概念，学明白至少一门语言，培养合理的编码风格。否则，学习 neovim 只会是浪费时间。

## 3 Neovim 的安装

初学者会有一个疑问：vim / neovim 在 Linux 上是公认的神器，那么它在 windows 上是否同样可用呢？不可否认，neovim 在 windows 上运行存在一些性能问题，但这种性能问题我觉得并不影响日常使用——事实上，我所遇到的性能问题无非是开机后首次启动会慢 50 ms；当然，成熟的 neovim 用户都对启动速度过度敏感，50 ms 还是太长了，但再不济，你也可以选择在 wsl 中使用 neovim。事实上，作为在 windows、linux、mac 上都高强度使用 neovim 进行开发的开发者，我个人认为这三个平台的使用体验没有什么太大的差异。

不过，无论你在哪个平台上使用 neovim，至少第一步的安装都还是比较简单的。你可以从 [GitHub](https://github.com/neovim/neovim/releases) 上下载安装文件、安装，也可以用包管理器（scoop / winget / apt / pacman / brew / ……）进行安装。

在安装后，我们可以在终端里输入 `nvim` 启动 neovim。如果想用 neovim 打开某个文件，可以用 `nvim filename` 的方式打开文件。

## 4 如何学习 neovim

Neovim 之所以这么劝退，是因为它确实很难学。难学并不仅仅体现在想要攒一份好用的配置费时费力；它还体现在大量的默认快捷键上——如果你在 26 个字母里看一圈，你会发现每个——是的，每一个——字母自己都能参与到了快捷键的构成中，而很多时候一个字母的大小写还代表了不同的快捷键；而 neovim 自身又提供了海量的命令。其体量有多么巨大，去看看文档就足以知道个大概（我用了 neovim 两年半了，我都不敢说我会了 neovim 一半的功能）。这往往给初学者带来了有一个困难——他们即使背诵了所有快捷键，也不知道什么时候用什么快捷键更好。

然而，初学者需要明确的一点是，这些并不是刚刚接触 neovim 的人应该 care 的。计算机知识浩如烟海，但初学 C 语言的同学并不需要掌握编译原理才能开始编程。事实上，学习 neovim 只需要从几个基础的概念、快捷键和命令开始，至于其他更加复杂的内容，你会在使用的过程中逐步发现自己目前掌握的内容不足以满足自己的需求，此时你自然而然就会去寻找解决方案。

学习 neovim 是一个积累的过程——在各种意义上。你的配置不可能一开始就是完美的，你的操作习惯不可能一开始就是合理的，但是你一定要有勇气推翻重来。当你自己定义的快捷键不合理，要敢于克服不习惯的感觉修改快捷键（事实上，我在开始写 [IceNvim](https://github.com/Shaobin-Jiang/IceNvim) 的时候就推翻了很多自己用了一年多的快捷键）；当你的操作习惯在 neovim 中有更好的方式，你也要敢于换成更好的那种方式（比如说，把 `ddo` 改成 `cc`，比如说从 `xi` 到 `r`，比如说`.s`）。

另一个需要注意的事情就是，不要尝试去直接抄成熟的配置。你可以在 GitHub 上找到大量的 neovim 配置，比如 LazyVim、NvChad、AstroNvim，我自己也开源了自己的配置，但是我觉得这些配置的作用一是给实在懒得自己配置的人直接拿去用，二是给已经了解 neovim 的朋友做参考。对于初学者来说，这些配置价值有限，因为你很难理清其中的一些脉络结构，也不一定看得懂一些优化操作的目的是什么。如果你愿意认真读懂其中一些片段并用到你自己的配置中倒也是可以，但是如果你直接把实现某些功能的片段抄下来，那真的就是毫无意义了。
