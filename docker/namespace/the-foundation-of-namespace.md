---
description: 'Namespace: isolate Controller'
---

# The concept of namespace

_container_ can be considered synonymous with a Linux network namespace.

容器的目的是进行**资源隔离**和**控制隔离**。

* 资源隔离：隔离计算资源，如CPU、MEMORY、DISK等。
* 控制隔离：隔离一些控制结构，如Net Stack、UID、PID等

 **资源隔离**依赖于linux内核的Cgroup实现，**控制隔离**依赖于linux内核的namespace.

> The Linux kernel provides the cgroups functionality that allows limitation and prioritization of resources \(CPU, memory, block I/O, network, etc.\) without the need for starting any virtual machines, and also namespace isolation functionality that allows complete isolation of an applications’ view of the operating environment, including process trees, networking, user IDs and mounted file systems.

Docker的本质实际上是宿主机上的一个进程，通过namespace实现了资源隔离，通过cgroup实现了资源限制。理解Linux Namespace的概念有助于我们理解和排查容器应用。

**Sandbox**

The fundamental idea behind sandboxing is to reduce risk by limiting the environment in which certain code executes.

"The whole idea, no matter what sandbox you're talking about, is putting someone in an environment so they can't access something outside the scope of what they should be doing," explains Marcus Carey, security researcher for Rapid7.

Sandbox类似VLAN的概念，减少环境整个坏掉的风险。

举个栗子：一台LNMP服务，Nginx被webshell攻陷了，如果Nginx没有运行在单独隔离的Sandbox中，那么攻击者就会看到整个host上的所有进程，进而引发更大的损失。所以Sandbox自身的隔离性便提供了一种安全防护措施。

一个鸡蛋不能放在一个篮子里，就算放在一个篮子里也不能被人发现。

Sandbox Cons:

* 由于隔离的特性，会导致有些参数需要设置多遍即针对每个sandbox都要进行设置。
* reserved

**Chroot**

chroot - run command or interactive shell with special root directory.

这个了解下就行了，因为基于chroot实现的Sandbox存在逃逸风险，这里不过多讨论，有兴趣自行Google.。

