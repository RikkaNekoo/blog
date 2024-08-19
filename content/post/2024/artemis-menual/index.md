---
date: 2024-01-07T01:35:33.008Z
title: Artemis不完全食用指南
slug: artemis-manual
categories: 开发
tags:
  - Dev
  - 教程
  - 音游
---

## 介绍

Artemis 没什么好介绍的了吧，给 SEGA 系 HDD 用的本地服，算是 Aqua 的继承者？  

> A network service emulator for games running SEGA'S ALL.NET service, and similar.

Artemis 更新真的很快，我的更新速度可能跟不上  
如果内容失效的话，评论区或者邮箱发我，看到会处理的）


## Windows

### 准备

所需要的东西有  

- Windows 10 或以上系统  
- Python  
- MariaDB  
- 良好的网络连接  
- 清醒的大脑  

### 安装 Python

Python 的安装就不多赘述了，一搜一大把，建议使用 3.11  
记得勾上 PATH

### 安装MariaDB 11
安装过程略  
在开始里找到 MySQL Client 打开登录  
逐行输入以下命令，`<Enter Password Here>`改为你想设置的密码  

```sql
CREATE USER 'aime'@'localhost' IDENTIFIED BY '<Enter Password Here>';
CREATE DATABASE aime;
GRANT Alter,Create,Delete,Drop,Index,Insert,References,Select,Update ON aime.* TO 'aime'@'localhost';
FLUSH PRIVILEGES;
exit;
```


### 下载 Artemis

有两种方式可选
直接下载[Artemis-develop](https://gitea.tendokyu.moe/Hay1tsme/artemis/archive/develop.zip)后解压  
或者使用 git (推荐，方便更新)  

```powershell
git clone https://gitea.tendokyu.moe/Hay1tsme/artemis.git -b develop
```

### 安装 Python 模块

在 Artemis 文件夹内打开 powershell，执行

```powershell
pip install -r requirements.txt
```

### 配置 Artemis

#### 将 example_config 文件夹改名为 config

#### 编辑配置文件
config/core.yaml:  

```yaml
server:
  listen_address: 0.0.0.0
database:
  password: "你之前设置的密码"
aimedb:
  key: "Copyright(C)SEGA"
```

如果你不需要游玩头文字 D 的话，可以在 idz.yaml 中将其关闭

#### 配置数据库

```powershell
python dbutils.py create
```

### Artemis，启动！

到这里，Artemis 的基础配置已经完成了  
使用

```powershell
python index.py
```
启动试试吧，如果一切正常，你将会看到会看到类似  

![Artemis](artemis.webp)  

的输出
## Linux

实际上没什么好讲的  
装个 MySQL，装个 Memcached，然后参考 Windows 的流程就好了  

## 游戏针对性设置

> [!NOTE]
> 除 **Chinithm** 外均未测试，不保证可用性  
> ~~如果你有资源的话欢迎发我测试~~

### Chunithm

本文假定你游玩的是 **Chunithm Sun Plus (2.16)** 以上版本  
如果你仍在游玩 Sun 及以下版本，请使用[AquaDX](https://github.com/hykilpikonna/AquaDX)

#### 导入资源

在 Artemis 目录下执行

```powershell
python read.py --game SDBT --version 14 --binfolder <data的路径> --optfolder <opt的路径>
```

坐和放宽，等待导入完成

#### 修改配置文件

编辑 config/chuni.yaml:

(P.S: 下方ROM和Data版本号视情况修改，当然不改也没关系)

```yaml
team:
  name: ARTEMiS # 默认队伍名
version:
  14:
    rom: 2.16.00
    data: 2.15.11
```


#### 完成

在 segatools.ini 里的 default= 填上你的**局域网 IP 地址**  

> [!NOTE]
> 不要使用 localhost 和 127.0.0.1  
> 否则 ALL.Net 会 NG

开始享受你的次新次热吧

## FAQ

此处收录常见问题，如果你遇到了可以发我）

### ALL.Net Authentication BAD

- 请检查游戏目录下 config_common.json 中 allnet_auth 是否为 2.0，如果是，改为 1.0
- 依然是 config_common.json ，检查 allnet_accounting 是否打开，如果是，关掉它

### Title BAD

- 如果你是服务器运行的话 config/core.yaml 中的 hostname 改为服务器 IP/域名，本地运行则为 localhost

### 全 GOOD 但是灰网

- 检查 amfs 下的两个 ICF 是否正确
- 请不要使用**中文**目录