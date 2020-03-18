---
title: nginx 反代 frp
date: 2019/4/22 14:46:0
tags: [NAT]
categories: [其他]
---
最近需要用到内网穿透服务，突然想到手头正好有一台主机挺闲的，用来搭内网穿透服务再合适不过了。于是在网上搜寻一番，发现使用较多的是 frp 和 ngrok，但是 ngrok 的源代码只停留在 1.x，2.x 版本以及更新的服务只能通过注册使用，于是放弃了它转而使用 frp。  

<!--more-->

服务端和客户端同时安装 frp 程序，安装和配置的过程都比较简单。在对应的平台下载对应的 release 包后，修改配置文件，启动即可。由于这台主机对外只开放了 80 和 443 端口，使用 nginx 提供反向代理，隐藏了内部的各个系统，同样的，我需要使用 nginx 反代 frp 的服务。  

![大体流程图](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242036/2019/06/18/xGr.png)

```conf
# nginx.conf
server {
    listen       443 ssl http2;
    server_name  nat.nekolr.com;
    
    ssl_protocols       TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_certificate      /etc/letsencrypt/live/nekolr.com/fullchain.pem;
    ssl_certificate_key  /etc/letsencrypt/live/nekolr.com/privkey.pem;
    ssl_trusted_certificate  /etc/letsencrypt/live/nekolr.com/chain.pem;
    
    ssl_session_cache    shared:SSL:1m;
    ssl_session_timeout  5m;

    ssl_ciphers  HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers  on;
    
    location / {
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Host $host;
            proxy_pass http://localhost:6081;
    }
    error_page  404              /404.html;
}
```

```ini
# frps.ini
[common]
bind_port = 7000
# 公网环境中主机对外提供 web 服务端口
vhost_http_port = 6081
log_file = ./frps.log
log_level = info
log_max_days = 3
```

```ini
# frpc.ini
[common]
server_addr = 公网环境中的主机 IP
server_port = 7000

[web]
type = http
# 内网环境中主机 web 服务端口
local_port = 8080
custom_domains = nat.nekolr.com
```