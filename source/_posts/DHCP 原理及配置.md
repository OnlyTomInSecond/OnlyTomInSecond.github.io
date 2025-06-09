## 工作原理 ##

7个阶段

1. 发现阶段，设备首次接入网络时，发送discover消息，寻找可用的DHCP服务器。消息包含空/临时的IP地址

2. 提供阶段，DHCP服务器收到discover消息，回应DHCP offer，包含可用IP，子网掩码，网关，DNS。有多个server则客户端收到多个offer，通常只能选一个

3. 请求阶段，客户端收到offer，选择其中一个DHCP服务器的配置，向其发送DHCP request消息，请求分配

4. 确认阶段，服务端收到request消息，确认IP及其他配置；发送DHCP acknowledgement消息；其他DHCP server停止提供配置

5. 配置生效，客户端收到acknowledgement消息后，配置网络接口。

6. 租约，续约：所分配的IP有一定时间期间，到期时客户端可选择继续使用原IP或重新分配新IP。

7. 释放，客户端发送release消息给服务端，即可释放IP。

## dhcpd 配置 ##

目前dhcpd已经不再维护，此处教程仅作记录，且均来自 [Arch WIKI DHCPD](https://wiki.archlinux.org/title/Dhcpd)

### 1. 设置静态ip ###

可以通过ip命令来设置静态ip

```sh
ip link set up dev eth0  # 启用eth0
ip addr add 139.96.30.100/24 dev eth0 #为eth0添加网址
```

### 2. 修改配置文件 ###

先备份原有的配置文件

```sh
cp /etc/dhcpd.conf /etc/dhcpd.conf.example
```

创建相关子网，可以参考如下配置：

```
option domain-name-servers 8.8.8.8, 8.8.4.4;
option subnet-mask 255.255.255.0;
option routers 139.96.30.100;
subnet 139.96.30.0 netmask 255.255.255.0 {
  range 139.96.30.150 139.96.30.250;
}
```

