# calico部署和基本概念

Calico’s network policy engine formed the original reference implementation of Kubernetes network policy during the development of the API.

即：Full Kubernetes network policy support

calico镜像中的`install-cni.sh` 脚本会将cni用的二进制文件丢在host的`/opt/cni/bin（默认目录）`中。

注意：部署前，要配置一个参数，让calico-node组件能够识别node的IP，node上可能有多块网卡，官方提供的yaml文件中，ip识别策略（IPDETECTMETHOD）没有配置，即默认为first-found，这会导致一个网络异常的ip作为nodeIP被注册，从而影响node-to-node mesh。我们可以修改成can-reach的策略，尝试连接某一个Ready的node的IP，以此选择出正确的IP。

he `interface` method uses the supplied interface regular expression \(golang syntax\) to enumerate matching interfaces and to return the first IP address on the first matching interface. The order that both the interfaces and the IP addresses are listed is system dependent.

e.g.

```text
# Valid IP address on interface eth0, eth1, eth2 etc.
IP_AUTODETECTION_METHOD=interface=eth.*
IP6_AUTODETECTION_METHOD=interface=eth.*
```

1. 修改yaml，配置etcd certificate

   ```text
   ## delete taints
   $ kubectl taint nodes --all node-role.kubernetes.io/master-
   ​
   ## 将证书信息写到calico.yaml
   ## 记得是 -w 0
   $ cat <path> | base64 -w 0
   ​
   ## calico 配置很多
   1. 启用IPIP模式
    - name: CALICO_IPV4POOL_IPIP
      value: "Always"
   2. 指定CIDR
   由于默认的node 子网是26，故这里CIDR随便指定，后续通过calicoctl修改IPPOOLS
    - name: CALICO_IPV4POOL_CIDR
      value: "10.100.0.0/16"
   3. 修改image，使用内网仓库镜像
   4. etcd证书配置
   5. 指定interface
   ## 部署calico
   $ kubectl apply -f calico-with-etcd.yaml
   ​
   ​
   --
   apiVersion: projectcalico.org/v3
   kind: Node
   metadata:
     annotations:
       projectcalico.org/kube-labels: '{"beta.kubernetes.io/arch":"amd64","beta.kubernetes.io/os":"linux","kubernetes.io/arch":"amd64","kubernetes.io/hostname":"10.212.0.101","kubernetes.io/os":"linux"}' #该字段可以删除
     creationTimestamp: null #该字段可以删除
     labels:
       beta.kubernetes.io/arch: amd64
       beta.kubernetes.io/os: linux
       kubernetes.io/arch: amd64
       kubernetes.io/hostname: 10.212.0.101
       kubernetes.io/os: linux
     name: 10.212.0.101
   spec:
     bgp:
       ipv4Address: 10.212.0.101/23
       ## CrossSubnet Mode 也会用这个ip
       ipv4IPIPTunnelAddr: 172.30.50.64 #当启用了IPIP模式则会有该字段，如果不启用则会没有，在"- name: CALICO_IPV4POOL_IPIP"当中如果是"never"时则没有。
   ```

2. 修改IPPOOL block size

   ```text
   ## 查看calico-node pod
   $ kubectl -n kube-system get pod -l k8s-app=calico-node -o go-template='{{range.items}}{{printf "%s\n" .metadata.name}}{{end}}'
   ​
   $ kubectl -n kube-system cp /usr/bin/calicoctl POD_NAME:/tmp
   ​
   ## 修改IPPOOL
   $ kubectl -n kube-system exec -it POD_NAME sh
   (pod)$ cat /tmp/ippool.yaml
   apiVersion: projectcalico.org/v3
   kind: IPPool
   metadata:
     name: default-ipv4-ippool
   spec:
     bolockSize: 26
     cidr: x.x.x.x/16
     ipipMode: Always
     disabled: true
     natOutgoing: true
   (pod)$ /tmp/calicoctl apply -f   /tmp/ippool.yaml
   (pod)$ cat /tmp/ippool-new.yaml
   apiVersion: projectcalico.org/v3
   kind: IPPool
   metadata:
     name: ipv4-ippool
   spec:
     bolockSize: 24
     cidr: y.y.y.y/16
     ipipMode: Always
     ## 默认都是nat默认出去的
     natOutgoing: true
  
   (pod)$ /tmp/calicoctl apply -f   /tmp/ippool-new.yaml
   (pod)$ /tmp/calicoctl get ippool -o wide
   ​
   ## 删除使用旧IPPOOL分配IP的pod
   $ kubectl delete pod -l k8s-app=kube-dns -n kube-system
   ​
   ```

