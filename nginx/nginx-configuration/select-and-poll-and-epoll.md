---
description: select and poll and epoll
---

# 生产级别的Nginx配置

Nginx作为优秀的7层代理，结合lua脚本可以做很多操作，限流，处理自定义header等等。

此外，web开发者操作的大都是http协议（加些奇怪的协议头了），Nginx+lua正好。

Nginx的优点

Nginx有五大优点：模块化、事件驱动\(epoll\)、异步\(AIO\)、非阻塞、多进程单线程。**由内核和模块组成的，**其中内核完成的工作比较简单，仅仅通过查找配置文件将客户端请求映射到一个location block，然后又将这个location block中所配置的每个指令将会启动不同的模块去完成相应的工作。

> nginx是采用多进程模型，master和worker之间主要通过pipe管道的方式进行通信，多进程的优势就在于各个进程互不影响。

**thread\_pools**

nginx使用的是多进程单线程的模型，那么这个线程池参数是干嘛的呢？？？

1. Nginx 在启动后，会有一个 master 进程和多个相互独立的 worker 进程。
2. Master接收来自外界的信号，向各worker进程发送信号，每个进程都有可能来处理这个连接。

   Worker怎么处理连接啊，当然是启动线程了，线程在哪个定义的就是这个Thread\_Pools啊

3. master 进程能监控 worker 进程的运行状态，当 worker 进程退出后\(异常情况下\)，会自动启动新的 worker 进程。

nginx代码中提供了一个thread\_pool（线程池）的核心模块来处理多任务的.线程池主要用于读取、发送文件等IO操作，避免慢速IO影响worker的正常运行。

流弊，这个thread\_pools是用来响应各类事件（IO之类的）的。

thread\_pool是有名字的，上面的线程数目以及队列大小都是指每个worker进程中的线程，而不是所有worker中线程的总数。一个线程池中所有的线程共享一个队列，队列中的最大人数数量为上面定义的max\_queue，如果队列满了的话，再往队列中添加任务就会报错。

In the event that all threads in the pool are busy, a new task will wait in the queue.

**Directives**

官方上的指令介绍都是三行即语法、默认值和Context。

着重说下Context，这个标识了该指令可以作用的范围。

```text
Syntax: error_log file [level];
Default:    error_log logs/error.log error;
Context:    main, http, mail, stream, server, location
​
$ cat nginx.conf
error_log /err_main.logs
http {
    error_log /err_http.logs
    ...
    server {
        listen 80;
        error_log /err_server_80.logs
        ...
    }
    
    upstream app_backend {
        ...
        error_log /err_stream.logs
    }
}
---
Syntax: debug_connection address | CIDR | unix:;
Default:    —
Context:    events
​
## example
events {
    debug_connection 127.0.0.1;
    debug_connection localhost;
    debug_connection 192.0.2.0/24;
    debug_connection ::1;
    debug_connection 2001:0db8::/32;
    debug_connection unix:;
    ...
}
```

**模块分类**

从官网来看，nginx的模块分为：核心模块、http模块（七层）、stream模块（四层）、mail模块和google\_perftools模块。

**Core module**

Nginx的核心模块主要负责建立nginx服务模型、管理网络层和应用层协议、以及启动针对特定应用的一系列候选模块,从下面这些指令也可以看出核心模块的主要功能。

这些directive的作用域都是`main`即直接写入`*.conf`的。

* accept\_mutex: `default accept_mutex off`;

  即默认是有新连接建立，所以worker唤醒但只要一个处理请求epoll惊群效应。高并发场景应该设置成`off`.

  Prior to version 1.11.3, the default value was `on`.

  假设你养了一百只小鸡，现在你有一粒粮食，那么有两种喂食方法：

  * 你把这粒粮食直接扔到小鸡中间，一百只小鸡一起上来抢，最终只有一只小鸡能得手，其它九十九只小鸡只能铩羽而归。这就相当于关闭了accept\_mutex。
  * 你主动抓一只小鸡过来，把这粒粮食塞到它嘴里，其它九十九只小鸡对此浑然不知，该睡觉睡觉。这就相当于激活了accept\_mutex。

  可以看到此场景下，激活accept\_mutex相对更好一些，让我们修改一下问题的场景，我不再只有一粒粮食，而是一盆粮食，怎么办？

  此时如果仍然采用主动抓小鸡过来塞粮食的做法就太低效了，一盆粮食不知何年何月才能喂完，大家可以设想一下几十只小鸡排队等着喂食时那种翘首以盼的情景。此时更好的方法是把这盆粮食直接撒到小鸡中间，让它们自己去抢，虽然这可能会造成一定程度的混乱，但是整体的效率无疑大大增强了。

