---
title: sddm屏幕亮度问题
tags:
  - Linux
---

之前一直遇到刚开机，未登录时，sddm的屏幕亮度总是比较低的情况，而且登录进桌面后，屏幕亮度恢复正常。一直上网搜原因，但都没找到。最近在重新看arch wiki里关于[sddm的配置](https://wiki.archlinux.org/title/SDDM#Match_Plasma_display_configuration) 时，注意到 `/var/lib/sddm` 中的内容，可以将plasma的配置同步到sddm上。wiki中这样写：

```
To enable correct display & monitor handling in SDDM (scaling, monitor resolution, hz,...), you can copy or modify the appropriate configuration file from your home directory to the one of SDDM:
```

```shell
# cp ~/.config/kwinoutputconfig.json /var/lib/sddm/.config/
# chown sddm:sddm /var/lib/sddm/.config/kwinoutputconfig.json
```

```
To also enable correct input handling in SDDM (tap-to-click, touchscreen mapping,...), you can copy the appropriate configuration file from your home directory to the one of SDDM:
```

```shell
# cp ~/.config/kcminputrc /var/lib/sddm/.config/
# chown sddm:sddm /var/lib/sddm/.config/kcminputrc
```

于是想到会不会是因为之前的sddm运行时配置有问题，导致亮度不对呢？于是删除 `/var/lib/sddm` ，设置中重新设置sddm样式，并执行上述的4条命令。重启后亮度便和plasma的一致，至此问题得到解决