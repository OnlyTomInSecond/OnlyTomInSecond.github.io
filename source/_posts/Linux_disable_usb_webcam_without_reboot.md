---
title: Linux 禁用usb设备及摄像头（无需重启）
tags:
  - Linux
---

最近有禁用摄像头的需求，到网上找了众多教程，发现都是通过禁用驱动来实现的，但是这需要重启设备，对于正在运行的服务来说很不友好，于是找到一篇可以不重启而禁用摄像头的方案

原教程 [地址](https://karlcode.owtelse.com/blog/2017/01/09/disabling-usb-ports-on-linux/#:~:text=Disabling%20the%20Webcam%20or%20USB%20Ports%20on%20Linux,want%20to%20disable%20this%20port%20at%20start-up.%20)

总的来说，就是将系统现有的usb设备和系统的bus解绑

### 找到需要禁用的usb设备的BusID ###

```
for device in $(ls /sys/bus/usb/devices/*/product); do echo $device;cat $device;done
```

我的显示结果如下:
```
/sys/bus/usb/devices/1-6/product
XiaoMi USB 2.0 Webcam
/sys/bus/usb/devices/1-7/product
USB2.0-CRW
/sys/bus/usb/devices/1-8/product
ELAN:Fingerprint
/sys/bus/usb/devices/1-9/product
USB OPTICAL MOUSE 
/sys/bus/usb/devices/usb1/product
xHCI Host Controller
/sys/bus/usb/devices/usb2/product
xHCI Host Controller
```

可以看到，我的网络摄像头对应的编号是`1-6`

## 禁用设备（解绑） ##
```
echo '1-6' | sudo tee /sys/bus/usb/drivers/usb/unbind
```

将id换成对应的id就可以

## 启用设备（重新绑定） ##
```
echo '1-6' | sudo tee /sys/bus/usb/drivers/usb/bind
```

将id换成对应的id就可以

## 开机设定（使用crontab) ##
```
sudo crontab -e
```

提示使用文本编辑器，改为自己熟悉的即可

**添加**
```
@reboot echo '1-6' > /sys/bus/usb/drivers/usb/unbind
```

保存退出即可