# kernel



**内核参数优化**

Memory 不可压缩资源，Memory 用光就OOM，系统挂掉。

**memory**

* **max\_map\_count**

  “This file contains the maximum number of memory map areas a process may have. Memory map areas are used as a side-effect of calling malloc, directly by mmap and mprotect, and also when loading shared libraries.

  While most applications need less than a thousand maps, certain programs, particularly malloc debuggers, may consume lots of them, e.g., up to one or two maps per allocation.

  The default value is 65536.”

  这个参数很关键啊，单个进程能使用的最大内存。这个默认65546太小了，就算JVM设置了`Xmx=4096m`也是不行的-- 因为操作系统层面已经给你做了限制

  设置修改

  ```text
  sysctl -w vm.max_map_count=262144
  echo "vm.max_map_count=262144" > /etc/sysctl.conf
  sysctl -p
  ```

  end

* end

**tcp numbers**

**server最大tcp连接数**

server通常固定在某个本地端口上监听，等待client的连接请求。不考虑地址重用（unix的SO\_REUSEADDR选项）的情况下，即使server端有多个ip，本地监听端口也是独占的，因此server端tcp连接4元组中只有remote ip（也就是client ip）和remote port（客户端port）是可变的，因此最大tcp连接为客户端ip数×客户端port数，对IPV4，不考虑ip地址分类等因素，最大tcp连接数约为2的32次方（ip数）×2的16次方（port数），也就是server端单机最大tcp连接数约为2的48次方。

but，每个socket 消耗内存吖，内存耗光了当然放不下2的48次方的链接。

影响一个socket占用内存的参数包括：

rmem\_max

wmem\_max

tcp\_rmem

tcp\_wmem

tcp\_mem

grep skbuff /proc/slabinfo

对server端，通过增加内存、修改最大文件描述符个数等参数，单机最大并发TCP连接数超过10万 是没问题的，国外 Urban Airship 公司在产品环境中已做到 50 万并发 。在实际应用中，对大规模网络应用，还需要考虑C10K 问题

* 用户级别文件数限制

  ```text
  $ ulimit -n
  1024
  ​
  ## 针对mysql用户的修改
  1. $ vi /etc/security/limits.conf
  mysql soft nofile 10240
  mysql hard nofile 10240
  其中mysql指定了要修改哪个用户的打开文件数限制。
  2. vi /etc/pam.d/login
  0.0 pam 一般不开的吧
  ```

  end

* 系统级别文件数限制

  ```text
  $ sysctl -a|grep file-max
  fs.file-max = 65535
  ​
  ## 修改
  $ vi /etc/sysctl.conf
  fs.file-max = 399530
  $ sysctl -p
  ​
  ​
  ```

  end

* 网络连接数

  ```text
  $ sysctl -a | grep ipv4.ip_conntrack_max
  net.ipv4.ip_conntrack_max = 20000
  这表明系统将对最大跟踪的TCP连接数限制默认为20000。
  ```

  end

* 端口数

  ```text
  # sysctl -a | grep ipv4.ip_local_port_range
  net.ipv4.ip_local_port_range = 1024 30000
  ​
  每个TCP客户端连接都要占用一个唯一的本地端口号(此端口号在系统的本地端口号范围限制中)，如果现有的TCP客户端连接已将所有的本地端口号占满。将不能创建新的TCP连接。
  ​
  ```

  end

* end

**SYN FLood**

 `net.ipv4.tcp_syncookies = 1`

从tcp握手包来看：

1. Client: 发送 SYN，连接状态进入 `SYN_SENT`
2. Server: 收到 SYN, 创建连接状态为 `SYN_RCVD/SYN_RECV` 的 Socket，响应 SYN/ACK
3. Client: 收到 SYN/ACK，连接状态从 `SYN_SENT` 变为 `ESTABLISHED`，响应 ACK
4. Server: 收到 ACK，连接状态变为 `ESTABLISHED`

Server 需要两个队列，分别存储 `SYN_RCVD` 状态的连接和 `ESTABLISHED` 状态的连接，这就是半连接队列和全连接队列。

SYN Flood 的思路很简单，发送大量的 SYN 数据包给 Server，然后不回应 Server 响应的 SYN/ACK，这样，Server 端就会存在大量处于 `SYN_RECV` 状态的连接，这样的连接会持续填充半连接队列，最终导致半连接队列溢出。

当半连接队列溢出时，Server 收到了新的发起连接的 SYN：

* 如果不开启 `net.ipv4.tcp_syncookies`：直接丢弃这个 SYN

  当收到 SYN Flood 攻击时，系统会直接丢弃新的 SYN 包，也就是正常客户端将无法和服务器通信。

  0.0 没有判断机制会影响normal client

* 如果开启：`net.ipv4.tcp_syncookies`

  开启后半连接队列即使已经满了，正常用户依旧可以和服务器进行通信

  * 如果全连接队列满了，并且 `qlen_young` 的值大于 1：丢弃这个 SYN
  * 否则，生成 syncookie 并返回 SYN/ACK 包

`qlen_young` 表示目前半连接队列中，没有进行 SYN/ACK 包重传的连接数量。

**establish队列溢出**

如果全连接队满了又会怎么样？应用程序调用 `accept` 的速度太慢，或者请求太多，都会导致这个情况。

当系统收到三次握手中第三次的 ACK 包，正常情况下应该是从半连接队列中取出连接，更改状态，放入全连接队列中。此时如果全连接队列满了的话：

* 如果设置了 `net.ipv4.tcp_abort_on_overflow`，那么直接回复 RST，同时，对应的连接直接从半连接队列中删除
* 否则，直接忽略 ACK，然后 TCP 的超时机制会起作用，一定时间以后，Server 会重新发送 SYN/ACK，因为在 Server 看来，它没有收到 ACK

