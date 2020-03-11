# 总结整理的参数优化

#### Linux Kernel

一个未经优化过的Linux OS Server不是一台合格的服务器。

**tools to modify Kernel parameters**

设置内核参数的几种方法：

Sysctl is a means of configuring certain aspects of the kernel at run-time, and the /proc/sys/ directory is there so that you don’t even need special tools to do it.

* sysctl -w Key=Value && sysctl -p
* vim /proc/sys/&lt;module\_name&gt;/

**/proc/sys/\***

This file contains the documentation for the sysctl files in /proc/sys/\*

* /proc/sys/abi

  This path is binary emulation relevant aka personality types aka abi.

* /proc/sys/debug
* /proc/sys/dev
* /proc/sys/fs

  The files in this directory can be used to tune and monitor miscellaneous and general things in the operation of the Linux kernel.

* /proc/sys/kernel
* /proc/sys/net

  The interface to the networking parts of the kernel is located in /proc/sys/net.

* /proc/sys/vm

  The files in this directory can be used to tune the operation of the virtual memory \(VM\) subsystem of the Linux kernel and the writeout of dirty data to disk.

**filesystem**

Linux下一切皆文件，比如一个连接要打开一个socket句柄文件（file descriptor）。

Linux中文件一旦被进程打开，那么如果该进程一直不释放该file的fd，那么该文件纵然被删除了也不会真正的释放disk space.通过`lsof`还是可以看到那些被标记delete但是并没有真正释放空间的文件。

> nviolability of private property

You can "free" space in two ways then:

1. as mentioned above - you can kill application, which open file. 即`restart Application`
2. you can... truncate file. Even if it's deleted即 `> filename`

这里，很明显重启应用去达到释放disk space是不可取的，所以以后清理文件时记得使用: `> filename`

* fs.file-max

  系统级限制： 所有用户总共能打开的文件描述符数.

* ulimit

  用户级别限制打开文件数使用：`ulimit`

  > ulimit -n 显示当前用户能打开的最大文件数
  >
  > ulimit -n &lt;filenumber&gt; 修改能打开的最大文件数

  持久化修改：

  ```text
  # vim /etc/security/limits.conf
  ​
  * soft nofile 65535
  ​
  * hard nofile 65535
  ```

* end

**Network**

linux Server作为优秀的服务器，支持大量网络连接，是它的基操。

**TCP Connection Number**

```text
# 对于 LISTEN 状态，Send-Q 表示 accept queue 的最大限制大小，Recv-Q 表示其实际大小。
# 下面的信息表示：全连接溢出了
$ ss -lnt
State      Recv-Q Send-Q Local Address:Port                Peer Address:Port
LISTEN     129    128                *:80                             *:*
​
```

Sync Queue ： max syn queue size = min\(backlog, net.core.somaxconn, net.ipv4.tcp\_max\_syn\_backlog\)

`net.ipv4.tcp_max_syn_backlog` : 直接决定半连接队列值

当半连接队列溢出时，Server 收到了新的发起连接的 SYN：

* 如果不开启 `net.ipv4.tcp_syncookies`：直接丢弃这个 SYN

  当收到 SYN Flood 攻击时，系统会直接丢弃新的 SYN 包，也就是正常客户端将无法和服务器通信。

  0.0 没有判断机制会影响normal client

* 如果开启：`net.ipv4.tcp_syncookies`

  开启后半连接队列即使已经满了，正常用户依旧可以和服务器进行通信

  * 如果全连接队列满了，并且 `qlen_young` 的值大于 1：丢弃这个 SYN
  * 否则，生成 syncookie 并返回 SYN/ACK 包

Accept/Listen Queue：max accept queue size = min\(backlog, net.core.somaxconn\)

`net.core.somaxconn`: 直接决定看全连接队列的值

当系统收到三次握手中第三次的 ACK 包，正常情况下应该是从半连接队列中取出连接，更改状态，放入全连接队列中。此时如果全连接队列满了的话：

* 如果设置了 `net.ipv4.tcp_abort_on_overflow=1`，那么直接回复 RST，同时，对应的连接直接从半连接队列中删除
* 否则，直接忽略 ACK，然后 TCP 的超时机制会起作用，一定时间以后，Server 会重新发送 SYN/ACK，因为在 Server 看来，它没有收到 ACK

