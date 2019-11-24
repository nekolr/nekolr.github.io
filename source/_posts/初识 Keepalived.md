---
title: 初识 Keepalived
date: 2019/8/16 11:47:0
tags: [Keepalived]
categories: [集群]
---
在初步了解了 keepalived 的作用后，手痒难耐，于是动手实践部署了一个 Nginx + keepalived + Tomcat 组合的双主高可用负载均衡集群。

<!--more-->

# 实现 VIP 漂移
在搭建集群之前，先简单使用 keepalived 实现 VIP 漂移。准备两台服务器（`10.5.96.3` 和 `10.5.96.4`），两台机器都要安装 keepalived。

```bash
# 安装依赖
yum install -y openssl openssl-devel gcc gcc-c++

# 下载解压 keepalived
wget https://www.keepalived.org/software/keepalived-2.0.18.tar.gz && tar xf keepalived-2.0.18.tar.gz

# 编译安装
mkdir /usr/local/keepalived && cd keepalived-2.0.18 && ./configure prefix=/usr/local/keepalived/
make && make install

# 复制配置文件
mkdir /etc/keepalived && cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/
cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/keepalived
```

接下来需要修改 keepalived 的配置文件，我选择将 `10.5.96.3` 作为主，`10.5.96.4` 作为备，虚拟 IP 地址设置成 `10.5.96.7`。主的配置为：

```conf
! Configuration File for keepalived

# 实例配置
vrrp_instance VI_1 {
    # 指定为 master 主机
    state MASTER
    # VIP 绑定的网卡设备，用来发送 VRRP 包做心跳检测
    interface eth1
    # 虚拟路由 id，是一个唯一值，取值 0 到 255 之间
    # 相同的组（主备）之间配置须一致，不同的组需要设置不同的值
    virtual_router_id 51
    # 优先级，在选举 master 时会使用该值，如果要成为 master，这个选项的值最好高于其他机器 50 个点
    priority 100
    # 健康检查时间间隔，即每隔 1 秒发起一次 VRRP 组播
    advert_int 1
    # 虚拟 IP 地址
    virtual_ipaddress {
        10.5.96.7
    }
}
```

备的配置为：

```conf
! Configuration File for keepalived

# 实例配置
vrrp_instance VI_1 {
    # 指定为 backup 主机
    state BACKUP
    # VIP 绑定的网卡设备，用来发送 VRRP 包做心跳检测
    interface eth1
    # 虚拟路由 id，是一个唯一值，取值 0 到 255 之间
    # 相同的组（主备）之间配置须一致，不同的组需要设置不同的值
    virtual_router_id 51
    # 优先级，在选举 master 时会使用该值，如果要成为 master，这个选项的值最好高于其他机器 50 个点
    priority 50
    # 健康检查时间间隔，即每隔 1 秒发起一次 VRRP 组播
    advert_int 1
    # 虚拟 IP 地址
    virtual_ipaddress {
        10.5.96.7
    }
}
```

接下来启动两台机器分别使用命令 `systemctl start keepalived` 启动 keepalived，然后使用 `ip ad` 命令观察网络配置情况。

Master 的网络配置：

![Master](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242036/2019/08/16/kxY.png)

Backup 的网络配置：

![Backup](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242036/2019/08/16/5Lk.png)

接下来停掉 Master 的 keepalived，再观察 Backup 的网络配置，会发现 VIP 漂移到了该机器上。然后重新启动 Master 的 keepalived，再次观察，会发现 VIP 又重新漂移到了主机器上。

## 可能会遇到的问题
主备同时获取到了 VIP 地址，也就是脑裂现象。keepalived 默认需要使用 D 类多播地址 `224.0.0.18` 进行心跳通信，因此需要修改防火墙的规则：

```bash
# 添加规则
firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT 0 --in-interface eth0 --destination 224.0.0.18 --protocol vrrp -j ACCEPT
firewall-cmd --direct --permanent --add-rule ipv4 filter OUTPUT 0 --out-interface eth0 --destination 224.0.0.18 --protocol vrrp -j ACCEPT
# 重载配置
firewall-cmd --reload
# 重启 keepalived
systemctl restart keepalived
```

