---
description: 推荐一个很好用的命令：systemd-cgtop
---

# Systemd and Cgroup

当 Linux 的 SysVinit 系统发展到 systemd 之后，systemd 与 cgroups 发生了融合\(或者说 systemd 提供了 cgroups 的使用和管理接口\)。本文将简单的介绍 cgroups 与 systemd 的关系以及如何通过 systemd 来配置和使用 cgroups。

> sysvinit 就是system V 风格的 init 系统.

**Systemd 的默认层级**

**通过将 cgroup 层级系统与 systemd unit 树绑定，systemd 可以把资源管理的设置从进程级别移至应用程序级别。因此，我们可以使用 systemctl 指令，或者通过修改 systemd unit 的配置文件来管理 unit 相关的资源。**

Systemd维护有自己的unit tree目录,可以通过 `mount |grep systemd|grep cgroup` 来查看。

我们可以通过 `systemd-cgls` 命令来查看 unit tree对应的cgroups 的层级结构：

The **Systemd** unit tree is made up of several parts:

* at the top, there is the root slice called **-.slice**,
* below, there are the **system.slice** \(**the default place for all system services**\), the **user.slice** \(the default place for all user sessions\) and the **machine.slice** \(the default place for all virtual machines and Linux containers\),
* still below there are **scopes** \(group of externally created processes started via **fork**\) and **services** \(group of processes created through a unit file\).

  可以认为scope是slice的打工仔。

  > scope : [http://man7.org/linux/man-pages/man5/systemd.scope.5.html](http://man7.org/linux/man-pages/man5/systemd.scope.5.html)
  >
  > Scope units are not configured via unit configuration files, but are only created programmatically using the bus interfaces of systemd. They are named similar to filenames. A unit whose name ends in ".scope" refers to a scope unit. Scopes units manage a set of system processes. Unlike service units, scope units manage externally created processes, and do not fork off processes on its own.
  >
  > The main purpose of scope units is grouping worker processes of a system service for organization and for managing resources.

默认情况下，系统会创建四种 slice，系统通过这些slice来划分和管理系统资源：

* **-.slice**：根 slice
* **system.slice**：所有系统 service 的默认位置，啧啧
* **user.slice**：所有用户会话的默认位置
* **machine.slice**：所有虚拟机和 Linux 容器的默认位置

```text
$ systemd-cgls |grep 'slice\|scope' 
-.slice
├─init.scope
├─system.slice (the default place for all system services)
│ ├─system-serial\x2dgetty.slice
└─user.slice
  └─user-0.slice
    ├─session-31378.scope
      └─init.scope
​
## 如下，docker.service 属于system.slice：
$ systemctl status docker
● docker.service - Docker Application Container Engine
   Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2018-12-03 14:49:11 CST; 3 weeks 3 days ago
     Docs: https://docs.docker.com
 Main PID: 32746 (dockerd)
    Tasks: 90
   Memory: 619.0M
      CPU: 8h 54min 14.127s
   CGroup: /system.slice/docker.service
           ...
           ├─19597 docker-containerd-shim -namespace moby -workdir /var/lib/docker/containerd/daemon/io.containerd.ru
           ├─32746 /usr/bin/dockerd -H fd://
           └─32753 docker-containerd --config /var/run/docker/containerd/containerd.toml
​
```

Note: If only system services run on a server, they get 100% of the available resources. If users connect to the server, they will get 100% of the available resources minus what the system services use. If both users and system services request 100% of the resources each, they will only get 50%. If there are system services, users and virtual machines and they all request 100% of the resources, they will only get 33% each. Any kind of slice can get 100% of the resources if nobody else wants them. But if resources are not available as much as one slice would, some limits occur.

**systemd hierarchy 和 cgroup hierarchy**

Systemd通过自己的API来调用cgroup的资源控制接口实现对应用进程的资源管理，所以Systemd自己维护的Systemd Unit Tree目录会在Cgroup Hierarchy中有一个对应是映射关系。例如，sshd.service 属于 system.slice，会直接映射到 cgroup\_hierarchy /system.slice/ssshd.service/ 中即在cgroup hierarchy也存在对应的目录。

```text
​
## systemd目录下的system.slice在cgroup各个hierarchy都有映射
$ find /sys/fs/cgroup/ -name "system.slice"
/sys/fs/cgroup/cpu,cpuacct/system.slice
/sys/fs/cgroup/memory/system.slice
/sys/fs/cgroup/devices/system.slice
/sys/fs/cgroup/blkio/system.slice
/sys/fs/cgroup/pids/system.slice
/sys/fs/cgroup/systemd/system.slice 
​
## transient service or scope 
# systemd-run may be used to create and start a transient .service or .scope unit and run the specified COMMAND in it. 
--unit=
 Use this unit name instead of an automatically generated one.
--scope
 Create a transient .scope unit instead of the default transient .service unit.
--slice=
 Make the new .service or .scope unit part of the specified slice, instead of the system.slice.
​
## systemd-run - Run programs in transient scope units, service units, or timer-scheduled service units
## transient
## 在system.slice 下创建一个service unit
## 没有--unit参数时，生成随机的unit name
$ systemd-run --slice=system sleep 600
Running as unit run-rcad2a007efa64bc7b1195db070547be7.service.
​
$ systemd-run --slice=system --unit s_600 sleep 600
Running as unit s_600.service.
## 可以看到映射关系
$ cat  /sys/fs/cgroup/systemd/system.slice/s_600.service/tasks 
24877
$ cat  /sys/fs/cgroup/cpu,cpuacct/system.slice/s_600.service/tasks 
24877
​
## reboot host, the transient scope units is disappear.
$ uptime
 16:47:32 up 0 min,  1 user,  load average: 0.89, 0.22, 0.07
$ ls /sys/fs/cgroup/systemd/system.slice/s__600.service
ls: cannot access '/sys/fs/cgroup/systemd/system.slice/s__600.service': No such file or directory
​
​
## 创建新的slice
$ systemd-run --slice=test sleep 60
Running as unit run-rc64a2fc981b94f63a14ef97fbdf85ff1.service.
$ root@fishong:~# systemd-cgls 
Control group /:
-.slice
├─test.slice
│ └─run-r1f2c1d2cf05048dd8f543c898ba37162.service
│   └─22596 /bin/sleep 60
​
$ cat /proc/22596/cgroup 
11:hugetlb:/
10:cpu,cpuacct:/test.slice
9:blkio:/test.slice
8:cpuset:/
7:pids:/test.slice
6:perf_event:/
5:net_cls,net_prio:/
4:devices:/test.slice
3:freezer:/
2:memory:/test.slice
1:name=systemd:/test.slice/run-r1f2c1d2cf05048dd8f543c898ba37162.service
​
​
```

**Systemd实现应用资源控制**

cgroup v1和v2

cgroup有两个版本，新版本的cgroup v2即Unified cgroup\(参考[cgroup v2](https://www.kernel.org/doc/Documentation/cgroup-v2.txt)\)和传统的cgroup v1（参考[cgroup v1](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt)），在新版的Linux（4.x）上，v1和v2同时存在，但同一种资源（CPU、内存、IO等）只能用v1或者v2一种cgroup版本进行控制。systemd同时支持这两个版本，并在设置时为两者之间做相应的转换。对于每个控制器，如果设置了cgroup v2的配置，则忽略所有v1的相关配置。 在systemd配置选项上，cgroup v2相比cgroup v1有如下不一样的地方： 1.CPU： `CPUWeight=`和`StartupCPUWeight=`取代了`CPUShares=`和`StartupCPUShares=`。cgroup v2没有"cpuacct"控制器。 2.Memory：`MemoryMax=`取代了`MemoryLimit=`. `MemoryLow=` and `MemoryHigh=`只在cgroup v2上支持。 3.IO：`BlockIO`前缀取代了`IO`前缀。在cgroup v2，Buffered写入也统计在了cgroup写IO里，这是cgroup v1一直存在的问题。

前面已经科普了systemd 是通过cgroup来实现对应用的资源限制。具体是设置方式有两种：

* 通过配置unit文件修改 cgroup

  直接查阅资源的对应配置参数，在unit file中填入合适的值就行。

  ```text
  ## 这是cgroup V1的一些参数用法，centos 7还在沿用但是RHEL 7 已经使用Cgroup V2了
  ## https://access.redhat.com/articles/3735611
  [Unit]
  Description=xx Server
  ​
  [Service]
  ExecStart=/usr/bin/xx
  ​
  LimitNOFILE=102400
  LimitNPROC=102400
  LimitCORE=infinity
  Restart=on-failure
  KillMode=process
  MemoryLimit=1G
  # 进程获取CPU运行时间的权重值，对应cgroup的"cpu.shares"参数，取值范围2-262144，默认值1024。
  CPUShares=2048
  ​
  [Install]
  WantedBy=multi-user.target
  ​
  ​
  ```

* 通过 systemctl 命令修改 cgroup的资源控制

  默认`systemctl set-property`会将属性写入unit file.

  To get the current **CPUShares** service value, type:

  ```text
  $ systemctl show -p CPUShares httpd
  CPUShares=500
  ```

  Or:

  `systemctl show service_name`可以列出该service的全部属性值。

  ```text
  $ systemctl show httpd | grep CPUShares
  CPUShares=500
  ```

  Note: Each time a resource limit is set on a service, a directory of the same name with the **.d**suffix is created in **/etc/systemd/system**. For example, in the previous case, a directory named **/etc/systemd/system/httpd.service.d** is created with a file called **90-CPUShares.conf** in it and the following content:

  ```text
  [Service]
  CPUShares=500
  ```

  To put resource limits on a service \(here **500** **CPUShares**\), type:

  ```text
  $ systemctl set-property httpd CPUShares=500
  $ systemctl daemon-reload
  ```

  Note1: The change is written into the service unit file. Use the **–runtime** option to avoid this behaviour. Note2: By default, each service owns 1024 **CPUShares**. Nothing prevents you from giving a value smaller or bigger.

**验证Cgroup的继承性**

验证：A child task automatically inherits the cgroup membership of its parent.

```text
##  cgexec - run the task in given control groups
$ cgcreate -a root:wyb -g memory:test
$ echo 10000000 >  /sys/fs/cgroup/memory/test/memory.kmem.limit_in_bytes
$ cgexec  -g memory:test sleep 66000
[1] 5103
## 因为是ssh过来的默认继承了ssh.service，所以大多数cgroup默认是/system.slice/ssh.service
$ cat /proc/5103/cgroup  
11:hugetlb:/
10:cpu,cpuacct:/system.slice/ssh.service
9:blkio:/system.slice/ssh.service
8:cpuset:/
7:pids:/system.slice/ssh.service
6:perf_event:/
5:net_cls,net_prio:/
4:devices:/system.slice/ssh.service 
3:freezer:/
2:memory:/test #(因为对memory做了修改，所以这里从继承的cgroup切换到自定义的cgroup了)
1:name=systemd:/system.slice/ssh.service
​
$ systemd-cgls |grep sleep -B 10
  ├─ssh.service
  │ ├─1701 /usr/sbin/sshd -D
  │ ├─1722 sshd: root@pts/0
  │ ├─1724 -bash
  │ ├─2365 sshd: root@pts/1
  │ ├─2367 -bash
  │ ├─3863 sshd: root@pts/2
  │ ├─3865 -bash
  │ ├─4228 sshd: root@pts/3
  │ ├─4230 -bash
  │ ├─5103 sleep 66000
  │ ├─5884 systemd-cgls
  │ └─5885 grep sleep -B 10
​
$ systemd-cgls 
...
├─system.slice
...
│ ├─ssh.service
...
│ │ ├─25386 sleep 66000
```

---- 以下是大神的例子就不翻译了 ---

**Some Other Examples**

As previously seen, system processes get 1024 CPUShares, user processes 1024 CPUShares and virtual machines 1024 CPUShares, which means 33% of CPU each by default \(you will only see it if each category executes some tasks\). To allocate 70% of CPU to the system processes, 20% of CPU to the user processes and 10% of CPU to the virtual machines, type:

> 相当于初始化系统资源

```text
$ systemctl set-property system.slice CPUShares=7168
$ systemctl set-property user.slice CPUShares=2048
$ systemctl set-property machine.slice CPUShares=1024
```

Note: Rebooting might be necessary to see the changes.

To restrict the user with the uid **1000** to use less than **20%** of cpu, type:

```text
$ systemctl set-property user-1000.slice CPUQuota=20%
```

To reduce the memory available for the same user to **1GB**, type:

```text
$ systemctl set-property user-1000.slice MemoryLimit=1024M
```

To limit the **mariadb** service to write below **2MB/s** onto the **/dev/vdb** partition, type:

```text
$ systemctl set-property mariadb.service BlockIOWriteBandwidth="/dev/vdb 2M"
```

**Case Study**

To better understand **CGroups**, let’s take an example. You want to run a website but you’ve got only one server. You plan to use the classical **LAMP** stack \(**Linux**, here **Centos 7**, **Apache**, **MariaDB** and **PHP**\).

Your server’s got **4**Gigabytes of memory and you want to allocate resources as follows:

* **Apache** service \(here **httpd.service**\): **40**% of CPU, **500M** of memory,
* **PHP** service \(here **php-fpm.service**\): **30**% of CPU, **1**G of memory,
* **MariaDB** service \(here **mariadb.service**\): **30**% of CPU, **1**G of memory.

You leave **1**G of memory for the other processes \(system, etc\).

Note1: The values given are only for the sake of the discussion. Note2: If you don’t configure **CGroups**, everything will work like in **RHEL 6**: all the processes will share the server power and the memory as they need.

As all your **LAMP** services are started from a **Systemd** unit file, they will be added in the **system.slice**.

Here is the configuration to set up with the **systemctl set-property** command:

* **Apache** service: **CPUShares**=**4096** \(4 x 1024\); **MemoryLimit**=**500M**,
* **PHP** service: **CPUShares**=**3072** \(3 x 1024\); **MemoryLimit**=**1G**,
* **MariaDB** service: **CPUShares**=**3072** \(3 x 1024\); **MemoryLimit**=**1G**,

Note1: The **Apache** service will get **4096/\(4096+3072+3072\) CPUShares**, the **PHP** service will get **3072/\(4096+3072+3072\) CPUShares**, etc. Note2: There are some other services in the **system.slice** \(**crond**, **postfix**, **chronyd**, etc\). But, as they are not very hungry, they will not consume their default allocated CPU resources \(**1024**\) and will not change anything to the situation. However, even though the **Apache+MariaDB+PHP**services use all their CPU resources, because the way it works, there will be still some resources for the other services.

**Caution**: Once you set up **CPUShares CGroup** restriction on one service in the **system.slice**, all the services there get **CPUShares CGroup** activated: even though you don’t specify anything, all new service started will be restricted to **1024** **CPUShares** by default. **It is not possible to CPU-restrict some services and let the others without restriction.** For a detailed explanation of the mechanism, see **All control groups belong to us!** video below in the **Additional Resources** section.

**Systemd features**

日后再来补充吧，现在感悟不深。

推荐一个很好用的命令：`systemd-cgtop`

Here are some examples of the features of systemd:

* **Logging**: Now, all system messages come in on a single stream and are stored in the **/run** directory. Messages can then be consumed by the rsyslog facility \(and redirected to traditional log files in the **/var/log** directory or to remote log servers\) or displayed using the **journalctl** command across a variety of attributes.

  journal接管全部日志

* **Dependencies**: With systemd, an explicit set of dependencies can be defined for each service, instead of being implied by boot order. This allows a service to start at any point that its dependencies are met. In this way, many services can start at the same time, making the boot process faster. Likewise, complex sets of dependencies can be set up, so the exact requirements of a service \(such as storage availability or file system checking\) can be met before a service starts.

  虽然 systemd 将大量的启动工作解除了依赖，使得它们可以并发启动。但还是存在有些任务，它们之间存在天生的依赖，不能用**"套接字激活"\(socket activation\)、D-Bus activation 和 autofs** 三大方法来解除依赖。比如：挂载必须等待挂载点在文件系统中被创建；挂载也必须等待相应的物理设备就绪。为了解决这类依赖问题，systemd 的配置单元之间可以彼此定义依赖关系。Systemd 用配置单元定义文件中的关键字来描述配置单元之间的依赖关系。比如：unit A 依赖 unit B，可以在 unit B 的定义中用"require A"来表示。这样 systemd 就会保证先启动 A 再启动 B。

* **Cgroups**: Services are identified by Cgroups, which allow every component of a service to be managed. For example, the older System V init scripts would start a service by launching a process which itself might start other child processes. When the service was killed, it was hoped that the parent process would do the right thing and kill its children. By using Cgroups, all components of a service have a tag that can be used to make sure that all of those components are properly started or stopped.

  > 着重看看。
  >
  > Cgroup 通过service来启停该service下的所有进程。
  >
  > 而不是像System V init 那样，子进程的关闭依赖其父进程来完成。

* **Activating services**: Services don't just have to be always running or not running based on runlevel, as they were previous to systemd. Services can now be activated based on path, socket, bus, timer, or hardware activation. Likewise, because systemd can set up sockets, if a process handling communications goes away, the process that starts up in its place can pick up the next message from the socket. To the clients using the service, it can look as though the service continued without interruption.
* **More than services**: Instead of just managing services, systemd can manage several different unit types. These unit types include:
  * **Devices**: Create and use devices.
  * **Mounts and automounts**: Mount file systems upon request or automount a file system based on a request for a file or directory within that file system.
  * **Paths**: Check the existence of files or directories or create them as needed.
  * **Services**: Start a service, which often means launching a service daemon and related components.
  * **Slices**: Divide up computer resources \(such as CPU and memory\) and apply them to selected units.
  * **Snapshots**: Take snapshots of the current state of the system.
  * **Sockets**: Set up sockets to allow communication paths to processes that can remain in place, even if the underlying process needs to restart.
  * **Swaps**: Create and use swap files or swap partitions.
  * **Targets**: Manage a set of services under a single unit, represented by a target name rather than a runlevel number.
  * **Timers**: Trigger actions based on a timer.
* **Resource management**
  * The fact that each systemd unit is always associated with its own cgroup lets you control the amount of resources each service can use. For example, you can set a percent of CPU usage by service which can put a cap on the total amount of CPU that service can use -- in other words, spinning off more processes won't allow more resources to be consumed by the service. Prior to systemd, _nice_ levels were often used to prevent processes from hogging precious CPU time. With systemd's use of cgroups, precise limits can be set on CPU and memory usage, as well as other resources.
  * A feature called _slices_ lets you slice up many different types of system resources and assign them to users, services, virtual machines, and other units. Accounting is also done on these resources, which can allow you to charge customers for their resource usage.

参考：

[https://www.certdepot.net/rhel7-get-started-cgroups/](https://www.certdepot.net/rhel7-get-started-cgroups/)

[https://www.cnblogs.com/jimbo17/p/9107052.html](https://www.cnblogs.com/jimbo17/p/9107052.html)

