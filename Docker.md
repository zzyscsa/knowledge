理解docker，主要从namesapce，cgroups，联合文件，运行时(runC)，网络几个方面。

namesapce主要是隔离作用，cgroups主要是资源限制，联合文件主要用于镜像分层存储和管理，runC是运行时，遵循了oci接口，一般来说基于libcontainer。网络主要是docker单机网络和多主机通信模式。

# namespace

## 概述

**Namespace是将内核的全局资源做封装，使得每个Namespace都有一份独立的资源，因此不同的进程在各自的Namespace内对同一种资源的使用不会互相干扰**。实际上，Linux内核实现namespace的主要目的就是为了实现轻量级虚拟化（容器）服务。在同一个namespace下的进程可以感知彼此的变化，而对外界的进程一无所知。这样就可以让容器中的进程产生错觉，仿佛自己置身于一个独立的系统环境中，以此达到独立和隔离的目的。

Linux内核总共实现了六种namespace

| namespace | 系统调用参数  | 隔离的内容                                             |
| --------- | ------------- | ------------------------------------------------------ |
| IPC       | CLONE_NEWIPC  | 隔离信号量、消息队列与共享内存                         |
| Network   | CLONE_NEWNET  | 隔离网络资源：网络设备、网络栈、端口等                 |
| Mount     | CLONE_NEWNS   | 隔离文件系统挂载点。每个容器能看到不同文件系统层次结构 |
| PID       | CLONE_NEWPID  | 隔离进程ID                                             |
| UTS       | CLONE_NEWUTS  | 隔离主机名和域名                                       |
| User      | CLONE_NEWUSER | 隔离用户ID和组ID                                       |

## namespace api调用的四种方式

### clone()

使用clone()来创建一个独立namespace的进程是最常见做法，它的调用方式如下。

```c
int clone(int (*child_func)(void *), void *child_stack, int flags, void *arg);
```

### setns()

通过setns()加入一个已经存在的namespace。 进程在结束的情况下，也可以通过挂载的形式把namespace保留下来，保留下来的目的是为了后续进程加入做准备，在docker中 使用docker exec 命令在已经运行着的容器中执行一个新命令，就需要用到该方法。通过setens()系统调用，进程从原来的namespace加入某个已存在的namespace，使用该方法，通常不影响进程的调用者，也为新加入的pid namespace 生效，会在setns() 函数执行后使用clone()创建子进程执行命令，让原先的进程结束运行

```c
int setns(int fd, int nstype);
```

### unshare()

通过unshare()在原先进程上进行namespace隔离。调用unshare()的主要作用就是不启动一个新进程就可以起到隔离的效果，相当于跳出原先的namespace进行操作。这样，你就可以在原进程进行一些需要隔离的操作。Linux中自带的unshare命令，就是通过unshare()系统调用实现的。

```c
int unshare(int flags);
```

## UTS Namespace

UTS namespace提供了**主机名和域的隔离**，这样Docker容器就有独立的主机名和域名了，在网络中视为一个独立节点，而非宿主机上一个进程，Docker中，每个镜像本身基本都以自身所提供的服务名称来命名镜像的hostname。可以使用主机名在网络上访问某台机器了，如果不做隔离，这个机制在容器里就会出问题。

## IPC Namespace

IPC是进程间通信，IPC Namespace针对的是SystemV IPC和Posix消息队列。这些IPC机制都会用到标识符，例如用标识符来区别不同的消息队列，然后两个进程通过标识符找到对应的消息队列进行通信等。IPC Namespace能做到的事情是，**使相同的标识符在两个Namespace中代表不同的消息队列，这样也就使得两个Namespace中的进程不能通过IPC进程通信了。**Docker也使用的IPC namespace实现了容器与宿主机、容器与容器之间的IPC隔离

## PID Namespace

PID Namespace用于隔离进程PID号，这样一来，不同的Namespace里的进程PID号就可以是一样的了。

## Network Namespace

每个Network Namespace都有自己的网络设备、IP地址、路由表、/proc/net目录、端口号等。举一个例子，在没有隔离的情况下，如果两个不同的容器都想运行同一个Web应用，而这个应用又需要使用80端口，那就会有冲突了。

## Mount namespace

隔离后，不同mount namespace中的文件结构发生变化也互不影响。

## User Namespace

