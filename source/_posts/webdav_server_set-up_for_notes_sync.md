---
title: 搭建个人webdav服务器同步
tags:
  - Linux
---

## 起因 ##

鉴于上次刷机未备份自己在手机里写的笔记，于是萌生了给手机笔记同步的想法。一开始我是使用便签这个软件写笔记，后来测试笔记同步时发现有问题，无奈之下将笔记软件换成了纯纯写作，同步问题才解决。由于没有onedrive和坚果云（没错就是我懒），我的笔记全部同步到了局域网服务器内。

## 准备 ##

-  纯纯写作或者其他支持webdav同步的笔记软件（便签我真不确定能不能同步，有兴趣的可以试试）

-  一台服务器（废话）

## 开始 ##

1. 服务器配置（默认服务器系统为Debian系列的Linux系统）

   - 首先ssh连接上服务器，输入命令 ` sudo apt install nginx-full nginx-extras` （没法sudo就把切换用户到root），安装Nginx软件包。

   -  修改 ``` /etc/nginx/sites-available/default``` 修改Nginx默认配置文件。在server的大花括号内加入（这个配置我也是从网上copy的）：

```nginx
location /webdav {
        autoindex on;
        autoindex_localtime on;
        alias /notes/; #放笔记的目录，自己改

        set $dest $http_destination;
        if (-d $request_filename) {                   # 对目录请求、对URI自动添加“/”
            rewrite ^(.*[^/])$ $1/;
            set $dest $dest/;
        }

        if ($request_method ~ (MOVE|COPY)) { # 对MOVE|COPY方法强制添加Destination请求头
            more_set_input_headers 'Destination: $dest';
        }

        if ($request_method ~ MKCOL) {
            rewrite ^(.*[^/])$ $1/ break;
        }

        dav_methods PUT DELETE MKCOL COPY MOVE;      # DAV支持的请求方法
        dav_ext_methods PROPFIND OPTIONS LOCK UNLOCK;# DAV扩展支持的请求方法
        create_full_put_path  on;                    # 启用创建目录支持
        dav_access user:rw group:r all:r;            # 设置创建的文件及目录的访问权限

        auth_basic "Authorized Users WebDAV";
        auth_basic_user_file /home/users/.htpasswd;  # 验证文件的位置
    }

```

用户和密码的认证文件可以用`htpasswd`这个工具生成，只需安装`apache2-utils`
```shell
$ sudo apt install apache2-utils
```
生成文件
```shell
$ htpasswd -c 目标文件 用户名
```

如果想配置Nginx端口还有SSL加密什么的，就自己去网上找教程，一大堆呢

- 配置权限

如果出现`403_Forbidden`，那大概率你的目录权限没有配置对（还有就是nginx组件没安装完全）。

权限设置建议使用`ACL`来控制，而不是无脑`chmod 0755`。相关命令：
```shell
$ sudo setfacl -R -m u:www-data:rwx directory（目录）
```

如果启动nginx服务的不是www-data用户，自己用systemctl看启动的用户，命令里改成相应的用户即可。改完最好重启一下nginx服务。


- 配置防火墙规则（没开防火墙的请忽略本步骤），默认使用iptables，用ufw的可以去翻官方wiki。
规则写入
```
-A INPUT -p tcp --dport 你的端口号（默认填80） -j ACCEPT
```
重启防火墙服务（一般是`netfilter-persistance.service`）

或者命令
```shell
$ sudo iptables -A INPUT -p tcp --dport 你的端口号（默认填80） -j ACCEPT
```
执行立马生效。如要永久生效，可以参考Debian wiki [iptables配置](https://wiki.debian.org/it/iptables)
