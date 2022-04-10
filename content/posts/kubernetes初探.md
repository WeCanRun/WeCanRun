---
title: "Kubernetes初探"
date: 2022-04-09T15:59:43+08:00
draft: true
toc: true
images:
tags: 
  - 云原生
  - kubernetes
---

# kubernetes 初探

## kubernetes

### 0. kubernetes 的设计理念

### 0.1 声明式 API

使用者直接描述期望状态，而不必在意过程，这样能够简化需要的代码，减少开发人员的工作；而命令式 API 更强调过程，它要求使用者提供每一个过程指令、虽然在配置上比较灵活，但是带来了更多的工作，与更在乎结果和状态的声明式 API 截然相反。

### 0.2 显示接口

Kubernetes 不存在内部的私有接口，所有的接口都是显示定义的，组件之间通信使用的接口对于使用者来说都是显式的，我们都可以直接调用。

### 0.3 无侵入性

每一个应用或者服务一旦被打包成了镜像就可以直接在 Kubernetes 中无缝使用，不需要修改应用程序中的任何代码。

### 0.4 可移植性

Kubernetes 使用 StatefulSet 能够很好的支持有状态服务，同时引入 PV （PersistentVolume）和 PVC（PersistentVolumeClaim） 概念屏蔽了底层存储的差异，与无侵入性的特点共同支持了可移植性。

### 1. kuberbetes 的组件

### 1.1 kube-apiserver

整个 Kubernetes 集群的 “灵魂”，是信息的汇聚中枢，提供了所有内部和外部的 API 请求操作的唯一入口。同时也负责整个集群的认证、授权、访问控制、服务发现等能力。用户可以通过命令行工具 `kubectl` 和 `apiserver` 进行交互，从而实现对集群中进行各种资源的增删改查等操作。`apiserver` 跟 BorgMaster 非常类似，会将所有的改动持久到 `etcd` 中，同时也保存着一份内存拷贝。

### 1.2 kube-controller-manager

负责维护整个 Kubernetes 集群的状态，比如多副本创建、滚动更新等。`kube-controller-manager` 并不是一个单一组件，内部包含了一组资源控制器，在启动的时候，会通过 `goroutine`拉起多个资源控制器。这些控制器的逻辑仅依赖于当前状态，因为在分布式系统中没办法保证全局状态的同步。同时在实现的时候避免使用过于复杂的状态机，因此每个控制器仅仅对自己对应的资源对象做操作。而且控制器做了很多容错处理，比如增加 `retry` 机制等。

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c28f5f52-9199-4a29-9cf9-ec8de9649b91/Untitled.png)

### 1.3 kube-scheduler

简单来说就是负责监听未调度的 Pod，按照预定的调度策略绑定到满足条件的节点上。需要考虑优先级、资源高效利用、高性能、QoS、affinity 和 anti-affinity、数据本地化、内部负载干扰等。

总的来说，调度的过程主要分为两个大步骤。

- 过滤一些不满足条件的节点，这个过程也称为 Predicate。
- 调度器会对这些合适的节点进行打分排序，从中选择一个最优的节点，这个过程也称为 Priority。

### 1.3.1 Predicate 策略

- PodFitsHostPort: 检查是否有 Host Port 冲突
- PodFitsPort：同 PodFitsHostPort
- PodFitsResources：检查 Node 的资源是否充足，包括允许的 Pod 数量、CPU、内存、GPU 个数以及其他的 OpaqueIntResources
- HostName: 检查 pod.Spec.NodeName 是否与候选节点一致
- MatchNodeSelector：检查 pod.Spec.NodeSelector 是否与候选节点一致
- NoVolumeZoneConflict：检查 `volume zone` 是否冲突
- MatchInterPodAffinity：检查是否匹配 pod 的亲和性要求
- NoDiskConflict：检查是否存在 Volume 冲突，仅限于 GCE PD、AWS EBS、Ceph RBD 及 iSCSI
- PodToleratesNodeTaints：检查 Pod 是否容忍 Node Taints
- CheckNodeMemoryPressure：检查 Pod 是否可以调度到 MemoryPressure 的节点上
- CheckNodeDiskPressure：检查 Pod 是否可以调度到 DiskPressure 的节点上
- NoVolumeNodeConflict： 检查节点是否满足 Pod 所引用的 `volume` 的条件
- ...

