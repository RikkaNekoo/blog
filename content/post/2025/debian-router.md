---
title: '让 Debian 作为一台路由器'
description: 'All in Debian!'
date: 2025-12-08T14:46:11+08:00
slug: 'debian-router'
categories: 开发
tags:
  - 软件
  - 教程
  - 网络
---

上个月给布置了一台 Dell R720，买了 4x2T 硬盘组硬 RAID5，底层要是虚拟化系统的话还要分别开路由器和文件服务器，实在不是很优雅，加之一直想组一个 All in Debian，遂行动  
过程真的踩了很多坑就是了，幸好有专家系统 (ChatGPT) 帮忙  

## 系统准备

### sysctl

首先创建 `/etc/sysctl.d/99-router.conf` 并写入以下内容

```
net.ipv4.ip_forward = 1
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.all.forwarding = 1
net.ipv6.conf.default.forwarding = 1
net.ipv6.conf.all.accept_ra = 2
net.ipv6.conf.default.accept_ra = 2
net.ipv6.conf.br-lan.accept_ra = 0
```

`net.ipv6.conf.br-lan.accept_ra` 的 `br-lan` 改为实际 LAN 网卡

然后执行 `sudo sysctl --system`  

碎碎念：Debian 13 的 systemd-sysctl 已经忽略 `/etc/sysctl.conf` ，当时还一直纳闷为什么 sysctl -p 为什么不生效

### 软件安装

`pppoeconf`：用于宽带拨号  
`wide-dhcpv6-client`：用于获取 IPv6 PD 前缀并配置到接口  
`dnsmasq`：用于为 LAN 提供 DNS 服务、DHCP 服务和 IPv6 路由通告服务  
`docker` 或 `podman`：可选，稍后解释用处

## 网络接口配置

如果你只有一个 LAN 口，则将以下配置添加到 `/etc/network/interfaces`

```
auto enp1s0
allow-hotplug enp1s0
iface enp1s0 inet static
    address 10.21.0.1/24
```

注意将 `enp1s0` 改成你的实际网卡名称， `10.21.0.1/24` 改成你想要的网段

如果有多个 LAN 口，则需要建立桥接，配置如下

```
auto br-lan
iface br-lan inet static
    address 10.21.0.1/24
    bridge-ports eno2 eno3 eno4
    bridge-stp off
```

同样，将 `eno2` `eno3` `eno4` 改成你的实际网卡名称， `10.21.0.1/24` 改成你想要的网段

## 拨号

插好 WAN 口网线，执行 `sudo pppconfig`，会自动检测 WAN 口并拨号  

```
┌─────────────────────┤ USE PEER DNS ├─────────────────────┐
│ You need at least one DNS IP address to resolve the      │
│ normal host names. Normally your provider sends you      │
│ addresses of useable servers when the connection is      │
│ established. Would you like to add these addresses       │
│ automatically to the list of nameservers in your local   │
│ /etc/resolv.conf file? (recommended)                     │
│               <Yes>                  <No>                │
└──────────────────────────────────────────────────────────┘
```

注意这里要选 No，因为后面我们要使用 Dnsmasq 作为 DNS 服务器  
其他选项全部 Yes 就行  

`pppconfig` 执行完之后，还需要进行一些修改  
首先打开 `/etc/ppp/peers/dsl-provider`，并在最后一行添加 `+ipv6`，修改好的配置如下

```
noipdefault
defaultroute
replacedefaultroute
hide-password
#lcp-echo-interval 30
#lcp-echo-failure 4
noauth
persist
#mtu 1492
#persist
#maxfail 0
#holdoff 20
plugin rp-pppoe.so
nic-eno1
user "0d000721"
+ipv6
```
然后将 `/etc/ppp/ip-up.d/0clampmss` 和 `/etc/ppp/ip-down.d/0clampmss` 分别复制到 `/etc/ppp/ipv6-up.d/0clampmss` 和 `/etc/ppp/ipv6-down.d/0clampmss`，并将里面的 `iptables` 改成 `ip6tables`  

修改完后执行 `sudo systemctl restart networking` 重新拨号，此时服务器已经可以上网了，但是由于还没有配置 DNS，所以暂时只能 ping IP  

## 分配 PD
这边使用 `wide-dhcpv6-client` 来获取 PD 并分配到 LAN  
打开 `/etc/wide-dhcpv6/dhcp6c.conf`，修改为如下内容

```
interface ppp0 {
    send ia-pd 0;
};

id-assoc pd 0 {
    prefix-interface br-lan {
        sla-len 0;
        ifid 0;
    };
};
```

注意 `br-lan` 修改为你实际的 LAN 网卡名称  
`sla-len` 代表前缀长度，如果下发的是 /60 则填写 4 获得 /60+4=/64，如果下发的是 /64 填写 0，/64+0=/64，以此类推，总之最后的长度需要为 /64

每次重新拨号都会有 PD 残留在 LAN 网卡，经过询问专家系统，在星野玲的方案上稍加修改，终于搞定了，还顺便解决了重启之后 wide-dhcpv6-client.sevice 自启动失败的问题  

创建 `/etc/ppp/ipv6-up.d/restart_wide-dhcpv6-client` 并写入以下内容  

