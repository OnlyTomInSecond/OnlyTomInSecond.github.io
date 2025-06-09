---
title: wireguard组网——内网穿透及安全设置
tags:
  - Network
---

有内网穿透的需求，一开始想着用frp， 但又不想直接使用域名，加上日后有组网的需求， 于是便打算用wireguard来实现自己的组网需求。

### 1. 网络结构（网络拓扑） ###

有一台公网的服务器，公网ip假定为 `1.2.3.4`， 现创建两个局域网段， 分别为
```
192.168.20.0/24: 这个网段给服务器，实际上只用到了一个ip，就是192.168.20.1，实际上完全可以设置成192.168.20.1/32，只是用这个ip地址
192.168.21.0/24: 这个网段给一些固定的内网服务器， 比如家庭nas， 内网的下载机等，主要用来实现一些功能
192.168.22.0/24: 客户端网段， 用来使用上述服务，比如手机远程访问等
```

设置两个网段的是为了区分服务的同时，也解决路由问题（同网段不同设备之间，如果跨区域则需要广播找主机， 使得广播要跨区， 这是个人不希望的）。给服务器一个单独的网段也是为了好配置路由。

配置之前，先熟悉以下wg组网的大致原理

wg总体原理是点对点组网，每个节点都可以充当客户端/服务端，分别体现在wg配置的两个字段中：`interface` 相当于配置服务器的“虚拟ip”，而 `peer` 字段配置了能访问这个节点网络的（客户端）ip。wg基于udp协议实现，跨省传输有概率被阻断，如果出现此情况，建议使用基于tcp的内网穿透方案，比如frp

公网服务器端的配置：
```
Address = 192.168.23.1/32
ListenPort = 8844
PrivateKey = 此处填写用wg工具生成的私钥
MTU = 1400

# substitute eth0 in the following lines to match the Internet-facing interface
# if the server is behind a router and receives traffic via NAT, these iptables rules are not needed
# 下面的配置是配置防火墙，允许转发流量
PostUp = iptables -I FORWARD -i %i -j ACCEPT; iptables -I FORWARD -o %i -j ACCEPT; iptables -t nat -I POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
# foo
PublicKey = 公钥1
AllowedIPs = 192.168.21.0/24

[Peer]
# bar
PublicKey = 公钥2
AllowedIPs = 192.168.22.0/24
```

而其中一个客户端的配置如下：

```
[Interface]
Address = 192.168.22.3/32
MTU = 1400
PostUp = firewall-cmd --zone=home --change-interface=wg1
PreDown = firewall-cmd --zone=public --change-interface=wg1
PrivateKey = 私钥2

[Peer]
AllowedIPs = 192.168.20.0/24,192.168.21.0/24
Endpoint = 公网ip:端口
PersistentKeepalive = 30
PublicKey = 公钥
```

注意的是，对于需要进行内网穿透的设备，所配置的 `peer` 中的 `AllowedIPs` 需包含连接的网段（比如用手机远程ssh电脑，那电脑的wg 的 `peer` 配置中需要包含手机的 wg ip 的网段），且防火墙需要针对wg的虚拟网卡采用对应的策略