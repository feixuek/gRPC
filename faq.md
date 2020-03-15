以下是一些常见问题。希望你在这里找到答案:-)

## 什么是`gRPC`？
`gRPC`是一个现代的，开源的远程过程调用（`RPC`）框架，可以在任何地方运行。它使客户端和服务器应用程序可以透明地进行通信，并使构建连接的系统更加容易。

阅读较长的[动机与设计原则](https://grpc.io/blog/principles/)一书，以了解创建`gRPC`的背景。

## `gRPC`代表什么？
`gRPC Remote Procedure Calls`

## 我为什么要使用`gRPC`？
主要使用场景：

- 低延迟，高度可扩展的分布式系统。
- 开发与云服务器通信的移动客户端。
- 设计一个新的协议，该协议必须准确，高效且独立于语言。
- 分层设计以实现扩展，例如。身份验证，负载平衡，日志记录和监视等

## 谁在使用它，为什么？
`gRPC`是[`Cloud Native Computing Foundation（CNCF）`项目](https://www.cncf.io/)。

长期以来，`Google`在`gRPC`中一直使用许多底层技术和概念。`Google`的几种云产品和`Google`外部`API`中都使用了当前的实现。它也被用于`Square`，`Netflix`，`CoreOS`，`Docker`，`CockroachDB`，`Cisco`，`Juniper网络公司`和许多其他组织和个人。

## 支持哪些编程语言？
请参阅[官方支持的语言和平台](https://grpc.io/about/#officially-supported-languages-and-platforms)。

## 如何开始使用`gRPC`？
您可以按照[此处](https://grpc.io/docs/quickstart/)的说明开始安装`gRPC` 。或转到[`gRPC GitHub org`](https://github.com/grpc)页面，选择您感兴趣的运行时或语言，然后按照`README`说明进行操作。

## `gRPC`使用哪个许可证？
所有实现均在[`Apache 2.0`](https://github.com/grpc/grpc/blob/master/LICENSE)下获得许可 。

## 我该怎么贡献？
非常欢迎贡献者，并且存储库托管在`GitHub`上。我们期待社区的反馈，补充和错误。个人贡献者和公司贡献者都需要签署我们的`CLA`。如果您有关于`gRPC`的项目的想法，请阅读准则并在[此处](https://github.com/grpc/grpc-community/blob/master/CONTRIBUTING.md)提交 。`GitHub`上的`gRPC`生态系统组织下的项目列表越来越多 。

## 文档在哪里？
在`grpc.io`上查看[文档](https://grpc.io/docs/)。

## 路线图是什么？
`gRPC`项目具有`RFC`流程，通过该流程可以设计并批准新功能以进行实施。在此存储库中对它们进行了跟踪 。

## `gRPC`版本支持多长时间？
`gRPC`项目不执行`LTS`发布。根据上述滚动发布模型，我们支持当前的最新版本以及之前的版本。这里的支持表示错误修复和安全修复。

## 最新的`gRPC`版本是什么？
最新的发行标签是`v1.27.2`。

## `gRPC`何时发布？
`gRPC`项目在主分支的尖端始终稳定的模型中工作。该项目（跨各种运行时）旨在尽最大努力每6周发布一次检查点发布。在此处查看发布时间表 。

## 我可以在浏览器中使用它吗？
该[`GRPC-Web`](https://github.com/grpc/grpc-web)项目全面上市。

## 我可以将`gRPC`与我喜欢的数据格式（`JSON`，`Protobuf`，`Thrift`，`XML`）一起使用吗？
是。`gRPC`被设计为可扩展的，以支持多种内容类型。初始版本包含对`Protobuf`的支持，并在不同的成熟度级别上对其他内容类型（例如`FlatBuffers`和`Thrift`）提供外部支持。

## `gRPC`如何帮助移动应用程序开发？
`gRPC`和`Protobuf`提供了一种简便的方法来精确定义服务并自动为`iOS`，`Android`和提供后端的服务器生成可靠的客户端库。客户端可以利用高级流传输和连接功能，这些功能有助于节省带宽，通过更少的`TCP`连接执行更多操作，并节省`CPU`使用率和电池寿命。

## 为什么`gRPC`比`HTTP/2`上的任何二进制`blob`更好？
这很大程度上就是`gRPC`的作用。但是，`gRPC`还是一组库，它们将在常见`HTTP`库通常不提供的平台之间一致地提供更高级别的功能。此类功能的示例包括：

- 在应用层与流控制进行交互
- 级联取消通话
- 负载平衡和故障转移

## 为什么`gRPC`比`REST`更好/更差？
`gRPC`很大程度上遵循`HTTP/2`上的`HTTP`语义，但我们明确允许全双工流传输。我们从典型的`REST`约定中脱颖而出，因为在呼叫分配过程中出于性能原因，我们使用静态路径，因为从路径，查询参数和有效内容主体中解析呼叫参数会增加延迟和复杂性。我们还确定了一系列错误，我们认为这些错误比`HTTP`状态代码更直接适用于`API`用例。

## 您如何发音gRPC？
`Jee-Arr-Pee-See`。