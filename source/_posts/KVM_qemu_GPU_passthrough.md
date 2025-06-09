---
title: KVM + qemu 显卡直通(passthrough)配置
tags:
  - Linux
---

### 这个是啥？ 

说白了，就是在linux系统下，使kvm虚拟机直接使用主机的PCI设备（包括显卡），且性能损失较小。这其中的技术原理简单来说就是将主机上的PCI设备通过内核模块直接映射至虚拟机上。

说白了我就是想在主机上虚拟windows然后用显卡玩游戏

### 开始

- 启用IOMMU

首先确定主板的有关虚拟化的功能是否开启

修改grub配置，添加`intel_iommu=on`或者`amd_iommu=on`，根据自己的cpu型号来改。

``` 
sudo vim /etc/default/grub 
```

在`GRUB_CMDLINE_LINUX_DEFAULT`变量的等号后面添加`intel_iommu=on`或者`amd_iommu=on`

改完更新grub，`sudo update-grub` 

- 将需要直通的GPU与host隔离

- 查看需要直通的GPU的id

运行命令`lspci -nn`，找到显卡的相关条目，记住显卡的id号码，像这样
```
01:00.0 3D controller [0302]: NVIDIA Corporation GP108BM [GeForce MX250] [10de:1d52] (rev ff)
```

其中`10de:1d52`为显卡的id号码，如果是nvidia的显卡，型号带Ti尾缀的就还有音频设备，记得两者都要记录下来。

- 配置`vfio.conf` 

- 首先安装ovmf`sudo apt install ovmf`

- 创建文件`/etc/modprobe.d/vfio.conf`,编辑加入内容
```
options vfio-pci ids=10de:1d52
```
其中硬件id根据自己的显卡的id来更改

- 配置vfio模块的加载

确保vfio-pci在其它图形驱动之前加载，修改 `/etc/initramfs-tools/modules`， 按照顺序将`vfio_pci`，`vfio`，`vfio_iommu_type1`，`vfio_virqfd` 的顺序添加到`modules`文件的末尾，如下所示

```
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
vhost-net
```
改完后运行`sudo update-initramfs -u`,使更改生效

然后重启电脑。

- 检验是否开启成功

- 检查vfio模块是否正常加载

运行`lsmod | grep vfio`有输出说明模块已经加载

- 检查IOMMU

运行`sudo dmesg | grep IOMMU`.结果大概如下即可

```
[    0.053226] DMAR: IOMMU enabled
[    0.096607] DMAR-IR: IOAPIC id 2 under DRHD base  0xfed91000 IOMMU 1
[    1.978717] AMD-Vi: AMD IOMMUv2 driver by Joerg Roedel <jroedel@suse.de>
[    1.978718] AMD-Vi: AMD IOMMUv2 functionality not available on this system
```
*由于我是intel的cpu，amd的IOMMU我当然用不了啦*
- 为虚拟机分配显卡
- 打开virt-manager ，选择其中一台虚拟机，然后添加pci设备，在添加的菜单中应该有显卡的选项。
- 启动虚拟机，检查设备管理器，看是否有显卡设备。
- 安装好驱动就可以用显卡了。
- 关于笔记本电脑中的bumblebee显卡自动切换
由于bumblebee将显卡彻底关闭，以达到省电的目的。所以在上述操作后，在添加pci设备的菜单中看不到显卡。解决方法就是手动将显卡开启。

- 关闭bumblebee，并以root身份运行`echo ON >> /proc/acpi/bbswitch`来开启显卡，完成后重启虚拟机服务`libvirtd.service` 。
- github上有一篇关于这个的配置教程，可以去看看[optimus laptop dGPU passthrough ](https://gist.github.com/Misairu-G/616f7b2756c488148b7309addc940b28)

- 遇到的问题

没错，安装显卡驱动的时候windows设备管理器显示错误43

网上一查发现竟然是nvidia的锅：不允许低端显卡进行虚拟化，发现是低端显卡，驱动就报错43！