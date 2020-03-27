# 高并发容器应用优化



**no route to host**

[https://mp.weixin.qq.com/s?\_\_biz=MzU5Mzc0NDUyNg==&mid=2247483806&idx=1&sn=7450d83002e4220354dda7dcfab033ab&chksm=fe0a867fc97d0f692dafa94b6e0c7323a9d72a8eec4cca000d85dda84d73148dab930d900d8f&scene=21\#wechat\_redirect](https://mp.weixin.qq.com/s?__biz=MzU5Mzc0NDUyNg==&mid=2247483806&idx=1&sn=7450d83002e4220354dda7dcfab033ab&chksm=fe0a867fc97d0f692dafa94b6e0c7323a9d72a8eec4cca000d85dda84d73148dab930d900d8f&scene=21#wechat_redirect)

这个问题通常发生的场景就是类似于我们测试环境这种：ServiceA 对外提供服务，当外部发起请求，ServiceA 会通过 RPC 或 HTTP 调用 ServiceB，如果外部请求量变大，ServiceA 调用 ServiceB 的量也会跟着变大，大到一定程度，ServiceA 所在节点源端口不够用，复用`TIME_WAIT`状态连接的源端口，导致五元组跟 IPVS 里连接转发表中的`TIME_WAIT`连接相同，IPVS 就认为这是一个存量连接的报文，就不判断权重直接转发给之前的 rs，导致转发到已销毁的 Pod，从而发生 “No route to host”。

如何规避？集群规模小可以使用 iptables 模式，**如果需要使用 ipvs 模式，可以增加 ServiceA 的副本，并且配置反亲和性 \(podAntiAffinity\)，让 ServiceA 的 Pod 部署到不同节点，分摊流量，避免流量集中到某一个节点，导致调用 ServiceB 时源端口复用。**

**tcp accept queue && syns queue**

`int listen(int sockfd, int backlog)` Linux的内核调用，backlog默认是128。listen 的 backlog 参数同时指定了 socket 的 syn queue 与 accept queue 大小

结合这里的解释提炼一下：

* listen 的 backlog 参数同时指定了 socket 的 syn queue 与 accept queue 大小。
* accept queue 最大不能超过`net.core.somaxconn`的值，即:

  ```text
  max accept queue size = min(backlog, net.core.somaxconn)
  ```

* 如果启用了 syncookies \(`net.ipv4.tcp_syncookies=1`\)，当 syn queue 满了，server 还是可以继续接收 SYN 包并回复 SYN+ACK 给 client，只是不会存入 syn queue 了。因为会利用一套巧妙的 syncookies 算法机制生成隐藏信息写入响应的 SYN+ACK 包中，等 client 回 ACK 时，server 再利用 syncookies 算法校验报文，校验通过后三次握手就顺利完成了。**所以如果启用了 syncookies，syn queue 的逻辑大小是没有限制的**，
* syncookies 通常都是启用了的，所以一般不用担心 syn queue 满了导致丢包。syncookies 是为了防止 SYN Flood 攻击 \(一种常见的 DDoS 方式\)，攻击原理就是 client 不断发 SYN 包但不回最后的 ACK，填满 server 的 syn queue 从而无法建立新连接，导致 server 拒绝服务。
* 如果 syncookies 没有启用，syn queue 的大小就有限制，除了跟 accept queue 一样受`net.core.somaxconn`大小限制之外，还会受到`net.ipv4.tcp_max_syn_backlog`的限制，即:

  ```text
  max syn queue size = min(backlog, net.core.somaxconn, net.ipv4.tcp_max_syn_backlog)
  ```

0.0 真的秀

**preStop 和 readinessProbe**

0.0 这个在应用上必须配置了，这样可以避免在滚动升级过程中出现连接中断的问题。

总结： 还是TCP+k8s rollout机制，tcp RST触发往往是由于端口\(网卡\)在但是进程不在或者进程在用户态直接被内核态rst了。

核心： 旧的pod应用停止但是网卡veth没有被回收，kube-proxy规则也没更新，导致请求依旧分发到旧pod导致，应增加`preStop`;同理，新pod也要加健康检测，避免网卡建好了，应用没起来也会报错。

先看下滚动更新导致连接异常有哪些常见的报错:

* `Connection reset by peer`: 连接被重置。通常是连接建立过，但 server 端发现 client 发的包不对劲就返回 RST，应用层就报错连接被重置。比如在 server 滚动更新过程中，client 给 server 发的请求还没完全结束，或者本身是一个类似 grpc 的多路复用长连接，当 server 对应的旧 Pod 删除\(没有做优雅结束，停止时没有关闭连接\)，新 Pod 很快创建启动并且刚好有跟之前旧 Pod 一样的 IP，这时 kube-proxy 也没感知到这个 IP 其实已经被删除然后又被重建了，针对这个 IP 的规则就不会更新，旧的连接依然发往这个 IP，但旧 Pod 已经不在了，后面继续发包时依然转发给这个 Pod IP，最终会被转发到这个有相同 IP 的新 Pod 上，而新 Pod 收到此包时检查报文发现不对劲，就返回 RST 给 client 告知将连接重置。针对这种情况，建议应用自身处理好优雅结束：Pod 进入 Terminating 状态后会发送 SIGTERM 信号给业务进程，业务进程的代码需处理这个信号，在进程退出前关闭所有连接。
* `Connection refused`: 连接被拒绝。通常是连接还没建立，client 正在发 SYN 包请求建立连接，但到了 server 之后发现端口没监听，内核就返回 RST 包，然后应用层就报错连接被拒绝。比如在 server 滚动更新过程中，旧的 Pod 中的进程很快就停止了\(网卡还未完全销毁\)，但 client 所在节点的 iptables/ipvs 规则还没更新，包就可能会被转发到了这个停止的 Pod \(由于 k8s 的 controller 模式，从 Pod 删除到 service 的 endpoint 更新，再到 kube-proxy watch 到更新并更新 节点上的 iptables/ipvs 规则，这个过程是异步的，中间存在一点时间差，所以有可能存在 Pod 中的进程已经没有监听，但 iptables/ipvs 规则还没更新的情况\)。针对这种情况，建议给容器加一个 preStop，在真正销毁 Pod 之前等待一段时间，留时间给 kube-proxy 更新转发规则，更新完之后就不会再有新连接往这个旧 Pod 转发了，preStop 示例:

  ```text
  lifecycle:
    preStop:
      exec:
        command:
        - /bin/bash
        - -c
        - sleep 30
  ```

  end

  另外，还可能是新的 Pod 启动比较慢，虽然状态已经 Ready，但实际上可能端口还没监听，新的请求被转发到这个还没完全启动的 Pod 就会报错连接被拒绝。针对这种情况，建议给容器加就绪检查 \(readinessProbe\)，让容器真正启动完之后才将其状态置为 Ready，然后 kube-proxy 才会更新转发规则，这样就能保证新的请求只被转发到完全启动的 Pod，readinessProbe 示例:

* `Connection timed out`: 连接超时。通常是连接还没建立，client 发 SYN 请求建立连接一直等到超时时间都没有收到 ACK，然后就报错连接超时。这个可能场景跟前面 Connection refused 可能的场景类似，不同点在于端口有监听，但进程无法正常响应了: 转发规则还没更新，旧 Pod 的进程正在停止过程中，虽然端口有监听，但已经不响应了；或者转发规则更新了，新 Pod 端口也监听了，但还没有真正就绪，还没有能力处理新请求。针对这些情况的建议跟前面一样：加 preStop 和 readinessProbe。

  ```text
  readinessProbe:
    httpGet:
      path: /healthz
      port: 80
      httpHeaders:
      - name: X-Custom-Header
        value: Awesome
    initialDelaySeconds: 15
    timeoutSeconds: 1
  ```

  end

