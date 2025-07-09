---
title: Docker基础知识
published: 2025-07-09
description: 'Docker基础知识'
image: ''
tags: ["docker", "k8s"]
category: 'Docker&k8s'
draft: false 
lang: ''
---
## 容器本质是进程
docker inspect 容器名，可以看到容器的cmd执行的命令
docker inspect 容器名字 | grep -i pid
查看到容器的进程号，根据进程号查找父进程
可以看到容器的父进程是runc
```
[root@jmp cgroup]# docker inspect b1 | grep -i pid
            "Pid": 21625,
            "PidMode": "",
            "PidsLimit": null,
[root@jmp cgroup]# ps -ef | grep 21625
root     21625 21606  0 00:30 pts/0    00:00:00 sh
root     23213 21684  0 01:13 pts/1    00:00:00 grep --color=auto 21625
[root@jmp cgroup]# ps  -ef | grep 21606
root     21606     1  0 00:30 ?        00:00:00 /usr/local/bin/containerd-shim-runc-v2 -namespace moby -id 11f06ad6b39fb6a861486c244e6bdf456287a92c53ff5521547bb906b686f722 -address /var/run/docker/containerd/containerd.sock
```

## 容器的namespce
查看系统namespace：`lsns`
### systemd进程的namespace
```
[root@jmp cgroup]# lsns | grep systemd
4026531836 pid      122     1 root    /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026531837 user     134     1 root    /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026531838 uts      122     1 root    /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026531839 ipc      122     1 root    /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026531840 mnt      120     1 root    /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026531956 net      122     1 root    /usr/lib/systemd/systemd --switched-root --system --deserialize 22
```

### 宿主机的namespace
当我们在系统中执行一个命令或启动一个进程后，查看比如启动一个vim
`ps -ef | grep vim
`lixuebin 22846  0.6  0.1 151488  5240 pts/2    S+   01:02  `` 0:00 vim
拿到pid通过/proc/pid/ns查看namespace
```
[root@jmp cgroup]# ll /proc/22846/ns
总用量 0
lrwxrwxrwx 1 lixuebin lixuebin 0 5月  22 01:02 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 lixuebin lixuebin 0 5月  22 01:02 mnt -> mnt:[4026531840]
lrwxrwxrwx 1 lixuebin lixuebin 0 5月  22 01:02 net -> net:[4026531956]
lrwxrwxrwx 1 lixuebin lixuebin 0 5月  22 01:02 pid -> pid:[4026531836]
lrwxrwxrwx 1 lixuebin lixuebin 0 5月  22 01:02 user -> user:[4026531837]
lrwxrwxrwx 1 lixuebin lixuebin 0 5月  22 01:02 uts -> uts:[4026531838]
```

可以看到宿主机中进程的namespace和systemd的namespace是相同的

### 容器的namespace
启动一个容器
`docker run -it --rm --name b1 busybox`
查看pid
```
[root@jmp cgroup]# docker inspect b1 | grep -i pid
            "Pid": 21625,
            "PidMode": "",
            "PidsLimit": null,
```
查看namespace
```
[root@jmp cgroup]# ll /proc/22846/ns
总用量 0
lrwxrwxrwx 1 lixuebin lixuebin 0 5月  22 01:02 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 lixuebin lixuebin 0 5月  22 01:02 mnt -> mnt:[4026531840]
lrwxrwxrwx 1 lixuebin lixuebin 0 5月  22 01:02 net -> net:[4026531956]
lrwxrwxrwx 1 lixuebin lixuebin 0 5月  22 01:02 pid -> pid:[4026531836]
lrwxrwxrwx 1 lixuebin lixuebin 0 5月  22 01:02 user -> user:[4026531837]
lrwxrwxrwx 1 lixuebin lixuebin 0 5月  22 01:02 uts -> uts:[4026531838]
```
可以看到除了user以外的namespace与systemd的namespace都是不同的。说明容器使用namespace对系统的ipc进程间通信，mnt挂载分区，net网络，pid进程，uts主机名和域名 都做了隔离。
**1. `ipc` 命名空间（IPC Namespace）**
- **隔离对象**：进程间通信（IPC）资源，如共享内存、消息队列、信号量。
- 作用：不同 `ipc` 命名空间中的进程无法通过 IPC 机制通信，确保容器间的通信隔离。例如，容器 A 中的进程无法访问容器 B 的共享内存或消息队列。

**2. `mnt` 命名空间（Mount Namespace）**

- **隔离对象**：文件系统挂载点（Mount Point）。
- **作用**：
    每个 `mnt` 命名空间拥有独立的文件系统视图，包括根目录（`/`）、挂载点（如 `/proc`、`/sys`）等。例如，在容器中挂载新磁盘不会影响宿主机或其他容器的文件系统。
- **特点**：
    - 内核默认启用，每个进程初始属于宿主的 `mnt` 命名空间。
    - 使用 `chroot` 命令可改变当前进程的根目录，但无法完全隔离挂载点，而 `mnt` 命名空间提供更彻底的隔离。
**3. `net` 命名空间（Network Namespace）**

