# k8s: authentication and authorization

核心： x509 and token

**users in kubernetes**

All Kubernetes clusters have two categories of users: service accounts managed by Kubernetes, and normal users.

Normal users are assumed to be managed by an outside, independent service. An admin distributing private keys, a user store like Keystone or Google Accounts, even a file with a list of usernames and passwords. In this regard, _Kubernetes does not have objects which represent normal user accounts._Regular users cannot be added to a cluster through an API call.

In contrast, service accounts are users managed by the Kubernetes API. They are bound to specific namespaces, and created automatically by the API server or manually through API calls. Service accounts are tied to a set of credentials stored as `Secrets`, which are mounted into pods allowing in-cluster processes to talk to the Kubernetes API.

k8s里面的用户分两种：Service Account 和normal users.

normal users: 由CA证书签发，使用数字签名技术根据用户信息生成独特的证书。

service accounts：由通过k8s API管理，有默认自动创建的sa，也可以自己调用api创建sa.

![](../../.gitbook/assets/k8s-user.jpg)

**user accounts vs service accounts**

Kubernetes distinguishes between the concept of a user account and a service account for a number of reasons:

* User accounts are for humans. Service accounts are for processes, which run in pods.

  > Service account for processes.  
  > 比如：基于SA的Dashboard Pod登录认证

* User accounts are intended to be global. Names must be unique across all namespaces of a cluster, future user resource will not be namespaced. Service accounts are namespaced.

  > User accounts 属于全局的，而Service account 属于namespace级别的。
  >
  > 正是由于SA的作用范围是NS级别的，所以才可以...细粒度的控制资源。

* Typically, a cluster’s User accounts might be synced from a corporate database, where new user account creation requires special privileges and is tied to complex business processes. Service account creation is intended to be more lightweight, allowing cluster users to create service accounts for specific tasks \(i.e. principle of least privilege\).

  > principle of least privilege -- 最小权限法则
  >
  >  就是说User一般是和公司现有的数据库打通的比较繁琐，权限还不好控制。

* Auditing considerations for humans and service accounts may differ.
* A config bundle for a complex system may include definition of various service accounts for components of that system. Because service accounts can be created ad-hoc and have namespaced names, such config is portable.

  SA热插拔，易组装。比如Dashboard的基于SA的token认证，随时创建随时生效，随时删除随时失效。

**authentication认证**

* X509 Client Certs

  这个玩多了不多说\(详见openssl X509\)。提一个关键：证书里包含user、group信息，具体为证书的subject.CommonName（user name）以及subject.Organization（group）。

* Static Token File

  认证token放在文件里面：`token,user,uid,"group1,group2,group3"`

* BootStarp Token

  先用一个低权限的token认证过，然后授权它可以发起CSR，自签证书，从而提权。

* Static Password File

  `password,user,uid,"group1,group2,group3"`

  和Static token 差不多

