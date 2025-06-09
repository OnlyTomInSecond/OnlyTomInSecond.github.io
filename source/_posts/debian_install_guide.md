---
title: Debian 安装小记
---

## Debian 安装小记

来来回回已经安装过 `Debian` 几回了，就系统本身而言，体验是很不错的。但要使其真正完全工作，做到bug最少，其中要注意的事情还是比较多，因此就有了这一篇记录

--- 

### 1.安装前准备

真正要准备的东西不多，U盘，安装镜像（推荐使用DVD镜像，里面包含了gnome和kde桌面，而且完全没有网络时安装也不会报错，比如[这个](https://mirrors.bfsu.edu.cn/debian-cd/current/amd64/iso-dvd/)），大概步骤如下：

- 下载镜像和 `ventoy` 软件，先制作 `ventoy` 启动盘，制作完成后将镜像复制过去即可， `ventoy` 制作的启动盘还可以日常使用，不需再次格式化。

- 检查bios，关闭bios的快速启动和安全启动。如果硬盘内有windows系统，先进入windows系统，关闭快速启动（右击任务栏win图标，设置，系统，页面左侧的“电源与睡眠”，右侧下方“其他电源设置”，更改电源按钮，页面上方“更改当前不可用的设置”，取消勾选“快速启动”）

---

### 2.安装
#### 2.1 从U盘启动

设置电脑从U启动，启动后看到ventoy界面，可以看到复制进去的安装镜像，选中回车即可加载

#### 2.2 安装流程

- 启动debian安装镜像后，选择 `Advanced options`，进去后选 `Expert insyall`，此时加载的界面是tui界面，而不是gui，避免了因显卡驱动带来的启动失败问题

- 按照顺序来慢慢设置，以下列出需要设置的选项

    - `Choose language`：选择语言，选英语，不然后面tty会出现方块字；地区选 `other` ，里面有 `China`，语系选 `United States - en_US.UTF-8` ，  `Additonal locales` 这里直接回车，中文语系等安装完后再去配置
    
    - 直接到 `Configure keyboard` ，选 `American English`
    
    - 之后就是探测安装介质，然后加载安装组件，进去后全部选择（用空格选）
    
    - 如果你有有线网络并且通畅（指能高速访问debian的官方镜像地址），并且网卡不是很新，可以选择配置网络，否则跳过，跳过的步骤有 `Detect network hardware` ， `Configure and start a PPPoE connection` ， `Continue installation remotely using SSH` ， `Choose a mirror of Debian archive`
    
    - 设置用户名和密码。启用shadow，允许root登录，并设置一个默认的非root用户。
    
    - 跳过内存释放，如果完全没有网络，则跳过 `Configure the clock` 
    
    - 检测磁盘并分区，选择手动分区。这里需要注意，如果电脑中有windows，则新建分区中不需要建立efi分区和boot分区（现在boot都不默认分出来了，意义不大）；如果有其他linux系统，如果有efi分区则不需额外建立，没有则建立（没有efi分区你的那个linux是咋启动的？？），原来的linux如果配置了swap，**分区时将该分区该成“不使用”**，否则你的另外一个linux的swap分区默认会被格式化，UUID发生变换，下次启动就会卡很长时间；如果有多个硬盘，并打算一个硬盘一个系统，则推荐每块硬盘建立一个efi分区（当然不建立也可以）。
    - 分区完毕后，检查无误，并将更改写入硬盘。选择安装基本系统，并完成。选择安装内核时默认即可；选择安装驱动时，要包括所有驱动。
    
    - 配置包管理器，不要扫描额外的安装介质，并不使用网络安装；取消安全更新（`security updates`）和 `release updates` ，回车下一步
    
    - 选择安装软件，选择不自动更新和不参加包使用调查后，便可以进入软件安装界面，如果要安装gnome，则最好选择 `Debian desktop environment` 和 `GNOME`，如果安装kde，则可以不用选择 `Debian desktop environment`，其中 `standard system utilities`，选择好后回车安装
    
    - 安装grub，直接安装即可。可以不用将grub安装在 `EFI removable media path`（也没必要，除非你的bios很烂很老）。这个阶段会自动查找已安装的操作系统，并建立grub启动条目。
    
    - 完成安装，直接下一步，重启。待系统关机后立马拔U盘，然后就可以启动debian。
    

### 3.相关系统配置
#### 3.1 NVIDIA显卡驱动及其他

如果你电脑有n卡，则需要安装闭源驱动，开源的 `nouveau` 有很多问题且容易崩溃。此操作最好不要加载图形界面，进入tty操作。

- 重启到grub，用键盘打断启动倒数，选择debian的启动条目，按e编辑，在 `linux /boot/vmlinuz....` 最后加入 `nomodeset` 来禁用显卡驱动加载，按 `Ctrl+x` 
    
- 先连接到网络，配置好镜像源后，更新软件。
    
- root权限运行 `apt install nvidia-driver`，安装N卡闭源驱动
    
- 然后安装其他驱动和固件（root权限）： `apt install firmware-linux`，根据电脑实际硬件配置安装相应固件。这里列出一些常用的驱动和固件软件名称
      
  - amd显卡驱动固件： `firmware-amd-graphics`
      
  - 博通网卡固件： `firmware-brcm80211`
      
  - intel网卡驱动： `firmware-iwlwifi`
      
  - linux固件和非自由固件： `firmware-linux`,`firmware-linux-free`,`firmware-linux-nonfree`,`firmware-misc-nonfree`
      
  - 高通相关： `firware-qcom-soc`
      
  - realtek网卡固件： `firmware-realtek`
      
- 安装完成后重启，nvidia显卡驱动就可以正常加载了

如果在执行`update-initramfs`这类操作时，安装了源里的相应固件软件包，但还是提示`possible missing firmware ......`时，可以前往清华镜像下载固件，相关地址在[这里](https://mirrors.tuna.tsinghua.edu.cn/help/linux-firmware.git/)，下载好后根据报错的信息，把缺少的固件复制到相应地方即可（~~实际上这里面的固件也不完全全，还有一部分在相关硬件厂商的官网上...~~），如果硬件工作正常，则可以不用此操作

#### 3.2 笔记本用户的配置
##### 3.2.1 电源相关

对于笔记本用户，推荐使用`tlp`来控制硬件和相关调度，相关配置可以参考我的（配置文件`/etc/tlp.conf`）：
```c
......
# ------------------------------------------------------------------------------
# tlp - Parameters for power saving

# Set to 0 to disable, 1 to enable TLP.
# Default: 1

# 默认开启tlp
TLP_ENABLE=1

# Select a CPU frequency scaling governor.
# Intel processor with intel_pstate driver:
#   performance, powersave(*).
# Intel processor with intel_cpufreq driver (aka intel_pstate passive mode):
#   conservative, ondemand, userspace, powersave, performance, schedutil(*).
# Intel and other processor brands with acpi-cpufreq driver:
#   conservative, ondemand(*), userspace, powersave, performance, schedutil(*).
# Use tlp-stat -p to show the active driver and available governors.
# Important:
#   Governors marked (*) above are power efficient for *almost all* workloads
#   and therefore kernel and most distributions have chosen them as defaults.
#   You should have done your research about advantages/disadvantages *before*
#   changing the governor.
# Default: <none>

CPU_SCALING_GOVERNOR_ON_AC=schedutil
CPU_SCALING_GOVERNOR_ON_BAT=powersave
# cpu的调度，省电建议 powersave，供电时建议schedutil或者ondemand

......

# Set the CPU "turbo boost" (Intel) or "turbo core" (AMD) feature:
#   0=disable, 1=allow.
# Note: a value of 1 does *not* activate boosting, it just allows it.
# Default: <none>

CPU_BOOST_ON_AC=1
CPU_BOOST_ON_BAT=1
# cpu的睿频，建议在电池和充电时均开启，不然用电池时体验不会很好

# Minimize number of used CPU cores/hyper-threads under light load conditions:
#   0=disable, 1=enable.
# Default: 0 (AC), 1 (BAT)

SCHED_POWERSAVE_ON_AC=0
SCHED_POWERSAVE_ON_BAT=1
# 省电相关，尽量减少轻负载下的cpu使用


# Define disk devices on which the following DISK/AHCI_RUNTIME parameters act.
# Separate multiple devices with spaces.
# Devices can be specified by disk ID also (lookup with: tlp diskid).
# Default: "nvme0n1 sda"

DISK_DEVICES=""
# 这里的默认值有两个，如果有机械硬盘的话可以填相关设备路径，是固态的话建议留空

......

# Runtime Power Management for PCIe bus devices: on=disable, auto=enable.
# Default: on (AC), auto (BAT)

RUNTIME_PM_ON_AC=on
RUNTIME_PM_ON_BAT=auto
# 运行时的pcie设备电源控制，当供电时建议on，用电池时auto即可

......

STOP_CHARGE_THRESH_BAT0=1
# 一部分笔记本支持在充电时只用充电器供电，从而减少电池的充电循环次数，延长电池使用寿命，可以通过设置这类参数来控制充电阈值，比如我的联想R7000 2020支持这个充电参数。可以通过命令 tlp-stat -b 查看电池充电状态，以及具体的设置参数。
```
理论上默认的`tlp`配置已经足够好了，如果有需求，可以自行根据wiki去修改

##### 3.2.2 音频和蓝牙

目前`linux`上的音频后端有`pulseaudio`和`pipewire`，前者有比较久的历史，兼容旧硬件；后者是新一代的linux音频后端，支持更多的特性以及更好的音质和延迟 （比如蓝牙的延迟和音质），个人更推荐使用`pipewire`作为音频服务，完全取代`pulseaudio`。具体配置可以参考`debian wiki`上的[这篇文章](https://wiki.debian.org/PipeWire)。

安装pipewire，并取代pulseaudio
```sh
sudo apt install pipewire pipewire-pulse pipewire-audio-client-libraries 

//蓝牙相关
sudo apt install libspa-0.2-bluetooth
```

以普通用户身份运行
```sh
// Check for new service files with:
systemctl --user daemon-reload
// Disable and stop the PulseAudio service with:
systemctl --user --now disable pulseaudio.service pulseaudio.socket
// Enable and start the new pipewire-pulse service with:
systemctl --user --now enable pipewire pipewire-pulse
```
安装配置切换管理，个人推荐`wireplumber`，功能更强大。如果你是`debian`稳定版的用户，则目前不推荐使用（说实话，对于新的硬件（3年内），更推荐使用`debian sid`，而不是stable，因为很多硬件驱动和固件，稳定版里都没有，稳定版更适合作服务器使用而不是日常使用）
```sh
sudo apt install wireplumber pipewire-media-session-
```
目前`debian sid`上的`pipewire`仍然有问题，比如会出现重新登录后音量大小被重置为74%，这种情况下，可以尝试把用户从`audio`用户组里移除，或者换成`pipewire-media-session`（这两个方法目前来说对我是有用的），或者干脆不使用`pipewire`作为后端音频服务。从作者的时间来看，目前这个bug貌似已经修复了。

在使用`pipewire`作为音频后端时，手机连接电脑的蓝牙，可以使用`aptx`等这类高音质的协议通信，并且延迟相比`pulseaudio`要低。

如果发现蓝牙无法保持关机时的开关状态，尝试关闭和`rfkill`相关的`systemd`服务



#### 3.4 中文输入法

输入法个人首选`rime`，输入法框架首选`fcitx5`，直接安装即可
```sh
sudo apt install --install-recommends fcitx5 fcitx5-chinese-addons
```

如果是kde用户，可以只安装这些
```sh
sudo apt install --no-install-recommends fcitx5 fcitx5-chinese-addons fcitx5-frontend-gtk3 fcitx5-frontend-qt5 fcitx5-module-xorg kde-config-fcitx5 im-config
```

Debian 使用`im-config`来自动配置输入法需要的环境变量。注意一定要安装这个软件包。 

遇到的问题：如果你的系统语系是非中文语系，则无论怎么配置，都无法在一些gtk应用中输入中文（比如emacs），这是debian上的fcitx5的问题（版本不够新，无法指定使用`fcitx5`来代替`fcitx`等），对于此问题的解决方法，将`fcitx5`换成`fcitx`即可。

关于rime输入法的配置，在此不多叙述

#### 3.5 FireFox 视频硬解（硬件加速）

具体配置可以参考Debian Wiki上的[这篇教程](https://wiki.debian.org/Firefox#Hardware_Video_Acceleration)

个人推荐：
```c
media.ffmpeg.vaapi.enabled=true
media.ffvpx.enabled=false
```
如果你的硬件可以硬解av1编码的视频，则可以不用调整以下设置，否则建议关闭av1的解码（目前能硬解av1的硬件不多，比如amd的rx 6000系列显卡，老黄的rtx 3000系列显卡，以及联发科的天玑8100等，否则开启这个后也会强行回到cpu软解上，不仅效果不好，而且还吃资源耗电），但是，如果视频网站只提供av1编码的视频，就会完全没有画面。
```c
media.av1.enabled=false
```

