# k8s平台搭建（仅上传了第一阶段其他组件比较乱还未整理）

**优化基础思路**

0.0 核心： 减少内核态和用户态的切换

任何一个分布式的系统，当集群规模上去后都需要考虑如下问题：

* 分布式系统内部API接口压力
* 选举问题
* HA
* 磁盘
* end

## 第一阶段

**Prerequisites**

* 内核版本升级到4.x，3.10有runc bug
* 关闭防火墙firewalld（2379等端口通信），hosts配上解析，ntp时间同步\(etcd cluster healthz\)及Selinux

  ```text
  systemctl stop firewalld 
  systemctl disable firewalld
  setenforce 0
  sed -i "s/^SELINUX=.*/SELINUX=disabled/" /etc/selinux/config
  ```

  end

* ntpd 同步时间和设置开机自动同步时间

  0.0 `ntpd -u ntp`

* 日志限制
  * systemd-journald 日志

    ```text
    vim /etc/systemd/journald.conf
    ​
    SystemMaxFileSize=12M
    SystemMaxFiles=10
    ​
    SystemMaxFileSize * SystemMaxFiles = SystemMaxUse
    ```

  * systemd-journald日志持久化

    systemd-journald 服务收集到的日志默认保存在 /run/log 目录中，重启系统会丢掉以前的日志信息。 我们可以通过两种方式让 systemd-journald 服务把所有的日志都保存到文件中，这样重新启动后就不会丢掉以前的日志。   
    方法一：创建目录 /var/log/journal，然后重启日志服务 systemd-journald.service。   
    方法二：修改配置文件 /etc/systemd/journald.conf，把 Storage=auto 改为 Storage=persistent，并取消注释，然后重启日志服务 systemd-journald.service。  
    方法一详细操作：  
  
    `$ sudo mkdir /var/log/journal  
    $ sudo chown root:systemd-journal /var/log/journal   
    $ sudo chmod 2775 /var/log/journal   
    $ sudo systemctl restart systemd-journald.service`  
* 开机自动挂载cephfs

  事说三，再网卡做Bond的环境下，不能将挂载分布式存储或网络存储写在`/etc/fstab`中

  因为开机时，bond还没做好，所以网络不通，所以导致cephfs挂载失败，最后导致开机失败....

  0\_0

* docker、etcd、calicoctl、kubelet和kubeadm 二进制文件下载与分发
* os配置
  * 集群node规格

    0.0 不要配置太高，每个机器上运行太多的Pod，万一那个进程影响到os kernel，全部挂掉。

  * 内核模块加载

    ```text
    ## 这几个模块写到/ect/rc.local 里开机加载，这觉得了是否能够用ipvs模式、
    $ modprobe -- ip_vs
    $ modprobe -- ip_vs_rr
    $ modprobe -- ip_vs_wrr
    $ modprobe -- ip_vs_sh
    $ modprobe br-netfilter
    $ modprobe -- nf_conntrack_ipv4
    ​
    ## 保证ipvs
    $ echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
    ​
    ```

  * 内核参数优化

    ```text
    ### sysctl - configure kernel parameters at runtime
    ###  
    $ sysctl -w fs.pipe-max-size=16777216
    $ sysctl -w net.ipv4.ip_forward=1
    $ echo 'fs.pipe-max-size=16777216' >>  /etc/sysctl.conf
    $ echo 'net.ipv4.ip_forward=1' >> /etc/sysctl.conf
    ​
    ```

  * hosts

    ```text
    $ cat  << EOF >> /etc/hosts
    172.29.33.1  cs1-k8s-m1
    172.29.33.2  cs1-k8s-m2
    172.29.33.3  cs1-k8s-m3
    EOF
    这个很重要，因为kubeadm签发证书时，很多用的主机名做签名，啧啧要是主机名不能被解析，会认证失败喔

    ```

  * reserved

    ssh 互信
* 工具脚本

  ```text
  ## scp file
  $ cat s2.sh
  for i in `seq 2 3`; do echo k8s-$i; echo "scp {$1}"; scp -r $1 k8s-$1:$1;done
  ​
  ## ssh exec command
  $ cat s1.sh
  for i in `seq 2 3`; do echo k8s-$i; echo "exec {$@}"; ssh  k8s-$1 "$@";done

  ## 批量执行带 '|' 的命令
  $ s1.sh "docker ps |grep haproxy"
  ```

* `/var/lib/docker` 挂载目录使用xfs 文件系统

  当使用`overlay2` 时，如果不使用xfs就无法做到给每个容器限制10G的大小，就可能出现一个容器的误操作导致把机器盘全占完我们使用了lvm去弄个分区出来做xfs文件系统，当然你也可以不用lvm

  ```text
  # cat /etc/docker/daemon.json
  {
    "storage-opts": [
      "overlay2.override_kernel_check=true",
      "overlay2.size=10G"
    ],
    "log-driver": "json-file",
    "log-opts": {
      "max-size": "10m"
    }
  }
  ```

* reserved



**docker && container**

shoot，containerd服务也要Unit file的

