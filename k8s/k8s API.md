## Job-CronJob

* Job: 临时离线任务

* CronJob: 定时离线任务

### Yaml 文件描述 Job:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: ...
  labels:
    k: v
    k: v
    ...

spec:
  template: 
    # Pod 模板
    spec:
       ...
       containers:
         - ...
         - ...
```

### Yaml 文件描述 CronJob:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "* * * * *"
  jobTemplate: 
    # Job 模板
    spec:
      template:
        # Pod 模板
        spec:
          containers:
          - name: hello
            image: busybox:1.28
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

CronJob 时间表语法:

```
# ┌───────────── 分钟 (0 - 59)
# │ ┌───────────── 小时 (0 - 23)
# │ │ ┌───────────── 月的某天 (1 - 31)
# │ │ │ ┌───────────── 月份 (1 - 12)
# │ │ │ │ ┌───────────── 周的某天 (0 - 6)（周日到周一；在某些系统上，7 也是星期日）
# │ │ │ │ │                          或者是 sun，mon，tue，web，thu，fri，sat
# │ │ │ │ │
# │ │ │ │ │
# * * * * *
```

| 输入                    | 描述                | 相当于       |
| --------------------- | ----------------- | --------- |
| @yearly (或 @annually) | 每年 1 月 1 日的午夜运行一次 | 0 0 1 1 * |
| @monthly              | 每月第一天的午夜运行一次      | 0 0 1 * * |
| @weekly               | 每周的周日午夜运行一次       | 0 0 * * 0 |
| @daily (或 @midnight)  | 每天午夜运行一次          | 0 0 * * * |
| @hourly               | 每小时的开始一次          | 0 * * * * |

例如，下面这行指出必须在每个星期五的午夜以及每个月 13 号的午夜开始任务：

`0 0 13 * 5`

要生成 CronJob 时间表表达式，你还可以使用 [crontab.guru](https://crontab.guru/) 之类的 Web 工具。

## ConfigMap-Secret

* ConfigMap: 保存明文配置信息

* Secret: 保存密文配置信息

### Yaml 文件描述 ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo 

data:
  # 类属性键；每一个键都映射到一个简单的值
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"

  # 类文件键
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5    
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true 
```

你可以使用四种方式来使用 ConfigMap 配置 Pod 中的容器：

1. 在容器命令和参数内
2. 容器的环境变量
3. 在只读卷里面添加一个文件，让应用来读取
4. 编写代码在 Pod 中运行，使用 Kubernetes API 来读取 ConfigMap

这些不同的方法适用于不同的数据使用方式。 对前三个方法，kubelet 使用 ConfigMap 中的数据在 Pod 中启动容器。

第四种方法意味着你必须编写代码才能读取 ConfigMap 和它的数据。然而， 由于你是直接使用 Kubernetes API，因此只要 ConfigMap 发生更改， 你的应用就能够通过订阅来获取更新，并且在这样的情况发生的时候做出反应。 通过直接进入 Kubernetes API，这个技术也可以让你能够获取到不同的名字空间里的 ConfigMap。

下面是一个 Pod 的示例，它通过使用 `game-demo` 中的值来配置一个 Pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: demo
      image: alpine
      command: ["sleep", "3600"]
      env:
        # 定义环境变量
        - name: PLAYER_INITIAL_LIVES # 请注意这里和 ConfigMap 中的键名是不一样的
          valueFrom:
            configMapKeyRef:
              name: game-demo           # 这个值来自 ConfigMap
              key: player_initial_lives # 需要取值的键
        - name: UI_PROPERTIES_FILE_NAME
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: ui_properties_file_name
      volumeMounts:
      - name: config
        mountPath: "/config"
        readOnly: true
  volumes:
  # 你可以在 Pod 级别设置卷，然后将其挂载到 Pod 内的容器中
  - name: config
    configMap:
      # 提供你想要挂载的 ConfigMap 的名字
      name: game-demo
      # 来自 ConfigMap 的一组键，将被创建为文件
      items:
      - key: "game.properties"
        path: "game.properties"
      - key: "user-interface.properties"
        path: "user-interface.properties"
