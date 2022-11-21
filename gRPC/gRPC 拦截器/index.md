# gRPC 拦截器

## 拦截器类型

1. UnaryServerInterceptor 服务端拦截，在服务端接收请求的时候进行拦截。
2. UnaryClientInterceptor 这是一个客户端上的拦截器，在客户端真正发起调用之前，进行拦截。
3. StreamClientInterceptor 在流式客户端调用时，通过拦截 clientstream 的创建，返回一个自定义的 clientstream, 可以做一些额外的操作。
4. StreamServerInterceptor 在服务端接收到流式请求的时候进行拦截。

## 普通拦截器

在 gRPC 中拦截器被定义成一个变量：

```go
type UnaryServerInterceptor func(ctx context.Context, 
req interface{}, info *UnaryServerInfo, handler UnaryHandler) (resp interface{}, err error)
```

参数含义如下:

- ctx context.Context：请求上下文
- req interface{}：RPC 方法的请求参数
- info *UnaryServerInfo：RPC 方法的所有信息
- handler UnaryHandler：RPC 方法真正执行的逻辑

它本质是一个方法，拦截器的应用是在服务端启动的时候要注册上去，从 `grpc.NewServer(opts...)` 这里开始，这里需要一个 ServerOption 对象：

```go
//注册拦截器 创建gRPC服务器
s := grpc.NewServer(grpc.UnaryInterceptor(myInterceptor))  
```

## 流拦截器

流拦截器过程和一元拦截器有所不同，同样可以分为3个阶段：

- 预处理(pre-processing)
- 调用RPC方法(invoking RPC method)
- 后处理(post-processing)

预处理阶段的拦截只是在流式请求第一次 发起的时候进行拦截，后面的流式请求不会再进入到处理逻辑。

后面两种情况对应着 Streamer api 提供的两个扩展方法来进行，分别是 SendMsg 和 RecvMsg 方法。

正常情况下实现一个流式拦截器与普通拦截器一样，实现这个已经定义好的拦截器方法即可：

```go
type StreamServerInterceptor func(srv interface{}, ss ServerStream, info *StreamServerInfo, handler StreamHandler) error
```

如果想在每次接收消息和发消息之前进行处理， 则实现 SendMsg 和 RecvMsg：

```go
type wrappedStream struct {
	grpc.ServerStream
}

func newWrappedStream(s grpc.ServerStream) grpc.ServerStream {
	return &wrappedStream{s}
}

func (w *wrappedStream) RecvMsg(m interface{}) error {
	fmt.Printf("Receive a message (Type: %T) at %s", m, time.Now().Format(time.RFC3339))
	return w.ServerStream.RecvMsg(m)
}

func (w *wrappedStream) SendMsg(m interface{}) error {
	fmt.Printf("Send a message (Type: %T) at %v", m, time.Now().Format(time.RFC3339))
	return w.ServerStream.SendMsg(m)
}
```

首先我们自己包装一个 grpc.ServerStream ，然后去实现它的 SendMsg 和 RecvMsg 方法。然后就是将这个 ServerStream 应用到拦截器中去：

```go
//发消息前后流式调用拦截器
func SendMsgCheckStreamServerInterceptor(srv interface{}, ss grpc.ServerStream, info *grpc.StreamServerInfo,
	handler grpc.StreamHandler) error {
	fmt.Printf("gRPC method: %s,", info.FullMethod)
	err := handler(srv, newWrappedStream(ss))
	fmt.Printf("gRPC method: %s", info.FullMethod)
	return err
}
```


