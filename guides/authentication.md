# 认证方式

本文档概述了`gRPC`身份验证，包括我们内置的受支持的身份验证机制，如何插入您自己的身份验证系统以及如何以我们支持的语言使用`gRPC`身份验证的示例。


## 总览
`gRPC`设计为可与多种身份验证机制配合使用，从而可以轻松安全地使用`gRPC`与其他系统进行通讯。您可以使用我们支持的机制-带有或不带有基于`Google`令牌的身份验证的`SSL/TLS`-或者您可以通过扩展我们提供的代码来插入自己的身份验证系统。

`gRPC`还提供了一个简单的身份验证`API`，`Credentials`可让您在创建频道或进行呼叫时提供所有必要的身份验证信息。

## 支持的身份验证机制
`gRPC`内置了以下身份验证机制：

- `SSL/TLS`：`gRPC`具有`SSL/TLS`集成，并促进使用`SSL/TLS`来认证服务器并加密客户端与服务器之间交换的所有数据。客户端可以使用可选机制来提供用于相互认证的证书。
- 基于`Google`令牌的身份验证：`gRPC`提供了一种通用机制（如下所述），可将基于元数据的凭据附加到请求和响应。某些身份验证流还提供了在通过`gRPC`访问`Google API`时获取访问令牌（通常是`OAuth2`令牌）的其他支持：您可以在下面的代码示例中了解其工作方式。通常，必须在通道上使用该机制以及`SSL/TLS`-`Google`不允许没有`SSL/TLS`的连接，并且大多数`gRPC`语言实现都不允许您在未加密的通道上发送凭据。

> 警告

> `Google`凭据只能用于连接到`Google`服务。将`Google`发行的`OAuth2`令牌发送到非`Google`服务可能会导致该令牌被盗并用于模拟客户端使用`Google`服务。

## 认证`API`
`gRPC`提供了基于凭据对象统一概念的简单身份验证`API`，可在创建整个`gRPC`通道或单个调用时使用该`API`。

### 凭证类型
凭证可以分为两种类型：

- 附加到`Channel`的通道凭据，例如`SSL`凭据。
- 呼叫凭证，附加到呼叫中（或在`C++`中的`ClientContext`）。

您也可以将它们合并到`CompositeChannelCredentials`中，从而可以指定通道的`SSL`详细信息以及通道上进行的每个呼叫的呼叫凭证。一个`CompositeChannelCredentials`相关联的`ChannelCredentials`和`CallCredentials`创建一个新的 `ChannelCredentials`。结果将发送`CallCredentials`与在通道上进行的每个呼叫组成的身份验证数据相关联。

例如，您可以创建一个`ChannelCredentials`通过`SslCredentials` 和`AccessTokenCredentials`。将结果应用于`Channel`时将为此通道上的每个呼叫发送适当的访问令牌。

个人也可以使用由`CompositeCallCredentials`组成的`CallCredentials`。`CallCredentials`当在呼叫中使用时，结果将触发发送与两者关联的身份验证数据`CallCredentials`。

### 使用客户端`SSL/TLS`
现在，让我们看看如何使用我们支持的身份验证机制之一`Credentials`。这是最简单的身份验证方案，客户端仅希望对服务器进行身份验证并加密所有数据。该示例使用`C++`，但所有语言的`API`均相似：您可以在下面的示例部分中了解如何以更多语言启用`SSL/TLS`。
```C++
// Create a default SSL ChannelCredentials object.
auto channel_creds = grpc::SslCredentials(grpc::SslCredentialsOptions());
// Create a channel using the credentials created in the previous step.
auto channel = grpc::CreateChannel(server_name, channel_creds);
// Create a stub on the channel.
std::unique_ptr<Greeter::Stub> stub(Greeter::NewStub(channel));
// Make actual RPC calls on the stub.
grpc::Status s = stub->sayHello(&context, *request, response);
```
对于高级用例，例如修改根`CA`或使用客户端证书，可以在`SslCredentialsOptions`传递给工厂方法的参数中设置相应的选项。