```text
#!/bin/bash
command='''
cat << EOF > tee /etc/docker/daemon.json 
{
  ## 这个storage-driver要明确指定 不然，docker info 中 Native Overlay Diff 显示false
  "storage-driver": "overlay2",
  ## 千万别开启这个参数，真的难受
  #"live-restore": true,
  ## 这个一定要设置 Defaults to -1 (unlimited) 
  ## 限制每个container的stdout logs即 Docker_ROOT/containers/xxx/xxx.json
  "log-opts": {
    "max-size": "20m",
    "max-file": "10"
  },
  "storage-opts": [
    "overlay2.override_kernel_check=true",
    "overlay2.size=10G"
  ],
  "insecure-registries": ["21.49.22.250"]
}
EOF
​
cat << EOF > /etc/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
BindsTo=containerd.service
After=network-online.target  firewalld.service
Wants=network-online.target
​
[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
#ExecStart=/usr/bin/dockerd -H fd://  
ExecStart=/usr/bin/dockerd -H unix://
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=1048576
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
# Uncomment TasksMax if your systemd version supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process
# restart the docker process if it exits prematurely
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s
​
[Install] 
WantedBy=multi-user.target
EOF
systemctl enable docker
systemctl start docker
docker info
'''
​
for i in `seq 1 6` ;do echo 172.29.33.$i; ssh 172.29.33.$i  "$command";done
​
--- 一定要搞上去 --
$ containerd.service  
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target
​
[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd
​
Delegate=yes
KillMode=process
Restart=always
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=1048576
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
​
[Install]
WantedBy=multi-user.target
​
​
```

overlay2 + xfs 

