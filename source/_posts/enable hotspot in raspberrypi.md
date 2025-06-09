---
title: 树莓派开 wifi 热点
---
由于所在大学校园网改造，宽带被废，校园网帐号限设备，想着能不能用树莓派做个软路由，去除设备限制。于是有了这个教程，记录一下只自己的折腾过程。注意，该教程只适用于 debian 系列的 linux 发行版

总的来说，这次的方案是 `hostapd + iptables +dnsmasq` ，`hostapd` 用来开启 `ap` ， `iptables` 用来转发流量，`dnsmasq` 用来作 `dhcp` 服务器。

### 1. `hostapd` 配置

- 1. 安装 `hostapd` ，直接 `sudo apt install hostapd` 
- 2. 配置。这里直接贴出对 树莓派3b+ 的配置 `/etc/hostapd/hostapd.conf`（默认的配置文件位置可以手动指定，修改 `/etc/default/hostapd` 文件中的 `DAEMON_CONF` 变量即可）。
```sh
# 无线网卡的名称interface=wlan0
# 网卡对应的驱动名 
driver=nl80211
# 无线网络的名称是name
ssid=name
# 无线路由器工作模式为802．11g（2.4G）
hw_mode=g
# 无线网卡使用的信道channel=10
# 支持 802.11n
ieee80211n=1
# 采用WPA2配置
wpa=2
# 无线网络密码是123456789
wpa_passphrase=123456789
# 认证方式为WPA-PSK
wpa_key_mgmt=WPA-PSK
# 开启 WMM
wmm_enabled=1
# 开启 40MHz channels 和 20ns guard interval
ht_capab=[HT40][SHORT-GI-20][DSSS_CCK-40]
# 接受所有 MAC 地址macaddr_acl=0
# 使用 WPA 认证auth_algs=1
# 需连接者知道ssid
ignore_broadcast_ssid=0
# 使用 WPA2
wpa=2
# 使用预先共享的 key
wpa_key_mgmt=WPA-PSK
# 使用 AES, 而非 TKIP
rsn_pairwise=CCMP
```

尝试启动 `hostapd.service` 后如果没有启动失败，则可以设置成开机启动。

### 2. `dnsmasq` 配置

- 1. 先安装 `dnsmasq` 软件。

- 2. 修改 `/etc/dnsmasq.conf` 文件，在文件的最后加

```sh
interface=wlan0 # make dnsmasq listen for requests only on intern0 (our LAN)
#no-dhcp-interface=intern0  # optionally disable the DHCP functionality of dnsmasq and use systemd-networkd instead
expand-hosts      # add a domain to simple hostnames in /etc/hosts
domain=rasp    # allow fully qualified domain names for DHCP hosts (needed when
                  # "expand-hosts" is used)
dhcp-range=192.168.12.10,192.168.12.100,255.255.255.0,2h # defines a DHCP-range for the LAN: 
                  # from 10.0.0.2 to .255 with a subnet mask of 255.255.255.0 and a
                  # DHCP lease of 1 hour (change to your own preferences)
dhcp-option=6,223.5.5.5
dhcp-option=121,192.168.12.0/24,192.168.12.1

```

注意，此配置指定需要进行动态ip分配的 `interface` 为 `wlan0` ，分配的ip范围为 `192.168.12.10` 到 `192.168.12.100` ，掩码为 `225.225.225.0` ，分配的时间为2小时， 局域网dns设置为 `223.5.5.5` ， 并假设 `192.168.12.1`  。

启动 `dnsmasq.service` 若没有错误，则可以设置成开机启动。

### 3. `iptables` 配置

