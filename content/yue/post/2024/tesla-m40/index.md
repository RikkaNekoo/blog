---
date: 2024-05-31T14:36:07.467Z
title: Tesla M40 上手体验
slug: tesla-m40
categories: 开发
tags:
    - 软件
    - 硬件
    - 电脑
    - 评测
fancybox: true
---

## 关于
买了台零刻 SER5 MAX，对 Vega 7 羸弱的性能实在是难以忍受  
但是又想要便携性不损失，于是便选择了用 NVMe 转 SFF-8612 (OCuLink)，要带走电脑的话直接把线拔了就行了  
使用的是开源宇宙的 EG01，有个架子还是美观点的  
显卡选择 Tesla M40 的最大原因其实是没钱（x  
性能还 OK，显存也大，重点是还便宜  
![](nvidia-smi.webp)  

## 折腾过程
到货，插电，开机，轻松卡自检  
其实是预料之中的问题，把整个 BIOS 翻烂了都找不到 Above 4G Decoding 的选项，用 UAFB 依然如此  
经过查找，看到一篇[教程](https://www.bilibili.com/read/cv20768695/)，使用 UEFITool 查找 Above 4G Decoding 的 VarStoreInfo 后使用 modGRUBShell 强开  
**（附：SER5 MAX 的 VarStoreInfo 为 0xF7）**  
开启完成之后就能成功开机了  
接下来就是正常流程的装驱动 ~~，改注册表开启 WDDM ([参考](https://www.bilibili.com/read/cv23955139/))~~  
事实上并不需要，只要**管理员**运行

```powershell
nvidia-smi -dm 0
```
即可切换 WDDM 模式  
Linux 下就更简单了，装好闭源驱动就能直接就能用于图形渲染  
不过要注意禁用 GSP-RM，否则会出现占用低帧数上不去的问题 ([出处](https://www.ctyun.cn/document/10029787/10356098))

```bash
sudo su -c 'echo options nvidia NVreg_EnableGpuFirmware=0 > /etc/modprobe.d/nvidia-gsp.conf'
sudo update-initramfs -u #Debian/Ubuntu
sudo mkinitcpio -P #ArchLinux
reboot
```

## 性能&跑分
1. 地平线4:
    - 在 Linux 下预设高，1080P 全程 60 帧但会有不频发的小卡顿，2K 下可以跑 55-60 帧，小卡顿较为频发但不太影响游戏  
    - 在 Windows 下预设高，2K 除了在地平线全程 60 帧  

**To do**

## 使用体验
显卡坞可以随机启停，这点很棒，不用一直按来按去开关了  
风扇是 EVGA 的 980Ti 的，改装了温控，不打游戏时很安静，打游戏也不会很吵  
温度控制的很好，不管玩什么游戏都不会超过 50℃