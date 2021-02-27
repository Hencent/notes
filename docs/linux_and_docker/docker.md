# Overview
容器 = cgroup + namespace + rootfs + 容器引擎
+ Cgroup： 资源控制
+ namespace： 访问隔离
+ rootfs：文件系统隔离。镜像的本质就是一个rootfs文件
+ 容器引擎：生命周期控制
  
[https://www.cnblogs.com/yds941268778/p/12688363.html](https://www.cnblogs.com/yds941268778/p/12688363.html)

Docker提供了在称为容器的松散隔离环境中打包和运行应用程序的能力。  
容器是轻量级的，因为它们不需要 hypervisor（虚拟机监视器） 的额外负载，而是直接在主机内核中运行。

> OS = 内核 + 文件系统/库
>
> linux 内核只有一个正统来源，那就是 linus 的开发团队。内核只提供操作系统的基本功能和特性，如内存管理，进程调度，文件管理等等。
> 
> ubuntu 这些是 linux 的发行版，基于 linus 维护的 linux 内核，是由一些厂商集成了漂亮易用的桌面和常用的软件而发布的商品。
> 
> 在 Docker 容器中运行的 Ubuntu 镜像只是 Ubuntu 文件系统/库 ，它不包含 Ubuntu 内核，该容器利用底层主机的内核，所有容器都在同一个内核上运行。发行版之间最大的区别是用户区（应用程序、库、文件系统、包管理器）。每个容器都包含它自己的版本。
> 
> 虽然 kernel 的版本可能不同，但是对于非驱动开发者来说，并没有什么关系。毕竟，内存管理，文件读写之类的系统调用都是基本没有区别的。

# Docker Engine
docker engine 是一个 client-server application。包含
+ daemon process (dockerd)： 管理容器的守护进程
+ A REST API： 联系与指导 daemon 工作的接口
+ A command line interface (CLI) client (the docker command)

CLI 使用 REST API 来控制 daemon 或与之交互。
> [https://docs.docker.com/get-started/overview/#docker-architecture](https://docs.docker.com/get-started/overview/#docker-architecture)

# docker container
Docker 容器本质上是宿主机上的进程。Docker 通过 namespace 实现了资源隔离。通过 cgroups 实现了资源限制，通过写时复制机制实现了高效的文件操作。

# docker volumn
Docker 镜像被存储在一系列的只读层。当我们开启一个容器，Docker 读取只读镜像并添加一个读写层在顶部。此只读层及在顶部的读写层的组合被Docker称为Union File System（联合文件系统）。

如果正在运行的容器修改了现有的文件，该文件将被拷贝出底层的只读层到最顶层的读写层。在读写层中的旧版本文件隐藏于该文件之下，但并没有被不破坏, **它仍然存在于镜像以下**。

当Docker的容器被删除，然后重新启动镜像时，将开启一个没有任何更改的新的容器, **这些更改会丢失**。

+ volume能在不同的容器之间共享和重用
+ 对volume中的数据的操作会马上生效
+ 对volume中数据操作不会影响到镜像本身，（比如 commit 新的 image 的时候，不会因为往 volume 中塞了很多数据就有庞大的体积
+ volume的生存周期独立于容器的生存周期，即使删除容器，volume仍然会存在，没有任何容器使用的volume也不会被Docker删除

# docker 文件系统
一个 linux 系统需要运行的话，至少需要两个文件系统。：
+ boot file system （bootfs）：包含 boot loader 和 **kernel**。在启动（boot）过程完成后，整个内核都会被加载进内存，此时 bootfs 会被卸载掉从而释放出所占用的内存。
+ root file system （rootfs）：根目录挂载的文件系统。包括` /dev, /proc, /bin, /etc, /lib, /usr, and /tmp ` 等运行用户应用所需要的所有配置文件，二进制文件和库文件。这个文件系统在不同的 Linux 发行版中是不同的。Linux 系统在启动时，roofs 首先会被挂载为只读模式，然后在启动完成后被修改为读写模式 (docker 中不会)。

在 docker 中：
+ 所有 Docker 容器都共享主机系统的 bootfs 即 Linux 内核
+ 每个容器有自己的 rootfs，它来自不同的 Linux 发行版的基础镜像，包括 Ubuntu，Debian 和 SUSE 等

Docker 在镜像的设计中，引入了层（layer）的概念。也就是说，用户制作镜像的每一步操作，都会生成一个层，也就是一个增量 rootfs。 用到的技术就是联合文件系统（Union File System），也叫 UnionFS ，最主要的功能是将多个不同位置的目录联合挂载（union mount）到同一个目录下。

在 docker 架构中，当 docker daemon 为 docker 容器挂载 rootfs 时，沿用了 Linux 内核启动时的做法，即将 rootfs 设为只读模式。在挂载完毕之后，利用联合挂载 (union mount) 技术**在已有的只读 rootfs 上再挂载一个读写层**。这样，**可读写的层处于 docker 容器文件系统的最顶层，其下可能联合挂载了多个只读的层**，只有在 **docker 容器运行过程中文件系统发生变化时，会把变化的文件内容写到可读写层，并隐藏只读层中的旧版本文件**。

# 连接 docker 的方式
1. docker attach  
    **过时了**！不建议使用。Docker attach可以attach到一个已经运行的容器的stdin，然后进行命令执行的动作。  
    但是需要注意的是，如果从这个stdin中exit，会导致容器的停止。
2. SSH 不建议
3. docker exec  
    `docker exec [OPTIONS] CONTAINER COMMAND [ARG...]`  
    例如使用 `docker exec -it CONTAINER_NAME /bin/bash` 即可
4. nsenter  
    nsenter (namespace enter) 可以访问另一个进程的名称空间(namespace)，这样就能间接访问某个 container 的环境了。  
    实际上，理解了 docker 底层是基于 namespace 的，就可以完全明白为什么了。namespace 的描述详见 linux 中的笔记。  
    获取容器中一个进程的真实 PID，然后使用 nsenter 即可进入相同 namespace，实际上就进入了这个容器的环境了。
