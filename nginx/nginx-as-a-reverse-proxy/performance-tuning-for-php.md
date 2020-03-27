# php 优化

#### PHP

要弄清下面几个概念： cgi、fastcgi、php-fpm

php重启解决大多数问题、、、

**cgi**

CGI是为了保证web server传递过来的数据是标准格式的，方便CGI程序的编写者。

web server（比如说nginx）只是内容的分发者。比如，如果请求`/index.html`，那么web server会去文件系统中找到这个文件，发送给浏览器，这里分发的是静态数据。好了，如果现在请求的是`/index.php`，根据配置文件，nginx知道这个不是静态文件，需要去找PHP解析器来处理，那么他会把这个请求简单处理后交给PHP解析器。Nginx会传哪些数据给PHP解析器呢？url要有吧，查询字符串也得有吧，POST数据也要有，HTTP header不能少吧，好的，CGI就是规定要传哪些数据、以什么样的格式传递给后方处理这个请求的协议。仔细想想，你在PHP代码中使用的用户从哪里来的。

当web server收到`/index.php`这个请求后，会启动对应的CGI程序，这里就是PHP的解析器。接下来PHP解析器会解析php.ini文件，初始化执行环境，然后处理请求，再以规定CGI规定的格式返回处理后的结果，退出进程。web server再把结果返回给浏览器。

**cgi的性能问题**

fork-and-execute

每当web server转发过来一个动态请求页面，cgi都会fork一个新进程来加载配置文件（比如，PHP的php.ini）来初始化执行环境，响应完请求后就会结束进程。周而复始。

这样的效率极其低下，而且耗资源，明显在高并发环境下扛不住。

类似每个worker是一个短连接。

**fastcgi**

Fastcgi是用来提高CGI程序性能的。

PHP-fpm就是针对于PHP的，Fastcgi的一种实现，它负责管理一个进程池，来处理来自Web服务器的请求。

FastCGI像是一个常驻\(long-live\)型的CGI，它可以一直执行着，只要激活后，不会每次都要花费时间去fork一次（这是CGI最为人诟病的fork-and-execute 模式）。它还支持分布式的运算，即 FastCGI 程序可以在网站服务器以外的主机上执行并且接受来自其它网站服务器来的请求。

FastCGI是语言无关的、可伸缩架构的CGI开放扩展，其主要行为是将CGI解释器进程保持在内存中并因此获得较高的性能。众所周知，CGI解释器的反复加载是CGI性能低下的主要原因，如果CGI解释器保持在内存中并接受FastCGI进程管理器调度，则可以提供良好的性能、伸缩性、Fail- Over特性等等。

fastcgi的工作原理：

1. Web Server启动时载入FastCGI进程管理器（IIS ISAPI或Apache Module\)
2. FastCGI进程管理器自身初始化，启动多个CGI解释器进程\(可见多个php-cgi\)并等待来自Web Server的连接。
3. 当客户端请求到达Web Server时，FastCGI进程管理器选择并连接到一个CGI解释器。Web server将CGI环境变量和标准输入发送到FastCGI子进程php-cgi。
4. FastCGI子进程完成处理后将标准输出和错误信息从同一连接返回Web Server。当FastCGI子进程关闭连接时，请求便告处理完成。FastCGI子进程接着等待并处理来自FastCGI进程管理器\(运行在Web Server中\)的下一个连接。 在CGI模式中，php-cgi在此便退出了。

Fastcgi会先启一个master，解析配置文件，初始化执行环境，然后再启动多个worker。当请求过来时，master会传递给一个worker，然后立即可以接受下一个请求。这样就避免了重复的劳动，效率自然是高。而且当worker不够用时，master可以根据配置预先启动几个worker等着；当然空闲worker太多时，也会停掉一些，这样就提高了性能，也节约了资源。这就是fastcgi的对进程的管理。

而且每个worker是长连接。这些个worker就cgi进程。

fastcgi的方式是，web服务器收到一个请求时，他不会重新fork一个进程（因为这个进程在web服务器启动时就开启了，而且不会退出），web服务器直接把内容传递给这个进程（进程间通信，但fastcgi使用了别的方式，tcp方式通信），这个进程收到请求后进行处理，把结果返回给web服务器，最后自己接着等待下一个请求的到来，而不是退出.

**fastcgi跟cgi的区别**

| 名称 | 在web服务器方面 | 在对数据进行处理的进程方面 |
| :--- | :--- | :--- |
| cgi | fork一个新的进程进行处理 | 读取参数，处理数据，然后就结束生命期 |
| fastcgi | 用tcp方式跟远程机子上的进程（远程即端口）或本地进程建立连接（本地进程了就是Socket文件了） | 要开启tcp端口，进入循环，等待数据的到来，处理数据 |

**php-fpm**

php-fpm实现了fastcgi协议。

php-fpm的管理对象是php-cgi。

Fastcgi协议定义的就是如何高效的管理cgi进程。

php-fpm: php FastCGI Process Manager\(FastCGI进程管理器\)

