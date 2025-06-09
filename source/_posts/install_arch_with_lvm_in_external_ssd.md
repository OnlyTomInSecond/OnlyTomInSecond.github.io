---
title: 记一次在移动硬盘上安装 arch + lvm
tags:
  - Linux
---

手头多余一块硬盘，加上刚买移动硬盘盒，于是想着拿来做一个类似 `WinToGO` 的 `ArchToGo` 设备，同时配置 `LVM` 和 `LUKS` 加密（其中 `LVM` 用于动态调整分区， `LUKS` 用于数据备份分区的加密）。本文主要记录操作时的注意事项。

### 1. 有关安装

实际上 arch wiki 上有关教程较为详细，对于加密，我采用的方案为 `LUKS over LVM`，这样一来可以做到只加密 `LVM` 中的某个逻辑卷。分区安排采用如下

| 分区      | 挂载点    | 文件系统 |
|:---------:|:---------:|:--------:|
| /dev/sda1 | /boot/efi | vfat     |
| /dev/sda2 | swap      |          |
| /dev/sda3 | /         | btrfs    |
| /dev/sda4 |           | LVM      |

其中 `/dev/sda4` 使用 `LVM`。新建一个 `LVM` 组，里面包含两个逻辑分区，分别是 `home` 和 `backup` ， `backup` 用于存储数据备份，并且使用 `LUKS` 加密。相关命令

创建分区
```sh
sudo fdisk /dev/sda
```

默认采用使用 `GUID` 分区，分别创建 `efi`，`swap`，`/`分区，并设置好分区类型，创建 `LVM` 时，将分区类型设置为 `Linux LVM` 。

创建物理卷 `physical volumes`
```sh
sudo pvcreate /dev/sda4
# 在 /dev/sda4 上创建 LVM
```

创建逻辑卷组 `volume group`
```sh
sudo vgcreate data /dev/sda4
# 在 /dev/sda4 上创建名为 data 的逻辑卷组，如果需要需要将其他物理卷/分区（比如/dev/sda5等），使用 sudo vgextend data /dev/sda5 
```

创建逻辑卷 `logical volume`
```sh
sudo lvcreate -L 60G data -n home
sudo lvcreate -l 100%FREE data -n backup
# 创建 60G 的 home 卷，其余空间全部分配给 backup 卷
```

格式化卷
`LVM` 相当于抽象的 `/dev/sdXY` 的设备，上述操作可以看作在进行分区，因此想要正常使用，需要将 “分区” 格式化。由于 `LVM` 可使用快照功能，且卷组内的逻辑卷也具有 `copy on write(COW)` 的功能，因此可不必使用自带快照的文件系统（比如`btrfs`和`zfs`文件系统）。 `home`卷不配置加密，因此可以直接先格式化
先激活现有的 LVM 卷
```sh
sudo modprobe dm_mod
sudo vgscan
sudo vgchange -ay
```
格式化 `home` 卷
```sh
sudo mkfs.ext4 /dev/data/home
# LVM 激活后会在 /dev/ 下生成 逻辑卷组名/逻辑卷 的路径，格式化时填映射后的路径而不是 /dev/sdaX 这类块设备路径
```
修改 `mkinitcpio hooks`
修改 `HOOKS` 变量，如果使用的是 `udev`，则
```c
/etc/mkinitcpio.conf
---
HOOKS=(base udev ... block lvm2 filesystems)
# 添加了 lvm2 
```

使用 `systemd`，则
```c
/etc/mkinitcpio.conf
---
HOOKS=(base systemd ... block lvm2 filesystems)
```

之后执行 `sudo mkinitcpio -P` 更新 `initramfs` 即可。对于新安装的 arch ，记得修改 `/etc/fstab` 里的挂载，同样使用 **逻辑卷** 的 `UUID` 即可。

加密的 `backup` 卷
上述步骤中仅创建了 `backup` 逻辑卷，并未进行格式化，因为需要先设置好 `LUKS` 后再格式化（这也许说明 `LUKS` 不支持在线加密，因为格式化在设置 `LUKS` 之后。至于 `BitLocker`，很可能是在安装windows时就设置好了加密，只不过完全没有密码而已）。
将 `backup` 分区缩小 `256M`
```
$ sudo -L -256M data/backup
# 由于采用 ext4 分区，需要最后一个 ext4 分区预留至少 256M 的空间来使用 e2scrub
```

生成加密密码文件
```
$ mkdir ~/luks-keys
$ dd if=/dev/random of=~/luks-keys/backup bs=1 count=256 status=progress
```

设置 `LUKS` 
```
$ sudo cryptsetup luksFormat -v /dev/data/backup ~/luks-keys/backup
# 使用生成的密码文件设置并加密 backup 逻辑卷
$ sudo cryptsetup -d ~/luks-keys/backup open /dev/data/backup backup
# 解密 backup 逻辑卷，只有解密后才可进行读写操作，否则将毁坏加密和逻辑卷
$ sudo mkfs.ext4 /dev/mapper/backup
# 格式化解密后的 backup 为 ext4，在 /dev/mapper/ 内可能会有 data-backup 这种 逻辑卷组-逻辑卷 的设备文件，但是格式化时千万不可选择，否则可能毁坏 LVM 逻辑卷！
```

格式化后便可以用 `mount` 进行挂载了，由于不需要开机挂载，因此没有设置开机挂载相关的 `HOOKS` 和 `/etc/fstab` ，若有需求，可参考 arch wiki 的[这篇文章](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Configuring_mkinitcpio_3)

貌似都设置好了，但是移动硬盘不会一直插在电脑上，因此需要安全移除硬盘， `LVM` 卷并不能在 `umount` 后直接移除，会有导致坏块丢数据的风险。按照下列操作来安全移除 `LVM` 设备，在 `umount` 挂载点后，

```
$ sudo cryptsetup luksClose /dev/mapper/backup
$ sudo cryptsetup luksClose /dev/mapper/data-backup
# 先关闭所有已解密的逻辑卷，再关闭逻辑卷组上的解密访问
$ sudo vgchange -an
# 关闭 LVM 映射
$ sudo udisksctl power-off -b /dev/sda
# 使用 udiskctl 关闭移动硬盘的电源
```

执行成功后即可安全移除移动硬盘

相关参考

[Arch wiki, Install Arch Linux on LVM](https://wiki.archlinux.org/title/Install_Arch_Linux_on_LVM)
[Arch wiki, dm-crypt/Encrypting an entire system](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system)
