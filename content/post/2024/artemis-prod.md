---
date: 2024-01-20T13:31:12.271Z
updated: 2024-07-09T05:17:42.578Z
title: Artemis反向代理配置参考
categories: 开发
tags:
- Software
- Dev
- 音游
- 教程
---

# Artemis反向代理配置参考


## 前言
如果你想要将 Artemis 放在服务器上使用的话，那应该很需要反向代理  
毕竟你也不想为了它而牺牲服务器的 80/443 端口吧  

## 需要的东西
- 一颗清醒的大脑
- 一个已经解析到服务器的域名
- 一台安装了 Nginx/Caddy 的服务器

## Artemis 配置
为了能让反代后的 Artemis 也能正常工作，需要对 ```core.yaml``` 作出以下修改

```yaml
server:
  listen_address: "127.0.0.1"
  hostname: "aime.example.com" #修改为你要使用域名
  is_develop: False
  is_using_proxy: True
  proxy_port: 80
  proxy_port_ssl: 443
allnet:
  standalone: False
billing:
  standalone: False
aimedb:
  listen_address: "0.0.0.0"
```

完整的配置文件如下  

<details>
core.yaml

```yaml
server:
  listen_address: "127.0.0.1"
  hostname: "aime.example.com" #修改为你要使用域名
  port: 8088
  ssl_key: "cert/title.key"
  ssl_cert: "cert/title.crt"
  allow_user_registration: True
  allow_unregistered_serials: True
  name: "ARTEMiS"
  is_develop: False
  is_using_proxy: True
  proxy_port: 80
  proxy_port_ssl: 443
  log_dir: "logs"
  check_arcade_ip: False
  strict_ip_checking: False

title:
  loglevel: "info"
  reboot_start_time: ""
  reboot_end_time : ""

database:
  host: "127.0.0.1"
  username: "aime"
  password: "password"
  name: "aime"
  port: 3306
  protocol: "mysql"
  sha2_password: False
  loglevel: "info"
  enable_memcached: True
  memcached_host: "localhost"

frontend:
  enable: False
  port: 8089
  loglevel: "info"
  secret: ""

allnet:
  standalone: False
  port: 80
  loglevel: "info"
  allow_online_updates: False
  update_cfg_folder: ""

billing:
  standalone: False
  loglevel: "info"
  port: 8443
  ssl_key: "cert/server.key"
  ssl_cert: "cert/server.pem"
  signing_key: "cert/billing.key"

aimedb:
  enable: True
  listen_address: "0.0.0.0"
  loglevel: "info"
  port: 22345
  key: "Copyright(C)SEGA"
  id_secret: ""
  id_lifetime_seconds: 86400

mucha:
  loglevel: "info"
```
  
</details>

## Nginx 配置
如果你需要 SSL Title Server 或者游戏没有打补丁的话  
那么你需要一个有效的 SSL 证书  
可以使用 acme/certbot 自动申请续签

### ALL.Net

::: warning
请使用 **naominet.jp** 作为 server_name  
否则网络自检将会 NG
:::

```nginx
server {
  listen 80;
  server_name naominet.jp;
	
  location / {
  	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  	proxy_pass_request_headers on;
  	proxy_pass http://localhost:8088/;
  }
}
```

### Title
```nginx
server {
  listen 80;
  server_name aime.example.com; #你在 core.yaml 里设置的 hostname

  location / {
  	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  	proxy_pass_request_headers on;
  	proxy_pass http://localhost:8088/;
  }
}

#如果无需SSL Title可删除下面这段
server {
  listen 443 ssl;
  server_name aime.example.com;

  ssl_certificate /path/to/your/certificate; #证书路径
  ssl_certificate_key /path/to/your/privatekey; #私钥路径
  ssl_session_timeout 1d;
  ssl_session_cache shared:MozSSL:10m;
  ssl_session_tickets off;

  ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
  ssl_ciphers "ALL:@SECLEVEL=0";
  ssl_prefer_server_ciphers off;

  location / {
  	proxy_pass http://localhost:8088/;
  }
}
```

### Billing

```nginx
server {
  listen 8443 ssl;	
  server_name ib.naominet.jp;
    
  ssl_certificate /path/to/your/certificate; #证书路径
  ssl_certificate_key /path/to/your/privatekey; #私钥路径
  ssl_session_timeout 1d;
  ssl_session_cache shared:MozSSL:10m;
  ssl_session_tickets off;

  ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
  ssl_ciphers "ALL:@SECLEVEL=0";
  ssl_prefer_server_ciphers off;

  location / {
  	proxy_pass http://localhost:8088/;
  }
}
```

## Caddy 配置
参考Nginx配置写的  
~~还没测试过，按理来说应该能跑起来~~  

```caddyfile
# ALL.Net
naominet.jp:80 {
  reverse_proxy http://localhost:8088
}

# Title
aime.example.com:80 {
  reverse_proxy http://localhost:8088
}

# SSL Title
aime.example.com:443 {
  reverse_proxy http://localhost:8088
  tls /path/to/your/certificate /path/to/your/privatekey
}

# Billing
ib.naominet.jp:8443 {
  reverse_proxy http://localhost:8088
  tls /path/to/your/certificate /path/to/your/privatekey
}

```


## 游戏配置
在 ```segatools.ini``` 的 ```default=``` 后写上**你前面使用的域名** (如 aime.example.com)  
**并确保 ```allnet_accounting``` 处于关闭状态**