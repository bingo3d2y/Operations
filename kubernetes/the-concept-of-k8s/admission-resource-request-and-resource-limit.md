# Admission：ResourceQuota and LimitRange



**ResourceQuota**

资源控制好理解，即对pod资源就行限制，先说这个。

**request resource**

Pod使用的最小资源需求, 作为Pod调度时资源分配的判断依赖。 只有当前节点上可分配的资源量 &gt;= request 时才允许将容器调度到该节点。 request参数不限制容器的最大可使用资源

**limit resource**

容器能使用资源的最大值 设置为0表示对使用的资源不做限制, 可无限的使用

**Pod QoS**

limit和request的值决定了pod属于哪一类Qos，当node出现资源紧张时，不同QoS的驱逐优先级也是不通的。

对于不可压缩资源，如果发生资源抢占，则会按照优先级的高低进行Pod的驱逐。驱逐的策略为：

优先驱逐BestEffortPos： Request=Limit=0的Pod即都没有限制的pod

其次驱逐 Burstable pod ：0&lt;Request&lt;Limit&lt;Infinity \(Limit为0的情况也包括在内\)。

 0&lt;Request==Limit的Pod的会被保留，除非出现删除其他Pod后，节点上剩余资源仍然没有达到Kubernetes需要的剩余资源的需求.才会驱逐Guaranteed POD。

如果Memory设置Request了，最好将Limit设置等于Request，这样确保容器不会因为内存的使用量超过了Request但没有超过Limit的情况下被意外的Kill掉。但是这样，node上pod数量就会变少...

**LimitRange**

Limit Range support is enabled by default for many Kubernetes distributions. It is enabled when the apiserver `--enable-admission-plugins=` flag has `LimitRanger` admission controller as one of its arguments.

A limit range, defined by a `LimitRange` object, provides constraints that can:

* Enforce minimum and maximum compute resources usage per Pod or Container in a namespace.
* Enforce minimum and maximum storage request per PersistentVolumeClaim in a namespace.
* Enforce a ratio between request and limit for a resource in a namespace.
* Set default request/limit for compute resources in a namespace and automatically inject them to Containers at runtim

```text
admin/resource/limit-mem-cpu-container.yaml 
​
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-mem-cpu-per-container
spec:
  limits:
  - max:
      cpu: "800m"
      memory: "1Gi"
    min:
      cpu: "100m"
      memory: "99Mi"
    # 这个default就是Default Limit
    default:
      cpu: "700m"
      memory: "900Mi"
    defaultRequest:
      cpu: "110m"
      memory: "111Mi"
    type: Container
# type 也可以是 Pod 、Namespace，不同类型，选项也有些不一样。
​
kubectl describe limitrange/limit-mem-cpu-per-container -n limitrange-demo
Type        Resource  Min   Max   Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---   ---   ---------------  -------------  -----------------------
Container   cpu       100m  800m  110m             700m           -
Container   memory    99Mi  1Gi   111Mi            900Mi          -
​
如果container设置了max， pod中的容器必须设置limit，如果未设置，则使用defaultlimt的值，如果defaultlimit也没有设置，则无法成功创建
​
如果设置了container的min，创建容器的时候必须设置request的值，如果没有设置，则使用defaultrequest，如果没有defaultrequest，则默认等于容器的limit值，如果limit也没有，启动就会报错
```