```
#!/bin/sh

systemctl restart wide-dhcpv6-client
systemctl restart dnsmasq

exit 0
```

创建 `/etc/ppp/ipv6-down.d/del_ipv6_pd_prefix` 并写入以下内容

```
#!/usr/bin/bash

INTERFACE=br-lan

IPV6_ADDRESS=$(ip a show dev br-lan scope global | awk '$2 ~ /::\/64$/ {print $2}')

if [[ -n "$IPV6_ADDRESS" ]]; then
  ip addr del $IPV6_ADDRESS dev $INTERFACE
fi

exit 0
```

这里的 `br-lan` 改为你实际的 LAN 网卡

解释：`del_ipv6_pd_prefix` 在 ppp0 掉线时删除分配给 LAN 的 PD  
`restart_wide-dhcpv6-client` 则是在 ppp0 上线时重启 wide-dhcpv6-client 和 dnsmasq，给 LAN 重新分配 PD 并让 Dnsmasq 更新 DHCP 下发的地址

## DNS 和 DHCP

修改 `/etc/dnsmasq.conf`，内容如下

```
interface=br-lan
interface=lo
bind-interfaces
port=53
server=223.5.5.5
enable-ra
log-dhcp
dhcp-range=10.21.0.2,10.21.0.254,12h
dhcp-range=::,constructor:br-lan,ra-stateless,12h
dhcp-option=option:router,10.21.0.1
dhcp-option=option:dns-server,10.21.0.1
dhcp-option=option6:dns-server,[::]
```

`br-lan` 修改为实际的 LAN 网卡，`dhcp-range` 根据自己的网段来填写，`10.21.0.1` 改为你设置的 LAN 口 IP  
执行 `sudo systemctl restart dnsmasq`，此时下游设备已经可以获取到 IP 了，但是 IPv4 是上不了网的，因为此时还没配置 NAT  
记得在 `/etc/resolv.conf` 第一行加入 `127.0.0.1` 哦

## 配置 NAT
通常的教程都是使用 iptables 或者 nftables 来进行 NAT 转换，但是笔者偏不走寻常路（x  
让我们看向 [einat](https://github.com/EHfive/einat-ebpf)  

> einat is an eBPF-based Endpoint-Independent NAT(Network Address Translation).  

笔者这边的移动十分大方的给了 NAT1，使用 einat 可以很方便的让局域网内设备都获得 Endpoint-Independent NAT  

首先下载二进制文件并移动到 `/usr/bin`

```
wget https://github.com/EHfive/einat-ebpf/releases/download/v0.1.9/einat-static-x86_64-unknown-linux-musl
chmod a+x einat-static-x86_64-unknown-linux-musl
mv einat-static-x86_64-unknown-linux-musl /usr/bin/einat
```

然后创建 `/etc/systemd/system/einat.service` 实现开机自启

```
[Unit]
Description=EINAT
Wants=network-online.target
After=network-online.target

[Service]
ExecStart=/usr/bin/einat --ifname ppp0 --hairpin-if lo br-lan 
ExecStop=/bin/kill -TERM $MAINPID
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

记得 `br-lan` 改为实际 LAN 网卡  
执行 `sudo systemctl daemon-reload` 和 `sudo systemctl enable --now einat.service`  
好啦，此时下游设备应该已经全部可以上网了

## UPnP
**注意！miniupnpd 在 Debian 13 无法正常工作，会导致 NAT 失效 ([参见](https://www.v2ex.com/t/1151385))**  
如果真的需要 UPnP 的话，只能在容器里跑，这就是上面 `docker` 或者 `podman` 可选安装的原因

compose.yaml 如下

```
services:  
  miniupnpd:
    cap_add:
      - NET_ADMIN
    container_name: miniupnpd
    image: hoshinorei/miniupnpd
    network_mode: host
    restart: always
    tmpfs:
      - /run
    volumes:
      - ./miniupnpd.conf:/etc/miniupnpd/miniupnpd.conf:ro
```

miniupnpd.conf 如下

```
ext_perform_stun=yes
ext_stun_host=stun.hot-chilli.net
ext_stun_port=3478
system_uptime=yes
uuid=a1837441-8d1f-43ef-ba15-d42432854810
ext_ifname=ppp0
ipv6_disable=yes
enable_upnp=yes
enable_natpmp=yes
listening_ip=br-lan
allow 1024-65535 0.0.0.0/0 1024-65535
deny 0-65535 0.0.0.0/0 0-65535
```

uuid 需要自己生成一个，可以使用 `uuidgen` 命令  
`listening_ip=br-lan` 记得改成自己的 LAN 网卡

## 结尾
弄了这么久，终于拥有了一个 Debian 软路由了  
Debian 可以折腾的可比 OpenWrt 多得多啦，慢慢发掘吧
未来笔者或许会折腾一些更复杂的东西，比如猫棒拨号然后 VLAN 划分上网和 IPTV 什么的，敬请期待啦

## 参考
[使用 Debian 作为路由器](https://blog.bling.moe/post/3/)  
[Debian 12 软路由配置随记](https://blog.h1ra.net/post/debian-router/)