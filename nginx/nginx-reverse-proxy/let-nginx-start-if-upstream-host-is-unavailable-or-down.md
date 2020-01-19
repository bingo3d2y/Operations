---
description: Let nginx start if upstream host is unavailable or down
---

# Let nginx start if upstream host is unavailable or down

**Let nginx start if upstream host is unavailable or down**

[https://sandro-keil.de/blog/let-nginx-start-if-upstream-host-is-unavailable-or-down/](https://sandro-keil.de/blog/let-nginx-start-if-upstream-host-is-unavailable-or-down/)

The trick is to use variables for the upstream domain name.

0.0 `fastcgi_pass`后面接变量啊话，就不会检测upstream host是不是 unreachable了。

The next example replaces the `fastcgi_pass` with a variable, so nginx will not check if the host is reachable on startup. This results in a `502 Bad Gateway` message if the host is unavailable and that's fine. As soon as the service is back, everything works as expected.

这个很关键啊，如果Nginx作为整体的网关了，有一个upstream出来问题就整个nginx启动不了，这太严重了。

```text
server {
    location ^~ /api/ {
        # other config entries omitted for breavity
    
        set $upstream api.awesome.com:9000;
​
        # nginx will now start if host is not reachable
        fastcgi_pass    $upstream; 
        fastcgi_index   index.php;
    }
}
```

end

... 真的厉害，

If you use `proxy_pass` or `fastcgi_pass` definitions in your nginx server config, then nginx checks the hostname during the startup phase. If one of these servers is not available, nginx will not start. This is not useful. If you use nginx as a gateway, why should all services unreachable if only one service is down due the nginx ramp time? This blog post shows a trick how to avoid such behaviour and exposes also the internal Docker DNS IP for the Docker DNS resolver.

