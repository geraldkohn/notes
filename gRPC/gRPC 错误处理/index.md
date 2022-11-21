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
