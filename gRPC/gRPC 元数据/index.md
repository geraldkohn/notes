## Metadata

元数据（[metadata](https://pkg.go.dev/google.golang.org/grpc/metadata)）是指在处理RPC请求和响应过程中需要但又不属于具体业务（例如身份验证详细信息）的信息，采用键值对列表的形式，其中键是`string`类型，值通常是`[]string`类型，但也可以是二进制数据。gRPC中的 metadata 类似于我们在 HTTP headers中的键值对，元数据可以包含认证token、请求标识和监控标签等。

metadata中的键是大小写不敏感的，由字母、数字和特殊字符`-`、`_`、`.`组成并且不能以`grpc-`开头（gRPC保留自用），二进制值的键名必须以`-bin`结尾。

元数据对 gRPC 本身是不可见的，我们通常是在应用程序代码或中间件中处理元数据，我们不需要在`.proto`文件中指定元数据。

如何访问元数据取决于具体使用的编程语言。 在Go语言中我们是用[google.golang.org/grpc/metadata](https://pkg.go.dev/google.golang.org/grpc/metadata)这个库来操作metadata。

```go
type MD map[string][]string
```

元数据可以像普通map一样读取。注意，这个 map 的值类型是`[]string`，因此用户可以使用一个键附加多个值。

### 创建新的metadata

常用的创建MD的方法有以下两种。

第一种方法是使用函数 `New` 基于`map[string]string` 创建元数据:

```go
md := metadata.New(map[string]string{"key1": "val1", "key2": "val2"})
```

另一种方法是使用`Pairs`。具有相同键的值将合并到一个列表中:

```go
md := metadata.Pairs(
    "key1", "val1",
    "key1", "val1-2", // "key1"的值将会是 []string{"val1", "val1-2"}
    "key2", "val2",
)
```

注意: 所有的键将自动转换为小写，因此“ kEy1”和“ Key1”将是相同的键，它们的值将合并到相同的列表中。这种情况适用于 `New` 和 `Pair`。

### 元数据中存储二进制数据

在元数据中，键始终是字符串。但是值可以是字符串或二进制数据。要在元数据中存储二进制数据值，只需在密钥中添加“-bin”后缀。在创建元数据时，将对带有“-bin”后缀键的值进行编码:

```go
md := metadata.Pairs(
    "key", "string value",
    "key-bin", string([]byte{96, 102}), // 二进制数据在发送前会进行(base64) 编码
                                        // 收到后会进行解码
)
```

### 从请求上下文中获取元数据

可以使用 `FromIncomingContext` 可以从RPC请求的上下文中获取元数据:

```go
func (s *server) SomeRPC(ctx context.Context, in *pb.SomeRequest) (*pb.SomeResponse, err) {
    md, ok := metadata.FromIncomingContext(ctx)
    // do something with metadata
}
```

### 发送和接收元数据-客户端

#### 发送metadata

有两种方法可以将元数据发送到服务端。推荐的方法是使用 `AppendToOutgoingContext` 将 kv 对附加到context。无论context中是否已经有元数据都可以使用这个方法。如果先前没有元数据，则添加元数据; 如果context中已经存在元数据，则将 kv 对合并进去。

```go
// 创建带有metadata的context
ctx := metadata.AppendToOutgoingContext(ctx, "k1", "v1", "k1", "v2", "k2", "v3")

// 添加一些 metadata 到 context (e.g. in an interceptor)
ctx := metadata.AppendToOutgoingContext(ctx, "k3", "v4")

// 发起普通RPC请求
response, err := client.SomeRPC(ctx, someRequest)

// 或者发起流式RPC请求
stream, err := client.SomeStreamingRPC(ctx)
```

或者，可以使用 `NewOutgoingContext` 将元数据附加到context。但是，这将替换context中的任何已有的元数据，因此必须注意保留现有元数据(如果需要的话)。这个方法比使用 `AppendToOutgoingContext` 要慢。这方面的一个例子如下:

```go
// 创建带有metadata的context
md := metadata.Pairs("k1", "v1", "k1", "v2", "k2", "v3")
ctx := metadata.NewOutgoingContext(context.Background(), md)

// 添加一些metadata到context (e.g. in an interceptor)
send, _ := metadata.FromOutgoingContext(ctx)
newMD := metadata.Pairs("k3", "v3")
ctx = metadata.NewOutgoingContext(ctx, metadata.Join(send, newMD))

// 发起普通RPC请求
response, err := client.SomeRPC(ctx, someRequest)

// 或者发起流式RPC请求
stream, err := client.SomeStreamingRPC(ctx)
```

#### 接收metadata

客户端可以接收的元数据包括header和trailer。

> trailer可以用于服务器希望在处理请求后给客户端发送任何内容，例如在流式RPC中只有等所有结果都流到客户端后才能计算出负载信息，这时候就不能使用headers（header在数据之前，trailer在数据之后）。

引申：[HTTP trailer](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Trailer)

##### 普通调用

可以使用 [CallOption](https://godoc.org/google.golang.org/grpc#CallOption) 中的 [Header](https://godoc.org/google.golang.org/grpc#Header) 和 [Trailer](https://godoc.org/google.golang.org/grpc#Trailer) 函数来获取普通RPC调用发送的header和trailer:

```go
var header, trailer metadata.MD // 声明存储header和trailer的变量
r, err := client.SomeRPC(
    ctx,
    someRequest,
    grpc.Header(&header),    // 将会接收header
    grpc.Trailer(&trailer),  // 将会接收trailer
)
// do something with header and trailer
```

##### 流式调用

流式调用包括：

- 客户端流式
- 服务端流式
- 双向流式

使用接口 [ClientStream](https://godoc.org/google.golang.org/grpc#ClientStream) 中的 `Header` 和 `Trailer` 函数，可以从返回的流中接收 Header 和 Trailer:

```go
stream, err := client.SomeStreamingRPC(ctx)

// 接收 header
header, err := stream.Header()

// 接收 trailer
trailer := stream.Trailer()
```

### 发送和接收元数据-服务器端

#### 接收metadata

要读取客户端发送的元数据，服务器需要从 RPC 上下文检索它。如果是普通RPC调用，则可以使用 RPC 处理程序的上下文。对于流调用，服务器需要从流中获取上下文。

##### 普通调用

```go
func (s *server) SomeRPC(ctx context.Context, in *pb.someRequest) (*pb.someResponse, error) {
    md, ok := metadata.FromIncomingContext(ctx)
    // do something with metadata
}
```

##### 流式调用

```go
func (s *server) SomeStreamingRPC(stream pb.Service_SomeStreamingRPCServer) error {
    md, ok := metadata.FromIncomingContext(stream.Context()) // get context from stream
    // do something with metadata
}
```

#### 发送metadata

##### 普通调用

在普通调用中，服务器可以调用 [grpc](https://godoc.org/google.golang.org/grpc) 模块中的 [SendHeader](https://godoc.org/google.golang.org/grpc#SendHeader) 和 [SetTrailer](https://godoc.org/google.golang.org/grpc#SetTrailer) 函数向客户端发送header和trailer。这两个函数将context作为第一个参数。它应该是 RPC 处理程序的上下文或从中派生的上下文：

```go
func (s *server) SomeRPC(ctx context.Context, in *pb.someRequest) (*pb.someResponse, error) {
    // 创建和发送 header
    header := metadata.Pairs("header-key", "val")
    grpc.SendHeader(ctx, header)
    // 创建和发送 trailer
    trailer := metadata.Pairs("trailer-key", "val")
    grpc.SetTrailer(ctx, trailer)
}
```

##### 流式调用

对于流式调用，可以使用接口 [ServerStream](https://godoc.org/google.golang.org/grpc#ServerStream) 中的 `SendHeader` 和 `SetTrailer` 函数发送header和trailer:

```go
func (s *server) SomeStreamingRPC(stream pb.Service_SomeStreamingRPCServer) error {
    // 创建和发送 header
    header := metadata.Pairs("header-key", "val")
    stream.SendHeader(header)
    // 创建和发送 trailer
    trailer := metadata.Pairs("trailer-key", "val")
    stream.SetTrailer(trailer)
}
```

### 普通RPC调用metadata示例

#### client端的metadata操作

下面的代码片段演示了client端如何设置和获取metadata。

```go
// unaryCallWithMetadata 普通RPC调用客户端metadata操作
func unaryCallWithMetadata(c pb.GreeterClient, name string) {
    fmt.Println("--- UnarySayHello client---")
    // 创建metadata
    md := metadata.Pairs(
        "token", "app-test-q1mi",
        "request_id", "1234567",
    )
    // 基于metadata创建context.
    ctx := metadata.NewOutgoingContext(context.Background(), md)
    // RPC调用
    var header, trailer metadata.MD
    r, err := c.SayHello(
        ctx,
        &pb.HelloRequest{Name: name},
        grpc.Header(&header),   // 接收服务端发来的header
        grpc.Trailer(&trailer), // 接收服务端发来的trailer
    )
    if err != nil {
        log.Printf("failed to call SayHello: %v", err)
        return
    }
    // 从header中取location
    if t, ok := header["location"]; ok {
        fmt.Printf("location from header:\n")
        for i, e := range t {
            fmt.Printf(" %d. %s\n", i, e)
        }
    } else {
        log.Printf("location expected but doesn't exist in header")
        return
    }
   // 获取响应结果
    fmt.Printf("got response: %s\n", r.Reply)
    // 从trailer中取timestamp
    if t, ok := trailer["timestamp"]; ok {
        fmt.Printf("timestamp from trailer:\n")
        for i, e := range t {
            fmt.Printf(" %d. %s\n", i, e)
        }
    } else {
        log.Printf("timestamp expected but doesn't exist in trailer")
    }
}
```

#### server端metadata操作

下面的代码片段演示了server端如何设置和获取metadata。

```go
// UnarySayHello 普通RPC调用服务端metadata操作
func (s *server) UnarySayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloResponse, error) {
    // 通过defer中设置trailer.
    defer func() {
        trailer := metadata.Pairs("timestamp", strconv.Itoa(int(time.Now().Unix())))
        grpc.SetTrailer(ctx, trailer)
    }()

    // 从客户端请求上下文中读取metadata.
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, status.Errorf(codes.DataLoss, "UnarySayHello: failed to get metadata")
    }
    if t, ok := md["token"]; ok {
        fmt.Printf("token from metadata:\n")
        if len(t) < 1 || t[0] != "app-test-q1mi" {
            return nil, status.Error(codes.Unauthenticated, "认证失败")
        }
    }

    // 创建和发送header.
    header := metadata.New(map[string]string{"location": "BeiJing"})
    grpc.SendHeader(ctx, header)

    fmt.Printf("request received: %v, say hello...\n", in)

    return &pb.HelloResponse{Reply: in.Name}, nil
}
```

### 流式RPC调用metadata示例

这里以双向流式RPC为例演示客户端和服务端如何进行metadata操作。

#### client端的metadata操作

下面的代码片段演示了client端在服务端流式RPC模式下如何设置和获取metadata。

```go
// bidirectionalWithMetadata 流式RPC调用客户端metadata操作
func bidirectionalWithMetadata(c pb.GreeterClient, name string) {
    // 创建metadata和context.
    md := metadata.Pairs("token", "app-test-q1mi")
    ctx := metadata.NewOutgoingContext(context.Background(), md)

    // 使用带有metadata的context执行RPC调用.
    stream, err := c.BidiHello(ctx)
    if err != nil {
        log.Fatalf("failed to call BidiHello: %v\n", err)
    }

    go func() {
        // 当header到达时读取header.
        header, err := stream.Header()
        if err != nil {
            log.Fatalf("failed to get header from stream: %v", err)
        }
        // 从返回响应的header中读取数据.
        if l, ok := header["location"]; ok {
            fmt.Printf("location from header:\n")
            for i, e := range l {
                fmt.Printf(" %d. %s\n", i, e)
            }
        } else {
            log.Println("location expected but doesn't exist in header")
            return
        }

        // 发送所有的请求数据到server.
        for i := 0; i < 5; i++ {
            if err := stream.Send(&pb.HelloRequest{Name: name}); err != nil {
                log.Fatalf("failed to send streaming: %v\n", err)
            }
        }
        stream.CloseSend()
    }()

    // 读取所有的响应.
    var rpcStatus error
    fmt.Printf("got response:\n")
    for {
        r, err := stream.Recv()
        if err != nil {
            rpcStatus = err
            break
        }
        fmt.Printf(" - %s\n", r.Reply)
    }
    if rpcStatus != io.EOF {
        log.Printf("failed to finish server streaming: %v", rpcStatus)
        return
    }

    // 当RPC结束时读取trailer
    trailer := stream.Trailer()
    // 从返回响应的trailer中读取metadata.
    if t, ok := trailer["timestamp"]; ok {
        fmt.Printf("timestamp from trailer:\n")
        for i, e := range t {
            fmt.Printf(" %d. %s\n", i, e)
        }
    } else {
        log.Printf("timestamp expected but doesn't exist in trailer")
    }
}
```

#### server端的metadata操作

下面的代码片段演示了server端在服务端流式RPC模式下设置和操作metadata。

```go
// BidirectionalStreamingSayHello 流式RPC调用客户端metadata操作
func (s *server) BidirectionalStreamingSayHello(stream pb.Greeter_BidiHelloServer) error {
    // 在defer中创建trailer记录函数的返回时间.
    defer func() {
        trailer := metadata.Pairs("timestamp", strconv.Itoa(int(time.Now().Unix())))
        stream.SetTrailer(trailer)
    }()

    // 从client读取metadata.
    md, ok := metadata.FromIncomingContext(stream.Context())
    if !ok {
        return status.Errorf(codes.DataLoss, "BidirectionalStreamingSayHello: failed to get metadata")
    }

    if t, ok := md["token"]; ok {
        fmt.Printf("token from metadata:\n")
        for i, e := range t {
            fmt.Printf(" %d. %s\n", i, e)
        }
    }

    // 创建和发送header.
    header := metadata.New(map[string]string{"location": "X2Q"})
    stream.SendHeader(header)

    // 读取请求数据发送响应数据.
    for {
        in, err := stream.Recv()
        if err == io.EOF {
            return nil
        }
        if err != nil {
            return err
        }
        fmt.Printf("request received %v, sending reply\n", in)
        if err := stream.Send(&pb.HelloResponse{Reply: in.Name}); err != nil {
            return err
        }
    }
}
```
