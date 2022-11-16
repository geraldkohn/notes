# Docker 常见面试题

## 隔离与限制

Linux 通过 Namespace 技术，只让一个进程只看见自己的进程空间，不能看见操作系统的进程空间。Namespace 包括：PID，Mount，UTS，IPC，Network，User Namespace。比如，Mount Namespace，用于让被隔离进程只看到当前 Namespace 里的挂载点信息；Network Namespace，用于让被隔离进程看到当前 Namespace 里的网络设备和配置。

Linux Cgroups （Linux Control Group）它最主要的作用，就是限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等。 

一个正在运行的 Docker 容器，其实就是一个启用了多个 Linux Namespace 的应用进程，而这个进程能够使用的资源量，则受 Cgroups 配置的限制。容器是一个“单进程”模型。

## 容器与虚拟机的区别

虚拟机是虚拟化一套完整的操作系统，而容器是共用底层的操作系统，实际上容器只是运行在宿主机上的一个进程。

## 文件系统

可以使用 `chroot 目录 进程` 来指定该进程的根目录。Mount Namespace 就是基于 chroot 不断改良，为了让容器这个根目录看起来更真实，我们一般会在这个根目录下挂载一个完整的 Linux 文件系统，这个文件系统就叫 `容器镜像` 或者 `rootfs(根文件系统)` 。

rootfs 只包括 Linux 的文件系统，不包括 Linux 内核，因为同一台宿主机上的容器进程，共享操作系统内核。rootfs 打包的不止是应用，而是整个操作系统的文件和目录，所以应用和应用所需要的依赖，都被封装在一起。

```bash
$ ls /
bin dev etc home lib lib64 mnt opt proc root run sbin sys tmp usr var
```

Docker 是如何创建容器的？

1. 启动 Linux Namespace 配置

2. 设置 Linux Cgroups 参数

3. 切换进程根目录

## Union File System

将多个不同位置的目录联合挂载（union mount）到同一个目录下。

```
$ tree
.
├── A
│  ├── a
│  └── x
└── B
  ├── b
  └── x
```

使用联合挂载，挂载到公共目录 C 上：

```bash
$ mkdir C
$ mount -t aufs -o dirs=./A:./B none ./C
```

挂载结果：

```
$ tree ./C
./C
├── a
├── b
└── x
```

> 一个例子

```bash
$ docker image inspect ubuntu:latest
...
     "RootFS": {
      "Type": "layers",
      "Layers": [
        "sha256:f49017d4d5ce9c0f544c...",
        "sha256:8f2b771487e9d6354080...",
        "sha256:ccd4d61916aaa2159429...",
        "sha256:c01d74f99de40e097c73...",
        "sha256:268a067217b5fe78e000..."
      ]
    }
```

可以看到，这个ubuntu镜像，由五个层构成，每一层都是Ubuntu操作系统文件与目录的一部分。然后联合挂载到一个统一的挂载点上。

挂载到一起的各个层的信息：

```bash
$ cat /sys/fs/aufs/si_972c6d361e6b32ba/br[0-9]*
/var/lib/docker/aufs/diff/6e3be5d2ecccae7cc...=rw
/var/lib/docker/aufs/diff/6e3be5d2ecccae7cc...-init=ro+wh
/var/lib/docker/aufs/diff/32e8e20064858c0f2...=ro+wh
/var/lib/docker/aufs/diff/2b8858809bce62e62...=ro+wh
/var/lib/docker/aufs/diff/20707dce8efc0d267...=ro+wh
/var/lib/docker/aufs/diff/72b0744e06247c7d0...=ro+wh
/var/lib/docker/aufs/diff/a524a729adadedb90...=ro+wh
```

![](./images/1.webp)

第一层：只读层

它是这个容器的 rootfs 最下面的五层，对应的正是 ubuntu:latest 镜像的五层。可以看到，它们的挂载方式都是只读的（ro+wh，即 readonly+whiteout）。

第二层：init 层 (更改不需提交)

它是一个以“-init”结尾的层，夹在只读层和读写层之间。Init 层是 Docker 项目单独生成的一个内部层，专门用来存放 /etc/hosts、/etc/resolv.conf 等信息。需要这样一层的原因是，这些文件本来属于只读的 Ubuntu 镜像的一部分，但是用户往往需要在启动容器时写入一些指定的值比如 hostname，所以就需要在可读写层对它们进行修改。可是，这些修改往往只对当前的容器有效，我们并不希望执行 docker commit 时，把这些信息连同可读写层一起提交掉。所以，Docker 做法是，在修改了这些文件之后，以一个单独的层挂载了出来。而用户执行 docker commit 只会提交可读写层，所以是不包含这些内容的。

第三层：可读写层 (更改需要提交)

它是这个容器的 rootfs 最上面的一层（6e3be5d2ecccae7cc），它的挂载方式为：rw，即 read write。在没有写入文件之前，这个目录是空的。而一旦在容器里做了写操作，你修改产生的内容就会以增量的方式出现在这个层中。删除操作，AuFS 会在可读写层创建一个 whiteout 文件，把只读层里的文件“遮挡”起来。

比如，你要删除只读层里一个名叫 foo 的文件，那么这个删除操作实际上是在可读写层创建了一个名叫.wh.foo 的文件。这样，当这两个层被联合挂载之后，foo 文件就会被.wh.foo 文件“遮挡”起来，“消失”了。这个功能，就是“ro+wh”的挂载方式，即只读 +whiteout 的含义。

所以，最上面这个可读写层的作用，就是专门用来存放你修改 rootfs 后产生的增量，无论是增、删、改，都发生在这里。而当我们使用完了这个被修改过的容器之后，还可以使用 docker commit 和 push 指令，保存这个被修改过的可读写层，并上传到 Docker Hub 上，供其他人使用；而与此同时，原先的只读层里的内容则不会有任何变化。这，就是增量 rootfs 的好处。
