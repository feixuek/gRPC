# 教程

常见用例的面向任务的演练


本文档向您介绍`gRPC`和`protocol buffer`。`gRPC`将`protocol buffer`用作其接口定义语言（`IDL`）和其基础消息交换格式。如果您不熟悉`gRPC`和/或`protocol buffer`，请阅读！如果您只是想深入了解`gRPC`的实际应用，请参阅[快速入门](https://grpc.io/docs/quickstart/)。

## 总览
在`gRPC`中，客户端应用程序可以直接调用在其他计算机上的服务器应用程序上的方法，就好像它是本地对象一样，从而使您更轻松地创建分布式应用程序和服务。与许多`RPC`系统一样，`gRPC`围绕定义服​​务的思想，指定可通过其参数和返回类型远程调用的方法。在服务器端，服务器实现此接口并运行`gRPC`服务器以处理客户端调用。在客户端，客户端具有一个存根（在某些语言中仅称为客户端），提供与服务器相同的方法。

从`Google`内部的服务器到您自己的台式机，`gRPC`客户端和服务器可以在各种环境中运行并相互通信，并且可以使用`gRPC`支持的任何语言编写。因此，例如，您可以使用`Go`，`Python`或`Ruby`中的客户端轻松地用`Java`创建`gRPC服`务器。此外，最新的`Google API`的接口将具有`gRPC`版本，可让您轻松地在应用程序中构建`Google`功能。

![](images/langing.svg)

## 使用`protocol buffer`
默认情况下，`gRPC`使用`protocol buffer`（`Google`的成熟的开源机制）来序列化结构化数据（尽管它可以与其他数据格式（例如`JSON`）一起使用）。这是它的工作原理的快速介绍。如果您已经熟悉`protocol buffer`，请随时跳到下一部分。

使用`protocol buffer`的第一步是为要在原始文件中序列化的数据定义结构：这是带有`.proto`扩展名的普通文本文件。`protocol buffer`数据被构造为消息，其中每个消息都是信息的小逻辑记录，其中包含一系列称为字段的名称/值对。这是一个简单的示例：
```proto
message Person {
  string name = 1;
  int32 id = 2;
  bool has_ponycopter = 3;
}
```

然后，一旦指定了数据结构，就可以使用`protocol buffer`编译器`protoc`从原型定义中以首选语言生成数据访问类。为每个字段提供了简单的访问器`name()`和`set_name()`方法，例如，以及将整个结构序列化为原始字节或从原始字节中解析出整个结构的方法。因此，例如，如果您选择的语言是`C++`，则在上面的示例中运行编译器将生成一个名为的类`Person`。然后，您可以在应用程序中使用此类来填充，序列化和检索`Person``protocol buffer`消息。

您可以在普通的原始文件中定义`gRPC`服务，并使用`RPC`方法参数和返回类型指定为`protocol buffer`消息：
```proto
// The greeter service definition.
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
`gRPC` `protoc`与特殊的`gRPC`插件一起使用，可从您的原型文件生成代码：您将生成`gRPC`客户端和服务器代码，以及用于填充，序列化和检索消息类型的常规`protocol buffer`代码。您将在下面看到一个示例。

要了解有关`protocol buffer`的更多信息，包括如何`protoc`使用所选语言与`gRPC`插件一起安装，请参阅 [`protocol buffer`文档](https://developers.google.com/protocol-buffers/docs/overview)。

## `protocol buffer`版本
虽然 `protocol buffer`已可供开放源代码用户使用一段时间，但本站点上的大多数示例都使用`protocol buffer`版本3（`proto3`），该`protocol buffer`的语法略有简化，提供了一些有用的新功能并支持更多语言。`Proto3`目前可用于`Java`，`C++`，`Dart`，`Python`，`Objective-C`，`C＃`，`lite-runtime（Android Java）`，`Ruby`和`JavaScript`，它们来自 `protocol buffer``GitHub repo`，以及来自[`golang/protobuf GitHub repo`](https://github.com/golang/protobuf)的`Go`语言生成器 ，正在开发更多语言。您可以在[`proto3`语言指南](https://developers.google.com/protocol-buffers/docs/proto3)和每种语言的[参考文档](https://developers.google.com/protocol-buffers/docs/reference/overview)中找到更多信息。参考文档还包括`.proto`[文件格式规范](https://developers.google.com/protocol-buffers/docs/reference/proto3-spec)。

通常，虽然您可以使用`proto2`（当前的默认`protocol buffer`版本），但建议您将`proto3`与`gRPC`一起使用，因为它可以使用所有`gRPC`支持的语言，并且可以避免`proto3`服务器与`proto2`客户端通信时出现的兼容性问题。反之亦然。