---
title: 指定git使用特定的密钥
tags:
  - Git
---

众所周知，ssh远程登录时可以使用参数 `-i identfyfile` 来指定特定的密钥来进行登录，但是在我学习git中，发现单纯使用`git push -u`是默认使用`ssh-keygen`产生的密钥文件名来进行登录验证，显然这对与有多个密钥的用户来说实在不好友好，于是在网上找到[这篇教程](https://www.cnblogs.com/chenkeyu/p/10440798.html)

## 最佳解决方案 ##

在`~/.ssh/config`中(没有这个文件可以自己新建)，添加：

```
host github.com
HostName github.com
IdentityFile ~/.ssh/id_rsa_github
User git
```

就ok了，不过密钥的文件名就要自己改了。同时注意设置密钥的权限。