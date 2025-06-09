---
title: linux kvm里win10和宿主机文件传输方法
---
## linux kvm里win10和宿主机文件传输方法

- 把windows系统里面的共享文件设置为共享
- 在linux系统里面 

` mount -t cifs //ip/folder mount_point -o username=user,password=passwd `

改一下相关参数即可。

如果挂载失败，可以试试用` sudo `来运行此条命令

完成后可以直接在文件夹中访问共享文件

***注意，如果无法挂载cifs文件系统，请检查相关软件或者驱动是否安装并设置完好***