- **隔离对象**：网络资源，包括网络设备、IP 地址、路由表、端口号、防火墙规则等。
- **作用**：
    每个 `net` 命名空间拥有独立的网络栈，相当于一个 “虚拟网络节点”。例如：
    - 容器可拥有独立的网卡（如 `eth0`）、IP 地址和端口，与宿主机或其他容器的网络栈完全隔离。
    - 宿主机的 `lo` 回环接口与容器的 `lo` 接口互不影响。
- **典型应用**：
    - Docker 容器通过 `veth` 设备连接宿主机与容器的 `net` 命名空间，实现网络通信。
**4. `pid` 命名空间（PID Namespace）**

- **隔离对象**：进程 ID（PID）空间。
- **作用**：
    每个 `pid` 命名空间维护独立的 PID 编号，允许不同命名空间中的进程拥有相同的 PID。例如：
    - 宿主机中 PID 为 1 的进程是 `systemd`，而在容器的 `pid` 命名空间中，PID 为 1 的进程可能是容器的初始化进程（如 `init` 或 `supervisord`）。
    - 子命名空间中的进程对父命名空间可见，但父命名空间的进程在子空间中不可见（需通过 `pid=host` 参数共享）。
- **层级关系**：
    - `pid` 命名空间支持嵌套（如父命名空间 → 子命名空间 → 孙命名空间），嵌套深度无固定限制。
 **5. `uts` 命名空间（UTS Namespace）**

- **隔离对象**：UTS（Unix Time-sharing System）标识符，包括主机名（`hostname`）和域名（`domain name`）。
- **作用**：
    不同 `uts` 命名空间中的进程看到的主机名和域名可以不同。例如：
    - 宿主机的主机名为 `host.example.com`，而容器的 `uts` 命名空间可设置为 `container.example.com`，二者互不影响。
**6.`user` 命名空间（User Namespace）**

- **隔离对象**：用户和组 ID（UID/GID）空间。
- **作用**：
    - 允许容器内的用户以不同的 UID/GID 映射到宿主机用户，实现权限隔离（如容器内的 `root` 用户对应宿主机的普通用户，避免容器内权限泄漏）。
    - 增强容器安全性，避免容器直接操作宿主机的用户资源。

**总结：Namespace 的作用与应用场景**

|命名空间类型|隔离资源|典型应用场景|
|---|---|---|
|`ipc`|进程间通信资源|容器间禁止直接通信|
|`mnt`|文件系统挂载点|容器拥有独立的根文件系统|
|`net`|网络栈|容器拥有独立 IP、端口、网卡|
|`pid`|进程 ID 空间|容器内进程 ID 从 1 开始重启|
|`uts`|主机名和域名|容器设置独立的主机名|
|`user`|用户和组 ID 空间|容器内权限与宿主机解耦|

## cgroup

目录：/sys/fs/cgroup/
用于限制和监控进程组（进程集合）对系统资源（如 CPU、内存、磁盘 I/O、网络带宽等）的使用

V1上级控制下级，通过目录层级进行上下级区分
### cpu：
/sys/fs/cgroup/cpu
[root@jmp cpu]# cat cpu.cfs_period_us
100000 调度周期
[root@jmp cpu]# cat cpu.cfs_quota_us
-1 份额
如果需要限制20%，将cpu.cfs_quota_us值改为cpu.cfs_period_us的20%
即可限制cpu使用率20%

如果docker所有容器限制20%, 需要让其中某个容器多占用，调整cpu.shares 值，默认是1024，增加一倍就多占用一倍。

在使用中真正限制容器的cpu，在容器启动时指定 --cpu-quota --cpu-shares 都是可以指定的



**Docker的cpu控制所有容器**
/sys/fs/cgroup/cpu/docker

该目录下有个容器的id目录，可以控制该容器的cpu使用
/sys/fs/cgroup/cpu/docker/容器id

### 内存
/sys/fs/cgroup/memory

1.  memory.limit_in_bytes  容器可以使用的物理内存上限
2. memory.memsw.limit_in_bytes 容器可以使用物理内存和交换分区加一起的上限 可以理解为最高上限
3. memory.oom_control  容器超出内存上限的策略，默认为kill

启动docker的时候指定
`docker run -it --rm --name b1 --memory 6M --memory-swap 10M  busybox`
#### **Cgroup 的核心功能**

1. **资源限制**：
    限制进程组使用的资源上限（如 CPU 时间、内存大小）。
    例如：限制容器最多使用 2 核 CPU 和 1GB 内存。

2. **优先级控制**：
    通过分配权重（Weight）调整进程组的资源使用优先级。
    例如：高优先级的容器获得更多 CPU 时间片。

3. **资源统计**：
    监控和统计进程组的资源使用情况（如 CPU 使用率、内存用量）。
    例如：统计容器的内存峰值，用于计费或性能分析。

4. **进程控制**：
    对进程组执行挂起、恢复等操作。
    例如：内存不足时，自动暂停某些低优先级容器。


#### **Cgroup 的基本概念**

