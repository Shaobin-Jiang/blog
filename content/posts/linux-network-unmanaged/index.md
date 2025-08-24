---
title: 记录一次 Linux 下网络无法连接的解决过程
description: 网络压根无法连接的时候该怎么办？
date: 2024-11-08 15:40:00
lastmod: 2024-11-08 15:40:00
image: cover.jpg
categories:
  - Linux
tags:
  - linux
  - network
---

（水一篇水一篇）

今天打开电脑的时候发现莫名其妙地无法连接网络了，waybar 里 nm-applet 的图标显示网络被禁用。就，非常莫名其妙，我前一天刚刚用过电脑，除了拿来看了一会文献好像也没做什么奇怪的工作，不知道是不是滚动更新的时候哪个地方出了问题还是咋的。

于是，运行了一下 `nmcli connection show` 命令发现：

```
NAME         UUID                                  TYPE  DEVICE 
BNU-mobile   c342ff23-cc12-49ab-966b-82aa24cca0aa  wifi  --     
BNU-Mobile   2226ebe3-c819-4d0a-a805-339710840397  wifi  --     
BNU-Student  c1b7f661-2fa1-421f-93d3-5e1ceaddc016  wifi  --
```

尝试连接：

```
> nmcli connection up BNU-mobile
Error: Connection activation failed: No suitable device found for this connection (device lo not available because device is strictly unmanaged).
```

进一步查看 NetworkManager 是否管理网络连接：

```
> nmcli networking
disabled
```

果然是没有，此时就需要手动开启：

```bash
nmcli networking on
```

此时网络连接恢复正常。
