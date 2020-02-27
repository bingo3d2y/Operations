# Cgroup + tc实现流量限速

#### Cgroup实现网速流控

转自大神：[https://www.cnblogs.com/sammyliu/p/5886833.html](https://www.cnblogs.com/sammyliu/p/5886833.html)

net\_cls 和 tc 一起使用可用于限制进程发出的网络包所使用的网络带宽。当使用 cgroups network controll net\_cls 后，指定进程发出的所有网络包都会被加一个 tag，然后就可以使用其他工具比如 iptables 或者 traffic controller （TC）来根据网络包上的 tag 进行流量控制。关于 TC 的文档，网上很多，这里不再赘述，只是用一个简单的例子来加以说明。

 关于 classid，它的格式是 0xAAAABBBB，其中，AAAA 是十六进制的主ID（major number），BBBB 是十六进制的次ID（minor number）。因此，0X10001 表示 10：1，而 0x00010001 表示 1:1。

 （1）首先在host 的网卡 eth0 上做如下设置：

```text
$ tc qdisc del dev eth0 root         #删除已有的规则
$ tc qdisc add dev eth0 root handle 10: htb default 12              
$ tc class add dev eth0 parent 10: classid 10:1 htb rate 1500kbit ceil 1500kbit burst 10k  #限速
#只处理 ping 参数的网络包
$ tc filter add dev eth0 protocol ip parent 10:0 prio 1 u32 match ip protocol 1 0xff flowid 10:1 
```

其结果是：

* 在网卡 eth0 上创建了一个 HTB root 队列，hangle 10： 表示队列句柄也就是major number 为 10
* 创建一个分类 10：1，限制它的出发网络带宽为 80 kbit （千比特每秒）
* 创建一个分类器，将 eth0 上 IP IMCP 协议 的 major ID 为 10 的 prio 为 1 的网络流量都分类到 10：1 类别

（2）启动容器

容器启动后，其 init 进程在host 上的 PID 就被加入到 tasks 文件中了：

```text
root@devstack:/sys/fs/cgroup/net_cls/docker/ff8d9715b7e11a5a69446ff1e3fde3770078e32a7d8f7c1cb35d51c75768fe33# ps -ef | grep 10047
231072   10047 10013  1 07:08 ?        00:00:00 python app.py
```

设置 net\_cls classid：

```text
$ cat /sys/fs/cgroup/net_cls/docker/ff8d9715b7e11a5a69446ff1e3fde3770078e32a7d8f7c1cb35d51c75768fe33/net_cls.classid
0
​
$ echo 0x100001 > net_cls.classid
```

再在容器启动一个 ping 进程，其 ID 也被加入到 tasks 文件中了。

（3）查看tc 情况： tc -s -d class show dev eth0

```text
$ tc -s -d class show dev eth0
Every 2.0s: tc -s class ls dev eth0 Wed Sep 21 04:07:56 2016
​
class htb 10:1 root prio 0 rate 1500Kbit ceil 1500Kbit burst 10Kb cburst 1599b
Sent 17836 bytes 182 pkt (dropped 0, overlimits 0 requeues 0)
rate 0bit 0pps backlog 0b 0p requeues 0
lended: 182 borrowed: 0 giants: 0
tokens: 845161 ctokens: 125161
​
```

我们可以看到 tc 已经在处理 ping 进程产生的数据包了。再来看一下 net\_cls 和 ts 合作的限速效果：

```text
10488 bytes from 192.168.1.1: icmp_seq=35 ttl=63 time=12.7 ms
10488 bytes from 192.168.1.1: icmp_seq=36 ttl=63 time=15.2 ms
10488 bytes from 192.168.1.1: icmp_seq=37 ttl=63 time=4805 ms
10488 bytes from 192.168.1.1: icmp_seq=38 ttl=63 time=9543 ms
```

其中：

* 后两条说使用的 tc class 规则是 tc class add dev eth0 parent 10: classid 10:1 htb rate 1500kbit ceil 15kbit burst 10k
* 前两条所使用的 tc class 规则是 tc class add dev eth0 parent 10: classid 10:1 htb rate 1500kbit ceil 10Mbit burst 10k

