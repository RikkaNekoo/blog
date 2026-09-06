---
title: 'NixOS 邪修小窍门'
date: 2026-08-23T15:14:57+08:00
slug: 'the-nixos-life-hack'
description: '为什么妳要想不开去用 NixOS 呢'
categories: 杂物间
tags:
  - 软件
  - 教程
---

仅用来存档一些在 NixOS 的 HomeLab 使用过程中的一些很 hacky 的 workaround  
~~既然本文的分类归到了杂物间而不是教程，那当然不是什么很正经的东西了~~

## PCI VF 设备直通

由于 CX3 系列网卡的 VF 由 PF 加载完成后 mlx4 驱动在产生，且一出现就会被 mlx4 驱动绑定，vfio-pci 驱动抢不过，即便使用了 ``vfio-pci.ids=xxxx:yyyy`` 也无济于事  

在直通 PCI 设备时如果使用 libvirtd 的 ``managed=yes`` 的话会出现虚拟机开机找不到 fd 设备的情况，于是就诞生了这么一个神奇的 workaround  

```nix
{ pkgs, ... }:

{
  systemd.services.vfio-bind-05001 = {
    description = "Bind 0000:05:00.1 to vfio-pci";
    wantedBy = [ "multi-user.target" ];
    # sys-devices-pci0000:00-0000:00:03.0-0000:05:00.1-net-enp5s0v0.device
    # 改为你实际的设备，上下文中的 0000:05:00.1 和 05001 同理
    after = [ "sys-devices-pci0000:00-0000:00:03.0-0000:05:00.1-net-enp5s0v0.device" ];

    serviceConfig = {
      Type = "oneshot";
      ExecStart = pkgs.writeShellScript "bind-vfio" ''
        DEV="0000:05:00.1"

        # 每 0.1s 轮询，5s 后设备未出现则中止
        for i in $(seq 1 50); do
          if [ -e /sys/bus/pci/devices/$DEV ]; then
            break
          fi
          sleep 0.1
        done

        # 解绑原有驱动
        if [ -L /sys/bus/pci/devices/$DEV/driver ]; then
          echo $DEV > /sys/bus/pci/devices/$DEV/driver/unbind
        fi

        # override vfio-pci 驱动
        echo vfio-pci > /sys/bus/pci/devices/$DEV/driver_override

        # 绑定到 vfio-pci
        echo $DEV > /sys/bus/pci/drivers/vfio-pci/bind
      '';
    };
  };

  # libvirtd 需在 vfio-pci 绑定完成后再启动
  systemd.services.libvirtd = {
    wants = [ "vfio-bind-05001.service" ];
    after = [ "vfio-bind-05001.service" ];
  };
}
```

## Bittorrent 容器打 DSCP 标

想必大家都不想让自己的 BT 流量走代理吧，毕竟流量可不便宜呢x  
但是 BT 软件跑在容器里，代理软件又不在本机上没法用 Process Name 分流该怎么办呢…?  
没关系！我们万能的 dae 支持 DSCP 的 TCP/UDP 分流，只要给 BT 流量全部打上标就好了  

但是这又引出另一个问题了，软件是跑在容器里的，应该怎么打标呢（？  
这里的 workaround 就是拿到进程的 cgroupsv2 路径后利用 nftables 打上 DSCP 标  
注意！``podman-container-name.service`` 服务的 cgroups 路径并非容器的路径，需要使用 ``podman inspect`` 才能拿到容器内进程真实的路径

```nix
{ pkgs, ... }: 

{
  networking.nftables = {
    enable = true;
    checkRuleset = false;
    ruleset = ''
      table inet filter {
        set qb_cgroup {
          type cgroupsv2
        }

        chain output {
          type filter hook output priority 0; policy accept;
          socket cgroupv2 level 3 @qb_cgroup ip dscp set ef counter
          socket cgroupv2 level 3 @qb_cgroup ip6 dscp set ef counter
        }
      }
    '';
  };

  # TODO: Shit implement
  systemd.services.podman-qbittorrent.postStart = ''
    cg=$(
      ${pkgs.podman}/bin/podman inspect qbittorrent \
        | ${pkgs.jq}/bin/jq -r '.[0].State.CgroupPath'
    )

    cg=$(echo "$cg" | sed 's#^/##')
    cg="$cg/container"

    ${pkgs.nftables}/bin/nft flush set inet filter qb_cgroup
    ${pkgs.nftables}/bin/nft add element inet filter qb_cgroup "{ \"$cg\" }"
  '';
}
```
