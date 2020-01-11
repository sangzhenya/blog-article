## Eureka

Eureka 是 Netflix 的一个子模块，也是核心模块之一。Eureka 是一个基于 REST 的服务，用于定位服务，实现云端中间层服务发现与故障转移。服务注册与发现对于微服务是非常的重要的，有了服务注册与发现只需要使用服务的标识符，就昆虫访问到服务，而不需要修改服务调用的配置文件。功能和 Dubbo 的注册中心类似。Eureka 遵守的 CAP 原则中的 AP 原则。

Spring Cloud 封装了 Netflix 的 Eureka 模块来实现服务注册发现。Eureka 采用 CS 架构，Eureka Server 作为服务注册功能的服务器，是服务注册的中心。系统中的其他微服务使用 Eureka Client 连接从 Eureka Server 并维持心跳。所以可以通过 Eureka 监控系统中的各个微服务是否正常运行。Spring Cloud 一些其他模块就可以通过 Eureka Server 来发现系统中的其他微服务并执行做相关的逻辑。如下图所示：

![Eureka Server 架构](http://img.programya.com/20200109230019.png)

Eureka 包含两个组件：Eureka Server 和 Eureka Client。其中 Eureka Server 提供服务注册服务，各个节点启动后会在 Eureka Server 中进行注册，这样 Eureka Server 中的服务注册表中将会存储所有可用服务节点的信息，服务节点的信息可以在见面中直观的看到。Eureka Client 是客户端，用于简化和 Eureka Server 的交互，客户端同时也具备一个内置的，基于轮询的负载均衡算法的负载均衡器。在应用启动后想 Eureka Server 发送心跳，默认是 30s。如果 Eureka Server 在多个心跳周期内没有接收的某个节点的心跳，那么 Eureka Server 会从服务注册列表中将该服务节点删除，默认是 90s。

Eureka 的自我保护机制：默认情况下，如果 Eureka 在一定时间内没有收到某个微服务实例的心跳，EurekaServer 将会注销该实例。但是的网络分区故障发生时，微服务与 EurekaServer 之间无法正常通讯，那么这个行为就可能变得危险，因为微服务本身其实是健康的，此时本不应该注销这个微服务。Eureka 通过自我保护模式来解决这个问题。当 EurekaServer 在短的时间内丢失过多的客户端，那么这个节点进入自我保护模式。一旦进入该模式，EurekaServer 就会保护注册表中信息，不再删除服务注册表中的数据。当网络故障恢复后，该 EurekaServer 会自动退出自我保护机制。

自我保护模式是一种应对网络异常的安全保护措施，其架构哲学是宁可同时保留所有微服务，也不盲目注销任何健康的微服务，使用自我保护模式，可以让 Eureka 集群更加健康稳定。

对于注册进 Eureka 里面的微服务，可以通过服务发现来获取该服务的信息。

启动 Eureka 后打开管理页面如下：

![Eureka](http://img.programya.com/20200111084730.png)

System Status： 就是当前环境信息和系统状态。

DS Replicas：集群环境下 Eureka 启动节点。

Instances currently registered with Eureka： 当前注册在 Eureka 的微服务信息。

General Info：当前环境的其他信息。