**php-fpm参数优化**

 PHP 是一个短暂的进程，它会泄漏内存，所以在出现高故障时重新启动主进程可以解决很多问题。

优化分为两个配置文件

* `php-fpm.conf`: 这个是php-fpm进程本身的参数配置，它通常配置何时及生命条件下重启cgi进程
* `XXX/pool.d/Thread_Pool_Name.conf`: 这个才是关键具体的处理动态数据的cgi进程配置

PHP请求的实际内存消耗是`max_children * max_requests * 每个请求使用内存`

**性能**

一般php-fpm进程占用20~30m左右的内存就按30m算。如果单独跑php-fpm，动态方式起始值可设置物理内存Mem/30M。

即4G内存的虚拟机，抛下系统本身消耗其他服务损耗的1G，可以内存按3G算，通常进程池设置为60 ~ 80

保险起见，我们都这是60的。

这是IO密集场景下php-fpm进程按30m算，CPU密集场景下，php-fpm woker为CPU合适的1~2倍、

php-fpm 全局配置

```text
$ cat php-fpm.conf
...
emergency_restart_threshold 10
emergency_restart_interval 1m
process_control_timeout 10s
...
```

前两个设置是警告性的，它们告诉 php-fpm 进程，如果 10 个子进程在一分钟内失败，主 php-fpm 进程应该重新启动自己。

这听起来可能不够稳健，但是 PHP 是一个短暂的进程，它会泄漏内存，所以在出现高故障时重新启动主进程可以解决很多问题。

第三个选项是 process\_control\_timeout，它告诉子进程在执行从父进程接收到的信号之前需要等待这么长的时间。这个设置是非常有用的。例如，当父进程发送终止信号时，子进程正在处理某些事情的时候。十秒的时间，他们会有一个更好的机会完成任务并且优雅地退出。

php 线程池配置

pm.max\_requests 太关键了，处理N个请求后，自动重启cgi进程，这样避免内存溢出。

request\_terminate\_timeout 这个也重要，避免php脚本执行时间太长出现502错误。

```text
## 4G虚拟机通常这样配置
## 2G ~ 3G 归php-fpm，其他1 ~ 2G归系统或其他服务消耗
## 一个php-cgi进程大约在30M
$ cat XXX/pool.d/xxx.conf
pm = dynamic
# max_children不能超过总内存的三分之二。
pm.max_children = 90
pm.start_servers = 20
pm.min_spare_servers = 5
pm.max_spare_servers = 35
#max_requests 每个进程处理多少个请求之后自动终止，可以有效防止内存溢出，如果为0则不会自动终止，默认为0#设置每个子进程重生之前服务的请求数. 对于可能存在内存泄漏的第三方模块来说是非常有用的. 如果设置为 '0' 则一直接受请求. 等同于 PHP_FCGI_MAX_REQUESTS 环境变量. 默认值: 0.
## 为避免频繁重启这个可以设置的大一些，但不能是0。
pm.max_requests = 1024
# 一般来说性能越好你可以设置越高，20分钟-30分钟都可以。由于我的服务器PHP脚本需要长时间运行，有的可能会超过10分钟因此我设置了900秒，这样不会导致PHP-CGI死掉而出现502 Bad gateway这个错误。
request_terminate_timeout = N
​
### 参数解释
; Choose how the process manager will control the number of child processes.
; Possible Values:
;   static  - a fixed number (pm.max_children) of child processes;
;   dynamic - the number of child processes are set dynamically based on the
;             following directives. With this process management, there will be
;             always at least 1 children.
;             pm.max_children      - the maximum number of children that can
;                                    be alive at the same time.
;             pm.start_servers     - the number of children created on startup.
;             pm.min_spare_servers - the minimum number of children in 'idle'
;                                    state (waiting to process). If the number
;                                    of 'idle' processes is less than this
;                                    number then some children will be created.
;             pm.max_spare_servers - the maximum number of children in 'idle'
;                                    state (waiting to process). If the number
;                                    of 'idle' processes is greater than this
;                                    number then some children will be killed.
​
```

pm\(process manager\)参数指定了进程管理方式，有两种可供选择：static或dynamic，从字面意思不难理解，为静态或动态方式。如果是静态方式，那么在php-fpm启动的时候就创建了指定数目的进程，在运行过程中不会再有变化\(并不是真的就永远不变\)；而动态的则在运行过程中动态调整，当然并不是无限制的创建新进程，受pm.max\_spare\_servers参数影响；动态适合小内存机器，灵活分配进程，省内存。静态适用于大内存机器，动态创建回收进程对服务器资源也是一种消耗

**nginx fastcgi timeout**

一般nginx响应php，都是通过FastCGI接口来调用，所以fastcgi参数配置很重要，当HTTP服务器每次遇到动态程序时，可以将其直接交付给FastCGI进程来执行，然后将得到的结果返回给浏览器，而很多php的网页都是采用动态程序。所以fastcgi的配置，也起的至关重要的作用。所以这是一个优化不可缺少的一部分。

