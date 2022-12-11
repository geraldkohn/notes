## k8s 基本架构

![](https://img2023.cnblogs.com/blog/2761052/202212/2761052-20221211215621004-722439029.png)

Master Node：控制面的节点

Worker Node：干活的节点

## Master 中的组件

API Server：API Server 是整个 k8s 集群的唯一入口。

Etcd：持久化存储，存储集群中各种资源的状态。

Controller Manager：负责实现故障检测，服务迁移，应用伸缩等，相当于一个运维工具。

Kube Scheduler：负责容器的编排工作，将 Pod 调度到最适合的节点上。

## Node 中的组件

Kubelet：Node 的代理，与 Master 中的 API Server 通信，管理 Node 的绝大部分操作。

Kube-proxy：代理容器网络之间的通信。

container-runtime：Docker 是一种 container-runtime，是干活的。