```

被挂载的 ConfigMap 内容会被自动更新, 以环境变量方式使用的 ConfigMap 数据不会被自动更新, 更新这些数据需要重新启动 Pod.

### Yaml 文件描述 Secret:

如果你希望在 Pod 中访问 Secret 内的数据，一种方式是让 Kubernetes 将 Secret 以 Pod 中一个或多个容器的文件系统中的文件的形式呈现出来。

要配置这种行为，你需要：

1. 创建一个 Secret 或者使用已有的 Secret。多个 Pod 可以引用同一个 Secret。
2. 更改 Pod 定义，在 `.spec.volumes[]` 下添加一个卷。根据需要为卷设置其名称， 并将 `.spec.volumes[].secret.secretName` 字段设置为 Secret 对象的名称。
3. 为每个需要该 Secret 的容器添加 `.spec.containers[].volumeMounts[]`。 并将 `.spec.containers[].volumeMounts[].readOnly` 设置为 `true`， 将 `.spec.containers[].volumeMounts[].mountPath` 设置为希望 Secret 被放置的、目前尚未被使用的路径名。
4. 更改你的镜像或命令行，以便程序读取所设置的目录下的文件。Secret 的 `data` 映射中的每个主键都成为 `mountPath` 下面的文件名。

下面是一个通过卷来挂载名为 `mysecret` 的 Secret 的 Pod 示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      optional: false # 默认设置，意味着 "mysecret" 必须已经存在
```

你要访问的每个 Secret 都需要通过 `.spec.volumes` 来引用。

如果 Pod 中包含多个容器，则每个容器需要自己的 `volumeMounts` 块， 不过针对每个 Secret 而言，只需要一份 `.spec.volumes` 设置。

以环境变脸的方式使用 Secret:

1. 创建 Secret（或者使用现有 Secret）。多个 Pod 可以引用同一个 Secret。
2. 更改 Pod 定义，在要使用 Secret 键值的每个容器中添加与所使用的主键对应的环境变量。 读取 Secret 主键的环境变量应该在 `env[].valueFrom.secretKeyRef` 中填写 Secret 的名称和主键名称。
3. 更改你的镜像或命令行，以便程序读取环境变量中保存的值。

下面是一个通过环境变量来使用 Secret 的示例 Pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  restartPolicy: Never
  containers:
  - name: mycontainer
    image: redis
    env:
      - name: SECRET_USERNAME
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: username
            optional: false # 此值为默认值；意味着 "mysecret"
                            # 必须存在且包含名为 "username" 的主键
      - name: SECRET_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: password
            optional: false # 此值为默认值；意味着 "mysecret"
                            # 必须存在且包含名为 "password" 的主键 
```

## Deployment

Deployment: 部署无状态在线业务

Yaml 文件描述 Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: ngx-dep
  name: ngx-dep

spec:
  replicas: 2    # 副本数量
  selector:      # 用于筛选要被 Deployment 管理的 Pod 对象
    matchLabels: # labels 用于筛选 Deployment 作用对象(标签k-v相同)
      k: v

  # Pod 模板, 以下可以写在其他 Pod YAML 里面, 是松耦合的关系
  template:
    metadata: # 元信息
      labels: # 标签
        k: v  
    spec:
      containers:
      - image: nginx:alpine
        name: nginx
```

Deployment 和 Pod 实际上是一对松散的关系, Deployment 唯一的作用就是让 Pod 保持确定的副本数量, Deployment 并不持有 Pod.

## Daemonset

> 在每一个 Worker Node 上都运行一个 Pod

Yaml 文件描述 Daemonset:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: redis-ds
  labels:
    app: redis-ds

spec:
  selector: # 用于筛选 Daemonset 管理的 Pod 对象
    matchLabels: # 标签匹配
      name: redis-ds

  # Pod 的模板, 可以写在 Pod YAML 文件中.
  template:
    metadata:
      labels:
        name: redis-ds
    spec:
      containers:
      - image: redis:5-alpine
        name: redis
        ports:
        - containerPort: 6379