### 1.3.2 Priorities 策略

- SelectorSpreadPriority:：优先减少节点上属于同一个 Service 的 pod 数量
- InterPodAffinityPriority：优先将 Pod 调度到相同的拓扑上（如同一个节点、Rack、Zone 等）
- LeastRequestedPriority：优先调度到请求资源少的节点上
- BalancedResourceAllocation：优先平衡各节点的资源使用
- NodePreferAvoidPodsPriority： [alpha.kubernetes.io/preferAviodPods](http://alpha.kubernetes.io/preferAviodPods) 字段判断，权重为 1000，避免其他优先级策略影响
- NodeAffinityPriority：优先调度到匹配 NodeAffinity 的节点上
- TaintTolerationPriority：优先调度到匹配 TaintToleration 的节点上
- ServiceSpreadingPriority：尽量将同一个 Service 的 Pod 分布到不同的节点上, 已经被SelectorSpreadPriority 替代（默认未使用）
- EqualPriority：将所有节点优先级设置为 1（默认未使用）
- ImageLocalityPriority：尽量将使用大镜像的容器调度到已经下拉了该镜像的节点上（默认未使用）
- MostRequestedPriority： 尽量调度到已经使用过的 Node 上，特别适用于 cluster-autoscaler （默认未使用）

### 1.4 CRI（container-runtime-interface）

容器运行时主要负责容器的镜像管理以及容器创建及运行。只要符合 CRI 的运行时就可以在 `kubernetes` 中运行

### 1.5 kubelet

- 接收并执行 `master`发来的指令
- 负责维护 Pod 的生命周期，比如创建和删除 Pod 对应的容器。同时也负责存储和网络的管理。一般会配合 CSI、CNI 插件一起工作。
- 每个 `kubelet` 进程会在 API Server 上注册节点自身信息，定期向 `master` 节点汇报节点的资源使用情况，并通过 `cAdvisor` 监控节点和容器的资源

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ce780321-468d-4819-ba0f-5e46366c3dca/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3f5b9047-0607-4ecc-8e95-4f0a33b41fe7/Untitled.png)

### 1.6 kube-proxy

主要负责 `kubernetes` 内部的服务通信，在主机上维护网络规则并提供转发及负载均衡能力。

### 1.7 其他非必须组件

CoreDNS 负责为整个集群提供 DNS 服务；

Ingress Controller 为服务提供外网接入能力；

Dashboard 提供 GUI 可视化界面；

Fluentd + Elasticsearch 为集群提供日志采集、存储与查询等能力。

Federation：提供跨可用区的集群；

### 2. master 和 node 的交互方式

Kubernetes 中所有的状态都是采用 `node` 向 `master`上报的方式实现的。`apiserver` 不会主动跟 `kubelet` 建立连接请求，所有容器状态汇报都是由 `kubelet` 主动向 `apiserver` 发起的。一旦新增的 `node` 被 `apiserver` 纳管进来后，`kubelet` 进程就会定时向 `apiserver` 汇报 “心跳”，即汇报自身的状态，包括自身健康状态、负载数据统计等。当一段时间内心跳包没有更新，那么此时 `kube-controller-manager` 就会将其标记为 NodeLost（失联）。

### 3. pod

pod 是 Kubernetes 中原子化的部署单元，是 Kubernetes 中不可变的基础设施， 它可以包含一个或多个容器，而且容器之间可以共享网络、存储资源。在日常使用过程中，也应该尽量避免在一个 Pod 内运行多个不相关的容器

### 3.1. 为什么 Kubernetes 不直接管理容器，而用 pod 来管理呢？

