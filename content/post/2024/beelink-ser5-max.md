---
date: 2024-05-04T16:55:23.609Z
title: 零刻SER5 MAX使用体验
slug: beelink-ser5-max
image: https://pic3.zhimg.com/v2-251c8c7b96949f2d045db157e46a5106_b.webp
categories: 杂物间
tags:
    - 电脑
    - 硬件
    - 评测
    - 随笔
---

## 参数以及外观

### 外观
[图片来源](https://www.zhihu.com/tardis/bd/art/651914200)  

![正面](https://pic3.zhimg.com/v2-251c8c7b96949f2d045db157e46a5106_b.webp)

![背面](https://pic2.zhimg.com/v2-d2e7c3084966a9a5e65cc08c6884f3c9_b.webp)

根据官方提供的参数，主机的尺寸为 126x113x42mm  
整机很轻便，要是口袋大点的甚至可以塞口袋里  

### 参数
I/O接口方面  
前面板：
- CLR CMOS键一个
- USB 3.2 Gen 1三个
    - Type-A两个
    - Type-C一个(支持DP视频输出，PD供电，但不支持反向供电)
- 极为先进的3.5mm TRRS一个
- 电源键兼状态指示灯一个  

后置：
- 1Gbps RJ45一个
- USB 3.2 Gen 1 Type-A一个
- USB 2.0 Type-A一个
- DP一个
- HDMI一个
- 19V DC-IN一个
- 我自己装的SFF-8612 (OCuLink)一个  

主机内部：
- SATA 3.0一个
- PCIe 3.0 x4 M.2 2280一个
- PCIe 2.0 x1 M.2 2230一个
- SO-DIMM DDR4两个

详细配置如表  

硬件 | 型号
:----- | :-----
CPU | AMD Ryzen 7 5800H
GPU | AMD Radeon Vega 8 & NVIDIA Tesla M40
RAM | Gloway DDR4-2666 16G
硬盘 | Zhitai SC001 Active 1T
声卡 | Realtek ALC897
有线网卡 | Realtek RTL8168
无线网卡 | Intel AX200

## 使用体验
做工很棒，拆开机器里面可以说是赏心悦目，仿佛一个工艺品，比我之前的天钡MN5X好很多  
到手之后插上硬盘和内存秒开，没有奇奇怪怪的内存兼容性问题  
很小巧，买个便携屏可以带着到处跑  
  
性能的话，将PBO三项拉满并将Scalar调整至10x后满载可以短暂的跑上全核4.1Ghz，但坚持不了几秒钟就会降频到3.6-3.7Ghz，CPU-Z的分数变化大概就是从5950+降到5280+  
试了下用核显来打CHUNITHM LMN，窗口化遇到特效复杂的地方会掉帧挺严重，其他游戏待测试  
BIOS自由度很高，能开放的基本都开放了，无需使用UniversalAMDFormBrowser设置  
  
风扇比较安静，没有负载的情况下放在身边睡觉都OK，不过满载噪音还是很大的（毕竟AMD祖传积热问题） 

外接M40之后性能有了质的飞跃，改了温控之后声音表现也很棒了，完全可以开着过夜  
**To do...**

## 总结
整体的使用体验还是不错的，日常办公写文章写代码玩Gal完全够用 ~~(废话)~~  
现正使用OCuLink外接M40，游戏性能完全过关了