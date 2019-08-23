---
title: Docker 入门
date: 2017/7/23 15:46:0
tags: [Docker]
categories: [Docker]
---
## Docker 是什么？
直接贴上官网的一句介绍：		
> Docker is the world’s leading software container platform		
		
Docker 是世界领先的软件容器平台。		
<!--more-->		
		
我们知道，如果想发布一个应用，需要将应用部署到服务器上。如果部署多个应用，应用依赖的环境可能会有所不同，不同的应用共用同一台服务器可能会出现冲突的现象，这个时候就需要隔离应用，最简单粗暴的方式就是使用虚拟机，但是虚拟机的开销比较大，这时 Docker 就派上用场了。  

Docker 与虚拟机不同，但是可以看做是一种轻量化的虚拟机。我们将在本地运行的应用和依赖的环境打包成 Docker 镜像，然后就可以在别的平台使用这个镜像的应用而不用担心环境部署等问题。但是使用虚拟机可以在一个 OS 中运行出多个不同或相同的 OS，而使用 Docker 只能在一个 OS 中模拟出多个相同的该 OS。  
	
![virtual machines](https://img.nekolr.com/images/2018/04/14/zj.png)
		
![docker](https://img.nekolr.com/images/2018/04/14/kx.png)

## Docker 解决了什么？
- **解决环境不一致问题**  
开发一个应用，开发环境与部署环境很可能是不一致的。开发者在本地运行正常的应用，交给运维实施人员部署时，可能会出现各种错误。而使用了 Docker 后，开发人员可以将应用连同开发环境一起打包到 Docker 镜像，运维实施人员只需要直接部署 Docker 即可。  

- **环境隔离**  
很多时候，我们的应用很有可能是和别的应用一起运行在一台服务器上，因此不可避免的会出现应用间相互影响的情况。一个应用将内存占满，另一个应用因内存不足挂掉。Docker 的隔离虽然不如虚拟机的隔离彻底，但是能够限制容器对系统资源的使用，并使容器间环境隔离，避免了应用间的互相影响。  

- **弹性伸缩、快速扩展**  
Docker 的这种将应用和环境一起打包的方式使得部署应用方便快捷，不用再为应用部署到多台服务器而头疼，通过标准统一的操作，能够迅速为几十台、上百台服务器部署应用。同时容器的销毁也很方便快捷，能够更好的应对弹性需求。		

## 关于 Docker 的一些概念
**LXC（Linux Container）**  
Linux 中的一种内核虚拟化技术，提供轻量级的虚拟化，以便隔离进程和资源。Docker 就是基于 LXC 的高级容器引擎。Docker 从 0.9 版本开始使用 libcontainer 替代 lxc。  

**UFS（Union File System 联合文件系统）**  
它是实现 Docker 镜像的技术基础，是一种轻量级的高性能分层文件系统，支持将文件系统中的修改进行提交和层层叠加，这个特性使得镜像可以通过分层实现和继承。同时支持将不同目录挂载到同一个虚拟文件系统下。  

**AUFS（Another Union File System）**  
也是一种联合文件系统，支持将不同目录挂载到同一个虚拟文件系统下（unite several directories into a single virtual filesystem）的文件系统形成一层 layer，更进一步地，AUFS 支持为每一个成员目录 (AKA branch) 设定只读 readonly，读写 readwrite 和 写出 whiteout-able 权限，对 read-only 目录只能读，而写操作只能在 read-write 目录中。写操作是在 read-only 上的一种增量操作，不影响 read-only 目录。当挂载目录的时候要严格按照各目录之间的这种增量关系，将被增量操作的目录优先于在它基础上增量操作的目录挂载，待所有目录挂载结束了，继续挂载一个 read-write 目录，如此便形成了一种层次结构。  

Docker 在启动容器的时候，需要创建文件系统，为 rootfs 提供挂载点。最初 Docker 仅能在支持 Aufs 文件系统的 Linux 发行版上运行，但是由于 Aufs 未能加入 Linux 内核，为了寻求兼容性、扩展性，Docker 在内部通过 graphdriver 机制这种可扩展的方式来实现对不同文件系统的支持。Docker 支持 AUFS、Device mapper、Btrfs、ZFS、ZFS 和 OverlayFS。  

典型的 Linux 文件系统由 bootfs 和 rootfs 两部分组成，bootfs（boot file system）主要包含 bootloader 和 kernel，bootloader 主要是引导加载 kernel，当 kernel 被加载到内存中后 bootfs 就被 umount 了。 rootfs（root file system）包含的就是典型的 Linux 系统中的 /dev，/proc，/bin，/etc 等标准目录和文件。	

![unionfs](https://img.nekolr.com/images/2018/04/14/Ob.png)
		
对于不同的 Linux 发行版，bootfs 基本是一致的，rootfs 会有差别，因此不同的发行版可以公用 bootfs。		
		
![unionfs_bootjs](https://img.nekolr.com/images/2018/04/14/rd.jpg)
		
**Docker Client**  
Docker 提供给用户的一个终端，用户输入 Docker 命令管理本地或者远程的服务器。  

**Docker Daemon**  
Docker 服务的守护进程。每台服务器（物理机或虚拟机）上只要安装了 Docker 的环境，基本上就跑了一个后台程序 Docker Daemon，Docker Daemon 接收 Docker Client 发过来的指令来对服务器进行具体操作。  

**Docker Images**  
Docker 镜像（Docker 容器的基础）。说白了镜像就是一系列的文件，只不过 Docker 的镜像使用的是联合文件系统。  

Docker 镜像的典型结构如下图。传统的 Linux 加载 bootfs 时会先将 rootfs 设为 read-only，然后在系统自检之后将 rootfs 从 read-only 改为 read-write，然后我们就可以在 rootfs 上进行写和读的操作了。但 Docker 的镜像却不是这样，它在 bootfs 自检完毕之后并不会把 rootfs 的 read-only 改为 read-write。而是利用 union mount（UFS 的一种挂载机制）将一个或多个 read-only 的 rootfs 加载到之前的 read-only 的 rootfs 层之上。在加载了这么多层的 rootfs 之后，仍然让它看起来只像是一个文件系统，在 Docker 的体系里把 union mount 的这些 read-only 的 rootfs 叫做 Docker 的镜像。但是，此时的每一层 rootfs 都是 read-only 的，我们此时还不能对其进行操作。当我们创建一个容器，也就是将 Docker 镜像进行实例化，系统会在一层或是多层 read-only 的 rootfs 之上分配一层空的 read-write 的 rootfs。  

![docker_image](https://img.nekolr.com/images/2018/04/14/Kl.png)
		
一组 Readonly 和一个 Readwrite 的结构构成一个 Container 的运行目录。得益于 AUFS 的特性，每一个对 readonly 层文件/目录的修改都只会存在于上层的 writeable 层中。这样由于不存在竞争，多个 container 可以共享 readonly 的 layer。所以 docker 将 readonly 的层称作“image”，对于 container 而言整个 rootfs 都是 read-write 的，但事实上所有的修改都写入最上层的 writeable 层中。
		
**Image 可以通过分层来继承，基于 Base Image（无父镜像）可以制作各种具体的应用镜像。Image 不保存用户状态，可以用于模板、重建和复制。**  

**Docker Registry**  
Docker Registry Server 就像 git 的仓库一样，提供镜像的存储和管理服务，它提供了 Docker 镜像的上传、下载和浏览等功能，并且提供安全的账号管理可以管理只有自己可见的私人 image。  

> Docker Hub：<https://hub.docker.com>（官方 Registry）

**Docker Container**  
Docker 的容器。Docker Container 是真正跑项目程序、消耗机器资源、提供服务的地方，Docker Container 通过 Docker Images 启动，在 Docker Images 的基础上运行你需要的代码。  

在 Docker 中，上层的 Image 依赖下层的 Image，因此 Docker 中把下层的 Image 称作父 Image，没有父 Image 的 Image 称作 Base Image。因此，想要从一个 Image 启动一个 Container，Docker 会逐次加载其父 Image 直到 Base Image，用户的进程运行在 Writeable 的层中。所有父 Image 中的数据信息以及 ID、网络和 LXC 管理的资源限制、具体 container 的配置等，构成一个 Docker 概念上的 Container。  

## 安装 Docker
Docker 提供 CE 和 EE 版本。		
		
![docker_ce_ee](https://img.nekolr.com/images/2018/07/26/Wqk.png)		
		
Docker 的安装非常简单，官方提供了详尽的安装方法，桌面系统只提供了 Docker CE 版。		
		
- Windows  
<https://docs.docker.com/docker-for-windows/install/>
- Mac OS  
<https://docs.docker.com/docker-for-mac/install/>
- CE for Ubuntu  
<https://docs.docker.com/install/linux/docker-ce/ubuntu/>
- EE for Ubuntu  
<https://docs.docker.com/install/linux/docker-ee/ubuntu/>
- 其他系统和具体信息参考  
<https://docs.docker.com/install/>

### Ubuntu 下安装 Docker
以 Ubuntu 16.04 安装 Docker CE 为例：		

最简单的安装方式就是直接通过包管理工具安装，使用这种方式安装的是系统自带的 Docker 安装包，可能不是最新版的 Docker。  

```cmd
$ sudo apt-get update
$ sudo apt-get install -y docker.io
```

或者使用官方提供的自动化脚本安装。  

```cmd
$ sudo curl -s https://get.docker.com | sh
```

### 免 sudo 使用 docker 命令
我们在安装 docker 时是使用 sudo 安装的，因此在使用普通用户使用 docker 时，会有提示：		

```
Got permission denied while trying to connect to the Docker daemon socket at UNIX:///var/run/docker.sock: Get http://%2Fvar%2Frun%2Fdocker.sock/v1.30/version: dial UNIX /var/run/docker.sock: connect: permission denied
```

docker 守护进程绑定到 UNIX 套接字而不是 TCP 端口。默认情况下，UNIX 套接字由用户 root 拥有，其他用户只能使用 sudo 访问它。docker 守护进程始终以 root 用户身份运行。  

如果不想在使用 docker 命令时使用 sudo，需要创建名为 docker 的 UNIX 组，并将用户添加到该组。当 docker 守护进程启动时，它会使 docker 组的 UNIX 套接字具有读/写权限。  

```cmd
:: 创建名为 docker 的用户组
$ sudo groupadd docker

:: 添加当前用户到该组
$ sudo usermod -aG docker $USER

:: 注销登录并重启以重新评估组成员资格，如果使用虚拟机，则重启虚拟机。

:: 验证不使用 sudo 使用 docker 命令
$ docker run hello-world
```
		
### 加速镜像

使用 Docker 在拉取官方镜像时，显然网络是个问题。因此我们可以使用一些镜像加速器或者直接从国内的镜像平台（阿里、网易等）拉取。		

Docker 官方镜像加速：<https://www.docker-cn.com/registry-mirror>
DaoCloud 镜像加速：<https://www.daocloud.io/mirror>		

## Docker 架构
![architecture](https://img.nekolr.com/images/2018/04/14/G7.png)

## Docker 体验
首先从远程仓库拉取一个镜像，使用 `docker pull [OPTIONS] name[:TAG]`。

```cmd
:: 从远程仓库拉取 hello-world 镜像，不加参数默认获取最新版本
:: 相当于 docker pull hello-world:latest
$ docker pull hello-world
```
拉取之后，可以使用 `docker images [OPTIONS] [REPOSITORY[:TAG]]` 命令查看本地仓库中的镜像。		

```cmd
:: 列出仓库中的镜像
$ docker images
```
运行镜像使用命令 `docker run [OPTIONS] IMAGE [COMMAND] [ARG...]`。

```cmd
$ docker run hello-world
```
运行后停止容器，使用命令 `docker stop [OPTIONS] CONTAINER [CONTAINER...]`。

```cmd
$ docker stop hello-world
```

如果想删除镜像，使用命令 `docker rmi [OPTIONS] IMAGE [IMAGE...]`。

```cmd
$ docker rmi hello-world
```
如果 image 被某个容器引用，则需要先销毁容器 `docker rm [OPTIONS] CONTAINER [CONTAINER...]`。

```cmd
:: 删除某个容器，需要指定容器 ID
$ docker rm <Container ID>

:: 接着删除 image，指定 image 名称或 ID
$ docker rmi <Image ID or Image Name>
```

刚才运行 hello-world 镜像是在前台运行的。如果想运行像 Nginx 这样的镜像，最好在后台运行。从后台运行 nginx，需要加一个参数，使用 `docker run --help 查看参数`。  

```cmd
:: 后台运行 nginx，返回容器 ID
$ docker run -d nginx
```

容器运行之后，我们可以使用命令 `docker exec` 进入到容器的内部，查看容器的相关信息，如日志等。		

```cmd
:: -i 即使没有附加也保持 STDIN 打开
:: -t 分配一个伪终端
:: 有时容器很少，可以不用写全 ID
:: command 为 bash，开启一个交互模式的终端
$ docker exec -it 0774be6e1c0c bash
```

我们知道，Linux 通过命名空间来隔离资源，如 network namespace 就是用来隔离网络的，每一个 network namespace 都提供了一个独立的网络环境。  

Docker 默认情况下会分配一个独立的 network namespace，也就是 Bridge 网络类型。由于与宿主机使用不同的网络环境，因此会涉及到**端口映射**，使通过宿主机能够访问 Docker 上的端口。Docker 能够将容器端口和宿主机端口做一个映射。		

如果指定 Docker 的网络类型为 Host，则不会分配一个独立的 network namespace，而是与宿主机使用同一命名空间。  

如果指定 Docker 的网络类型为 Null，则 Docker 不会获得网络资源。
		
![docker_network](https://img.nekolr.com/images/2018/04/14/0J.jpg)
		
上图中，eth0 是宿主机的网卡，在 Host 模式下，Docker 容器使用的网卡与宿主机相同。在 Bridge 模式下，Docker 首先创建一个 docker0 这样的网桥与宿主机的网卡连接，同时自己会虚拟出一个网卡（因此容器中会有自己的 IP、端口等），这个网卡与网桥相连，这样 Docker 就可以与宿主机进行通信了。		
		
```cmd
:: 开放一个端口到宿主机，前一个为主机端口，后一个为容器端口
$ docker run -d -p 8080:80 nginx
```

映射完之后，就可以使用浏览器访问宿主机的 8080 端口来访问容器的 80 端口了。
		
更多 Docker 命令：<https://docs.docker.com/reference/>