---
title: arch linux更新initramfs失败解决方法
tags:
  - Linux
---

遇到过好几次，arch linux在内核更新，重新生成initramfs时，会遇到生成失败的问题，具体体现为找不到相关的内核模块，实际上解决只需要一行命令

```sh
$ sudo depmod -a /lib/modules/(新内核版本)
$ sudo mkinitcpio -P
```

问题解决
