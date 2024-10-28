---
title: 优化 hyprland 显示器配置
description: 多显示器时自动关闭笔记本显示器
date: 2024-10-28 17:30:00
lastmod: 2024-10-28 17:30:00
image: cover.jpg
categories:
  - Linux
  - Tools
tags:
  - linux
  - hyprland
---

## 1 我遇到的问题

我在我的笔记本上安装了 hyprland 有一段时间了，作为一个 WM，它的功能自然是比不上各种强大的桌面系统全面，不过我在使用起来的时候倒也没有特别感受到不方便，毕竟我在大多数情况下不会被一些特定的功能所绑定，如果和原来的操作习惯有冲突，在可以接受的情况下，我一般会选择改变我的操作习惯。

当然，这只是一般情况下。偶尔，我也会遇到一些让我确实十分依赖的功能。譬如，我在使用 hyprland 的时候十分想念的一个功能就是，显示器的管理。

诚然，hyprland 自带了禁用启用显示器、调整布局分辨率刷新率等一系列功能。但是，自从我在购置了显示器之后，我对于显示器的管理又新增了一个需求：在笔记本外接显示器的时候自动关闭笔记本自己的显示器。我的宿舍桌面空间并不是很充足，双屏协同工作效果不甚理想，但如果就这么把笔记本电脑屏幕晾在一边，不美观不说，还经常容易出现鼠标被一不小心移动到笔记本屏幕上，很是烦躁。

这当然是在 `hyprland.conf` 中一行配置就可以禁用的事：

```
monitor = eDP-1, disable
```

但是我还经常需要把笔记本带出去，这个时候我就会惊喜地发现，我的笔记本黑屏了，还需要 <kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>F3</kbd> 进入一个新的 tty，把配置文件修改回来，再回到 hyprland，还是很麻烦。所以，有没有一种办法，可以自动化这个流程呢？

## 2 需求分析

以上的需求，大概可以拆分成以下几个功能：

- 在检测到新显示器接入的时候，禁用笔记本显示器
- 在检测到显示器移除的时候，如果只有一个显示器，则启用笔记本显示器
- 在刚开机的时候，检测显示器数量，决定是否启用笔记本显示器

## 3 功能实现