* accept\_mutex\_delay: 单位ms

  If [accept\_mutex](http://nginx.org/en/docs/ngx_core_module.html#accept_mutex) is enabled, specifies the maximum time during which a worker process will try to restart accepting new connections if another worker process is currently accepting new connections.

  这个可以忽略，因为默认accept\_mutex的值，我们都会设置成`off`

* daemon : Determines whether nginx should become a daemon

  `nginx daemon off;` 这个在容器中见多了，这里就不说了。

* debug\_connection: Enables debugging log for selected client connections.
* debug\_points: This directive is used for debugging

  这两个`debug_*` 指令不常用，跳过。

* env: By default, nginx removes all environment variables inherited from its parent process except the TZ variable. This directive allows preserving some of the inherited variables, changing their values, or creating new environment variables.

  这个指令也不怎么用。

* events： 这个指令比较关键，决定nginx处理事件的模型，通常使用高效事件模型epoll.
* include: Includes another `file`, or files matching the specified `mask`, into configuration.

  通常在Nginx的`nginx.conf`配置Core module指令，用它引入其他虚拟主机的httpd module指令配置文件。

  `include /etc/nginx/conf.d/*.conf`

* load\_module: Loads a dynamic module. 有用
* lock\_file: 实现acept\_mutex的锁文件。这个指令通常忽视它

  On most systems the locks are implemented using atomic operations, and this directive is ignored.

* master\_process: 默认值是on，这个倾向于开发者使用，请忽略。

  Determines whether worker processes are started. This directive is intended for nginx developers.

* multi\_accept： `multi_accept off`

  If `multi_accept` is disabled, a worker process will accept one new connection at a time. Otherwise, a worker process will accept all new connections at a time.

  因为`accept_mutex`设置成off，所以为了保障惊群效应，这里`multi_accept`也要设置成off,即默认值不变。

  multi\_accept 告诉nginx收到一个新连接通知后接受尽可能多的连接，默认是on，设置为on后，多个worker按串行方式来处理连接，也就是一个连接只有一个worker被唤醒，其他的处于休眠状态，设置为off后，多个worker按并行方式来处理连接，也就是一个连接会唤醒所有的worker，直到连接分配完毕，没有取得连接的继续休眠。当你的服务器连接数不多时，开启这个参数会让负载有一定的降低，但是当服务器的吞吐量很大时，为了效率，可以关闭这个参数。

* pcre\_jit 这个看不懂，跳过
* pid ： 指定pid file路径。
* ssl\_engine：Defines the name of the hardware SSL accelerato.没有默认值，还得结合硬件，跳过。
* thread\_pool: thread\_pool default threads=32 max\_queue=65536;

  In the event that all threads in the pool are busy, a new task will wait in the queue. max\_queue设置成0就是0.

  这个线程池和JVM的有点像，处理不过来的event就丢到wait\_queue，要是wait\_queue满了就报错，拒绝事件。

  比如那种cpu密集型的服务，有时候会出现并发请求但是cpu跑不满的情况 这个时候其实不是磁盘io的瓶颈了，瓶颈在cpu但是因为队列等待导致cpu没有均匀分配，这时候aio thread就起作用了，可以把cpu跑满 吞吐增加不少。

* timer\_resolution： 跳过
* use： 使用哪个事件驱动，linux高效的事件驱动就选择epoll.
* user: Defines `user` and `group` credentials used by worker processes.
* worker\_aio\_requests: 注意这个Context是在event下面，开启异步io\(aio\)的情况下，单个工作进程的异步io操作数量上限.默认值是32。 这个值可以设置的大一些，这样可以充分压榨CPU的性能。
* worker\_connections: 默认是512 太小了，这个可以直接设置成65535,反正线程池max\_queue是65536。

  Sets the maximum number of simultaneous connections that can be opened by a worker process.

  Nginx作为反向代理服务器，计算公式最大连接数 = worker\_processes \* worker\_connections / 4

* worker\_cpu\_affinity： 设置worker进程，cpu亲和，emm 一般不指定就用`auto`就行了。
* worker\_priority: 设置worker进程的优先级，数值越小优先级越高。

  一般建议将nginx的优先级设置得更低一些，这样才能保证其执行的权限，但是建议不要设置得比内核的进程优先级（其值为-5）还要低。

  Allowed range normally varies from -20 to 20.

* worker\_processes： Defines the number of worker processes.

  通常设置为cpu核数，或者设置为`auto`自动探测cpu个数。避免worker进程资源竞争。

* worker\_rlimit\_core: 限制coredump核心转储文件的大小

  在Linux操作系统中，如果一个进程由于错误或者收到信号而终止时，会将进程执行时的内存内容写入一个文件（core文件），以作为调试之用，这就是所谓的核心转储。在nginx进程宕机时，其就会产生核心转储文件，而且该文件一般都有几个G，因而如果不限制该文件的大小，那么很有可能会把服务器磁盘占满。该参数的作用就是限制核心转储文件的大小的。

* worker\_rlimit\_nofile: 设置nginx最大能打开的文件描述符数量。受OS的最大文件数限制。
* worker\_shutdown\_timeout: 如果一个 worker 在接收到退出的指令后经过 `worker_shutdown_timeout` 时长后还不能退出，就会被强制退出。
* working\_directory

Core module 部分配置总结：

```text
pid /tmp/nginx.pid;
​
load_module /etc/nginx/modules/ngx_http_modsecurity_module.so;
​
daemon off;
​
worker_processes 144;
​
## 建议设置成65535因为epoll的惊群效应可能导致event分布到worker并不均匀。
worker_rlimit_nofile 65535;
​
worker_shutdown_timeout 10s ;
​
events {
    multi_accept        on;
    worker_connections  16384;
    use                 epoll;
}
```

**http module**

这里是Nginx的主战场了http module使得Nginx成为一个优秀的7层反向代理服务器

nginx和lua脚本结合的很密切，来做一些安全上的限制。

```text
http {
​
​
    # If aio is enabled, specifies whether it is used for writing files. Currently, this only works when using aio threads and is limited to writing temporary files with data received from proxied servers.
    # aio组合拳，使用异步IO来处理从proxied server的写事件
    aio                 threads;
    aio_write           on;
    
    #这两个参数只有在sendfile指令设置成on的时候才有效
    #优化tcp 连接建立但是好像用处不大--
    #tcp_nopush          on;
    #tcp_nodelay         on;
    
    log_subrequest      on;
    
    # ???
    reset_timedout_connection on;
    
    ## http 长连接的参数
    ## Nginx 的默认值是 75 秒，有些浏览器最多只保持 60 秒，所以建议统一设置为 60。
    keepalive_timeout  60s;
    keepalive_requests 100;
    
    ## 各类temporary文件的存放目录
    client_body_temp_path           /tmp/client-body;
    fastcgi_temp_path               /tmp/fastcgi-temp;
    # Defines a directory for storing temporary files with data received from proxied servers
    proxy_temp_path                 /tmp/proxy-temp;
    ajp_temp_path                   /tmp/ajp-temp;
    
    ## 如果buffer太小，Nginx会不停的写一些临时文件，这样会导致磁盘不停的去读写
    # 用于设置客户端请求的Header头缓冲区大小，大部分情况1KB大小足够
    client_header_buffer_size       1k;
    # 允许客户端请求的最大单个文件字节数
    client_body_buffer_size         8k;
    # 设置客户端能够上传的文件大小，默认为1m Context: http, server, location
    client_max_body_size            1m;
    large_client_header_buffers     4 8k;
​
​
    ## timeouts
    # client_header_timeout和client_body_timeout设置请求头和请求体(各自)的超时时间，如果没有发送请求头和请求体，Nginx服务器会返回408错误或者request time out。
    client_header_timeout           60s;
    client_body_timeout             60s;
    
    ## 开启gzip压缩 Gzip Compression
    ## 代码（JS、CSS 和 HTML）会做压缩，图片也会做压缩（PNGOUT、Pngcrush、JpegOptim、Gifsicle 等）。对于文本文件，在服务端发送响应之前进行 GZip 压缩也很重要，通常压缩后的文本大小会减小到原来的 1/4 - 1/3。
    gzip on;
    # 要注意gzip_comp_level的设置，太高的话，Nginx服务会浪费CPU的执行周期。压缩消耗CPU
    gzip_comp_level 5;
    gzip_http_version 1.1;
    gzip_min_length 256;
    # gzip_disable 指令接受一个正则表达式，当请求头中的 UserAgent 字段满足这个正则时，响应不会启用 GZip，这是为了解决在某些浏览器启用 GZip 带来的问题。特别地，指令值 msie6 等价于 MSIE [4-6]\.，但性能更好一些
    gzip_disable       "msie6";
    
    gzip_types application/atom+xml application/javascript application/x-javascript application/json application/rss+xml application/vnd.ms-fontobject application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml application/xml font/opentype image/svg+xml image/x-icon text/css text/plain text/x-component;
    gzip_proxied any;
    gzip_vary on;
    
    ## Context: http
    #如果请求发送到Origin Server，则由Cache Server读取源服务器的响应头，以确定响应是缓存还是简单传递。
    ## 东西好多：https://blog.text.wiki/2017/04/10/nginx-cache.html
    proxy_cache_path  Proxy_cache_path levels=1:2 keys_zone=pnc:300m inactive=7d max_size=10g;
    proxy_temp_path   Proxy_temp_path;
    proxy_cache_key   $host$uri$is_args$args;
​
    http2_max_field_size            4k;
    http2_max_header_size           16k;
    http2_max_requests              1000;
    
    types_hash_max_size             2048;
    server_names_hash_max_size      1024;
    server_names_hash_bucket_size   64;
    map_hash_bucket_size            64;
    
    proxy_headers_hash_max_size     512;
    proxy_headers_hash_bucket_size  64;
    
    variables_hash_bucket_size      128;
    variables_hash_max_size         2048;
    
    underscores_in_headers          off;
    ignore_invalid_headers          on;
    
    limit_req_status                503;
    limit_conn_status               503;
    
    include /etc/nginx/mime.types;
    default_type text/html;
    
​
    
    # Custom headers for response
    
    server_tokens on;
    
    # disable warnings
    uninitialized_variable_warn off;
    
    # Additional available variables:
    # $namespace
    # $ingress_name
    # $service_name
    # $service_port
    log_format upstreaminfo '$the_real_ip - [$the_real_ip] - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" $request_length $request_time [$proxy_upstream_name] $upstream_addr $upstream_response_length $upstream_response_time $upstream_status $req_id';
    
​
    
    access_log /var/log/nginx/access.log upstreaminfo  if=$loggable;
    
    error_log  /var/log/nginx/error.log notice;
    
    
    ssl_protocols TLSv1.2;
    
    # turn on session caching to drastically improve performance
    
    ssl_session_cache builtin:1000 shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # allow configuring ssl session tickets
    ssl_session_tickets on;
    
    # slightly reduce the time-to-first-byte
    ssl_buffer_size 4k;
    
    # allow configuring custom ssl ciphers
    ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256';
    ssl_prefer_server_ciphers on;
    
    ssl_ecdh_curve auto;
    
    proxy_ssl_session_reuse on;
    
    ## upstream
    upstream upstream_balancer {
        server 0.0.0.1; # placeholder
        
        balancer_by_lua_block {
            balancer.balance()
        }
        
        keepalive 32;
        
        keepalive_timeout  60s;
        keepalive_requests 100;
        
    }
​
​
server {
    location / {
        resolver                  127.0.0.1;  
        proxy_cache               pnc;
        proxy_cache_valid         200 304 2h;
        proxy_cache_lock          on;
        proxy_cache_lock_timeout  5s;
        proxy_cache_use_stale     updating error timeout invalid_header http_500 http_502;
​
        proxy_http_version        1.1;
​
        proxy_ignore_headers      Set-Cookie;
        ... ...
    }
    ... ...
}
    
    ## start server _
    server {
        server_name _ ;
        
        listen 80 default_server reuseport backlog=511;
        
        set $proxy_upstream_name "-";
        set $pass_access_scheme $scheme;
        set $pass_server_port $server_port;
        set $best_http_host $http_host;
        set $pass_port $pass_server_port;
        
        listen 443  default_server reuseport backlog=511 ssl http2;
        
        # PEM sha: 4ee9f77aa920980f995205ed55938cbd675c4529
        ssl_certificate       /etc/ingress-controller/ssl/default-fake-certificate.pem;
        ssl_certificate_key   /etc/ingress-controller/ssl/default-fake-certificate.pem;
        
        ssl_certificate_by_lua_block {
            certificate.call()
        }
        
        location / {
            
            set $namespace      "";
            set $ingress_name   "";
            set $service_name   "";
            set $service_port   "0";
            set $location_path  "/";
            
            rewrite_by_lua_block {
                lua_ingress.rewrite({
                    force_ssl_redirect = false,
                    use_port_in_redirects = false,
                })
                balancer.rewrite()
                plugins.run()
            }
            
            header_filter_by_lua_block {
                
                plugins.run()
            }
            body_filter_by_lua_block {
                
            }
            
            log_by_lua_block {
                
                balancer.log()
                
                monitor.call()
                
                plugins.run()
            }
            
            if ($scheme = https) {
                more_set_headers                        "Strict-Transport-Security: max-age=15724800; includeSubDomains";
            }
            
            access_log off;
            
            port_in_redirect off;
            
            set $proxy_upstream_name    "upstream-default-backend";
            set $proxy_host             $proxy_upstream_name;
            
            
            proxy_set_header Host                   $best_http_host;
            
            # Pass the extracted client certificate to the backend
            
            # Allow websocket connections
            proxy_set_header                        Upgrade           $http_upgrade;
            
            proxy_set_header                        Connection        $connection_upgrade;
            
            proxy_set_header X-Request-ID           $req_id;
            proxy_set_header X-Real-IP              $the_real_ip;
            
            proxy_set_header X-Forwarded-For        $the_real_ip;
            
            proxy_set_header X-Forwarded-Host       $best_http_host;
            proxy_set_header X-Forwarded-Port       $pass_port;
            proxy_set_header X-Forwarded-Proto      $pass_access_scheme;
            
            proxy_set_header X-Original-URI         $request_uri;
            
            proxy_set_header X-Scheme               $pass_access_scheme;
            
            # Pass the original X-Forwarded-For
            proxy_set_header X-Original-Forwarded-For $http_x_forwarded_for;
            
            # mitigate HTTPoxy Vulnerability
            # https://www.nginx.com/blog/mitigating-the-httpoxy-vulnerability-with-nginx/
            proxy_set_header Proxy                  "";
            
            # Custom headers to proxied server
            
            proxy_connect_timeout                   5s;
            proxy_send_timeout                      60s;
            proxy_read_timeout                      60s;
            
            proxy_buffering                         off;
            proxy_buffer_size                       4k;
            proxy_buffers                           4 4k;
            proxy_request_buffering                 on;
            
            proxy_http_version                      1.1;
            
            proxy_cookie_domain                     off;
            proxy_cookie_path                       off;
            
            # In case of errors try the next upstream server before returning an error
            proxy_next_upstream                     error timeout;
            proxy_next_upstream_tries               3;
            
            proxy_pass http://upstream_balancer;
            
            proxy_redirect                          off;
            
        }
        
        # health checks in cloud providers require the use of port 80
        location /healthz {
            
            access_log off;
            return 200;
        }
        
        # this is required to avoid error if nginx is being monitored
        # with an external software (like sysdig)
        location /nginx_status {
            
            allow 127.0.0.1;
            
            deny all;
            
            access_log off;
            stub_status on;
        }
        
    }
```

**stream module**

Nginx为了支持TCP和UDP负载均衡实现的模块。

但是，四层的负载都要LVS啊，Nginx不行，因为它的四层代理流量走向如下：

linux收包顺序： [https://blog.502.li/linux-net-and-iptables.html](https://blog.502.li/linux-net-and-iptables.html)

client --&gt; kernel space --&gt; user space --&gt; kernel space

流量经过userspace \(Nginx\)转了一圈，性能怎么能比得过lvs。

so，Nginx的Stream模块了解下就行了。

```text
stream {
  
    upstream backend {
        hash $remote_addr consistent;
​
        server backend1.example.com:12345 weight = 5;
        server 127.0.0.1:12345 max_fails = 3 fail_timeout = 30s;
        server unix：/ tmp / backend3;
    }
    
    upstream dns {
       server 192.168.0.1:53535;
       server dns.example.com:53;
    }
​
    server {
        listen 12345;
        proxy_connect_timeout 1s;
        proxy_timeout 3s;
        proxy_pass backend;
    }
​
    server {
        listen 127.0.0.1:53 udp reuseport;
        proxy_timeout 20s;
        proxy_pass dns;
    }
​
    server {
        listen [::1]:12345;
        proxy_pass unix:/tmp/stream.socket;
    }
}
```

**mail module**

略略略，跳过。

**Nginx 工作模式**

![nginx&#x6574;&#x4F53;&#x67B6;&#x6784;](file://D:/worker/%E6%BC%A0%E5%9D%A6%E5%B0%BC/%E7%96%AB%E6%83%85%E6%96%87%E6%A1%A3/%E6%88%90%E5%93%81/alicloudnative-image/nginx%E6%95%B4%E4%BD%93%E6%9E%B6%E6%9E%84.webp?lastModify=1585308012)

**主进程**

主进程并不处理网络请求，主要负责调度工作进程，也就是图示的3项：加载配置、启动工作进程、非停升级。

主程序Masterprocess启动后，通过一个for循环来接收和处理外部信号

主进程通过fork\(\)函数产生worker子进程，每个子进程执行一个for循环来实现Nginx服务器对事件的接收和处理

一般推荐worker进程数与CPU内核数一致，这样一来不存在大量的子进程生成和管理任务，避免了进程之间竞争CPU资源和进程切换的开销。

**worker processes**

服务器实际处理网络请求及响应的是工作进程（worker），在类unix系统上，Nginx可以配置多个worker，而每个worker进程都可以同时处理数以千计的网络请求。

所有worker进程的listenfd会在新连接到来时变得可读，为保证只有一个进程处理该连接，所有worker进程在注册listenfd读事件前抢占accept\_mutex

抢到互斥锁的那个进程注册listenfd读事件，在读事件里调用accept接受该连接。

当一个worker进程在accept这个连接之后，就开始读取请求，解析请求，处理请求，产生数据后，再返回给客户端，最后才断开连接，一个完整的请求就是这样。

**模块化设计**

Nginx的worker进程，包括核心和功能性模块，核心模块负责维持一个运行循环（run-loop），执行网络请求处理的不同阶段的模块功能。

比如：网络读写、存储读写、内容传输、外出过滤，以及将请求发往上游服务器等。

**事件驱动模型**

在计算机编程领域，事件驱动模型对应一种程序设计方式，Event-driven programming，即事件驱动程序设计。 事件驱动模型一般是由`事件收集器`，`事件发送器`，`事件处理器`三部分基本单元组成。

对于nginx而言，事件机制的处理无非就是几个部分：

* 网络IO事件的处理
* 文件IO事件的处理
* 定时器事件的处理

Nginx服务器的事件驱动模型，如下图所示：

![nginx&#x4E8B;&#x4EF6;&#x9A71;&#x52A8;&#x6A21;&#x578B;](file://D:/worker/%E6%BC%A0%E5%9D%A6%E5%B0%BC/%E7%96%AB%E6%83%85%E6%96%87%E6%A1%A3/%E6%88%90%E5%93%81/alicloudnative-image/nginx%E4%BA%8B%E4%BB%B6%E9%A9%B1%E5%8A%A8%E6%A8%A1%E5%9E%8B.webp?lastModify=1585308012)

如上图所示，Nginx的事件驱动模型由事件收集器、事件发送器和事件处理器三部分基本单元组成。

事件收集器：负责收集worker进程的各种IO请求；

事件发送器：负责将IO事件发送到事件处理器；

事件处理器：负责各种事件的响应工作。

事件发送器将每个请求放入一个待处理事件列表，使用非阻塞I/O方式调用事件处理器来处理该请求。

其处理方式称为“多路IO复用方法”，常见的包括以下三种：select模型、poll模型、epoll模型。

IO multiplexing\(select/poll/epoll\)

虽然I/O多路复用的函数也是阻塞的，但是其与以上两种还是有不同的，I/O多路复用是阻塞在select，epoll这样的系统调用之上，而没有阻塞在真正的I/O系统调用如recvfrom之上。

epoll同步的。属于IO多路复用的一种（IO多路复用还有一个名字叫做事件驱动，这个概念和异步的概念有点相似，所以很容易混）

所有I/O多路复用操作都是同步的，涵盖`select/poll`。 阻塞/非阻塞是相对于同步I/O来说的，与异步I/O无关。 `select/poll/epoll`本身是同步的，可以阻塞也可以不阻塞。

在使用`epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout)`函数来获取是否有发生变化/事件的文件描述符时，可以通过指定timeout来指定该调用是否阻塞（当timeout=-1时，会无限期阻塞；当timeout=0时，会立即返回）

当前是否有发生变化/事件的文件描述符需要通过`epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout)`显式地进行查询，因此不是异步；

> 异步的核心是消息通知机制。

epoll将多个IO请求合并发给一个事件处理器。

_高效IO复用机制要满足_：协调者消耗最少的系统资源、最小化FD的等待时间、最大化FD的数量、任务处理线程最少的空闲、多快好省完成任务等。

**AIO**

多线程仅用在aio模型（IO模型）中对本地文件的操作上，出发点就是以非阻塞模式来提高文件IO的效率和并发能力；

对Nginx来说，线程池的作用跟快递点一样。它包括一个任务队列以及配套线程。当一个worker进行需要处理阻塞操作时，它会将这个任务交给线程池来完成。

**Nginx进程通信**

* 共享内存

  nginx进程间共享的数据就需要使用共享内存，比如服务器中HTTP的连接数：成功建立连接的TCP数、正在发送TCP流的连接数、正在接收TCP流的连接数等等。对于这些共享的数据，就应该使用共享内存。每个worker进程修改的都是共享内存中的数据，修改是对所有进程都有效的。

* Socket

  我们知道nginx有master进程和worker进程，那么master进程是如何向worker进程发送消息的呢？worker进程又是如何接收master发送过来的消息呢？答案就是使用套接字。注意的是虽然套接字是双工的，但目前套接字仅用于master进程管理worker进程，而没用于worker发消息给master，或者worker进程间通信。

  单向通道，只有master --&gt; worker。

* 信号：signal

**Nginx Reload**

当reload的时候，发生了以下事件：

1. master进程检查配置的正确性，如果不正确则不reload，nginx按照原配置工作。
2. 如果正确，则nginx启动新的worker，采用新的配置文件。
3. nginx将新的请求分配给新的worker。
4. nginx等以前的worker处理完旧的请求，关闭以前的woker。
5. 重复上面过程，知道全部旧的worker进程都被关闭掉

Reload时可能导致的问题：

1. 如果有大量请求，则有可能请求丢失。 - 在某一时刻worker进程是之前的二倍，导致系统资源的下降，多个worker之间会进行锁的竞争。

   惊群可能导致，请求丢失。

2. 如果worker子进程比较多则会耗费内存。 - 如果老的worker和后端的机器保持长连接，则在一段时间内nginx的worker进程个数会随着reload的次数成倍数增加

   系统负载增加

**模块处理**

1. 当服务器启动，**每个handlers\(处理模块\)都有机会映射到配置文件中定义的特定位置（location）**；如果有多个handlers\(处理模块\)映射到特定位置时，只有一个会“赢”（说明配置文件有冲突项，应该避免发生）。

   处理模块以三种形式返回：

   > OK
   >
   > ERROR
   >
   > 或者放弃处理这个请求而让默认处理模块来处理（主要是用来处理一些静态文件，事实上如果是位置正确而真实的静态文件，默认的处理模块会抢先处理）。

2. **如果handlers\(处理模块\)把请求反向代理到后端的服务器，就变成另外一类的模块：load-balancers（负载均衡模块）**。负载均衡模块的配置中有一组后端服务器，当一个HTTP请求过来时，它决定哪台服务器应当获得这个请求。

   **Nginx的负载均衡模块采用两种方法：**

   > **轮转法**，它处理请求就像纸牌游戏一样从头到尾分发；
   >
   > **IP哈希法**，在众多请求的情况下，它确保来自同一个IP的请求会分发到相同的后端服务器。

3. **如果handlers\(处理模块\)没有产生错误，filters（过滤模块）将被调用**。多个filters（过滤模块）能映射到每个位置，所以（比如）每个请求都可以被压缩成块。它们的执行顺序在编译时决定。

   **filters（过滤模块）是经典的“接力链表（CHAIN OF RESPONSIBILITY）”模型**：一个filters（过滤模块）被调用，完成其工作，然后调用下一个filters（过滤模块），直到最后一个filters（过滤模块）。

   **过滤模块链的特别之处在于：**

   > **每个filters（过滤模块）不会等上一个filters（过滤模块）全部完成；**
   >
   > **它能把前一个过滤模块的输出作为其处理内容；有点像Unix中的流水线；**

   **过滤模块能以buffer（缓冲区）为单位进行操作，这些buffer一般都是一页（4K）大小，当然你也可以在nginx.conf文件中进行配置**。这意味着，比如，模块可以压缩来自后端服务器的响应，然后像流一样的到达客户端，直到整个响应发送完成。

   **总之，过滤模块链以流水线的方式高效率地向客户端发送响应信息。**

4. **所以总结下上面的内容，一个典型的HTTP处理周期是这样的：**

   > 客户端发送HTTP请求 –&gt;
   >
   > Nginx基于配置文件中的位置选择一个合适的处理模块 -&gt;
   >
   > \(如果有\)负载均衡模块选择一台后端服务器 –&gt;
   >
   > 处理模块进行处理并把输出缓冲放到第一个过滤模块上 –&gt;
   >
   > 第一个过滤模块处理后输出给第二个过滤模块 –&gt;
   >
   > 然后第二个过滤模块又到第三个 –&gt;
   >
   > 依此类推 –&gt; 最后把响应发给客户端。

**下图展示了Nginx模块处理流程：**

![Nginx&#x6A21;&#x5757;&#x5904;&#x7406;&#x6D41;&#x7A0B;](file://D:/data_files/MarkDown/Images/Nginx%E6%A8%A1%E5%9D%97%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B.png?lastModify=1585308012)

Nginx本身做的工作实际很少，当它接到一个HTTP请求时，它仅仅是通过查找配置文件将此次请求映射到一个location block，而此location中所配置的各个指令则会启动不同的模块去完成工作，因此模块可以看做Nginx真正的劳动工作者。通常一个location中的指令会涉及一个handler模块和多个filter模块（当然，多个location可以复用同一个模块）。handler模块负责处理请求，完成响应内容的生成，而filter模块对响应内容进行处理。\*\*

location模块是nginx中用的最多的，也是最重要的模块了，什么负载均衡啊、反向代理啊、虚拟域名啊都与它相关。

**Riot配置**

现在来修改默认配置： 1、使Nginx不以daemon方式运行：

```text
daemon off;
```

 这是因为命令行中调用nginx，Nginx将以daemon方式运行在后台。这会返回exit 0，Docker会认为进程已经退出，然后停止容器。你会发现这种现象经常发生。对于Nginx来说，只要简单地修改下配置就可以解决这个问题。

 2、将Nginx的worker数目提升为2：

```text
worker_processes 2;
```

 这是我每次设置Nginx时必定做的事。当然，你可以选择保持该配置为1。Nginx调优可以单独写一篇文章。我不能告诉你什么是对的。粗略地说，该配置 指定了多少个单独的Nginx进程。CPU数目是一个不错的参考值，当然，很多NGINX专家会说情况远比这个要复杂。

 3、事件调优（Event tuning）：

```text
use epoll;
accept_mutex off; 
```

 打开epolling可以使用高效的连接模型。为了加速，我们关闭了accept\_mutex，因为我们不在乎较低的连接请求数造成的资源浪费。

 4、设置代理头：

```text
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; 
```

 除了关闭daemon模式之外，这是第二个必须的Jenkins代理配置。只有这样，Jenkins才能正确地处理请求，否则会出现一些警告。

 5、客户端大小：

```text
client_max_body_size 300m;
client_body_buffer_size 128k; 
```

 你可能需要这些配置，也可能不需要。不可否认的是，300MB是一个很大的body大小。然而，我们的用户上传文件到Jenkins服务器，其中一些是HPI插件，一些是真实文件。

 6、打开GZIP：

```text
gzip on;
gzip_http_version 1.0;
gzip_comp_level 6;
gzip_min_length 0;
gzip_buffers 16 8k;
gzip_proxied any;
gzip_types text/plain text/css text/xml text/javascript application/xml application/xml+rss application/javascript application/json;
gzip_disable "MSIE [1-6]\.";
gzip_vary on; 
```

 为了加速，我们打开了gzip压缩。

 保存文件为conf/nginx.conf。下一步就是为Jenkins添加特定的配置文件。

代理后端Jenkins的NGINX配置

就像上一章那样，我会先提供一份完整的配置文件，然后再修改特定的配置。你会发现大多数内容可以在Jenkins的 [官方文档](https://www.open-open.com/misc/goto?guid=4959647255221780570)找到。

```text
server {
    listen       80;
    server_name  "";
    access_log off;
    location / {
        proxy_pass         http://jenkins-master:8080;
        proxy_set_header   Host             $host;
        proxy_set_header   X-Real-IP        $remote_addr;
        proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto http;
        proxy_max_temp_file_size 0;
        proxy_connect_timeout      150;
        proxy_send_timeout         100;
        proxy_read_timeout         100;
        proxy_buffer_size          8k;
        proxy_buffers              4 32k;
        proxy_busy_buffers_size    64k;
        proxy_temp_file_write_size 64k; 
    }
}
```

end

**配置Nginx显示真实客户端IP**

SLB会转化为内网请求，通过内网IP进行发送请求。即slb是个外网IP映射到一个内网的VIP,VIP后面接了多个内网IP负责转发，实现负载均衡。

```text
举个栗子：
SLB-->(192.168.97.8/14/63/64)四个Nginx-->后端接Tomcat
但是后端的Tomcat日志信息记录的IP如下：
100.97.181.136, 100.97.183.20, 100.97.182.250, 100.97.183.94（这些是SLB的VIP后端的真实转发IP）
日志内容如下：
2017-10-30 10:29:08,881 [,100.97.181.136,POST,/zzswy/api/zzs/gxrz/saveTokenData?,Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 5.1; Trident/4.0; .NET4.0C; .NET4.0E)] Receive http request(traceId=e1286e2feca54c3fa734da1e853b433d)
。。。
经阿里客服验证，SLB会转化为内网请求，通过内网IP进行发送请求到SLB的后端服务器。
就是类似Haproxy+Nginx这样的结构。通过slb实现的四层代理实现了Nginx的负载
```

负载均衡提供获取客户端真实IP地址的功能，该功能默认是开启的。

* 四层负载均衡（TCP协议）服务可以直接在后端ECS上获取客户端的真实IP地址，无需进行额外的配置。
* 七层负载均衡（HTTP/HTTPS协议）服务需要对应用服务器进行配置，然后使用`X-Forwarded-For`的方式获取客户端的真实IP地址。

  > 注意：负载均衡的HTTPS监听是在负载均衡服务上的加密控制，后端仍旧使用HTTP协议，因此，在Web应用服务器配置上HTTPS和HTTP监听没有区别。

nginx显示后端真实ip模块

1. 重新编译安装`http_realip_module`。
2. 打开`nginx.conf`文件。

   `vi /alidata/server/nginx/conf/nginx.conf`

3. 在以下配置信息后添加新的配置字段和信息。

   需要添加的配置字段和信息为：

   ```text
    set_real_ip_from IP_address real_ip_header X-Forwarded-For;
   ```

   > set\_real\_ip\_from IP：此IP地址不是负载均衡提供的公网IP，具体IP请检查Nginx日志，通常两个IP地址都要写上。

4. 重启Nginx。

源自：Nginx ingress controller，与去掉自动发现pod ip的lua脚本部分

```text
​
# Configuration checksum: 14673928730021971241
​
# setup custom paths that do not require root access
pid /tmp/nginx.pid;
​
load_module /etc/nginx/modules/ngx_http_modsecurity_module.so;
​
daemon off;
​
worker_processes 144;
​
worker_rlimit_nofile 6257;
​
worker_shutdown_timeout 10s ;
​
events {
    multi_accept        on;
    worker_connections  16384;
    use                 epoll;
}
​
​
    ## start server bc.zpepc.com.cn
    server {
        server_name bc.zpepc.com.cn ;
        
        listen 80;
        
        set $proxy_upstream_name "-";
        set $pass_access_scheme $scheme;
        set $pass_server_port $server_port;
        set $best_http_host $http_host;
        set $pass_port $pass_server_port;
        
        location / {
            
            set $namespace      "default";
            set $ingress_name   "bc";
            set $service_name   "ingress-3cafa9da0d8765926c92f81e241f4811";
            set $service_port   "8080";
            set $location_path  "/";
            
            rewrite_by_lua_block {
                lua_ingress.rewrite({
                    force_ssl_redirect = false,
                    use_port_in_redirects = false,
                })
                balancer.rewrite()
                plugins.run()
            }
            
            header_filter_by_lua_block {
                
                plugins.run()
            }
            body_filter_by_lua_block {
                
            }
            
            log_by_lua_block {
                
                balancer.log()
                
                monitor.call()
                
                plugins.run()
            }
            
            port_in_redirect off;
            
            set $proxy_upstream_name    "default-ingress-3cafa9da0d8765926c92f81e241f4811-8080";
            set $proxy_host             $proxy_upstream_name;
            
            client_max_body_size                    1m;
            
            proxy_set_header Host                   $best_http_host;
            
            # Pass the extracted client certificate to the backend
            
            # Allow websocket connections
            proxy_set_header                        Upgrade           $http_upgrade;
            
            proxy_set_header                        Connection        $connection_upgrade;
            
            proxy_set_header X-Request-ID           $req_id;
            proxy_set_header X-Real-IP              $the_real_ip;
            
            proxy_set_header X-Forwarded-For        $the_real_ip;
            
            proxy_set_header X-Forwarded-Host       $best_http_host;
            proxy_set_header X-Forwarded-Port       $pass_port;
            proxy_set_header X-Forwarded-Proto      $pass_access_scheme;
            
            proxy_set_header X-Original-URI         $request_uri;
            
            proxy_set_header X-Scheme               $pass_access_scheme;
            
            # Pass the original X-Forwarded-For
            proxy_set_header X-Original-Forwarded-For $http_x_forwarded_for;
            
            # mitigate HTTPoxy Vulnerability
            # https://www.nginx.com/blog/mitigating-the-httpoxy-vulnerability-with-nginx/
            proxy_set_header Proxy                  "";
            
            # Custom headers to proxied server
            
            proxy_connect_timeout                   5s;
            proxy_send_timeout                      60s;
            proxy_read_timeout                      60s;
            
            proxy_buffering                         off;
            proxy_buffer_size                       4k;
            proxy_buffers                           4 4k;
            proxy_request_buffering                 on;
            
            proxy_http_version                      1.1;
            
            proxy_cookie_domain                     off;
            proxy_cookie_path                       off;
            
            # In case of errors try the next upstream server before returning an error
            proxy_next_upstream                     error timeout;
            proxy_next_upstream_tries               3;
            
            proxy_pass http://upstream_balancer;
            
            proxy_redirect                          off;
            
        }
        
    }
    ## end server bc.zpepc.com.cn
    。。。
        
    
    # TCP services
    
    # UDP services
    
}
​
​
```