**Socket Memory**

高并发场景必须考虑tcp对内存的消耗

这三个参数就是TCP使用内存的大小，单位是页，每个页是4K的大小。

流弊：[https://colobu.com/2014/09/18/linux-tcpip-tuning/](https://colobu.com/2014/09/18/linux-tcpip-tuning/)

```text
$ cat /proc/sys/net/ipv4/tcp_mem  
4096    16384   4194304
# 这三个数值分别代表：Low    Pressure  High
​
## 给一个参考设置
net.ipv4.tcp_wmem = 8192 131072 16777216
net.ipv4.tcp_rmem = 32768 131072 16777216
net.ipv4.tcp_mem = 786432 1048576 1572864
```

net.ipv4.tcp\_wmem

发送缓存设置

net.ipv4.tcp\_rmem

接收缓存设置

net.ipv4.tcp\_mem

tcp连接消耗总内存，如果超过High值，TCP 连接将被拒绝。

**tcp keepalive**

A和B两边通过三次握手建立好TCP连接，然后突然间B就宕机了，之后时间内B再也没有起来。如果B宕机后A和B一直没有数据通信的需求，A就永远都发现不了B已经挂了，那么A的内核里还维护着一份关于A&B之间TCP连接的信息，浪费系统资源。于是在TCP层面引入了keepalive的机制，A会定期给B发空的数据包，通俗讲就是心跳包，一旦发现到B的网络不通就关闭连接。这一点在LVS内尤为明显，因为LVS维护着两边大量的连接状态信息，一旦超时就需要释放连接。

TCP Keep-Alive is managed by OS in the TCP layer.

Linux内核对于tcp keepalive的调整主要有以下三个参数

```text
1. tcp_keepalive_time(过多久不响应开始探测)
​
 the interval between the last data packet sent (simple ACKs are not considered data) and the first keepalive probe; after the connection is marked to need keepalive, this counter is not used any further
​
2. tcp_keepalive_intvl（探测频率）
​
 the interval between subsequential keepalive probes, regardless of what the connection has exchanged in the meantime
​
3. tcp_keepalive_probes（探测失败次数）
​
 the number of unacknowledged probes to send before considering the connection dead and notifying the application layer
​
​
```

Example

```text
$ cat /proc/sys/net/ipv4/tcp_keepalive_time
  7200
$ cat /proc/sys/net/ipv4/tcp_keepalive_intvl
  75
$ cat /proc/sys/net/ipv4/tcp_keepalive_probes
  9
​
当tcp发现有tcp_keepalive_time(7200)秒未收到对端数据后，开始以间隔tcp_keepalive_intvl(75)秒的频率发送的空心跳包，如果连续tcp_keepalive_probes(9)次以上未响应代码对端已经down了，close连接
​
```

在socket编程时候，可以调用setsockopt指定不同的宏来更改上面几个参数

```text
TCP_KEEPCNT: tcp_keepalive_probes
TCP_KEEPIDLE: tcp_keepalive_time
TCP_KEEPINTVL: tcp_keepalive_intvl
​
```

**ip conntrack table \(mark\)**

**nf\_conntrack**为防火墙记录用户连接的状态连接表.

netfilter的哈希表存储在内核空间，这部分内存不能swap,so这个参数不能无脑设置很大，需要根据系统参数，细细磨合。

> 微操，当lvs或者其他负载均衡器，维护的连接状态表没有及时更新（backend server ips change.），尝试清空`> /proc/net/nf_contrack`文件（read only file so can not edit），从而获取到新的变化ip

```text
# 哈希表大小（只读）（64位系统、8G内存默认 65536，16G翻倍，如此类推）
# 跟踪的连接用哈希表存储，每个桶（bucket）里都是1个链表，默认长度为4KB
# 而且，哈希表大小 64位 最大连接数/8 32 最大连接数/4
$ cat /proc/sys/net/netfilter/nf_conntrack_buckets 
16384
​
# 最大跟踪连接数，默认 nf_conntrack_buckets * 4
$ cat /proc/sys/net/netfilter/nf_conntrack_max     
65536
​
# 链接信息都在这里
$ cat /proc/net/nf_contrack
...
...
```

`conntrack`，即connection tracking，这是netfilter提供的连接跟踪机制，此机制允许内核”审查”通过此处的所有网络数据包，并能识别出此数据包属于哪个网络连接\(比如数据包a属于`IP1:8888->IP2:80`这个tcp连接，数据包b属于`ip3:9999->IP4:53`这个udp连接\)。因此，连接跟踪机制使内核能够跟踪并记录通过此处的所有网络连接及其状态。图中可以清楚看到连接跟踪代码所处的网络栈位置，如果不想让某些数据包被跟踪\(`NOTRACK`\),那就要找位于椭圆形方框`conntrack`之前的表和链来设置规则。conntrack机制是iptables实现状态匹配\(`-m state`\)以及NAT的基础，它由单独的内核模块`nf_conntrack`实现。

当加载内核模块`nf_conntrack`后，conntrack机制就开始工作，`conntrack`在内核中有两处位置\(PREROUTING和OUTPUT之前\)能够跟踪数据包。对于每个通过`conntrack`的数据包，内核都为其生成一个conntrack条目用以跟踪此连接，对于后续通过的数据包，内核会判断若此数据包属于一个已有的连接，则更新所对应的conntrack条目的状态\(比如更新为ESTABLISHED状态\)，否则内核会为它新建一个conntrack条目。所有的conntrack条目都存放在一张表里，称为连接跟踪表

那么内核如何判断一个数据包是否属于一个已有的连接呢，我们先来了解下连接跟踪表

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

因此可以得到，在centos6/7系统上，设置320w的最大连接跟踪数，所消耗的内存大约为1GB，对现代服务器来说，占用内存并不多，但conntrack实在让人又爱又恨

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

**conntrack条目（流弊）**

conntrack从经过它的数据包中提取详细的，唯一的信息，因此能保持对每一个连接的跟踪。关于conntrack如何确定一个连接，对于tcp/udp，连接由他们的源目地址，源目端口唯一确定。对于icmp，由type，code和id字段确定。

```text
$ cat /proc/net/nf_contrack
...
ipv4     2 tcp      6 33 SYN_SENT src=172.16.200.119 dst=172.16.202.12 sport=54786 dport=10051 [UNREPLIED] src=172.16.202.12 dst=172.16.200.119 sport=10051 dport=54786 mark=0 zone=0 use=2
```

如上是一条conntrack条目，它代表当前已跟踪到的某个连接，conntrack维护的所有信息都包含在这个条目中，通过它就可以知道某个连接处于什么状态

* 此连接使用ipv4协议，是一条tcp连接\(tcp的协议类型代码是6\)
* `33`是这条conntrack条目在当前时间点的生存时间\(每个conntrack条目都会有生存时间，从设置值开始倒计时，倒计时完后此条目将被清除\)，可以使用`sysctl -a |grep conntrack | grep timeout`查看不同协议不同状态下生存时间设置值，当然这些设置值都可以调整，注意若后续有收到属于此连接的数据包，则此生存时间将被重置\(重新从设置值开始倒计时\)，并且状态改变，生存时间设置值也会响应改为新状态的值
* `SYN_SENT`是到此刻为止conntrack跟踪到的这个连接的状态\(内核角度\)，`SYN_SENT`表示这个连接只在一个方向发送了一初始TCP SYN包，还未看到响应的SYN+ACK包\(只有tcp才会有这个字段\)。
* `src=172.16.200.119 dst=172.16.202.12 sport=54786 dport=10051`是从数据包中提取的此连接的源目地址、源目端口，是conntrack首次看到此数据包时候的信息。
* `[UNREPLIED]`说明此刻为止这个连接还没有收到任何响应，当一个连接已收到响应时，\[UNREPLIED\]标志就会被移除
* 接下来的`src=172.16.202.12 dst=172.16.200.119 sport=10051 dport=54786`地址和端口和前面是相反的，这部分不是数据包中带有的信息，是conntrack填充的信息，代表conntrack希望收到的响应包信息。意思是若后续conntrack跟踪到某个数据包信息与此部分匹配，则此数据包就是此连接的响应数据包。注意这部分确定了conntrack如何判断响应包\(tcp/udp\)，icmp是依据另外几个字段

上面是tcp连接的条目，而udp和icmp没有连接建立和关闭过程，因此条目字段会有所不同，后面iptables状态匹配部分我们会看到处于各个状态的conntrack条目

注意conntrack机制并不能够修改或过滤数据包，它只是跟踪网络连接并维护连接跟踪表，以提供给iptables做状态匹配使用，也就是说，如果你iptables中用不到状态匹配，那就没必要启用conntrack

**利用conntrack entry来排查问题（6）**

利用找出前五位的方法查到当前表中记录的连接最多的主机,然后进一步分析是不是被黑了，或者存在什么性能问题。

```text
$ cat /proc/net/ip_conntrack | cut -d ' ' -f 10 | cut -d '=' -f 2 | sort | uniq -c | sort -nr | head -n 5
IP_1
IP_2
IP_3
IP_4
IP_%
```

**优化**

[http://www.mamicode.com/info-detail-2422830.html?\_\_cf\_chl\_jschl\_tk\_\_=22d6d4d38e12118c59fa6cdd24945d25db33dce8-1582882497-0-AZvRnkdShalzpnzVAUncPo1qh0FskWORbciBsuzXmiiY63CIbxhRsNKauT3T\_aSlXlofHqVwJiIZsm0qwAT3P2DWSA0454ZsGxX\_3ZJo4oOQDGFSqjkKoqbQJnNZurVVkr2W9VXa79aso-O9yV4XOwbdFDJwj\_uIKz\_LKDRH9PTZ2ctiy3-uNAibrvij8P1TdLME42wZnHCKemSbGJX7D7MFpKyI8Ak2Ao28TEJSUblbIVwdEy8klPwrIx\_sSIRzCi-FGo6xLsKzUI0m\_SYqCr7gHVn5PPhjN5nKSTmqb\_QchQNMDDK0eW2\_7E1FTVMwpw](http://www.mamicode.com/info-detail-2422830.html?__cf_chl_jschl_tk__=22d6d4d38e12118c59fa6cdd24945d25db33dce8-1582882497-0-AZvRnkdShalzpnzVAUncPo1qh0FskWORbciBsuzXmiiY63CIbxhRsNKauT3T_aSlXlofHqVwJiIZsm0qwAT3P2DWSA0454ZsGxX_3ZJo4oOQDGFSqjkKoqbQJnNZurVVkr2W9VXa79aso-O9yV4XOwbdFDJwj_uIKz_LKDRH9PTZ2ctiy3-uNAibrvij8P1TdLME42wZnHCKemSbGJX7D7MFpKyI8Ak2Ao28TEJSUblbIVwdEy8klPwrIx_sSIRzCi-FGo6xLsKzUI0m_SYqCr7gHVn5PPhjN5nKSTmqb_QchQNMDDK0eW2_7E1FTVMwpw)

**port range**

```text
#表示用于向外连接的端口范围。缺省情况下很小：32768到61000，改为1024到65000。
net.ipv4.ip_local_port_range = 1024 65000
​
​
#设置系统对最大跟踪的TCP连接数的限制
# emm 这个好像不太建议设置
net.ipv4.netfilter.ip_conntrack_max=204800
​
```

**network security**

```text
# 避免放大攻击
net.ipv4.icmp_echo_ignore_broadcasts = 1
​
# 开启恶意icmp错误消息保护
net.ipv4.icmp_ignore_bogus_error_responses = 1
​
#防止不正确的udp包的攻击 
net.inet.udp.checksum=1
​
#是否接受含有源路由信息的ip包。参数值为布尔值，1表示接受，0表示不接受。
#在充当网关的linux主机上缺省值为1，在一般的linux主机上缺省值为0。
#从安全性角度出发，建议你关闭该功能。 
net.ipv4.conf.default.accept_source_route = 0
​
```

**tcp socket**

```text
#表示如果套接字由本端要求关闭，这个参数决定了它保持在FIN-WAIT-2状态的时间。 
# 默认值是60，可以减少。
net.ipv4.tcp_fin_timeout = 10
​
#表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭；
net.ipv4.tcp_tw_reuse = 1
​
#表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭。
net.ipv4.tcp_tw_recycle = 1
​
​
#开启TCP时间戳
#以一种比重发超时更精确的方法（请参阅 RFC 1323）来启用对 RTT 的计算；为了实现更好的性能应该启用这个选项。
net.ipv4.tcp_timestamps = 1
​
```

**Memory**

Memory 不可压缩资源，Memory 用光就OOM，系统挂掉。

**max\_map\_count \(mark\)**

You may need to increase the max\_map\_count kernel parameter to avoid running out of map areas for the Vector Server process.

To increase the max\_map\_count parameter

Add the following line to /etc/sysctl.conf:

`vm.max_map_count=map_count`

**where** _**map\_count**_ **should be around 1 per 128 KB of system memory. For example**:

`vm.max_map_count=2097152`

on a 256 GB system.

**牛逼配置：百万JVM线程**

[https://blog.csdn.net/Dream\_Flying\_BJ/article/details/67632448](https://blog.csdn.net/Dream_Flying_BJ/article/details/67632448)

0.0 前提是JVM保证有足够的多的内存

如果要达到单个JVM开启100w以上的线程数，需要配置`vm.max_map_count=2048000`或者以上。

因为默认`vm.max_map_count=65530`，因此缺省配置下，单个jvm能开启的最大线程数为其一半，即3w左右，大概32k的量

实际中，可以通过命令【cat /proc//maps \|wc -l】来监控，当前进程使用到的vm映射数量。

**es和VMA**

max\_map\_count文件包含限制一个进程可以拥有的VMA\(虚拟内存区域\)的数量。

VMA是的user processes都觉得自己拥有一段连续且足够大的内存、（RAM 4G但是每个process还是可以拿到4G的VMA-- 便于用户进程管理和使用内存）

es启动时，会报错: max virtual memory areas vm.max\_map\_count \[65530\] is too low, increase to at least \[262144\]

0.0 熟悉内核参数或者Linux 内存机制了就会捕捉到virtual memory areas关键字，从而知道去哪解决这个问题。

**top command**

top显示的关于memory的几个参数：

RES -- Resident Memory Size \(KiB\) The non-swapped physical memory a task is using.

SHR -- Shared Memory Size \(KiB\) The amount of shared memory available to a task, not all of which is typically resident. It simply reflects memory that could be potentially shared with other processes.

VIRT -- Virtual Memory Size \(KiB\) The total amount of virtual memory used by the task. It includes all code, data and shared libraries plus pages that have been swapped out and pages that have been mapped but not used.

> [https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-store.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-store.html)
>
> 0.0

**汇总**

[https://colobu.com/2014/09/18/linux-tcpip-tuning/](https://colobu.com/2014/09/18/linux-tcpip-tuning/)

```text
## net
net.core.netdev_max_backlog = 400000
#该参数决定了，网络设备接收数据包的速率比内核处理这些包的速率快时，允许送到队列的数据包的最大数目。
 
net.core.optmem_max = 10000000
#该参数指定了每个套接字所允许的最大缓冲区的大小
 
net.core.rmem_default = 10000000
#指定了接收套接字缓冲区大小的缺省值（以字节为单位）。
 
net.core.rmem_max = 10000000
#指定了接收套接字缓冲区大小的最大值（以字节为单位）。
 
net.core.somaxconn = 100000
#Linux kernel参数，表示socket监听的backlog(监听队列)上限
 
net.core.wmem_default = 11059200
#定义默认的发送窗口大小；对于更大的 BDP 来说，这个大小也应该更大。
 
net.core.wmem_max = 11059200
#定义发送窗口的最大大小；对于更大的 BDP 来说，这个大小也应该更大。
 
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
#严谨模式 1 (推荐)
#松散模式 0
 
net.ipv4.tcp_congestion_control = bic
#默认推荐设置是 htcp
 
net.ipv4.tcp_window_scaling = 0
#关闭tcp_window_scaling
#启用 RFC 1323 定义的 window scaling；要支持超过 64KB 的窗口，必须启用该值。
 
net.ipv4.tcp_ecn = 0
#把TCP的直接拥塞通告(tcp_ecn)关掉
 
net.ipv4.tcp_sack = 1
#关闭tcp_sack
#启用有选择的应答（Selective Acknowledgment），
#这可以通过有选择地应答乱序接收到的报文来提高性能（这样可以让发送者只发送丢失的报文段）；
#（对于广域网通信来说）这个选项应该启用，但是这会增加对 CPU 的占用。
 
net.ipv4.tcp_max_tw_buckets = 10000
#表示系统同时保持TIME_WAIT套接字的最大数量
 
net.ipv4.tcp_max_syn_backlog = 8192
#表示SYN队列长度，默认1024，改成8192，可以容纳更多等待连接的网络连接数。
 
net.ipv4.tcp_syncookies = 1
#表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭；
 
net.ipv4.tcp_timestamps = 1
#开启TCP时间戳
#以一种比重发超时更精确的方法（请参阅 RFC 1323）来启用对 RTT 的计算；为了实现更好的性能应该启用这个选项。
 
net.ipv4.tcp_tw_reuse = 1
#表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭；
 
net.ipv4.tcp_tw_recycle = 1
#表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭。
 
net.ipv4.tcp_fin_timeout = 10
#表示如果套接字由本端要求关闭，这个参数决定了它保持在FIN-WAIT-2状态的时间。
 
net.ipv4.tcp_keepalive_time = 1800
#表示当keepalive起用的时候，TCP发送keepalive消息的频度。缺省是2小时，改为30分钟。
 
net.ipv4.tcp_keepalive_probes = 3
#如果对方不予应答，探测包的发送次数
 
net.ipv4.tcp_keepalive_intvl = 15
#keepalive探测包的发送间隔
 
net.ipv4.tcp_mem
#确定 TCP 栈应该如何反映内存使用；每个值的单位都是内存页（通常是 4KB）。
#第一个值是内存使用的下限。
#第二个值是内存压力模式开始对缓冲区使用应用压力的上限。
#第三个值是内存上限。在这个层次上可以将报文丢弃，从而减少对内存的使用。对于较大的 BDP 可以增大这些值（但是要记住，其单位是内存页，而不是字节）。
 
net.ipv4.tcp_rmem
#与 tcp_wmem 类似，不过它表示的是为自动调优所使用的接收缓冲区的值。
 
net.ipv4.tcp_wmem = 30000000 30000000 30000000
#为自动调优定义每个 socket 使用的内存。
#第一个值是为 socket 的发送缓冲区分配的最少字节数。
#第二个值是默认值（该值会被 wmem_default 覆盖），缓冲区在系统负载不重的情况下可以增长到这个值。
#第三个值是发送缓冲区空间的最大字节数（该值会被 wmem_max 覆盖）。
 
net.ipv4.ip_local_port_range = 1024 65000
#表示用于向外连接的端口范围。缺省情况下很小：32768到61000，改为1024到65000。
 
net.ipv4.netfilter.ip_conntrack_max=204800
#设置系统对最大跟踪的TCP连接数的限制
 
net.ipv4.tcp_slow_start_after_idle = 0
#关闭tcp的连接传输的慢启动，即先休止一段时间，再初始化拥塞窗口。
 
net.ipv4.route.gc_timeout = 100
#路由缓存刷新频率，当一个路由失败后多长时间跳到另一个路由，默认是300。
 
net.ipv4.tcp_syn_retries = 1
#在内核放弃建立连接之前发送SYN包的数量。
 
net.ipv4.icmp_echo_ignore_broadcasts = 1
# 避免放大攻击
 
net.ipv4.icmp_ignore_bogus_error_responses = 1
# 开启恶意icmp错误消息保护
 
net.inet.udp.checksum=1
#防止不正确的udp包的攻击
 
net.ipv4.conf.default.accept_source_route = 0
#是否接受含有源路由信息的ip包。参数值为布尔值，1表示接受，0表示不接受。
#在充当网关的linux主机上缺省值为1，在一般的linux主机上缺省值为0。
#从安全性角度出发，建议你关闭该功能。
```

参考：

[https://docs.actian.com/vector/5.0/index.html\#page/User/Increase\_max\_map\_count\_Kernel\_Parameter\_\(Linux\).htm](https://docs.actian.com/vector/5.0/index.html#page/User/Increase_max_map_count_Kernel_Parameter_%28Linux%29.htm)