en

**calico masquerade**

0.0 在pod中访问`21.49.22.3`,然后在22.3上抓包`tcp -i any -vv port 80 and host POD_IP`是抓不到包的，只有

`tcp -i any -vv port 80 and host POD_in_node_IP`才能抓到包。

How can I enable NAT for outgoing traffic from containers with private IP addresses?

If you want to allow containers with private IP addresses to be able to access the internet then you can use your data center’s existing outbound NAT capabilities \(typically provided by the data center’s **border routers**\).

> 0.0 要想podip直接在网络中传输，还需要边界路由的配置

Alternatively you can use Calico’s built in outbound NAT capability by enabling it on any Calico IP pool. In this case Calico will perform outbound NAT locally on the compute node on which each container is hosted.

```text
## 默认都是nat出去的即使用node的IP
## iptables可以看到masquerade的配置
cat << EOF | calicoctl apply -f -
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: ippool-1
spec:
  cidr: <CIDR>
  natOutgoing: true
EOF
```

Where `<CIRD>` is the CIDR of your IP pool, for example `192.168.0.0/16`.

Remember: the security profile for the container will need to allow traffic to the internet as well. Refer to the appropriate guide for your orchestration system for details on how to configure policy.

**How can I enable NAT for incoming traffic to containers with private IP addresses?**

