---
title: "Kubernetes服务发现与负载均衡"
date: 2022-04-10T22:47:27+08:00
draft: false
toc: true
images:
tags: 
  - 云原生
  - kubernetes

---
## 什么是服务发现和负载均衡

### 服务发现

在微服务架构是由一系列职责单一的细粒度服务构成的分布式网状结构，服务之间通过轻量机制进行通信，这时候必然引入一个服务注册发现问题，也就是说服务提供方要注册通告服务地址，服务的调用方要能发现目标服务。本质上就是一种了解集群中是否有进程监听在某个 DUP 或者 TCP 端口的机制，并且通过服务名字就能进行查找和链接，因为在微服务中，一个服务重启后其 `endpoint` 极有可能发生变化。所以需要有一个**服务注册表**来维护服务名到服务实例列表的映射，并且实时更新。**服务注册表是联系服务提供者和服务消费者的桥梁。**

### 负载均衡

一般情况下服务提供方都会以多个实例的形式运行，并且使用同一个服务名对外提供服务，那么在服务调用方试图连接服务提供方时，就必须选择一个实例， 可以由客户端选择，也可以由服务端选择，这就是负载均衡，一般情况下服务发现都伴随着负载均衡。负载均衡可以有多种不同的算法实现，常见的有轮询，随机，源地址哈希，加权轮询，加权随机等。

## 传统微服务架构下的服务发现与负载均衡实现

除了服务提供方和调用方，一般还会引入一个**服务中介**来维护服务注册表。

### 最简单的服务发现机制实现

1. 服务提供方在服务启动的时候，把服务名和实例更新到服务注册表中，在服务退出的时候删除自身的服务注册信息。
2. 服务调用方在请求服务的时候，按照服务名到服务注册表中获取服务实例列表，然后从列表里挑选一个服务实例，向该实例请求服务。

以上就是最简单的服务发现机制实现，这样的实现实际上还存在不少问题

### 问题

1. 如果服务不是正常停止，而是被强制 `kill`，那么就没有机会通知服务中介删除自身的服务注册信息，这样服务注册表就多一条无效服务实例的信息；要解决这个问题就需要服务中介具备 TTL 的能力，即为注册的服务实例设置一个过期时间，然后由服务提供方在过期时间内进行**保活**，如果在超过了过期时间，那么服务中介就自动剔除该实例的注册信息，实际上这就是简单的**健康检查**机制
2. 服务调用方如果每次调用都要先查询服务中介获取实例列表，再请求服务，那么效率就太低了，而且对服务中介来说也是请求压力；所以在实现时，通常都会让客户端**缓存服务实例列表**，这样对同名服务的多次请求，便不用重复查询，既减少了延迟又减轻了对服务中介的访问压力。
3. 在过期时间和客户端缓存失效内服务提供方挂了， 服务调用方是无法感知的，所以在这个短暂时间内服务调用方的请求还是会发到无效地址上，这个问题无法从根本上解决，所以客户端需要有容忍机制，比如在出现多次错误之后在一段时间内不再向此实例发送请求
4. 服务提供方的实例列表发生变化之后如何通知服务调用方，一般是通过服务调用方 **watch (监听)**服务中介，当服务调用方通知服务中介更新服务注册表时，服务中介需要具备**主动通知**服务调用方的能力，然后服务调用方就可以更新自己的缓存了
5. 服务调用方如何从多个服务实例中选择一个，即负载均衡算法的实现，常见的有轮询，随机，源地址哈希，加权轮询，加权随机等。
6. 服务中介如果挂了怎么办，一般都是使用**集群**解决这种单点问题

综上、要解决这些问题，还需要服务中介具备**强一致性、高可用的服务存储、TTL 、watch、健康检查、集群方案**等能力，常见的开源解决组件有 `etcd`、`zookeeper`、`consul`，本质上**使用分布式一致性数据库**来保存注册表信息，它既解决读写性能问题又提高了系统稳定性和可用性。