因为使用一个新的逻辑对象 Pod 来管理容器，可以在不重载容器信息的基础上，添加更多的属性，而且也方便跟容器运行时进行解耦，兼容度高；而且这些容器在通过 Kubernetes 创建之后就能共享网络和存储，可以构建比较复杂的服务拓扑和依赖关系

### 3.2 为什么要允许一个 pod 内可以包含多个容器？

用一个 pod 管理多个容器，既能够保持容器之间的隔离性，还能保证相关容器的环境一致性。使用粒度更小的容器，不仅可以使应用间的依赖解耦，还便于使用不同技术栈进行开发，同时还可以方便各个开发团队复用，减少重复造轮子。

### 3.3 pod 的状态

1. Pending:
   1. pod 还未被调度
   2. pod 内的容器镜像在待运行的节点上不存在，需要从镜像中心拉取
2. Running: pod 内的所有容器均被创建出来了，且至少有一个容器为在运行状态中
3. Succeeded: 表示 Pod 内的所有容器均成功运行结束，即正常退出，退出码为 0；
4. Failed: pod 内的所有容器均运行终止，且至少有一个容器终止失败了，一般这种情况，都是由于容器运行异常退出，或者被系统终止掉了；
5. Unknown: 一般是由于 Node 失联导致的 Pod 状态无法获取到

### 3.4 pod 的重启策略

Kubernetes 中定义了如下三种重启策略，可以通过 spec.restartPolicy 字段在 Pod 定义中进行设置。

- always: 表示一直重启，这也是默认的重启策略。Kubelet 会定期查询容器的状态，一旦某个容器处于退出状态，就对其执行重启操作；
- onFailure: 表示只有在容器异常退出，即退出码不为 0 时，才会对其进行重启操作；
- never: 表示从不重启；

### 3.6 容器生命周期内的 hook

kubernetes 中目前有以下两种 hook：

- PostStart： 可以在容器启动之后就执行。但需要注意的是，此 hook 和容器里的 ENTRYPOINT 命令的执行顺序是不确定的。
- PreStop： 则在容器被终止之前被执行，是一种阻塞式的方式。执行完成后，Kubelet 才真正开始销毁容器。

### 3.7 init 容器

一种特殊容器，通常用来做一些初始化工作，比如环境检测、OSS 文件下载、工具安装，等等。

一个 Pod 中允许有一个或多个 init 容器。init 容器和其他一般的容器非常像，其与众不同的特点主要有：

- 总是运行到完成，可以理解为一次性的任务，不可以运行常驻型任务，因为会 block 应用容器的启动运行；
- 顺序启动执行，下一个的 init 容器都必须在上一个运行成功后才可以启动；
- 禁止使用 readiness/liveness 探针，可以使用 Pod 定义的 activeDeadlineSeconds，这其中包含了 Init Container 的启动时间；
- 禁止使用 lifecycle hook。

### 3.8 Pod 的生命周期

Create –> Probe –> Running –> Shutdown –> Restart

### 3.8.1 Pod 的创建流程

1. 计算 Pod 中沙盒和容器的变更
2. 如果 Pod 有变化， 强制停止 Pod 对应的沙盒
3. 强制停止所有不应该运行的容器
4. 如果有必要为 Pod 创建新的沙盒
5. 创建 Pod 规格中指定的 init 容器
6. 依次创建 Pod 规格中指定的常规容器

### 3.8.2 Pod 健康检查

Kubernetes 中提供了一系列的健康检查，可以定制调用，来帮助解决类似的问题，我们称之为 Probe（探针）

- livenessProbe: 存活探针，可以用来探测容器是否真的在 “运行”，如果检测失败的话，这个时候 kubelet 就会停掉该容器，容器的后续操作会受到其重启策略的影响。
- readinessProbe：就绪探针，常常用于指示容器是否可以对外提供正常的服务请求
- startupProbe：启动探针，可以用于判断容器是否已经启动好，我们可以通过参数，保证有足够长的时间来应对 “超长” 的启动时间。 如果检测失败的话，同 livenessProbe 的操作。这个 Probe 是在 1.16 版本才加入进来的，到 1.18 版本变为 beta。也就是说如果 Kubernetes 版本小于 1.18 的话，你需要在 kube-apiserver 的启动参数中，显式地在 feature gate 中开启这个功能。