参考 [arch wiki](https://wiki.archlinux.org/title/Internet_sharing#With_iptables) 的 `iptables` 配置。自己的配置命令如下：

```sh
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT
```

上面的规则是把 `wlan0` （也就是热点）的网络请求转发到可连接外部网络的 `eth0` 。如果之前没有配置过 `iptables` ，则此命令可直接执行。如果之前有配置，则酌情修改命令。这里贴出清除 `iptables` 原有规则的命令

```sh
iptables -F
iptables -F -t nat
iptables -F -t mangle
```

配置好路由规则后，使用 `iptables-save` 将规则保存，启用 `netfilter-persistent.service` 将规则永久保存（没有该服务则去安装 `iptables-persistant` 这个包）。

### 4. 静态ip

从 `arch wiki` 上看，有两种方法，一种是 `systemd-networkd` ，另外一种是 `netctl` 。但是吧，个人比较懒，用的网络管理器是 `NetworkManager` ，而且之前通过在 `/etc/network/interface` 内配置静态ip会导致 `hostapd` 服务概率性启动失败，于是干脆直接写个 `systemd` 服务来执行一个脚本，跑在 `hostapd` 之后，经尝试，该方法不会导致开机时 `hostapd` 启动失败。

执行的脚本：

```sh
#!/bin/sh
# 将设备连接开启
sudo ip link set up dev wlan0
# 将wlan0的ip设置为 192.168.12.1
sudo ip addr add 192.168.12.1/24 dev wlan0
```

服务项：
```
[Unit]
Description="Setup static ip for wlan0"
Requires=hostapd.service
After=hostapd.service

[Service]
Type=oneshot
User=somebody
ExecStart=/path/to/script.sh

[Install]
WantedBy=multi-user.target
```

根据需要修改脚本文件中的设备和服务项中脚本的位置（ `ExecStart` 字段），并分别保存为脚本文件和服务项文件后，将服务项文件复制到 `/usr/lib/systemd/system/` 目录，然后执行 `systemctl daemon-reload` ，**测试无误后**并设置为开机启动。

完成上述工作后，可重启。至此，树莓派wifi热点的**初步**配置已完成。

为什么说是初步呢？因为目前所做的工作，仅仅只是复刻了一个路由器，转发相应网络接口的流量到另外一个接口。但众所周知，校园网这类网有登录验证和设备验证，甚至需要专门的客户端来验证登录。因此，我们需要解决这两类问题。

先说一下总体思路，个人需要的效果是，树莓派连接校园网，客户端连接树莓派的热点，在客户端上登录校园网获得上网能力，但不被检测出多设备。当然，完全可以把登录验证放在树莓派上，这样可以做到一连即用。

### 4.登录验证相关

一般登录验证有走网页的强制认证( `captive portal` ) 和 专门的客户端验证，“解决” 这种验证有两种方式：

- 1. 扫网关的端口，并在外部网络布置代理，内部网络使用代理走特定端口来绕过验证。此方法无视登录验证甚至流量限制。
- 2. 对网页/客户端进行抓包/破解，写脚本定时发请求来实现自动连接和保活。此方法只是实现自动登录和登录掉线相关的处理，让登录验证 “无感” 化。一般来说，网页的登录验证界面都是不加密的（指使用 `http` ，至于传输的内容加密，你可以试试抓包直接拿解密密钥）

~~不巧的是，我两个都不会~~

所以我选摆烂方式，就是局域网内一个设备登录，通过路由转发来实现共享。

### 5.设备限制相关

校园网检测路由器和多设备方法可以参考 [这篇文章](https://blog.sunbk201.site/posts/crack-campus-network/) 配置相关的 `iptables` 规则。不过在此基础上可能要在路由器加上过滤来自检测网关的 arp 包。

这里简单说一下相关检测。

- 1. ipv4 数据包头 `TTL` 检测

对于此类检测，可以用 `iptables` 把数据包的ttl值设置成64来解决，命令如下：

    ```sh
    sudo iptables -t mangle -A POSTROUTING -j TTL --ttl-set 64
    ```

- 2. http数据请求头内的 User-Agent 字段检测

一般 https 的流量是加密的，网关无法分析其中的ua，但是 http 流量并没有加密，因此需要转发 80 端口的 http 流量到特定软件处理，去除或者统一修改ua，这里使用的方案是 `privoxy` 代理。

    - 1. 首先， 安装 `privoxy`: `sudo apt install privoxy`
    
    - 2. 修改 `/etc/privoxy/config` 文件，将默认配置的 `action` 和 `filter` 注释掉。 找到 `actionsfile` 字段和 `filterfile` 字段，注释原有的值，添加 `actionsfile format_ua.action` 。

        添加 `accept-intercepted-requests 1` 以允许处理来自iptables转发的流量。

        新建文件 `/etc/privoxy/format_ua.action`， 加入以下内容：

        ```
        { \
        +hide-user-agent{你自己的ua} \
        }
        /
        ```

        保存后重启 `privoxy.service` 。

    - 3. 配置 iptables 转发。给出命令：

        ```
        sudo iptables -t nat -N http_ua_drop
        sudo iptables -t nat -I PREROUTING -p tcp --dport 80 -j http_ua_drop
        sudo iptables -t nat -A http_ua_drop -m mark --mark 1/1 -j RETURN
        sudo iptables -t nat -A http_ua_drop -d 0.0.0.0/8 -j RETURN
        sudo iptables -t nat -A http_ua_drop -d 127.0.0.0/8 -j RETURN
        sudo iptables -t nat -A http_ua_drop -d 192.168.0.0/16 -j RETURN
        sudo iptables -t nat -A http_ua_drop -p tcp -j REDIRECT --to-port 8118
        ```
    
        保存 iptables 配置，并重启 `netfilter-persistent.service`。用 [这个链接](ua.233996.xyz) 测试修改后的 ua 是否生效， 如果生效， 看到的两个 ua 有一个是在 `privoxy` 中设置的 ua 。

- 3. DPI 深度包检测。

对于此类检测，最好的应对方法是把流量加密，不过需要一台校园网外部的服务器来充当代理。 至于代理，网上的教程十分多，不在此赘述。

- 4. 其他检测。 

除了上述几种检测和应对方法， [这篇文章](https://blog.sunbk201.site/posts/crack-campus-network/) 还讲述了时钟偏移检测， 网络协议栈时钟偏移检测等，并且有相关的应对方法，这些方法在现在看来甚至没有过时。

最后，所有配置检查无误后， 重启树莓派， 看有无配置失效的情况。