### 负载均衡的作用

- 解决并发压力，提高应用处理性能，增加吞吐量，加强网络处理能力
- 提供故障转移，实现高可用
- 通过增加或减少服务数量，提供伸缩性、可扩展性
- 安全防护，负载均衡设备上做一些过滤，黑名单等处理

### 负载均衡的分类

1. 按负载均衡载体划分：
   
    硬件负载均衡： 如 F5、A10 等，功能强大、性能强悍、安全性高，但成本昂贵、扩展性差
    
    软件负载均衡：如 Nginx、Envoy/Istio、 HAProxy、LVS 等，扩展性好、成本低廉，但性能较硬件负载均衡略差 
    
2. 按网络通信分类
    1. **七层负载均衡**：就是可以根据访问用户的 HTTP 请求头、URL 信息将请求转发到特定的主机
        - **DNS 负载均衡**
          
            最早的负载均衡技术， 利用域名解析实现负载均衡*，在 DNS 服务器配置多个 A 记录这些 A 记录对应的服务器构成集群，省掉了负载均衡服务器维护的麻烦，但有缓存 TTL 的问题， 支持的负载均衡算法少，只能用于业务不敏感的系统*
            
        - **HTTP 重定向**
          
            *基于 HTTP 重定向实现负载均衡， 方案简单，但需要客户端发起两次请求，性能差*
            
        - **反向代理软件**
          
            *在代理服务器上设定好负载均衡规则。然后当收到客户端请求，反向代理服务器拦截指定的域名或 IP 请求，根据负载均衡算法，将请求分发到候选服务器上。支持多种负载均衡算法，可以监控转发服务器状态，如：系统负载、响应时间、是否可用、连接数、流量等，从而动态调整负载均衡的转发策略*
            
        
    2. **四层负载均衡**：基于 IP 地址和端口进行请求的转发。
        - **IP 负载均衡**
          
            IP 负载均衡是在网络层通过修改请求数据包的目的地址进行负载均衡。
            
            IP 负载均衡在内核进程完成数据分发，较反向代理负载均衡有更好的从处理性能。但是，由于所有请求响应都要经过负载均衡服务器，集群的吞吐量受制于负载均衡服务器的带宽。
            
        - **数据链路层负载均衡**
          
            数据链路层负载均衡是指在通信协议的数据链路层修改 `mac` 地址进行负载均衡。代表的开源产品是 LVS，LVS 是基于 Linux 内核中 `netfilter` 框架实现的负载均衡系统。可以在数据包流经过程中，根据规则设置若干个关卡（hook 函数）来执行相关的操作。
            
            LVS 的工作原理：ipvs(内核态)+ ipvsadm(用户态，负责编写 ipvs 规则)
            
            1. 当用户向负载均衡调度器（Director Server）发起请求，调度器将请求发往至内核空间
            2. PREROUTING 链首先会接收到用户请求，判断目标 IP 确定是本机 IP，将数据包发往 INPUT 链
            3. IPVS 是工作在 INPUT 链上的，当用户请求到达 INPUT 时，IPVS 会将用户请求和自己已定义好的集群服务进行比对，如果用户请求的就是定义的集群服务，那么此时 IPVS 会强行修改数据包里的目标 IP 地址及端口，并将新的数据包发 POSTROUTING 链
            4. POSTROUTING 链接收数据包后发现目标 IP 地址刚好是自己的后端服务器，那么此时通过选路，将数据包最终发送给后端的服务器

## Kubernetes 是怎么实现服务发现和负载均衡的

Kubernetes 支持两种基本的服务发现模式 —— 环境变量和 DNS。

**环境变量**

- 当 Pod 运行在 `Node`上，kubelet 会为每个活跃的 Service 添加一组环境变量并注入 Pod 中
- 如果要在 Pod 中使用基于环境变量的服务发现方式，必须先创建 Service，再创建调用 Service 的 Pod。否则，Pod 中不会有该 Service 对应的环境变量。

