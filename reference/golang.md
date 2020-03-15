# `Go`生成的代码参考


本指南介绍了使用`grpc`插件生成的代码， 以`protoc-gen-go`使用`protoc`编译`.proto`文件。

您可以`.proto`在服务定义中找到如何在文件中定义`gRPC`服务。

线程安全性：请注意，客户端`RPC`调用和服务器端`RPC`处理程序是线程安全的，并且可以在并发`goroutines`上运行。但也请注意，对于单个流，传入和传出的数据是双向的，但是串行的。因此，例如，单个流不支持并发读取或并发写入（但读取与写安全地并发）。

## 生成的服务器接口上的方法
在服务器端，`.proto`文件中的每个`service Bar`都将产生以下功能：
```Golang
func RegisterBarServer(s *grpc.Server, srv BarServer)
```
应用程序可以定义该`BarServer`接口的具体实现，并`grpc.Server`通过使用此功能在启动实例之前将其注册到实例。

### 一元方法
这些方法在生成的服务接口上具有以下签名：
```Golang
Foo(context.Context, *MsgA) (*MsgB, error)
```
在这种情况下，`MsgA`是从客户端发送的`protobuf`消息，`MsgB`是从服务器发送回的`protobuf`消息。

### 服务器流方法
这些方法在生成的服务接口上具有以下签名：
```Golang
Foo(*MsgA, <ServiceName>_FooServer) error
```
在这种情况下，`MsgA`是来自客户端的单个请求，并且该`<ServiceName>_FooServer`参数表示服务器到客户端的`MsgB`消息流。

`<ServiceName>_FooServer`具有嵌入式`grpc.ServerStream`和以下接口：
```Golang
type <ServiceName>_FooServer interface {
	Send(*MsgB) error
	grpc.ServerStream
}
```
服务器端处理程序可以通过此参数的`Send`方法向客户端发送`protobuf`消息流。服务器到客户端流的流结束是由`return`处理程序方法引起的。

### 客户端流式传输方法
这些方法在生成的服务接口上具有以下签名：
```Golang
Foo(<ServiceName>_FooServer) error
````
在这种情况下，`<ServiceName>_FooServer`既可以用来读取客户端到服务器的消息流，也可以用来发送单个服务器的响应消息。

`<ServiceName>_FooServer`具有嵌入式`grpc.ServerStream`和以下接口：
```Golang
type <ServiceName>_FooServer interface {
	SendAndClose(*MsgA) error
	Recv() (*MsgB, error)
	grpc.ServerStream
}
```
服务器端处理程序可以重复调用此参数的`Recv`方法，以便从客户端接收完整的消息流。一旦到达流的末尾，则`Recv`返回`(nil, io.EOF)`。通过调用此`<ServiceName>_FooServer`参数上的`SendAndClose`方法来发送来自服务器的单个响应消息。请注意，`SendAndClose`必须仅调用一次。

### 双向流方法
这些方法在生成的服务接口上具有以下签名：
```Golang
Foo(<ServiceName>_FooServer) error
```
在这种情况下，`<ServiceName>_FooServer`可用于访问客户端到服务器消息流和服务器到客户端消息流。 `<ServiceName>_FooServer`具有嵌入式`grpc.ServerStream`和以下接口：
```Golang
type <ServiceName>_FooServer interface {
	Send(*MsgA) error
	Recv() (*MsgB, error)
	grpc.ServerStream
}
```
服务器端处理程序可以重复调用此参数`Recv`方法，以读取客户端到服务器的消息流。 一旦到达客户端到服务器流的末尾，则`Recv`返回`(nil, io.EOF)`。通过重复调用此`ServiceName>_FooServer`参数上的`Send`方法来发送响应服务器到客户端的消息流。服务器到客户端流的流结束由`return`方法处理程序的指示。

## 生成的客户端接口上的方法
用于客户端的使用，每一个`.proto`文件中的`service Bar`也生成方法：`func BarClient(cc *grpc.ClientConn) BarClient`，它返回的具体实现`BarClient`接口（在此具体的实现也在所生成的`.pb.go`文件中）。

### 一元方法
这些方法在生成的客户端`stub`上具有以下签名：
```Golang
(ctx context.Context, in *MsgA, opts ...grpc.CallOption) (*MsgB, error)
```
在这种上下文下，`MsgA`是从客户端到服务器的单个请求，`MsgB`其中包含从服务器发送回的响应。

### 服务器流方法
这些方法在生成的客户端`stub`上具有以下签名：
```Golang
Foo(ctx context.Context, in *MsgA, opts ...grpc.CallOption) (<ServiceName>_FooClient, error)
```
在此上下文下，`<ServiceName>_FooClient`代表服务器到客户端`stream`的`MsgB`消息。

此流具有嵌入式`grpc.ClientStream`和以下接口：
```Golang
type <ServiceName>_FooClient interface {
	Recv() (*MsgB, error)
	grpc.ClientStream
}
```
当客户端在`stub`上调用`Foo`方法时，流开始。然后，客户端可以在返回的`<ServiceName>_FooClient` 流上重复调用该`Recv`方法，以读取服务器到客户端的响应流。一旦服务器到客户端流被完全读取，此`Recv`方法将返回`(nil, io.EOF)`。

### 客户端流传输方法
这些方法在生成的客户端`stub`上具有以下签名：
```Golang
Foo(ctx context.Context, opts ...grpc.CallOption) (<ServiceName>_FooClient, error)
```
在此上下文下，`<ServiceName>_FooClient`代表客户机到服务器`stream`的`MsgA`消息。

`<ServiceName>_FooClient`具有嵌入式`grpc.ClientStream`和以下接口：
```Golang
type <ServiceName>_FooClient interface {
	Send(*MsgA) error
	CloseAndRecv() (*MsgA, error)
	grpc.ClientStream
}
```
当客户端在`stub`上调用`Foo`方法时，流开始。然后，客户端可以在返回的`<ServiceName>_FooClient`流上重复调用`Send`方法，以发送客户端到服务器的消息流。`CloseAndRecv`为了关闭客户端到服务器流并从服务器接收单个响应消息，必须仅一次调用此流上的方法。

### 双向流方法
这些方法在生成的客户端`stub`上具有以下签名：
```Golang
Foo(ctx context.Context, opts ...grpc.CallOption) (<ServiceName>_FooClient, error)
```
在这种上下文下，`<ServiceName>_FooClient`代表客户端到服务器和服务器到客户端消息流。

`<ServiceName>_FooClient`具有嵌入式`grpc.ClientStream`和以下接口：
```Golang
type <ServiceName>_FooClient interface {
	Send(*MsgA) error
	Recv() (*MsgB, error)
	grpc.ClientStream
}
```
当客户端在存根上调用`Foo`方法时，流开始。然后，客户端可以在返回的`<SericeName>_FooClient`流上重复调用`Send`方法，以发送客户端到服务器的消息流。客户端还可以重复调用`Recv`此流，以接收完整的服务器到客户端消息流。

服务器到客户端流的流结束由流的`Recv`方法上的`(nil, io.EOF)`返回值指示。客户端到服务器流的流结束可以通过`CloseSend`在流上调用方法来从客户端指示。

## 包和命名空间
当使用`protoc`调用编译器时`--go_out=plugins=grpc:``proto package`到 `Go`包翻译的工作原理与`protoc-gen-go`不使用`grpc`插件的情况下使用插件相同。

因此，例如，如果`foo.proto`声明自己位于`package foo`，则生成的`foo.pb.go`文件也将位于`Go`包中`foo`。