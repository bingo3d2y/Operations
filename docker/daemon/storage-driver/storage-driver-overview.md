# Storage Driver Overview

了解docker storage driver有助于我们理解`docker build`是如何执行的，`docker images`是怎样存储的，以及容器是如何使用images的.

**About storage drivers**

**COW**

cow，copy-on-write ,字面意思：需要修改时才去复制。

cow是一种计算机程序设计领域的优化策略。其核心思想是，如果有多个调用者（callers）同时请求相同资源（如内存或磁盘上的数据存储），他们会共同获取相同的指针指向相同的资源，直到某个调用者试图修改资源的内容时，系统才会真正复制一份专用副本（private copy）给该调用者，而其他调用者所见到的最初的资源仍然保持不变。这过程对其他的调用者都是透明的（transparently）。此作法主要的优点是如果调用者没有修改该资源，就不会有副本（private copy）被建立，因此多个调用者只是读取操作时可以共享同一份资源。

**linux process cow**

fork\(\)会产生一个和父进程完全相同的子进程\(除了pid\)，但是往往fork\(\)会结合exec\(\)连用，即创造的子进程会和父进程做不同的事情，这样创建子进程时复制过去的数据是没用的\(因为子进程执行`exec()`，原有的数据会被清空\)。

基于cop on worite的优化

fork\(\)之后，kernel把父进程中所有的内存页的权限都设为read-only，然后子进程的地址空间指向父进程。当父子进程都只读内存时，相安无事。当其中某个进程写内存时，CPU硬件检测到内存页是read-only的，于是触发页异常中断（page-fault），陷入kernel的一个中断例程。中断例程中，kernel就会**把触发的异常的页复制一份**，于是父子进程各自持有独立的一份。

**filesystem cow**

Copy-on-write在对数据进行修改的时候，**不会直接在原来的数据位置上进行操作**，而是重新找个位置修改，这样的好处是一旦系统突然断电，重启之后不需要做Fsck。好处就是能**保证数据的完整性，掉电的话容易恢复**。

* 比如说：要修改数据块A的内容，先把A读出来，写到B块里面去。如果这时候断电了，原来A的内容还在！

**pros**

实现资源的共享，减少不必要的复制，减少资源损耗。

**cons**

对大文件操作不友好，如果一个10G的文件，在COW的机制下，纵然是只修改1k的内容，也要把整个10g的文件copy过来。

**UnionFS**

Union File System, 联合文件系统，所谓UnionFS就是把不同物理位置的目录合并mount到同一个目录中。

把多个目录 mount 成一个，那么读写操作是怎样的呢，不同是驱动实现方式是不一样的：

**OverlaFS和AUFS的实现类似:**

* 默认情况下，最上层的目录为读写层，只能有一个；下面可以有一个或者多个只读层
* 读文件，打开文件的时候使用了 `O_RDONLY` 选项：从最上面一个开始往下逐层去找，打开第一个找到的文件，读取其中的内容

  即上层文化会屏蔽下次文件，因为是“深度优先搜索”。

* 写文件，打开文件时用了`O_WRONLY` 或者 `O_RDWR`选项
  * 如果在最上层找到了该文件，直接打开
  * 否则，从上往下开始查找，找到文件后，把文件复制到最上层，然后再打开这个 copy（所以，如果要读写的文件很大，这个过程耗时会很久 ，cow机制的弊端）
* 重命名：同层文件调用传统的`rename()`， 不同层的重命名操作，需要借助`EXDEV`自己捕获异常并处理即先copy在rename.
* 删除文件：在最上层创建一个 whiteout 文件，从而达到unlink\(屏蔽底层文件\)。

  aufs是：`.wh.<origin_file_name>` ,overlay/overlay2则直接就是`<origin_file_name>`

  即UnionFS无法实现真正的删除，它的删除是通在上层创建whiteout文件来屏蔽底层的文件。

Device mapper:

\*\*\*\*

**images layers**

A Docker image is built up from a series of read-only layers.

