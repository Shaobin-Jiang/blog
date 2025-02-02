---
title: pwsh 启动超级慢？可能是 conda 的锅
description: 并非通解，主要是记录定位问题的过程
date: 2025-02-02 20:30:00
lastmod: 2025-02-02 20:30:00
image: cover.jpg
categories:
  - Tools
tags:
  - windows
---

（我需要把话说在前面，这篇博客多少有点标题党，因为这根本不是一个通解。虽然我已经在简介里写了，但这里我还是要重申一下，这篇博客更主要的是记录解决这个问题的过程）

---

我买了一台 mac。在拿到新电脑之前，我还在 windows 上尝试部署一个 ai 的项目。此前，我使用 python 主要是用 poetry 管理项目依赖，因为之前我主要是写 Django，版本冲突什么的问题不大。但是到了 ai 这面，python 的版本、pytorch 的版本、各种各样的包的版本冲突宛如一场噩梦，尤其是使用 poetry 管理 python 版本并没有那么方便，所以我选择了诸多机器学习前辈都选择的解决方案：conda。

拿到 mac 的时候，我刚好安装好了 conda 并启动了它的虚拟环境。鉴于那个 ai 项目仅仅是我出于兴趣才去尝试部署的，我决定先把 mac 捣鼓明白再回来这边。然后，就因为各种原因，我就一直在 mac 上进行各种操作，windows 快有一个月除了打 2k25 没有进行任何正经工作。

然后有一天，我突然想起来打开了 windows terminal——是的，我废话了这么多，终于要讲到我遇到的问题了——然后我发现，这个东西启动花了我将近 3 秒钟。这我可就不能忍了，我可是高贵的 mac 用户（笑），不说这上万的机子上 kitty 终端启动多么流畅，哪怕是在我的旧笔记本上装的 endeavouros 上启动终端也比这快得多。更主要的是，我依稀记得在我之前启动 windows terminal 没有这么慢啊。

我的 windows terminal 做了如下配置：启动时默认运行 `pwsh -NoExit`；然后，我编写了 `$PROFILE` 文件，内容如下：

```ps1
Invoke-Expression (&starship init powershell)

fastfetch --logo Windows
```

总结起来，windows terminal 在启动后会做三件事：

- 启动 pwsh（powershell core）
- 启动 starship（一个美化工具，类似 zsh 的 powerlevel10k）
- 启动 fastfetch

于是我尝试 `scoop update pwsh`（顺道一提，这个操作需要在 powershell.exe 中运行），无果——看来不是 pwsh 本身版本的问题。

注释掉启用 starship 和 fastfetch 的代码，启动仍然很慢。

这下我就很懵了，不是 pwsh 本身的问题，也不是 starship 或 fastfetch 拖慢了启动速度，那是怎么回事？于是我求助了搜索引擎，并看到了这样一条解决方案：

> PowerShell 启动时会加载配置文件（如 `$PROFILE`），如果配置文件中有复杂的脚本或加载了过多的模块，可能会导致启动变慢。
>
> ...
>
> 临时禁用配置文件：
>
> 启动 PowerShell 时加上 -NoProfile 参数，跳过配置文件加载：
>
> ```ps1
> pwsh -NoProfile
> ```

虽然我确信我的配置文件并没有很臃肿，但我还是按照它的建议临时禁用了配置文件。然后，奇迹发生了，pwsh 的启动速度回归到了 1 秒以内——不是很快，但是我记忆中的水平。

除了速度的变化，我还注意到了一个变化——相比于启用配置文件的时候，此时的 pwsh 没有显示 `(base)` 字样——是的，如果你使用过 conda，你应该知道默认启用 conda 环境的时候会在命令提示符上添加这一字样。这意味着，conda 在 pwsh 的配置文件里进行了启动，拖慢了 pwsh 的启动速度。我记得 mac 上，conda 会在 `.zshrc` 末尾添加上启动 conda 的相关代码；而鉴于我的 pwsh 配置中并没有 conda 相关的代码，那么答案就是，pwsh 的配置文件不止一个？

于是，我运行了如下命令：

```ps1
$PROFILE | Format-List * -Force
```

得到了如下结果：

```
AllUsersAllHosts       : C:\Installed\Scoop\apps\pwsh\7.5.0\profile.ps1
AllUsersCurrentHost    : C:\Installed\Scoop\apps\pwsh\7.5.0\Microsoft.PowerShell_profile.ps1
CurrentUserAllHosts    : C:\Users\dell\Documents\PowerShell\profile.ps1
CurrentUserCurrentHost : C:\Users\dell\Documents\PowerShell\Microsoft.PowerShell_profile.ps1
Length                 : 67
```

四条配置文件中，前两条明显是全局的配置，料想 conda 不会在这里面拉屎；第四条配置文件就是 `$PROFILE` 返回的文件路径，所以最有嫌疑应该是第三条？打开文件后，我果然看到了 conda 用来初始化的代码：

```ps1
#region conda initialize
# !! Contents within this block are managed by 'conda init' !!
(& "C:\Installed\Scoop\apps\miniconda3\current\Scripts\conda.exe" "shell.powershell" "hook") | Out-String | ?{$_} | Invoke-Expression
#endregion
```

至此，问题解决，我只需要手动删除这部分代码即可。当然，因为我还是需要用到 conda，所以我在我的配置文件里编写了一个函数，用来按需激活 conda：

```ps1
function condaini {
    conda shell.powershell hook | Out-String | Invoke-Expression
}
```

---

（在写完这篇之后，我发现此事在 github 上亦有记载：<https://github.com/conda/conda/issues/11648>。而且，这甚至是 22 年就有的旧 issue 了。所以为什么 conda 不愿意单独设计一个命令用来初始化、而不是默认强制启用 conda 呢？）
