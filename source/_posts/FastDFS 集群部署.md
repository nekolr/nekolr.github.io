---
title: FastDFS 集群部署
date: 2018/10/27 12:09:0
tags: [FastDFS]
categories: [FastDFS]
---
关于 FastDFS 的介绍就先不说了，直接上干货。  

<!--more-->  

# 环境准备
- 两台 Tracker（10.170.0.6,10.170.0.7），四台 Storage（10.170.0.2,10.170.0.3,10.170.0.4,10.170.0.5）  
- 统一系统环境为 CentOS Linux release 7.5.1804  

# 架构示意图
![架构图](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/10/28/Y7P.png)

# 安装 FastDFS
需要在所有的环境上都安装 FastDFS。  

首先需要安装依赖包。  

```
yum -y install gcc gcc-c++ libstdc++-devel pcre-devel zlib-devel wget make
yum -y groupinstall 'Development Tools'
```

安装 libfastcommon。FastDFS 5.x 取消了对 libevent 的依赖，添加了对 libfastcommon 的依赖。这里选择直接克隆源代码项目，保证使用的代码都是最新的。  

```
git clone https://github.com/happyfish100/libfastcommon.git /app/libfastcommon
cd /app/libfastcommon
./make.sh
./make.sh install
```

安装 FastDFS。这里同样克隆最新的源代码编译安装。  

```
git clone https://github.com/happyfish100/fastdfs.git /app/fastdfs
cd /app/fastdfs
./make.sh
./make.sh install
```

安装完成之后，可以在 /usr/bin/ 下看到一些以 fdfs 开头的命令。  

# 配置 Tracker
两台 Tracker 的配置过程相同，首先修改 tracker.conf 配置文件。  

```
:: 首先创建存储日志和数据的根目录
mkdir -p /app/nekolr/fastdfs

vi /app/fastdfs/conf/tracker.conf
```

这里重点只需要修改 base_path （创建存储日志和数据的根目录）即可。  

```
base_path=/app/nekolr/fastdfs
```

同时配置文件中还有几个参数，可以根据需要调整。  

```
# 是否启用配置文件，false 表示启用
disabled=false
# tracker 的端口号
port=22122
# tracker 的数据文件和日志目录，这个目录需要手动创建
base_path=/app/nekolr/fastdfs
# http 服务端口号
http.server_port=9090
```

配置完毕后，就可以通过配置文件来启动 tracker 了。  

```
/usr/bin/fdfs_trackerd /app/fastdfs/conf/tracker.conf start
```

如果没有报错，则可以查看 22122 端口是否被监听来确认启动成功。  

```
# ps -ef|grep fdfs

root     14395     1  0 05:40 ?        00:00:00 /usr/bin/fdfs_trackerd /app/fastdfs/conf/tracker.conf start
root     14411  1438  0 05:41 pts/0    00:00:00 grep --color=auto fdfs

# netstat -tunlp|grep fdfs

tcp        0      0 0.0.0.0:22122           0.0.0.0:*               LISTEN      14395/fdfs_trackerd
```

也可以通过日志查看启动是否成功。  

```
# cat /app/nekolr/fastdfs/logs/trackerd.log

INFO - FastDFS v5.11, base_path=/app/nekolr/fastdfs, run_by_group=,...
```

# 配置 Storage
将四台存储节点分成两组，其中 group1 为 10.170.0.2 和 10.170.0.3，group2 为 10.170.0.4 和 10.170.0.5。  

四台 Storage 的配置也是类似的，首先修改 storage.conf 配置文件。  

```
:: 首先创建存储日志和数据的根目录
mkdir -p /app/nekolr/fastdfs

vi /app/fastdfs/conf/storage.conf
```

主要修改 base_path、store_path、group_name 和 tracker_server。  

```
# 组名（第一组为 group1，第二组为 group2，依次类推）
group_name=group1
# 数据和日志文件存储根目录
base_path=/app/nekolr/fastdfs
# 第一个存储目录，第二个存储目录起名为：store_path1=xxx，其它存储目录名依次类推
store_path0=/app/nekolr/fastdfs
# 存储路径个数，需要和 store_path 个数匹配
store_path_count=1
# tracker 服务器 IP 和端口，有几个 tracker 就要配置几个
tracker_server=10.170.0.6:22122
tracker_server=10.170.0.7:22122
```

配置完成后，就可以通过配置来启动了。  

```
/usr/bin/fdfs_storaged /app/fastdfs/conf/storage.conf start
```

与 tracker 相同，可以通过查看端口监听或者日志来确认启动是否成功。如果启动成功，可以验证一下 storage 是否登记到了 tracker 上。  