**DNS**

DNS 服务是 Kuberneets 集群的附加组件，例如 CoreDNS， CoreDNS 监听 Kubernetes API 上创建和删除 Service 的事件，并为每一个 Service 创建一条 DNS A 记录。集群中所有的 Pod 都可以使用 DNS 将 ServiceName 解析到 Service 的 IP 地址(Headless 服务解析为 Endpoint 列表，其他服务解析为 ClusterIP)。

同时也支持 DNS SRV 记录，假设 `my-service.my-ns`Service 有一个 TCP 端口名为 `http`，则，您可以 `nslookup _http._tcp.my-service.my-ns`以发现该 Service 的 IP 地址、端口 `http` 。

对于 `ExternalName`类型的 Service，只能通过 DNS 的方式进行服务发现。

对于 “Headless” Service 而言：

- 没有 Cluster IP
- `kube-proxy` 不处理这类 Service
- Kubernetes 不提供负载均衡或代理支持

### Service

在 Kubernetes 中，可以为 pod 设置标签（Label）进行标记，并通过 Service 对象的 Selector 定义基于 Pod 标签的过滤规则，通过 Ports 定义协议类型和端口映射

```yaml
**apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376**
```

在 Kubernetes 中创建了以上的 Service 对象之后，Kubenetes 中的 kube-controller-manager 会生成用于暴露 Pod 的 Kubernetes 对象，即 Endpoint 对象；运行在每个节点上的 kube-proxy 中会监听到 Service 和 Endpoint 的事件更新，然后生成主机节点的 iptables 或者 ipvs 规则，从而实现负载均衡。

### 控制器

除了 `kube-proxy` 之外， `kube-controller-manager` 中有两个控制器监听了 Service 变动的事件，一个是 ServiceController，另一个是 EndpointController，在四种 Service 类型中，ServiceController 只处理 LoadBlancer 类型，其他三种类型由 EndpointController 进行处理，EndpointController 本身不监听 Endpoint 的变化，但是它却同时订阅了 Service 和 Pod 资源的增删事件，并根据当前集群中的对象生成 Endpoint 对象将两者进行关联。

### 代理

Kubernetes 集群中的每个节点都运行了一个 `kube-proxy`，负责为 Service（没有 Selector 的除外）提供虚拟 IP 访问****

**代理模式**

`kube-proxy` 当前的代理模式有三种：`userspace`、`iptables`、`ipvs`，这三种模式只有 `userspace`运行在用户空间，`iptables` 和 `ipvs` 运行在内核空间能够为 Kubernetes 集群提供更加强大的性能支持，Kubernetes 当前默认使用的是 `iptables` 代理模式 

**userspace 代理模式**

