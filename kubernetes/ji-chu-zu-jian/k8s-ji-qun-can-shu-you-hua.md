# k8s集群参数优化

#### 优化配置（卧槽，整理下）

**内核参数**

容器还是进程，进程还是依赖宿主机的内核的参数优化的。

open file 耗尽导致网络连接:

问题的具体表现包括：服务域名监控返回502，用户无法访问服务。在容器中无法连上Redis服务器、容器中无法对Redis域名进行DNS解析、容器中无法解析任何域名。

原因

根据同事的反馈逐项进行排查，发现容器系统服务正常，DNS服务正常，问题似乎并不是出在容器系统服务上。

排查calico网络，发现3个节点的calico-node处于`start`状态（正常状态是`up`），而这三个异常节点正是项目externalIPs所绑定的节点。

查看calico-node的log发现如下信息：

```text
Active Socket: Connection reset by peer
```

显然，这三个节点的calico-node处于异常状态。尝试重启`calico-node`，但问题并未得到解决。

查看节点的网络连接，发现大量连接处于TIME\_WAIT状态。查看nofile限制，发现上限是1024（Ubuntu默认配置）：

```text
# ulimit -n
1024
```

修改nofile限制，重启calico-node，问题解决。

解决方案

修改节点的`/etc/security/limits.conf` ，将`root`用户的open file数增加到102400：

```text
root soft nofile 102400
root hard nofile 102400
* soft nofile 102400
* hard nofile 102400
* hard nproc 8192
* soft nproc 8192
```

将修改后的系统做成镜像，以后申请机器时采用新镜像。

```text
fs.file-max=1000000
# max-file 表示系统级别的能够打开的文件句柄的数量， 一般如果遇到文件句柄达到上限时，会碰到
"Too many open files"或者Socket/File: Can’t open so many files等错误。
​
# 配置arp cache 大小
net.ipv4.neigh.default.gc_thresh1=1024
# 存在于ARP高速缓存中的最少层数，如果少于这个数，垃圾收集器将不会运行。缺省值是128。
​
net.ipv4.neigh.default.gc_thresh2=4096
# 保存在 ARP 高速缓存中的最多的记录软限制。垃圾收集器在开始收集前，允许记录数超过这个数字 5 秒。缺省值是 512。
​
net.ipv4.neigh.default.gc_thresh3=8192
# 保存在 ARP 高速缓存中的最多记录的硬限制，一旦高速缓存中的数目高于此，垃圾收集器将马上运行。缺省值是1024。
以上三个参数，当内核维护的arp表过于庞大时候，可以考虑优化
​
net.netfilter.nf_conntrack_max=10485760
# 允许的最大跟踪连接条目，是在内核内存中netfilter可以同时处理的“任务”（连接跟踪条目）
net.netfilter.nf_conntrack_tcp_timeout_established=300
net.netfilter.nf_conntrack_buckets=655360
# 哈希表大小（只读）（64位系统、8G内存默认 65536，16G翻倍，如此类推）
net.core.netdev_max_backlog=10000
# 每个网络接口接收数据包的速率比内核处理这些包的速率快时，允许送到队列的数据包的最大数目。
关于conntrack的详细说明：https://testerhome.com/topics/7509
​
fs.inotify.max_user_instances=524288
# 默认值: 128 指定了每一个real user ID可创建的inotify instatnces的数量上限
​
fs.inotify.max_user_watches=524288
# 默认值: 8192 指定了每个inotify instance相关联的watches的上限
```

end

**ectd**

* 搭建高可用的etcd集群, 集群规模增大时可以自动增加etcd节点；

  目前的解决方案是使用etcd operator来搭建etcd 集群，operator是CoreOS推出的旨在简化复杂有状态应用管理的框架，它是一个感知应用状态的控制器，通过扩展Kubernetes API来自动创建、管理和配置应用实例。

  etcd operator 有如下特性：

* * ceate/destroy: 自动部署和删除 etcd 集群，不需要人额外干预配置。
  * resize：可以动态实现 etcd 集群的扩缩容。
  * backup：支持etcd集群的数据备份和集群恢复重建
  * upgrade：可以实现在升级etcd集群时不中断服务。
* 配置etcd使用ssd固态盘存储；
* 设置自动压缩
* 设置 --quota-backend-bytes 增大etcd的存储限制。默认值是 2G；
* 需要配置单独的 Etcd 集群存储 kube-apiserver 的 event。
* 跨数据中心时，需要调整心跳间隔和选举超时时间 默认的心跳时间是100ms，建议为round trip时间 默认选举超时是1000ms
* 提供etcd的磁盘IO优先级

  由于 ETCD 必须将数据持久保存到磁盘日志文件中，因此来自其他进程的磁盘活动可能会导致增加写入时间，结果导致 ETCD 请求超时和临时 leader 丢失。当给定高磁盘优先级时，ETCD 服务可以稳定地与这些进程一起运行:

  `ionice -c2 -n0 -p $(pgrep etcd)`