```
:: 该命令可以在任意一台 Storage 上使用
# /usr/bin/fdfs_monitor /app/fastdfs/conf/storage.conf

server_count=2, server_index=1
tracker server is 10.170.0.7:22122
group count: 1

Group 1:
group name = group1
disk total space = 10229 MB
disk free space = 7712 MB
trunk free space = 0 MB
storage server count = 1
active server count = 1
storage server port = 23000
storage HTTP port = 8888
store path count = 1
subdir count per path = 256
current write server index = 0
current trunk file id = 0

        Storage 1:
                id = 10.170.0.2
                ip_addr = 10.170.0.2 (common1.c.avalon-192611.internal)  ACTIVE
                http domain =
                version = 5.12
                join time = 2018-10-26 07:40:54
                up time = 2018-10-27 06:08:42
                ...
```

可以看到 Storage 的状态为 active，说明 Storage 服务器已经成功登记到 Tracker 服务器上了。  

# 在 Storage 上安装 nginx
> 注意：需要在所有的 Storage 上都按照下面的步骤操作。  

FastDFS 将 Storage 服务器分组，相同组下的存储服务器之间需要进行文件的同步，这会有同步延迟的问题。假如客户端向 Tracker 服务器发送上传文件指令，Tracker 服务器经过选择，将文件上传到了 10.170.0.2，上传成功后将文件的 ID 返回给了客户端，此时 FastDFS 会将这个文件同步到同组的 10.170.0.3，在文件还没有复制完成的情况下，客户端如果使用这个 ID 在 10.170.0.3 上取文件就会出现无法访问的错误。fastdfs-nginx-module 模块就是为了解决这个问题的，它可以将文件链接重定向到源服务器，避免由于复制延迟导致的客户端无法访问文件的错误。  

首先获取 fastdfs-nginx-module 和 nginx，如果要开启 https，nginx 需要依赖 openssl 库。  

```
git clone https://github.com/happyfish100/fastdfs-nginx-module.git /app/fastdfs-nginx-module

wget -P /app http://nginx.org/download/nginx-1.15.5.tar.gz
wget -P /app https://www.openssl.org/source/openssl-1.1.1.tar.gz

tar -zxvf /app/nginx-1.15.5.tar.gz -C /app
tar -zxvf /app/openssl-1.1.1.tar.gz -C /app
```

接下来开始安装 nginx，需要添加 fastdfs-nginx-module 模块。  

```
cd /app/nginx-1.15.5
./configure --add-module=../fastdfs-nginx-module/src
make && make install
```

nginx 安装完成后，查看版本以及模块安装情况。  

```
# /usr/local/nginx/sbin/nginx -V

nginx version: nginx/1.15.5
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-28) (GCC)
configure arguments: --add-module=../fastdfs-nginx-module/src
```

接下来需要配置 fastdfs-nginx-module，修改 md_fastdfs.conf 文件。  

```
:: 将 mod_fastdfs.conf 配置文件复制到 /etc/fdfs/ 下
# cp /app/fastdfs-nginx-module/src/mod_fastdfs.conf /etc/fdfs/

:: 编辑配置文件
# vi /etc/fdfs/mod_fastdfs.conf
```

```
# 保存日志的目录
base_path=/app/nekolr/fastdfs
# tracker 服务器
tracker_server=10.170.0.6:22122
tracker_server=10.170.0.7:22122
# storage 服务器的端口号
storage_server_port=23000
# 当前服务器的 group 名
group_name=group1
# 文件 url 中是否有 group 名
url_have_group_name = true
# 存储路径个数，需要和 store_path 个数匹配
store_path_count=1
# 存储路径
store_path0=/app/nekolr/fastdfs
# 设置组的个数
group_count = 2
```

再在配置文件的末尾增加两个组的具体信息。  

```
[group1]
group_name=group1
storage_server_port=23000
store_path_count=1
store_path0=/app/nekolr/fastdfs

[group2]
group_name=group2
storage_server_port=23000
store_path_count=1
store_path0=/app/nekolr/fastdfs
```

建立 M00 至存储目录的符号连接。  

```
ln -s /app/nekolr/fastdfs/data /app/nekolr/fastdfs/data/M00
ll /app/nekolr/fastdfs/data/M00
```

接下来再配置 nginx，修改 nginx.conf 配置文件。  

```
:: 修改配置文件
# vi /usr/local/nginx/conf/nginx.conf

:: 下面是需要修改的地方的配置
user root;
server {
    listen       8080;
    location ~/group[1-2]/M00 {
        root /app/nekolr/fastdfs/data;
        ngx_fastdfs_module;
    }
}
```

```
:: 复制 fastdfs 中的 http.conf、mime.types 文件到 /etc/fdfs
cp /app/fastdfs/conf/http.conf /app/fastdfs/conf/mime.types  /etc/fdfs
```

配置完成后，接下来将添加防火墙的放行策略，启动 nginx。  

```
/usr/local/nginx/sbin/nginx
```

# 在 Tracker 上安装 nginx
> 注意：需要在所有的 Tracker 上都按照下面的步骤操作。  

在 Tracker 上安装 nginx 主要是为了提供 http 访问的反向代理、负载均衡以及缓存服务。  

