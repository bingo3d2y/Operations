# HA Harbor

**Harbor**

本篇基于Harbor 1.7.X ，目前最新版1.10.X 已略有不同，请自行转换。

**Harbor架构**

官方的架构图：

![Harbor-arch](file://D:/data_files/MarkDown/Images/Harbor-arch.png?lastModify=1584022442)

Harbor通常使用`docker-compose`进行安装部署，通过阅读`docker-compose.yaml`，我们可以看到Harbor服务由下面这些services构成：

```text
$ cat docker-compose.yml|grep image -B 1
  log:
    image: goharbor/harbor-log:v1.7.6
--
  registry:
    image: goharbor/registry-photon:v2.6.2-v1.7.6
--
  registryctl:
    image: goharbor/harbor-registryctl:v1.7.6
--
  postgresql:
    image: goharbor/harbor-db:v1.7.6
--
  adminserver:
    image: goharbor/harbor-adminserver:v1.7.6
--
  core:
    image: goharbor/harbor-core:v1.7.6
--
  portal:
    image: goharbor/harbor-portal:v1.7.6
--
  jobservice:
    image: goharbor/harbor-jobservice:v1.7.6
--
  redis:
    image: goharbor/redis-photon:v1.7.6
--
  proxy:
    image: goharbor/nginx-photon:v1.7.6
​
```

These containers are linked via DNS service discovery in Docker. By this means, each container can be accessed by their names.

借助docker内置的DNS，不同服务之间可以使用服务名来实现对容器的调用。

* log： rsyslogd 容器作为日志驱动来收集其他容器组件的日志。
* registry ： Responsible for storing Docker images and processing Docker push/pull commands.

  registry类似官方的docker registry，其配置了webhook到`Core Services`，这样当镜像的状态发生变化时，可通知到`Core Services`了。

* registryctl ：registryctl主要提供一些操纵`Registry`的API接口.
* adminserver：主要实现一些配置管理的API接口
* core： Harbor’s core functions, which mainly provides the following services: UI、Webhook and Token service

  这是个API聚合层，它是一个标准的API服务，主要完成以下功能：

  1. 监听`Registry`上镜像的变化，做相应处理，比如记录日志、发起复制等
  2. 充当Docker Authorization Service的角色，对镜像资源进行基于角色的鉴权
  3. 连接`Database`，提供存取projects、users、roles、replication policies和images元数据的API接口
  4. 提供UI界面

* portal： 一个nginx，作web server，提供静态页面访问。
* jobservice：used for image replication, local images can be replicated\(synchronized\) to other Harbor instances.

  删除镜像啦，什么之类的都属于jobservice

* proxy： 另一个Nginx，做proxy server，代理其他Harbor组件给用户提供一个统一的入口。

   For the end user, only the service port of the proxy \(Nginx\) needs to be revealed.

* postgresql（Database）： Database stores the meta data of projects, users, roles, replication policies and images.
* redis：缓存服务

**单点Harbor部署**

自行准备：

部署环境： docker 和 docker-compose

部署包：harbor-offline-installer-v1.7.6.tgz\(里面包含了全部所需镜像，内网环境下使用非常方便\)

```text
$ tree  -L 3
.
|-- LICENSE
|-- common
|   `-- templates
|       |-- adminserver
|       |-- chartserver
|       |-- clair
|       |-- core
|       |-- db
|       |-- jobservice
|       |-- log
|       |-- nginx
|       |-- notary
|       |-- registry
|       `-- registryctl
|-- docker-compose.chartmuseum.yml
|-- docker-compose.clair.yml
|-- docker-compose.notary.yml
|-- docker-compose.yml
|-- harbor.cfg
|-- harbor.v1.7.6.tar.gz
|-- install.sh
|-- open_source_license
`-- prepare
​
13 directories, 10 files
​
```

* 核心配置文件： `harbor.cfg`
* `common/`: 给目录下存放了，`prepare`基于`harbor.cfg`生成的各个组件的配置文件。
* 各组件配置文件生成工具： `prepare`
* 安装工具：`install.sh`

  主要功能：导入镜像、生成配置文件和启动服务

`install.sh` 和`prepare`都是shell脚本，有兴趣可以阅读下。

但由于`harbor.cfg`大多数参数都有默认值，所以最简单的配置就是你设置好`hostname`参数即可。

`hostname`是访问Harbor的域名入口，所以一个要填可以访问到本机的IP或者域名。

傻瓜式安装，配置好`harbor.cfg`执行`install.sh`即可

**Notary和Clair**

Notary是一套docker镜像的签名工具， 用来保证镜像在pull，push和传输工程中的一致性和完整性。避免中间人攻击，避免非法的镜像更新和运行。

clair是 coreos 开源的容器漏洞扫描工具，在容器逐渐普及的今天，容器镜像安全问题日益严重。clair 是目前少数的开源安全扫描工具，主要提供OS（centos，debian，ubuntu等）的软件包脆弱性扫描。clair既可以单机部署也可以部署到k8s上，并可以与现有的registry集成。harbor 很好的整合了 clair ，通过简单的UI就可以对上传的镜像扫描，还可以通过每天的定时扫描对所有镜像进行统一扫描.

Notary与Clair\(漏洞扫描\)相关的镜像，harbor集成了这两个服务，但默认不安装；如果需要安装，执行`/install.sh --with-notary --with-calir"`

Note: If you enable Content Trust with Notary to properly sign all images, you must use HTTPS.

**chartmuseum**

首次安装时，可以通过`./install.sh --with-chartmuseum`来直接开启这个功能。

**https**

Harbor默认不是生成证书，如果需要启用https，就需要自己生成证书或者购买证书。其他https需要如下配置：

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

**功能扩充**

如果第一次部署Harbor时，没有启用`charmuseum`或者`Notary`功能，那么后续可以通过以下两种方式进行扩展：

*  再次执行： `./install.sh --with-chartmuseum`

  经测试，这样做数据也不会丢下，如果怕有风险可以选择第二种。

* `docker-compose -f docker-compose.yml -f docker-compose.chartmusemu.yml start`

  这种方式可以增量增加服务，两个`-f`参数用来解决`docker-compose.chartmusemu.yml`的依赖问题

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

**单机配置**

最简单配置：只修改`hostname`

```text
$ cat harbor.cfg |grep -v ^# |grep -v ^$
_version = 1.7.0
## 这里需要修改成可以访问到你服务器的IP或域名。
hostname = $IP_or_Daemonname
ui_url_protocol = http
max_job_workers = 10 
customize_crt = on
##如果ui_url_protocol = https 这里要放入证书，如果是http则忽略
ssl_cert = /data/cert/server.crt
ssl_cert_key = /data/cert/server.key
secretkey_path = /data
​
admiral_url = NA
log_rotate_count = 50
log_rotate_size = 200M
http_proxy =
https_proxy =
no_proxy = 127.0.0.1,localhost,core,registry
# email，有则填，没则跳过
email_identity = 
email_server = smtp.mydomain.com
email_server_port = 25
email_username = sample_admin@mydomain.com
email_password = abc
email_from = admin <sample_admin@mydomain.com>
email_ssl = false
email_insecure = false
# Harbor登录时的默认密码
harbor_admin_password = Harbor12345
## 认证方式，默认是数据库认证，支持LDAP等其他方式，使用db_auth时，可以忽略ldap配置。
auth_mode = db_auth
ldap_url = ldaps://ldap.mydomain.com
ldap_basedn = ou=people,dc=mydomain,dc=com
ldap_uid = uid 
ldap_scope = 2 
ldap_timeout = 5
ldap_verify_cert = true
ldap_group_basedn = ou=group,dc=mydomain,dc=com
ldap_group_filter = objectclass=group
ldap_group_gid = cn
ldap_group_scope = 2
self_registration = on
token_expiration = 30
project_creation_restriction = everyone
## 其他组件的信息
db_host = postgresql
db_password = root123
db_port = 5432
db_user = postgres
redis_host = redis
redis_port = 6379
redis_password = 
redis_db_index = 1,2,3
clair_db_host = postgresql
clair_db_password = root123
clair_db_port = 5432
clair_db_username = postgres
clair_db = postgres
clair_updaters_interval = 12
uaa_endpoint = uaa.mydomain.org
uaa_clientid = id
uaa_clientsecret = secret
uaa_verify_cert = true
uaa_ca_cert = /path/to/ca.pem
## 这里可以是本机filesystem、NFS、CephFS、CephRBD、GFS都行
## HA模式下必须为一个共享存储系统
registry_storage_provider_name = filesystem
registry_storage_provider_config =
registry_custom_ca_bundle = 
​
```

傻瓜式安装，配置好`harbor.cfg`执行`install.sh`即可

```text
## 执行install.sh 直接开始安装
## install.sh = docker load + prepare + docker-compose up -d
$ ./install.sh -h
​
Note: Please set hostname and other necessary attributes in harbor.cfg first. DO NOT use localhost or 127.0.0.1 for hostname, because Harbor needs to be accessed by external clients.
Please set --with-notary if needs enable Notary in Harbor, and set ui_url_protocol/ssl_cert/ssl_cert_key in harbor.cfg bacause notary must run under https. 
Please set --with-clair if needs enable Clair in Harbor
Please set --with-chartmuseum if needs enable Chartmuseum in Harbor
​
## 生成配置文件
$ ./prepare -h     
usage: prepare [-h] [--conf CFGFILE] [--with-notary] [--with-clair]
               [--with-chartmuseum]
​
optional arguments:
  -h, --help          show this help message and exit
  --conf CFGFILE      the path of Harbor configuration file
  --with-notary       the Harbor instance is to be deployed with notary
  --with-clair        the Harbor instance is to be deployed with clair
  --with-chartmuseum  the Harbor instance is to be deployed with chart
                      repository supporting
​
```

**HA Harbor**

HA的需要启动多个应用实例/副本，来提高容错率。但是对于有状态的应用来说，在做HA的情况一定要考虑状态数据的一致性。

通过前面对Harbor各个组件的介绍，它的需要HA的组件包括：

* Redis的高可用，对外提供一个高可用的redis地址
* Database的高可用, 对外提供一个高可用的database地址
* Registry 的一组服务： proxy，portal（ui）,core，registry等等

**对外访问入口一致性**

多个Harbor实例对外提供服务时，要提供一个固定的IP来屏蔽后面的多个实例。

可以通过常见的Keepalived+Haprox/Nginx或者lvs都行--

**client获取token一致性**

client与Harbor registry交互时，需要先获取token，流程如下：

\(a\) 首先， client执行类似登录时的流程发送一个请求到registry，然后返回一个token service的URL

\(b\) 然后， client通过提供一些额外的信息与ui/token交互以获得push镜像library/hello-world的token

\(c\) 在成功获得来自Nginx转发的请求之后，Token Service查询数据库以寻找用户推送镜像的`角色`及`权限`。假如用户有相应的权限，token service就会编码相应的push操作信息，并用一个private key进行签名。然后返回一个token给Docker client

\(d\) 在client获得token之后，其就会发送一个push请求到registry，在该push请求的头中包含有上面返回的token信息。一旦registry收到了该请求，其就会使用public key来解码该token，然后再校验其内容。该public key对应于token service处的private key。假如registry发现该token有效，则会开启镜像的传输流程。

一致性由`harbor/common/config/core/private_key.pem`和`/root/harbor/common/config/registry/root.crt`两个文件保证。

Harbor的HA要保证client从不同节点获取到的token具有一致性，避免携带从Registry\_A获取的Token无法访问Registry\_B或者是出现权限不足的情况.

**registry数据一致性**

整个Harbor Cluster对外展示的镜像信息、用户信息、各类元数据要保存一致。

默认情况下，Harbor将镜像存储在本地文件系统中。在生产环境中，可以考虑使用其他存储后端而不是本地文件系统，如S3，OpenStack Swift，Ceph等。这些参数是register的配置。

> 也可下host上先挂载共享存储，然后在 Harbor中还是使用filesystem的默认参数
>
>  即： OS &lt;--mount--&gt; GFS/CephFS &lt;--filesystem--&gt; Harbor

* **registry\_storage\_provider\_name**：register的存储提供程序名称，可以是filesystem，s3，gcs，azure等。默认为filesystem。
* **registry\_storage\_provider\_config**：存储提供程序配置的逗号分隔“key：value”对，例如“key1：value，key2：value2”。默认为空字符串。
* **registry\_custom\_ca\_bundle**：自定义根ca证书的路径，它将注入到register和镜像存储库容器的信任库中。当用户使用自签名证书托管内部存储时，通常需要这样做。

**最简单的HA Harbor 部署方案：**

1. 准备harbor的离线部署包和共享存储服务（OSS/GlusterFS/CephFS）
2. 单独部署高可用的database（将Harbor初始化的数据copy到自己的db集群）和redis

   修改`harbor.cfg` 修改redis和database的`external_ip`

3. Haproxy+keepalive 或LVS + keepalive等高可用方案，对外配置Harbor vip
4. 修改harbor.cfg和docker-compose.yml 配置Harbor
5. 执行`prepare`和`docker-compose up -d` 启动服务。

核心：使用自己熟悉的Load Balancer，将DB、Redis、和Harbor http/https都配置上VIP，同时将harbor.cfg里指向的地址都改成VIP地址，就可以实现完全的HA模式。

配置如下：

环境：三台机器，1台模拟高可用的database和redis，其余两台提高Harbor Registry Services

 keepalived+haproxy提供负载VIP，这里就省略了，很简单。

```text
## database and redis 
$ vim harbor.cfg
...
hostname = $DB_IP
...
$ ./prepare
$ vim docker-compose.yml
version: '2'
services:
  log:
  ...
  postgresql:
  ...
  redis:
...
$ docker-compose up -d
​
## Harbor1 和 Harbor2挂有相同的共享存储
## Harbor 1
## 去掉docker-compose.yml中的 postgresql 和 redis service及其他服务对它们的启动依赖
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
hostname = $VIP
## 这个secretkey 很关键，一定要检查，确保这个file是个文件，
## 而且多个Harbor Registry要公用这个secret
secretkey_path = /cfs 
db_host = $DB_IP
redis_host = $DB_IP
clair_db_host = $DB_IP
​
## 生成配置文件和启动services
$ cd ~/harbor/
$ ./prepare
## 确保配置文件中IP配置正确
$ grep -inr "$vip" ./common/config
$ docker-compose up -d
​
## Harbor2 启动
# 先在Harbor1上将配置文件copy到Harbor2
$ cd ~/harbor
$ scp harbor.cfg k8s-3:/root/harbor/
## 注意有几个文件的user是10000，记得保证权限正确
##  -P     Preserves modification times, access times, and modes from the original file.
## 在scp中是-P选项
$ scp -p -r common/ k8s-3:/root/harbor/
$ scp docker-compose.yml k8s-3:/root/harbor/
## 启动Harbor 2
$ docker-compose up -d
```

生产环境，建议db和redis使用更稳定的库。

**HA Harbor key不一致问题**

ha 一定要注意避免key不一致，导致Harbor 操作时401错误\(eg: 界面上查看image tag 提示错误\)

要注意部署\(执行`prepare`\)过程中会生成一个ui、adminserver和jobservice共用的secretkey，默认保存在/data/secretkey。部署多个节点时，要保证各个节点使用相同的secretkey。

docker向registry发起请求，由于registry是基于token auth的，因此registry回复应答，告诉docker client去哪个URL去获取token； docker client根据应答中的URL向token service\(ui\)发起请求，通过user和passwd获取token；如果user和passwd在db中通过了验证，那么token service将用自己的私钥\(harbor/common/config/core/private\_key.pem\)生成一个token，返回给docker client端； docker client获得token后再向registry发起login请求，registry用自己的证书\(harbor/common/config/registry/root.crt\)对token进行校验。通过则返回成功，否则返回失败。

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
## 还有一个证书是：
$ ls /root/harbor/common/config/registry/root.crt
```

**Harbor GC**

Repositories删除时，有两个步骤：

首先，在Harbor的UI上删除一个repository。这只是一个软删除\(soft deletion\)。你可以删除整个repository，或者只是该repository的一个标签。在软删除之后，该repository将不再由Harbor进行管理，然而，该repository的文件还仍然存放在Harbor的存储系统中（即后端存储中文件不会消失，存储空间不会释放）。

registry的GC使用“标记-清理”法。

* 第一步，标记。registry扫描元数据，元数据能够索引到的blob标记为**不能删除**。
* 第二步，清理。registry扫描所有blobs，如果改blob没有被标记，则删除它。

实际执行：

```text
## 先查看registry volume的目录
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
# 这个回去遍历: /data/registry/docker/registry/v2/repositories/project_name ...
# 然后，标记--删除
$ docker run -it --name gc --rm --volumes-from registry vmware/registry:2.6.2-photon garbage-collect --dry-run /etc/registry/config.yml
​
```

end

鉴权失败： [https://tonybai.com/2017/06/15/fix-auth-fail-when-login-harbor-registry/](https://tonybai.com/2017/06/15/fix-auth-fail-when-login-harbor-registry/)

**通过https形式访问Harbor**

* 通过浏览器访问

  这里首先需要将上面产生的`ca.crt`导入到浏览器的`受信任的根证书`中。然后就可以通过https进行访问（这里经过测试，Chrome浏览器、IE浏览器可以正常访问，但360浏览器不能正常访问）

* 通过docker命令来访问

  首先新建`/etc/docker/certs.d/192.168.69.128`目录，然后将上面产生的`ca.crt`拷贝到该目录:

  Harbor使用https的好处就是避免docker daemon的重启。

  ```text
  $ mkdir -p /etc/docker/certs.d/192.168.69.128
  $ cp ca.crt /etc/docker/certs.d/192.168.69.128/
  ```

  然后登录到docker registry:

  ```text
  # docker login 192.168.69.128
  Username (admin): admin
  Password: 
  Login Succeeded
  ```

* curl令来访问Harbor中有哪些镜像

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

**Harbor权限划分**

harbor的角色很少就三个：

Project Admin 和 Developer 可以有 push 的权限，而 Guest 只能查看和 pull

创建用户，绑定到项目，然后分配Role即可完成权限的分配

在Harbor中projects有两种类型：

* `Public`: 所有的用户都具有读取public project的权限， 其主要是为了方便你分享一些repositories

  不需要`docker login` 也能pull --

* `Private`: 一个私有project只能被特定用户以适当的权限进行访问

**Harbor 1.10.x**

变化挺大的，配置文件变成`harbor.yaml`,而且配置信息也变得明显了，分为可选和必选参数。

参考：

[https://cloud.tencent.com/developer/article/1402496](https://cloud.tencent.com/developer/article/1402496)

[https://www.a-programmer.top/2019/03/26/Habor%E5%88%9D%E5%8D%B0%E8%B1%A1/](https://www.a-programmer.top/2019/03/26/Habor%E5%88%9D%E5%8D%B0%E8%B1%A1/)

