# Table of contents

* [Operations](README.md)

## Nginx

* [Nginx as a reverse proxy](nginx/nginx-as-a-reverse-proxy/README.md)
  * [general proxy configuration](nginx/nginx-as-a-reverse-proxy/proxy-configuration.md)
  * [Let nginx start if upstream host is unavailable or down](nginx/nginx-as-a-reverse-proxy/let-nginx-start-if-upstream-host-is-unavailable-or-down.md)
  * [Nginx HTTP Status Code（带完善）](nginx/nginx-as-a-reverse-proxy/nginx-http-status-code.md)
  * [Nginx + PHP](nginx/nginx-as-a-reverse-proxy/nginx+php.md)
  * [php 优化](nginx/nginx-as-a-reverse-proxy/performance-tuning-for-php.md)
  * [Nginx + Python（未完成）](nginx/nginx-as-a-reverse-proxy/t.md)
  * [Web Socket（未完成）](nginx/nginx-as-a-reverse-proxy/web-socket.md)
  * [upstream（未完成）](nginx/nginx-as-a-reverse-proxy/upstream.md)
* [Nginx configuration](nginx/nginx-configuration/README.md)
  * [生产级别的Nginx配置](nginx/nginx-configuration/select-and-poll-and-epoll.md)
  * [Nginx Location priority](nginx/nginx-configuration/nginx-location-you-xian-ji.md)
  * [Nginx Virtual Servers priority](nginx/nginx-configuration/server-blocks-or-virtual-hosts.md)
  * [https and SNI](nginx/nginx-configuration/ssl.md)
  * [Nginx Module（未完成）](nginx/nginx-configuration/nginx-module.md)
  * [backlog（转载）](nginx/nginx-configuration/backlog.md)
* [Nginx as a Web server](nginx/nginx-as-a-web-server.md)

## tomcat

* [JVM](tomcat/t2333/README.md)
  * [JVM问题排查](tomcat/t2333/jvm-wen-ti-pai-cha.md)
  * [jvm配置](tomcat/t2333/jvm-pei-zhi.md)

## Linux <a id="test_group_3"></a>

* [Kernel](test_group_3/kernel/README.md)
  * [总结整理的参数优化](test_group_3/kernel/zong-jie-zheng-li-de-can-shu-you-hua.md)
* [基础](test_group_3/t3/README.md)
  * [kernel](test_group_3/t3/kernel.md)
  * [Memory（未）](test_group_3/t3/memory.md)
  * [CPU](test_group_3/t3/cpu/README.md)
    * [top load average in Linux](test_group_3/t3/cpu/top-load-average-in-linux.md)

## Kubernetes

* [The concept of k8s](kubernetes/the-concept-of-k8s/README.md)
  * [kube-proxy](kubernetes/the-concept-of-k8s/kube-proxy.md)
  * [kube-controller-manager](kubernetes/the-concept-of-k8s/kube-controller-manager.md)
  * [k8s: authentication and authorization](kubernetes/the-concept-of-k8s/k8s-authentication-and-authorization.md)
  * [Admission：ResourceQuota and LimitRange](kubernetes/the-concept-of-k8s/admission-resource-request-and-resource-limit.md)
* [基本搭建](kubernetes/ji-chu-zu-jian/README.md)
  * [k8s平台搭建（仅上传了第一阶段其他组件比较乱还未整理）](kubernetes/ji-chu-zu-jian/k8s-ping-tai-da-jian.md)
  * [HA Harbor](kubernetes/ji-chu-zu-jian/ha-harbor.md)
  * [k8s集群参数优化](kubernetes/ji-chu-zu-jian/k8s-ji-qun-can-shu-you-hua.md)
* [Network](kubernetes/network/README.md)
  * [calico部署和基本概念](kubernetes/network/apiserver.md)
* [无状态容器应用](kubernetes/rong-qi-ying-yong/README.md)
  * [高并发容器应用优化](kubernetes/rong-qi-ying-yong/gao-bing-fa-rong-qi-ying-yong-you-hua.md)
* [有状态应用](kubernetes/you-zhuang-tai-ying-yong.md)
* [监控（未整理）](kubernetes/jian-kong.md)

## Docker

* [runtime](docker/runtime/README.md)
  * [OCI and CRI（待整）](docker/runtime/oci-and-cri-dai-zheng.md)
  * [runc\(带整理\)](docker/runtime/runc-dai-zheng-li.md)
* [Namespace](docker/namespace/README.md)
  * [The concept of namespace](docker/namespace/the-foundation-of-namespace.md)
* [Cgroup](docker/cgroup/README.md)
  * [The concept of cgroup](docker/cgroup/the-concept-of-cgroup.md)
  * [Systemd and Cgroup](docker/cgroup/systemd-with-cgroup.md)
  * [Docker and Cgroup](docker/cgroup/docker-with-cgroup.md)
  * [Cgroup + tc实现流量限速](docker/cgroup/cgroup-+-tc-shi-xian-liu-liang-xian-su.md)
* [Daemon](docker/daemon/README.md)
  * [Storage driver](docker/daemon/storage-driver/README.md)
    * [Storage Driver Overview](docker/daemon/storage-driver/storage-driver-overview.md)
    * [Use the OverlayFS storage driver](docker/daemon/storage-driver/use-the-overlayfs-storage-driver.md)
  * [Logging Driver](docker/daemon/logging-driver.md)
* [Image](docker/image/README.md)
  * [核心概念](docker/image/he-xin-gai-nian.md)
  * [Dockerfile](docker/image/t1.md)
* [Container](docker/container/README.md)
  * [docker exec and docker attach](docker/container/docker-exec-and-docker-attach.md)
  * [PID 1](docker/container/pid-1.md)
* [Network（未整理）](docker/network.md)
* [Storage Driver（未整理）](docker/storage-driver.md)

## Shell

* [Script](shell/script/README.md)
  * [Shell Script（待整理细分）](shell/script/shell-script.md)

## TCP/IP

* [IP](tcp-ip/ip.md)
* [TCP](tcp-ip/tcp.md)

## HTTP

* [基础](http/ji-chu.md)

## Python

* [待整理](python/dai-zheng-li.md)