# 实现双主高可用负载均衡集群
双机高可用一般分为两种。一种是主从模式，即一台机器为主，另一台机器为从。正常情况下，主服务器绑定一个公网虚拟 IP，作为流量入口提供负载均衡服务，而热备服务器则处于空闲状态。当主服务器发生故障时，VIP 发生漂移，热备服务器接管主服务器的公网虚拟 IP，继续提供负载均衡服务。这种方式永远会有一台服务器处于空间状态，浪费资源。另一种就是双主模式，即两台服务器互为主备，同时处于活动状态，各自绑定一个公网虚拟 IP。当其中一台发生故障时，另一台接管发生故障的服务器的公网虚拟 IP，承担所有的请求。

> 公网虚拟 IP 其实就是公网 IP，只不过我们将它用作 VIP。很多云服务厂商提供的弹性公网 IP 服务就可以满足我们的需求。

接下来一共需要四台服务器，其中两台（`10.5.96.3` 和 `10.5.96.4`）使用 Nginx 做负载均衡，另外两台（`10.5.96.5` 和 `10.5.96.6`）作为真实节点服务器，运行 Tomcat 容器，两个 VIP 分别设置为 `10.5.96.7` 和 `10.5.96.8`。首先在两台真实节点服务器上安装 JDK 和 Tomcat，安装完毕后简单编写一个测试的 jsp 页面。然后在两台负载均衡服务器上安装 Nginx 并添加配置如下：

```conf
#gzip  on;
upstream tomcat {
    server 10.5.96.5:8080 weight=1 max_fails=1 fail_timeout=10s;
    server 10.5.96.6:8080 weight=1 max_fails=1 fail_timeout=10s;
}

server {
    listen       80;
    server_name  localhost;
    location / {
        root   html;
        index  index.html index.htm;
        proxy_pass http://tomcat;
    }
}
```

接下来需要在两台负载均衡机器上分别安装 keepalived，用于实现故障切换，`10.5.96.3` 的配置文件如下：

```conf
! Configuration File for keepalived

vrrp_script check_nginx {
    script "/root/nginx.sh"
    interval 2
    weight 10
}

vrrp_instance VI_1 {
    state MASTER
    interface eth1
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }

    track_script {
        check_nginx
    }

    virtual_ipaddress {
        10.5.96.7
    }
}

vrrp_instance VI_2 {
    state BACKUP
    interface eth1
    virtual_router_id 52
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }

    track_script {
        check_nginx
    }

    virtual_ipaddress {
        10.5.96.8
    }
}
```

`10.5.96.4` 的配置文件如下：

```conf
! Configuration File for keepalived

vrrp_script check_nginx {
    script "/root/nginx.sh"
    interval 2
    weight 10
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth1
    virtual_router_id 51
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }

    track_script {
        check_nginx
    }

    virtual_ipaddress {
        10.5.96.7
    }
}

vrrp_instance VI_2 {
    state MASTER
    interface eth1
    virtual_router_id 52
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }

    track_script {
        check_nginx
    }

    virtual_ipaddress {
        10.5.96.8
    }
}
```

其中 `nginx.sh` 是 Nginx 进程状态检测脚本。设置该脚本后，keepalived 会定期执行该脚本，在 Nginx 服务没有启动或者出现问题关闭时重新启动 Nginx 服务，在重启失败后会关闭该机器上的 keepalived 服务，从而导致 VIP 漂移，实现故障切换。

> keepalived 会根据脚本执行的结果动态调整 vrrp_instance 的优先级（可能是降低或者提高），从而引发新一轮的 Master 选举。

```bash
#!/bin/bash
# 检测进程脚本
jc=`ps -C nginx --no-header|wc -l`
if [ $jc -eq 0 ];then
    /usr/local/nginx/sbin/nginx
    sleep 2
    # 如果尝试启动 nginx 失败，则关闭 keepalived 服务，进行 VIP 漂移
    jc2=`ps -C nginx -no-header|wc -l`
    if [ $jc2 -eq 0 ];then
        systemctl stop keepalived
    fi
fi
```

一切准备就绪后，分别启动两台机器上的 Nginx 服务和 keepalived 服务。查看网络配置情况，其中 `10.5.96.3` 的配置如下：

![3号机器配置](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242036/2019/08/18/E3E.png)

`10.5.96.4` 的配置如下：

![4号机器配置](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242036/2019/08/18/eyg.png)

当某一台机器发生故障时，VIP 漂移，另一个机器就会同时接管这两个 VIP，当机器故障恢复后，VIP 又会漂移回去，恢复到一开始的配置。