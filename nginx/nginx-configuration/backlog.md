# backlog（转载）

  
**nginx 的 backlog**  
给 nginx pod 的 `somaxconn` 调高到 8096 后观察:

```text
$ ss -lnt
State      Recv-Q Send-Q Local Address:Port                Peer Address:Port
LISTEN     512    511                *:80                             *:*
```

WTF? 还是溢出了，而且调高了 `somaxconn` 之后虽然 accept queue 的最大大小 \(`Send-Q`\) 变大了，但跟 8096 还差很远呀！  
  
在经过一番研究，发现 nginx 在 `listen()` 时并没有读取 `somaxconn` 作为 backlog 默认值传入，它有自己的默认值，也支持在配置里改。通过`ngx_http_core_module` 的官方文档我们可以看到它在 linux 下的默认值就是 511:

```text
backlog=number   sets the backlog parameter in the listen() call that limits the maximum length for the queue of pending connections.
 By default, backlog is set to -1 on FreeBSD, DragonFly BSD, and macOS, and to 511 on other platforms.
```

配置示例:

```text
listen  80  default  backlog=1024;
```

所以，在容器中使用 nginx 来支撑高并发的业务时，记得要同时调整下`net.core.somaxconn`内核参数和 `nginx.conf` 中的 backlog 配置。

转自： roc，腾讯云容器服务\(TKE\)团队

