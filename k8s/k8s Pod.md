## Pod 是 k8s 的核心对象

![](https://img2023.cnblogs.com/blog/2761052/202212/2761052-20221211215811011-1661801329.png)

![](https://img2023.cnblogs.com/blog/2761052/202212/2761052-20221211215811810-452791639.png)

Pod 是在 k8s 中创造，管理的最小可部署计算单元。

通常不需要直接创建 Pod，k8s 集群中 Pod 主要有两种用法：

* 运行单个容器的 Pod

* 运行包含多个容器的 Pod，Pod 中的容器紧密管理，必须放在一起。Pod 包含多个容器的时，Pod 作为一个整体来被调度。比如有些 Pod 里面有多个 init 容器和应用容器。

一个简单的 Pod 模板如下（官方文档）

```yaml
apiVersion: ...
kind: ...
metadata:
  name: hello 
  labels: 
    k: v
    k: v

spec:
  template:
    # 这里是 Pod 模板
    spec:
      restartPolicy: OnFailure
      containers:
      - name: hello
        image: busybox:1.28
        imagePullPolicy: IfNotPresent
        ports: 12345
        env: 
          - name: os
            value: "ubuntu"
          - name: debug
            value: "on"
        command: ['sh', '-c', 'echo "Hello, Kubernetes!" && sleep 3600']
    # 以上为 Pod 模板
```

template 里面定义了 Job 的 Pod 。

如果要更新 Pod，那么需要使用更新后的模板来创建 Pod。这样才能触发更新操作。

## 字段解释

* containers:

```yaml
name: 容器名字
ports: 暴露的端口号
env: 定义容器的环境，与Docker中env指令类似；形式：k-v list
imagePullPolicy: 容器拉取策略
command: 容器启动时要运行的命令, 相当于 Docker 中的 entrypoint 指令
args: command 运行时的参数, 相当于 Docker 中的 cmd 指令
```

* 其他

```yaml
restartPolicy: 容器重启策略
```

## 静态 Pod

静态 Pod 不受 k8s 的管控, 所以叫做静态的. 但是由于它是 Pod, 所以也要跑在容器运行时上, 也会有 YAML 文件来描述它, 而唯一能够管理它的 Kubernetes 组件也就只有在每个节点上运行的 kubelet 了. 

“静态 Pod”的 YAML 文件默认都存放在节点的 /etc/kubernetes/manifests 目录下，它是 Kubernetes 的专用目录.

你可以看到，Kubernetes 的 4 个核心组件 apiserver、etcd、scheduler、controller-manager 原来都以静态 Pod 的形式存在的，这也是为什么它们能够先于 Kubernetes 集群启动的原因.

## 平滑更新
