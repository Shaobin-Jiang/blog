---
title: 解决 neofetch 在 wsl 上运行缓慢的问题
description: 这可能是 wsl 上包管理器的问题
date: 2023-10-03 14:00:00
updated: 2023-10-04 14:00:00
image: cover.jpg
categories:
  - Linux
tags:
  - linux
---

neofetch 是一个 linux 下非常好用的打印系统信息的工具，而且它除了打印系统信息之外还会在旁边打印系统的 logo（如封面图所示），很是漂亮。所以，我也会在我用的每一个 linux 发行版的 `.bashrc` 中加入一句 `neofetch`，这样每次打开终端模拟器的时候都会有好心情。

但是，一直以来我都没有将这个习惯应用在 WSL 中。这是因为，不管是 neofetch，还是 screenfetch（类似 neofetch），它们在 WSL 下运行都**极 其 缓 慢**。

这个缓慢是真的缓慢，在 WSL 下每次运行 neofetch 我都需要等待 3 - 5 秒，这可就完全违背我的初衷了，毕竟每次启动 WSL 都要等上好几秒钟真的很不爽（相比之下在物理机上用 neofetch 就是秒出结果）。于是，我决定排查一下问题出在哪里。

首先，一个重要的前置知识来自 neofetch 的[文档](https://github.com/dylanaraps/neofetch/wiki/Customizing-Info#speed-up-the-script-by-running-the-functions-asynchronously)。其中提到，我们可以通过在 neofetch 的配置文件语句后面加上一个 `&` 以让这些语句异步执行。这就意味着，如果不加上 `&` 的话，配置文件中的语句就是同步执行的。

默认情况下，neofetch 的配置文件（位于 `~/.config/neofetch/config.conf`）部分内容如下：

```conf
# See this wiki page for more info:
# https://github.com/dylanaraps/neofetch/wiki/Customizing-Info
print_info() {
    info title
    info underline

    info "OS" distro
    info "Host" model
    info "Kernel" kernel
    info "Uptime" uptime
    info "Packages" packages
    info "Shell" shell
    info "Resolution" resolution
    info "DE" de
    info "WM" wm
    info "WM Theme" wm_theme
    info "Theme" theme
    info "Icons" icons
    info "Terminal" term
    info "Terminal Font" term_font
    info "CPU" cpu
    info "GPU" gpu
    info "Memory" memory

    # info "GPU Driver" gpu_driver  # Linux/macOS only
    # info "CPU Usage" cpu_usage
    # info "Disk" disk
    # info "Battery" battery
    # info "Font" font
    # info "Song" song
    # [[ "$player" ]] && prin "Music Player" "$player"
    # info "Local IP" local_ip
    # info "Public IP" public_ip
    # info "Users" users
    # info "Locale" locale  # This only works on glibc systems.

    info cols
}
```

可以看到，这些信息的获取都是同步的。那么排查方案就很简单了，我们可以看看打印这些信息的时候，在何处有停滞就可以了。

不难看到，neofetch 先是输出到 uptime，然后停顿，接着输出 packages 和 shell，再停顿，然后输出 terminal、CPU 等信息。因此我们大概可以将问题定位到出来了：统计包的数量似乎是一个很费时间的操作，统计 DE 等信息似乎也很费时间。

WSL 统计安装的 package 数量费时似乎是一个 bug，可以参加 GitHub 上的这条 issue：[WSL2 Linux enumerating "packages" takes a long time](https://github.com/dylanaraps/neofetch/issues/2022)。至于获取 DE 等信息缓慢，虽然这大概率是因为没有桌面环境所以获取不到 DE、WM、主题等信息，但是我还是没太明白为什么这会导致 neofetch 卡住。

不过，解决方案倒是很简单，直接在配置文件中把这些会卡住 neofetch 的选项统统注释掉即可：

```conf
print_info() {
    info title
    info underline

    info "OS" distro
    info "Host" model
    info "Kernel" kernel
    info "Uptime" uptime
    # info "Packages" packages
    info "Shell" shell
    # info "Resolution" resolution
    # info "DE" de
    # info "WM" wm
    # info "WM Theme" wm_theme
    # info "Theme" theme
    # info "Icons" icons
    info "Terminal" term
    # info "Terminal Font" term_font
    info "CPU" cpu
    info "GPU" gpu
    info "Memory" memory
}
```

---

Update：

今天（2023.10.04）发现获取 DE 等信息又不会卡住了，很神奇……

以及，想了想如果还是需要显示包的数量，也可以自己采用更快的方式进行统计。比如说，在 openSUSE 上可以用这一行替代原本统计包数量的那行：

```conf
prin "Packages" "$(zypper search -i | grep ^i | wc -l) (rpm)"
```

这个应该不难理解，`zypper search -i` 会列出所有已经安装的包，当然由于这些信息中存在一些和安装了哪些包无关的信息，所以可以用 `grep` 找出所有以 `i` 开头的项目（可以自己列出所有的包看看格式），然后交给 `wc -l` 进行统计。
