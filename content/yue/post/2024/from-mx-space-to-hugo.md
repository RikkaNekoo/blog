---
title: 从 Mix-Space 迁移至 Hugo
date: 2024-08-18T20:23:08+08:00
slug: from-mix-space-to-hugo
categories: 杂谈
tags:
    - 软件
    - 随笔
---

## 为什么？
起因是 2024/8/18 起床水 TG 群的时候突发奇想想把博客换成静态（  
虽然原先在用的 Mix-Space 很好用，Shiro 前端也很好看，但是占用毕竟还是太大了，速度也有点慢  
于是便光速翻了一遍，最终敲定下来使用 Hugo+Stack 主题这一套方案

## 过程
由于我是完完全全的前端废物，所以几乎全都是照葫芦画瓢   
不过我并没有选择比较普遍的 Github Pages/Vercel 部署，反正手上有台年付的阿里云 HK 轻量，不用也是浪费了  
~~用服务器部署静态博客的我一定是有什么大病~~
网页服务器用的是 Caddy，配置简单不用管 TLS 还支持 H/3  
文章直接从 Mix-Space 后台下载 md 稍微改改 front-matter 就行了，不过笔记(类似于推文/朋友圈？)还不知道扔哪里比较好

## 最后
又水了一篇文章啦  
之后应该还会继续完善的，如果有空学了前端说不定还能爆改一番x

## 部署参考
[带着Stack主题入坑Hugo](https://blog.linsnow.cn/p/join-hugo-and-stack/)  
[使用hugo stack主题快速搭建博客](https://www.liuhouliang.com/post/hugo_theme/)  
[使用Hugo部署博客以及Stack主题的美化](https://vofficial233.com/archives/deploy-my-hugo-blog)  
[Stack官方文档](https://stack.jimmycai.com/)  
[Hugo官方文档](https://gohugo.io/)