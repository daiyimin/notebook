我们从不同角度来学习K8S的安全机制。
参考文档: [Security Checklist](https://kubernetes.io/docs/concepts/security/security-checklist)

[TOC]
# 集群控制面的安全
## Namespace
K8S的命名空间的主要作用是把集群中的各组资源进行隔离。通过对资源的隔离，就能对不同的租户设置不同的安全策略。
K8S中有资源是Namespace范围内起作用的。比如，NetworkPolicy只限制指定的命名空间内的POD直接的连接。
而有些资源是在Cluster范围内起作用的。比如，PersistentVolume是集群范围的资源，所有Namespace都能访问。
| Namespace-wise API Resource|Cluster-wise API Resource|
|-|-|
|| Namespace |
|Role, RoleBinding|ClusterRole, ClusterRoleBinding|
|NetworkPolicy|  |
|Resource Quota|  |
|PersistentVolumeClaim| PersistentVolume, StorageClass |
|POD|Node |
|Service| |
||CRD|
* Network Policy
下例中，metadata.namespace定义了该NetworkPolicy所属的命名空间（Namespace）为default命名空间。
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
```
* PersistentVolumeClaim
下例中，metadata.namespace定义了该PersistentVolumeClaim所属的命名空间（Namespace）为foo命名空间。
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: foo-pvc
  namespace: foo
spec:
  storageClassName: "" # Empty string must be explicitly set otherwise default StorageClass will be set
  volumeName: foo-pv
  ...
```
## API access control
K8S的API Server是集群控制面的核心，它提供了大量的API，使得用户，POD能够管理和访问集群中的资源对象。所以，对API server的访问控制对保护集群资源对象至关重要。
通过赋予不同的用户、POD不同的角色，RBAC机制可以让他们只拥有某个命名空间内范围访问API的权限。
参考：[API access control](/Kubenetes/API%20access%20control.md)
## Quota
管理员可以为每个命名空间创建ResourceQuota，例如CPU、memory的配额。K8S的配额系统会确保命名空间内申请的资源总数不超过给它设定的配额。
如果命名空间内设定了ResourceQuota，那么用户在创建资源（POD,server）时，必须也要指明需要的Resource request和limit。否则，创建请求会被拒绝。



# 集群数据面的安全
## Network Policy
# POD安全机制
应用程序是以POD的形式运行在集群中。考虑到K8S支持多租户功能，不同租户的POD杂处于集群中，租户间POD的安全隔离就尤为重要。
## Node隔离
不同安全等级的POD要避免放置在同一个Node上。
Pods that are on different tiers of sensitivity, for example, an application pod and the Kubernetes API server, should be deployed onto separate nodes. The purpose of node isolation is to prevent an application container breakout to directly providing access to applications with higher level of sensitivity to easily pivot within the cluster. 
可以通过K8S的调度机制实现POD的Node隔离，包括
* NodeSelector
* Taint & Tolerance
* RuntimeClass
## POD的Memory/CPU request & limit
同一个Node上，所有POD共享Node的Memory和CPU资源。因此，每个POD必须设置Memory和CPU request & limit，以防止一个有漏洞的POD被DOS攻击后，把整个Node的CPU和Memory耗尽。这样Node上的其它POD也得不到足够的Memory和CPU，无法正常运行。
另外，参见[Quota](#quota)

## POD Security Context 
Security Context的是PodSpec的一部分，它描述了POD运行时的安全行为规范。

Security Context在Kubernetes中用于限制容器对宿主节点的可访问范围，以防止容器进行非法的系统级别操作，影响宿主节点或其他容器组。

Security Context支持多种安全设定，包括访问权限控制（基于userID和groupID）、SELinux安全标签、Linux Capabilities（为容器分配部分特权）、AppArmor（安全模块，用于限制进程权限）以及Seccomp（过滤系统调用）等。

另一个关键设定是AllowPrivilegeEscalation，它定义了容器进程是否可以比其父进程获得更多的特权。

可以看出Security Context的设定大都是和[Linux内核安全有关的限制](https://kubernetes.io/docs/concepts/security/linux-kernel-security-constraints/)。

如何利用Security Context实现这些内核安全限制，请看 [Configure a Security Context for a Pod or Container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)


## POD Security Standard [^1]
The Pod Security Standards 定义了三种安全策略： privileged, baseline and restricted。它们能够限制PodSpec中安全相关的字段的取值范围，包括但不限于SecurityContext中的字段。
这三种策略对POD的纵容程度由高到低，
* privileged是完全不作限制的安全策略，一般只有K8S系统或基础设施的workload才会使用它。
* baseline提供了一个较为平衡的安全策略，可以作为具体应用的安全限制的基准，在此基础上添加应用所需的其它安全限制。
* restricted则是最为严格的安全策略，基于POD安全的最佳实践，但往往不适用于一般的应用需求。
### Baseline 策略
下面，我们来看baseline策略里的一些安全规则：
* 主机命名空间 规则
```
限制的字段
  * spec.hostNetwork
  * spec.hostPID
  * spec.hostIPC
允许的取值
  * nil
  * false
```
这个规则针对POD是否能够使用主机命名空间内的网络，进程号等。
以spec.hostPID为例，它是一个布尔值字段，可以设置为 true 或 false。当设置为 true 时，Pod 中的容器将共享宿主机的 PID 命名空间。这意味着容器中的进程将能够看到并访问宿主机上的所有进程，并且它们的 PID 将与宿主机上的 PID 直接对应。
使用 hostPID 增加了安全风险，因为容器中的进程将具有比通常更高的权限来访问和操作宿主机上的进程。除非有明确的需求，否则应避免使用 hostPID 以保持 Pod 的安全性和隔离性。
所以，baseline策略防止了POD把hostPID设置成true的可能。
* AppArmor 规则
```
限制的字段
  * spec.securityContext.appArmorProfile.type
  * spec.containers[*].securityContext.appArmorProfile.type
  * spec.initContainers[*].securityContext.appArmorProfile.type
  * spec.ephemeralContainers[*].securityContext.appArmorProfile.type
允许的取值
  * Undefined/nil
  * RuntimeDefault
  * Localhost
```
这个规则针对POD能够使用的AppArmor的Profile。在支持AppArmor的Node上，POD默认应用RuntimeDefault profile。这条规则能够防止RuntimeDefault profile被覆盖，或者只能用允许的profile去覆盖POD的AppArmor设置。
> 注释：AppArmor是一个Linux内核安全模块，用于限制程序（包括进程和它们的子进程）的访问权限。它允许系统管理员定义精细的访问控制策略，以限制程序可以执行的操作。
在Kubernetes中，可以使用AppArmor来增强容器的安全性。这通常是通过在容器上设置特定的AppArmor配置文件来实现的，这些配置文件定义了容器进程可以执行的操作。
### POD Security Standard的工作原理
POD Security Standard是K8S控制面的机制，用来限制PODSpec中的SecurityContext字段和其它与安全相关的字段的取值。通过排除不安全的取值，来提高POD的安全性。
具体来讲，在新建POD对象时，API Server中的POD Security Admission Controller会根据命名空间或者集群中的POD Security Standard设置，检查PODSpec中的相关字段是否只含有不被允许的值。比如上面提到的spec.hostPID是否是true。如果发现了不被允许的值，那么POD创建就会失败（假设应用了enforce模式），或者会触发审计日志（假设应用audit）。
如果通过了POD Security Admission Controller的检查，POD创建成功。这时，PODSpec中的SecurityContext定义了POD的运行时的安全相关行为。例如spec.hostPID=false就决定POD的进程号不属于host的PID命名空间，不能访问host命名空间中的其它进程。
### 设置POD Security Standard策略
可以通过namespace或者cluster对象的label来定义POD Security Standard策略。
在应用策略时，admission controller支持三种模式
* enforce
发现有策略违例的情况，就拒绝创建POD。
* audit
策略违例会触发审计日志中记录新事件时添加审计注解；但是 Pod 仍是被接受的。
* warn
策略违例会触发用户可见的警告信息，但是 Pod 仍是被接受的。

下面来看一个例子：
```
kubectl label namespace verify-pod-security pod-security.kubernetes.io/enforce=baseline
```
在例子里，我们为verify-pod-security命名空间添加了一个label：pod-security.kubernetes.io/enforce=baseline\. 意思是在该命名空间创建的POD都要服从baseline策略。应用策略的模式为enforce。



[^1]: https://kubernetes.io/blog/2021/12/09/pod-security-admission-beta/