官方文档： [https://docs.docker.com/engine/reference/commandline/dockerd/](https://docs.docker.com/engine/reference/commandline/dockerd/) 

Sets the default max size of the container. It is supported only when the backing fs is `xfs` and mounted with `pquota` mount option. Under these conditions the user can pass any size less then the backing fs size.

> 这些quota 是xfs带的特性，`pquota --> Project quota`
>
> **pquota\|prjquota**: Enable project quotas and enforce usage limits.

> `mount -o pquota disk mount_point` and `disk m_point xfs defaults,pquota 0 0 (in fstab)`

1. 给`Docker Root Dir: /var/lib/docker` 挂载单独的硬盘，并格式成`xfs` 文件系统。

   因为`overlay2`的磁盘配额只支持`xfs`文件系统。

   > It is supported only when the backing fs is xfs and mounted with pquota mount option.

2. 创建目录和配置文件

   ```text
   $ mkdir /etc/docker/
   ## custom docker daemon
   ## insecure-registries,添加私有仓库地址
   $ cat << EOF > /etc/docker/daemon.json 
   {
     "storage-driver": "overlay2",
     "live-restore": true,
     "log-opts": {
       "max-size": "20m",
       "max-file": "10"
     },
     "storage-opts": [
       "overlay2.override_kernel_check=true",
       #  It is supported only when the backing fs is xfs and mounted with pquota mount option.
       "overlay2.size=10G"
     ],
     "insecure-registries": []
   }
   EOF
   ​
   ## docker unit file
   cat << EOF > /etc/systemd/system/docker.service
   [Unit]
   Description=Docker Application Container Engine
   Documentation=https://docs.docker.com
   #BindsTo=containerd.service
   After=network-online.target  firewalld.service
   Wants=network-online.target
   ​
   [Service]
   Type=notify
   # the default is not to use systemd for cgroups because the delegate issues still
   # exists and systemd currently does not support the cgroup feature set required
   # for containers run by docker
   #ExecStart=/usr/bin/dockerd -H fd://  
   ExecStart=/usr/bin/dockerd -H unix://
   ExecReload=/bin/kill -s HUP $MAINPID
   LimitNOFILE=1048576
   # Having non-zero Limit*s causes performance problems due to accounting overhead
   # in the kernel. We recommend using cgroups to do container-local accounting.
   LimitNPROC=infinity
   LimitCORE=infinity
   # Uncomment TasksMax if your systemd version supports it.
   # Only systemd 226 and above support this version.
   TasksMax=infinity
   TimeoutStartSec=0
   # set delegate yes so that systemd does not reset the cgroups of docker containers
   Delegate=yes
   # kill only the docker process, not all processes in the cgroup
   KillMode=process
   # restart the docker process if it exits prematurely
   Restart=on-failure
   StartLimitBurst=3
   StartLimitInterval=60s
   ​
   [Install] 
   WantedBy=multi-user.target
   EOF
   ​
   ```

3. 启动验证

   ```text
   $ systemctl enable docker
   $ systemctl start docker
   $ docker info
   ```

4. upgrade

   To upgrade your manual installation of Docker CE, first stop any `dockerd` or `dockerd.exe`processes running locally, then follow the regular installation steps to install the new version on top of the existing version.

   > [https://docs.docker.com/install/linux/docker-ce/binaries/](https://docs.docker.com/install/linux/docker-ce/binaries/)

   即，执行如下操作即可：

   ```text
   $ tar xzvf /path/to/<FILE>.tar.gz
   $ sudo cp docker/* /usr/bin/
   ```

   end

5. journalctl 日志速率优化

   现象：用`kubectl -n some-ns logs some-pod -c one-container`命令查看不了该pod中container的日志，而用该命令去查看其它pod的日志，是可以正常显示的。

   ```text
   docker run --name some-name -d some-image
   docker logs some-name # 这里还是没有输出日志
   ​
   ## 由于默认的docker logging driver 是 json-file 及日志输出到journald
   $ journalctl _SYSTEMD_UNIT=systemd-journald.service
   ...
   Jan 10 19:18:50 tshift-master systemd-journal[547]: Suppressed 1833 messages from /system.slice/docker.service
   Jan 10 19:19:20 tshift-master systemd-journal[547]: Suppressed 1849 messages from /system.slice/docker.service
   Jan 10 19:19:50 tshift-master systemd-journal[547]: Suppressed 1857 messages from /system.slice/docker.service
   Jan 10 19:20:20 tshift-master systemd-journal[547]: Suppressed 2010 messages from /system.slice/docker.service
   Jan 10 19:20:50 tshift-master systemd-journal[547]: Suppressed 1820 messages from /system.slice/docker.service
   ...
   ​
   ```

   这里可以看到有大量的`Suppressed xxx messages from /system.slice/docker.service`的日志，从字面意思来理解，有很多日志被抑制了。

   ```text
   # 查看配置，可以指定是哪两个限制了日志的速率
   ​
   $ man journald.conf
   ...
     RateLimitInterval=, RateLimitBurst=
          Configures the rate limiting that is applied to all messages generated on the system. If, in the time interval defined by RateLimitInterval=, more messages than specified in RateLimitBurst= are logged by a service, all further messages within the interval are dropped until the interval is over. A message about the number of dropped messages is generated. This rate limiting is applied per-service, so that two services which log do not
   interfere with each other's limits. Defaults to 1000 messages in 30s.
   The time specification for RateLimitInterval= may be specified in the following units: "s", "min", "h", "ms", "us". To turn off any kind of rate limiting, set either value to 0.
   ...
   ```

   统计journalctl日志速率：`journalctl -u docker --since "" --until "" |wc -l`

   避免docker日志丢失`/etc/systemd/journald.conf`

   ```text
   # 直接修改/etc/systemd/journald.conf，增大RateLimitBurst配置项的取值，修改完毕后重启journald服务
   $ vim /etc/systemd/journald.conf
   [Journal]
   ...
   ​
   #RateLimitInterval=30s
   #RateLimitBurst=1000  
   ...
   ​
   $ systemctl restart systemd-journald
   ​
   ```

   end

6. Docker Daemon添加自签证书（不需要重启奥）

   [https://docs.docker.com/registry/insecure/](https://docs.docker.com/registry/insecure/) 0.0 还是官网讲究--

   Instruct every Docker daemon to trust that certificate. The way to do this depends on your OS.

   * **Linux**: Copy the `domain.crt` file to`/etc/docker/certs.d/myregistrydomain.com:5000/ca.crt` on every Docker host. **You do not need to restart Docker**.

     eg： `/etc/docker/certs.d/21.49.22.250/ca.crt`

7. end

**etcd**

最佳的ETCD规划，三套etcd： calico一套，apiserver两个（一个独立出来events）

这里仅一套etcd集群。

1. 生成kubeadm配置文件

   ```text
   # Update HOST0, HOST1, and HOST2 with the IPs or resolvable names of your hosts
   export HOST0=172.29.33.1
   export HOST1=172.29.33.2
   export HOST2=172.29.33.3
   ​
   # Create temp directories to store files that will end up on other hosts.
   mkdir -p /tmp/${HOST0}/ /tmp/${HOST1}/ /tmp/${HOST2}/
   mkdir -p /etc/kubernetes/etcd/pki/
   ​
   ETCDHOSTS=(${HOST0} ${HOST1} ${HOST2})
   NAMES=("infra0" "infra1" "infra2")
   ​
   for i in "${!ETCDHOSTS[@]}"; do
   HOST=${ETCDHOSTS[$i]}
   NAME=${NAMES[$i]}
   cat << EOF > /tmp/${HOST}/kubeadmcfg.yaml
   apiVersion: "kubeadm.k8s.io/v1beta1"
   kind: ClusterConfiguration
   kubernetesVersion: v1.14.0-rc.1
   etcd:
       local:
           dataDir: "/data/etcd"
           serverCertSANs:
           - "${HOST}"
           peerCertSANs:
           - "${HOST}"
           extraArgs:
               initial-cluster: ${NAMES[0]}=https://${ETCDHOSTS[0]}:2380,${NAMES[1]}=https://${ETCDHOSTS[1]}:2380,${NAMES[2]}=https://${ETCDHOSTS[2]}:2380
               initial-cluster-state: new
               name: ${NAME}
               listen-peer-urls: https://${HOST}:2380
               listen-client-urls: https://${HOST}:2379
               advertise-client-urls: https://${HOST}:2379
               initial-advertise-peer-urls: https://${HOST}:2380
   imageRepository: 21.49.22.250/kubeadm
   certificatesDir: "/etc/kubernetes/etcd/pki"
   ​
   ​
   EOF
   done
   ​
   cat << EOF > /tmp/initetcd.yaml
   apiVersion: "kubeadm.k8s.io/v1beta1"
   kind: ClusterConfiguration
   kubernetesVersion: v1.14.0-rc.1
   certificatesDir: "/etc/kubernetes/etcd/pki"
   EOF
   ​
   ```

2. 利用kubeadm生成etcd证书

   ```text
   echo 'generate ca certificates'
   kubeadm init phase certs etcd-ca --config=/tmp/initetcd.yaml
   ​
   HOST0=172.29.33.1
   HOST1=172.29.33.2
   HOST2=172.29.33.3
   ​
   echo 'generate server&client certificates'
   ​
   ​
   kubeadm init phase certs etcd-server --config=/tmp/${HOST1}/kubeadmcfg.yaml
   kubeadm init phase certs etcd-peer --config=/tmp/${HOST1}/kubeadmcfg.yaml
   kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
   kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
   cp -R /etc/kubernetes/etcd/pki /tmp/${HOST1}/
   find /etc/kubernetes/etcd/pki -not -name ca.crt -not -name ca.key -type f -delete
   ​
   kubeadm init phase certs etcd-server --config=/tmp/${HOST2}/kubeadmcfg.yaml
   kubeadm init phase certs etcd-peer --config=/tmp/${HOST2}/kubeadmcfg.yaml
   kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
   kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
   cp -R /etc/kubernetes/etcd/pki /tmp/${HOST2}/
   # cleanup non-reusable certificates
   find /etc/kubernetes/etcd/pki -not -name ca.crt -not -name ca.key -type f -delete
   ​
   kubeadm init phase certs etcd-server --config=/tmp/${HOST0}/kubeadmcfg.yaml
   kubeadm init phase certs etcd-peer --config=/tmp/${HOST0}/kubeadmcfg.yaml
   kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
   kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
   # No need to move the certs because they are for HOST0
   ​
   # clean up certs that should not be copied off this host
   find /tmp/${HOST2} -name ca.key -type f -delete
   find /tmp/${HOST1} -name ca.key -type f -delete
   scp -r /tmp/${HOST1}/* ${HOST1}:/etc/kubernetes/etcd/
   scp -r /tmp/${HOST2}/* ${HOST2}:/etc/kubernetes/etcd/
   ​
   ```

   Ps： 涉及到node间证书的下发需提前配置好ssh互信。

3. 生成etcd unit file

   ```text
   # Update HOST0, HOST1, and HOST2 with the IPs or resolvable names of your hosts
   export HOST0=172.29.33.1
   export HOST1=172.29.33.2
   export HOST2=172.29.33.3
   ​
   # Create temp directories to store files that will end up on other hosts.
   mkdir -p /tmp/${HOST0}/ /tmp/${HOST1}/ /tmp/${HOST2}/
   ​
   ETCDHOSTS=(${HOST0} ${HOST1} ${HOST2})
   NAMES=("infra0" "infra1" "infra2")
   ​
   # ${!ETCDHOSTS[@]} is index of array
   for i in "${!ETCDHOSTS[@]}"; do
   HOST=${ETCDHOSTS[$i]}
   NAME=${NAMES[$i]}
   cat << EOF > /tmp/${HOST}/etcd.service
   [Unit]
   Description=Etcd Server
   After=network.target
   After=network-online.target
   Wants=network-online.target
   Documentation=https://github.com/coreos
   ​
   [Service]
   Type=notify
   WorkingDirectory=
   ExecStart=/usr/bin/etcd \\
     --data-dir=/data/etcd \\
     --advertise-client-urls=https://${HOST}:2379 \\
     --cert-file=/etc/kubernetes/etcd/pki/etcd/server.crt \\
     --client-cert-auth=true \\
     --data-dir=/data/etcd \\
     --initial-advertise-peer-urls=https://${HOST}:2380 \\
     --initial-cluster=${NAMES[0]}=https://${ETCDHOSTS[0]}:2380,${NAMES[1]}=https://${ETCDHOSTS[1]}:2380,${NAMES[2]}=https://${ETCDHOSTS[2]}:2380 \\
     --initial-cluster-state=new \\
     --key-file=/etc/kubernetes/etcd/pki/etcd/server.key \\
     --listen-client-urls=https://${HOST}:2379 \\
     --listen-peer-urls=https://${HOST}:2380 \\
     --name=${NAME} \\
     --peer-cert-file=/etc/kubernetes/etcd/pki/etcd/peer.crt \\
     --peer-client-cert-auth=true \\
     --peer-key-file=/etc/kubernetes/etcd/pki/etcd/peer.key \\
     --peer-trusted-ca-file=/etc/kubernetes/etcd/pki/etcd/ca.crt \\
     --snapshot-count=10000 \\
     --trusted-ca-file=/etc/kubernetes/etcd/pki/etcd/ca.crt \\
     --auto-compaction-mode=periodic \\
     #etcd can be set to automatically compact the keyspace with the --auto-compaction option with a period of hours
     --auto-compaction-retention=1 \\
     --max-request-bytes=33554432 \\
     # ETCDdb数据大小，默认是２G,当数据达到２G的时候就不允许写入，必须对历史数据进行压缩才能继续写入；　
     # 这是 6G
     # $((6*1024*1024*1024)) 这么写直观些
     --quota-backend-bytes=6442450944 \\
     --heartbeat-interval=250 \\
     --election-timeout=2000 
   Restart=on-failure
   RestartSec=5
   LimitNOFILE=65536
   ​
   [Install]
   WantedBy=multi-user.target
   EOF
   scp /tmp/${HOST}/etcd.service ${HOST}:/etc/systemd/system/
   done
   ​
   ```

4. 启动并验证etcd cluster

   ```text
   ## 集群初始化，建议单独登录到每个node上启动etcd
   $ systemctl daemon-reload
   $ systemctl enable etcd
   $ systemctl start etcd
   ​
   ## cluster health status
   ## 证书目录和上面自定义的证书目录保持一致
   $ ETCDCTL_API=2 \etcdctl --cert-file /etc/kubernetes/etcd/pki/etcd/peer.crt --key-file /etc/kubernetes/etcd/pki/etcd/peer.key --ca-file /etc/kubernetes/etcd/pki/etcd/ca.crt --endpoints https://172.29.33.1:2379 cluster-health
   ​
   ```

**kubelet**

Kubelet、kubeadm和kubectl下载地址： `kubeadm config images list`

 [https://kubernetes.io/docs/setup/release/notes/\#server-binaries](https://kubernetes.io/docs/setup/release/notes/#server-binaries)

用脚本创建kubelet unit文件.

```text
#!/bin/bash
command='''
echo "mkdir manifest"
mkdir -p /etc/kubernetes/manifests
mkdir -p /etc/systemd/system/kubelet.service.d/
​
echo "unit file"
cat << EOF > /etc/systemd/system/kubelet.service
[Unit]
Description=kubelet: The Kubernetes Node Agent
Documentation=http://kubernetes.io/docs/
​
[Service]
ExecStart=/usr/bin/kubelet
Restart=always
StartLimitInterval=0
RestartSec=10
​
[Install]
WantedBy=multi-user.target
EOF
​
​
echo 'unit file config'
cat << EOF > /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS=--pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true --pod-infra-container-image=21.49.22.250/k8s/pause-amd64:3.1 --node-ip
EOF
​
cat << EOF  > /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
EnvironmentFile=-/etc/sysconfig/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet   \$KUBELET_KUBECONFIG_ARGS \$KUBELET_CONFIG_ARGS \$KUBELET_KUBEADM_ARGS \$KUBELET_EXTRA_ARGSDM_ARGS
​
EOF
echo 'run with start'
systemctl daemon-reload
systemctl  enable kubelet
'''
for i in `seq 1 6` ;do echo 172.29.33.$i; ssh 172.29.33.$i  "$command";done
```

end

1. 创建kubelet unit file

   ```text
   #!/bin/bash
   echo "create requisite directory"
   mkdir -p /etc/kubernetes/manifests
   mkdir -p /etc/systemd/system/kubelet.service.d/
   ​
   echo "unit file"
   cat << EOF > /etc/systemd/system/kubelet.service
   [Unit]
   Description=kubelet: The Kubernetes Node Agent
   Documentation=http://kubernetes.io/docs/
   ​
   [Service]
   ExecStart=/usr/bin/kubelet
   Restart=always
   StartLimitInterval=0
   RestartSec=10
   ​
   [Install]
   WantedBy=multi-user.target
   EOF
   ​
   ​
   echo 'unit file config'
   cat << EOF > /etc/sysconfig/kubelet
   KUBELET_EXTRA_ARGS=--pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true --pod-infra-container-image=21.49.22.250/k8s/pause-amd64:3.1 --node-ip ----image-pull-progress-deadline=4m
   EOF
   ​
   cat << EOF  > /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
   [Service]
   Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
   Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
   EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
   EnvironmentFile=-/etc/sysconfig/kubelet
   ExecStart=
   ExecStart=/usr/bin/kubelet   \$KUBELET_KUBECONFIG_ARGS \$KUBELET_CONFIG_ARGS \$KUBELET_KUBEADM_ARGS \$KUBELET_EXTRA_ARGS
   ​
   EOF
   ​
   ## 提高kubelet镜像拉取时间 默认是1m0s 拉取大镜像可能时间不够，从而报错：ImagePullBackOff
   --image-pull-progress-deadline duration
   If no pulling progress is made before this deadline, the image pulling will be cancelled. This docker-specific flag only works when container-runtime is set to docker. (default 1m0s)
   ```

2. 启动验证kubelet

   ```text
   # Please don't start kubelet
   systemctl daemon-reload
   systemctl  enable kubelet
   ​
   ```

**证书自动更新**

还需要创建一些role和赋权`--rotate-certificates`

Auto rotate the kubelet client certificates by requesting new certificates from the kube-apiserver when the certificate expiration approaches. \(DEPRECATED: This parameter should be set via the config file specified by the Kubelet's --config flag.

注意哦，这个和kubeadm join 自动签发证书是两回事。

**haproxy + keepalive**

HA其实就是保证vip的可用性--

所谓HA\(High Availability\),高可用,就是指在本地系统的某个组件出现故障的情况下，系统依然可访问,应用的能力。 而为了达到这种目的，我们一般采用节点冗余的方法组成集群来提供服务。通常来说，可以分为两种冗余的方式：

* Active/Passive HA: 即A/P模式，中文主备模式，通常会有一个主节点来提供服务，备用节点会同步/异步与主节点进行数据同步。当主节点故障时，备用节点会启用代替主节点。通常会使用CRM（Cluster Resource Manager ）软件比如Pacemaker来进行主备节点之间的切换并提供一个VIP提供服务。

  0.0 我想明白为什么keepalive为什么总和LVS或者Haproxy一起出现使用了。

  因为Keepalived只会转移网络层面或者Keepalived本身的故障，当它后端的服务出现问题是时，默认情况下（无vrrp\_script的情况下）它是不会故障转移的，所以呀，`keepalived+HAproxy/LVS/Nginx`，代理服务器屏蔽了大量的后端服务，而Keepalived只需要在通过`vrrp_script`保证最外层lb server`HAproxy/lvs/nginx`的状态就行了。

  0.0 keepalive 一般两个节点做主备。`vrrp_script`在非抢占模式下，检测到服务中断后，停止service在vip发生漂移后在启动service。

* Active/Active HA:也就是我们常说的A/A方式，中文也称作双活或多主模式。这种模式下，系统在集群的所有服务器上运行同样的负载，也就是说集群内服务器的功能都是一样的。这种模式下，通常会采用负载均衡软件如HAProxy提供VIP进行负载均衡。

涉及到具体的服务时，HA的部署方式也不尽相同，一般来说，一些无状态的服务比如提供http请求转发的http服务器等，可以直接部署A/A模式，无需考虑数据不一致的问题。而一些有状态的服务，比如数据库等，则需要具体问题具体对待，如果这类服务提供原生的A/A服务，那么尽量利用他们提供的原生A/A服务。如果实在不行那么只能从A/P角度去部署。

每个master上都要启动`haproxy+keepalive`

keepalived

单开一篇，自己去看：VRRP协议只关心网络链路层的问题，keepalived基于vrrp协议，但是又加了自己的东西（检测后端应用的vrrp\_script）才能变成一个可靠的软件。

很详细： [https://www.keepalived.org/manpage.html](https://www.keepalived.org/manpage.html)

failover： [https://www.keepalived.org/doc/case\_study\_failover.html](https://www.keepalived.org/doc/case_study_failover.html)

sync\_group:

vrrp\_sync\_group的应用场景为：如果路由有2个网段，一个内网，一个外网，每个网段开启一个VRRP实例，假设VRRP配置为检查内网，那么当外网出现问题 时，VRRPD会认为自己是健康的，则不会发送Master和Backup的切换，从而导致问题，Sync Group可以把两个实例都放入Sync Group，这样的话，Group 里任何一个实例出现问题都会发生切换。

```text
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.23.8.80
    }
}
vrrp_instance VI_N {
    state MASTER
    interface eth1
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        172.18.1.254
    }
}
```

end

我想明白为什么keepalive为什么总和LVS或者Haproxy一起出现使用了。

因为Keepalived只会转移网络层面或者Keepalived本身的故障，当它后端的服务出现问题是时，默认情况下（无vrrp\_script的情况下）它是不会故障转移的，所以呀，`keepalived+HAproxy/LVS/Nginx`，代理服务器屏蔽了大量的后端服务，而Keepalived只需要在通过`vrrp_script`保证`HAproxy/lvs/nginx`的状态就行了。

```text
vrrp_script check_nginx {
  script " if [ -f /usr/local/nginx/logs/nginx.pid ]; then exit 0 ; else exit 1; fi"
  interval 2
  fall 1
  rise 1
  weight 30
}
​
track_script {
  check_nginx
}
​
在非抢占模式下，要在检测脚本中增加杀掉keepalived进程（或者停用keepalived服务）的方式，做到业务进程出现问题时完成主备切换。
---
vrrp_script checkhaproxy
{
    script "/home/check.sh"
    interval 3
    weight -20
}
​
vrrp_instance test
{
    ...
    
    track_script
    {
        checkhaproxy
    }
    
    ...

```

抢占模式：

* state：分为`MASTER` 和 `BACKUP`两种
* priority： 按priority数值大小，决定优先级，数值越大，优先级越高，当优先级高的机器恢复时，回去preempt\(抢占\) master

非抢占模式：

非抢占模式有个大问题： 当master上的业务出现问题但是keepalived出现问题时，不会发生抢占，这样业务还是受影响的。所以需要在`vrrp_script`中先中断keepalived，使VIP漂移后在启动keepalived、

检测脚本中，要加上终止Master node的keepalived

* state：仅当state全为`BACKUP`，才会生效`nopreempt`参数配置
* nopreempt： nopreempt" allows the lower priority machine to maintain the master role, even when a higher priority machine comes back online.

  0.0 无视，priority参数，lower priority machine也可维持 master role.

1.keepalived宕机或者关闭keepalived后负载均衡会自动切换至从keepalived,即使当主keepalived恢复后负载均衡也不会切回主keepalived减少了不必要的多次切换，损失性能和减少了切换导致的数据丢失风险。

2.采用选举的方式选举产生主keepalived，且主keepalived与实际提供服务的keepaliived为同一台服务器，防止了主keepalived转发错误导致的数据包丢失不完整以及脑裂等时断时续可能出现的问题。

3.由于没有主keepalived，所以当提供服务的主keepaived宕机后，剩余的从keepalived服务器会自动根据优先级选举出新的主keepalived并提供服务，所以主keepalived宕机后不会影响服务。

```text
## 抢占式配置
## KEEPALIVED_STATE The starting state of keepalived; it can either be MASTER or BACKUP.
## KEEPALIVED_PRIORITY Keepalived node priority. Defaults to 150
$ $ docker run -d --restart=always --name=keepalive  --cap-add=NET_ADMIN --net=host --env KEEPALIVED_INTERFACE="interface_name" --env KEEPALIVED_UNICAST_PEERS="['IP1','IP2','IP3']" --env KEEPALIVED_VIRTUAL_IPS="VIP_address" --env KEEPALIVED_PRIORITY="100"  --env KEEPALIVED_STATE="master" IMAGE_NAME
​
​
# keepalive镜像默认是非抢占模式 ,所有master node都要运行
# KEEPALIVED_STATE: BACKUP
$ docker run -d --restart=always --name=keepalive  --cap-add=NET_ADMIN --net=host --env KEEPALIVED_INTERFACE="interface_name" --env KEEPALIVED_UNICAST_PEERS="['IP1','IP2','IP3']" --env KEEPALIVED_VIRTUAL_IPS="VIP_address" IMAGE_NAME
​
​
​
## 自动生成的配置文件，非抢占模式。
## 
# VRRP will normally preempt a lower priority machine when a higher priority
# machine comes online.  "nopreempt" allows the lower priority machine to
# maintain the master role, even when a higher priority machine comes back
# online.
# NOTE: For this to work, the initial state of this
# entry must be BACKUP.
# --
# nopreempt、
​
## 这默认配置不行啊，当haproxy进程挂掉不会触发 keepalived转移
[root@cs1-k8s-m1 ~]# docker exec -it aa cat /usr/local/etc/keepalived/keepalived.conf
global_defs {
  default_interface bond2
}
​
vrrp_instance VI_1 {
  interface bond2
​
  state BACKUP 
  virtual_router_id 51
  priority 150
  nopreempt # 关闭VIP的抢占，state都为BACKUP时生效
​
  unicast_peer {
    ['172.29.33.1','172.29.33.2','172.29.33.3']
  }
​
  virtual_ipaddress {
    172.29.33.230
  }
​
  authentication {
    auth_type PASS
    auth_pass d0cker
  }
​
  notify "/container/service/keepalived/assets/notify.sh"
}
0.0
​
```

haproxy

```text
$ cat /etc/haproxy/haproxy.cfg 
global
  log 127.0.0.1 local0 # 日志输出配置，所有日志都记录在本机，通过local0输出
  log 127.0.0.1 local1 notice
  tune.ssl.default-dh-param 2048
​
defaults
  log global
  mode http ##所处理的类别,默认采用http模式 默认有tcp、http和health
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
frontend kube-apiserver-https
   mode tcp
   #bind :8443
   bind :6443
   default_backend kube-apiserver-backend
​
backend kube-apiserver-backend
    mode tcp
    balance roundrobin
    stick-table type ip size 200k expire 30m
    stick on src
    server apiserver1 172.29.33.1:6443 check
    server apiserver2 172.29.33.2:6443 check
    server apiserver3 172.29.33.3:6443 check
    
 $ docker run -d  --restart=always -v /path/to/etc/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro Image_Name   
```

**control plane**

X509 SAN： [http://liaoph.com/openssl-san/](http://liaoph.com/openssl-san/)

kubeadm的config file 放到一个固定目录下，保持好。以后可能会用到。

这里需要注意默认的Kubeadm生成是证书是1年，需要手动编译kubeadm将生成证书时间设置为10年或者更久，详见其他文章。

1. 使用kubeadm 初始化第一个master

   ```text
   $ cat config.yaml
   apiVersion: kubeadm.k8s.io/v1beta1
   kind: InitConfiguration
   bootstrapTokens:
   - token: "zjsgcc.3f89s0fje9f38fhf"
     description: "another bootstrap token"
     ttl: 24h0m0s
     usages:
     - authentication
     - signing
   localAPIEndpoint:  ## It's crucial. It can assign LivenessProbe IP with mutliinterfaces.
     advertiseAddress: "InterfaceX"
     bindPort: 6443
   ---
   apiVersion: kubeadm.k8s.io/v1beta1
   kind: ClusterConfiguration
   kubernetesVersion: v1.14.0-rc.1
   ## vip
   controlPlaneEndpoint: "10.147.248.19:80"
   etcd:
      external:
        endpoints:
        - "https://172.29.33.1:2379"
        - "https://172.29.33.2:2379"
        - "https://172.29.33.3:2379"
        caFile: "/etc/kubernetes/etcd/pki/etcd/ca.crt"
        certFile: "/etc/kubernetes/etcd/pki/apiserver-etcd-client.crt"
        keyFile: "/etc/kubernetes/etcd/pki/apiserver-etcd-client.key"
   networking:
     serviceSubnet: "10.96.0.0/12"
     podSubnet: "10.100.0.0/16"
   ## 追加进主机名，避免IP变化
   apiServer:
     certSANs:
     - 172.29.33.1
     - 172.29.33.2
     - 172.29.33.3
     - hostname_1
     - hostname_2
     - hostname_3
     - vip
     extraArgs:
       advertise-address: "172.29.33.1"
       bind-address: "172.29.33.1"
       authorization-mode: "Node,RBAC"
     timeoutForControlPlane: 4m0s
   certificatesDir: "/etc/kubernetes/pki"
   imageRepository: "21.49.22.250/kubeadm"
   useHyperKubeImage: false
   clusterName: "example-cluster"
   ---
   apiVersion: kubelet.config.k8s.io/v1beta1 
   kind: KubeletConfiguration
   maxPods: 200
   # kubelet specific options here
   ---
   apiVersion: kubeproxy.config.k8s.io/v1alpha1
   kind: KubeProxyConfiguration
   mode: ipvs
   # testing, failed.
   # while decoding JSON: json unknown field "serverTLSBootstrap"
   #serverTLSBootstrap: true
   # kube-proxy specific options here
   ​
   ​
   $ kube-proxy -h
   ...
   --nodeport-addresses strings                   A string slice of values which specify the addresses to use for NodePorts. Values may be valid IP blocks (e.g. 1.2.3.0/24, 1.2.3.4/32). The default empty string slice ([]) means to use all local addresses.
   ​
   ### 初始化集群
   $ kubeadm init --config=config.yaml
   ```

2. 将其余master节点依次加入集群

   ```text
   # prerequisite
   $ cat kubeadm-join-prepare.sh
   #!/bin/bash
   s2.sh /etc/kubernetes/pki/ca.crt 
   s2.sh /etc/kubernetes/pki/ca.key
   s2.sh /etc/kubernetes/pki/sa.key
   s2.sh /etc/kubernetes/pki/sa.pub
   s2.sh /etc/kubernetes/pki/front-proxy-ca.crt
   s2.sh /etc/kubernetes/pki/front-proxy-ca.key
   ​
   # 指明为--control-plane 
   $ kubeadm join
   $ kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866 --control-plane --certificate-key f8902e114ef118304e561c3ecd4d0b543adc226b7a07f675f56564185ffe0c07
   ​
   # 修改apiserver地址
   $ cat  apiserver_patch.sh
   ​
   export HOST0=172.29.33.1
   export HOST1=172.29.33.2
   export HOST2=172.29.33.3
   ​
   HOSTS=(${HOST1} ${HOST2})
   ​
   ​
   for i in "${!HOSTS[@]}"; do
     HOST=${HOSTS[$i]}
     ssh ${HOST} "sed -i 's/advertise-address=${HOST0}/advertise-address=${HOST}/' /etc/kubernetes/manifests/kube-apiserver.yaml"
     ssh ${HOST} "sed -i 's/bind-address=${HOST0}/bind-address=${HOST}/' /etc/kubernetes/manifests/kube-apiserver.yaml"
   done
   ```

3. kubelet

   config.yaml会生成`kubectl get cm kubeadm-config` 和`kubectl get cm kubelet-config-1.14`

   这个会进一步转成`/var/lib/kubelet/config.yaml` emmm，每次nede注册的时候会生成个`/var/lib/kubelet/config.yaml`文件

4. worker节点加入集群

   ```text
   kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866
   0.0 参数少
   ```

5. end

**kube-proxy的优化**

kube-proxy是daemonSet方式部署

* --nodeport-addresses strings 指定nodePort 时，在node上监听在哪个IP上
* service的`externalTrafficPolicy":"Local"`

  [https://kubernetes.io/docs/tutorials/services/source-ip/](https://kubernetes.io/docs/tutorials/services/source-ip/)

  ```text
          client
         ^ /   \
        / /     \
       / v       X
     node 1     node 2
      ^ |
      | |
      | v
   endpoint
 
   $ kubectl run source-ip-app --image=k8s.gcr.io/echoserver:1.4
   $ kubectl expose deployment source-ip-app --name=clusterip --port=80 --target-port=8080
   $ kubectl expose deployment source-ip-app --name=nodeport --port=80 --target-port=8080 --type=NodePort
 
  ## 加externalTrafficPolicy":"Local"时，每个node上的nodeport端口都可以访问到pod
  ## pod on 21.49.22.9
            client
               \ ^
                \ \
                 v \
     node 1 <--- node 2
      | ^   SNAT
      | |   --->
      v |
   endpoint
 
  # 在node-21.49.22.4上执行的，client_address应该是21.49.22.4
  $ curl 21.49.22.5:31783
  CLIENT VALUES:
  client_address=21.49.22.5 ## 又转了下，client_address 也被改变了-- 有些应用需要源IP
  command=GET
  real path=/
  query=nil
  request_version=1.1
  request_uri=http://21.49.22.5:8080/
  ​
  SERVER VALUES:
  server_version=nginx: 1.10.0 - lua: 10001
  ​
  HEADERS RECEIVED:
  accept=*/*
  host=21.49.22.5:31783
  user-agent=curl/7.29.0
  BODY:
  -no body in request-
  ​
  ## 启用externalTrafficPolicy":"Local"
  $ kubectl patch svc nodeport -p '{"spec":{"externalTrafficPolicy":"Local"}}
  ​
  # 不会转发了-- 只能访问22.9了 pod在这上面
  $ curl -I 21.49.22.5:31783
  curl: (7) Failed connect to 21.49.22.5:31783; Connection refused
  ​
  $ curl 21.49.22.9:31783
  CLIENT VALUES:
  client_address=21.49.22.4
  command=GET
  real path=/
  query=nil
  request_version=1.1
  request_uri=http://21.49.22.9:8080/
  ​
  SERVER VALUES:
  server_version=nginx: 1.10.0 - lua: 10001
  ​
  HEADERS RECEIVED:
  accept=*/*
  host=21.49.22.9:31783
  user-agent=curl/7.29.0
  BODY:
  -no body in request-
  ```

  end

**controller-manager**

0.0可以设置TLS bootstrap自动签发的证书的时间

```text
--experimental-cluster-signing-duration duration     Default: 8760h0m0s
The length of duration signed certificates will be given.
0.0 默认8760h就是一年--
```

end

**Kubeadm token**

kubeadm join 的 token会过期再生成一个，便可用于node加入集群。

这里注意： 自动生成证书和证书自动自动更新是两回事。 TLS bootstrap是自动签发证书，而证书的自动更新需要kubelet开启参数的。

