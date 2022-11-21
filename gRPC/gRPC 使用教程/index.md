# gRPC 使用教程

## 安装 gRPC

### 安装插件

因为本文我们是使用Go语言做开发，接下来执行下面的命令安装`protoc`的Go插件：

安装go语言插件：

```bash
go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.28
```

该插件会根据`.proto`文件生成一个后缀为`.pb.go`的文件，包含所有`.proto`文件中定义的类型及其序列化方法。

安装grpc插件：

```bash
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.2
```

该插件会生成一个后缀为`_grpc.pb.go`的文件，其中包含：

- 一种接口类型(或存根) ，供客户端调用的服务方法。
- 服务器要实现的接口类型。

上述命令会默认将插件安装到`$GOPATH/bin`，为了`protoc`编译器能找到这些插件，请确保你的`$GOPATH/bin`在环境变量中。

[protocol-buffers 官方Go教程](https://developers.google.com/protocol-buffers/docs/gotutorial)

### 检查

依次执行以下命令检查一下是否开发环境都准备完毕。

1. 确认 protoc 安装完成。
   
   ```bash
   ❯ protoc --version
   libprotoc 3.20.1
   ```

2. 确认 protoc-gen-go 安装完成。
   
   ```bash
   ❯ protoc-gen-go --version
   protoc-gen-go v1.28.0
   ```
   
   如果这里提示`protoc-gen-go`不是可执行的程序，请确保你的 GOPATH 下的 bin 目录在你电脑的环境变量中。

3. 确认 protoc-gen-go-grpc 安装完成。
   
   ```bash
   ❯ protoc-gen-go-grpc --version
   protoc-gen-go-grpc 1.2.0
   ```
   
   如果这里提示`protoc-gen-go-grpc`不是可执行的程序，请确保你的 GOPATH 下的 bin 目录在你电脑的环境变量中。

## gRPC 入门示例

### 非流式

```protobuf
syntax = "proto3";

package proto;

option go_package = "pb/base"; // 指定生成的Go代码在项目中的导入路径

// 定义服务
service Greeter {
    // SayHello
    rpc SayHello (HelloRequest) returns (HelloResponse) {}
}

message HelloRequest {
    string requst = 1;
}

message HelloResponse {
    string reply = 1;
}

// 在项目下执行

/**
protoc -I=proto --go_out=pb --go_opt=paths=source_relative \
--go-grpc_out=pb --go-grpc_opt=paths=source_relative \
base/hello.proto

解释：
-I=proto 是告诉protoc，要编译的文件路径的一部分。
--go_out=pb 表示输出目录在pb, --go_opt=paths=source_relative 表示输出路径和输入路径一样。
也就是说：输入：proto/输入路径，输出：pb/输入路径。这里的输入路径就是 base/hello.proto
--go-grpc 和 --go-grpc_opt 与 --go_out 和 --go-opt 一样。
*/
```

```go
// server.go

type server struct {
    pb.UnimplementedGreeterServer
}

func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloResponse, error) {
    return &pb.HelloResponse{Reply: "hello from server. " + in.GetRequst()}, nil
}

func main() {
    lis, _ := net.Listen("tcp", ":8972")
    s := grpc.NewServer()                  // 创建grpc服务器
    pb.RegisterGreeterServer(s, &server{}) // 在grpc注册服务
    fmt.Println("Listen: 127.0.0.1:8972")
    _ = s.Serve(lis)                       // 启动服务
}
```

```go
// client.go

func main() {
    conn, err := grpc.Dial("127.0.0.1:8972", grpc.WithTransportCredentials(insecure.NewCredentials()))
    if err != nil {
        fmt.Println(err)
        fmt.Println("Failed to connect")
        return
    }
    defer conn.Close()
    c := pb.NewGreeterClient(conn)

    // 执行 grpc 远程调用
    ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
    defer cancel()
    r, _ := c.SayHello(ctx, &pb.HelloRequest{Requst: "client"})
    fmt.Println(r.GetReply())
}
```

### 客户端流式

```protobuf
syntax = "proto3";

package proto;

option go_package = "pb/stream-server"; // 生成的Go代码在项目中的导入路径

// 定义服务
service Greeter {
    // SayHello
    rpc SayHello (stream HelloRequest) returns (HelloResponse) {}
}

message HelloRequest {
    string requst = 1;
}

message HelloResponse {
    string reply = 1;
}
```

```go
// server.go

type server struct {
    pb.UnimplementedGreeterServer
}

func (s *server) SayHello(stream pb.Greeter_SayHelloServer) error {
    reply := "hello, this is server, request from client: "
    res := ""
    for {
        // 接收客户端发来的流数据
        req, err := stream.Recv()
        res += req.GetRequst()
        if err == io.EOF {
            return stream.SendAndClose(&pb.HelloResponse{
                Reply: fmt.Sprintf("%s %s", reply, res),
            })
        }
        if err != nil {
            return err
        }
    }
}

func main() {
    lis, _ := net.Listen("tcp", ":8972")
    s := grpc.NewServer()                  // 创建grpc服务器
    pb.RegisterGreeterServer(s, &server{}) // 在grpc注册服务
    fmt.Println("Listen: 127.0.0.1:8972")
    _ = s.Serve(lis) // 启动服务
}
```

```go
// client.go

func main() {
    conn, err := grpc.Dial("127.0.0.1:8972", grpc.WithTransportCredentials(insecure.NewCredentials()))
    if err != nil {
        fmt.Println(err)
        fmt.Println("Failed to connect")
        return
    }
    defer conn.Close()
    c := pb.NewGreeterClient(conn)

    // 执行 grpc 远程调用
    ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
    defer cancel()

    // 客户端流式 RPC
    stream, _ := c.SayHello(ctx)
    reqs := []string{"client-1 ", "client-2 ", "client-3 "}

    // 发送流式数据
    for _, req := range reqs {
        err = stream.Send(&pb.HelloRequest{Requst: req})
        if err != nil {
            fmt.Println(err)
            return
        }
    }

    // 接收服务端数据
    res, _ := stream.CloseAndRecv()
    fmt.Println(res)
}
```

### 服务端流式

```protobuf
syntax = "proto3";

package proto;

option go_package = "pb/stream-server"; // 指定生成的Go代码在项目中的导入路径

// 定义服务
service Greeter {
    // SayHello
    rpc SayHello (HelloRequest) returns (stream HelloResponse) {}
}

message HelloRequest {
    string requst = 1;
}

message HelloResponse {
    string reply = 1;
}
```

```go
// server.go

type server struct {
    pb.UnimplementedGreeterServer
}

func (s *server) SayHello(in *pb.HelloRequest, stream pb.Greeter_SayHelloServer) error {
    log.Println("Use SayHello Method")
    words := []string{
        "hello",
        "你好",
        "hello again",
        "你好吗",
    }

    for _, word := range words {
        data := &pb.HelloResponse{
            Reply: word + in.GetRequst(),
        }
        // 使用 send 方法返回多个数据
        if err := stream.Send(data); err != nil {
            fmt.Println("服务端错误")
            return err
        }
    }

    return nil
}

func main() {
    lis, _ := net.Listen("tcp", ":8972")
    s := grpc.NewServer()                  // 创建grpc服务器
    pb.RegisterGreeterServer(s, &server{}) // 在grpc注册服务
    fmt.Println("Listen: 127.0.0.1:8972")
    _ = s.Serve(lis) // 启动服务
}
```

```go
// client.go

func main() {
    conn, err := grpc.Dial("127.0.0.1:8972", grpc.WithTransportCredentials(insecure.NewCredentials()))
    if err != nil {
        fmt.Println(err)
        fmt.Println("Failed to connect")
        return
    }
    defer conn.Close()
    c := pb.NewGreeterClient(conn)

    // 执行 grpc 远程调用
    ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
    defer cancel()
    stream, _ := c.SayHello(ctx, &pb.HelloRequest{Requst: "client"})

    for {
        // 接收服务端返回的流式数据，当收到io.EOF或者错误时退出
        res, err := stream.Recv()
        if err == io.EOF {
            fmt.Println("传输结束")
            break
        }
        if err != nil {
            fmt.Println(err)
            fmt.Println("服务端错误")
            break
        }
        fmt.Println(res)
    }
}
```

### 双向流式

```protobuf
syntax = "proto3";

package proto;

option go_package = "pb/stream-server"; // 指定生成的Go代码在项目中的导入路径

// 定义服务
service Greeter {
    // SayHello
    rpc SayHello (stream HelloRequest) returns (stream HelloResponse) {}
}

message HelloRequest {
    string requst = 1;
}

message HelloResponse {
    string reply = 1;
}
```

```go
// server.go

type server struct {
    pb.UnimplementedGreeterServer
}

func (s *server) SayHello(stream pb.Greeter_SayHelloServer) error {
    for {
        // 接收流式请求
        in, err := stream.Recv()
        // 都客户端关闭发送连接, 就退出
        if err == io.EOF {
            return nil
        }
        if err != nil {
            return err
        }

        reply := handle(in.GetRequst())

        // 返回流式响应
        if err := stream.Send(&pb.HelloResponse{Reply: reply}); err != nil {
            return err
        }
    }
}

func handle(s string) string {
    time.Sleep(100 * time.Second) // 模拟处理请求时间
    s += " Server Response"
    return s
}

func main() {
    lis, _ := net.Listen("tcp", ":8972")
    s := grpc.NewServer()                  // 创建grpc服务器
    pb.RegisterGreeterServer(s, &server{}) // 在grpc注册服务
    fmt.Println("Listen: 127.0.0.1:8972")
    _ = s.Serve(lis) // 启动服务
}
```

```go
// client.go

func main() {
    conn, err := grpc.Dial("127.0.0.1:8972", grpc.WithTransportCredentials(insecure.NewCredentials()))
    if err != nil {
        fmt.Println(err)
        fmt.Println("Failed to connect")
        return
    }
    defer conn.Close()
    c := pb.NewGreeterClient(conn)

    // 执行 grpc 远程调用
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // 双向流式 RPC
    stream, _ := c.SayHello(ctx)

    wait := make(chan struct{})
    sendClose := make(chan struct{})

    // 发送流数据
    go func() {
        for {
            select {
            case <-sendClose:
                return
            default:
                err := stream.Send(&pb.HelloRequest{Requst: fmt.Sprintf("Client send at %s", time.Now().String())})
                if err != nil {
                    return
                }
                time.Sleep(10 * time.Millisecond) // 每隔 10 毫秒发送一个信息
            }
        }
    }()

    // 接收流数据
    go func() {
        for {
            res, err := stream.Recv()
            // 已经接收到全部消息
            if err == io.EOF {
                wait <- struct{}{} // 服务器处理完毕
                return
            }
            if err != nil {
                fmt.Println(err)
                return
            }
            fmt.Println(res.GetReply())
        }
    }()

    time.Sleep(1 * time.Second)
    sendClose <- struct{}{} // 停止发送消息
    stream.CloseSend()      // 客户端关闭连接
    <-wait                  // 等待客户端接收到全部信息
}
```

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

## 错误处理

### gRPC code

类似于HTTP定义了一套响应状态码，gRPC也定义有一些状态码。Go语言中此状态码由[codes](https://pkg.go.dev/google.golang.org/grpc/codes)定义，本质上是一个uint32。

```go
type Code uint32
```

使用时需导入`google.golang.org/grpc/codes`包。

```go
import "google.golang.org/grpc/codes"
```

目前已经定义的状态码有如下几种。

| Code               | 值   | 含义                                                                                                 |
| ------------------ | --- | -------------------------------------------------------------------------------------------------- |
| OK                 | 0   | 请求成功                                                                                               |
| Canceled           | 1   | 操作已取消                                                                                              |
| Unknown            | 2   | 未知错误。如果从另一个地址空间接收到的状态值属 于在该地址空间中未知的错误空间，则可以返回此错误的示例。 没有返回足够的错误信息的API引发的错误也可能会转换为此错误                |
| InvalidArgument    | 3   | 表示客户端指定的参数无效。 请注意，这与 FailedPrecondition 不同。 它表示无论系统状态如何都有问题的参数（例如，格式错误的文件名）。                       |
| DeadlineExceeded   | 4   | 表示操作在完成之前已过期。对于改变系统状态的操作，即使操作成功完成，也可能会返回此错误。 例如，来自服务器的成功响应可能已延迟足够长的时间以使截止日期到期。                     |
| NotFound           | 5   | 表示未找到某些请求的实体（例如，文件或目录）。                                                                            |
| AlreadyExists      | 6   | 创建实体的尝试失败，因为实体已经存在。                                                                                |
| PermissionDenied   | 7   | 表示调用者没有权限执行指定的操作。 它不能用于拒绝由耗尽某些资源引起的（使用 ResourceExhausted ）。 如果无法识别调用者，也不能使用它（使用 Unauthenticated ）。 |
| ResourceExhausted  | 8   | 表示某些资源已耗尽，可能是每个用户的配额，或者整个文件系统空间不足                                                                  |
| FailedPrecondition | 9   | 指示操作被拒绝，因为系统未处于操作执行所需的状态。 例如，要删除的目录可能是非空的，rmdir 操作应用于非目录等。                                         |
| Aborted            | 10  | 表示操作被中止，通常是由于并发问题，如排序器检查失败、事务中止等。                                                                  |
| OutOfRange         | 11  | 表示尝试超出有效范围的操作。                                                                                     |
| Unimplemented      | 12  | 表示此服务中未实施或不支持/启用操作。                                                                                |
| Internal           | 13  | 意味着底层系统预期的一些不变量已被破坏。 如果你看到这个错误，则说明问题很严重。                                                           |
| Unavailable        | 14  | 表示服务当前不可用。这很可能是暂时的情况，可以通过回退重试来纠正。 请注意，重试非幂等操作并不总是安全的。                                              |
| DataLoss           | 15  | 表示不可恢复的数据丢失或损坏                                                                                     |
| Unauthenticated    | 16  | 表示请求没有用于操作的有效身份验证凭据                                                                                |
| _maxCode           | 17  | -                                                                                                  |

### gRPC Status

Go语言使用的gRPC Status 定义在[google.golang.org/grpc/status](https://pkg.go.dev/google.golang.org/grpc/status)，使用时需导入。

```go
import "google.golang.org/grpc/status"
```

RPC服务的方法应该返回 `nil` 或来自`status.Status`类型的错误。客户端可以直接访问错误。

#### 创建错误

当遇到错误时，gRPC服务的方法函数应该创建一个 `status.Status`。通常我们会使用 `status.New`函数并传入适当的`status.Code`和错误描述来生成一个`status.Status`。调用`status.Err`方法便能将一个`status.Status`转为`error`类型。也存在一个简单的`status.Error`方法直接生成`error`。下面是两种方式的比较。

```go
// 创建status.Status
st := status.New(codes.NotFound, "some description")
err := st.Err()  // 转为error类型

// vs.

err := status.Error(codes.NotFound, "some description")
```

#### 为错误添加其他详细信息

在某些情况下，可能需要为服务器端的特定错误添加详细信息。`status.WithDetails`就是为此而存在的，它可以添加任意多个`proto.Message`，我们可以使用`google.golang.org/genproto/googleapis/rpc/errdetails`中的定义或自定义的错误详情。

```go
st := status.New(codes.ResourceExhausted, "Request limit exceeded.")
ds, _ := st.WithDetails(
    // proto.Message
)
return nil, ds.Err()
```

然后，客户端可以通过首先将普通`error`类型转换回`status.Status`，然后使用`status.Details`来读取这些详细信息。

```go
s := status.Convert(err)
for _, d := range s.Details() {
    // ...
}
```

### 代码示例

我们现在要为`hello`服务设置访问限制，每个`name`只能调用一次`SayHello`方法，超过此限制就返回一个请求超过限制的错误。

#### 服务端

使用map存储每个name的请求次数，超过1次则返回错误，并且记录错误详情。

```go
package main

import (
    "context"
    "fmt"
    "hello_server/pb"
    "net"
    "sync"

    "google.golang.org/genproto/googleapis/rpc/errdetails"
    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

// grpc server

type server struct {
    pb.UnimplementedGreeterServer
    mu    sync.Mutex     // count的并发锁
    count map[string]int // 记录每个name的请求次数
}

// SayHello 是我们需要实现的方法
// 这个方法是我们对外提供的服务
func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloResponse, error) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.count[in.Name]++ // 记录用户的请求次数
    // 超过1次就返回错误
    if s.count[in.Name] > 1 {
        st := status.New(codes.ResourceExhausted, "Request limit exceeded.")
        ds, err := st.WithDetails(
            &errdetails.QuotaFailure{
                Violations: []*errdetails.QuotaFailure_Violation{{
                    Subject:     fmt.Sprintf("name:%s", in.Name),
                    Description: "限制每个name调用一次",
                }},
            },
        )
        if err != nil {
            return nil, st.Err()
        }
        return nil, ds.Err()
    }
    // 正常返回响应
    reply := "hello " + in.GetName()
    return &pb.HelloResponse{Reply: reply}, nil
}

func main() {
    // 启动服务
    l, err := net.Listen("tcp", ":8972")
    if err != nil {
        fmt.Printf("failed to listen, err:%v\n", err)
        return
    }
    s := grpc.NewServer() // 创建grpc服务
    // 注册服务，注意初始化count
    pb.RegisterGreeterServer(s, &server{count: make(map[string]int)})
    // 启动服务
    err = s.Serve(l)
    if err != nil {
        fmt.Printf("failed to serve,err:%v\n", err)
        return
    }
}
```

#### 客户端

当服务端返回错误时，尝试从错误中获取detail信息。

```go
package main

import (
    "context"
    "flag"
    "fmt"
    "google.golang.org/grpc/status"
    "hello_client/pb"
    "log"
    "time"

    "google.golang.org/genproto/googleapis/rpc/errdetails"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
)

// grpc 客户端
// 调用server端的 SayHello 方法

var name = flag.String("name", "七米", "通过-name告诉server你是谁")

func main() {
    flag.Parse() // 解析命令行参数

    // 连接server
    conn, err := grpc.Dial("127.0.0.1:8972", grpc.WithTransportCredentials(insecure.NewCredentials()))
    if err != nil {
        log.Fatalf("grpc.Dial failed,err:%v", err)
        return
    }
    defer conn.Close()
    // 创建客户端
    c := pb.NewGreeterClient(conn) // 使用生成的Go代码
    // 调用RPC方法
    ctx, cancel := context.WithTimeout(context.Background(), time.Second)
    defer cancel()
    resp, err := c.SayHello(ctx, &pb.HelloRequest{Name: *name})
    if err != nil {
        s := status.Convert(err)        // 将err转为status
        for _, d := range s.Details() { // 获取details
            switch info := d.(type) {
            case *errdetails.QuotaFailure:
                fmt.Printf("Quota failure: %s\n", info)
            default:
                fmt.Printf("Unexpected type: %s\n", info)
            }
        }
        fmt.Printf("c.SayHello failed, err:%v\n", err)
        return
    }
    // 拿到了RPC响应
    log.Printf("resp:%v\n", resp.GetReply())
}metadata 类型定义如下：
```

## 拦截器

gRPC 为在每个 ClientConn/Server 基础上实现和安装拦截器提供了一些简单的 API。 拦截器拦截每个 RPC 调用的执行。用户可以使用拦截器进行日志记录、身份验证/授权、指标收集以及许多其他可以跨 RPC 共享的功能。

在 gRPC 中，拦截器根据拦截的 RPC 调用类型可以分为两类。第一个是普通拦截器（一元拦截器），它拦截普通RPC 调用。另一个是流拦截器，它处理流式 RPC 调用。而客户端和服务端都有自己的普通拦截器和流拦截器类型。因此，在 gRPC 中总共有四种不同类型的拦截器。

### 客户端端拦截器

#### 普通拦截器/一元拦截器

[UnaryClientInterceptor](https://godoc.org/google.golang.org/grpc#UnaryClientInterceptor) 是客户端一元拦截器的类型，它的函数前面如下：

```go
func(ctx context.Context, method string, req, reply interface{}, cc *ClientConn, invoker UnaryInvoker, opts ...CallOption) error
```

一元拦截器的实现通常可以分为三个部分: 调用 RPC 方法之前（预处理）、调用 RPC 方法（RPC调用）和调用 RPC 方法之后（调用后）。

- 预处理：用户可以通过检查传入的参数(如 RPC 上下文、方法字符串、要发送的请求和 CallOptions 配置)来获得有关当前 RPC 调用的信息。
- RPC调用：预处理完成后，可以通过执行`invoker`执行 RPC 调用。
- 调用后：一旦调用者返回应答和错误，用户就可以对 RPC 调用进行后处理。通常，它是关于处理返回的响应和错误的。 若要在 `ClientConn` 上安装一元拦截器，请使用`DialOptionWithUnaryInterceptor`的`DialOption`配置 Dial 。

#### 流拦截器

[StreamClientInterceptor](https://godoc.org/google.golang.org/grpc#StreamClientInterceptor)是客户端流拦截器的类型。它的函数签名是

```go
func(ctx context.Context, desc *StreamDesc, cc *ClientConn, method string, streamer Streamer, opts ...CallOption) (ClientStream, error)
```

流拦截器的实现通常包括预处理和流操作拦截。

- 预处理：类似于上面的一元拦截器。
- 流操作拦截：流拦截器并没有事后进行 RPC 方法调用和后处理，而是拦截了用户在流上的操作。首先，拦截器调用传入的`streamer`以获取 `ClientStream`，然后包装 `ClientStream` 并用拦截逻辑重载其方法。最后，拦截器将包装好的 `ClientStream` 返回给用户进行操作。

若要为 `ClientConn` 安装流拦截器，请使用`WithStreamInterceptor`的 DialOption 配置 Dial。

### server端拦截器

服务器端拦截器与客户端类似，但提供的信息略有不同。

#### 普通拦截器/一元拦截器

[UnaryServerInterceptor](https://godoc.org/google.golang.org/grpc#UnaryServerInterceptor)是服务端的一元拦截器类型，它的函数签名是

```go
func(ctx context.Context, req interface{}, info *UnaryServerInfo, handler UnaryHandler) (resp interface{}, err error)
```

服务端一元拦截器具体实现细节和客户端版本的类似。

若要为服务端安装一元拦截器，请使用 `UnaryInterceptor` 的`ServerOption`配置 `NewServer`。

#### 流拦截器

[StreamServerInterceptor](https://godoc.org/google.golang.org/grpc#StreamServerInterceptor)是服务端流式拦截器的类型，它的签名如下：

```go
func(srv interface{}, ss ServerStream, info *StreamServerInfo, handler StreamHandler) error
```

实现细节类似于客户端流拦截器部分。

若要为服务端安装流拦截器，请使用 `StreamInterceptor` 的`ServerOption`来配置 `NewServer`。

### 拦截器示例

下面将演示一个完整的拦截器示例，我们为一元RPC和流式RPC服务都添加上拦截器。

我们首先定义一个名为`valid`的校验函数。

```go
// valid 校验认证信息.
func valid(authorization []string) bool {
    if len(authorization) < 1 {
        return false
    }
    token := strings.TrimPrefix(authorization[0], "Bearer ")
    // 执行token认证的逻辑
    // 这里是为了演示方便简单判断token是否与"some-secret-token"相等
    return token == "some-secret-token"
}
```

#### 客户端拦截器定义

##### 一元拦截器

```go
// unaryInterceptor 客户端一元拦截器
func unaryInterceptor(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
    var credsConfigured bool
    for _, o := range opts {
        _, ok := o.(grpc.PerRPCCredsCallOption)
        if ok {
            credsConfigured = true
            break
        }
    }
    if !credsConfigured {
        opts = append(opts, grpc.PerRPCCredentials(oauth.NewOauthAccess(&oauth2.Token{
            AccessToken: "some-secret-token",
        })))
    }
    start := time.Now()
    err := invoker(ctx, method, req, reply, cc, opts...)
    end := time.Now()
    fmt.Printf("RPC: %s, start time: %s, end time: %s, err: %v\n", method, start.Format("Basic"), end.Format(time.RFC3339), err)
    return err
}
```

其中，`grpc.PerRPCCredentials()`函数指明每个 RPC 请求使用的凭据，它接收一个`credentials.PerRPCCredentials`接口类型的参数。`credentials.PerRPCCredentials`接口的定义如下：

```go
type PerRPCCredentials interface {
    // GetRequestMetadata 获取当前请求的元数据,如果需要则会设置token。
    // 传输层在每个请求上调用，并且数据会被填充到headers或其他context。
    GetRequestMetadata(ctx context.Context, uri ...string) (map[string]string, error)
    // RequireTransportSecurity 指示该 Credentials 的传输是否需要需要 TLS 加密
    RequireTransportSecurity() bool
}
```

而示例代码中使用的`oauth.NewOauthAccess()`是内置oauth包提供的一个函数，用来返回包含给定token的`PerRPCCredentials`。

```go
// NewOauthAccess constructs the PerRPCCredentials using a given token.
func NewOauthAccess(token *oauth2.Token) credentials.PerRPCCredentials {
    return oauthAccess{token: *token}
}

func (oa oauthAccess) GetRequestMetadata(ctx context.Context, uri ...string) (map[string]string, error) {
    ri, _ := credentials.RequestInfoFromContext(ctx)
    if err := credentials.CheckSecurityLevel(ri.AuthInfo, credentials.PrivacyAndIntegrity); err != nil {
        return nil, fmt.Errorf("unable to transfer oauthAccess PerRPCCredentials: %v", err)
    }
    return map[string]string{
        "authorization": oa.token.Type() + " " + oa.token.AccessToken,
    }, nil
}

func (oa oauthAccess) RequireTransportSecurity() bool {
    return true
}
```

##### 流式拦截器

自定义一个`ClientStream`类型。

```go
type wrappedStream struct {
    grpc.ClientStream
}
```

`wrappedStream`重写`grpc.ClientStream`接口的`RecvMsg`和`SendMsg`方法。

```go
func (w *wrappedStream) RecvMsg(m interface{}) error {
    logger("Receive a message (Type: %T) at %v", m, time.Now().Format(time.RFC3339))
    return w.ClientStream.RecvMsg(m)
}

func (w *wrappedStream) SendMsg(m interface{}) error {
    logger("Send a message (Type: %T) at %v", m, time.Now().Format(time.RFC3339))
    return w.ClientStream.SendMsg(m)
}

func newWrappedStream(s grpc.ClientStream) grpc.ClientStream {
    return &wrappedStream{s}
}
```

> 这里的`wrappedStream`嵌入了`grpc.ClientStream`接口类型，然后又重新实现了一遍`grpc.ClientStream`接口的方法。

下面就定义一个流式拦截器，最后返回上面定义的`wrappedStream`。

```go
// streamInterceptor 客户端流式拦截器
func streamInterceptor(ctx context.Context, desc *grpc.StreamDesc, cc *grpc.ClientConn, method string, streamer grpc.Streamer, opts ...grpc.CallOption) (grpc.ClientStream, error) {
    var credsConfigured bool
    for _, o := range opts {
        _, ok := o.(*grpc.PerRPCCredsCallOption)
        if ok {
            credsConfigured = true
            break
        }
    }
    if !credsConfigured {
        opts = append(opts, grpc.PerRPCCredentials(oauth.NewOauthAccess(&oauth2.Token{
            AccessToken: "some-secret-token",
        })))
    }
    s, err := streamer(ctx, desc, cc, method, opts...)
    if err != nil {
        return nil, err
    }
    return newWrappedStream(s), nil
}
```

#### 服务端拦截器定义

##### 一元拦截器

服务端定义一个一元拦截器，对从请求元数据中获取的`authorization`进行校验。

```go
// unaryInterceptor 服务端一元拦截器
func unaryInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    // authentication (token verification)
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, status.Errorf(codes.InvalidArgument, "missing metadata")
    }
    if !valid(md["authorization"]) {
        return nil, status.Errorf(codes.Unauthenticated, "invalid token")
    }
    m, err := handler(ctx, req)
    if err != nil {
        fmt.Printf("RPC failed with error %v\n", err)
    }
    return m, err
}
```

#### 流拦截器

同样为流RPC也定义一个从元数据中获取认证信息的流式拦截器。

```go
// streamInterceptor 服务端流拦截器
func streamInterceptor(srv interface{}, ss grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler) error {
    // authentication (token verification)
    md, ok := metadata.FromIncomingContext(ss.Context())
    if !ok {
        return status.Errorf(codes.InvalidArgument, "missing metadata")
    }
    if !valid(md["authorization"]) {
        return status.Errorf(codes.Unauthenticated, "invalid token")
    }

    err := handler(srv, newWrappedStream(ss))
    if err != nil {
        fmt.Printf("RPC failed with error %v\n", err)
    }
    return err
}
```

#### 注册拦截器

客户端注册拦截器

```go
conn, err := grpc.Dial("127.0.0.1:8972",
    grpc.WithTransportCredentials(creds),
    grpc.WithUnaryInterceptor(unaryInterceptor),
    grpc.WithStreamInterceptor(streamInterceptor),
)
```

服务端注册拦截器

```go
s := grpc.NewServer(
    grpc.Creds(creds),
    grpc.UnaryInterceptor(unaryInterceptor),
    grpc.StreamInterceptor(streamInterceptor),
)
```

### go-grpc-middleware

社区中有很多开源的常用的grpc中间件——[go-grpc-middleware](https://github.com/grpc-ecosystem/go-grpc-middleware)，大家可以根据需要选择使用。
