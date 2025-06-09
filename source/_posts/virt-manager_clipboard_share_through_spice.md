---
title: spice文件与剪贴板共享
tags:
  - Linux
---

由于之前在linux上用kvm+qemu安装完windows后，主机和虚拟机之间的数据传输成了我感到很头疼的一个问题，之前通过windows下文件共享实现了两者之间的文件传输，但是未能实现剪贴板共享，而我有很多事情都需要在主机中找资料然后复制到虚拟机中，这样以来不能共享剪贴板实在是不方便

于是上网搜，找到了解决方法，那就是用`spice agent`

上官网一看，原来这个技术支持的不止是文件传输和剪贴板共享，还可以实现音频分路，显卡加速（qlx），USB重定向等等功能，实属强大。接下来说一下怎么配置文件剪贴板共享。

首先在主机是linux的情况下，安装`spice-vdagent`和`xserver-xorg-video-qxl`,运行命令`sudo apt install xserver-xspice xserver-xorg-video-qxl`

安装完后，打开`virt-manager`，在虚拟机中添加硬件，步骤为`Add Hardware`->`Channel`，选`spicevmc`。如果虚拟机是linux，那同样在其中安装 `spice-vdagent`和`xserver-xorg-video-qxl`即可，如果虚拟机是windows，则安装官网上的`spice-guest-tools`[link](https://www.spice-space.org/download/windows/spice-guest-tools/spice-guest-tools-latest.exe)，安装完重启虚拟机即可