探针的三种检测方式：

- ExecAction: 可以在容器内执行 shell 脚本，当返回码为 9 时，探测结果为成功
- HTTPGetAction: 方便对指定的端口和 IP 地址执行 HTTP Get 操作，当返回码为 200~ 400 之间时，探测结果视为成功
- TCPSocketAction: 可以对指定端口进行 TCP 检查，当端口可达时，探测结果为成功

### 3.8.3 Pod 的删除

停止运行容器的大致流程，先从 Pod 的规格中计算出当前优雅停止的超时时间，然后运行钩子方法和内部的生命周期方法，最后将容器停止并清除引用

### 4. ReplicaController

ReplicaController 来做 Pod 的副本控制，即确保该服务的 Pod 数量维持在特定的数量，如果副本数少于预定的值，则创建新的 Pod。如果副本数大于预定的值，就删除多余的副本。

### 5. ReplicaSet

ReplicaSet（可简写为 rs） 用来替代 ReplicaController，虽然 ReplicaController 目前依然可以使用，但是社区已经不推荐继续使用了。这两者的功能和目的完全相同，但是 ReplicaSet 具备更强大的基于集合的标签选择器，这样你可以通过一组值来进行标签匹配选择。目前支持三种操作符：`in`、`notin` 和 `exists`。

### 6. Deployment

Deployment 是一种比 ReplicatonSet 更加高级的资源对象，通过 Deployment 我们可以管理多个 `label` 各不相同的 ReplicationSet，每个 ReplicaSet 负责保证对应数目的 Pod 在运行。我们就不需要再关心和操作 ReplicaSet 和 Pod 了。