```

介绍两个名词, 为了应对 Pod 在某些节点的调度与驱逐的问题, k8s 定义了两个新的名词: 污点(taint), 容忍度(toleration)

* taint: 是 k8s 节点的一个属性, 本质上是给节点贴标签

* toleration: 是 Pod 的一个属性, 是一个数组, 本质上是给 Pod 贴标签

[污点和容忍度 | Kubernetes](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/taint-and-toleration/)

## Service

首先复习一下 OSI 七层模型:

- 应用层 --- HTTP/HTTPS

- 表示层

- 会话层

- 传输层 --- TCP/UDP

- 网络层 --- IP (路由器)

- 数据链路层 --- MAC (网关)

- 物理层

它本质上就是一个由 kube-proxy 控制的四层负载均衡，在 TCP/IP 协议栈上转发流量.

![](https://img2023.cnblogs.com/blog/2761052/202212/2761052-20221211215653658-1515099030.png)

Service 使用 iptables 技术,  每个节点上的 kube-proxy 组件自动维护 iptables 规则，客户不再关心 Pod 的具体地址，只要访问 Service 的固定 IP 地址，Service 就会根据 iptables 规则转发请求给它管理的多个 Pod，是典型的负载均衡架构。

Yaml 描述 Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ngx-svc
  
spec:
  type: ClusterIP    # Service 只能在集群内部访问
  type: NodePort     # Service 可以在集群外部访问

  selector:          # 筛选器, 筛选出被 Service 管理的 Pod
    app: ngx-dep
    
  ports:
  - port: 80         # 外部端口(容器外部, 集群中)
    targetPort: 80   # 内部端口(容器内部)
    protocol: TCP    # 使用的协议
```

通过访问 Service 的 IP 地址, Service 就能将流量转发到管理的 Pod 身上. Service 的 IP 地址是虚拟的, 所以无法 ping 通. 

可以通过 URL 地址来访问 Service:  (serviceName).(namespace).svc.cluster.local  serviceName 是名字, namespace 是这个 Service 所在的命名空间.

如果想要将 Service 暴露给集群外部来访问, 那么将 type 字段设置为 NodePort, 意思是在部署 Service 节点上暴露出端口给集群外部访问. type 字段的默认值是 ClusterIP, 意思是只能在集群内部访问, 节点上没有对外暴露端口. 如果将 Service 设置为集群外部可以访问, 那么除了在集群内可以通过 Service IP 访问外, k8s 还在集群外部随机提供一个端口, 比如 35651(随机), 通过 WorkerNode IP + 35651 就可以在集群外部访问 Service.

## Ingress/IngressClass

首先复习一下 OSI 七层模型:

* 应用层 --- HTTP/HTTPS

* 表示层

* 会话层 

* 传输层 --- TCP/UDP

* 网络层 --- IP (路由器)

* 数据链路层 --- MAC (网关)

* 物理层

Service 只在 TCP/IP 协议上转发流量, 是一个四层的负载均衡协议, 功能有限, 只能依据 IP 地址 + 端口号来做一些逻辑判断和组合, 而我们据大多数的应用都是跑在七层协议上 --- HTTP/HTTPS, 有更多的高级路由条件: 请求头, URL 等. 这些在 TCP/IP  四层协议中是看不见的.

Service 暴露集群外部服务只用 NodePort 这种方式, 必然不可以.

Ingress:

* 七层负载均衡

* 作为流量的总入口，统管集群的进出口数据，“扇入”“扇出”流量（也就是我们常说的“南北向”）