##### **1. 层级结构（Hierarchy）**

- Cgroup 以树状结构组织，子节点继承父节点的配置。
- 每个 Hierarchy 可关联多个子系统（Subsystem）。

##### **2. 子系统（Subsystem）**

子系统是一组资源控制器，负责特定类型的资源管理。常见子系统包括：



- **cpu**：CPU 时间分配（通过权重或配额）。
- **memory**：内存使用限制与监控。
- **blkio**：块设备（磁盘）I/O 限制。
- **net_cls**：网络流量分类（标记网络数据包）。
- **devices**：设备访问控制。
- **cpuset**：为进程分配特定 CPU 核心和内存节点（NUMA）。

##### **3. 控制组（Control Group）**

- 控制组是 Cgroup 树中的节点，每个节点可配置资源限制参数。
- 进程加入控制组后，遵循该组的资源限制规则。

##### **4. 进程（Task）**

- 进程是 Cgroup 的管理对象，一个进程可同时属于多个控制组（需关联不同子系统）。

#### **Cgroup 的实现版本**

Linux 内核提供了两代 Cgroup 实现：

##### **1. Cgroup v1（legacy）**

- **特点**：
    - 每个子系统独立挂载，形成多个层级结构。
    - 配置复杂，不同子系统可能存在冲突。
- **路径**：通常位于 `/sys/fs/cgroup/` 目录下，每个子系统对应一个子目录（如 `/sys/fs/cgroup/cpu/`）。
- **兼容性**：所有 Linux 内核版本均支持（2.6.24+）。

##### **2. Cgroup v2（unified）**

- **特点**：
    - 统一所有子系统到单个层级结构，简化配置。
    - 支持动态启用 / 禁用子系统。
    - 增强了对嵌套控制组的支持。
- **路径**：通常挂载在 `/sys/fs/cgroup/unified/`。
- **兼容性**：Linux 内核 4.5+ 支持，需手动启用。

#### **Cgroup v2 的核心概念**

- **Controllers**：即 v1 中的子系统，如 `cpu`、`memory` 等。
- **Delegation**：允许子控制组自主管理部分控制器，适用于容器嵌套场景。
- **Pressure Stall Information（PSI）**：实时监控系统资源压力，提前预警资源不足。

#### **Cgroup 的应用场景**

1. **容器技术**：
    Docker、Kubernetes 等容器平台基于 Cgroup 实现资源隔离。
    例如：`docker run -m 1g --cpus=2` 命令通过 Cgroup 限制容器内存和 CPU。

2. **云计算**：
    云服务商通过 Cgroup 实现多租户资源隔离与计费。
    例如：限制虚拟机或租户进程的资源使用上限。

3. **高性能计算（HPC）**：
    分配专用 CPU 核心给关键任务，确保性能稳定性。

4. **资源回收**：
    当系统资源不足时，自动终止或限制低优先级进程。


#### **Cgroup 配置示例（以 Cgroup v1 为例）**

##### **1. 创建 CPU 控制组**

bash

```bash
# 创建控制组目录（自动挂载 cpu 子系统）
mkdir /sys/fs/cgroup/cpu/mygroup

# 限制 CPU 使用率为 50%（假设系统为 1 核）
echo 50000 > /sys/fs/cgroup/cpu/mygroup/cpu.cfs_quota_us  # 每 100ms 中允许运行 50ms
echo 100000 > /sys/fs/cgroup/cpu/mygroup/cpu.cfs_period_us  # 周期为 100ms

# 将进程 ID 加入控制组
echo $PID > /sys/fs/cgroup/cpu/mygroup/tasks
```

##### **2. 创建内存控制组**

bash

```bash
# 创建控制组目录
mkdir /sys/fs/cgroup/memory/mygroup

# 限制内存使用为 200MB
echo 200M > /sys/fs/cgroup/memory/mygroup/memory.limit_in_bytes

# 限制 swap 使用为 0（禁止使用交换空间）
echo 0 > /sys/fs/cgroup/memory/mygroup/memory.swappiness

# 将进程加入控制组
echo $PID > /sys/fs/cgroup/memory/mygroup/tasks
```

#### **Cgroup 与容器的关系**

- **Docker/Kubernetes**：通过 Cgroup 实现容器资源隔离，每个容器对应一个或多个控制组。
- **OCI 规范（Open Container Initiative）**：定义了容器运行时接口，底层依赖 Cgroup 和 Namespace 实现。
- **工具链**：
    - `cgexec`：直接操作 Cgroup 的命令行工具。
    - `libcgroup`：Cgroup 的 C 语言库，用于编程控制。
    - `systemd`：现代 Linux 系统默认使用 systemd 管理 Cgroup，每个服务对应一个控制组。

#### **总结**

Cgroup 是 Linux 内核提供的强大资源管理工具，通过控制组、子系统和层级结构，实现了对进程组的精细化资源控制。它是容器技术的核心基础之一，广泛应用于云计算、容器编排和资源隔离场景。理解 Cgroup 的工作原理，有助于优化容器部署、排查资源争用问题，并充分发挥系统性能。