```text
$ cat nginx.conf
...
    # Defines a timeout for establishing a connection with a FastCGI server.
    # 默认60s
    fastcgi_connect_timeout 300;
    # Sets a timeout for transmitting a request to the FastCGI server.
    # 默认60s
    fastcgi_send_timeout 300;
    # Defines a timeout for reading a response from the FastCGI server.
    # 默认60s
    fastcgi_read_timeout 300;
...
```

**安全**

**cgi.fix\_pathinfo​**

```text
# cat     php.ini                                                                       
; cgi.fix_pathinfo provides *real* PATH_INFO/PATH_TRANSLATED support for CGI.  PHP's
; previous behaviour was to set PATH_TRANSLATED to SCRIPT_FILENAME, and to not grok
; what PATH_INFO is.  For more information on PATH_INFO, see the cgi specs.  Setting
; this to 1 will cause PHP CGI to fix its paths to conform to the spec.  A setting
; of zero causes PHP to behave as before.  Default is 1.  You should fix your scripts
; to use SCRIPT_FILENAME rather than PATH_TRANSLATED.
; http://php.net/cgi.fix-pathinfo                                               ;cgi.fix_pathinfo=1 //需要改成0
​
cgi.fix_pathinfo=0
```

**若不将值设置为0则会出现下面的情况**

比如访问下面这个 URL：

`http://phpvim.net/foo.jpg/a.php/b.php/c.php`

那么根据上面给出的配置，nginx 传递给 FastCGI 的 SCRIPT\_FILENAME 的值为：

`/home/verdana/public_html/unsafe/foo.jpg/a.php/b.php/c.php`

也就是 `$_SERVER['ORIG_SCRIPT_FILENAME']`。

当 php.ini 中 `cgi.fix_pathinfo = 1` 时，PHP CGI 以 / 为分隔符号从后向前依次检查如下路径：

_Setting this to 1 will cause PHP CGI to fix its paths to conform to the spec.（设置为1，逐个确认文件是否存在）_

1./home/verdana/public\_html/unsafe/foo.jpg/a.php/b.php/c.php 2./home/verdana/public\_html/unsafe/foo.jpg/a.php/b.php 3./home/verdana/public\_html/unsafe/foo.jpg/a.php 4./home/verdana/public\_html/unsafe/foo.jpg

直到找个某个存在的文件，如果这个文件是个非法的文件，so… 悲剧了~

比如黑客一段代码放到foo.jpg这个文件里。

不要这么单纯jpg不一定是单纯的jpg,比如我们制作一个 PHP webshell命名为hacker.php，复制命名为hacker.jpg

PHP 会把这个文件\(找到某个存在的文件\)当成 cgi 脚本执行， 并赋值路径给 CGI 环境变量——SCRIPT\_FILENAME，也就是 `$_SERVER['SCRIPT_FILENAME']`的值了。

综上，要修改php.ini，将fix\_pathinfo设成0.即`cgi.fix_pathinfo = 0`

**问题排查**

开大量的 PHP-FPM Worker 进程并不能提升高并发的承载能力，反而会导致大量的 CPU 上下文切换，尤其是如果你的业务偏向于计算、单次请求所消耗的时间很长，增加 PHP-FPM 进程数量反而会导致平均请求处理时长飙升。

在php的运行中，无非是两种场景

1. 大运算
2. 高io

1 大运算的场景，即 php程序需要用大量的cpu资源来进行数据计算之类的操作，在这种场景下，fpm进程可以设置为cpu数量的一倍或者两倍 2 高io场景，php的使用场景中（最起码是本电商场景中）基本上属于高io，因为程序花了大量的时间在等待redis返回等待数据库返回。高io场景下，因为cpu大多处在wa状态下，所以可以尽量的加大fpm进程数，所以这个时候使用内存/30m是更为合理的

经过我们自己真实压测，大量redis和mysql读写的io密集情况下，16G的内存，fpm我们设置为400个的时候qps比fpm 16个 32个要好不少.

**cpu消耗高**

1. 使用 top 找出CPU最高的进程pid
2. strace -p PID（进程数） 来跟踪进程
3. ll /proc/PID/fd 来查看该进程在处理哪些文件

**memory 消耗**

可以查看是不是php进程设置太多了。

`pm.max_spare_servers` : 该值表示保证空闲进程数最大值，如果空闲进程大于此值，此进行清理 `pm.min_spare_servers` : 保证空闲进程数最小值，如果空闲进程小于此值，则创建新的子进程;

　　这两个值均不能不能大于 `pm.max_children` 值，通常设置 `pm.max_spare_servers` 值为 `pm.max_children` 值的60%-80%。

　　正常情况下，一个php--fpm占用内存20~30M

参考

[https://learnku.com/php/t/34358](https://learnku.com/php/t/34358)

[https://www.jianshu.com/p/c94c16a42384](https://www.jianshu.com/p/c94c16a42384)

[https://blog.csdn.net/sinat\_22991367/article/details/73431269](https://blog.csdn.net/sinat_22991367/article/details/73431269)