- ![image-20220410224950514](https://s2.loli.net/2022/04/10/vXEcnL9RAkxCJTy.png)
- `kube-proxy` 监听 `kube-api-server`以获得添加和移除 Service / Endpoint 的事件
- `kube-proxy` 在其所在的节点（每个节点都有 `kube-proxy`）上为每一个 Service 打开一个随机端口
- `kube-proxy` 安装 iptables 规则，将发送到该 Service 的 ClusterIP（虚拟 IP）/ Port 的请求重定向到该随机端口
- 任何发送到该随机端口的请求将被代理转发到该 Service 的后端 Pod 上（kube-proxy 从 Endpoint 信息中获得可用 Pod）
- `kube-proxy` 在决定将请求转发到后端哪一个 Pod 时，默认使用 `round-robin`（轮询）算法，并会考虑到 Service 中的 SessionAffinity 的设定

此代理模式的唯一优势是如果一个连接被目标服务拒绝，kube-proxy 能够重新尝试连接其他的服务，但是数据包从网卡进入内核空间之后又转发到用户空间，最终还是需要在内核空间处理，所以性能比较差

**iptables 代理模式**

- ![image-20220410225023640](https://s2.loli.net/2022/04/10/gksTPYjAzG1Ii8D.png)
- `kube-proxy` 监听 `kube-api-server` 以获得添加和移除 Service / Endpoint 的事件
- `kube-proxy` 在其所在的节点（每个节点都有 `kube-proxy`）上为每一个 Service 安装 `iptables`规则
- `iptables` 将发送到 Service 的 ClusterIP / Port 的请求重定向到 Service 的后端 Pod 上
    - 对于 Service 中的每一个 Endpoint，`kube-proxy` 安装一条 iptable 规则
    - 默认情况下，`kube-proxy` 随机选择一个 Service 的后端 Pod
    

为了避免 `kube-proxy` 将请求转发到已经存在问题的 Pod 上， 可以配置 Pod 就绪检查（`readiness probe`）确保后端 Pod 正常工作，此时，在 `iptables` 模式下 `kube-proxy` 将只使用健康的后端 Pod。

**iptables 原理**

iptables 是基于 Netfilter 框架实现的，Netfilter 实现了 5 个不同的 Hook 点，每到达一个 Hook 点时，都会调用内核模块定义的处理函数，具体调用哪个函数取决于数据包的流向，进站流量和出站流量触发的 Hook 点不同。

针对 Netfilter 的五个 Hook 点，iptables 定义了五个规则链（iptables Chain）

- PREROUTING：对数据包做路由决策前应用此链中的规则
- INPUT：路由决策之后进入本机的数据包应用此规则链中的策略
- FORWARD：转发数据包时应用此规则链中的策略
- OUTPUT：本机的数据包外出时应用此规则链中的策略
- POSTROUTING：对数据包做路由决策后应用此链中的规则

根据处理数据包的目的，规则又被划分不同的表

- fileter：根据数据包 IP、端口等信息来决定是接受还是丢弃该数据包， 多用于防火墙规则的配置
- nat：用于实现网络地址的转换规则
- managle：主要用来修改数据包的服务类型，生存周期，为数据包设置标记，实现流量整形、策略路由等
- raw：主要用来决定是否对数据包进行状态跟踪

![image-20220410225109166](https://s2.loli.net/2022/04/10/vz4ulfGSrqcVgb3.png)

当我们使用 `iptables` 的方式启动节点上的代理时，所有的流量都会先经过 `PREROUTING`或者 `OUTPUT` 链，随后进入 Kubernetes 自定义的链入口 KUBE-SERVICES、单个 Service 对应的链 `KUBE-SVC-XXXX`以及每个 Pod 对应的链 `KUBE-SEP-XXXX`，经过这些链的处理，最终才能够访问当一个服务的真实 Pod IP 地址。

如果是本机 的 Pod 发起请求，请求在经过容器网络 Namespace 转发至主机网络的 Namespace 后，依次交由 OUTPUT 链、KUBE-SERVICES 链进行处理，根据 KUBE-SERVICES 链的匹配规则检查包头，若匹配到目标 ClusterIP, 则该请求会被转发到 KUBE-SVC-XXX  链进行处理，最终随机匹配到 KUBE-SEP-XXX 的 dnat 规则，然后经过路由决策进入 POSTROUTING 链，所有经过 POSTROUTING 链的数据包都要被 KUBE-POSTROUTING 链处理，而 KUBE-POSTROUTING 链只有一个作用，那就是执行 IP 伪装，将数据包的源地址改为本机 IP 。

如果数据包是从外部通过主机IP + 服务 nodePort 进入的，那么数据包先流经网卡，在进入 Linux 内核网络协议栈，首先触发的是 PREROUTING 链上的规则，KUBE-SERVICES 依次匹配所有 ClusterIP，由于数据包目的地址是主机 IP，所有任何一条 ClusterIP 规则都匹配不到，交由最后一条跳转到 KUBE-NODEPORTS 链进行处理，匹配到 nodePort 规则之后 ，数据包先后流经 KUBE-SVC 链、KUBE-SEP 链处理， 最终随机匹配到 KUBE-SEP-XXX 的 dnat 规则。

`iptables-save -t nat` 

```bash
105 -A PREROUTING -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
108 -A OUTPUT -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
111 -A POSTROUTING -m comment --comment "kubernetes postrouting rules" -j KUBE-POSTROUTING
134 -A KUBE-POSTROUTING -m mark ! --mark 0x4000/0x4000 -j RETURN
135 -A KUBE-POSTROUTING -j MARK --set-xmark 0x4000/0x0
136 -A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -j MASQUERADE --random-fully

240 -A KUBE-SERVICES -d 11.254.6.23/32 -p tcp -m comment --comment "default/test-nodeport:test-nodeport cluster IP" -m tcp --dport 7443 -j KUBE-SVC-AODHVUMZDDCZ4Z2P
303 -A KUBE-SERVICES -m comment --comment "kubernetes service nodeports; NOTE: this must be the last rule in this chain" -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS
123 -A KUBE-NODEPORTS -p tcp -m comment --comment "default/test-nodeport:test-nodeport" -m tcp --dport 47565 -j KUBE-SVC-AODHVUMZDDCZ4Z2P

310 -A KUBE-SVC-AODHVUMZDDCZ4Z2P -m comment --comment "default/test-nodeport:test-nodeport" -j KUBE-SEP-PJWVY5FQJPODECLS
206 -A KUBE-SEP-PJWVY5FQJPODECLS -p tcp -m comment --comment "default/test-nodeport:test-nodeport" -m tcp -j DNAT --to-destination 172.244.166.141:6443
```

**iptables 缺点：**

- 每次对规则进行匹配时都会遍历 `iptables`中的所有 Service 链
- 无法进行增量式更新

大规模集群中使用 `iptables` 作为代理模式存在严重的性能问题，是完全不可用的。

**IPVS 代理模式**

![image-20220410225138842](https://s2.loli.net/2022/04/10/vIX23AqHhgZDoy1.png)

- `kube-proxy` 监听 `kube-api-server` 以获得添加和移除 Service / Endpoint 的事件
- kube-proxy 根据监听到的事件，调用 `netlink` 接口，创建 **IPVS** 规则；并且将 Service/Endpoint 的变化同步到 **IPVS** 规则中
- 当访问一个 Service 时，**IPVS** 将请求重定向到后端 Pod

![image-20220410225213489](https://s2.loli.net/2022/04/10/vz3s4mxSaHMqGP7.png)

**IPVS 优点**

IPVS 就是用于解决在大量 Service 时，`iptables` 规则同步变得不可用的性能问题。与 `iptables` 比较像的是，`ipvs` 的实现虽然也基于 `netfilter` 的钩子函数，但是它却使用哈希表作为底层的数据结构并且工作在内核态，这也就是说 IPVS 在重定向流量和同步代理规则有着更好的性能。

**IPVS 提供更多的负载均衡选项：**

- `rr`：轮替（Round-Robin）
- `lc`：最少链接（Least Connection），即打开链接数量最少者优先
- `dh`：目标地址哈希（Destination Hashing）
- `sh`：源地址哈希（Source Hashing）
- `sed`：最短预期延迟（Shortest Expected Delay）
- `nq`：从不排队（Never Queue）






## References

- [一文讲清服务发现和负载均衡](https://bbs.huaweicloud.com/blogs/193876)
- [深入浅出负载均衡](http://blog.itpub.net/69912579/viewspace-2775774)
- [使用 LVS 实现负载均衡原理及安装配置详解](https://www.cnblogs.com/liwei0526vip/p/6370103.html)
- [详解 Kubernetes Service 的实现原理](https://draveness.me/kubernetes-service/)