* \(重要\)分离 events 存储

  集群规模大的情况下，集群中包含大量节点和服务，会产生大量的 event，这些 event 将会对 etcd 造成巨大压力并占用大量 etcd 存储空间，为了在大规模集群下提高性能，可以将 events 存储在单独的 ETCD 集群中。

  配置apiserver:

  ```text
  --etcd-servers="http://etcd1:2379,http://etcd2:2379,http://etcd3:2379" --etcd-servers-overrides="/events#http://etcd4:2379,http://etcd5:2379,http://etcd6:2379"
  --etcd-servers-overrides stringSlice
  Per-resource etcd servers overrides, comma separated. The individual override format: group/resource#servers, where servers are URLs, semicolon separated.
  ```

  end

* end减小网络延迟\(牛逼\)

  如果有大量并发客户端请求 ETCD leader 服务，则可能由于网络拥塞而延迟处理 follower 对等请求。在 follower 节点上的发送缓冲区错误消息：

  ```text
  dropped MsgProp to 247ae21ff9436b2d since streamMsg's sending buffer is full
  dropped MsgAppResp to 247ae21ff9436b2d since streamMsg's sending buffer is full
  ```

  可以通过在客户端提高 ETCD 对等网络流量优先级来解决这些错误。在 Linux 上，可以使用 tc 对对等流量进行优先级排序：

  ```text
  $ tc qdisc add dev eth0 root handle 1: prio bands 3
  $ tc filter add dev eth0 parent 1: protocol ip prio 1 u32 match ip sport 2380 0xffff flowid 1:1
  $ tc filter add dev eth0 parent 1: protocol ip prio 1 u32 match ip dport 2380 0xffff flowid 1:1
  $ tc filter add dev eth0 parent 1: protocol ip prio 2 u32 match ip sport 2379 0xffff flowid 1:1
  $ tc filter add dev eth0 parent 1: protocol ip prio 2 u32 match ip dport 2379 0xffff flowid 1:1
  ```

  end

* end

0.0

**镜像拉取优化**

1. Docker 配置

* 设置 max-concurrent-downloads=10 配置每个pull操作的最大并行下载数，提高镜像拉取效率，默认值是3。

  --max-concurrent-downloads int Set the max concurrent downloads for each pull \(default 3\)

* 使用 SSD 存储。
* 预加载 pause 镜像，比如 docker image save -o /opt/preloaded\_docker\_images.tar 和docker image load -i /opt/preloaded\_docker\_images.tar 启动pod时都会拉取pause镜像，为了减小拉取pause镜像网络带宽，可以每个node预加载pause镜像。

1. Kubelet配置

* 设置 --serialize-image-pulls=false 该选项配置串行拉取镜像，默认值时true，配置为false可以增加并发度。但是如果docker daemon 版本小于 1.9，且使用 aufs 存储则不能改动该选项。
* 设置 --image-pull-progress-deadline=30 配置镜像拉取超时。默认值时1分，对于大镜像拉取需要适量增大超时时间。
* Kubelet 单节点允许运行的最大 Pod 数：--max-pods=110（默认是 110，可以根据实际需要设置）。

1. 镜像registry p2p分发

**apiserver**

apiserver HA（重要）

* 方式一: 启动多个 kube-apiserver 实例通过外部 LB 做负载均衡。
* 方式二: 设置 `--apiserver-count` 和 `--endpoint-reconciler-type`，可使得多个 kube-apiserver 实例加入到 Kubernetes Service 的 endpoints 中，从而实现高可用。

不过由于 TLS 会复用连接，所以上述两种方式都无法做到真正的负载均衡。为了解决这个问题，可以在服务端实现限流器，在请求达到阀值时告知客户端退避或拒绝连接，客户端则配合实现相应负载切换机制。