首先安装 nginx。  

```
wget -P /app http://nginx.org/download/nginx-1.15.5.tar.gz
wget -P /app https://www.openssl.org/source/openssl-1.1.1.tar.gz

tar -zxvf /app/nginx-1.15.5.tar.gz -C /app
tar -zxvf /app/openssl-1.1.1.tar.gz -C /app

cd /app/nginx-1.15.5
./configure --prefix=/usr/local/nginx
make && make install

:: 修改配置文件
vi /usr/local/nginx/conf/nginx.conf
```

配置反向代理、负载均衡。  

```
events {
    # 最大连接数
    worker_connections  65535;
    # 新版本的 Linux 可使用 epoll 加快处理性能
    use epoll;
}
http {
    # 设置 group1 的服务器
    upstream fdfs_group1 {
        server 10.170.0.2:8080 weight=1 max_fails=2 fail_timeout=30s;
        server 10.170.0.3:8080 weight=1 max_fails=2 fail_timeout=30s;
    }
    # 设置 group2 的服务器
    upstream fdfs_group2 {
        server 10.170.0.4:8080 weight=1 max_fails=2 fail_timeout=30s;
        server 10.170.0.5:8080 weight=1 max_fails=2 fail_timeout=30s;
    }

    server {
        # 设置服务器端口
        listen       8080;
        # 设置 group1 的负载均衡参数
        location /group1/M00 {
            proxy_pass http://fdfs_group1;
        }
        # 设置 group2 的负载均衡参数
        location /group2/M00 {
            proxy_pass http://fdfs_group2;
        }
    }

}
```

配置完成后，启动 nginx。  

```
# /usr/local/nginx/sbin/nginx

:: 查看监听
# netstat -tunlp | grep nginx

tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      7328/nginx: master
```

# 上传测试
配置完成后，集群的情况可以在任意一台 Storage 上使用命令查看。  

```
# /usr/bin/fdfs_monitor /app/fastdfs/conf/storage.conf

server_count=2, server_index=1
tracker server is 10.170.0.7:22122
group count: 2

Group 1:
group name = group1
disk total space = 10229 MB
disk free space = 7711 MB
trunk free space = 0 MB
storage server count = 2
active server count = 2
storage server port = 23000
storage HTTP port = 8888
store path count = 1
subdir count per path = 256
current write server index = 0
current trunk file id = 0

        Storage 1:
                id = 10.170.0.2
                ip_addr = 10.170.0.2 (common1.c.avalon-192611.internal)  ACTIVE
                http domain =
                version = 5.12
                ...
        Storage 2:
                id = 10.170.0.3
                ip_addr = 10.170.0.3 (common2.c.avalon-192611.internal)  ACTIVE
                http domain =
                version = 5.12
                ...
Group 2:
group name = group2
disk total space = 10229 MB
disk free space = 8164 MB
trunk free space = 0 MB
storage server count = 2
active server count = 1
storage server port = 23000
storage HTTP port = 8888
store path count = 1
subdir count per path = 256
current write server index = 0
current trunk file id = 0

        Storage 1:
                id = 10.170.0.4
                ip_addr = 10.170.0.4 (common3.c.avalon-192611.internal)  ACTIVE
                http domain =
                version = 5.12
                ...
        Storage 2:
                id = 10.170.0.5
                ip_addr = 10.170.0.5 (common4.c.avalon-192611.internal)  ACTIVE
                http domain =
                version = 5.12
                ...
```

此时可以通过修改某个 Tracker 上的客户端 client.conf 配置，然后再通过上传文件命令来测试。  

```
cp /app/fastdfs/conf/client.conf  /etc/fdfs

:: 修改客户端配置
# vi /etc/fdfs/client.conf

# 日志存放路径
base_path=/app/nekolr/fastdfs
# tracker 服务器
tracker_server=192.168.53.85:22122         
tracker_server=192.168.53.86:22122 
http.tracker_server_port=8080
```

使用命令上传文件。  

```
:: 上传文件
# /usr/bin/fdfs_upload_file /etc/fdfs/client.conf /app/favicon.png

:: 返回的地址
group2/M00/00/00/CqoABFvUnLmAAhVgAAGMpR-vGEI079.png
```

接下来就可以通过任意一台 Tracker 来访问该文件了。这里使用 `10.170.0.6` 这台来访问：<http://10.170.0.6:8080/group2/M00/00/00/CqoABFvUnLmAAhVgAAGMpR-vGEI079.png>  

> 这里直接使用内网地址来访问的，如果有公网地址，可以在任意能够连接互联网的地方通过 Tracker 的公网 IP 来访问。  

除了在 Tracker 上直接使用指令上传文件，还可以通过 FastDFS 提供的客户端来上传文件，目前支持 PHP 和 Java。  

# 参考
> [FastDFS 集群 安装 配置](http://www.ityouknow.com/fastdfs/2017/10/10/cluster-building-fastdfs.html)