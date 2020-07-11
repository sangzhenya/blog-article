---
title: "Kubernetes"
tags: ["Kubernetes"]
categories: ["微服务"]
date: "2020-01-25T09:00:00+08:00"
---

K8S 是一个工业级的容器编排平台。主要有以下一些特点：轻量级，消耗资源少；弹性伸缩；负载均衡。

K8S 是基于 Borg 开发的，Borg 的架构如如下图所示：

![Borg 架构图](http://img.programya.com/20200106233509.png)

首先是 BorgMaster 为了防止单点故障一般情况下都是以集群的形式出现，而且通常应该设置三个以上的奇数个 BorgMaster。Scheduler 调度器，有任务的的时候会写入到 Paxos中，下面的 Borglet 会监听读取 Paxos 内容，发现有自己的任务的时候回去处理。

### 基本概念

下面看一下 K8S 的架构：

![K8S 架构](http://img.programya.com/20200107215538.png)

Scheduler 还是调度者的角色，会通过 api server 将数据写入到 etcd中。下面的 node 相当于 borglet 的角色真正的去处理任务。其中 replication controller 用于控制容器的副本数。api server 也是所有访问的入口。etcd 的官网将其它定位成一个可信赖的分布式键值存储服务，其能够为整个分布式存储集群存储一些关键数据，写出分布式集群的正常运转。node 中的 kubelet 直接跟容器引擎交互实现容器的生命周期管理。Kube-Proxy 负责写入规则到 IPTABLES，IPVS 实现服务的映射访问的。

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

### POD

POD：是 K8S 管理，创建，计划的最小单元，一个共享context的配置组，同一个 POD 上的容器共享网络和存储。

ReplicationController 用来确保容器应用的副本数始终保持在用户定义的副本数，如果容器异常退出，会自动创建新的 POD 来踢动，而如果异常多出来的容器也会被自动回收。在新版本的 K8S  中建议使用 ReplicaSet 取代 ReplicationController，但是其二者没有本质上的不同，ReplicaSet 支持 集合式的 seletor。虽然 ReplicaSet 支持独立部署，但是还是推荐使用 Deployment 自动部署。

Horzontal Pod Autoscaling 仅使用与 Deployment 和 ReplicaSet，其支持根据 POD 的CPU 使用率扩缩容，在最新的版本中也支持根据内存和用户自定义的 metric 扩缩容。

StatefulSet 是为了解决有状态服务的问题，其应用场景主要有以下：

1. 稳定的持久化存储，即 POD 重新调度后还是能访问相同的持久化数据，基于 PVC 实现
2. 稳定的网络标志，即 POD 重新调度后其 PODName 和 HostName 不变，基于 Headless Service 实现
3. 有序部署，有序扩展：即 POD 是有顺序的，在部署或者扩展的时候根据定义的顺序依次进行，即在下一个 POD 运行之前所有之前的 POD 都必须是 Running 和 Ready 状态的，基于 Init Containers 来实现
4. 有序收缩，有序删除。

DaemonSet 确保全部或一些 Node 上运行的一个 POD 的副本。当有 Node 加入集群时，也会为其增加一个 POD。当有 Node 从 集群中移除时，这些 POD 也会被回收。删除 DaemonSet 将会删除它创建的所有 POD。通常用法如下：

1. 运行集群存储 Daemon，例如在每个 Node 上运行 glushterd, ceph
2. 在每个 Node 上运行日志手机 Daemon，例如 fluentd, logstas
3. 在每个 Node 上运行监控 Daemon，例如 Prometheus Node Exporter

Job 负责批处理任务，即仅执行一次任务，其保证批处理任务的一个或多个 POD 成功结束。

Cron Job 管理基于时间的 Job，即在给定时间点运行一次，周期性的在给定的时间点运行。

### 网络通讯模式

K8S 的网络模型假定了所有 POD 都在一个可以直接连通的扁平化网络空间中，者在 GCE 里面是现成的网络模型， K8S 假定这个网络已经存在，在私有云中搭建 K8S 集群的时候需要自己实现这个网络假设，将不同节点的上的 Docker 容器之间的相互访问先打通，然后运行 K8S。

同一个 POD 内多个容器之间共享同一个网络命名空间，共享 Linux 协议栈；各 POD 之间的通讯如果在同一个主机上，则由 Docker 网桥直接转发不经过 Flannel，如果在不同主机上则 POD 的 IP 和 Node 的 IP 关联起来，通过这个关联让 POD 之间可以互相访问（通过 Flannel 完成该操作）；POD 与 Service 之间的通讯基于性能考虑全部为 lvs 维护和转发。POD 访问外网，查找路由表转发数据包到宿主主机网卡，宿主主机网卡完成路由选择后，把源 IP 更改我宿主网卡 IP 然后向外网服务器发送请求。外网访问 POD 则通过 Service。

Flannel 是 CoreOS 团队针对于 K8S 设计的一个网络规划服务，其目的是让集群中的不同节点主机创建的 Docker 都具有全局唯一的虚拟 IP 地址。此外其还能再这些 IP 地址之间建立一个覆盖网络（Overlay Networ），通过这个覆盖网络，将数据包原封不动的传递到目标容器内。原理图如下：

![Flannel](http://img.programya.com/20200107234926.png)

ECTD 为 Flannel 提供存储管理 Flannel 可分配的 IP 地址段资源，监控 ETCD 中每个 POD 的实际地址，并在内存中建立维护 POD 节点路由表。



附录：

IAAS, PAAS 和 SAAS：

1. IAAS: Infrastructure as a service 基础设置即服务，例如阿里云。

2. PAAS: Platform as a service 平台即服务，例如Google App Engine。

3. SAAS: Software as a service 软件即服务， 例如 Netflix 等