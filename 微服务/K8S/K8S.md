## Kubernetes

K8S 是一个工业级的容器编排平台。主要有以下一些特点：轻量级，消耗资源少；弹性伸缩；负载均衡。

K8S 是基于 Borg 开发的，Borg 的架构如如下图所示：

![Borg 架构图](http://img.programya.com/20200106233509.png)

首先是 BorgMaster 为了防止单点故障一般情况下都是以集群的形式出现，而且通常应该设置三个以上的奇数个 BorgMaster。Scheduler 调度器，有任务的的时候会写入到 Paxos中，下面的 Borglet 会监听读取 Paxos 内容，发现有自己的任务的时候回去处理。

下面看一下 K8S 的架构：

![K8S 架构](http://img.programya.com/20200107215538.png)

Scheduler 还是调度者的角色，会通过 api server 将数据写入到 etcd中。下面的 node 相当于 borglet 的角色真正的去处理任务。其中 replication controller 用于控制 api server 的副本数。api server 也是所有访问的入口。etcd 的官网将其它定位成一个可信赖的分布式键值存储服务，其能够为整个分布式存储集群存储一些关键数据，写出分布式集群的正常运转。node 中的 kubelet 直接跟容器引擎交互实现容器的生命周期管理。Kube-Proxy 负责写入规则到 IPTABLES，IPVS 实现服务的映射访问的。

ETCD 结构如下：

![ETCD 结构图](http://img.programya.com/20200107221104.png)

ETCD 使用 HTTP Server 作为入口，Raft 作为类似于 Paxos 是一个分布式一致性算法，WAL 是记录日志和快照的地方。

K8S 的常用的几个插件如下：

1. CoreDNS：可以为集群中的 SVC 创建一个域名 IP 的对应关系解析
2. Ingress Controller： Ingress 可以实现 7层的代理
3. Prometheus：可以提供一个可以跨集群中心多 K8S 统一管理功能
4. Dashboard：给 K8S 集群提供一个 B/S 结构的访问体系
5. Federation：提供 K8S  集群的监控能力
6. ELK：提供 K8S 集群日志统一分析接入平台





附录：

IAAS, PAAS 和 SAAS：

1. IAAS: Infrastructure as a service 基础设置即服务，例如阿里云。

2. PAAS: Platform as a service 平台即服务，例如Google App Engine。

3. SAAS: Software as a service 软件即服务， 例如 Netflix 等