![https://i.loli.net/2021/06/06/JNpxk36gf4rXF7c.png](https://i.loli.net/2021/06/06/JNpxk36gf4rXF7c.png)

### 7. StatefulSet

与以上的资源对象对比， StatefulSet 最大的不同就是**有状态**，在部署一个 StatefulSet 的时候，有个前置依赖对象，即 Service（服务）

通过 `spec.serviceName` 这个字段，保证了 StatefulSet 关联的 Pod 可以有稳定的网络身份标识，即 Pod 的序号、主机名、DNS 记录名称等。

StatefulSet 通过 PersistentVolumeClaim（PVC）可以保证 Pod 的存储卷之间一一对应的绑定关系。同时，删除 StatefulSet 关联的 Pod 时，不会删除其关联的 PVC。

StatefulSet 的特点：

- 对于一个拥有 N 个副本的 StatefulSet 来说， Pod 在部署时按照 {0 …… N-1} 的序号顺序创建的，而删除的时候按照逆序逐个删除，
- StatefulSet 创建出来的 Pod 都具有固定的、且确切的主机名
- 支持持久化存储，而且最好能够跟实例一一绑定；
- 在进行滚动升级的时候，也会按照一定顺序。

### 8. ConfigMap 和 Secret

ConfigMap 和 Secret 是 Kubernetes 常用的保存配置数据的对象，其中 Secret 会将存储的配置数据用`base64` 加密，可以根据需要选择合适的对象存储数据。通过 Volume 方式挂载到 Pod 内的，kubelet 都会定期进行更新。但是通过环境变量注入到容器中，这样无法感知到 ConfigMap 或 Secret 的内容更新。

### 9. PV 和 PVC

为了将计算和存储进行分离，Kubernetes 中引入了一个专门的对象 Persistent Volume（简称 PV）， 可以使用不同的控制器来分别管理。同时通过 PV，我们也可以和 Pod 自身的生命周期进行解耦。一个 PV 可以被几个 Pod 同时使用，即使 Pod 被删除后，PV 这个对象依然存在，其他新的 Pod 依然可以复用。为了更好地描述这种关联绑定关系，易于使用，并且屏蔽更多用户并不关心的细节参数（比如 PV 由谁提供、创建在哪个 zone/region、怎么去访问到，等等），我们通过一个抽象对象 Persistent Volume Claim（PVC）来使用 PV。

我们可以把 PV 理解成是对实际的物理存储资源描述，PVC 是便于使用的抽象 API。在 Kubernetes 中，我们都是在 Pod 中通过 PVC 的方式来使用 PV 的

创建 PV 的方式有两种：

1. 静态 PV:
   
   需要提前创建好，无法做到按需创建，根据相同的 StorageClassName 值绑定 PV 和 PVC, 如果没指定 StorageClass 且开启了 defaultStorageClass 的 adminssion plugin， 则为 PVC 添加一个默认 StorageClass，否则 StorageClassName 为 “” 。 使用过程中，经常会遇到由于资源大小不匹配，规格不对等，造成 PVC 无法绑定 PV 的情况。同时还会造成资源浪费，比如一个只需要 1G 空间的 Pod，绑定了 10G 的 PV。

2. 动态 PV：
   
   创建动态 PV，需要用 StorageClass 这个对象来描述。当用户创建好 Pod 以后，指定了 PVC，这个时候 Kubernetes 就会根据 StorageClass 中定义的 Provisioner 来调用对应的 plugin 来创建 PV。PV 创建成功后，跟 PVC 进行绑定，挂载到 Pod 中使用。

### 10 Service

Kubernetes 中用于实现服务发现和负载均衡，与 Deployment、StatefulSet 一样通过标签选择器来管理 Pod。

### 10.1. Service 一共有四种类型：

1. ClusterIP: 集群 IP, 也叫虚拟 IP, 通过集群的内部 IP 暴露服务，选择该值时服务只能够在集群内部访问。 这也是默认的 `ServiceType`。
2. NodePort：通过每个节点上的 IP 和静态端口（`NodePort`）暴露服务。 `NodePort` 服务会路由到自动创建的 `ClusterIP` 服务。 通过请求 `<节点 IP>:<节点端口>`，你可以从集群的外部访问一个 `NodePort` 服务
3. LoadBalancer ：使用云提供商的负载均衡器向外部暴露服务。 外部负载均衡器可以将流量路由到自动创建的 `NodePort` 服务和 `ClusterIP` 服务上。
4. ExternalName：常用于访问非运行在 kubernetes 中的服务，通过 ExternalName 类型的 Service便于后续迁移

### 10.2. 集群内访问 Service

一般来说，在 Kubernetes 集群内，我们有两种方式可以访问到一个 Service。

1. 如果该 Service 有 ClusterIP，我们就可以直接用这个虚拟 IP 去访问。

当然我们也可以使用该 Service 的域名，依赖于集群内部的 DNS 即可访问。相同 `namespace` 下的 `pod` 可以直接使用 `serviceName` ，如果是不同 `namespace` 下的 pod 则需要加上该 Service 所在的 `namespace` 名，即 `serviceName.namespace`。

如果在某个`namespace` 下，Service 先于 Pod 创建出来，那么 kubelet 在创建 Pod 的时候，会自动把这些 `namespace` 相同的 Service 访问信息当作环境变量注入 Pod 中，即 {SVCNAME}_SERVICE_HOST 和 {SVCNAME}_SERVICE_PORT。这里 SVCNAME 对应是各个 Service 的大写名称，名字中的横线会被自动转换成下划线。

### 10.3 Headless Service

所谓 Headless Service 其实是 ClusterIP 的一种， 即 ClusterIP 为 None，则不会分配给 Service 分配IP

1. 用户可以自己选择要连接哪个 Pod，通过查询 Service 的 DNS 记录来获取后端真实负载的 IP 地址，自主选择要连接哪个 IP；
2. 可用于部署有状态服务。每个 StatefulSet 管理的 Pod 都有一个单独的 DNS 记录，且域名保持不变，即 `<PodName>.<ServiceName>.<NamespaceName>.svc.cluster.local`。这样 Statefulset 中的各个 Pod 就可以直接通过 Pod 名字解决相互间身份以及访问问题。

**注意**

没有设置 selector 的 service 不会自动创建 endpoint，用户可以自行创建 endpoint，一般用于指向集群外的 IP

### 11. DaemonSet

可以在集群内的每个节点上（或者指定的一堆节点上）都只运行一个副本，即 Pod 和 Node 是一一对应的关系。当集群新增或下线节点时，对应的 Pod 会自动新建和删除

### 12. PriorityClass

PriorityClass 通过给 Pod 设置高优先级，让其比其他 Pod 显得更为重要，通过这种 “插队” 的方式优先获得调度器的调度。这个对象是个集群级别的定义，并不属于任何 `namespace`，可以被全局使用，即各个 `namespace` 中创建的 Pod 都能使用 PriorityClass。我们在定义这样一个 PriorityClass 对象的时候，名字不可以包含 `system-`这个前缀。

```
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:  
  name: high-priority
  value: 1000000
  globalDefault: 
  falsedescription: "This priority class should be used for XYZ service pods only."
```

### Namespace

命名空间为 Kubernetes 集群提供虚拟的隔离作用， Kubernetes 集群初始有两个名字空间，分别是默认名字空间 default 和系统名字空间 kube-system，除此以外，管理员可以创建新的名字空间满足需要。

### 用户（User Account）& 服务帐户（Service Account）

顾名思义，用户帐户为人提供账户标识，而服务账户为计算机进程和 Kubernetes 集群中运行的 Pod 提供账户标识。用户帐户和服务帐户的一个区别是作用范围；用户帐户对应的是人的身份，人的身份与服务的 Namespace 无关，所以用户账户是跨 Namespace 的；而服务帐户对应的是一个运行中程序的身份，与特定 Namespace 是相关的。

### RBAC 访问授权

Kubernetes 在 1.3 版本中发布了 alpha 版的基于角色的访问控制（Role-based Access Control，RBAC）的授权模式。相对于基于属性的访问控制（Attribute-based Access Control，ABAC），RBAC 主要是引入了角色（Role）和角色绑定（RoleBinding）的抽象概念。

在 ABAC 中， Kubernetes 集群中的访问策略只能跟用户直接关联；而在 RBAC 中，访问策略可以跟某个角色关联，具体的用户在跟一个或多个角色相关联。显然，RBAC 像其他新功能一样，每次引入新功能，都会引入新的 API 对象，从而引入新的概念抽象，而这一新的概念抽象一定会使集群服务管理和使用更容易扩展和重用。

### 13. Kubernetes 网络模型

### 13.1 pod 内容器之间的网络通信

`kubernetes` 会为每一个 `pod` 创建独立的网络命名空间，`pod` 内的容器共享一个网络命名空间，这些容器之间可以通过 `localhost` 直接访问彼此的端口。

### 13.2 pod 之间的网络通信

需满足三个条件：

- pod IP 地址唯一
- pod 发出的数据包不进行 NAT
- 需要知道 pod IP 和所在 Node IP 之间的映射关系

### 13.3 pod 到 service 之间的网络通信

当我们创建一个 Service 时，Kubernetes 会为这个服务分配一个虚拟 IP。我们通过这个虚拟 IP 访问服务时，会由 iptables 负责转发。iptables 的规则由 Kube-Proxy 负责维护。

kube-proxy 在每个节点上都运行，并会不断地查询和监听 Kube-APIServer 中 Service 与 Endpoints 的变化，来更新本地的 iptables 规则，实现其主要功能，包括为新创建的 Service 打开一个本地代理对象，接收请求针对发生变化的 Service 列表，kube-proxy 会逐个处理，处理流程如下：

1. 对已经删除的 Service 进行清理，删除不需要的 iptables 规则；
2. 如果一个新的 Service 没设置 ClusterIP，则直接跳过，不做任何处理；
3. 获取该 Service 的所有端口定义列表，并逐个读取 Endpoints 里各个示例的 IP 地址，生成或更新对应的 iptables 规则。

### 13.4 集群外部与内部组件之间的网络通信

Ingress 可以将集群内服务的 HTTP 和 HTTPS 暴露出来，以方便从集群外部访问。流量路由 Ingress 资源上定义的规则控制。

### 14. API 设计原则

- 低层 API 根据高层 API 的控制需要设计
- 尽量避免简单封装，不要有在外部 API 无法显式知道的内部隐藏的机制
- API 操作复杂度与对象数量成正比
- API 对象状态不能依赖于网络连接状态
- 尽量避免让操作机制依赖于全局状态

### 15. 架构设计原则

- 只有 APIServer 可以直接访问 etcd 存储，其他服务必须通过 Kubernetes API 来访问集群状态；
- 所有组件都应该在内存中保持所需要的状态，APIServer 将状态写入 etcd 存储，而其他组件则通过 APIServer 更新并监听所有的变化；
- 优先使用事件监听而不是轮询。
- 假定我们的系统是一个开放的环境：应该不断的去验证系统假设，优雅地接受外部的事件和修改。比如用户可以随意地删除正在被 replica set 管理的 pods，而 replica set 发现了之后只是简单的重新创建一个新的 pod 而已。
- 不要为 object 建立大而全的状态机，从而把系统的行为和状态机的变迁关联起来。
- 单节点故障不应该影响集群的状态；不要假设所有的组件都能正常运行，任何组件都有可能出错或者拒绝你的请求。etcd 可能会拒绝写入，kubelet 可能会拒绝 pod， scheduler 可能会拒绝调度，尽量进行重试或者有别的解决方案。
- 在没有新请求的情况下，所有组件应该在故障恢复后继续执行上次最后收到的请求
  （比如网络分区或服务重启等）；
- 系统组件能够自愈：比如说 cache 需要定期的进行同步，这样如果有一些 object 被错误的修改或者存储了， 删除的事件被丢失等问题能够在人类发现之前被自动修复。
- 优雅地进行降级和熔断，优先满足最重要的功能而忽略一些无关紧要的小错误。

### 16. kubenetes API 习俗

**Spec and status**

- Spec 表示系统希望到达的状态，Status 表示系统目前观测到的状态。
- PUT 和 POST 的请求中应该把 Status 段的数据忽略掉，Status 只能由系统组件来修改。
- 有一些对象可能跟 Spec 和 Status 模型相去甚远，可以吧 Spec 改成更加适合的名字。
- 如果对象符合 Spec 和 Status 的标准的话，那么除了 type，object metadata 之外不应该有其他顶级的字段。
- Status 中 phase 已经是 deprecated。因为 pahse 本质上是状态机的枚举类型，它不太符合 Kubernetes 系统设计原则， 并且阻碍系统发展，因为每当你需要往里面加一个新的 pahse 的时候你总是很难做到向后兼容性，建议使用 Condition 来代替。

**Primitive types**

- 避免使用浮点数，永远不要在 Spec 中使用它们，浮点数不好规范化，在不同的语言和计算机体系结构中有 不同的精度和表示。
- 在 JavaScript 和其他的一部分语言中，所有的数字都会被转换成 float，所以数字超过了一定的大小最好使 用 string。
- 不要使用 unsigned integers，因为不同的语言和库对它的支持不一样。
- 不要使用枚举类型，建立一个 string 的别名类型。
- API 中所有的 integer 都必须明确使用 Go（int32, int64）, 不要使用 int，在 32 位和 64 位的操作系统中他们的位数不一样。
- 谨慎地使用 bool 类型的字段，很多时候刚开始做 API 的时候是 true or false，但是随着系统的扩张，它可能 有多个可选值，多为未来打算。
- 对于可选的字段，使用指针来表示，比如 *string *int32 , 这样就可以用 nil 来判断这个值是否设置了， 因为 Go 语言中 string int 这些类型都有零值，你无法判断他们是没被设置还是被设置了零值。
