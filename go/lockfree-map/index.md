# Lock-free Map

在开始之前，先定义一个概念 *Write-Rarely-Read-Many*，也就是*读多写少*。这代表了一个典型的优化并发性能的场景（例如 Linux 内核中的路由表）。我们来实现一个Write-Rarely-Read-Many 场景下的 map。

## Lock-based  Map

如果要实现一个支持并发访问的 Map 数据结构，最简单的方法就是使用互斥锁锁住临界区。

使用锁的代码非常清晰，也能保证并发的正确性，但是从性能上考虑，多个并发线程会产生锁竞争，从而降低访问性能。一个直观的展示可以参考[这里](http://preshing.com/20111118/locks-arent-slow-lock-contention-is/)。

## Shard Map

使用分片锁能够降低锁的细粒度，能在一定程度上提升性能。[这里](https://github.com/orcaman/concurrent-map)是一个流行的分片锁实现。

## Lock-free Map

阅读[论文](https://dl.acm.org/doi/pdf/10.1145/224964.224988)




