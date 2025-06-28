---
title: '机械革命无界 15X Linux 优化指南'
description: '怎么就想不开买机革呢.webp'
slug: mechrevo-wujie15x-linux
date: 2025-06-01T23:04:18+08:00
categories: 开发
tags: 
 - 软件
 - 电脑
 - Linux
---

虽然说标题写着 15X 不过 14X 和 15X 都是通用的  
这破机子什么都得自己来，麻烦死了）  
不过幸好德国一个叫 TUXEDO 的公司也用了相同的模具，于是就有了驱动和控制面板

## RJ45 网卡

你机革不知道从哪里捡的网卡，没进主线导致没驱动，只能自己装  

Arch Linux 系用户: 用 AUR 的 [yt6801-dkms](https://aur.archlinux.org/packages/yt6801-dkms) (我维护的)  
Debian/Ubuntu 系用户: [官网下载](https://www.motor-comm.com/Public/Uploads/uploadfile/files/20250430/yt6801-linux-driver-1.0.30.zip)

## 键盘背光等驱动

Arch Linux 系用户: 用 AUR 的 [mechrevo-drivers-dkms](https://aur.archlinux.org/packages/mechrevo-drivers-dkms) (也是我维护的xwx，TUXEDO 的驱动打了 patch 来支持机革)  
Debian/Ubuntu 系用户: 自己拉[源码](https://github.com/tuxedocomputers/tuxedo-drivers)打 [patch](https://github.com/sund3RRR/Mechrevo14X-linux/raw/master/patches/add_mechrevo_vendor.patch) 然后编译安装吧  

## 控制面板

Linux 上的面板只能改一下性能配置、键盘背光等，聊胜于无吧  
装好之后在 KDE 的状态栏亮度设置里就能调整键盘背光强度和颜色了  

Arch Linux 系用户: 用 AUR 的 [tuxedo-control-center-bin](https://aur.archlinux.org/packages/tuxedo-control-center-bin) (AUR 万岁！)  
Debian/Ubuntu 系用户: 按照 TUXEDO 的[教程](https://www.tuxedocomputers.com/en/Add-TUXEDO-software-package-sources.tuxedo)添加源后安装 `tuxedo-control-center` 包

## 无法睡眠
  
14X 可以试试添加 `acpi.ec_no_wakeup=1` 到内核参数里  

15X 新建一条 udev 规则禁用 PS/2 键盘的 Wakeup Trigger[^1]  
`/etc/udev/rules.d/99-disable-keyboard-wakeup.rules`   

```
# Disable wakeup for PS/2 keyboard controller
ACTION=="add", SUBSYSTEM=="serio", KERNEL=="serio0", ATTR{power/wakeup}="disabled"
```

然后重载 udev 规则并重启  

```
sudo udevadm control --reload-rules
sudo udevadm trigger
reboot
```

## 更新 Plasma 6.4.0 后鼠标光标卡顿

在设置-显示和监视器里关掉自适应同步就好了  
就这个破问题搞了我两个星期……

## 人脸识别

说真的在 Linux 其实不好用，唯一一个还算能用的方案是 [howdy](https://github.com/boltgolt/howdy)   
14X/15X 的红外摄像头是 `/dev/video2`

## 性能优化

用 CachyOS，有针对 Zen4 专门优化的仓库

## 参考

[Github - mechrevo14X-linux](https://github.com/sund3RRR/mechrevo14X-linux)  
[^1]: [在机械革命无界 15XPro 暴风雪上运行 Linux](https://zeeko.dev/2025/06/running-linux-on-mechanical-revolution-15xpro-blizzard/) (感谢 Zeeko 的补充！)
