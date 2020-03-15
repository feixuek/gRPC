# `Golang`快速入门

本指南通过一个简单的示例使您开始使用`Go`中的`gRPC`。


## 先决条件
- `Golang`的版本是1.6版或更高。

有关安装说明，请参阅[`Go`的入门指南](https://golang.org/doc/install)。

### `gRPC`
使用以下命令安装`gRPC`。
```bash
$ go get -u google.golang.org/grpc
```

### `protocol buffer v3`
安装用于生成`gRPC`服务代码的协议编译器。最简单的方法是从此处下载适用于您的平台的预编译二进制文件(`protoc-<version>-<platform>.zip`)：https://github.com/google/protobuf/releases

- 解压缩该文件。
- 更新环境变量`PATH`以包括协议二进制文件的路径。

接下来，为Go安装protoc插件
```bash
$ go get -u github.com/golang/protobuf/protoc-gen-go
```
编译器插件`protoc-gen-go`将会安装在中`$GOBIN`，默认为`$GOPATH/bin`。`protoc`协议编译器必须在您的`PATH`目录中才能找到它。
```bash
$ export PATH=$PATH:$GOPATH/bin
```
## 下载示例
随同获取的`grpc`代码`go get google.golang.org/grpc`也包含示例。它们可以在以下目录下找到`$GOPATH/src/google.golang.org/grpc/examples`。

## 构建例子
转到示例目录
```bash
$ cd $GOPATH/src/google.golang.org/grpc/examples/helloworld
```
`gRPC`服务在`.proto`文件中定义，该文件用于生成相应的`.pb.go`文件。该`.pb.go`文件是由`protoc`编译器使用`.proto`文件编译生成的。

就本示例而言，该`helloworld.pb.go`文件已经生成（通过编译`helloworld.proto`），可以在以下目录中找到： `$GOPATH/src/google.golang.org/grpc/examples/helloworld/helloworld`

该`helloworld.pb.go`文件包含：

- 生成的客户端和服务器代码。
- 用于填充，序列化和检索我们`HelloRequest`和`HelloReply`消息类型的代码。

## 尝试一下！
要编译并运行服务器和客户端代码，`go run`可以使用该命令。在示例目录中：
```bash
$ go run greeter_server/main.go
```
从另一个终端：
```bash
$ go run greeter_client/main.go
```
如果一切顺利，您将在客户端输出中看到`Greeting: Hello world`。

恭喜你！您刚刚使用`gRPC`运行了客户端服务器应用程序。

## 更新`gRPC`服务
现在让我们看一下如何使用服务器上的附加方法更新应用程序，以供客户端调用。我们的`gRPC`服务是使用`protocol buffer`定义的；您可以在[什么是gRPC？](https://www.grpc.io/docs/guides/)和[`gRPC`基础知识：`Go`](https://www.grpc.io/docs/tutorials/basic/go/)中找到更多有关如何在`.proto `文件中定义服务的信息。现在，您只需要知道服务器和客户端“存根”都有一个`SayHello``RPC`方法，该方法从客户端获取`HelloRequest`参数并从服务器返回`HelloReply` ，并且该方法的定义如下：
```proto
// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```
让我们对其进行更新，以便该`Greeter`服务具有两种方法。确保您位于与上述（`$GOPATH/src/google.golang.org/grpc/examples/helloworld`）相同的示例目录中。

`helloworld/helloworld.proto`使用`SayHelloAgain` 具有相同请求和响应类型的新方法来编辑和更新它：
```proto
// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
  // Sends another greeting
  rpc SayHelloAgain (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```

## 生成`gRPC`代码
接下来，我们需要更新应用程序使用的`gRPC`代码以使用新的服务定义。从与上述（`$GOPATH/src/google.golang.org/grpc/examples/helloworld`）相同的示例目录中：
```bash
$ protoc -I helloworld/ helloworld/helloworld.proto --go_out=plugins=grpc:helloworld
```
这将通过我们的新更改重新生成`helloworld.pb.go`。

## 更新并运行应用程序
现在，我们有了新生成的服务器和客户端代码，但是仍然需要在示例应用程序的人工编写部分中实现并调用新方法。

### 更新服务器
编辑`greeter_server/main.go`并添加以下功能：
```Golang
func (s *server) SayHelloAgain(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
        return &pb.HelloReply{Message: "Hello again " + in.GetName()}, nil
}
```
### 更新客户端
编辑`greeter_client/main.go`以将以下代码添加到主要功能。
```Golang
r, err = c.SayHelloAgain(ctx, &pb.HelloRequest{Name: name})
if err != nil {
        log.Fatalf("could not greet: %v", err)
}
log.Printf("Greeting: %s", r.GetMessage())
```
### 运行
1. 运行服务器： 
```bash
$ go run greeter_server/main.go
```
2. 在另一个终端上，运行客户端：
```bash
$ go run greeter_client/main.go
```
您将看到以下输出：
```shell
Greeting: Hello world
Greeting: Hello again world
```

## 下一步是什么
- 在[什么是`gRPC`？](https://www.grpc.io/docs/guides/)和[`gRPC`概念](https://www.grpc.io/docs/guides/concepts/)中阅读有关`gRPC`如何工作的完整说明。
- 在[`gRPC`基础知识：`Go`](https://www.grpc.io/docs/tutorials/basic/go/)中完成更详细的教程 。
- 在其[参考文档](https://godoc.org/google.golang.org/grpc)中探索`gRPC Go`核心`API` 。