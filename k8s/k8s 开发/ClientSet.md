# ClientSet

## 介绍

ClientSet 是调用 k8s 资源最常用的客户端，可以操作所有的资源对象。在讲 Schema 的时候我们说过，在 staging/src/k8s.io/api 下面定义了各种类型资源的规范，然后将这些规范注册到了全局的 Schema 中，这样就可以在 ClientSet 中使用这些资源了。

## 示例

首先我们看下通过 ClientSet 来获取资源对象的一个示例：

1. 创建 ClientSet 对象；

2. 获取 default namespace 下 Deployment list；

```go

```

## ClientSet 源码分析

clientset 包含所有的 API 对象：

```go

```

这些 API 对象就是 staging/src/k8s.io/api 中所有的 API 对象。
