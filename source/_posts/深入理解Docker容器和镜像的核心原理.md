---
title: "深入理解Docker容器和镜像的核心原理"
catalog: true
toc_nav_num: true
date: 2019-05-12 20:51:24
subtitle: "从Namespace、CGroups和Mount介绍容器和镜像的技术原理"
header-img: "/img/article_header/tizi.jpg"
tags:
- Docker
catagories:
- Docker
---
> 本篇文章介绍容器的本质是一种特殊进程，Docker利用Namespace实现容器中进程的隔离，并且利用CGroups机制实现容器对宿主机资源的访问限制。

上一篇文章["通过容器化Python Web应用了解Docker容器的核心功能"](https://liuchunming.net/article/%E9%80%9A%E8%BF%87%E5%AE%B9%E5%99%A8%E5%8C%96Python%20web%E5%BA%94%E7%94%A8%E4%BA%86%E8%A7%A3Docker%E5%AE%B9%E5%99%A8%E6%A0%B8%E5%BF%83%E5%8A%9F%E8%83%BD/)中，我们了解了如何通过镜像启动容器以及对容器的各种操作方法。这篇文章，我们来深入学习容器背后的技术原理。

## 容器是一种特殊进程
从容器的使用上，我们可以这样理解：容器其实是一种沙盒技术。也就是说容器像一个集装箱一样，把你的应用“装”起来。这样，应用与应用之间，就因为有了**边界**而不至于相互干扰。下面，我们就了解一下这个“边界”是如何实现的。

在了解"边界"是如何实现之前，我们来看看到底这个"边界"是什么。首先来运行一个容器来试试：
```text
docker run -d -p 5000:80 helloworld /bin/sh
```
-it 参数告诉了 Docker 项目在启动容器后，需要给我们分配一个文本输入 / 输出环境，也就是 TTY，跟容器的标准输入相关联，这样我们就可以和这个 Docker 容器进行交互了。而 /bin/sh 就是我们要在 Docker 容器里运行的程序。

这样，执行`docker run`命令的机器就变成了一个宿主机，而一个运行着 /bin/sh 的容器，就跑在了这个宿主机里面。

执行上面的命令后，我们就进入到了由镜像helloworld启动的容器中，在容器中我们执行ps命令：
```text
# ps -a
PID   USER     TIME  COMMAND
    1 root      0:15 python app.py
   17 root      0:00 /bin/sh
   29 root      0:00 ps -a
```
可以看到，python app.py，是这个容器内部的第 1 号进程（PID=1），而这个容器里一共只有3个进程在运行。我们知道在一台Linux机器上，只能有一个PID=1的进程，也就是init进程。而容器中又出现了一个PID=1的进程，这也就意味着容器中的三个进程被 Docker 隔离在了一个跟宿主机完全不同的世界当中。

这究竟是怎么做到呢？

本来，每当我们在宿主机上运行 python app.py 程序，操作系统都会给它分配一个进程编号，比如 PID=100。而现在，我们要通过 Docker 把这个 python app.py 程序运行在一个容器当中，他的PID就变成了1。

这其实是对应用的进程空间做了手脚，使得 python app.py 进程被隔离起来，在这个隔离的空间中，进程编号PID=1。可实际上，他们在宿主机的操作系统里，还是原来的第 100 号进程。

实现这种隔离的技术，就是 Linux 里面的 Namespace 机制。 Namespace其实只是 Linux 创建新进程的一个可选参数。

在Linux系统中，创建新进程需要调用系统调用clone()，当我们用 clone() 创建一个新进程时，可以在参数中指定 CLONE_NEWPID 参数，这时，新创建的这个进程将会“看到”一个全新的进程空间，在这个进程空间里，它的 PID 是 1。之所以说“看到”，是因为这只是一个“障眼法”，在宿主机真实的进程空间里，这个进程的 PID 还是真实的数值，比如 100。

当然，我们还可以多次执行上面的 clone() 调用，这样就会创建多个 PID Namespace，而每个 Namespace 里的应用进程，都会认为自己是当前容器里的第 1 号进程，它们既看不到宿主机里真正的进程空间，也看不到其他 PID Namespace 里的具体情况。

除了我们刚刚用到的 PID Namespace，Linux 操作系统还提供了 Mount、UTS、IPC、Network 和 User 这些 Namespace，用来对各种不同的进程上下文进行“障眼法”操作。

比如，Mount Namespace，用于让被隔离进程只看到当前 Namespace 里的挂载点信息；Network Namespace，用于让被隔离进程看到当前 Namespace 里的网络设备和配置。

Docker在帮助用户启动容器中的进程时，指定了这个进程所需要启用的各种 Namespace 参数。这样，在容器中就只能“看”到当前 Namespace 所限定的资源、文件、设备、状态，或者配置。而对于宿主机以及其他不相关的程序，它就完全看不到了。

所以说，容器，其实是一种特殊的进程。

容器化后的用户应用，依然还是一个宿主机上的普通进程，这就意味着这些因为虚拟化而带来的性能损耗都是不存在的；而另一方面，使用 Namespace 作为隔离手段的容器并不需要想虚拟化软件那样单独的 Guest OS，这就使得容器额外的资源占用几乎可以忽略不计。所以说，“高性能”是容器相较于虚拟机最大的优势。

既然容器只是运行在宿主机上的一种特殊的进程，那么多个容器之间使用的就还是同一个宿主机的操作系统内核的。虽然Docker利用了Namespace实现了进程的隔离，但是在 Linux 内核中，有很多资源和对象是不能被 Namespace 化的，最典型的例子就是：时间。如果你的容器中的程序使用 settimeofday(2) 系统调用修改了时间，整个宿主机的时间都会被随之修改。

## 限制容器的资源使用
为什么还需要对容器做“限制”呢？

容器内的第 1 号进程在“障眼法”的干扰下只能看到容器里的情况，但是宿主机上，它作为第 100 号进程与其他所有进程之间依然是平等的竞争关系。这就意味着，虽然第 100 号进程表面上被隔离了起来，但是它所能够使用到的资源（比如 CPU、内存），却是可以随时被宿主机上的其他进程（或者其他容器）占用的。当然，这个 100 号进程自己也可能把所有资源吃光。

Linux Cgroups 就是 Linux 内核中用来为进程设置资源限制的一个重要功能。Linux Cgroups 的全称是 Linux Control Group，它最主要的作用，就是限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等。Docker正式利用了这一机制实现对容器进程的资源限制。

在 Linux 中，Cgroups 给用户暴露出来的操作接口是文件系统，即它以文件和目录的方式组织在操作系统的 /sys/fs/cgroup 路径下。在CentOS下，可以用 mount 指令把它们展示出来
```text
mount -t cgroup
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_prio,net_cls)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpuacct,cpu)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
```
它的输出结果，是一系列文件系统目录。如果你在自己的机器上没有看到这些目录，那你就需要自己去挂载 Cgroups。

在 /sys/fs/cgroup 下面有很多诸如 cpuset、cpu、 memory 这样的子目录，也叫子系统。这些是当前可以被 Cgroups 进行限制的资源种类。而在子系统对应的资源种类下，你就可以看到该类资源具体可以被限制的方法。

比如，对 CPU 子系统来说，我们就可以看到如下几个配置文件，这个指令是`ls /sys/fs/cgroup/cpu`：
```text
ls /sys/fs/cgroup/cpu
cgroup.clone_children  cgroup.procs          cpu.cfs_period_us  cpu.rt_period_us   cpu.shares  cpuacct.stat   cpuacct.usage_percpu  release_agent
cgroup.event_control   cgroup.sane_behavior  cpu.cfs_quota_us   cpu.rt_runtime_us  cpu.stat    cpuacct.usage  notify_on_release     tasks
```
在它的输出里注意到有 cfs_period 和 cfs_quota这两个参数，它们需要组合使用，可以用来限制进程在长度为 cfs_period 的一段时间内，只能被分配到总量为 cfs_quota 的 CPU 时间。

## 镜像是怎么组成的

## 总结
本篇文章通过非常经典的 Python web应用作为案例，讲解了 Docker 容器使用的主要场景。包括构建镜像、启动镜像、分享镜像、在镜像中操作、在镜像中挂载宿主机目录、对容器使用的资源进行限制、管理容器的状态和如何保持容器始终运行。熟悉了这些操作，你也就基本上摸清了 Docker 容器的核心功能。