> TLS 复用？避免每次连接都要去验证证书：
>
> [https://imququ.com/post/optimize-tls-handshake.html](https://imququ.com/post/optimize-tls-handshake.html)
>
> 0.0

node节点数量 &gt;= 3000， 推荐设置如下配置：

```text
--max-requests-inflight=3000
--max-mutating-requests-inflight=1000
---
--max-requests-inflight int     Default: 400
The maximum number of non-mutating requests in flight at a given time. When the server exceeds this, it rejects requests. Zero for no limit.
--max-mutating-requests-inflight int     Default: 200
The maximum number of mutating requests in flight at a given time. When the server exceeds this, it rejects requests. Zero for no limit.
```

node节点数量在 1000 -- 3000， 推荐设置如下配置：

```text
--max-requests-inflight=1500
--max-mutating-requests-inflight=500
```

内存配置选项和node数量的关系，单位是MB：

```text
--target-ram-mb=node_nums * 60
```

**Pod 配置**

在运行 Pod 的时候也需要注意遵循一些最佳实践，比如:

0.0 这个必须设置资源配置，不然会一直耗光node的资源--

* 为容器设置资源请求和限制，尤其是一些基础插件服务

  ```text
  spec.containers[].resources.limits.cpu
  spec.containers[].resources.limits.memory
  spec.containers[].resources.requests.cpu
  spec.containers[].resources.requests.memory
  spec.containers[].resources.limits.ephemeral-storage
  spec.containers[].resources.requests.ephemeral-storage
  ```

  在k8s中，会根据pod不同的limit 和 requests的配置将pod划分为不同的qos类别： - Guaranteed

  * Every Container in the Pod must have a memory limit and a memory request, and they must be the same.
  * Every Container in the Pod must have a CPU limit and a CPU request, and they must be the same.

  - Burstable

  * The Pod does not meet the criteria for QoS class Guaranteed.
  * At least one Container in the Pod has a memory or CPU request.

  - BestEffort

  * For a Pod to be given a QoS class of BestEffort, the Containers in the Pod must not have any memory or CPU limits or requests.

  当机器可用资源不够时，kubelet会根据qos级别划分迁移驱逐pod。

  被驱逐的优先级：`BestEffort > Burstable > Guaranteed`

* 对关键应用使用 nodeAffinity、podAffinity 和 podAntiAffinity 等保护， 使其调度分散到不同的node上。比如kube-dns 配置：

```text
affinity:
 podAntiAffinity:
   requiredDuringSchedulingIgnoredDuringExecution:
   - weight: 100
     labelSelector:
       matchExpressions:
       - key: k8s-app
         operator: In
         values:
         - kube-dns
     topologyKey: kubernetes.io/hostname
```

* 尽量使用控制器来管理容器（如 Deployment、StatefulSet、DaemonSet、Job 等）

  0.0

**Kube-scheduler 配置**

* 设置 --kube-api-qps=100 默认值是50

scheduler 和 controller-manager的高可用

```text
--leader-elect=true
--leader-elect-lease-duration=15s
# lease > renew 才能保证，及时续上leader--
--leader-elect-renew-deadline=10s
--leader-elect-resource-lock=endpoints
--leader-elect-retry-period=2s
```

end

**Kube-controller-manager 配置**

* 设置 --kube-api-qps=100 默认值是20

  ```text
  --kube-api-qps float32     Default: 50
  DEPRECATED: QPS to use while talking with kubernetes apiserver
  --kube-api-burst int32     Default: 30
  Burst to use while talking with kubernetes apiserver.
  ```

  end

* 设置 --kube-api-burst=100 默认值是30

**conntrack 优化**

（对于已存在的失效链接，可以清空conntrack缓存来释放）

**连接跟踪表**

连接跟踪表存放于系统内存中，可以用`cat /proc/net/nf_conntrack`查看当前跟踪的所有conntrack条目。如下是代表一个tcp连接的conntrack条目，根据连接协议不同，下面显示的字段信息也不一样，比如icmp协议

ipv4 2 tcp 6 431955 ESTABLISHED src=172.16.207.231 dst=172.16.207.232 sport=51071 dport=5672 src=172.16.207.232 dst=172.16.207.231 sport=5672 dport=51071 \[ASSURED\] mark=0 zone=0 use=2

> 蓝色： src=172.16.207.231 dst=172.16.207.232 sport=51071 dport=5672
>
> 红色： src=172.16.207.232 dst=172.16.207.231 sport=5672 dport=51071

每个conntrack条目表示一个连接，连接协议可以是tcp，udp，icmp等，**它包含了数据包的原始方向信息\(蓝色\)和期望的响应包信息\(红色\)，这样内核能够在后续到来的数据包中识别出属于此连接的双向数据包，并更新此连接的状态，**各字段意思的具体分析后面会说。连接跟踪表中能够存放的conntrack条目的最大值，即系统允许的最大连接跟踪数记作`CONNTRACK_MAX`

![netfilter\_hash\_table](https://opengers.github.io/images/openstack/openstack-netfilter/hash_table.jpeg)

> 连接追踪表是Hash Table，Hash Table Entry是一个Bucket，每个Bucket可以包含N个Linked list.

在内核中，连接跟踪表是一个二维数组结构的哈希表\(hash table\)，哈希表的大小记作`HASHSIZE`，哈希表的每一项\(hash table entry\)称作bucket，因此哈希表中有`HASHSIZE`个bucket存在，每个bucket包含一个链表\(linked list\)，每个链表能够存放若干个conntrack条目\(`bucket size`\)。对于一个新收到的数据包，内核使用如下步骤判断其是否属于一个已有连接：

* 内核提取此数据包信息\(源目IP，port，协议号\)进行hash计算得到一个hash值，在哈希表中以此hash值做索引，索引结果为数据包所属的bucket\(链表\)。这一步hash计算时间固定并且很短
* 遍历hash得到的bucket，查找是否有匹配的conntrack条目。这一步是比较耗时的操作，`bucket size`越大，遍历时间越长

**如何设置最大连接跟踪数**

根据上面对哈希表的解释，系统最大允许连接跟踪数`CONNTRACK_MAX` = `连接跟踪表大小(HASHSIZE) * Bucket大小(bucket size)`。从连接跟踪表获取bucket是hash操作时间很短，而遍历bucket相对费时，因此为了conntrack性能考虑，`bucket size`越小越好，默认为8

```text
#查看系统当前最大连接跟踪数CONNTRACK_MAX
sysctl -a | grep net.netfilter.nf_conntrack_max
#net.netfilter.nf_conntrack_max = 3203072
​
#查看当前连接跟踪表大小HASHSIZE
sysctl -a | grep net.netfilter.nf_conntrack_buckets
#400384
#或者这样
cat /sys/module/nf_conntrack/parameters/hashsize
#400384    
```

这两个的比值即为`bucket size` = 3203072 / 400384

**如下，现在需求是设置系统最大连接跟踪数为320w**，由于`bucket size`不能直接设置，为了使`bucket size`值为8，我们需要同时设置`CONNTRACK_MAX`和`HASHSIZE`，因为他们的比值就是`bucket size`

```text
#HASHSIZE (内核会自动格式化为最接近允许值)       
echo 400000 > /sys/module/nf_conntrack/parameters/hashsize
​
#系统最大连接跟踪数
sysctl -w net.netfilter.nf_conntrack_max=3200000      
​
#注意nf_conntrack内核模块需要加载     
```

为了使nf\_conntrack模块重新加载或系统重启后生效

```text
#nf_conntrack模块提供了设置HASHSIZE的参数          
echo "options nf_conntrack hashsize=400000" > /etc/modprobe.d/nf_conntrack.conf                  
```

只需要固化HASHSIZE值，nf\_conntrack模块在重新加载时会自动设置CONNTRACK\_MAX = `hashsize * 8`，当然前提是你`bucket size`使用系统默认值8。如果自定义`bucket size`值，就需要同时固化CONNTRACK\_MAX，以保持其比值为你想要的`bucket size`

上面我们没有改变`bucket size`的默认值8，但是若内存足够并且性能很重要，你可以考虑每个bucket一个conntrack条目\(`bucket size` = 1\)，最大可能降低遍历耗时，即`HASHSIZE = CONNTRACK_MAX`

```text
#HASHSIZE
echo 3200000 > /sys/module/nf_conntrack/parameters/hashsize
#CONNTRACK_MAX
sysctl -w net.netfilter.nf_conntrack_max=3200000
```

end

**如何计算连接跟踪所占内存（6）**

连接跟踪表存储在系统内存中，因此需要考虑内存占用问题，可以用下面公式计算设置不同的最大连接跟踪数所占最大系统内存

```text
total_mem_used(bytes) = CONNTRACK_MAX * sizeof(struct ip_conntrack) + HASHSIZE * sizeof(struct list_head)
```

例如我们需要设置最大连接跟踪数为320w，在centos6/7系统上，`sizeof(struct ip_conntrack)` = 376，`sizeof(struct list_head)` = 16，并且`bucket size`使用默认值8，并且`HASHSIZE = CONNTRACK_MAX / 8`，因此

```text
total_mem_used(bytes) = 3200000 * 376 + (3200000 / 8) * 16
# = 1153MB ~= 1GB
```

因此可以得到，**在centos6/7系统上，设置320w的最大连接跟踪数，所消耗的内存大约为1GB，对现代服务器来说，占用内存并不多，**但conntrack实在让人又爱又恨

关于上面两个`sizeof(struct *)`值在你系统上的大小，如果会C就好说，如果不会，可以使用如下python代码计算

```text
import ctypes
​
#不同系统可能此库名不一样，需要修改             
LIBNETFILTER_CONNTRACK = 'libnetfilter_conntrack.so.3.6.0'
​
nfct = ctypes.CDLL(LIBNETFILTER_CONNTRACK)
print 'sizeof(struct nf_conntrack):', nfct.nfct_maxsize()
print 'sizeof(struct list_head):', ctypes.sizeof(ctypes.c_void_p) * 2
```

end  


