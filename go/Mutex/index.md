## 1 Go sync.RWMutex 怎么实现的

sync.RWMutex 是 Go 语言标准库中的一种典型的读写锁(reader-writer lock)实现。

* 加读锁：先判断是否有 goroutine 持有写锁。如果有持有写锁的 goroutine，那么就休眠当前的读 goroutine；如果没有，就将 readers 加1，然后读。

* 释放读锁：将 readers 减去1，如果减到 0 了，那么说明没有读 goroutine，先唤醒阻塞的读 goroutine，再唤醒写阻塞 goroutine。

* 加写锁：判断是否有 goroutine 持有读锁，如果有则加入写阻塞队列；没有则加互斥锁。

* 释放写锁：唤醒阻塞的读 goroutine，然后释放互斥锁。

只有在操作写锁的时候才使用互斥锁，读的时候使用 readers 来计数。

## 2 Go 如何实现可重入锁

实现一个可重入锁需要两点：

* 记住持有锁的线程

* 统计重入的次数

```go
type ReentrantLock struct {
    sync.Mutex
    recursion int32 // 这个 goroutine 重入的次数
    owner     int64 // 当前持有锁的 goroutine id
}
```


