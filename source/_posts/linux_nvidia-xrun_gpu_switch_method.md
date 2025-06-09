---
title: "linux nvidia-xrun : 高性能显卡切换方案"
tags:
  - Linux
---

由于之前的小米笔记本被我换成了联想的R7000,原来运用与I+N组合（intel cpu和nvidia显卡）的方案放在AN组合下失效了，bumblebee无法正确运行，同时bbswitch模块也无法将显卡正确关闭，于是寻找新的显卡切换方案，找到了nvidia-xrun这个东西。下面来简单说一下配置教程。本教程基于debian，其他发行版可以参考

## 注意：由于5.9-5.10之间的内核中nvidia的驱动程序所使用的软件协议变更，导致内核拒绝加载nvidia的内核驱动（错误信息是协议不符合） 如果想要其加载，可以自己编译内核并打上相应补丁 ##

- ## 1.安装好显卡的闭源驱动 并卸载bumblebee ##

这个教程网上很多，由于我的发行版是Debain，于是按照Debian wiki上的教程来安装驱动

[Debian wiki](https://wiki.debian.org/NvidiaGraphicsDrivers#Configuration)

安装驱动
```sh
sudo apt install nvidia-driver
```

如果之前安装了`bumblebee`和`bbswitch`，卸载之

```
sudo apt purge bumblebee* bbswitch* && sudo apt autopurge 
```

- ## 2.配置nvidia-xrun ##

[Debian wiki about nvidia-xrun](https://wiki.debian.org/NvidiaGraphicsDrivers/NvidiaXrun)

先下载`nvidia-xrun`，如果是arch linux及其相关系列的发行版，在aur源里应该有`nvidia-xrun`相关的软件，直接一键安装就可以，相关操作可以查看[Arch Wiki](https://wiki.archlinux.org/index.php/Nvidia-xrun)，但是在debian上没有，所以直接去github上下载

```sh
git clone https://github.com/Witko/nvidia-xrun.git
```

下载后是一个文件夹，里面有如下文件（有些需要自己创建）和对应目录关系

```
/usr/bin/nvidia-xrun - the executable script  
------------------即nvidia-xrun/nvidia-xrun
/etc/X11/nvidia-xorg.conf - the main X confing file 
------------------即nvidia-xrun/nvidia-xorg.conf
/etc/X11/xinit/nvidia-xinitrc - xinitrc config file. Contains the setting of provider output source 
------------------即nvidia-xrun/nvidia-xinitrc
/etc/X11/xinit/nvidia-xinitrc.d - custom xinitrc scripts directory 
------------------这个目录需要自己创建
/etc/X11/nvidia-xorg.conf.d - custom X config directory
/etc/systemd/system/nvidia-xrun-pm.service systemd service 
------------------开机关闭显卡的一个服务，复制完后执行 systemctl daemon-reload 和 systemctl enable nvidia-xrun-pm.service 来启用服务
/etc/default/nvidia-xrun - nvidia-xrun config file 
------------------即nvidia-xrun/config/nvidia-xrun，这个一定不要忘记了，要不然会无法关闭和打开显卡。（这个文件是nvidia-xrun.service 这个服务和nvidia-xrun 这个命令的配置文件）
/usr/share/xsessions/nvidia-xrun-openbox.desktop - xsession file for openbox 
------------------这个可以不要
/usr/share/xsessions/nvidia-xrun-plasma.desktop - xsession file for plasma 
------------------这个也可以不要
[OPTIONAL] $XDG_CONFIG_HOME/X11/nvidia-xinitrc - user-level custom xinit script file. You can put here your favourite window manager for example 
------------------这个也可以不要
```
上面的文件说明意思是要把下载的文件复制到相应的目录里，比如`nvidia-xrun`这个可执行脚本应复制到`/usr/bin/`或者其他在`PATH`变量的路径里。按照文件的对应关系把文件复制到相应位置。

### 关于`/etc/default/nvidia-xrun` ###

要配置好显卡的pci  bus id，修改方法参照下面。

- 2.2 修改bus id

通过命令`lspci | grep -i nvidia | awk '{print $1}'`查看显卡的bus id，如果`nvidia-xorg.conf`的bus id和命令显示的大致相同（只有数字0的差，表现形式不同）则不用改。
```
Section "Device"
  Identifier "nvidia"
  Driver "nvidia"
  BusID "PCI:1:0:0"  这里的pci id要改为相应的id，格式参照这个例子（比如lspci 中显示的是 06:00.0 ，这里就修改为 6:0:0
EndSection

```

- 2.3 修改`nvidia-xorg.conf`模块搜索路径

修改模块搜索路径（仅限Debian，其他发行版参考相应路径）

```
ModulePath "/usr/lib/nvidia/current"
ModulePath "/usr/lib/x86_64-linux-gnu/nvidia"
ModulePath "/usr/lib/x86_64-linux-gnu/nvidia/xorg"
ModulePath "/usr/lib/x86_64-linux-gnu/nvidia/xorg/modules"
ModulePath "/usr/lib/xorg"
ModulePath "/usr/lib/xorg/modules"
```

在`nvidia-xinitrc` 文件中，替换这一行

```sh
export LD_LIBRARY_PATH=/usr/lib/x86_64-linux-gnu/nvidia:/usr/lib:${LD_LIBRARY_PATH}
```

到这一步，`nvidia-xrun`所需要的最基本的环境准备好了，可以重启电脑。重启完后用查看`nvidia-xrun-pm.service`这个服务是否成功执行，若无，则检查其配置文件是否正确。成功则下一步

- 3.0 使用`nvidia-xrun`运行程序

切换到tty环境，输入账号密码登陆后，尝试启动另一个桌面环境（这里一openbox为例）

```shell
nvidia-xrun openbox-session
```
这期间会要输入sudo的密码，输入后`nvidia-xrun`会打开显卡，读取`nvidia-xorg.conf` ，并运行程序。默认的openbox运行后会有一个鼠标可以移动，可以右击鼠标看到鼠标的菜单。若失败，参照下列的`TroubleShooting`


## TroubleShooting ##

- 1 按要求配置好并重启后，运行`nvidia-xrun`后tty里未出现图形界面，同时tty里报错

**Reasons**

出现这样的原因是因为`nvidia-xorg.conf`配置文件有问题

**Solution**

首先停止所有图形界面，切换至一个`tty`，以`root`身份登陆，并运行
```shell
Xorg --configure :1
```

在`/root`下会生成`xorg.conf.new`，内容类似这样
```
Section "ServerLayout"
	Identifier     "X.org Configured"
......
```

将开头的`ServerLayout`的`section`部分修改为这样

```
Section "ServerLayout"
	Identifier     "X.org Configured"
	Screen      1  "Screen0" 0 0 # there starts frome 1 to avoid conflict
#	Screen		1  "Screen1"
	Screen      2  "Screen1" RightOf "Screen0" # Avoid conflict 
#	Inactive	   "Screen0"
	InputDevice    "Mouse0" "CorePointer"
	InputDevice    "Keyboard0" "CoreKeyboard"
EndSection
```

核心是将`Screen`部分的参数修改，默认情况下标号为0的screen在你开机启用图形界面时就已被占据，所以修改为1和2

AMD核显部分

```
Section "Device"
        ### Available Driver options are:-
        ### Values: <i>: integer, <f>: float, <bool>: "True"/"False",
        ### <string>: "String", <freq>: "<f> Hz/kHz/MHz",
        ### <percent>: "<f>%"
        ### [arg]: arg optional
        #Option     "Accel"              	# [<bool>]
        #Option     "SWcursor"           	# [<bool>]
        #Option     "EnablePageFlip"     	# [<bool>]
        #Option     "SubPixelOrder"      	# [<str>]
        #Option     "ZaphodHeads"        	# <str>
        #Option     "AccelMethod"        	# <str>
        #Option     "DRI3"               	# [<bool>]
        #Option     "DRI"                	# <i>
        #Option     "ShadowPrimary"      	# [<bool>]
        #Option     "TearFree"           	# [<bool>]
        #Option     "DeleteUnusedDP12Displays" 	# [<bool>]
        #Option     "VariableRefresh"    	# [<bool>]
	Identifier  "Card0"
	Driver      "amdgpu"  # 驱动是amdgpu
	BusID       "PCI:6:0:0" # pci 的bus id要写对，这里是核显的
EndSection
```

nvidia部分

```
Section "Device"
        ### Available Driver options are:-
        ### Values: <i>: integer, <f>: float, <bool>: "True"/"False",
        ### <string>: "String", <freq>: "<f> Hz/kHz/MHz",
        ### <percent>: "<f>%"
        ### [arg]: arg optional
        #Option     "SWcursor"           	# [<bool>]
        #Option     "HWcursor"           	# [<bool>]
        #Option     "NoAccel"            	# [<bool>]
        #Option     "ShadowFB"           	# [<bool>]
        #Option     "VideoKey"           	# <i>
        #Option     "WrappedFB"          	# [<bool>]
        #Option     "GLXVBlank"          	# [<bool>]
        #Option     "ZaphodHeads"        	# <str>
        #Option     "PageFlip"           	# [<bool>]
        #Option     "SwapLimit"          	# <i>
        #Option     "AsyncUTSDFS"        	# [<bool>]
        #Option     "AccelMethod"        	# <str>
        #Option     "DRI"                	# <i>
	Identifier  "Card1"
	Driver      "nvidia" # Use the non-free nvidia driver 
	BusID       "PCI:1:0:0" # Set the right pci id of Gpu
EndSection
```

修改完成后将该文件替换原本的`nvidia-xorg.conf`便可。若再次失败则试试重启

- 2 关于5.9-5.10内核的nvidia驱动补丁

补丁内容如下，来源于nvidia 的开发者社区论坛，到5.11及以后的内核则无需此补丁

```diff
--- linux-source-5.10/kernel/module.c.old	2020-10-14 06:51:57.598066293 +0200
+++ linux-source-5.10/kernel/module.c	2020-10-14 07:58:16.504570606 +0200
@@ -1431,6 +1431,7 @@
 	return 0;
 }
 
+#if 0
 static bool inherit_taint(struct module *mod, struct module *owner)
 {
 	if (!owner || !test_bit(TAINT_PROPRIETARY_MODULE, &owner->taints))
@@ -1449,6 +1450,7 @@
 	}
 	return true;
 }
+#endif
 
 /* Resolve a symbol for this module.  I.e. if we find one, record usage. */
 static const struct kernel_symbol *resolve_symbol(struct module *mod,
@@ -1474,6 +1476,7 @@
 	if (!sym)
 		goto unlock;
 
+#if 0
 	if (license == GPL_ONLY)
 		mod->using_gplonly_symbols = true;
 
@@ -1481,6 +1484,7 @@
 		sym = NULL;
 		goto getname;
 	}
+#endif
 
 	if (!check_version(info, name, mod, crc)) {
 		sym = ERR_PTR(-EINVAL);


```

将内容保存为文本文件，再在源码目录下用`patch`命令打上，再编译内核并安装就可加载驱动

## 关于原理 ##

原理其实类似 `prime-offload`， 因为都是通过渲染一个虚拟的 `display`(配置文件中的 `Screen 1`) 和一个显示的 `display`(文件中的 `Screen 2`)， 由于输出都是走核显， 因此对于第二个 `display` 只是跑在独显上的一个 `xorg` 进程， 然后通过 `xrandr --setprovideroutputsource` 将核显输出来源设置成独显。 

## Reference ##

[Github](https://github.com/Witko/nvidia-xrun.git)

[How to use Nvidia-Xrun to unlock your Nvidia laptop's full potential on linux](https://www.devpy.me/nvidia-xrun/)

[Debian wiki](https://wiki.debian.org/NvidiaGraphicsDrivers/NvidiaXrun)

[Arch wiki](https://wiki.archlinux.org/index.php/Nvidia-xrun)
