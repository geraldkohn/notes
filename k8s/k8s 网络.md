## Docker 网络

Docker 网络是通过 Bridge 来实现的，每个容器都会创建一个虚拟网卡对（Veth Pair），一个插在容器里，一个插在 docker0 这个网桥上（交换机），这样容器之间就可以互相通信了。

Docker 跨主机通信是通过 VXLAN 来实现的，Virtual Extensible LAN，每个宿主机会创建一个虚拟网络设备 VTEP （VXLAN Tunnel End Point），这个网络设备可以将 docker0 的 IP 包进行二次封装（加上 VTEP 的 MAC 地址），完全由 Linux 内核进行封装和解封装，不需要经过用户态。docker0 和 VTEP 都是基于 MAC 地址进行转发的，属于交换机。

## Kubernetes 网络

由于 Docker 跨主机网络需要做复杂的地址转换，所以 k8s 提出了一个自己的网络模型 ”IP-per-pod“，能够很好的适应集群的网络需求。它有如下四点基本要求：

* 集群里的每个 Pod 都会有唯一的一个 IP 地址。

* Pod 里的所有容器共享这个 IP 地址。

* 集群里的所有 Pod 都属于同一个网段。

* Pod 直接可以基于 IP 地址直接访问另一个 Pod，不需要做麻烦的网络地址转换（NAT）。

网络模型图：

![](https://img2023.cnblogs.com/blog/2761052/202212/2761052-20221213120833929-1724148646.png)

因为 Pod 都具有独立的 IP 地址，相当于一台虚拟机，而且直连互通，也就可以很容易地实施域名解析、负载均衡、服务发现等工作，以前的运维经验都能够直接使用，对应用的管理和迁移都非常友好。

## CNI

CNI 就是实现上述网络模型的标准：Container Networking Interface。

依据实现技术的不同，CNI 插件可以大致上分成“Overlay”“Route”和“Underlay”三种。

* Overlay 的原意是“覆盖”，是指它构建了一个工作在真实底层网络之上的“逻辑网络”，把原始的 Pod 网络数据封包，再通过下层网络发送出去，到了目的地再拆包。因为这个特点，它对底层网络的要求低，适应性强，缺点就是有额外的传输成本，性能较低。

* Route 也是在底层网络之上工作，但它没有封包和拆包，而是使用系统内置的路由功能来实现 Pod 跨主机通信。它的好处是性能高，不过对底层网络的依赖性比较强，如果底层不支持就没办法工作了。

* Underlay 就是直接用底层网络来实现 CNI，也就是说 Pod 和宿主机都在一个网络里，Pod 和宿主机是平等的。它对底层的硬件和网络的依赖性是最强的，因而不够灵活，但性能最高。

### 选型

Flannel（ https://github.com/flannel-io/flannel/ ）是最早的一种 Overlay 模式的网络插件，使用 UDP 和 VXLAN 技术。Flannel 简单易用，是 Kubernetes 里最流行的 CNI 插件，但它在性能方面表现不是太好，所以一般不建议在生产环境里使用。

Calico（ https://github.com/projectcalico/calico ）是一种 Route 模式的网络插件，使用 BGP 协议（Border Gateway Protocol）来维护路由信息，性能要比 Flannel 好，而且支持多种网络策略，具备数据加密、安全隔离、流量整形等功能。

Cilium（ https://github.com/cilium/cilium ）是一个比较新的网络插件，同时支持 Overlay 模式和 Route 模式，它的特点是深度使用了 Linux eBPF 技术，在内核层次操作网络数据，所以性能很高，可以灵活实现各种功能。在 2021 年它加入了 CNCF，成为了孵化项目，是非常有前途的 CNI 插件。

### CNI 插件工作方式

Fannel 插件工作方式如图，这也是 Docker 跨主通信的方式：

![](https://img2023.cnblogs.com/blog/2761052/202212/2761052-20221213120834928-372382666.png)

Calico 插件的工作方式：

可以在 Calico 的网站（ https://www.tigera.io/project-calico/ ）上找到它的安装方式，我选择的是“本地自助安装（Self-managed on-premises）”，可以直接下载 YAML 文件：

```yaml
wget https://projectcalico.docs.tigera.io/manifests/calico.yaml
```

Calico 的安装非常简单，只需要用 kubectl apply 就可以（记得安装之前最好把 Flannel 删除）：

```bash
kubectl apply -f calico.yaml
```

安装之后查看一下 Calico 的运行状态，它是在 kube-system 的 Namespace 中。

```bash
kubectl get pod -n kube-system
```

![](https://img2023.cnblogs.com/blog/2761052/202212/2761052-20221213120835387-1108630281.png)

我们创建三个 Nginx 来进行实验：

```bash
kubectl create deploy ngx-dep --image=nginx:alpine --replicas=3
```

我们看到了它们的 IP 地址，分别是 10.10.171.* ，10.10.219.*

![](https://img2023.cnblogs.com/blog/2761052/202212/2761052-20221213120835865-977217183.png)

然后我们来看看 Pod 里的网卡情况，你会发现虽然还是有虚拟网卡，但宿主机上的网卡名字变成了 calica17a7ab6ab@if4，而且并没有连接到“cni0”网桥上：

![](https://img2023.cnblogs.com/blog/2761052/202212/2761052-20221213120836813-1485513433.png)

这是 Calico 的工作模式导致的正常现象。因为 Calico 不是 Overlay 模式，而是 Route 模式，所以它就没有用 Flannel 那一套，而是在宿主机上创建路由规则，让数据包不经过网桥直接“跳”到目标网卡去。

来看一下节点上的路由表就能明白：

![](https://img2023.cnblogs.com/blog/2761052/202212/2761052-20221213120838105-936520179.png)

假设 Pod A“10.10.219.67”要访问 Pod B“10.10.219.68”，那么查路由表，知道要走“cali051dd144e34”这个设备，而它恰好就在 Pod B 里，所以数据就会直接进 Pod B 的网卡，省去了网桥的中间步骤。

Calico 的网络架构示意图，你可以再对比 Flannel 来学习：

![](https://img2023.cnblogs.com/blog/2761052/202212/2761052-20221213120839157-415497667.png)

至于在 Calico 里跨主机通信是如何路由的，你完全可以对照着路由表，一步步地“跳”到目标 Pod 去（提示：tunl0 设备）。

tunl0 设备待更新 ~~

小结：

* Kubernetes 使用的是“IP-per-pod”网络模型，每个 Pod 都会有唯一的 IP 地址，所以简单易管理。

* CNI 是 Kubernetes 定义的网络插件接口标准，按照实现方式可以分成“Overlay”“Route”和“Underlay”三种，常见的 CNI 插件有 Flannel、Calico 和 Cilium。

* Flannel 支持 Overlay 模式，它使用了 cni0 网桥和 flannel.1 设备，本机通信直接走 cni0，跨主机通信会把原始数据包封装成 VXLAN 包再走宿主机网卡发送，有性能损失。

* Calico 支持 Route 模式，它不使用 cni0 网桥，而是创建路由规则，把数据包直接发送到目标网卡，所以性能高。