![](https://img2023.cnblogs.com/blog/2761052/202212/2761052-20221211215654426-677515753.png)

Service 本身是没有服务能力的, 他只是一些 iptables, 应用这些规则的是节点中的 kube-proxy 组件. 同样的, Ingress 只是一些 HTTP 的路由集合, 相当于一份静态描述文件, 需要 Ingress Controller 才能正确的处理流量.

k8s 将 Ingress Controller 的实现交给社区, 有许多实现最著名的就是 Nignx. 以下都是 Nignx 二次开发变成 Ingress Controller 的项目:

* 社区的 Kubernetes Ingress Controller https://github.com/kubernetes/ingress-nginx

* Nginx 公司自己的 Nginx Ingress Controller https://github.com/nginxinc/kubernetes-ingress

* 基于 OpenResty 的 Kong Ingress Controller https://github.com/Kong/kubernetes-ingress-controller

这里介绍使用最多的 Nignx 公司自己的 Nginx Ingress Controller

![](https://img2023.cnblogs.com/blog/2761052/202212/2761052-20221211215655163-958930498.png)

只有一个 Ingress 会出现问题:

* 由于某些原因，项目组需要引入不同的 Ingress Controller，但 Kubernetes 不允许这样做；

* Ingress 规则太多，都交给一个 Ingress Controller 处理会让它不堪重负；

* 多个 Ingress 对象没有很好的逻辑分组方式，管理和维护成本很高；

* 集群里有不同的租户，他们对 Ingress 的需求差异很大甚至有冲突，无法部署在同一个 Ingress Controller 上。

所以提出了 Ingress Class, 基于逻辑进行分组. 每个 Ingress Class 都有若干个 Ingress.

![](https://img2023.cnblogs.com/blog/2761052/202212/2761052-20221211215655739-1102632819.png)

### Yaml 文件描述 Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ngx-ing
  
spec:

  ingressClassName: ngx-ink # 指定 Ingress 所属的 Ingress Class
  
  rules: # 路由规则
  - host: ngx.test # host 名字
    http:
      paths: # URLs
      - path: / # URL
        pathType: Exact
        backend: # 负责处理的 Service
          service:
            name: ngx-svc
            port:
              number: 80
```

### Yaml 文件描述 Ingress Class

Ingress Class 本身并没有什么实际的功能，只是起到联系 Ingress 和 Ingress Controller 的作用，所以它的定义非常简单，在“spec”里只有一个必需的字段“controller”，表示要使用哪个 Ingress Controller，具体的名字就要看实现文档了.

如果我要用 Nginx 开发的 Ingress Controller，那么就要用名字“nginx.org/ingress-controller”. 

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: ngx-ink

spec:
  controller: nginx.org/ingress-controller
```

### Yaml 部署 Ingress Controller

Nignx Ingress Controller https://github.com/nginxinc/kubernetes-ingress

因为它是以 Pod 的形式运行在 k8s 中, 所以同时支持 Deployment 和 DaemonSet 两种部署方式. 这里选择 Deployment 进行部署.

部署方式网络有介绍, 这里不赘述了.

## PV/PVC/StorageClass

PV 是从 k8s 的 Volume 对数据存储延伸出来的, Volumes 和 VolumeMounts, 相当于给 Pod 挂载了一个虚拟盘, 将配置信息以文件的形式注入到 Pod 中供进程使用.

PV 实际上就是一些存储设备的抽象: 文件系统: Ceph, GlusterFS, NFS, 甚至是裸盘. 

PV 不能直接提供给 Pod, 所以引入了 PVC 和 StorageClass.

PVC 是给 Pod 使用的对象, 用来向 k8s 申请 Pod 所要使用的资源. 一旦资源申请成功, k8s 就会将 PV 和 PVC 绑定在一起, 这个动作叫做 " 绑定 ".

进一步, 如果 PVC 直接去找 PV 也很麻烦, 我们就将一类 PV 组合起来, 构成 StorageClass.

![](https://img2023.cnblogs.com/blog/2761052/202212/2761052-20221211215656339-48401220.png)

### Yaml 文件描述 PV

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: host-10m-pv

spec:
  storageClassName: host-test # PV 所属的 StorageClass
  accessModes: # 存储设备的访问模式, 有三种
    # ReadWriteOnce：存储卷可读可写，但只能被一个节点上的 Pod 挂载。
    # ReadOnlyMany：存储卷只读不可写，可以被任意节点上的 Pod 多次挂载。
    # ReadWriteMany：存储卷可读可写，也可以被任意节点上的 Pod 多次挂载。
  - ReadWriteOnce
  capacity: # 容量
    storage: 10Mi
  hostPath: # 存储卷的本地路径, WorkerNode 的路径 
    path: /tmp/host-10m-pv/
```

### Yaml 文件描述 PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: host-5m-pvc

# 使用 StorageClassName 和 requests 来找 PV
spec:
  storageClassName: host-test # 在 StorageClass 中找 PV
  accessModes: # 期望存储设备的访问模式
    # ReadWriteOnce：存储卷可读可写，但只能被一个节点上的 Pod 挂载。
    # ReadOnlyMany：存储卷只读不可写，可以被任意节点上的 Pod 多次挂载。
    # ReadWriteMany：存储卷可读可写，也可以被任意节点上的 Pod 多次挂载。
    - ReadWriteOnce
  resources: 
    requests: # 需要的容量
      storage: 5Mi
```

### Pod 挂载 PVC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: host-pvc-pod

spec:
  volumes: # 挂载卷
  - name: host-pvc-vol # PVC 名字
    persistentVolumeClaim: # PVC
      claimName: host-5m-pvc # 给挂载卷起个名字

  containers:
    - name: ngx-pvc-pod
      image: nginx:alpine
      ports:
      - containerPort: 80
      volumeMounts: # 这个容器挂载哪个或哪几个卷
      - name: host-pvc-vol # 挂载卷的名字
        mountPath: /tmp # 挂载路径
```

## 网络存储

PV PVC StorageClass 只能将存储挂载到 Worker Node 上, 由于 Pod 经常变换位置, 所以需要一个网络存储来挂载虚拟卷. 这样无论 Pod 在哪一个 Worker Node 上, 都可以访问卷.

网络存储是一个比较热门的领域, 有许多知名的产品, 比如 AWS, Ceph, Azure. k8s 还专门定义了 CSI (Container Storage Interface) 规范.

### PV + NFS

NFS 作为一个经典的网络存储系统，有着近 40 年的发展历史，基本上已经成为了各种 UNIX 系统的标准配置，Linux 自然也提供对它的支持。

NFS 采用的是 Client/Server 架构，需要选定一台主机作为 Server，安装 NFS 服务器，其他要使用存储的主机作为 Client，安装 NFS 客户端工具。

所以接下来，我们在自己的 Kubernetes 集群里再增添一台名字叫 Storage 的服务器，在上面安装 NFS，实现网络存储、共享网盘的功能。不过这台 Storage 也只是一个逻辑概念，我们在实际安装部署的时候完全可以把它合并到集群里的某台主机里。

#### 部署 NFS 服务器

待更新

#### 部署 NFS 客户端

待更新

#### Pod + PVC + PV + NFS

![](https://img2023.cnblogs.com/blog/2761052/202212/2761052-20221211215657180-1869925805.png)

#### 自动化创建 PV

NFS Provisoner 待更新。动态存储卷不需要手工定义 PV，而是要定义 StorageClass，由关联的 Provisioner 自动创建 PV 完成绑定。

![](https://img2023.cnblogs.com/blog/2761052/202212/2761052-20221211215657830-125467036.png)

## StatefulSet

有状态应用: 运行状态很重要, 不能够丢失. 有状态应用相比于无状态应用在调度的过程总有三个难点需要解决:

1. 启动顺序

2. 自身身份

3. 网络标识

Yaml 文件描述 StatefulSet:

```yaml
# Redis 的 StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-sts

spec:
  serviceName: redis-svc # Service 对象的名称
  replicas: 2 # 实例个数
  selector: # 筛选器
    matchLabels: # 根据标签筛选处理的 Pod
      app: redis-sts
# -------------------------------------------------------
  # Pod
  template:
    metadata:
      labels:
        app: redis-sts
    spec:
      containers:
      - image: redis:5-alpine
        name: redis
        ports:
        - containerPort: 6379
```

比如以上 YAML 会创建两个 Redis 实例, 一个的 Name 为 redis-sts-0, 一个的 Name 为 redis-sts-1. 然后 k8s 会通过名称来创建实例, 先创建 redis-sts-0, 再创建 redis-sts-1, 这样第一个问题 -- 启动顺序就解决了.

k8s 通过环境变量的方式使得容器确定自身身份:

```bash
~#: kubectl exec -it redis-sts-0 -- sh
~#: echo $HOSTNAME
redis-sts-0
~#: hostname
redis-sts-0
```

在 Pod 中查看环境变量 $HOSTNAME 或者是执行命令 hostname, 都可以得到这个 Pod 的名字 redis-sts-0. 有了这个唯一的名字, 应用就可以确定依赖关系了. 这样第二个问题 -- Pod 自身身份, 就解决了.

### StatefulSet 集群内部网络标识

使用 Service 对象实现集群内部四层通信, 下面是写好的 Service 对象:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-svc # 必须与 StatefulSet 对象中的 serviceName 相同

spec:
  clusterIP: None # 不必为当前 Service 分配 IP 地址
  selector:
    app: redis-sts # 筛选标签相同的 Pod

  ports:
  - port: 6379
    protocol: TCP
    targetPort: 6379


```

 当将 Service 对象应用于 StatefulSet 的时候, Pod 的域名是: "Pod名.Service名.名字空间.svc.cluster.local".

上述 Redis Pod 的域名就是: 

* redis-sts-0.redis-svc.default.svc.cluster.local

* redis-sts-1.redis-svc.default.svc.cluster.loacl

StatefulSet 里的这两个 Pod 都有了各自的域名，也就是稳定的网络标识。那么接下来，外部的客户端只要知道了 StatefulSet 对象，就可以用固定的编号去访问某个具体的实例了，虽然 Pod 的 IP 地址可能会变，但这个有编号的域名由 Service 对象维护，是稳定不变的。

通过 StatefulSet 和 Service 的联合使用，Kubernetes 就解决了“有状态应用”的依赖关系、启动顺序和网络标识这三个问题，剩下的多实例之间内部沟通协调等事情就需要应用自己去想办法处理了。

Service 原本的目的是负载均衡，应该由它在 Pod 前面来转发流量，但是对 StatefulSet 来说，这项功能反而是不必要的，因为 Pod 已经有了稳定的域名，外界访问服务就不应该再通过 Service 这一层了。所以，从安全和节约系统资源的角度考虑，我们可以在 Service 里添加一个字段 clusterIP: None ，告诉 Kubernetes 不必再为这个对象分配 IP 地址。

StatefulSet 与 Service 对象的关系：

![](https://img2023.cnblogs.com/blog/2761052/202212/2761052-20221211215658449-996284304.png)

### StatefulSet 数据的持久化

为了强调持久化存储与 StatefulSet 的一对一绑定关系，Kubernetes 为 StatefulSet 专门定义了一个字段“volumeClaimTemplates”，直接把 PVC 定义嵌入 StatefulSet 的 YAML 文件里。这样能保证创建 StatefulSet 的同时，就会为每个 Pod 自动创建 PVC，让 StatefulSet 的可用性更高。

把刚才的 Redis StatefulSet 对象稍微改造一下，加上持久化存储功能：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-pv-sts

spec:
  # PVC
  volumeClaimTemplates:
  - metadata:
      name: redis-100m-pvc
    spec: 
      storageClassName: nfs-client
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 100Mi

  serviceName: redis-pv-svc
  replicas: 2
  selector:
    matchLabels:
      app: redis-pv-sts

  # Pod
  template:
    metadata:
      labels:
        app: redis-pv-sts
    spec:
      containers:
      - image: redis:5-alpine
        name: redis
        ports:
        - containerPort: 6379

        volumeMounts:
        - name: redis-100m-pvc
          mountPath: /data
```

首先 StatefulSet 对象的名字是 redis-pv-sts，表示它使用了 PV 存储。然后“volumeClaimTemplates”里定义了一个 PVC，名字是 redis-100m-pvc，申请了 100MB 的 NFS 存储。在 Pod 模板里用 volumeMounts 引用了这个 PVC，把网盘挂载到了 /data 目录，也就是 Redis 的数据目录。

下面的这张图就是这个 StatefulSet 对象完整的关系图：

![](https://img2023.cnblogs.com/blog/2761052/202212/2761052-20221211215659472-1056690096.png)

小结：

1. StatefulSet 的 YAML 描述和 Deployment 几乎完全相同，只是多了一个关键字段 serviceName。

2. 要为 StatefulSet 里的 Pod 生成稳定的域名，需要定义 Service 对象，它的名字必须和 StatefulSet 里的 serviceName 一致。

3. 访问 StatefulSet 应该使用每个 Pod 的单独域名，形式是“Pod 名. 服务名”，不应该使用 Service 的负载均衡功能。

4. 在 StatefulSet 里可以用字段“volumeClaimTemplates”直接定义 PVC，让 Pod 实现数据持久化存储。


