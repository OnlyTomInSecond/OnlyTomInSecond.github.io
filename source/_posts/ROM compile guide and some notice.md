---
title: ROM 编译指南及踩坑
---
今年暑假编译了 `lineageos19.1` ，写篇笔记稍微记录一下这过程中存在的问题和解决方法，以及一些资源的位置

- `repo` 相关配置和同步

众所周知，由于谷歌官方的android镜像站在国内早已404了，因此在执行 `repo init -u`  后直接同步是无法访问的（除非配置代理，但实际无必要，因为android源码很大，下载完加checkout大小在100g以上），因此需要修改同步地址，修改项目目录下 `.repo/manifests/default.xml` 文件，修改以下内容

```xml
<remote  name="aosp"
           fetch="https://mirrors.tuna.tsinghua.edu.cn/git/AOSP"
           review="android-review.googlesource.com"
           revision="refs/tags/android-12.1.0_r22" />
```

把原来的 `fetch=...` 的地址修改为清华源的地址。如果你所在地区github访问速度还行的话，可以不用修改lineageos19.1的自己默认同步地址（这里说一下，一般类原生rom基于aosp开发，在此之上添加一些自己的功能和修改，因此同步源码时，可以用清华的镜像加速aosp源码的部分，剩下的部分可以直接github拉取）。如果你cpu线程够多，你可以设置
`sync-j=8` ，后面的8就是同步线程数。

修改好后执行 `repo sync` 即可。如果是服务器上执行此命令，可以使用 `tmox` 或者 `screen` 这类的终端复用软件来执行，这样可以断开ssh连接是仍然能运行。

- 关于设备树和编译所需

一般同步完后，[lineageos官方的教程](https://wiki.lineageos.org/devices/dipper/build#prepare-the-device-specific-code) 会让你用 `breakfast` 来选择官方支持的设备，来同步设备的设备树、内核等源码。但是如果设备不是官方所支持，那可以选择去github上找齐相关的代码，并下载到相应位置来进行编译。比如说小米8的设备代号是`dipper`，在执行 `breakfast dipper` 以后，会先下载 `.repo/local_manifests/roomservice.xml` ，这个文件里指明了设备特定代码的下载地址和下载后需要放置的位置，内容大概长这样：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <project path="device/xiaomi/dipper" remote="github" name="LineageOS/android_device_xiaomi_dipper" />
  <project path="device/xiaomi/sdm845-common" remote="github" name="LineageOS/android_device_xiaomi_sdm845-common" />
  <project path="hardware/xiaomi" remote="github" name="LineageOS/android_hardware_xiaomi" />
  <project path="kernel/xiaomi/sdm845" remote="github" name="LineageOS/android_kernel_xiaomi_sdm845" />
</manifest>
```

容易看出，需要3个git仓库，分别是
- `device/xiaomi/dipper`，所下载的地址是`LineageOS/android_device_xiaomi_dipper`，前面省略了 `https://github.com/` ，这就是设备树。
- `device/xiaomi/sdm845-common`，这个下载的是 `sdm845` 这个芯片的设备所共用的部分，可以理解为 `common deivce tree` （至于这部分，可能部分厂家同处理器型号的手机少，可能没有这一部分，直接合并到上一条去了。
-  `kernel/xiaomi/sdm845` ，看名字便知是内核源码，如果厂商开源了内核，可以尝试使用（一般来说lineageos 的内核是自己单独开发，基于上游的caf，并加入厂商特定的驱动而成，直接基于厂商的内核可能会无法通过编译）

如果设备没有被官方支持，你可以通过修改这个配置文件的地址，对应到github仓库上，来使用 `repo` 这个命令来统一管理和同步，但也可以直接clone对应仓库到项目的相关位置。

- 关于编译

同步完成后，就可以继续按照官方教程，用 `brunch` 命令来编译。如果你设备内存足够（32G以上），可以不考虑开swap（但还是建议开，毕竟linux休眠要用到），16g用户建议开16gswap，并采用默认配置即可通过编译，8g用户需要修改编译内存的使用量和编译时使用的线程数量，来减少内存占用。（不过我忘记在哪里改了，现在改貌似不会生效，原因未知。）

编译用时：以我笔记本4800H 8C16H ，16g内存，在archlinux 中，内核版本为5.18.6情况下，从头开始编译需要4.5h，内存占用最高时为 16g RAM + 10g swap，系统无死机情况。

同时，如果下一次编译时，同步源码后不必执行 `m clean` 这类的清除编译产物的命令，直接编译即可，除非出现难以解决或无法解决的编译错误，再尝试来清除编译产物来排除问题。一般情况下，第一次成功后，如果对源码无特大修改，第二次编译时间会大大缩短（6分钟即可出一个包）。

- 关于自行修改代码及相关维护策略

如果自己要对系统进行源码上修改或者修复bug，一般来说直接修改是可以，但如果你源码要跟从上游，基于上游再来应用自己的更改时，推荐使用 `git` 开一个自己的单独分支，比如你自己修改了 `framework/base` 里的内容，可以先新建一个自己的分支，然后把自己的更改提交到自己的分支里，这样同步时不会出现说你有更改未提交，而同步失败了。同时下一次同步源码后，只需要进入你修改过的仓库（`repo`），执行 `git merge` 或者 `rebase` 来应用自己的修改就可以了。

- 关于二次编译及其一些问题

- 关于 `breakfast` 、`lunch` 、`brunch`

首先，如果按照官方的编译教程，会教你直接 `breakfast <device>` 后 `brunch <device>` ，其中第一个操作是下载相关设备的代码（设备树，内核，vendor blobs等），并初始化必要的环境变量（ `lunch` 后选devices就是干这个的，进程还在时只需要跑一遍即可）

首次编译成功后，下一次编译的时间会大大缩短，因为编译系统只会再次编译有源码修改的组件，但位清除编译缓存的情况下，即使有部分源码修改，修改的部分并不一定能正确编译。因此需要这几个命令

- 几个编译命令
    - `m` ： 相当于 `make all`
    - `mm` ： 后接源码目录，编译某个目录下所有的一定义的编译目标（一般在各 `makefile` 里或者 `Android.bp` 里）
    - `mmm` ：和 `mm` 类似用法，个人用起来看不出具体区别
    - `make clean` ：清除全部编译产物（除了ccache），直接删除源码目录下的 `out/` 。
    - `cmka` ：后接 `Makefile` 里定义的编译目标，先对此目标执行clean后编译，如果出现更改未生效或者资源文件（`resource file`） 未更新（比如翻译的文本和其他文本错位甚至乱码），可尝试执行此命令

