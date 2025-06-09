---
title: 在手机上实现从刷机包中提取内容
--- 

## 本教程基于 [LineageOS](https://wiki.lineageos.org/extracting_blobs_from_zips) 的这篇教程 ##

LOS的那一篇教程需要在电脑上运行， 但看了一下主要用到的工具， 发现大多是python脚本， 而在android上跑python脚本十分简单， 用termux就行， 于是想着看能不能那篇教程运用到手机上来。

### 1. 软件准备 ###

要安装 `brotli`和 `git`
```sh
apt install brotli git
```
安装 `protobuf`， termux 中并没有这个包，但python里有，直接用pip装就行; 还需要装个依赖 `six`
```sh
pip install protobuf six
```
git下载LOS的解包脚本
```sh
git clone https://github.com/LineageOS/android_tools_extract-utils android/tools/extract-utils
git clone https://github.com/LineageOS/android_system_update_engine android/system/update_engine
```
解压整个包到当前目录，不推荐直接跑，在手机上这个要跑比较久。
```sh
python3 android/tools/extract-utils/extract_ota.py pack.zip
```
解压特定分区到特定目录，比如解压包里的boot分区到out文件夹里
```sh
python3 android/tools/extract-utils/extract_ota.py -p boot -o out/
```

解压后得到的是可以直接挂载的镜像， 如果要在手机上挂载这些镜像， 需要root权限才能实现。对于没有root权限的手机， 可以使用proot来实现挂载
