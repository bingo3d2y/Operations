# Nginx HTTP Status Code

**Nginx HTTP Status Code**

在 http status 的 定义中：

* 502 **Bad Gateway**: The server was acting as a gateway or proxy and received an invalid response from the upstream server.

  0.0 无效或者错误的响应

* 504 **Gateway Timeout** ：The server was acting as a gateway or proxy and did not receive a timely response from the upstream server.

  0.0 不能及时收到响应\(timely response\)

Shopee 面试问道这两个区别了。

[https://juejin.im/post/5b54635ae51d451951133d85](https://juejin.im/post/5b54635ae51d451951133d85)

这个人的文章真的流弊

502 的错误原因是 Bad Gateway，一般是由于上游服务（总感觉upstream翻译这样怪怪的）的故障引起的；而 504 则是 nginx 访问上游服务超时，二者完全是两个意思。但在某些情况下，上游服务的超时（触发 tcp reset）也可能引发 502。tcp 的 reset即RST 算是异常/错误响应。

总结：

504： 是超时response，这时TCP连接是建立和正常的

```text
<?php
sleep(70);
echo 'hello world';
```

0.0 如上，由于代码逻辑问题，或者事务操作时间太长。

502：是upstream出来问题，比如upstream 进程挂掉了，导致tcp连接异常。

* server进程不断挂掉和重启，这时是502， nginx提示：recv\(\) failed \(104: Connection reset by peer\) **wh**
* Server进程挂掉，但没有重启，这时也是502，nginx提示： connect\(\) failed \(113: Host is unreachable\)

  0.0 没端口监听就是不可达了。。

0.0 504 到 502 的转变，TCP is establish state，进程挂掉了，没有再起来就tcp keepalive介绍后记2MSL后，无端口监听，就会发RST指令去响应Client 的 SYN，这时也是502.

