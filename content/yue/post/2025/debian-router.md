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

`network-manager`：用于替代 ifupdown 管理网卡   
`ppp`：用于宽带拨号  
`dnsmasq`：用于为 LAN 提供 DNS 服务、DHCP 服务和 IPv6 路由通告服务  
`docker` 或 `podman`：可选，稍后解释用处

## 网络接口配置

首先，将 `/etc/network/interfaces` 中除 loopback 之外的网卡配置全部删除

如果你只有一个 LAN 口，则使用   

```
sudo nmcli c add type "ethernet" con-name "lan" ifname "enp1s0" ipv4.method "manual" ipv4.addresses "10.21.0.1/24"
```

注意将 `enp1s0` 改成你的实际网卡名称， `10.21.0.1/24` 改成你想要的网段

如果有多个 LAN 口，则需要建立桥接，配置如下

```
# 创建桥
sudo nmcli c add type "bridge" con-name "br-lan" ifname "br-lan" ipv4.method "manual" ipv4.addresses "10.21.0.1/24"
# 将网卡添加到桥
sudo nmcli c add type "bridge-slave" ifname "eno2" master "br-lan"
sudo nmcli c add type "bridge-slave" ifname "eno3" master "br-lan"
# 以此类推
```

同样，将 `eno2` `eno3` 改成你的实际网卡名称， `10.21.0.1/24` 改成你想要的网段

## 拨号

最初 Rikka 使用的是 ifupdown + pppconfig 的方案，但是这套方案不仅过时而且稳定性欠佳（用着用着突然出现开机不自启了  
最终还是切换到了 NetworkManager，稳定性更好更先进而且配置也简单ww  

只需要很简单的一行  

```
sudo nmcli c add type "pppoe" con-name "pppoe" ifname "eno0" username "0d00" password "0721"
```

将 `eno0` 改为实际的 WAN 网卡并修改账号密码即可完成配置  

使用   

```
sudo nmcli c modify "br-lan" ipv6.method "shared"
```

给 LAN 分配 PD，不过 NetworkManager 无法修改前缀长度，只能固定 /64  
（记得将 `br-lan` 改为实际的 LAN 配置名称  

然后在 `/etc/nftables.conf` 的最下面添加以下内容来开启 MSS 钳制  

```
table inet filter {
    chain forward {
        type filter hook forward priority 0; policy accept;
        tcp flags syn tcp option maxseg size set rt mtu
    }
}
```

并使用 `sudo systemctl restart nftables` 重启 nftables  
（不开的话大包会因为禁止分片被丢掉的哦  

如果不想使用 ISP 下发的 DNS 的话，可以执行下面的命令来禁用  

```
sudo nmcli c modify pppoe ipv4.ignore-auto-dns "yes" ipv6.ignore-auto-dns "yes"
sudo nmcli c modify br-lan ipv6.ignore-auto-dns "yes"
```

全部弄完后执行 `sudo nmcli c up pppoe` 拨号，此时服务器已经可以上网了  

每次重新拨号都会有 PD 残留在 LAN 网卡，所以需要在 IPv6 下线时重启 PPPoE 以获取新的 PD，并重启 dnsmasq 重新分发  
创建 `/etc/ppp/ipv6-up.d/restart_dnsmasq` 并写入以下内容  

```
#!/bin/sh
systemctl restart dnsmasq
```

创建 `/etc/ppp/ipv6-down.d/restart_pppoe` 并写入以下内容

```
#!/usr/bin/bash
nmcli c down pppoe && nmcli c up pppoe
```

这里的 `pppoe` 改为 NetroowkManager 的 PPPoE 配置名称 

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
通常的教程都是使用 iptables 或者 nftables 来进行 NAT 转换，但是 Rikka 偏不走寻常路（x  
让我们看向 [einat](https://github.com/EHfive/einat-ebpf)  

> einat is an eBPF-based Endpoint-Independent NAT(Network Address Translation).  

笔者这边的移动十分大方的给了 NAT1，使用 einat 可以很方便的让局域网内设备都获得 Endpoint-Independent NAT  

首先下载二进制文件并移动到 `/usr/bin`

```
wget https://github.com/EHfive/einat-ebpf/releases/download/v0.1.9/einat-static-x86_64-unknown-linux-musl
chmod a+x einat-static-x86_64-unknown-linux-musl
sudo mv einat-static-x86_64-unknown-linux-musl /usr/bin/einat
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
[使用 NetworkManager 配置 Debian 路由器](https://blog.bling.moe/post/19/)   
[Debian 12 软路由配置随记](https://blog.h1ra.net/post/debian-router/)