---
description: 'Control Groups: isolate/limits resource'
---

# The concept of cgroup

容器的目的是进行**资源隔离**和**控制隔离**。

* 资源隔离：隔离计算资源，如CPU、MEMORY、DISK等。
* 控制隔离：隔离一些控制结构，如UID、PID等

 **资源隔离**依赖于linux内核的Cgroup实现，**控制隔离**依赖于linux内核的namespace.

> The Linux kernel provides the cgroups functionality that allows limitation and prioritization of resources \(CPU, memory, block I/O, network, etc.\) without the need for starting any virtual machines, and also namespace isolation functionality that allows complete isolation of an applications’ view of the operating environment, including process trees, networking, user IDs and mounted file systems.



**What are cgroups ?**

A _cgroup_ associates a set of tasks with a set of parameters for one or more subsystems.

Cgroup将指定的tasks\(processes\)与指定的subsystems关联，从而实现进程的资源控制。

下面通过对subsystems 和 hierarchy的介绍来，来对Cgroup有个更清晰的认识。

安装`libcgroup-tools` 软件包，即可获取`cgroup`的操纵命令。

* subsystem ：resource controller，用来调度和限制进程资源

  每个subsystem会自带许多的文件来控制该子系统的资源、

  ```text

  ## list all subsystems
  $ lssubsys  -m  ## 列出挂载点
  cpuset /sys/fs/cgroup/cpuset
  cpu,cpuacct /sys/fs/cgroup/cpu,cpuacct
  blkio /sys/fs/cgroup/blkio
  memory /sys/fs/cgroup/memory
  devices /sys/fs/cgroup/devices
  freezer /sys/fs/cgroup/freezer
  net_cls,net_prio /sys/fs/cgroup/net_cls,net_prio
  perf_event /sys/fs/cgroup/perf_event
  hugetlb /sys/fs/cgroup/hugetlb
  pids /sys/fs/cgroup/pids
  ​

  ## 每个subsystem中的参数文件
  $ ls /sys/fs/cgroup/memory/
  cgroup.clone_children  init.scope	    memory.kmem.limit_in_bytes	    memory.kmem.tcp.max_usage_in_bytes	memory.move_charge_at_immigrate  memory.stat		release_agent
  cgroup.event_control   kubepods		    memory.kmem.max_usage_in_bytes  memory.kmem.tcp.usage_in_bytes	memory.numa_stat		 memory.swappiness	system.slice
  cgroup.procs	       memory.failcnt	    memory.kmem.slabinfo	    memory.kmem.usage_in_bytes		memory.oom_control		 memory.usage_in_bytes	tasks
  cgroup.sane_behavior   memory.force_empty   memory.kmem.tcp.failcnt	    memory.limit_in_bytes		memory.pressure_level		 memory.use_hierarchy	user.slice
  docker		       memory.kmem.failcnt  memory.kmem.tcp.limit_in_bytes  memory.max_usage_in_bytes		memory.soft_limit_in_bytes	 notify_on_release

  ```

* hierarchy：每一个hierarchy即subsystem mount point是一个目录，必有一个或多个subsystem 挂载在上面。  
  A _hierarchy_ is a set of cgroups arranged in a tree, such that every task in the system is in exactly one of the cgroups in the hierarchy, and a set of subsystems.

  ```text
  ## list all cgroups
  ## <controllers>:<path>
  ## defines the control groups whose subgroups will be shown. 
 
  ## 每个subsystem的挂载点可以被看做一个root hierarchy
  $lscgroup |grep '/$'  
  memory:/  ## 下面有好多的control group
  freezer:/
  devices:/
  net_cls,net_prio:/
  perf_event:/
  pids:/
  cpuset:/
  blkio:/
  cpu,cpuacct:/
  hugetlb:/
  ```

* cgroup = （subsystem+ hierarchy） +  declare new\_cgroup

  cgroup即在hierarchy目录下新建limit resource目录--

  ```text
  ## 创建两个cgroup, 继承CPU_Subsystem和Memory_Subsystem
  ## 即会在cpu和memory的这个两个root hierarchy下各建一个名为mycoolgroup和mycoolgroup/test的目录
  ## -g <controllers>:<path> controllers即subsystem
  ## defines  control  groups to be added.  controllers is a list of controllers and path is the relative path to control groups in the given controllers list.
  ## Charater "*" can be used as a shortcut for "all mounted controllers".

  ##
  $ cgcreate -a root:wyb -g memory,cpu:test_cg    
  $ cgcreate -a root:wyb -g memory,cpu:test_cg/test
  $ lscgroup |grep test
  memory:/test_cg
  memory:/test_cg/test
  cpu,cpuacct:/test_cg
  cpu,cpuacct:/test_cg/test


  ## 每个subsystem都有许多自带的参数文件来实现资源的限制
  $ ls -l /sys/fs/cgroup/memory/test_cg/ |wc -l
  29
  $ ls -l /sys/fs/cgroup/cpu/test_cg/ |wc -l
  13
     

  ​
  ## 基于新建的cgroup来启动process
  $ cgexec -g memory:test_cg sleep 666  &
  [1] 25205
  $ cat /sys/fs/cgroup/memory/test_cg/tasks 
  25205


  ## 或者 先启动进程，然后将进程pid手动写到cgroup_dir/tasks 文件中
  $ sleeep 666 &
  [1] 21928
  $ echo '21928' > /sys/fs/cgroup/memory/test_cg/tasks
  ```

**How do I use cgroups ?**

