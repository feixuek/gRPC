# 错误处理

本页介绍`gRPC`如何处理错误，包括`gRPC`的内置错误代码。可以在[here](https://github.com/avinassh/grpc-errors)中找到不同语言的示例代码。


## 标准错误模型
正如您将在概念文档和示例中看到的那样，当`gRPC`调用成功完成时，服务器`OK`会向客户端返回状态（取决于语言，该`OK`状态可能会或可能不会直接在您的代码中使用）。但是，如果通话不成功怎么办？

如果发生错误，则`gRPC`会返回其错误状态代码之一，并带有可选的字符串错误消息，该消息提供有关所发生事件的更多详细信息。错误信息可用于所有支持的语言的`gRPC`客户端。

## 更丰富的错误模型
上述错误模型是官方的`gRPC`错误模型，受所有`gRPC`客户端/服务器库支持，并且独立于`gRPC`数据格式（无论`protocol buffer`还是其他）。您可能已经注意到，它的功能非常有限，并且不包含传达错误详细信息的功能。

如果您在使用`protocol buffer`为您的数据格式，不过，你不妨考虑使用开发和所描述的由谷歌所使用的更丰富的误差模型 这里。此模型使服务器可以返回，而客户端可以使用表示为一个或多个`probubuf`消息的其他错误详细信息。它还进一步指定了一组[标准的错误消息类型](https://github.com/googleapis/googleapis/blob/master/google/rpc/error_details.proto)， 以涵盖最常见的需求（例如无效参数，配额违反和堆栈跟踪）。此额外错误信息的`protobuf`二进制编码在响应中作为尾随元数据提供。

`C++`，`Go`，`Java`，`Python`和`Ruby`库已经支持这种更丰富的错误模型，至少g`rpc-web`和`Node.js`库在请求它方面存在开放性问题。如果需要，其他语言库可能会在将来增加支持，因此如果有兴趣，请检查其`github`存储库。但是请注意，用`C`编写的`grpc-core`库可能永远不会支持它，因为它是与数据格式无关的。

如果您不使用`protocol buffer`，则可以使用类似的方法（将错误详细信息放在跟踪响应元数据中），但是您可能需要查找或开发用于访问此数据的库支持，以便在您的程序中实际使用它。蜜蜂。

但是，在决定是否使用这种扩展错误模型时，需要注意一些重要的注意事项，包括：

- 就错误详细信息有效负载的要求和期望而言，扩展错误模型的库实现在各个语言之间可能不一致
- 现有代理，记录器和其他标准`HTTP`请求处理器无法查看错误详细信息，因此无法利用它们进行监视或其他目的
- 预告片中的其他错误详细信息会干扰行头阻塞，并且由于更频繁的缓存未命中而将降低`HTTP/2`标头压缩效率
- 较大的错误详细信息有效负载可能会遇到协议限制（例如最大报头大小），从而有效地丢失了原始错误

## 错误状态码
`gRPC`在各种情况下都会引发错误，从网络故障到未经身份验证的连接，每种错误都与特定的状态码相关联。所有`gRPC`语言均支持以下错误状态代码。

### 一般错误

| 案例 | 	状态码 |
| --- | --- |
| 客户申请取消了请求 | 	`GRPC_STATUS_CANCELLED` |
| 服务器返回状态之前的截止日期已过期 |	`GRPC_STATUS_DEADLINE_EXCEEDED` |
| 在服务器上找不到方法 |	`GRPC_STATUS_UNIMPLEMENTED` |
| 服务器关闭	| `GRPC_STATUS_UNAVAILABLE` |
|服务器引发异常（或执行其他操作（除了返回状态代码以终止`RPC`））	| `GRPC_STATUS_UNKNOWN` |

### 网络故障

| 案例 |	状态码 |
| --- | --- |
| 在截止日期到期前没有数据传输。也适用于在截止期限到期之前传输某些数据并且未检测到其他故障的情况 | `GRPC_STATUS_DEADLINE_EXCEEDED` |
| 连接断开之前传输的一些数据（例如，请求元数据已写入`TCP`连接） |	 `GRPC_STATUS_UNAVAILABLE` |

### 协议错误

| 案例 |	状态码 |
| --- | --- |
| 无法解压缩，但支持压缩算法 |	`GRPC_STATUS_INTERNAL` |
| 服务器不支持客户端使用的压缩机制 |	`GRPC_STATUS_UNIMPLEMENTED` |
| 达到流量控制资源限制 |	`GRPC_STATUS_RESOURCE_EXHAUSTED` |
| 违反流量控制协议 |	`GRPC_STATUS_INTERNAL` |
| 解析返回状态时出错 | 	`GRPC_STATUS_UNKNOWN` |
| 未经身份验证：凭据无法获取元数据 | `GRPC_STATUS_UNAUTHENTICATED` |
| 授权元数据中设置的主机无效 |	 `GRPC_STATUS_UNAUTHENTICATED` |
| 错误解析响应`protocol buffer` |	 `GRPC_STATUS_INTERNAL` |
| 错误解析请求`protocol buffer` | `GRPC_STATUS_INTERNAL` |
