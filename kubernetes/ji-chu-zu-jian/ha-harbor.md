# HA Harbor

**Harbor**

\*\*\*\*

```text
    0.0 再看下配置文件验证下--
    #  这写个tag 就是日志的名字比如： registry.log
    # 由syslogd接收并处理这个tags字段。
```

PS： `prepare`会根据`harbor.cfg`生成`harbor/common/config`下的配置文件。

但是，用户会根据自己的意愿再修改`harbor/common/config`下的配置，所以。确定最终配置时要看`harbor/common/config`下的配置文件，而不是`harbor.cfg`

要注意部署过程中会生成一个ui、adminserver和jobservice共用的secretkey，默认保存在/data/secretkey。部署多个节点时，要保证各个节点使用相同的secretkey。

使用自己熟悉的Load Balancer，将MariaDB、Redis、Ceph Radosgw和Harbor http都配置上VIP，同时将harbor.cfg里指向的地址都改成VIP地址，就可以实现完全的HA模式。

 [https://github.com/goharbor/harbor/wiki/Architecture-Overview-of-Harbor](https://github.com/goharbor/harbor/wiki/Architecture-Overview-of-Harbor)

**端口**

| Port | 协议 | 描述 |
| :--- | :--- | :--- |
| 443 | HTTPS | Harbor入口和核心API将接受此端口上的https协议请求 |
| 4443 | HTTPS | 只有在启用“Notary”时才需要连接到Dock的Docker Content Trust服务 |
| 80 | HTTP | Harbor端口和核心API将接受此端口上的http协议请求 |

**配置存储后端**

> 0.0 可以直接接存储耶，不需要OS先挂载cephfs、
>
> --我使用了， OS &lt;--mount--&gt; CephFS &lt;-filesystem-&gt; Harbor

默认情况下，Harbor将镜像存储在本地文件系统中。在生产环境中，可以考虑使用其他存储后端而不是本地文件系统，如S3，OpenStack Swift，Ceph等。这些参数是register的配置。

* **registry\_storage\_provider\_name**：register的存储提供程序名称，可以是filesystem，s3，gcs，azure等。默认为filesystem。
* **registry\_storage\_provider\_config**：存储提供程序配置的逗号分隔“key：value”对，例如“key1：value，key2：value2”。默认为空字符串。
* **registry\_custom\_ca\_bundle**：自定义根ca证书的路径，它将注入到register和镜像存储库容器的信任库中。当用户使用自签名证书托管内部存储时，通常需要这样做。

> 给一个ceph rbd的配置
>
> ```text
> storage:
>  cache:
>      layerinfo: inmemory
>  swift:
>      authurl: http://21.48.9.1:35357/v2.0
>      username: yfwzx
>      password: passw0rd
>      tenant: yfwzx
>      container: harbor
>  maintenance:
>      uploadpurging:
>          enabled: false
>  delete:
>      enabled: true
> ```
>
> ...
>
> 0.0 有点麻烦，不如挂文件系统
>
> [https://mojo-zd.github.io/2018/04/12/ceph%E5%AF%B9%E8%B1%A1%E5%AD%98%E5%82%A8%E4%B8%BAHarbor%E6%8F%90%E4%BE%9B%E5%AD%98%E5%82%A8%E9%A9%B1%E5%8A%A8/](https://mojo-zd.github.io/2018/04/12/ceph%E5%AF%B9%E8%B1%A1%E5%AD%98%E5%82%A8%E4%B8%BAHarbor%E6%8F%90%E4%BE%9B%E5%AD%98%E5%82%A8%E9%A9%B1%E5%8A%A8/)

**HA**

 ha 一定要注意避免key不一致，导致Harbor 操作时401错误\(eg: 界面上查看image tag 提示错误\)

**docker向registry发起请求，由于registry是基于token auth的，因此registry回复应答，告诉docker client去哪个URL去获取token；** **docker client根据应答中的URL向token service\(ui\)发起请求，通过user和passwd获取token；如果user和passwd在db中通过了验证，那么token service将用自己的私钥\(harbor/common/config/core/private\_key.pem\)生成一个token，返回给docker client端；** **docker client获得token后再向registry发起login请求，registry用自己的证书\(harbor/common/config/registry/root.crt\)对token进行校验。通过则返回成功，否则返回失败。**

因为，harbor的认证分为两部分：

client 先请求Registry，然后Registry告诉client你去token server获取token，然后client再带着token去访问registry。

* docker向registry发起请求，由于registry是基于token auth的，因此registry回复应答，告诉docker client去哪个URL去获取token；
* docker client根据应答中的URL向token service\(ui\)发起请求，通过user和passwd获取token；如果user和passwd在db中通过了验证，那么token service将用自己的私钥\(harbor/common/config/ui/private\_key.pem\)生成一个token，返回给docker client端；

  > 这里，如果两个Harbor token service 不一样，则会导致，Harbor\_A利用自己的certificate签发了token\_A
  >
  > 然后client再带着token\_A去访问Harbor\_VIP，这次被路由到Harbor\_B了就会导致，token\_A操作Harbor\_B出现401错误提示。

* docker client获得token后再向registry发起login请求，registry用自己的证书\(harbor/common/config/registry/root.crt\)对token进行校验。通过则返回成功，否则返回失败。

```text
## 这个例子就是private_key.pem  不一致导致，401错误
[root@cs1-harbor-3 ~]# md5sum  /root/harbor/common/config/core/private_key.pem 
ce28eac89f7a98243dc05e0438985489  /root/harbor/common/config/core/private_key.pem
[root@cs1-harbor-2 harbor]# md5sum  /root/harbor/common/config/core/private_key.pem 
1dad0ffcac66b534547a682298d659c2  /root/harbor/common/config/core/private_key.pem
​
Dec 16 15:08:28 172.21.0.1 registry[10974]: time="2019-12-16T07:08:28.160073285Z" level=info msg="token signed by untrusted key with ID: \"ZI34:FHLC:U4ZF:XTKV:ZT2I:226B:OODS:HRWG:44WU:HTST:
NO3Y:AEVB\"" Dec 16 15:08:28 172.21.0.1 registry[10974]: time="2019-12-16T07:08:28.160168466Z" level=warning msg="error authorizing context: invalid token" go.versio
​
​
## 还有一个正式就是：
$ ls /root/harbor/common/config/registry/root.crt
```

end

HA Harbor:

基础知识：Harbor各个组件的作用，有`docker-compose.yml`可以看到其service有：

These containers are linked via DNS service discovery in Docker. By this means, each container can be accessed by their names.

* log： Container that runs rsyslogd, used for collecting logs from other containers through the log-driver mode.
* registry ： Responsible for storing Docker images and processing Docker push/pull commands.
* registryctl ：
* adminserver：
* core： Harbor’s core functions, which mainly provides the following services:

  UI: a graphical user interface to help users manage images on the Registry

  Webhook: Webhook is a mechanism configured in the Registry so that image status changes in the Registry can be populated to the Webhook endpoint of Harbor. Harbor uses webhook to update logs, initiate replications, and some other functions.

  Token service: Responsible for issuing a token for every docker push/pull command according to a user’s role of a project. If there is no token in a request sent from a Docker client, the Registry will redirect the request to the token service.

  Database: Database stores the meta data of projects, users, roles, replication policies and images.

  0.0 Harbor 不能用了先看这个core.log的日志

* portal： 一个nginx，作web server，提供静态页面访问。
* jobservice：used for image replication, local images can be replicated\(synchronized\) to other Harbor instances.

  0.0 删除镜像啦，什么之类的都属于jobservice

* proxy： 另一个Nginx，做proxy server，代理其他Harbor组件给用户提供一个统一的入口。

   For the end user, only the service port of the proxy \(Nginx\) needs to be revealed.

* postgresql： auth\_db
* redis：0.0 缓存

所以：HA Harbor分为四部分

1. Redis的高可用，对外提供一个高可用的redis地址
2. Database的高可用, 对外提供一个高可用的database地址
3. Registry 的一组服务： proxy，portal（ui）,core，registry等等
4. log

最简单的HA Harbor 部署方案：

> 掌握这几点就行了：
>
> ① prepare 根据harbor.cfg生成配置文件在./common/config目录下
>
> ②docker-compose 启动service 会挂载配置config/下的配置文件
>
> ③多个Registry，一定要保证secretkey相同。
>
> ④两nginx，一个做web server 一个做proxy server
>
> ⑤redis和database的高可用记得保障

1. 准备harbor的离线部署包和共享存储服务（用GlusterFS，CephFS奇慢无比）
2. 单独部署高可用的database（将Harbor初始化的数据copy到自己的db集群）和redis

   修改`harbor.cfg` 修改redis和database的`external_ip`

3. Haproxy+keepalive 或LVS + keepalive等高可用方案，对外配置Harbor vip
4. 修改harbor.cfg和docker-compose.yml 配置Harbor
5. 执行`prepare`和`docker-compose up -d` 启动服务。

配置如下：

```text
#HaProxy
docker run -d --net=host -v /root/harbor/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro 21.49.22.250/kubeadm/haproxy:1.9
$ cat /root/harbor/haproxy.cfg
global
  log 127.0.0.1 local0 
  log 127.0.0.1 local1 notice
  tune.ssl.default-dh-param 2048
​
defaults
  log global
  mode http 
  option dontlognull
  timeout connect 5000ms
  timeout client  600000ms
  timeout server  600000ms
​
listen stats
    bind :9090
    mode http
    balance
    stats uri /haproxy_stats
    stats auth admin:admin123
    stats admin if TRUE
​
frontend harbor
   mode tcp
   bind :80
   default_backend harbor
​
backend harbor
    mode tcp
    balance roundrobin
    stick-table type ip size 200k expire 30m
    stick on src
    server apiserver2 21.48.5.83:80 check
    server apiserver2 21.48.5.84:80 check
​
## database and redis
$ vim harbor.cfg
...
hostname = 21.48.5.82
...
​
$ vim docker-compose.yml
version: '2'
services:
  log:
  ...
  postgresql:
  ...
  redis:
...
​
## Harbor 1
## 去掉docker-compose.yml中的 postgresql 和 redis service及其他服务的dependence
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
        tag: "registry"
  registryctl:
    image: goharbor/harbor-registryctl:v1.7.1
    container_name: registryctl
    env_file:
      - ./common/config/registryctl/env
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
      - ./common/config/registryctl/config.yml:/etc/registryctl/config.yml:z
    networks:
      - harbor
    dns_search: .
    depends_on:
      - log
    logging:
      driver: "syslog"
      options:  
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "registryctl"
  adminserver:
    image: goharbor/harbor-adminserver:v1.7.1
    container_name: harbor-adminserver
    env_file:
      - ./common/config/adminserver/env
    restart: always
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
    volumes:
      - /cfs/config/:/etc/adminserver/config/:z
      - /cfs/secretkey:/etc/adminserver/key:z
      - /cfs/:/data/:z
    networks:
      - harbor
    dns_search: .
    depends_on:
      - log
    logging:
      driver: "syslog"
      options:  
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "adminserver"
  core:
    image: goharbor/harbor-core:v1.7.1
    container_name: harbor-core
    env_file:
      - ./common/config/core/env
    restart: always
    cap_drop:
      - ALL
    cap_add:
      - SETGID
      - SETUID
    volumes:
      - ./common/config/core/app.conf:/etc/core/app.conf:z
      - ./common/config/core/private_key.pem:/etc/core/private_key.pem:z
      - ./common/config/core/certificates/:/etc/core/certificates/:z
      - /cfs/secretkey:/etc/core/key:z
      - /cfs/ca_download/:/etc/core/ca/:z
      - /cfs/psc/:/etc/core/token/:z
      - /cfs/:/data/:z
    networks:
      - harbor
    dns_search: .
    depends_on:
      - log
      - adminserver
      - registry
    logging:
      driver: "syslog"
      options:  
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "core"
  portal:
    image: goharbor/harbor-portal:v1.7.1
    container_name: harbor-portal
    restart: always
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
      - NET_BIND_SERVICE
    networks:
      - harbor
    dns_search: .
    depends_on:
      - log
      - core
    logging:
      driver: "syslog"
      options:
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "portal"
​
  jobservice:
    image: goharbor/harbor-jobservice:v1.7.1
    container_name: harbor-jobservice
    env_file:
      - ./common/config/jobservice/env
    restart: always
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
    volumes:
      - /cfs/job_logs:/var/log/jobs:z
      - ./common/config/jobservice/config.yml:/etc/jobservice/config.yml:z
    networks:
      - harbor
    dns_search: .
    depends_on:
      - core
      - adminserver
    logging:
      driver: "syslog"
      options:
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "jobservice"
  proxy:
    image: goharbor/nginx-photon:v1.7.1
    container_name: nginx
    restart: always
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
      - NET_BIND_SERVICE
    volumes:
      - ./common/config/nginx:/etc/nginx:z
    networks:
      - harbor
    dns_search: .
    ports:
      - 80:80
      - 443:443
      - 4443:4443
    depends_on:
      - registry
      - core
      - portal
      - log
    logging:
      driver: "syslog"
      options:  
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "proxy"
networks:
  harbor:
    external: false
​
##修改这些配置     
$ cat harbor.cfg
## 域名或vip
hostname = 21.48.5.82
## 这个secretkey 很关键，一定要检查，确保这个file是个文件，
## 而且多个Harbor Registry要公用这个secret
secretkey_path = /cfs 
db_host = 21.48.5.82
redis_host = 21.48.5.82
clair_db_host = 21.48.5.82
​
## 生成配置文件和启动services
$ cd ~/harbor/
## 回去读harbor.cfg
$ ./prepare
## 确保配置文件中IP配置正确
$ grep -inr '21.48.5.' ./common/config
$ docker-compose up -d
​
## Harbor2 启动
# 先在Harbor1上将配置文件cp到Harbor2
$ cd ~/harbor
$ scp harbor.cfg k8s-3:/root/harbor/
## 注意有几个文件的user是10000，记得找到改回来。
##  -p      Preserves modification times, access times, and modes from the original file.
## ??? -p 不生效？？？
$ scp -p -r common/ k8s-3:/root/harbor/
$ scp docker-compose.yml k8s-3:/root/harbor/
## 启动Harbor 2
$ docker-compose up -d
​
```

end

HTTPS配置

```text
hostname = yourdomain.com:port
ui_url_protocol = https
ssl_cert = /data/cert/server.crt
ssl_cert_key = /data/cert/server.key
​
## 改了harbor.cfg 再生成下配置文件
$ ./prepare 
然后docker-compose down 清掉之前的容器 不要紧的数据不会删了
```

end

需要在配置文件中设置这些参数。如果用户更新它们harbor.cfg并运行install.sh脚本以重新安装Harbor，它们将生效。

* hostname：目标主机的主机名，用于访问UI和注册服务。必须为目标计算机的IP地址或完全限定的域名（FQDN），例如192.168.1.10或reg.yourdomain.com。不可使用localhost或127.0.0.1作为主机名。

  可以理解为_Harbor 服务器域名_

* secretkey\_path：用于加密或解密复制策略中远程注册表密码的密钥路径。

  这个很重要，多个Registry时，要共有一个secretkey.

* ui\_url\_protocol：\(http或https，默认为http）用于访问UI和令牌/通知服务的协议。如果启用了认证，则此参数必须为https。
* db\_password：用于db\_auth的MySQL数据库的root密码。
* max\_job\_workers：\(默认值为3）作业服务中的最大复制工作数。对于每个映像复制作业，工作程序将存储库的所有标记同步到远程目标。增加此数量可以在系统中实现更多并发复制作业。但是，由于每个工作者都消耗一定量的网络/CPU/IO资源，请根据主机的硬件资源仔细选择该属性的值。
* customize\_crt：（开启或关闭，默认为开启），如果此属性开启，在准备脚本创建注册表的令牌生成/验证私钥和根证书。当外部源提供密钥和根证书时，将此属性设置为off。
* ssl\_cert：SSL证书的路径，仅在协议设置为https时应用。
* ssl\_cert\_key：SSL密钥的路径，仅在协议设置为https时应用。
* log\_rotate\_count：日志文件在被删除之前会被轮询log\_rotate\_count次数。如果count为0，则删除旧版本而不是轮询。
* log\_rotate\_size：仅当日志文件大于log\_rotate\_size字节时才会轮换日志文件。如果大小后跟k，则大小以千字节为单位。如果使用M，则大小以兆字节为单位，如果使用G，则大小为千兆字节。

**chartmuseum**

首次安装时，可以通过`./install.sh --with-chartmuseum`来直接开启这个功能。

我测了下已部署的环境进行`./install.sh --with-chartmuseum` 数据也不会丢失的

唯一的收获：看了下`install.sh`

知道了怎么`docker-compose -f docker-compose.yml -f docker-compose.chartmusemu.yml start`

可以解决`docker-compose.chartmusemu.yml`的依赖问题

因为单独启动`docker-compose.chartmusemu.yml`会报错：service core has neither an image nor a build context

```text
## 因为core 和 redis的定义在  docker-compose.yml 里面，所以单独执行会报错。
$ cat docker-compose.chartmusemu.yml
version: '2'
services:
  core:
    networks:
      harbor-chartmuseum:
        aliases:
          - harbor-core
  redis:
    networks:
      harbor-chartmuseum:
        aliases:
          - redis
  chartmuseum:
    ...
```

**clair**

Notary与Clair\(漏洞扫描\)相关的镜像，harbor集成了这两个服务，但默认不安装；如果需要安装，执行" ./install.sh --with-notary --with-calir"

**GC**

Repositories删除时，有两个步骤：

首先，在Harbor的UI上删除一个repository。这只是一个软删除\(soft deletion\)。你可以删除整个repository，或者只是该repository的一个标签。在软删除之后，该repository将不再由Harbor进行管理，然而，该repository的文件还仍然存放在Harbor的存储系统中（即后端存储中文件不会消失，存储空间不会释放）。

registry的GC使用“标记-清理”法。

* 第一步，标记。registry扫描元数据，元数据能够索引到的blob标记为**不能删除**。
* 第二步，清理。registry扫描所有blobs，如果改blob没有被标记，则删除它。

_跟JVM老年代的GC是不是很像？_

实际执行：

```text
$ cat docker-compose.yml
version: '2'
services:
...
  registryctl:
    image: goharbor/harbor-registryctl:v1.7.1
...
    volumes:
      - /data/registry:/storage:z
...
​
$ docker run -it --name gc --rm --volumes-from registry vmware/registry:2.6.2-photon garbage-collect --dry-run /etc/registry/config.yml
这个回去遍历: /data/registry/docker/registry/v2/repositories/project_name ...
然后，标记--删除
0.0 
$ docker run -it --name gc --rm --volumes-from registry vmware/registry:2.6.2-photon garbage-collect  /etc/registry/config.yml
```

end

鉴权失败： [https://tonybai.com/2017/06/15/fix-auth-fail-when-login-harbor-registry/](https://tonybai.com/2017/06/15/fix-auth-fail-when-login-harbor-registry/)

 通过https形式访问Harbor

* 通过浏览器访问

这里首先需要将上面产生的`ca.crt`导入到浏览器的`受信任的根证书`中。然后就可以通过https进行访问（这里经过测试，Chrome浏览器、IE浏览器可以正常访问，但360浏览器不能正常访问）

* 通过docker命令来访问

首先新建`/etc/docker/certs.d/192.168.69.128`目录，然后将上面产生的`ca.crt`拷贝到该目录:

```text
# mkdir -p /etc/docker/certs.d/192.168.69.128
# cp ca.crt /etc/docker/certs.d/192.168.69.128/
```

然后登录到docker registry:

```text
# docker login 192.168.69.128
Username (admin): admin
Password: 
Login Succeeded
```

用向`registry`中上传一个镜像：

```text
# docker images
192.168.69.128/library/redis   alpine              c27f56585938        3 weeks ago         27.7MB
​
[root@localhost test]# docker push 192.168.69.128/library/redis:alpine
The push refers to repository [192.168.69.128/library/redis]
f6b9463783dc: Pushed 
222a85888a99: Pushed 
1925395eabdd: Pushed 
c3d278563734: Pushed 
ad9247fe8c63: Pushed 
cd7100a72410: Pushed 
alpine: digest: sha256:9d017f829df3d0800f2a2582c710143767f6dda4df584b708260e73b1a1b6db3 size: 1568
```

* 通过curl命令来访问 registry API版本号

查询registry API版本号：

```text
# curl -iL -X GET https://192.168.69.128/v2 --cacert ca.crt
HTTP/1.1 301 Moved Permanently
Server: nginx
Date: Tue, 10 Apr 2018 09:33:39 GMT
Content-Type: text/html
Content-Length: 178
Location: https://192.168.69.128/v2/
Connection: keep-alive
​
HTTP/1.1 401 Unauthorized
Server: nginx
Date: Tue, 10 Apr 2018 09:33:39 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 87
Connection: keep-alive
Docker-Distribution-Api-Version: registry/2.0
Set-Cookie: beegosessionID=575f32ac760f52c8cf1cdb748e48ab5e; Path=/; HttpOnly
Www-Authenticate: Bearer realm="https://192.168.69.128/service/token",service="harbor-registry"
​
{"errors":[{"code":"UNAUTHORIZED","message":"authentication required","detail":null}]}
​
# curl -iL -X GET -u admin:Harbor12345 https://192.168.69.128/service/token?account=admin\&service=harbor-registry --cacert ca.crt
HTTP/1.1 200 OK
Server: nginx
Date: Tue, 10 Apr 2018 09:34:39 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 1100
Connection: keep-alive
Set-Cookie: beegosessionID=77bc62dcdc4a810a0e208487a89f069a; Path=/; HttpOnly
​
{
  "token": "nHLZqMPw",
  "expires_in": 1800,
  "issued_at": "2018-04-10T09:34:39Z"
}
​
# curl -iL -X GET -H "Content-Type: application/json" -H "Authorization: Bearer nHLZqMPw" https://192.168.69.128/v2 --cacert ca.crt
HTTP/1.1 301 Moved Permanently
Server: nginx
Date: Tue, 10 Apr 2018 09:36:48 GMT
Content-Type: text/html
Content-Length: 178
Location: https://192.168.69.128/v2/
Connection: keep-alive
​
HTTP/1.1 200 OK
Server: nginx
Date: Tue, 10 Apr 2018 09:36:48 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 2
Connection: keep-alive
Docker-Distribution-Api-Version: registry/2.0
Set-Cookie: beegosessionID=e651b65d891617a999254ec875c1c63c; Path=/; HttpOnly
```

上面为了显示，我们对返回过来的`token`做了适当的裁剪。此外这里`curl`命令不适用`-k`选项，表示需要对服务器证书进行检查。

* 通过curl来访问registry中的镜像列表

```text
# curl -iL -X GET -u admin:Harbor12345 https://192.168.69.128/service/token?account=admin\&service=harbor-registry\&scope=registry:catalog:* --cacert ca.crt
HTTP/1.1 200 OK
Server: nginx
Date: Mon, 09 Apr 2018 09:33:52 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 1166
Connection: keep-alive
Set-Cookie: beegosessionID=648fd5a5ec4f06389d45c02f7f5971b4; Path=/; HttpOnly
​
{
  "token": "A7yfEdUBYD3bDhLM",
  "expires_in": 1800,
  "issued_at": "2018-04-09T09:33:52Z"
}
​
# curl -iL -X GET -H "Content-Type: application/json" -H "Authorization: Bearer LA7yfEdUBYD3bDhLM" http://192.168.69.128/v2/_catalog --cacert ca.crt
HTTP/1.1 200 OK
Server: nginx
Date: Mon, 09 Apr 2018 09:36:35 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 34
Connection: keep-alive
Docker-Distribution-Api-Version: registry/2.0
Set-Cookie: beegosessionID=1b84e760ab0234045f06680e56e28818; Path=/; HttpOnly
​
{"repositories":["library/redis"]}
```

上面为了显示，我们对返回过来的`token`做了适当的裁剪。此外这里`curl`命令不适用`-k`选项，表示需要对服务器证书进行检查。

end

**拉取私有仓库镜像问题**

Rancher 公共项目，不需`docker login` 就可以拉取镜像；私有项目则需要`docker login`，才能运行`docker pull`.but，kubelet 拉取镜像时不走docker 的api，so，还需也在yaml文件中通过secret机制来加载Harbor的登录凭证。

[https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)

**Harbor的RBAC**

harbor的角色很少就三个：

**Project Admin 和 Developer 可以有 push 的权限，而 Guest 只能查看和 pull**

创建用户，绑定到项目，然后分配Role即可完成权限的分配

在Harbor中projects有两种类型：

* `Public`: 所有的用户都具有读取public project的权限， 其主要是为了方便你分享一些repositories

  不需要`docker login` 也能pull --

* `Private`: 一个私有project只能被特定用户以适当的权限进行访问

0.0

[https://ivanzz1001.github.io/records/post/docker/2018/04/11/docker-harbor-uage\#3-harbor%E5%B7%A5%E7%A8%8B%E7%AE%A1%E7%90%86](https://ivanzz1001.github.io/records/post/docker/2018/04/11/docker-harbor-uage#3-harbor%E5%B7%A5%E7%A8%8B%E7%AE%A1%E7%90%86)

**查看Harbor Catalog**

0.0 可以查看Harbor中有哪些镜像

```text
#!/bin/bash
if [ $# -ne 1 ]
then
   echo "args number is : $#"
   echo "please input image name"
   exit 1
fi
token=$(curl  -ksi -L -u admin:Harbor12345 https://21.49.22.250/service/token?account=admin\&service=harbor-registry\&scop
e=registry:catalog:\*|grep token|gawk -F "\"" '{print $4}')curl  -sk -H "Content-Type: application/json" -H "Authorization: Bearer $token"  -X GET https://21.49.22.250/v2/_catalog|p
ython -m "json.tool"|grep $1
```

end