chroot escape tool ：[chw00t](https://github.com/earthquake/chw00t)

This tool can help you escape from chroot environments. Even though chroot is not a security feature in any Unix systems, it is misused in a lot of setups. With this tool there is a chance that you can bypass this barrier.

**Namespce**

核心： 实现资源隔离

 A namespace wraps a global system resource in an abstraction that makes it appear to the processes within the namespace that they have their own isolated instance of the global resource. Changes to the global resource are visible to other processes that are members of the namespace, but are invisible to other processes.

**Namespace Type**

Namespace核心的功能就是隔离资源，可以隔离的资源类型如下：

```text
Linux provides the following namespaces:
​
Namespace   Constant          Isolates
Cgroup      CLONE_NEWCGROUP   Cgroup root directory
IPC         CLONE_NEWIPC      System V IPC, POSIX message queues
Network     CLONE_NEWNET      Network devices, stacks(ip/routes/iptables), ports, etc.
Mount       CLONE_NEWNS       Mount points
PID         CLONE_NEWPID      Process IDs
User        CLONE_NEWUSER     User and group IDs
UTS         CLONE_NEWUTS      Hostname and NIS domain nam
```

* Cgroup: since Linux 4.6 用来限制该Process下各namespaces的资源，详见Cgroup。
* IPC: IPC namespace 隔离的是 IPC（Inter-Process Communication） 资源，也就是进程间通信的方式，包括 System V IPC 和 POSIX message queues。

  注意这些 `/proc` 中内容对于每个 namespace 都是不同的（可能需要根据不同应用设置合适的值）：

  * `/proc/sys/fs/mqueue` 下的 POSIX message queues
  * `/proc/sys/kernel` 下的 System V IPC，包括 msgmax, msgmnb, msgmni, sem, shmall, shmmax, shmmni, and shm\_rmid\_forced
  * `/proc/sysvipc/`：保存了该 namespace 下的 system V ipc 信息

  在 linux 下和 ipc 打交道，需要用到以下两个命令：

  * `ipcs`：查看IPC\(共享内存、消息队列和信号量\)的信息
  * `ipcmk`：创建IPC\(共享内存、消息队列和信号量\)的信息

* Network: Net namespace 隔离的是相关网络资源：Network devices, stacks\(ip/routes/iptables\), ports, etc.

  这里需要着重提下的是也会隔离`/proc/net`和`/proc/sys/net` 目录。这两个目录至关重要，它们是关于网络参数的内核配置目录。

  eg： `/proc/sys/net/core/somaxconn` 来设置 tcp accept queue size。

  如果不单独对net ns进行参数调整，它就会使用默认值。这也就是隔离带来的弊端了。

* Mount: Mount namespace 隔离的是 mount points（挂载点），也就是说不同 namespace 下面的进程看到的文件系统结构是不同的，namespace 内的 mount points 可以通过 `mount(2)` 和 `umount(2)` 来修改，因此Mount namespace 可以用来实现容器文件系统的隔离。

  > 因为 Mount namespace 是最早加入到 linux 的，当时并没有预计到其他 namespace 的可能性，所以它被取名为 `CLONE_NEWNS`，而不是 `CLONE_NEWMOUNT` 之类的名字。为了保持兼容性，这个名字就一直延续到现在。

  * `/proc/[pid]/mounts` 文件保存了进程所在 namespace 所有已经 mount 的文件系统。
  * `/proc/[pid]/mountstats` 文件保存了进程所在 namespace mount point 的统计信息
  * `/proc/[pid]/mountinfo`

  linux = Rootfs\(GUN toolchain\) + Kernel

  Container = Image\(rootfs\) + Process

* PID: 隔离的是进程的 pid 属性，也就是说不同的 namespace 中的进程可以有相同的 pid。

  这里的PID ns和常用系统一样从`pid 1`开始，也就是说ns中的第一个进程就是init process，所以当有指定ns中运行程序时要慎重，因为它需要承担init process的责任（回收子进程之类的）。

* User: User namespace 隔离的是用户和组信息，在不同的 namespace 中用户可以有相同的 UID 和 GID，它们之间互相不影响，也可以在不同的ns中永远特定的用户。

  这里有个很重要的事情：Parent NS中的非root用户也可以拥有Child NS中的root权限，而Child NS中的root权限则不等于Parent NS中的root。

  放到Docker中理解就是，Container中的root权限是被限制的root，要想提权则需要通过`--cap-add`将对应的Linux capabilities 机制中细分的权限添加给Container.

  > 从内核2.2版本开始，Linux把原来和超级用户相关的高级权限划分成为不同的单元，称为Capability。这样管理员就可以独立对特定的Capability进行使能或禁止。

* UTS: UNIX _Time_-sharing System隔离主机名和域名信息

这里的隔离不是完全的隔离，典型例子：PID Namespace中的`PID 1`在host中的RootNamespace中只一个一般进程，当这个进程吃光host资源时，就会被OS直接kill。这也是Docker Container因为资源消耗被强行终止的原因。

此外，还有一些资源是没有对应的Namespace的，所以无法隔离，比如：time

同一台 Linux 启动的容器时间都是相同的。

```text
# 容器的时间和host一致，容器中不能修改时间
# 这里注意时间一致是指，从UTC1970年1月1日0时0分0秒起至现在的总秒数，不考虑闰秒。
# 而不是时区不同造成的时差
$ date -s "20091112 18:30:50"
date: cannot set date: Operation not permitted
Thu Nov 12 18:30:50 CST 2009
$ date
Mon Feb 24 22:36:55 CST 2020
​
# host和Container的这个UTC计时是不变的
$  date  "+%s"
1582620961
​
```

**The /proc/\[pid\]/ns/ directory**

 Each process has a /proc/\[pid\]/ns/ subdirectory containing one entry for each namespace that supports being manipulated by setns\(2\):

> 每个进程的都由setns\(\)创建file discrete 指向 它的 namespaces.

```text
$ ls -l /proc/$$/ns
total 0
lrwxrwxrwx. 1 mtk mtk 0 Jan 14 01:20 ipc -> ipc:[4026531839]
lrwxrwxrwx. 1 mtk mtk 0 Jan 14 01:20 mnt -> mnt:[4026531840]
lrwxrwxrwx. 1 mtk mtk 0 Jan 14 01:20 net -> net:[4026531956]
lrwxrwxrwx. 1 mtk mtk 0 Jan 14 01:20 pid -> pid:[4026531836]
lrwxrwxrwx. 1 mtk mtk 0 Jan 14 01:20 user -> user:[4026531837]
lrwxrwxrwx. 1 mtk mtk 0 Jan 14 01:20 uts -> uts:[4026531838]
​
$ readlink  /proc/17/ns/net
net:[4026531956]
​
```

In Linux 3.7 and earlier, these files were visible as hard links. Since Linux 3.8, they appear as symbolic links. If two processes are in the same namespace, then the inode numbers of their /proc/\[pid\]/ns/xxx symbolic links will be the same.

处在相同ns下的两个进程，`/proc/[pid]/ns/xxx`也是一样的

```text
# container network mode
$ docker run -itd --name test alpine sh
101d536509fc4321387b2b98bd04ee0cd79847d31bdbc68bb02c2c7319875358
$ docker run -itd --name same --net=container:test alpine sh
ca670526971e0964f030e7aa34d98281d5b753e17e552d2fae416f748d90cef8
​
# the process in container
$ docker top 101
UID    PID       PPID   C    STIME  TTY     TIME       CMD
root   6126    6110    0     14:22   pts/0   00:00:00    sh
$ docker top ca6
UID   PID    PPID   C      STIME    TTY    TIME    CMD
root  8221  8205  0       14:26    pts/0   00:00:00   sh
​
​
# Verify： container test net_ns is just like container same net_ns.
$ readlink /proc/6126/ns/net 
net:[4026532512]
$ readlink /proc/8221/ns/net 
net:[4026532512]
​
# Other namespaces is different.
$ readlink /proc/8221/ns/ipc 
ipc:[4026532574]
$ readlink /proc/6126/ns/ipc
ipc:[4026532508]
​
```

**The namespaces API**

 As well as various /proc files described below, the namespaces API includes the following system calls:

* clone\(2\)

  类似fork\(\)函数在创建进程时，可以通过`CLONE_NEW*`flag 来为新进程创建指定的namespace resource.

  clone\(\)创建新的进程，并将子进程加入新建namespace.

  The clone\(2\) system call creates a new process. If the flags argument of the call specifies one or more of the `CLONE_NEW*` flags listed below, then new namespaces are created for each flag, and the child process is made a member of those namespaces.

  > clone\(\) creates a new process, in a manner similar to fork\(2\).

  ```text
    // 创建并启动子进程，调用该函数后，父进程将继续往后执行，也就是执行后面的waitpid
      pid_t child_pid = clone(container_func,  // 子进程将执行container_func这个函数
                      container_stack + sizeof(container_stack),
                      // 这里SIGCHLD是子进程退出后返回给父进程的信号，跟namespace无关
                      SIGCHLD|CLON_NEWUTS,
                      NULL);  // 传给child_func的参数
  ```

* setns\(2\)

  setns\(\) 创建fd将进程加入到已存在的ns.

  Given a file descriptor referring to a namespace, reassociate the calling thread with that namespace.

  The content of this symbolic link is a string containing the namespace type and inode number as in the following example： `uts:[4026531838]`

  setns\(\)来生成`/proc/<PID>/ns/`目录下的各个文件

  ```text
  // int setns(int fd, int nstype);
  // nstype: 
  // 0      Allow any type of namespace to be joined.
  // other nstype： CLONE_NEW*
  ...
  fd = open(argv[1], O_RDONLY);  /* Get descriptor for namespace */
  if (setns(fd, 0) == -1)        /* Join that namespace */
                 errExit("setns");
  ...
  ```

* unshare\(2\)

  unshare\(\)让当前进程离开当前namespace加入到新的namespace.

  The unshare\(2\) system call moves the calling process to a new namespace.

  ```text
  #   -p, --pid[=file]
  #Unshare the pid namespace. If file is specified then persistent namespace is created by bind mount. See also the --fork and --mount-proc options.
  # --mount-proc 
  # Just  before  running the program, mount the proc filesystem at mountpoint (default is /proc).  This is useful when creating a new pid namespace.
  # --fork
  # Fork the specified program as a child process of unshare rather than running it directly.  
  # In Linux, when you run a new process, the process inherits its namespaces from the parent process. The way that you run a process in a new namespace is by "unsharing" the namespace with the parent process thus creating a new namespace.
  ​
  $ unshare --fork --pid --mount-proc bash
  $ ps -ef
  UID        PID  PPID  C STIME TTY          TIME CMD
  root         1     0  0 17:17 pts/0    00:00:00 bash
  root        11     1  0 17:17 pts/0    00:00:00 ps -ef
  $ sleep 1000&
  [1] 12
  $ ps -ef
  UID        PID  PPID  C STIME TTY          TIME CMD
  root         1     0  0 17:17 pts/0    00:00:00 bash
  root        12     1  0 17:18 pts/0    00:00:00 sleep 1000
  root        13     1  0 17:18 pts/0    00:00:00 ps -ef
  $ exit
  ​
  ```

**Container**

容器的隔离特性由kernel namespace feature提供。最典型的例子就是，每个容器有独立的Network Stack ，从而实现端口和网络的复用，example：

* 容器里面的TCP连接和宿主机看到的不一致。

  `docker container_id exec ss -aon`

* HAproxy的namespace有独特的ip和负载

  `nsenter -t <pid> -u -n -p -i` 可以不加`-m` 这个里利用宿主机上已安装的软件进行操作，比如`curl、ss`

了解namespace特性可以便捷的排查容器应用问题。

容器里什么命令都没有是因为容器的rootfs里什么都没有，但是宿主机有。在宿主机上通过`nsenter`不挂载容器的rootfs方式进入ns即不使用容器的Mount Namespace进入容器进行调试，便可以借助宿主机上工具排查容器问题。

```text
## 如下，指定进程只挂载了container的 uts ipc 和net ,不挂载mount，就可以借助宿主机的工具调试容器
$  nsenter --target  28580  --uts --ipc --net /bin/bash
root@e3a4e8ea5066:~# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1
    link/ipip 0.0.0.0 brd 0.0.0.0
145: eth0@if146: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:12:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.18.0.3/16 brd 172.18.255.255 scope global eth0
       valid_lft forever preferred_lft forever
​
​
root@e3a4e8ea5066:~# ping 172.18.0.5 -c 2
PING 172.18.0.5 (172.18.0.5) 56(84) bytes of data.
64 bytes from 172.18.0.5: icmp_seq=1 ttl=64 time=0.205 ms
64 bytes from 172.18.0.5: icmp_seq=2 ttl=64 time=0.107 ms
​
--- 172.18.0.5 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1000ms
rtt min/avg/max/mdev = 0.107/0.156/0.205/0.049 ms
root@e3a4e8ea5066:~# curl  -I 172.18.0.5
HTTP/1.1 200 OK
Server: nginx/1.15.7
Date: Thu, 17 Jan 2019 11:40:38 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 27 Nov 2018 12:31:56 GMT
Connection: keep-alive
ETag: "5bfd393c-264"
Accept-Ranges: bytes
​
```

**LXCFS**

容器的隔离实现基本就是通过Linux内核提供的这7种namespace实现。但是这些ns依旧没有实现完全的环境隔离。比如: `SELinux`,`Cgroups`以及`/sys`,`/proc/sys`, `/dev/sd*`等目录下的资源依据是没有被隔离的。因此在容器中通常使用的`ps`， `top`命令查看到的数据依旧是宿主机的数据。因为它们的数据来源于`/proc`等目录下的文件。如果想要在可视化的角度来实现这方便的可视化隔离。可以看看之前调研的[lxcfs对docker容器隔离](https://xigang.github.io/2018/06/30/lxcfs/)。

息。

```text
# ls /sys/fs/cgroup/memory/system.slice/docker-b8055e53c7c37a7850916bc00f314f760025423be4a164619494a4ac70852960.scope/
cgroup.clone_children           memory.kmem.tcp.max_usage_in_bytes  memory.oom_control
cgroup.event_control            memory.kmem.tcp.usage_in_bytes      memory.pressure_level
cgroup.procs                    memory.kmem.usage_in_bytes          memory.soft_limit_in_bytes
memory.failcnt                  memory.limit_in_bytes               memory.stat
memory.force_empty              memory.max_usage_in_bytes           memory.swappiness
memory.kmem.failcnt             memory.memsw.failcnt                memory.usage_in_bytes
memory.kmem.limit_in_bytes      memory.memsw.limit_in_bytes         memory.use_hierarchy
memory.kmem.max_usage_in_bytes  memory.memsw.max_usage_in_bytes     notify_on_release
memory.kmem.slabinfo            memory.memsw.usage_in_bytes         tasks
memory.kmem.tcp.failcnt         memory.move_charge_at_immigrate
memory.kmem.tcp.limit_in_bytes  memory.numa_stat
0.0 都是一个个目录啊。
$ docker  run -itd -m=10m    ad87b /bin/bash
b8055e53c7c37a7850916bc00f314f760025423be4a164619494a4ac70852960
## 由于docker的cgroupderive设置成了systemd所以是下面的层级结构
##  呐，内存确实被限制了
$ cat /sys/fs/cgroup/memory/system.slice/docker-b8055e53c7c37a7850916bc00f314f760025423be4a164619494a4ac70852960.scope/memory.max_usage_in_bytes 
10485760
# pwd
/sys/fs/cgroup/memory/system.slice/docker-6980bbe4ac5a8e24b1d324d24f54e9062452a5ab58f5af724656ec3c3f577e1a.scope
不做内存限制时，max_usage_memory如下：
$ cat memory.max_usage_in_bytes 
8634368
​
# but，容器里面显示还是8G，因为container里面没有读到cgroup。0.0
$ docker exec -it  698 /bin/bash
[root@6980bbe4ac5a /]# free -h
              total        used        free      shared  buff/cache   available
Mem:           7.6G        3.8G        1.3G        816K        2.6G        3.5G
Swap:          4.0G        6.9M        4.0G
​
```

参考:

[https://www.darkreading.com/risk/the-pros-and-cons-of-application-sandboxing/d/d-id/1138452](https://www.darkreading.com/risk/the-pros-and-cons-of-application-sandboxing/d/d-id/1138452)

[https://cizixs.com/2017/08/29/linux-namespace/](https://cizixs.com/2017/08/29/linux-namespace/)

