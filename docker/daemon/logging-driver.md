# Logging Driver

Docker 包含多个日志记录机制，来帮助你从运行的容器或服务获取日志信息。

docker 支持的`log deriver` ,如下

| Driver | Description |
| :--- | :--- |
| `none` | No logs are available for the container and `docker logs` does not return any output. |
| [`local`](https://docs.docker.com/config/containers/logging/local/) | Logs are stored in a custom format designed for minimal overhead. |
| [`json-file`](https://docs.docker.com/config/containers/logging/json-file/) | The logs are formatted as JSON. The default logging driver for Docker. （默认日志驱动） |
| [`syslog`](https://docs.docker.com/config/containers/logging/syslog/) | Writes logging messages to the `syslog` facility. The `syslog` daemon must be running on the host machine. |
| [`journald`](https://docs.docker.com/config/containers/logging/journald/) | Writes log messages to `journald`. The `journald` daemon must be running on the host machine. |
| [`gelf`](https://docs.docker.com/config/containers/logging/gelf/) | Writes log messages to a Graylog Extended Log Format \(GELF\) endpoint such as Graylog or Logstash. |
| [`fluentd`](https://docs.docker.com/config/containers/logging/fluentd/) | Writes log messages to `fluentd` \(forward input\). The `fluentd` daemon must be running on the host machine. |
| [`awslogs`](https://docs.docker.com/config/containers/logging/awslogs/) | Writes log messages to Amazon CloudWatch Logs. |
| [`splunk`](https://docs.docker.com/config/containers/logging/splunk/) | Writes log messages to `splunk` using the HTTP Event Collector. |
| [`etwlogs`](https://docs.docker.com/config/containers/logging/etwlogs/) | Writes log messages as Event Tracing for Windows \(ETW\) events. Only available on Windows platforms. |
| [`gcplogs`](https://docs.docker.com/config/containers/logging/gcplogs/) | Writes log messages to Google Cloud Platform \(GCP\) Logging. |
| [`logentries`](https://docs.docker.com/config/containers/logging/logentries/) | Writes log messages to Rapid7 Logentries. |

Docker-CE 和 Docker-EE的`docker logs`命令对logging driver支持也有区别：

* Users of Docker Enterprise can make use of “dual logging”, which enables you to use the `docker logs` command for any logging driver.
* When using Docker Community Engine, the `docker logs` command is only available on the following drivers:

  * `local`
  * `json-file`
  * `journald`

  ```text
  ## 由于这个是docker-ce下面也给出了提示，无法查看syslog驱动的日志
  $ docker logs df6e
  "logs" command is supported only for "json-file" and "journald" logging drivers (got: syslog)
  ```

**default logging driver**

Each Docker daemon has a default logging driver, which each container uses unless you configure it to use a different logging driver.

```text
##  查看Docker Daemon 的默认Logging Driver
$ docker info | grep 'Logging Driver'
Logging Driver: json-file
```

设置docker默认日志驱动和参数选项

If the logging driver has configurable options, you can set them in the `daemon.json` file as a JSON object with the key `log-opts`.

```text
## 通常在daemon.json这个文件中配置
## 通常这个文件路径为：/etc/docker/daemon.json
## 配置json-file
​
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3",
    "labels": "production_status",
    "env": "os,customer"
  }
}
​
## 配置fluentd
 {
   "log-driver": "fluentd",
   "log-opts": {
   # 执行自定义的fluentd
     "fluentd-address": "fluentdhost:24224"
   }
 }
 
## 配置syslog
{
  "log-driver": "syslog",
  "log-opts": {
  # 执行自定义的syslogd
    "syslog-address": "udp://1.2.3.4:1111"
  }
}
```

**Use environment variables or labels with logging drivers**

abels 和 env会作为日志的attributes来便于日后的日志分析

```text
# If the logging driver supports it, this adds additional fields to the logging output.
$ docker run -dit --label production_status=testing -e os=ubuntu alpine sh
"attrs":{"production_status":"testing","os":"ubuntu"}
​
```

**Use different logging driver for containers**

When you start a container, you can configure it to use a different logging driver than the Docker daemon’s default, using the `--log-driver` flag. If the logging driver has configurable options, you can set them using one or more instances of the `--log-opt <NAME>=<VALUE>` flag. Even if the container uses the default logging driver, it can use different configurable options.

当你想给容器配置不同于`docker daemon` 的`Logging Driver` 可以在容器启动时，通过`--log-driver` 加 `--log-opt`来启动容器。比如：Harbor的yaml文件就是个不错的例子

```text
## 先定义syslog daemon，然后其他的services在将日志驱动指向这syslogd
$ cat docker-compose.yml 
version: '2'
services:
  log:
    image: goharbor/harbor-log:v1.7.1
    container_name: harbor-log 
    restart: always
    dns_search: .
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - DAC_OVERRIDE
      - SETGID
      - SETUID
    volumes:
      - /var/log/harbor/:/var/log/docker/:z
      - ./common/config/log/:/etc/logrotate.d/:z
    ports:
      - 127.0.0.1:1514:10514
    networks:
      - harbor
  registry:
    image: goharbor/registry-photon:v2.6.2-v1.7.1
    container_name: registry
    restart: always
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
    volumes:
      - /cfs/registry:/storage:z
      - ./common/config/registry/:/etc/registry/:z
      - ./common/config/custom-ca-bundle.crt:/harbor_cust_cert/custom-ca-bundle.crt:z
    networks:
      - harbor
    dns_search: .
    depends_on:
      - log
    logging:
      driver: "syslog"
      options:  
        syslog-address: "tcp://127.0.0.1:1514"
        #  这写个tag 就是日志的名字比如： registry.log
        # 由syslogd接收并处理这个tags字段。
        tag: "registry"
  registryctl:
    ...
​
​
### 这个registry 类似
$ docker run -itd --log-driver syslog --log-opt syslog-address="tcp://127.0.0.1:1514"   --log-opt tag="registry"
```

The following example starts an Alpine container with the `none` logging driver.

```text
$ docker run -it --log-driver none alpine ash
​
```

To find the current logging driver for a running container, if the daemon is using the `json-file` logging driver, run the following `docker inspect` command, substituting the container name or ID for `<CONTAINER>`:

```text
# 之前说了，container的logging driver可以和docker daemon的不一样所以 可以用inspect查看 
$ docker inspect -f '{{.HostConfig.LogConfig.Type}}' <CONTAINER>
json-file
​
```

**Configure logging drivers**

**josn-file**

json-file log dervier : The default logging driver for Docker.

默认都是`json-file`日志都保存在`<>/containers/<Contain_ID>/<Contain_ID>-json.log`

```text
## 启动一个容器
$ docker run -itd nginx:1.12
1b32997e42af9c7eff5b2076873744437ebdcd99b2ca0251cbb68ae4d639d2ae
​
$ docker inspect 1b32 |grep -i "ipaddress"
            "SecondaryIPAddresses": null,
            "IPAddress": "172.18.0.4",
                    "IPAddress": "172.18.0.4",
## 访问两次，提供访问日志
$ curl  172.18.0.4 -I 
$ curl  172.18.0.4 -I 
​
## docker logs 命令显示日志
$ docker logs 1b32
172.18.0.1 - - [01/Mar/2020:08:32:13 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.47.0" "-"
172.18.0.1 - - [01/Mar/2020:08:32:16 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.47.0" "-"
## 直接读取json-file访问日志
$ tail /var/lib/docker/containers/1b32997e42af9c7eff5b2076873744437ebdcd99b2ca0251cbb68ae4d639d2ae/1b32997e42af9c7eff5b2076873744437ebdcd99b2ca0251cbb68ae4d639d2ae-json.log 
{"log":"172.18.0.1 - - [01/Mar/2020:08:32:13 +0000] \"GET / HTTP/1.1\" 200 612 \"-\" \"curl/7.47.0\" \"-\"\r\n","stream":"stdout","time":"2020-03-01T08:32:13.575596037Z"}
{"log":"172.18.0.1 - - [01/Mar/2020:08:32:16 +0000] \"HEAD / HTTP/1.1\" 200 0 \"-\" \"curl/7.47.0\" \"-\"\r\n","stream":"stdout","time":"2020-03-01T08:32:16.684651617Z"}
​
```

The `json-file` logging driver supports the following logging options:

| Option | Description | Example value |
| :--- | :--- | :--- |
| `max-size` | The maximum size of the log before it is rolled. A positive integer plus a modifier representing the unit of measure \(`k`, `m`, or `g`\). Defaults to -1 \(unlimited\). | `--log-opt max-size=10m` |
| `max-file` | The maximum number of log files that can be present. If rolling the logs creates excess files, the oldest file is removed. **Only effective when max-size is also set.** A positive integer. Defaults to 1. | `--log-opt max-file=3` |
| `labels` | Applies when starting the Docker daemon. A comma-separated list of logging-related labels this daemon accepts. Used for advanced [log tag options](https://docs.docker.com/config/containers/logging/log_tags/). | `--log-opt labels=production_status,geo` |
| `env` | Applies when starting the Docker daemon. A comma-separated list of logging-related environment variables this daemon accepts. Used for advanced [log tag options](https://docs.docker.com/config/containers/logging/log_tags/). | `--log-opt env=os,customer` |
| `env-regex` | Similar to and compatible with `env`. A regular expression to match logging-related environment variables. Used for advanced [log tag options](https://docs.docker.com/config/containers/logging/log_tags/). | `--log-opt env-regex=^(os|customer).` |

示例，设置该容器日志默认每份只存10M共保存3份

```text
$ docker run -it --log-opt max-size=10m --log-opt max-file=3 alpine ash
```

**journald**

Writes log messages to `journald`.

使用`journald log deriver` 时，容器日志会被重定向到`journald` ,可以用`journalctl` 查看

```text
#  container 使用journald log deriver的情况
$ docker logs 2a440addaa8b
172.17.0.1 - - [13/Apr/2018:06:20:19 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.29.0" "-"
172.17.0.1 - - [13/Apr/2018:06:20:21 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.29.0" "-"
$ systemctl list-units |grep 2a440addaa8b
docker-2a440addaa8b2162f1c8337b11c9ee0d7ab58ce5e8595f1de5567e0d95a8017b.scope       loaded active running   docker container 2a440addaa8b2162f1c8337b11c9ee0d7ab58ce5e8595f1de5567e0d95a8017b
​
$ journalctl  -u docker-2a440addaa8b2162f1c8337b11c9ee0d7ab58ce5e8595f1de5567e0d95a8017b.scope 
-- Logs begin at Thu 2018-01-25 16:24:04 CST, end at Fri 2018-04-13 15:00:23 CST. --
Apr 13 14:19:21 docker-registry systemd[1]: Started docker container 2a440addaa8b2162f1c8337b11c9ee0d7ab58ce5e8595f1de5567e0d95a8017b.
Apr 13 14:19:21 docker-registry systemd[1]: Starting docker container 2a440addaa8b2162f1c8337b11c9ee0d7ab58ce5e8595f1de5567e0d95a8017b.
​
# Docker daemon 使用journald  ，而 container使用自定义使用syslog
$  docker inspect -f '{{.HostConfig.LogConfig.Type}}'  6a
syslog
$ journalctl  -u docker-6a323ddab5c9fd49aef7716458727d0e3c5035932c656067dbe08efe4220d56b.scope -n 5
-- Logs begin at Thu 2018-01-25 16:24:04 CST, end at Fri 2018-04-13 15:01:02 CST. --
Apr 02 13:36:10 docker-registry systemd[1]: Starting docker container 6a323ddab5c9fd49aef7716458727d0e3c5035932c656067dbe08efe4220d56b.
Apr 02 13:51:13 docker-registry systemd[1]: Started docker container 6a323ddab5c9fd49aef7716458727d0e3c5035932c656067dbe08efe4220d56b.
Apr 02 13:51:13 docker-registry systemd[1]: Starting docker container 6a323ddab5c9fd49aef7716458727d0e3c5035932c656067dbe08efe4220d56b.
Apr 11 18:40:51 docker-registry systemd[1]: Started docker container 6a323ddab5c9fd49aef7716458727d0e3c5035932c656067dbe08efe4220d56b.
Apr 11 18:40:51 docker-registry systemd[1]: Starting docker container 6a323ddab5c9fd49aef7716458727d0e3c5035932c656067dbe08efe4220d56b.
$ ls /var/log/harbor/2018-04-12/
CRON.log                       docker-current_proxy.log     docker-current_registry.log.1  docker-current_registry.log.3
docker-current_jobservice.log  docker-current_registry.log  docker-current_registry.log.2  docker-current_ui.log
​
如上，当containers自定义使用syslog时，journalctl 只能看到container的重启信息，具体日志内容记录到文件里面了。
```

Use the `--log-opt NAME=VALUE` flag to specify additional `journald` logging driver options.

| Option | Required | Description |
| :--- | :--- | :--- |
| `tag` | optional | Specify template to set `CONTAINER_TAG` and `SYSLOG_IDENTIFIER` value in journald logs. Refer to [log tag option documentation](https://docs.docker.com/engine/admin/logging/log_tags/) to customize the log tag format |
| `label` | optional | Comma-separated list of keys of labels, which should be included in message, if these labels are specified for the container. |
| `env` | optional | Comma-separated list of keys of environment variables, which should be included in message, if these variables are specified for the container. |
| `env-regex` | optional | Similar to and compatible with env. A regular expression to match logging-related environment variables. Used for advanced [log tag options](https://docs.docker.com/engine/admin/logging/log_tags/). |

If a collision occurs between label and env keys, the value of the env takes precedence. Each option adds additional fields to the attributes of a logging message.

Example：

```text
$ docker run --log-driver=journald \
--log-opt labels=location \
--log-opt env=TEST \
--env "TEST=false" \
--label location=west \
your/application
```

**syslog**

The `syslog` logging driver routes logs to a `syslog` server.

The `syslog` daemon must be running on the host machine.

这里，需要注意的时，syslog daemon 必须和容器在同一个宿主机上。

syslog作为日志驱动的典型应用就是harbor,可以直接查看Harbor的docker-compose.yaml.

```text
# 启动syslog driver 的容器并接入rsyslog server 
## 注意这里一定要用--net=deploy_default 因为这个log containers在这个网桥下面--
$ docker run -d   --net=deploy_default --log-driver=syslog --log-opt syslog-address="tcp://172.18.0.2:514" --log-opt tag="nginx-test"  05a
022b93cc5434ef50da4fd213bcca5f789af55a2fcdb76c5a67ffd995e887da74
​
$ docker logs 022
"logs" command is supported only for "json-file" and "journald" logging drivers (got: syslog)
$ docker exec  022  ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
inet 127.0.0.1/8 scope host lo
 valid_lft forever preferred_lft forever
inet6 ::1/128 scope host 
 valid_lft forever preferred_lft forever
144: eth0@if145: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
link/ether 02:42:ac:12:00:08 brd ff:ff:ff:ff:ff:ff
inet 172.18.0.8/16 scope global eth0
 valid_lft forever preferred_lft forever
inet6 fe80::42:acff:fe12:8/64 scope link 
 valid_lft forever preferred_lft forever
​
$ curl -I   172.18.0.8 
HTTP/1.1 200 OK
Server: nginx/1.11.5
Date: Fri, 13 Apr 2018 08:26:49 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 11 Oct 2016 15:03:01 GMT
Connection: keep-alive
ETag: "57fcff25-264"
Accept-Ranges: bytes
​
## 进到log容器中确认日志：  具体的rsyslog的配置文件也在容器里面  /etc/rsyslog.d/rsyslog_docker.conf 
## 通知日志目录使用volume挂载到外部存储。
$ docker exec -it ea8 /bin/bash
root@ea86b272cd09:/#  tail -2 /var/log/docker/2018-04-13/docker-current_nginx-test.log
Apr 13 16:05:30 docker-registry docker-current/nginx-test[950]: 172.18.0.1 - - [13/Apr/2018:08:05:30 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.29.0" "-"
Apr 13 16:26:49 docker-registry docker-current/nginx-test[950]: 172.18.0.1 - - [13/Apr/2018:08:26:49 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.29.0" "-"
​
​
$ cd /var/log/harbor/2018-04-13/
$ tail docker-current_nginx-test.log 
Apr 13 15:20:51 docker-registry docker-current/nginx-test[950]: 172.17.0.1 - - [13/Apr/2018:07:20:51 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.29.0" "-"
Apr 13 15:58:05 docker-registry docker-current/nginx-test[950]: 172.18.0.1 - - [13/Apr/2018:07:58:05 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"
Apr 13 16:04:34 docker-registry docker-current/nginx-test[950]: 172.18.0.1 - - [13/Apr/2018:08:04:34 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"
Apr 13 16:05:30 docker-registry docker-current/nginx-test[950]: 172.18.0.1 - - [13/Apr/2018:08:05:30 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.29.0" "-"
Apr 13 16:26:49 docker-registry docker-current/nginx-test[950]: 172.18.0.1 - - [13/Apr/2018:08:26:49 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.29.0" "-"
​
###  使用rsyslog服务的publish端口1514
$ docker run -d   --net=deploy_default --log-driver=syslog --log-opt syslog-address="tcp://127.0.0.1:1514" --log-opt tag="nginx-test-2"  05a
1d8ca68a1215fd2b54b9693a720d31488ff1614251e5cf106e4045723880811a
$ curl  -I 172.18.0.9
$ tail /var/log/harbor/2018-04-13/docker-current_nginx-test-2.log 
Apr 13 16:33:51 docker-registry docker-current/nginx-test-2[950]: 172.18.0.1 - - [13/Apr/2018:08:33:51 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.29.0" "-"
​
### 使用默认网桥而非rsyslog container所在的网桥
$ docker run -d  --log-driver=syslog --log-opt syslog-address="tcp://127.0.0.1:1514" --log-opt tag="nginx-test-3"  05a
43526aeffcf21d260e3df2b3c9b9ff318a894d88f90eee165a43650f46d49f7a
$ curl  -I 172.17.0.3
$ tail /var/log/harbor/2018-04-13/docker-current_nginx-test-3.log 
Apr 13 16:35:55 docker-registry docker-current/nginx-test-3[950]: 172.17.0.1 - - [13/Apr/2018:08:35:55 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.29.0" "-"
​
```

**Note**: The syslog-address supports both UDP and TCP.

**View logs for a container or service**

By default, `docker logs` shows the command’s `STDOUT` and `STDERR`.

默认情况下，docker logs只会显示 `STDOUT` and `STDERR` 的信息。

In some cases, `docker logs` may not show useful information unless you take additional steps.

但是有些情况下，`docker logs` 可能不会显示有用的信息。

* If you use a [logging driver](https://docs.docker.com/config/containers/logging/configure/) which sends logs to a file, an external host, a database, or another logging back-end, `docker logs` may not show useful information.

  如前面介绍的，如果设置`docker logs`不支持的日志驱动类型，则`docker logs` 无法显示日志信息，

* If your image runs a non-interactive process such as a web server or a database, that application may send its output to log files instead of `STDOUT` and `STDERR`.

  如果应用的日志没有输出到stdout/stderr，那么`docker logs`就无法读取到了。

In the first case, your logs are processed in other ways and you may choose not to use `docker logs`. In the second case, the official `nginx` image shows one workaround, and the official Apache `httpd`image shows another.

> 第一种情况，你可以用你选择合适的docker logging deriver 来管理和显示你的日志、eg：docker-ce仅支持json-file、journald和local.

第二种情况则需要将日志文件重定向到stdouth和stderr,Docker Hub上nginx镜像的Dockerfile就是个不错的例子。

The official `nginx` image creates a symbolic link from `/dev/stdout` to `/var/log/nginx/access.log`, and creates another symbolic link from `/dev/stderr` to `/var/log/nginx/error.log`, overwriting the log files and causing logs to be sent to the relevant special device instead.

`RUN ln -sf /dev/stdout /var/log/nginx/access.log && ln -sf /dev/stderr /var/log/nginx/error.log`

简单方便的将日志输出重定向到stdout和stderr，即减少了容器消耗的磁盘，又方便`docker logs`查看。

The official `httpd` driver changes the `httpd` application’s configuration to write its normal output directly to `/proc/self/fd/1` \(which is `STDOUT`\) and its errors to `/proc/self/fd/2` \(which is `STDERR`\). See the [Dockerfile](https://github.com/docker-library/httpd/blob/b13054c7de5c74bbaa6d595dbe38969e6d4f860c/2.2/Dockerfile#L72-L75).

Apache的Dockerfile也是类似的操作，

```text
$ cat Dockerfile
...
    && sed -ri \
        -e 's!^(\s*CustomLog)\s+\S+!\1 /proc/self/fd/1!g' \
        -e 's!^(\s*ErrorLog)\s+\S+!\1 /proc/self/fd/2!g' \
        "$HTTPD_PREFIX/conf/httpd.conf" \
...
```

参考：

[https://docs.docker.com/](https://docs.docker.com/)

