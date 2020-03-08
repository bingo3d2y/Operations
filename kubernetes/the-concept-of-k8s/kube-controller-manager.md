# kube-controller-manager

#### kube-controller-manager

The Kubernetes controller manager is a daemon that embeds the core control loops shipped with Kubernetes. In applications of robotics and automation, a control loop is a non-terminating loop that regulates the state of the system. In Kubernetes, a controller is a control loop that watches the shared state of the cluster through the apiserver and makes changes attempting to move the current state towards the desired state.

k8s controller manager包含了很多个不会终止的control loops，不断watch集群当前状态并使保障它们和期望状态一致。

Logically, each controller is a separate process, but to reduce complexity, they are all compiled into a single binary and run in a single process.

> 呐，按照功能可以划分出很多controller，但是为了降低复杂性它们被集成在一个二进制文件了--

These controllers include:

* **Node Controller**: Responsible for noticing and responding when nodes go down.
* **Replication Controller**: Responsible for maintaining the correct number of pods for every replication controller object in the system.

  > RC --&gt; RS 丰富了Selector 操作
  >
  > `RC`只支持基于等式的`selector`（env=dev或app=nginx），但`RS`还支持基于集合的`selector`（version in \(v1, v2\)）

  This controller will watch both the ReplicaSet resource and a set of Pods based on the selector in that resource. It then takes action to create/destroy Pods in order to maintain a stable set of Pods as described in the ReplicaSet.

* **Endpoints Controller**: Populates the Endpoints object \(that is, joins Services & Pods\).
* **Service Account & Token Controllers**: Create default accounts and API access tokens for new namespaces.

这里，后续再研究某个control loop代码在来深入吧。