[https://docs.projectcalico.org/v3.5/usage/troubleshooting/faq\#how-can-i-enable-nat-for-outgoing-traffic-from-containers-with-private-ip-addresses](https://docs.projectcalico.org/v3.5/usage/troubleshooting/faq#how-can-i-enable-nat-for-outgoing-traffic-from-containers-with-private-ip-addresses)

0.0

**rr-routereflector**

目的： 实现pod ip可以在集群外被访问到。

RR 模式可以实现pod ip 和 svc ip 被集群外访问。

而且这个必Flannel-gw好，f-gw模式还得加N个路由，而这个bpg只需一条路由即可。

pod\_subnet 和 svc\_subnet 都可以通过加路由暴露出去。

[https://mritd.me/2019/06/18/calico-3.6-forward-network-traffic/](https://mritd.me/2019/06/18/calico-3.6-forward-network-traffic/)

在 Calico 3.3 后支持了集群内节点的 RR 模式，即将某个集群内的 Calico Node 转变为 RR 节点；将某个节点设置为 RR 节点只需要增加 `routeReflectorClusterID` 即可。

[https://docs.projectcalico.org/v3.6/networking/bgp](https://docs.projectcalico.org/v3.6/networking/bgp)

0.0 官网都有，注意选择对应版本。

For a larger deployment you can [disable the full node-to-node mesh](https://docs.projectcalico.org/v3.6/networking/bgp#disabling-the-full-node-to-node-bgp-mesh) and configure some of the nodes to provide in-cluster route reflection. Then every node will still get all workload routes, but using a much smaller number of BGP connections.

**Note**: For a simple deployment, all the route reflector nodes should have the same cluster ID.

![calico-rr](file://D:/data_files/MarkDown/Images/calico-rr.jpg?lastModify=1582436898)

Calico方案只是把"hostnetwork"（这个是加引号的hostnetwork）方案配置自动化了。由于还是通过物理设备进行虚拟设备的管理和通信，所以整个网络依然受物理设备的限制影响；另外因为项目及设备数量的增加，内网IP也面临耗尽问题。

> Host Network”将游戏服务器的虚拟IP也配置成实体IP，并注入到主路由中。但这使得主路由的设备条目数比设计容量多了三倍。
>
> ![pod-hostnet](file://D:/data_files/MarkDown/Images/pod-hostnet.jpg?lastModify=1582436898)
>
> 0.0

0.0 也说明了这个问题，当使用bgp将网络打通后podIP也充斥着整个网络拓扑，私有ip可能不足，，，这得多少节点。

核心思路：

1. 取消node-to-node mesh
2. 配置node作为route reflector node
3. 验证node status
4. 配置集群外机器/路由器 到 podSubnet 和 svcSubnet的路由，使外部node可以访问pod

PS: 在node-to-node-mesh方式下，每个calico node执行`calicoctl node status`时，可以看到`N-1`个node的状态。但是使用RouteReflector时，RR-node和Normal-node显示的node status是不一样的。

```text
​
## 我采用deamonset方式部署的，所以在任何一个pod中操作calicoctl即可。
​
## 原集群互通方式：node-to-node mesh 
/ # /tmp/calicoctl node status
Calico process is running.
​
IPv4 BGP status
+--------------+-------------------+-------+------------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |   SINCE    |    INFO     |
+--------------+-------------------+-------+------------+-------------+
| 21.49.22.4   | node-to-node mesh | up    | 2019-11-25 | Established |
| 21.49.22.5   | node-to-node mesh | up    | 2019-11-25 | Established |
| 21.49.22.6   | node-to-node mesh | up    | 2019-11-25 | Established |
| 21.49.22.7   | node-to-node mesh | up    | 2019-11-25 | Established |
| 21.49.22.8   | node-to-node mesh | up    | 2019-11-25 | Established |
+--------------+-------------------+-------+------------+-------------+
​
IPv6 BGP status
No IPv6 peers found.
​
## 选择node1 和 node2 作为 router reflect
​
/ # vim /tmp/n1.yaml 
apiVersion: projectcalico.org/v3
kind: Node
metadata:
  labels:
    beta.kubernetes.io/arch: amd64
    beta.kubernetes.io/os: linux
    ingress: nginx
    kubernetes.io/arch: amd64
    kubernetes.io/hostname: cs1-k8s-n1.zpepc.com.cn
    kubernetes.io/os: linux
    node-role.kubernetes.io/worker: ""
    ## 增加一个label 用于区分 node 和 routeReflector， 这里命名也随意--
    route-reflector: true
  name: cs1-k8s-n1.zpepc.com.cn
spec:
  bgp:
    ipv4Address: 21.49.22.7/24
    ##个人猜测 这个clusterID随便写
    routeReflectorClusterID: 1.0.0.1
  orchRefs:
  - nodeName: cs1-k8s-n1.zpepc.com.cn
    orchestrator: k8s
/ # vim /tmp/n2.yaml 
apiVersion: projectcalico.org/v3
kind: Node
metadata:
  labels:
    beta.kubernetes.io/arch: amd64
    beta.kubernetes.io/os: linux
    ingress: nginx
    kubernetes.io/arch: amd64
    kubernetes.io/hostname: cs1-k8s-n2.zpepc.com.cn
    kubernetes.io/os: linux
    monitor: prometheus
    node-role.kubernetes.io/worker: ""
    route-reflector: true
  name: cs1-k8s-n2.zpepc.com.cn
spec:
  bgp:
    ipv4Address: 21.49.22.8/24
    routeReflectorClusterID: 1.0.0.1
  orchRefs:
  - nodeName: cs1-k8s-n2.zpepc.com.cn
    orchestrator: k8s
​
## 应用配置，设置node1 和 node2 作为 rr
/ # /tmp/calicoctl apply -f /tmp/n1.yaml 
Successfully applied 1 'Node' resource(s)
/ # /tmp/calicoctl apply -f /tmp/n2.yaml 
Successfully applied 1 'Node' resource(s)
​
## 再次查看node状态，n1 和 n2，已经变了
/ # /tmp/calicoctl node status
Calico process is running.
​
IPv4 BGP status
+--------------+-------------------+-------+------------+--------------------------------+
| PEER ADDRESS |     PEER TYPE     | STATE |   SINCE    |              INFO              |
+--------------+-------------------+-------+------------+--------------------------------+
| 21.49.22.4   | node-to-node mesh | up    | 2019-11-25 | Established                    |
| 21.49.22.5   | node-to-node mesh | up    | 2019-11-25 | Established                    |
| 21.49.22.6   | node-to-node mesh | up    | 2019-11-25 | Established                    |
| 21.49.22.7   | node-to-node mesh | start | 06:11:21   | Active Socket: Connection      |
|              |                   |       |            | refused                        |
| 21.49.22.8   | node-to-node mesh | start | 06:11:25   | Active Socket: Connection      |
|              |                   |       |            | refused                        |
+--------------+-------------------+-------+------------+--------------------------------+
​
IPv6 BGP status
No IPv6 peers found.
​
​
​
## Configure a BGPPeer resource to tell the route reflector nodes to peer with each other.
## 即RR之间的互联
/ # cat r-to-r.yaml 
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: rr-to-rr-peer
spec:
  nodeSelector: has(route-reflector)
  peerSelector: has(route-reflector)
  
## Configure a BGPPeer resource to tell the other Calico nodes to peer with the route reflector nodes.
## 即Calico nodes 与 rr-nodes直接的互联
/ # cat n-to-no.yaml 
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: node-peer-to-rr
spec:
  nodeSelector: !has(route-reflector)
  peerSelector: has(route-reflector)
​
## apply
/ # /tmp/calicoctl apply -f n-to-no.yaml 
Successfully applied 1 'BGPPeer' resource(s)
/ # /tmp/calicoctl apply -f r-to-r.yaml 
Successfully applied 1 'BGPPeer' resource(s)
/ # 
​
## 这个pod 不是 rr-node所以看到两个global
/ # /tmp/calicoctl node status
Calico process is running.
​
IPv4 BGP status
+--------------+-------------------+-------+------------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |   SINCE    |    INFO     |
+--------------+-------------------+-------+------------+-------------+
| 21.49.22.4   | node-to-node mesh | up    | 2019-11-25 | Established |
| 21.49.22.5   | node-to-node mesh | up    | 2019-11-25 | Established |
| 21.49.22.6   | node-to-node mesh | up    | 2019-11-25 | Established |
| 21.49.22.7   | node-to-node mesh | up    | 06:13:01   | Established |
| 21.49.22.8   | node-to-node mesh | up    | 06:13:03   | Established |
| 21.49.22.7   | global            | start | 06:12:59   | Idle        |
| 21.49.22.8   | global            | start | 06:12:59   | Idle        |
+--------------+-------------------+-------+------------+-------------+
​
IPv6 BGP status
No IPv6 peers found.
​
### 禁用mesh
/ # /tmp/calicoctl apply -f bgp-config.yaml 
Successfully applied 1 'BGPConfiguration' resource(s)
​
$ cat << EOF | calicoctl create -f -
 apiVersion: projectcalico.org/v3
 kind: BGPConfiguration
 metadata:
   name: default
 spec:
   logSeverityScreen: Info
   nodeToNodeMeshEnabled: false
   asNumber: 63400
​
## 非rr-node calicoctl
/ # /tmp/calicoctl node status
Calico process is running.
​
IPv4 BGP status
+--------------+-----------+-------+----------+-------------+
| PEER ADDRESS | PEER TYPE | STATE |  SINCE   |    INFO     |
+--------------+-----------+-------+----------+-------------+
| 21.49.22.7   | global    | up    | 06:14:36 | Established |
| 21.49.22.8   | global    | up    | 06:14:34 | Established |
+--------------+-----------+-------+----------+-------------+
​
IPv6 BGP status
No IPv6 peers found.
​
## rr-node calicoctl node status display
[root@cs1-k8s-m1 ~]# k exec -it calico-node-hnvfd sh
/ # /tmp/calicoctl node status
Calico process is running.
​
IPv4 BGP status
+--------------+---------------+-------+----------+-------------+
| PEER ADDRESS |   PEER TYPE   | STATE |  SINCE   |    INFO     |
+--------------+---------------+-------+----------+-------------+
| 21.49.22.5   | node specific | up    | 06:14:34 | Established |
| 21.49.22.6   | node specific | up    | 06:14:36 | Established |
| 21.49.22.8   | global        | up    | 06:14:36 | Established |
| 21.49.22.4   | node specific | up    | 06:14:34 | Established |
| 21.49.22.9   | node specific | up    | 06:14:36 | Established |
+--------------+---------------+-------+----------+-------------+
​
## 查看bgp 监听状态，判断calico network是否正常
[root@cs1-k8s-m1 zabbix]# netstat -anp|grep ESTABLISH|grep bird
tcp        0      0 21.49.22.4:43833        21.49.22.8:179          ESTABLISHED 28676/bird          
tcp        0      0 21.49.22.4:179          21.49.22.7:57003        ESTABLISHED 28676/bird          
[root@cs1-k8s-m1 zabbix]# s1.sh "netstat -anp|grep ESTABLISH|grep bird"
172.29.33.2
exec: {netstat -anp|grep ESTABLISH|grep bird}
tcp        0      0 21.49.22.5:179          21.49.22.8:47399        ESTABLISHED 9475/bird           
tcp        0      0 21.49.22.5:51054        21.49.22.7:179          ESTABLISHED 9475/bird           
172.29.33.3
exec: {netstat -anp|grep ESTABLISH|grep bird}
tcp        0      0 21.49.22.6:179          21.49.22.7:58982        ESTABLISHED 13563/bird          
tcp        0      0 21.49.22.6:51025        21.49.22.8:179          ESTABLISHED 13563/bird          
172.29.33.4
exec: {netstat -anp|grep ESTABLISH|grep bird}
tcp        0      0 21.49.22.7:179          21.49.22.5:51054        ESTABLISHED 183331/bird         
tcp        0      0 21.49.22.7:58982        21.49.22.6:179          ESTABLISHED 183331/bird         
tcp        0      0 21.49.22.7:57003        21.49.22.4:179          ESTABLISHED 183331/bird         
tcp        0      0 21.49.22.7:179          21.49.22.9:45545        ESTABLISHED 183331/bird         
tcp        0      0 21.49.22.7:53892        21.49.22.8:179          ESTABLISHED 183331/bird         
172.29.33.5
exec: {netstat -anp|grep ESTABLISH|grep bird}
tcp        0      0 21.49.22.8:179          21.49.22.6:51025        ESTABLISHED 147181/bird         
tcp        0      0 21.49.22.8:179          21.49.22.7:53892        ESTABLISHED 147181/bird         
tcp        0      0 21.49.22.8:47399        21.49.22.5:179          ESTABLISHED 147181/bird         
tcp        0      0 21.49.22.8:179          21.49.22.4:43833        ESTABLISHED 147181/bird         
tcp        0      0 21.49.22.8:179          21.49.22.9:36948        ESTABLISHED 147181/bird         
172.29.33.6
exec: {netstat -anp|grep ESTABLISH|grep bird}
tcp        0      0 21.49.22.9:36948        21.49.22.8:179          ESTABLISHED 30004/bird          
tcp        0      0 21.49.22.9:45545        21.49.22.7:179          ESTABLISHED 30004/bird 
​
/ # /tmp/calicoctl get bgpPeer
NAME              PEERIP   NODE                   ASN   
node-peer-to-rr            (global)               0     
rr-to-rr-peer              has(route-reflector)   0   
​
​
/ # /tmp/calicoctl get ippool
NAME                  CIDR             SELECTOR   
default-ipv4-ippool   192.168.0.0/16   all()      
​
/ # /tmp/calicoctl get ippool -o yaml
apiVersion: projectcalico.org/v3
items:
- apiVersion: projectcalico.org/v3
  kind: IPPool
  metadata:
    creationTimestamp: 2019-06-20T12:11:48Z
    name: default-ipv4-ippool
    resourceVersion: "3553201"
    uid: 942e9bba-9354-11e9-8482-6c92bf52e922
  spec:
    blockSize: 26
    cidr: 192.168.0.0/16
    ## 这里设置成CrossSubnet也行
    ipipMode: Never
    natOutgoing: true
    nodeSelector: all()
kind: IPPoolList
metadata:
  resourceVersion: "37065994"
​
​
​
# workloadendpoint 即pod
/ # /tmp/calicoctl get workloadEndpoint --all-namespaces
NAMESPACE        WORKLOAD                                      NODE                      NETWORKS             INTERFACE         
cattle-system    cattle-cluster-agent-6c858f878b-xx5fl         cs1-k8s-m1.zpepc.com.cn   192.168.36.85/32     calib2b8b0c572a   
...
yxwyy            eunomia-cas-67fc99f4cb-f77st                  cs1-k8s-n3.zpepc.com.cn   192.168.11.236/32    calibe0045ced01   
​
​
## 呐，重头戏了。在外部机器上添加路由即可实现
## svc ip也可以真的野啊
[root@cs1-harbor-1 ~]# ip r add 192.168.0.0/16 via 21.49.22.7
​
[root@cs1-harbor-1 ~]# curl  192.168.1.161:8080 -I
HTTP/1.1 200 OK
Server: nginx/1.10.0
Date: Sun, 29 Dec 2019 08:59:33 GMT
Content-Type: text/plain
Connection: keep-alive
​
## 添加node-1 或 node-2 都行
[root@cs1-harbor-1 ~]# ip r add 10.96.0.0/12 via 21.49.22.8
[root@cs1-harbor-1 ~]# curl  10.102.102.13:8080 -I
HTTP/1.1 200 OK
Server: nginx/1.10.0
Date: Sun, 29 Dec 2019 09:07:08 GMT
Content-Type: text/plain
Connection: keep-alive
​
## 当然最方便的肯定是将这一步在开发网络的路由上做，设置完成后开发网络就可以直连集群内的 Pod IP 和 Service IP 了；至于想直接访问 Service Name 只需要调整上游 DNS 解析既可
https://blog.foxsar.black/?p=246
```

end

**IPIP（和overlay相反）**

这个好有意思，overlay 是Node\_IP带飞Pod\_IP，而Calico正好相反，它是Pod\_IP带飞Node\_ip.2333

因为Calico是纯三层的解决方式啊，直接路由PodIP。calico的IPIP可以解决node 不在同一网段的问题。

0.0 ipip 还是为了解决node间二层不通的问题

一个msater节点，ip 172.171.5.95，一个node节点 ip 172.171.5.96

pod1落在master节点上 ip地址为192.168.236.3，pod2落在node节点上 ip地址为192.168.190.203

打开ICMP 285，pod1 ping pod2的数据包，能够看到该数据包一共5层，其中IP所在的网络层有两个，分别是pod之间的网络和主机之间的网络封装。

![ipip-1](file://D:/data_files/MarkDown/Images/ipip-1.png?lastModify=1582436898)

根据数据包的封装顺序，应该是在pod1 ping pod2的ICMP包外面多封装了一层主机之间的数据包。

![ipip-2](file://D:/data_files/MarkDown/Images/ipip-2.png?lastModify=1582436898)

之所以要这样做是因为tunl0是一个隧道端点设备，在数据到达时要加上一层封装，便于发送到对端隧道设备中。

两层IP封装的具体内容

![ipip-3](file://D:/data_files/MarkDown/Images/ipip-3.png?lastModify=1582436898)

IPIP的连接方式：

![ipip-4](file://D:/data_files/MarkDown/Images/ipip-4.png?lastModify=1582436898)

0.0

吆西，ipip tunnelIP是紧挨着mac层的，用来到本机后进行解包的东西。本质上还是用来实现“大二层”

但是这个封装方式和Flannel截然不同。

FLannel是Overlay网络，是Node\_IP在Pod\_ip前面，而calico直接用Pod\_IP进行路由了，所以它的IPIP模式是Pod\_IP带着Node\_IP. 详见&lt;A Deep Dive into Flannel&gt;

**without-IPIP**

BGP网络下，没有使用IPIP模式，数据包是正常的封装。

值得注意的是mac地址的封装。192.168.236.0是pod1的ip，192.168.190.198是pod2的ip。而源mac地址是 master节点网卡的mac，目的mac是node节点的网卡的mac。这说明，在 master节点的路由接收到数据，重新构建数据包时，**使用arp请求，将node节点的mac拿到，然后封装到数据链路层**。

> 如果，二层不可达呢？？？这就无法通信了。so，需要IPIP模式了

![calico-without-ipip](file://D:/data_files/MarkDown/Images/calico-without-ipip.png?lastModify=1582436898)

0.0 啦啦啦

**Network Policy**

详见《》