Each layer is only a set of differences from the layer before it.

Docker镜像由多层不可变的layers组成，每一层都记录与前面一层不同的文件，通过UnionFS实现统一访问。

![](../../../.gitbook/assets/container-layers.jpg)

**Container layers**

The major difference between a container and an image is the top writable layer. All writes to the container that add new or modify existing data are stored in this writable layer. When the container is deleted, the writable layer is also deleted. The underlying image remains unchanged.

容器层位于镜像层的最顶端，容器层是可读写的，容器运行产生的数据会存储在容器层，但也会随之容器的删除，容器层数据也会丢失。

基于一个镜像可以启动多个容器，这些容器各自维护自己container layers，如下图所示。

![](../../../.gitbook/assets/sharing-layers.jpg)

理想状态下，应该将数据都写入容器的数据卷，从而避免向容器的写入层写入数据，这样数据就不会随着容器启停而丢失。

However, some workloads require you to be able to write to the container’s writable layer. This is where storage drivers come in.

**Docker storage drivers**

The storage driver controls how images and containers are stored and managed on your Docker host.

镜像和容器的存储和管理都是storage driver来负责的。不同的storage driver，性能上也会有差异。

Docker 支持的存储驱动有： overlay/overlay2、aufs、devicemapper、btrfs、zfs and vfs.

其中`btrfs` 和 `zfs` 需要对应的宿主机文件系统支持，提供快照功能但需要更多维护和选项设置，不常用。

> The `btrfs` and `zfs` storage drivers are used if they are the backing filesystem \(the filesystem of the host on which Docker is installed\). These filesystems allow for advanced options, such as creating “snapshots”, but require more maintenance and setup. Each of these relies on the backing filesystem being configured correctly.

`vfs` 倾向于测试使用，或者是不支持COW的机器上使用，更不常见。

> The `vfs` storage driver is intended for testing purposes, and for situations where no copy-on-write filesystem can be used. Performance of this storage driver is poor, and is not generally recommended for production use.

目前主流常见的storage drivers是： `overlay/overlay2` 、`devicemapper` 和 `aufs`

* `overlay/overlay2`: 目前主流使用overlay2，支持所有的linux发行版，也无需额外的配置。

  对内核版本有一定要求： **you need version 4.0 or higher of the Linux kernel, or RHEL or CentOS using version 3.10.0-514 and above.**

  > overlay2是主流方案，devicemapper和aufs是针对不同linux distributions的候补方案。

* `devicemapper` : 用于内核版本较老不支持overlay2的CentOS and RHEL。

  `devicemapper` 有两种实现方式，`direct-lvm` 和 `loopback-lvm`.其中`loopback-lvm`性能很差只能用于测试环境，生成环境中需要配置`loopback-lvm`模式。

* `aufs` : 用于Ubuntu 14.04 on kernel 3.13 which has no support for `overlay2`.

除了内核版本的要求外，不同的storage drivers对filesystem也有要求，如下表：

With regard to Docker, the backing filesystem is the filesystem where `/var/lib/docker/` is located. Some storage drivers only work with specific backing filesystems.

| Storage driver | Supported backing filesystems |
| :--- | :--- |
| `overlay2`, `overlay` | `xfs` with ftype=1, `ext4` |
| `aufs` | `xfs`, `ext4` |
| `devicemapper` | `direct-lvm` |
| `btrfs` | `btrfs` |
| `zfs` | `zfs` |
| `vfs` | any filesystem |

本篇简单讲述下docker支持的存储驱动，针对不同存储驱动的特性和优化，会有对应的篇幅去详细介绍。

参考：

[https://docs.docker.com/storage/storagedriver/](https://docs.docker.com/storage/storagedriver/)

[https://docs.docker.com/storage/storagedriver/select-storage-driver/](https://docs.docker.com/storage/storagedriver/select-storage-driver/)

[https://juejin.im/post/5bd96bcaf265da396b72f855](https://juejin.im/post/5bd96bcaf265da396b72f855)

