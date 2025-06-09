---
title: AOSP 编译&开发技巧
tags:
  - Android
---

最基础的 `brunch xxx` 就不多说了，主要记录一下如何模块编译和清理编译产物重新编译。

### 模块编译
- `make xxx`， 其中 `xxx` 是在 `Android.bp` 中定义的编译目标，需要在aosp源码目录执行
- `mm -B` 或者 `mma -B` ，直接到对应模块目录下执行，只编译该模块
- `mmm packages/apps/xxxx`，`mmm` 后接模块的目录，需要在aosp源码目录执行
- 编译镜像文件（比如`boot.img`, `system.img` 等），参数可以是 `make bootimage`，`make recoveryimage` 。具体可以用 `m help` 查看。
- `make -j10 dist`，也是编译整个包的命令之一，不过和brunch不同的是，这个可以指定编译用的线程数量，防止线程太多而oom。

~~以上命令均可用 `-j` 指定编译所用线程数。~~

### 编译产物清理
- `make clean-<module>`，clean后接编译的模块名称，模块需要在构建系统中有定义（make的目标，Android.bp 定义的编译目标等），比如清理编译后 `Recorder` ，命令为 `make clean-Recorder`。此命令会清除编译的中间产物，十分适合调试开发
-  `make installclean` ，此命令会清理app生成目录 `out/target/product/<product_name>/obj/APPS/` ，但不会清除中间产物的目录 `out/target/common/obj/APPS/`

做个笔记记录下，以防忘记。

参考来源

- [# Android 源码编译技巧--模块清理](https://blog.csdn.net/weixin_44021334/article/details/118183291)
- [# Android 源码编译技巧--模块编译](https://blog.csdn.net/weixin_44021334/article/details/106944138)