* Service Account Tokens

  The signed JWT can be used as a bearer token to authenticate as the given service account.

  随着Service Account自动secret中的token，是用来认证用的。鉴权这一步，是根据sa名字来做的。

  A service account provides an identity for processes that run in a Pod.

  > When using bearer token authentication from an http client, the API server expects an Authorization header with a value of Bearer THETOKEN. The bearer token must be a character sequence that can be put in an HTTP header value using no more than the encoding and quoting facilities of HTTP. For example: if the bearer token is 31ada4fd-adec-460c-809a-9e56ceb75269 then it would appear in an HTTP header as shown below.
  >
  > Authorization: Bearer 31ada4fd-adec-460c-809a-9e56ceb75269
  >
  > 即，控制谁（哪些token）可以访问这个pod 进程。
  >
  > 例子

  Use the Default Service Account to access the API server 何解？

  When you create a pod, if you do not specify a service account, it is automatically assigned the `default`service account in the same namespace.

  ```text
  $ kubectl get pods nginx-7cdbd8cdc9-2dwvx -o jsonpath={.spec.serviceAccountName}
  default
  ```

  You can access the API from inside a pod using automatically mounted service account credentials, as described in [Accessing the Cluster](https://kubernetes.io/docs/user-guide/accessing-the-cluster/#accessing-the-api-from-a-pod). The API permissions of the service account depend on the [authorization plugin and policy](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#authorization-modules) in use.

**authorization授权**

一般使用RBAC，apiserver参数：`--authorization-mode=Node,RBAC`

* node

  详见 [https://kubernetes.io/docs/reference/access-authn-authz/node/](https://kubernetes.io/docs/reference/access-authn-authz/node/)

  可以玩性很高，可以限制kubelet的权限。

  A special-purpose authorizer that grants permissions to kubelets based on the pods they are scheduled to run.

  这么说，node是专门给kubelet准备的鉴权咯。就是为了kubelet保障pod运行

  In order to be authorized by the Node authorizer, kubelets must use a credential that identifies them as being in the `system:nodes` group, with a username of `system:node:<nodeName>`. This group and user name format match the identity created for each kubelet as part of [kubelet TLS bootstrapping](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/).

   node通过使用包含the `system:nodes` group, with a username of `system:node:<nodeName>`的证书来获取操作权限---

* ABAC

  To enable `ABAC` mode, specify `--authorization-policy-file=SOME_FILENAME` and `--authorization-mode=ABAC` on startup.

  很low的模式

* RBAC

  Role-based access control \(RBAC\) is a method of regulating access to computer or network resources based on the roles of individual users within an enterprise.

  这个主流，RBAC一种很好的设计方式。

* Webhook

  新特性 没玩过-

  [https://kubernetes.io/docs/reference/access-authn-authz/webhook/](https://kubernetes.io/docs/reference/access-authn-authz/webhook/)

k8s 提供的查询权限的接口

```text
$ kubectl  auth can-i create ns 
yes
$ kubectl  auth can-i create deployment -n kube-system
yes
​
```

**基于Service Account的认证和授权**

① 创建sa时，会自动生成一个secret。

②自动生成的secret包含了一个token.这个token类似bearer token，用来证明sa的身份。

> emmm, 和x509证书，用来证明User 一样的。

所以呢，这个secret中的token可以用来登录的凭证。可通过下面命令拿到Token

`$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep <Secret_Name> | awk '{print $1}')`

③通过Rolebinding和ClusterRolebinding 将sa与role关联起来，完成授权操作。

①和②是认证，③是授权

这两句很关键

A service account provides an identity for processes that run in a Pod.

All Kubernetes clusters have two categories of users: service accounts managed by Kubernetes, and normal users.

例子，创建不同的sa，来登录k8s Dashboard并细化权限。

```text
# 单独的role，绑定到test命名空间，用这个token来登录dashboard
1. 创建role，分配权限
$ cat test-admin-role.yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata: 
  name: test-admin  
  namespace: test   ## 这个role属性，test命名空间
rules:   ## admin嘛权限放的比较宽，后面可以根据具体用户细化权限
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - '*'
2. 在test这个ns下创建sa
 $ kubeclt create sa admin-test -n=test
3.完成对sa授权即rolebinding
 $ kubectl create rolebinding dash-test --role=test-admin --serviceaccount=test:admin-test -n=test   ## 注意咯，此处要不是不加-n=test，则这个rolebinding就会创建到default下面
4. 用token登录验证
 $ kubectl get sa admin-test -o yaml -n=test ## 获取sa对应的secrets
   ...
 secrets:
 -  name: admin-test-token-8f4mc
 $ kubectl describe secret admin-test-token-8f4mc -n=test
 ...
将token，复制粘贴到dashboard的登录界面，则可以且仅可以显示test这个ns下的所有内容。 
```



**Service account automation**

 Three separate components cooperate to implement the automation around service accounts:

* A Service account admission controller

  0.0 保障pod里面的sa创建及/var/run/secrets/kubernetes.io/serviceaccount目录挂载

  It acts synchronously to modify pods as they are created or updated.

  1. If the pod does not have a `ServiceAccount` set, it sets the `ServiceAccount` to `default`.
  2. It ensures that the `ServiceAccount` referenced by the pod exists, and otherwise rejects it.
  3. If the pod does not contain any `ImagePullSecrets`, then `ImagePullSecrets` of the `ServiceAccount`are added to the pod.
  4. It adds a `volume` to the pod which contains a token for API access.
  5. It adds a `volumeSource` to each container of the pod mounted at `/var/run/secrets/kubernetes.io/serviceaccount`

* A Token controller

  It acts asynchronously. It:

  * observes serviceAccount creation and creates a corresponding Secret to allow API access.
  * observes serviceAccount deletion and deletes all corresponding ServiceAccountToken Secrets.
  * observes secret addition, and ensures the referenced ServiceAccount exists, and adds a token to the secret if needed.
  * observes secret deletion and removes a reference from the corresponding ServiceAccount if needed.

* A Service account controller

  确保每个namespace下有一个名为default的sa

  Service Account Controller manages ServiceAccount inside namespaces, and ensures a ServiceAccount named “default” exists in every active namespace.

参考：



