# Docker and Cgroup

Docker底层也是通过Cgroup来对容器资源进行限制，具体的Docker资源限制参数请自行查询，这里就不赘述了。

> Docker 和 Systemd都是通自己的api去配置cgroup目录来达到容器/进程的资源限制。所以本质上来说，它们并没有多大区别

**docker cgroupdriver：cgroupfs**

默认情况下，Docker 启动一个容器后，会在 /sys/fs/cgroup 目录下的各个资源目录下生成以容器 ID 为名字的目录（group），比如：

```text
## /sys/fs/cgroup/cpu/docker/Container_ID
## 启动容器
$ docker run -d --name web --cpu-quota 25000 --cpu-period 100 --cpu-shares 30 nginx
06bd180cd340f8288c18e8f0e01ade66d066058dd053ef46161eb682
​
## 验证
$ cat /sys/fs/cgroup/cpu/docker/06bd180cd340f8288c18e8f0e01ade66d066058dd053ef46161eb682ab69ec24/cpu.cfs_quota_us
25000
$ cat /sys/fs/cgroup/cpu/docker/06bd180cd340f8288c18e8f0e01ade66d066058dd053ef46161eb682ab69ec24/cpu.cfs_period_us
2000
```

**docker cgroupdriver：systemd**

从上面可以看到Docker通过守护进程维护了自己的cgroup目录，而Systemd本身已经高度集成了cgroup，因为我们可以修改docker的cgroup驱动参数，将systemd作为默认的cgroup管理工具，这样其他进程调用docker runtime时，更加稳健，比如：kubelet.

```text
## 修改native.cgroupdriver
$ cat /etc/docker/daemon.json 
{
"exec-opts": ["native.cgroupdriver=systemd"]
}
​
# 启动容器并查看容器所在cgroup目录
$ docker run -itd -m 512m 21.48.0.230/tools/echoserver:1.4 
25a085a3c8186cc0f691acad85b735a763ec389fd14e426f5ae9e51d5eef6ce8
$ ls /sys/fs/cgroup/memory/docker
ls: cannot access /sys/fs/cgroup/memory/docker: No such file or directory
$ cat /sys/fs/cgroup/memory/system.slice/docker-25a085a3c8186cc0f691acad85b735a763ec389fd14e426f5ae9e51d5eef6ce8.scope/memory.limit_in_bytes
536870912
​
​
$ systemd-cgls
...
  ├─system.slice
  	├─docker-25a085a3c8186cc0f691acad85b735a763ec389fd14e426f5ae9e51d5eef6ce8.scope
	  ├─ 2592 nginx: master ...
	  ├─ 2592 nginx: worker ...		
	

## 对比 native.cgroupdriver=cgroupfs ,cgroupfs是默认值
$ cat /etc/docker/daemon.json 
{
"exec-opts": ["native.cgroupdriver=cgroupfs"]
}
$ ls -ld /sys/fs/cgroup/memory/docker/
drwxr-xr-x 4 root root 0 Nov 26 14:58 /sys/fs/cgroup/memory/docker/
$ ls -ld /sys/fs/cgroup/cpu/docker/
drwxr-xr-x 4 root root 0 Nov 26 14:58 /sys/fs/cgroup/cpu/docker/
#  使用相同image去测试--
$ docker run -itd -m 512m 21.48.0.230/tools/echoserver:1.4 
53b6e64550442a13b9251bcfe247d24528377a044e1d04f6ab824f7fba830d61
$ cat /sys/fs/cgroup/memory/docker/53b6e64550442a13b9251bcfe247d24528377a044e1d04f6ab824f7fba830d61/memory.limit_in_bytes 
536870912
```

这样做的理由：

[https://kubernetes.io/docs/setup/production-environment/container-runtimes/\#cgroup-drivers](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cgroup-drivers)

When systemd is chosen as the init system for a Linux distribution, the init process generates and consumes a root control group \(`cgroup`\) and acts as a cgroup manager. Systemd has a tight integration with cgroups and will allocate cgroups per process. It’s possible to configure your container runtime and the kubelet to use `cgroupfs`. **Using `cgroupfs` alongside systemd means that there will then be two different cgroup managers.**

Control groups are used to constrain resources that are allocated to processes. A single cgroup manager will simplify the view of what resources are being allocated and will by default have a more consistent view of the available and in-use resources. When we have two managers we end up with two views of those resources. We have seen cases in the field where nodes that are configured to use `cgroupfs` for the kubelet and Docker, and `systemd` for the rest of the processes running on the node becomes unstable under resource pressure.

**Changing the settings such that your container runtime and kubelet use `systemd` as the cgroup driver stabilized the system. Please note the `native.cgroupdriver=systemd` option in the Docker setup below.**

大致意思，Systemd已经高度集成了Cgroup了。不要在使用docker 默认的`native.cgroupdriver=cgropufs`管理方式。因为存在kubelet默认使用`systemd`作为cgroup\_drivers 而docker 默认使用`cgroups`作为cgroup\_drivers。两者若不统一则会报错（因为存在两个resources managers）。所以生成环境需要统一，设置成`systemd`.即给docker daemon配置 `native.cgroupdriver=systemd` 。

