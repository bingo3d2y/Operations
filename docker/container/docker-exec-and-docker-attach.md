# docker exec and docker attach

#### docker attach and docker exec

对比下docker attach 和 docker exec，先说结论：

* docker attach 会attach到容器的init-stdout/stdin/stderr
* docker exec 会创建属于自己的HASH-stdout/stdin/stderr

**docker attach**

To detach the tty without exiting the shell, use the escape sequence Ctrl+P followed by Ctrl+Q. More details [here](https://docs.docker.com/engine/reference/commandline/attach/).

Additional info from [this source](https://groups.google.com/forum/#!msg/docker-user/nWXAnyLP9-M/kbv-FZpF4rUJ):

* docker run -t -i → can be detached with `^P^Q`and reattached with docker attach

  ```text
  -t              : Allocate a pseudo-tty
  -i              : Keep STDIN open even if not attached
  ## 确实只有在同时使用 -t 和 -i 的时候才能使用<Ctrl+p> + <Ctrl+q> 去detached container
  $ docker attach c2
  read escape sequence
  ​
  # 下面两种情况直接“退出容器了”了
  # 不过结束方式不一样，一个是中断另个一是kill
  ```

* docker run -i → **cannot be** detached with `^P^Q`; will disrupt stdin
* docker run → **cannot be detached** with `^P^Q`; can SIGKILL client; can reattach with docker attach

wow, 其实官网上已经给了说明了：

`CTRL-c` sends a `SIGINT` to the container. If the container was run with `-i` and `-t`, you can detach from a container and leave it running using the `CTRL-p CTRL-q` key sequence.

docker 可以通过

> **Note:** A process running as PID 1 inside a container is treated specially by Linux: it ignores any signal with the default action. So, the process will not terminate on `SIGINT` or `SIGTERM` unless it is coded to do so.
>
> 厉害--

`-t`参数，意思是让docker为容器创建一个pseudo-tty，**在这种情况下，stdout和stderr将采用同样的通道，即容器中进程往stderr中输出数据时，会写到init-stdout中。**

```text
​
$ docker run -d nginx:1.12
762493fad83758d28459ea7cbd57b013c6074d37a4f6e2457f70d53fea957957
$ ls /run/docker/containerd/762493fad83758d28459ea7cbd57b013c6074d37a4f6e2457f70d53fea957957/
init-stderr  init-stdout
​
# -i 提供init-stdin
$ docker run -id nginx:1.12
4331c5cd19b9c5e531510eb011d11bfb99c4f0f8c47ed030789cbd3cb5d5a4bc
$ ls /run/docker/containerd/4331c5cd19b9c5e531510eb011d11bfb99c4f0f8c47ed030789cbd3cb5d5a4bc/
init-stderr  init-stdin  init-stdout
​
# -t 合并stdout和stderr
$ docker run -itd nginx:1.12
9906aad8709549f5ab9fe1f926de747bb2c703762c9db5ebd2da49d373d3c9f4
$ ls /run/docker/containerd/9906aad8709549f5ab9fe1f926de747bb2c703762c9db5ebd2da49d373d3c9f4/
init-stdin  init-stdout
​
​
## 因为nginx image默认执行的是nginx -g daemon（run-in-front）即全部输出到stdout
## 又因为-t参数是stdout和stderr合并了，所以error日志也会显示。
## RUN ln -sf /dev/stdout /var/log/nginx/access.log && ln -sf /dev/stderr /var/log/nginx/error.log
$ docker logs 433
172.18.0.1 - - [01/Mar/2020:12:56:47 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.47.0" "-"
2020/03/01 12:56:50 [error] 5#5: *2 open() "/usr/share/nginx/html/kk" failed (2: No such file or directory), client: 172.18.0.1, server: localhost, request: "HEAD /kk HTTP/1.1", host: "172.18.0.6"
172.18.0.1 - - [01/Mar/2020:12:56:50 +0000] "HEAD /kk HTTP/1.1" 404 0 "-" "curl/7.47.0" "-"
​
$ docker logs 76 
2020/03/01 12:56:54 [error] 5#5: *1 open() "/usr/share/nginx/html/kk" failed (2: No such file or directory), client: 172.18.0.1, server: localhost, request: "HEAD /kk HTTP/1.1", host: "172.18.0.7"
172.18.0.1 - - [01/Mar/2020:12:56:54 +0000] "HEAD /kk HTTP/1.1" 404 0 "-" "curl/7.47.0" "-"
​
$ docker logs 990
2020/03/01 12:56:58 [error] 5#5: *1 open() "/usr/share/nginx/html/kk" failed (2: No such file or directory), client: 172.18.0.1, server: localhost, request: "HEAD /kk HTTP/1.1", host: "172.18.0.8"
172.18.0.1 - - [01/Mar/2020:12:56:58 +0000] "HEAD /kk HTTP/1.1" 404 0 "-" "curl/7.47.0" "-
​
```

**docker exec**

docker exec 会创建属于自己的pid-stdout/stdin/stderr（pipes ）, 所以退出pid-stdout时不会影响container的pid 1.

 **docker 不是容器里面的 进程号为1的进程fork处理。docker的1，只有形没有神的。**

docker exec 时，`ctrl+c`无效，`ctrl+d`才能退出，因为你要退出的不是一个process而是pipes\(FIFOs\).

docker 通过这个些pipes模拟了IO,所以你要与process首先就是与这些pipes通信。所以退出使用: Ctrl+D

```text
# docker exec时，pipes (FIFOs) 增加
$ ls /run/docker/containerd/9906aad8709549f5ab9fe1f926de747bb2c703762c9db5ebd2da49d373d3c9f4/
init-stdin  init-stdout
(tty-2)$ docker exec -it 990 bash 
(tty-2)root@9906aad87095:/# 
$ ls /run/docker/containerd/9906aad8709549f5ab9fe1f926de747bb2c703762c9db5ebd2da49d373d3c9f4/
0641b78dcf3dcfa2dfdc41b767e45d60f41f0880a5ac8267e249746d10511e21-stdin  0641b78dcf3dcfa2dfdc41b767e45d60f41f0880a5ac8267e249746d10511e21-stdout  init-stdin  init-stdout
​
# 向pipe file输入字符，容器中也有显示
$ echo "test" > /run/docker/containerd/9906aad8709549f5ab9fe1f926de747bb2c703762c9db5ebd2da49d373d3c9f4/0641b78dcf3dcfa2dfdc41b767e45d60f41f0880a5ac8267e249746d10511e21-stdin 
(tty-2)root@9906aad87095:/# test
(tty-2)root@9906aad87095:/# 
​
$ echo "test2" > /run/docker/containerd/9906aad8709549f5ab9fe1f926de747bb2c703762c9db5ebd2da49d373d3c9f4/0641b78dcf3dcfa2dfdc41b767e45d60f41f0880a5ac8267e249746d10511e21-stdout 
(tty-2)root@9906aad87095:/# test2
​
# 退出docker exec(ctrl+d), hash-stdio文件消失
# ctrl+C是无效的。因为你不是要终止一个进程--
(tty-2)root@9906aad87095:/# exit
root@fishong:~# 
root@fishong:~# ls /run/docker/containerd/9906aad8709549f5ab9fe1f926de747bb2c703762c9db5ebd2da49d373d3c9f4/
init-stdin  init-stdout
​
​
## docker attach
## attach不会产生新的stdout/in 所以，直接对attach使用ctrl+d 会终止容器）
## （然后crtl+d退出了）[docker attach吸附到运行在容器的进程上，所以直接Ctrl+D时，该进程会被干掉。]
$ docker attach  ab  
【这个不仅会导致init-stdin消失容器退出。而且当容器COMMAN运行'nginx -g daemon'时会出现“连不上”容器的假象】
# (再次ls发现/run/docker/containerd/Container_ID/目录消失)
$ ls 
​
​
```

**inviolability of private property**

前面提到了容器被终止,`init-std*` 会随着容器目录在/run/docker/containerd/中消失，那么容器正在运行时，删掉`init-*`会影响容器的运行嘛。

答案是不会-- 因为：_inviolability of private property_ 哈哈哈

```text
$ ls /run/docker/containerd/4331c5cd19b9c5e531510eb011d11bfb99c4f0f8c47ed030789cbd3cb5d5a4bc/
init-stderr  init-stdin  init-stdout
$ \rm /run/docker/containerd/4331c5cd19b9c5e531510eb011d11bfb99c4f0f8c47ed030789cbd3cb5d5a4bc/init-std*
$ ls -l /run/docker/containerd/4331c5cd19b9c5e531510eb011d11bfb99c4f0f8c47ed030789cbd3cb5d5a4bc/
total 0
## 当前日志
$ docker logs 433
172.18.0.1 - - [01/Mar/2020:12:56:47 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.47.0" "-"
2020/03/01 12:56:50 [error] 5#5: *2 open() "/usr/share/nginx/html/kk" failed (2: No such file or directory), client: 172.18.0.1, server: localhost, request: "HEAD /kk HTTP/1.1", host: "172.18.0.6"
172.18.0.1 - - [01/Mar/2020:12:56:50 +0000] "HEAD /kk HTTP/1.1" 404 0 "-" "curl/7.47.0" "-"
## 新增访问日志
$ curl  -Isq 172.18.0.6
HTTP/1.1 200 OK
Server: nginx/1.12.2
Date: Mon, 02 Mar 2020 03:20:00 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 11 Jul 2017 13:29:18 GMT
Connection: keep-alive
ETag: "5964d2ae-264"
Accept-Ranges: bytes
​
## 对比日志变化
## 说明： 删掉init-stdin、init-stdout及init-stderr之后，容器仍然正常运行且，日志能正常输出。
$ docker logs 433
172.18.0.1 - - [01/Mar/2020:12:56:47 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.47.0" "-"
2020/03/01 12:56:50 [error] 5#5: *2 open() "/usr/share/nginx/html/kk" failed (2: No such file or directory), client: 172.18.0.1, server: localhost, request: "HEAD /kk HTTP/1.1", host: "172.18.0.6"
172.18.0.1 - - [01/Mar/2020:12:56:50 +0000] "HEAD /kk HTTP/1.1" 404 0 "-" "curl/7.47.0" "-"
172.18.0.1 - - [02/Mar/2020:03:20:00 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.47.0" "-"
root@fishong:~# ls /run/docker/containerd/4331c5cd19b9c5e531510eb011d11bfb99c4f0f8c47ed030789cbd3cb5d5a4bc/
​
​
# filesystem 显示被删掉但是空间并没有被释放
$ lsof  |grep -i del
...
docker-co 17554 17591            root   18w     FIFO               0,19       0t0       1111 /run/docker/containerd/4331c5cd19b9c5e531510eb011d11bfb99c4f0f8c47ed030789cbd3cb5d5a4bc/init-stdin (deleted)
docker-co 17554 17591            root   19w     FIFO               0,19       0t0       1118 /run/docker/containerd/4331c5cd19b9c5e531510eb011d11bfb99c4f0f8c47ed030789cbd3cb5d5a4bc/init-stdout (deleted)
docker-co 17554 17591            root   20u     FIFO               0,19       0t0       1118 /run/docker/containerd/4331c5cd19b9c5e531510eb011d11bfb99c4f0f8c47ed030789cbd3cb5d5a4bc/init-stdout (deleted)
docker-co 17554 17591            root   21r     FIFO               0,19       0t0       1118 /run/docker/containerd/4331c5cd19b9c5e531510eb011d11bfb99c4f0f8c47ed030789cbd3cb5d5a4bc/init-stdout (deleted)
docker-co 17554 17591            root   22u     FIFO               0,19       0t0       1111 /run/docker/containerd/4331c5cd19b9c5e531510eb011d11bfb99c4f0f8c47ed030789cbd3cb5d5a4bc/init-stdin (deleted)
docker-co 17554 17591            root   23r     FIFO               0,19       0t0       1111 /run/docker/containerd/4331c5cd19b9c5e531510eb011d11bfb99c4f0f8c47ed030789cbd3cb5d5a4bc/init-stdin (deleted)
...
​
```

every time a file is opened, the process actually holds the "link" to the same space.

每次文件被进程打开后，都会用一个fd 指向这个 “link”。

```text
$  ll -i /proc/7289/fd/4
199026748 l-wx------ 1 admin admin 64 Nov 29 12:31 /proc/7289/fd/4 -> /backup/applogs/access.log
​
# 软连接不会增加the number of links。
$ ll -iL /backup/applogs/access.log
273819 -rw-r--r-- 1 root root 0 Nov 29 13:32 /backup/applogs/access.log
```

文件名（inodes）仅仅是一个指针，point the memory where the file resides.

The space is physically freed only if there are no links left \(therefore, it's impossible to get to it\). That's the only sensible choice: while the file is being used, it's not important if someone else can no longer access it: you are using it and until you close it, you still have control over it - you won't even notice the filename is gone or moved or whatever. That's even used for tempfiles: some implementations create a file and immediately unlink it, so it's not visible in the filesystem, but the process that created it is using it normally. **Flash plugin is especially fond of this method: all the downloaded video files are held open, but the filesystem doesn't show them.**

> 一个文件一旦被打开使用，则对该文件有绝对的控制直到你主动关闭它。在你使用期间，该文件无论被删除或者重命名都不会影响你的使用。虽然它被删除后对系统来说变的不可见了。（losf\|grep -i del可见）
>
> 只要你还在使用就不会被释放空间。（像warden日志一样，删除之后空间不会被释放。这不怪warden是linux 的特性）。

You can "free" space in two ways then:

1. as mentioned above - you can kill application, which open file. 即`restart Application`
2. you can... truncate file. Even if it's deleted即 `> filename`

这里，很明显重启应用去达到释放disk space是不可取的，所以以后清理文件时记得使用: `> filename`

So, the answer is, while the processes have the files still opened, you shouldn't expect to get the space back. It's not freed, it's being actively used. This is also one of the reasons that applications should really close the files when they finish using them. In normal usage, you shouldn't think of that space as free, and this also shouldn't be very common at all - with the exception of temporary files that are unlinked on purpose, there shouldn't really be any files that you would consider being unused, but still open. Try to review if there is a process that does this a lot and consider how you use it, or just find more space.

> 综上，答案是，你别期望通过删除一个正在被进程打开的文件来达到释放空间的目的。当文件再被使用时，是不会被释放的空间的。正因如此，写程序时一定要记得file\_close\(\),当结束程序时。

**pipe**

管道，顾名思义，两端分别连接输入输出，完成通信。

Like un-named/anonymous pipes, named pipes provide a form of IPC \(Inter-Process Communication\). With anonymous pipes, there's one reader and one writer, but that's not required with named pipes—any number of readers and writers may use the pipe.

> 命名管道提供了一个路径名与之关联，以FIFO文件的形式存储于文件系统中，能够**实现任何两个进程之间通信**。而**匿名管道对于文件系统是不可见的，它仅限于在父子进程之间的通信**。

Named pipes are created via `mkfifo` or `mknod`:

```text
$ mkfifo /tmp/testpipe
$ mknod /tmp/testpipe p
```

The following shell script reads from a pipe. It first creates the pipe if it doesn't exist, then it reads in a loop till it sees "quit":

```text
#!/bin/bash
​
pipe=/tmp/testpipe
​
trap "rm -f $pipe" EXIT
​
if [[ ! -p $pipe ]]; then
    mkfifo $pipe
fi
​
while true
do
    if read line <$pipe; then
        if [[ "$line" == 'quit' ]]; then
            break
        fi
        echo $line
    fi
done
​
echo "Reader exiting"
```

The following shell script writes to the pipe created by the read script. First, it checks to make sure the pipe exists, then it writes to the pipe. If an argument is given to the script, it writes it to the pipe; otherwise, it writes "Hello from PID".

```text
#!/bin/bash
​
pipe=/tmp/testpipe
​
if [[ ! -p $pipe ]]; then
    echo "Reader not running"
    exit 1
fi
​
​
if [[ "$1" ]]; then
    echo "$1" >$pipe
else
    echo "Hello from $$" >$pipe
fi
```

Running the scripts produces:

```text
$ sh rpipe.sh &
[1] 1424
$ sh wpipe.sh
Hello from 23846
$ bash /tmp/wpipe.sh
Hello from 1661
$ bash /tmp/wpipe.sh
Hello from 1666
$ bash /tmp/wpipe.sh quit
Reader exiting
$ 
[1]+  Done                    bash /tmp/rpipe.sh.sh
​
```

根据上面的例子，我们稍微修改下 将从pipe中读取的内容重定向到文件中：`echo $line >> /tmp/logs.json`

即可以模拟docker日志从pipe记录到文件中的了。

参考：

[https://stackoverflow.com/questions/19688314/how-do-you-attach-and-detach-from-dockers-process](https://stackoverflow.com/questions/19688314/how-do-you-attach-and-detach-from-dockers-process)

[https://docs.docker.com/engine/reference/commandline/attach/](https://docs.docker.com/engine/reference/commandline/attach/)

[https://www.linuxjournal.com/content/using-named-pipes-fifos-bash](https://www.linuxjournal.com/content/using-named-pipes-fifos-bash)