一个进程在Namespace里的用户和组ID与它在host里的ID可以不一样，host的普通用户进程在容器里可以是0号用户，也就是root用户。这样，进程在容器内可以做各种特权操作，但是它的特权被限定在容器内，离开了这个容器它就只有普通用户的权限了。

# cgroups控制组

Cgroup是control group的简写，属于Linux内核提供的一个特性，用于限制和隔离一组进程对系统资源的使用，也就是做资源QoS，这些资源主要包括CPU、内存、block I/O和网络带宽。

## 四大功能

- **资源限制**（Resource Limitation）：cgroups可以对进程组使用的资源总额进行限制。如设定应用运行时使用内存的上限，一旦超过这个配额就发出OOM（Out of Memory）。
- **优先级分配**（Prioritization）：通过分配的CPU时间片数量及硬盘IO带宽大小，实际上就相当于控制了进程运行的优先级。
- **资源统计**（Accounting）： cgroups可以统计系统的资源使用量，如CPU使用时长、内存用量等等，这个功能非常适用于计费。
- **进程控制**（Control）：cgroups可以对进程组执行挂起、恢复等操作。

## 三个组件

- cgroup 控制组 。cgroup 是对进程分组管理的一种机制，一个cgroup包含一组进程，并可以在这个cgroup上增加Linux subsystem的各种参数的配置，将一组进程和一组subsystem的系统参数关联起来。
- subsystem 子系统。subsystem 是一组资源控制的模块。这块在下面会详细介绍。
- hierarchy 层级树。hierarchy 的功能是把一组cgroup串成一个树状的结构，一个这样的树便是一个hierarchy，通过这种树状的结构，Cgroups可以做到继承。比如我的系统对一组定时的任务进程通过cgroup1限制了CPU的使用率，然后其中有一个定时dump日志的进程还需要限制磁盘IO，为了避免限制了影响到其他进程，就可以创建cgroup2继承于cgroup1并限制磁盘的IO，这样cgroup2便继承了cgroup1中的CPU的限制，并且又增加了磁盘IO的限制而不影响到cgroup1中的其他进程。

## cgroups子系统

cgroup中实现的子系统及其作用如下：

- devices：设备权限控制。
- cpuset：分配指定的CPU和内存节点。
- cpu：控制CPU占用率。
- cpuacct：统计CPU使用情况。
- memory：限制内存的使用上限。
- freezer：冻结（暂停）Cgroup中的进程。
- net_cls：配合tc（traffic controller）限制网络带宽。
- net_prio：设置进程的网络流量优先级。
- huge_tlb：限制HugeTLB的使用。
- perf_event：允许Perf工具基于Cgroup分组做性能监测。

# 网络模式

我们在使用docker run创建Docker容器时，可以用--net选项指定容器的网络模式，Docker有以下4种网络模式：

- host模式，使用--net=host指定。
- container模式，使用--net=container:NAME_or_ID指定。
- none模式，使用--net=none指定。
- bridge模式，使用--net=bridge指定，默认设置。

## none

不为容器配置任何网络功能。

## container

与另一个运行中的容器共享Network Namespace，共享相同的网络视图。使用--net=container：d25864df1a3b参数启动的容器，其IP地址、DNS、hostname都继承了容器d25864df1a3b。实质上两个容器是共享同一个Network Namespace的，自然网络配置也是完全相同。

## host

与主机共享Root Network Namespace，容器有完整的权限可以操纵主机的协议栈、路由表和防火墙等，所以被认为是不安全的。容器可以操纵主机的网络配置，这是危险的，除非万不得已，应该尽可能避免使用host模式。

## bridge

<font color='red'>Docker设计的NAT网络模型。</font>

Docker daemon启动时会在主机创建一个Linux网桥（默认为docker0，可通过-b参数手动指定）。容器启动时，Docker会创建一对veth pair（虚拟网络接口）设备，veth设备的特点是成对存在，从一端进入的数据会同时出现在另一端。Docker会将一端挂载到docker0网桥上，另一端放入容器的Network Namespace内，从而实现容器与主机通信的目的。bridge模式下的网络拓扑图如下图所示。

![preview](https://segmentfault.com/img/bVbgOLa?w=443&h=413/view)

在桥接模式下，Docker容器与Internet的通信，以及不同容器之间的通信，都是通过iptables规则控制的。Docker网络的初始化动作包括：创建docker0网桥、为docker0网桥新建子网及路由、创建相应的iptables规则等。

