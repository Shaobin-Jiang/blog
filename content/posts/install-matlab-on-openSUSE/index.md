---
title: 在 openSUSE 上安装 Matlab R2022b
description: 解决了权限、桌面图标等问题
date: 2023-10-27 12:00:00
lastmod: 2023-10-27 12:00:00
image: cover.jpg
categories:
  - Linux
tags:
  - linux
  - matlab
---

Matlab 官方目前完美支持的 linux 发行版包括 Ubuntu、Debian、Red Hat、SUSE Linux Enterprise。至于 openSUSE 嘛……

> MathWorks supports commercial SUSE Linux Enterprise Desktop (SLED) 11.3 and higher, not openSUSE 11.3 and higher. A current release of openSUSE should work just as well, but MathWorks is in a limited position to provide support for Linux distributions which we haven't fully qualified.

话虽如此，我还是需要用 matlab 的。而且此前我在 Arch 上已经成功用上了 matlab，那么官方明确表明 "should work just as well" 的 openSUSE 应该安装起来问题也不大。

不过实际上，我在安装的时候还是遇到了一些小小的问题，在此做一些记录。

## 无法运行 installer

第一个问题，是我无法运行 matlab installer……是的，我在第一步就卡死了，每次运行那个 `install` 文件的时候都会报这样一个错误：

```
terminate called after throwing an instance of 'std::runtime_error'  
what(): Failed to launch web window with error: Unable to launch the MATLABWindow application.
The exit code was: 127
```

我用 openSUSE 做关键词搜索了很久，但都没找到，最后却歪打正着在 Manjaro 相关的一个[帖子](https://juejin.cn/post/7230860299984273467)下面找到了答案。问题似乎出在了动态链接库上，我们需要手动修改一个 `LD_PRELOAD` 变量：

```bash
export LD_PRELOAD=/lib64/libfreetype.so.6
```

具体的取值可能根据系统有所不同，在 openSUSE Tumbleweed 上是这样的。在做完这个设置之后，就可以运行 installer 了。

## 缺少相应权限

运行 installer 的时候有一点需要注意：我们不可以用 `sudo` 运行，否则还是会报错。但这样的话，在后续安装的时候，如果我想要将 matlab 安装在需要管理员权限的路径下的时候，比如说 `/opt` 下，就做不到了。对此，我的解决方案是先创建好安装目录，然后用 `chown` 修改一下：

```bash
sudo chown <username> /opt/Matlab/R2022b
```

## 创建相应的 `.desktop` 文件

默认情况下，在安装之后，matlab 并不会为我们创建相应的 `.desktop` 文件，需要手动创建，因此我执行了这样一段命令：

```bash
sudo echo "[Desktop Entry]
Type=Application
Terminal=false
MimeType=text/x-matlab
Exec=/opt/Matlab/R2022b/bin/matlab -desktop
Name=MATLAB
Icon=/opt/Matlab/R2022b/bin/glnxa64/cef_resources/matlab_icon.png
Categories=Development;Math;Science
Comment=Scientific computing environment
StartupNotify=true" > /usr/share/applications/matlab.desktop
```

其中比较 tricky 的是怎么找 matlab 的 icon，这个在网上实在是不太好找，不过还是被我找到了（得意）。如此这般操作之后，我终于是将 matlab 成功安装在了我的 openSUSE 上。
