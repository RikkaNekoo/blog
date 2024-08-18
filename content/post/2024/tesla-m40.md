---
date: 2024-05-31T14:36:07.467Z
updated: 2024-08-11T11:24:53.278Z
title: Tesla M40 上手体验
categories: 开发
tags:
- Dev
- Software
- Computer
---

# Tesla M40 上手体验


## 关于
买了台零刻SER5 MAX，对Vega 7羸弱的性能实在是难以忍受  
但是又想要便携性不损失，于是便选择了用NVMe转SFF-8612 (OCuLink)，要带走电脑的话直接把线拔了就行了  
使用的是开源宇宙的EG01，有个架子还是美观点的  
显卡选择Tesla M40的最大原因其实是没钱（x  
性能还OK，显存也大，重点是还便宜  
![](https://rikka.im/api/v2/objects/file/osvmoq42gyt3mrjgsb.png)  

## 折腾过程
到货，插电，开机，轻松卡自检  
其实是预料之中的问题，把整个BIOS翻烂了都找不到Above 4G Decoding的选项，用UAFB依然如此  
经过查找，看到一篇[教程](https://www.bilibili.com/read/cv20768695/)，使用UEFITool查找Above 4G Decoding的VarStoreInfo后使用modGRUBShell强开  
**（附：SER5 MAX的VarStoreInfo为0xF7）**  
开启完成之后就能成功开机了  
接下来就是正常流程的装驱动~~，改注册表开启WDDM ([参考](https://www.bilibili.com/read/cv23955139/))~~  
事实上并不需要，只要**管理员**运行

```powershell
nvidia-smi -dm 0
```
即可切换WDDM模式  
Linux下就更简单了，装好闭源驱动就能直接就能用于图形渲染  
不过要注意禁用GSP-RM，否则会出现占用低帧数上不去的问题 ([出处](https://www.ctyun.cn/document/10029787/10356098))

```bash
sudo su -c 'echo options nvidia NVreg_EnableGpuFirmware=0 > /etc/modprobe.d/nvidia-gsp.conf'
sudo update-initramfs -u #Debian/Ubuntu
sudo mkinitcpio -P #ArchLinux
reboot
```

## 性能&跑分
地平线4: 在Linux下预设高，1080P全程60帧但会有不频发的小卡顿，2K下可以跑55-60帧，小卡顿较为频发但不太影响游戏  
To do

## 使用体验
显卡坞可以随机启停，这点很棒，不用一直按来按去开关了  
风扇是EVGA的980Ti的，改装了温控，不打游戏时很安静，打游戏也不会很吵  
温度控制的很好，不管玩什么游戏都不会超过50℃