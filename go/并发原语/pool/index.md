# Pool

## 连接池

Pool 的另一个很常用的一个场景就是保持 TCP 的连接。一个 TCP 的连接创建，需要三次握手等过程，如果是 TLS 的，还会需要更多的步骤，如果加上身份认证等逻辑的话，耗时会更长。所以，为了避免每次通讯的时候都新创建连接，我们一般会建立一个连接的池子，预先把连接创建好，或者是逐步把连接放在池子中，减少连接创建的耗时，从而提高系统的性能。

事实上，我们很少会使用 sync.Pool 去池化连接对象，原因就在于，sync.Pool 会无通知地在某个时候就把连接移除垃圾回收掉了，而我们的场景是需要长久保持这个连接。Pool特性就是 如果对象没有被引用，那么就可能会被回收，（如果被引用了并且再次被PUT，又会复活），所以如果是TCP链接的话 一段时间没有被引用可能就没了。又要重新链接，显然不符合连接池特性。

所以，我们一般会使用其它方法来池化连接，比如接下来我要讲到的几种需要保持长连接的 Pool。

### HTTP Client 连接池

标准库的 http.Client 是一个 http client 的库，可以用它来访问 web 服务器。为了提高性能，这个 Client 的实现也是通过池的方法来缓存一定数量的连接，以便后续重用这些连接。http.Client 实现连接池的代码是在 Transport 类型中，它使用 idleConn 保存持久化的可重用的长连接：

```go
type Transport struct {
    idleMu       sync.Mutex
    closeIdle    bool                                // user has requested to close all idle conns
    idleConn     map[connectMethodKey][]*persistConn // most recently used at end
    idleConnWait map[connectMethodKey]wantConnQueue  // waiting getConns
    idleLRU      connLRU

    reqMu       sync.Mutex
    reqCanceler map[cancelKey]func(error)

    altMu    sync.Mutex   // guards changing altProto only
    altProto atomic.Value // of nil or map[string]RoundTripper, key is URI scheme

    connsPerHostMu   sync.Mutex
    connsPerHost     map[connectMethodKey]int
    connsPerHostWait map[connectMethodKey]wantConnQueue // waiting getConns
```