### 使用基于`Google`令牌的身份验证
`gRPC`应用程序可以使用一个简单的`API`来创建一个凭证，该凭证可以在各种部署方案中与`Google`进行身份验证。同样，我们的示例使用`C++`，但是您可以在示例部分中找到其他语言的示例。
```C++
auto creds = grpc::GoogleDefaultCredentials();
// Create a channel, stub and make RPC calls (same as in the previous example)
auto channel = grpc::CreateChannel(server_name, creds);
std::unique_ptr<Greeter::Stub> stub(Greeter::NewStub(channel));
grpc::Status s = stub->sayHello(&context, *request, response);
```
此通道凭据对象适用于使用服务帐户的应用程序以及在`Google Compute Engine（GCE）`中运行的应用程序 。在前一种情况下，服务帐户的私钥是从环境变量中命名的文件中加载的`GOOGLE_APPLICATION_CREDENTIALS`。密钥用于生成承载令牌，这些令牌附加到相应通道上的每个传出`RPC`。

对于在`GCE`中运行的应用程序，可以在`VM`设置期间配置默认服务帐户和相应的`OAuth2`范围。在运行时，此凭据处理与身份验证系统的通信以获得`OAuth2`访问令牌，并将它们附加到相应通道上的每个传出`RPC`。

### 扩展`gRPC`以支持其他身份验证机制
凭据插件`API`允许开发人员插入自己的凭据类型。这包括：

- `MetadataCredentialsPlugin`抽象类，其中包含纯虚`GetMetadata`需要由由开发者创建的子类来实现的方法。
- `MetadataCredentialsFromPlugin`功能，它创建了一个`CallCredentials`从`MetadataCredentialsPlugin`。

这是一个简单的凭据插件的示例，该插件在自定义标头中设置身份验证票证。
```C++
class MyCustomAuthenticator : public grpc::MetadataCredentialsPlugin {
 public:
  MyCustomAuthenticator(const grpc::string& ticket) : ticket_(ticket) {}

  grpc::Status GetMetadata(
      grpc::string_ref service_url, grpc::string_ref method_name,
      const grpc::AuthContext& channel_auth_context,
      std::multimap<grpc::string, grpc::string>* metadata) override {
    metadata->insert(std::make_pair("x-custom-auth-ticket", ticket_));
    return grpc::Status::OK;
  }

 private:
  grpc::string ticket_;
};

auto call_creds = grpc::MetadataCredentialsFromPlugin(
    std::unique_ptr<grpc::MetadataCredentialsPlugin>(
        new MyCustomAuthenticator("super-secret-ticket")));
```
通过在核心级别插入`gRPC`凭证实现，可以实现更深层次的集成。`gRPC`内部组件还允许使用其他加密机制来切换`SSL/TLS`。

## 例子
这些身份验证机制将在所有`gRPC`支持的语言中可用。以下各节说明上述身份验证和授权功能如何以每种语言显示：即将推出更多语言。

### `Go`
#### 基本情况-没有加密或身份验证
客户端：
```Golang
conn, _ := grpc.Dial("localhost:50051", grpc.WithInsecure())
// error handling omitted
client := pb.NewGreeterClient(conn)
// ...
```
服务器：
```Golang
s := grpc.NewServer()
lis, _ := net.Listen("tcp", "localhost:50051")
// error handling omitted
s.Serve(lis)
```
#### 通过服务器身份验证`SSL/TLS`
客户端：
```Golang
creds, _ := credentials.NewClientTLSFromFile(certFile, "")
conn, _ := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(creds))
// error handling omitted
client := pb.NewGreeterClient(conn)
// ...
```
服务器：
```Golang
creds, _ := credentials.NewServerTLSFromFile(certFile, keyFile)
s := grpc.NewServer(grpc.Creds(creds))
lis, _ := net.Listen("tcp", "localhost:50051")
// error handling omitted
s.Serve(lis)
```

#### 向Google进行身份验证
```Golang
pool, _ := x509.SystemCertPool()
// error handling omitted
creds := credentials.NewClientTLSFromCert(pool, "")
perRPC, _ := oauth.NewServiceAccountFromFile("service-account.json", scope)
conn, _ := grpc.Dial(
	"greeter.googleapis.com",
	grpc.WithTransportCredentials(creds),
	grpc.WithPerRPCCredentials(perRPC),
)
// error handling omitted
client := pb.NewGreeterClient(conn)
// ...
```