在使用 ags 编写我的桌面小组件的时候，我可以方便地使用 hyprland service 提供的事件监听。但是当我要编写 bash 脚本的时候，该如何监听 hyprland 事件呢？答案是，使用 [IPC](https://wiki.hyprland.org/IPC/) 功能。此时，我们需要先安装 `socat`。以 Arch 系为例，我们可以使用 `sudo pacman -S socat` 来进行安装。

官方文档为我们提供了这样一个例子：

```bash
#!/bin/sh

handle() {
  case $1 in
    monitoradded*) do_something ;;
    focusedmon*) do_something_else ;;
  esac
}

socat -U - UNIX-CONNECT:$XDG_RUNTIME_DIR/hypr/$HYPRLAND_INSTANCE_SIGNATURE/.socket2.sock | while read -r line; do handle "$line"; done
```

不难看到，我们将定义好的 `handle` 函数作为了 hyprland 事件的回调函数，hyprland 会在事件触发的时候调用它并传入一个参数。经过实验，这个传入参数了包含了事件的名称和一些数据，像下面这样：

```
monitorremoved>>HDMI-A-1
monitoradded>>HDMI-A-1
monitoraddedv2>>1,HDMI-A-1,AOC Q27G2S 18DPCHA00488
```

这下，问题就简单了。我们只需要根据传入的事件和数据进行判断即可。首先，我们要判断当前事件。这里，我们只需要截取传入参数的前 x 位字符进行判断即可。这里因为我只想监听 monitoradded 而不想监听 monitoraddedv2 事件，所以我的代码是这样的：

```bash
handle() {
    # 获取第一个函数参数 $1 的第 1 - 14 个字符，判断是否为 monitorremoved
    if [[ ${1:0:14} == "monitorremoved" ]]; then
        # 调用 callback，并将显示器信息传入
        # ${1:16} 是截取函数参数 $1 的第 17 个字符至末尾
        # 多提一嘴，我后面并没有在 callback 中使用到这个传入参数，但总感觉有可能会用到
        callback ${1:16}
    fi
    if [[ ${1:0:14} == "monitoradded>>" ]]; then
        callback ${1:14}
    fi
}

socat -U - UNIX-CONNECT:$XDG_RUNTIME_DIR/hypr/$HYPRLAND_INSTANCE_SIGNATURE/.socket2.sock | while read -r line; do handle "$line"; done
```

这里我用了一个统一的函数 `callback` 来处理显示器添加和移除的回调，这是因为我们前面说了，还要在开机的时候检测当前的显示器数量。这个时候，直接调用我们的这个 `callback` 就可以了。

我们来继续编写 `callback`：

```bash
callback() {
    monitor_count=$(hyprctl monitors | grep -c " (ID [0-9]):")
    if (($monitor_count > 1)); then
        hyprctl keyword monitor eDP-1,disable
    else
        hyprctl keyword monitor eDP-1,preferred,0x0,1
    fi
}
```

首先，我们通过 `hyprctl monitors` 获取当前的显示器数量。这句命令的输出结果类似这样：

```
Monitor eDP-1 (ID 0):
	1920x1080@59.97700 at 0x0
	description: LG Display 0x05F2
	make: LG Display
	model: 0x05F2
	serial: 
	active workspace: 0 ()
	special workspace: 0 ()
	reserved: 0 0 0 0
	scale: 1.00
	transform: 0
	focused: no
	dpmsStatus: 1
	vrr: false
	solitary: 0
	activelyTearing: false
	disabled: true
	currentFormat: XRGB8888
	availableModes: 1920x1080@59.98Hz 1920x1080@47.98Hz 

Monitor HDMI-A-1 (ID 1):
	2560x1440@144.00101 at 0x1080
	description: AOC Q27G2S 18DPCHA004885
	make: AOC
	model: Q27G2S
	serial: 18DPCHA004885
	active workspace: 3 (3)
	special workspace: 0 ()
	reserved: 0 42 0 0
	scale: 1.25
	transform: 0
	focused: yes
	dpmsStatus: 1
	vrr: false
	solitary: 0
	activelyTearing: false
	disabled: false
	currentFormat: XRGB8888
	availableModes: 2560x1440@60.00Hz 3840x2160@59.94Hz 3840x2160@50.00Hz 2560x1440@144.00Hz 2560x1440@120.00Hz 1920x1080@119.88Hz 1920x1080@60.00Hz 1920x1080@59.94Hz 1920x1080@50.00Hz 1280x1440@59.91Hz 1280x1024@75.03Hz 1280x1024@60.02Hz 1280x720@59.94Hz 1280x720@50.00Hz 1024x768@119.99Hz 1024x768@100.00Hz 1024x768@75.03Hz 1024x768@70.07Hz 1024x768@60.00Hz 800x600@119.97Hz 800x600@100.00Hz 800x600@75.00Hz 800x600@72.19Hz 800x600@60.32Hz 800x600@56.25Hz 720x576@50.00Hz 720x480@59.94Hz 640x480@120.01Hz 640x480@99.99Hz 640x480@75.00Hz 640x480@72.81Hz 640x480@59.94Hz 640x480@59.93Hz
```

不难注意到，每一个显示器后面都有一个 `(ID x):` 这样的模式。所以，我们就可以通过 `grep -c` 来获取显示器的数量。

接下来，我们就可以通过获取的显示器数量判断到底要禁用还是启用笔记本显示器。那么，禁用 or 启用的功能该怎么实现呢？可能会有朋友想到，在 hyprland 文档的 dispatchers 一部分提到过 `dpms` 这个东西——通过 `hyprctl dispatch dpms off eDP-1` 这样就可以“禁用”笔记本显示器。

我一开始也使用了这个，但是发现不太复合我的预期。`dpms` 所做的只是让显示器黑屏，实际它并没有禁用掉，你仍然可以将鼠标移动到这个黑屏的显示器上，只不过你看不到而已。那么怎么通过命令实现 `monitor = eDP-1, disable` 这样的效果呢？

答案是使用 `hyprctl keyword`。在命令行运行 `hyprctl keyword monitor eDP-1,disable` 和在配置文件中写下 `monitor = eDP-1,disable` 是完全一致的。所以，我们就有了这样的代码：

```bash
if (($monitor_count > 1)); then
    hyprctl keyword monitor eDP-1,disable
else
    hyprctl keyword monitor eDP-1,preferred,0x0,1
fi
```

OK，不要忘了我们还要在开机的时候运行这个 `callback`。值得一提的是，hyprland 在初始化的时候并不会触发 `monitoradded` 事件，需要我们手动进行调用。调用 `callback` 需要在 `socat` 调用之前，因为后者会阻塞进程。

另外，一个比较恼人的点在于，hyprland 会默认把工作区 1 留给笔记本显示器，即便在它被禁用后，外接显示器上的工作区还是 2。所以，我们要手动将工作区修改一下：

```bash
hyprctl dispatch workspace 1
```

## 4 完整代码

完整代码如下：

```bash
callback() {
    monitor_count=$(hyprctl monitors | grep -c " (ID [0-9]):")
    if (($monitor_count > 1)); then
        hyprctl keyword monitor eDP-1,disable
    else
        hyprctl keyword monitor eDP-1,preferred,0x0,1
    fi
}

handle() {
    if [[ ${1:0:14} == "monitorremoved" ]]; then
        callback ${1:16}
    fi
    if [[ ${1:0:14} == "monitoradded>>" ]]; then
        callback ${1:14}
    fi
}

callback
hyprctl dispatch workspace 1

socat -U - UNIX-CONNECT:$XDG_RUNTIME_DIR/hypr/$HYPRLAND_INSTANCE_SIGNATURE/.socket2.sock | while read -r line; do handle "$line"; done
```

我们可以将其保存在 `~/.config/hypr/monitor.sh`，然后在 `hyprland.conf` 中使用 `exec-once = sh ~/.config/hypr/monitor.sh` 保证其在 hyprland 启动的时候被调用即可。
