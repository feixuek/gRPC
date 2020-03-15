# `gRPC`基础知识-`Go`


本教程提供了`Go`程序员使用`gRPC`的基本介绍。

通过遍历此示例，您将学习如何：

- 在`.proto`文件中定义服务。
- 使用`protocol buffer`编译器生成服务器和客户端代码。
- 使用`Go gRPC API`为您的服务编写一个简单的客户端和服务器。

假定您已阅读概述，并且熟悉`protocol buffer`。请注意，本教程中的示例使用`protocol buffer`语言的`proto3`版本：您可以在`proto3`语言指南和`Go`生成的代码指南中找到更多信息 。

## 为什么要使用`gRPC`？
我们的示例是一个简单的路由映射应用程序，它使客户端可以获取有关其路由功能的信息，创建其路由的摘要以及与服务器和其他客户端交换路由信息（例如流量更新）。

借助`gRPC`，我们可以在`.proto`文件中一次定义我们的服务，并以`gRPC`支持的任何语言来实现客户端和服务器，而这些语言又可以在从`Google`内的服务器到您自己的平板电脑的各种环境中运行-`gRPC`为您处理所有不同语言和环境通信之间的复杂性。我们还获得了使用`protocol buffer`的所有优点，包括有效的序列化，简单的`IDL`和轻松的接口更新。

## 示例代码和设置
本教程的示例代码位于[`grpc/grpc-go/examples/route_guide`](https://github.com/grpc/grpc-go/tree/master/examples/route_guide)中。要下载示例，请`grpc-go`通过运行以下命令来克隆存储库：
```bash
$ go get google.golang.org/grpc
```
然后将当前目录更改为`grpc-go/examples/route_guide`：
```bash
$ cd $GOPATH/src/google.golang.org/grpc/examples/route_guide
```
您还应该安装相关工具来生成服务器和客户端接口代码-如果尚未安装，请按照[`Go`快速入门指南](https://grpc.io/docs/quickstart/go/)中的设置说明进行操作 。

## 定义服务
我们的第一步（您将从概述中了解 ）是使用`protocol buffer`定义`gRPC`服务以及方法请求和响应类型 。您可以在`examples/route_guide/routeguide/route_guide.proto`中看到完整的`.proto`文件 。

要定义服务，请在`.proto`文件中指定一个`service`名称：
```proto
service RouteGuide {
   ...
}
```
然后，在服务定义中定义`rpc`方法，并指定其请求和响应类型。`gRPC`允许您定义四种服务方法，所有这些方法都在`RouteGuide`服务中使用：

- 一个简单的`RPC`，客户端使用`stub`将请求发送到服务器，然后等待响应返回，就像正常的函数调用一样。
```proto
// Obtains the feature at a given position.
rpc GetFeature(Point) returns (Feature) {}
```
- 服务器端流`RPC`，其中客户端发送请求到服务器，并获得一个流中读取消息的序列后面。客户端从返回的流中读取，直到没有更多消息为止。如我们的示例所示，您可以通过将`stream`关键字放在响应类型之前来指定服务器端流方法。
```proto
// Obtains the Features available within the given Rectangle.  Results are
// streamed rather than returned at once (e.g. in a response message with a
// repeated field), as the rectangle may cover a large area and contain a
// huge number of features.
rpc ListFeatures(Rectangle) returns (stream Feature) {}
```

- 客户端流传输的`RPC`，其中客户端将消息写入的序列，并且将它们发送到服务器，再次使用提供的流。客户端写完消息后，它将等待服务器读取所有消息并返回其响应。您可以通过将`stream`关键字放在请求类型之前来指定客户端流方法。
```proto
// Accepts a stream of Points on a route being traversed, returning a
// RouteSummary when traversal is completed.
rpc RecordRoute(stream Point) returns (RouteSummary) {}
```
- 双向流`RPC`双方都派出使用读写流的消息序列。这两个流独立运行，因此客户端和服务器可以按照自己喜欢的顺序进行读写：例如，服务器可以在写响应之前等待接收所有客户端消息，或者可以先读取消息再写入消息，或其他一些读写组合。每个流中的消息顺序都会保留。您可以通过`stream` 在请求和响应之前放置关键字来指定这种类型的方法。
```proto
// Accepts a stream of RouteNotes sent while a route is being traversed,
// while receiving other RouteNotes (e.g. from other users).
rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}
```

我们的`.proto`文件还包含用于服务方法中所有请求和响应类型的`protocol buffer`消息类型定义-例如，以下是`Point`消息类型：
```proto
// Points are represented as latitude-longitude pairs in the E7 representation
// (degrees multiplied by 10**7 and rounded to the nearest integer).
// Latitudes should be in the range +/- 90 degrees and longitude should be in
// the range +/- 180 degrees (inclusive).
message Point {
  int32 latitude = 1;
  int32 longitude = 2;
}
```

## 生成客户端和服务器代码
接下来，我们需要根据`.proto`服务定义生成`gRPC`客户端和服务器接口。我们使用`protoc`带有特殊`gRPC Go`插件的`protocol buffer`编译器进行此操作。这类似于我们在快速入门指南中所做的

从`route_guide`示例目录运行：
```bash
 protoc -I routeguide/ routeguide/route_guide.proto --go_out=plugins=grpc:routeguide
```
运行此命令将`routeguide`在`route_guide`示例目录下的目录中生成以下文件：
```Golang
route_guide.pb.go
```
其中包含：

- 用于填充，序列化和检索我们的请求和响应消息类型的所有`protocol buffer`代码
- 客户端使用服务中定义的方法调用的接口类型（或`stub`）`RouteGuide`。
- 服务器要实现的接口类型，以及`RouteGuide`服务中定义的方法。

## 创建服务器
首先让我们看一下如何创建`RouteGuide`服务器。如果仅对创建`gRPC`客户端感兴趣，则可以跳过本节，直接转到 创建客户端（尽管您可能仍然会发现它很有趣！）。

使我们的`RouteGuide`服务发挥作用有两个部分：

- 实现根据我们的服务定义生成的服务接口：完成服务的实际“工作”。
- 运行`gRPC`服务器以侦听来自客户端的请求，并将其分派到正确的服务实现。

您可以在`grpc-go/examples/route_guide/server/server.go`中找到我们的`RouteGuide`示例服务器 。让我们仔细看看它是如何工作的。

### 实现`RouteGuide`
如您所见，我们的服务器具有`routeGuideServer`实现所生成`RouteGuideServer`接口的`struct`类型：
```Golang
type routeGuideServer struct {
        ...
}
...

func (s *routeGuideServer) GetFeature(ctx context.Context, point *pb.Point) (*pb.Feature, error) {
        ...
}
...

func (s *routeGuideServer) ListFeatures(rect *pb.Rectangle, stream pb.RouteGuide_ListFeaturesServer) error {
        ...
}
...

func (s *routeGuideServer) RecordRoute(stream pb.RouteGuide_RecordRouteServer) error {
        ...
}
...

func (s *routeGuideServer) RouteChat(stream pb.RouteGuide_RouteChatServer) error {
        ...
}
...
```

### 简单的`RPC`
`routeGuideServer`实现我们所有的服务方法。首先让我们看一下最简单的类型`GetFeature`，该类型仅从客户端获取一个`Point`，然后从其数据库中返回相应的特征信息`Feature`。
```Golang
func (s *routeGuideServer) GetFeature(ctx context.Context, point *pb.Point) (*pb.Feature, error) {
	for _, feature := range s.savedFeatures {
		if proto.Equal(feature.Location, point) {
			return feature, nil
		}
	}
	// No feature was found, return an unnamed feature
	return &pb.Feature{"", point}, nil
}
```
该方法将传递给`RPC`的上下文对象和客户端的`Point` `protocol buffer`请求。它返回一个`protocol buffer`对象`Feature`和带有响应信息`error`。在方法中，我们`Feature` 使用适当的信息填充，然后将`return`其与`nil`错误一起告诉`gRPC`我们已经完成了对`RPC`的处理，并且`Feature`可以将其返回给客户端。

### 服务器端流式`RPC`
现在，让我们看一下其中的流式`RPC`。`ListFeatures`是服务器端的流式`RPC`，因此我们需要将多个`Features`发送回客户端。
```Golang
func (s *routeGuideServer) ListFeatures(rect *pb.Rectangle, stream pb.RouteGuide_ListFeaturesServer) error {
	for _, feature := range s.savedFeatures {
		if inRange(feature.Location, rect) {
			if err := stream.Send(feature); err != nil {
				return err
			}
		}
	}
	return nil
}
```
如您所见，这次我们没有获得简单的请求和响应对象，而是获得了一个请求对象（`Rectangle`客户端希望在其中找到`Feature`）和一个特殊 `RouteGuide_ListFeaturesServer`对象来编写我们的响应。

在该方法中，我们填充了需要返回的尽可能多的`Feature`对象，使用`Send()`方法将它们写入`RouteGuide_ListFeaturesServer`。最后，就像在简单的`RPC`中一样，我们返回一个`nil`错误来告诉`gRPC`我们已经完成了响应的编写。如果此调用中发生任何错误，我们将返回一个非`nil`错误；`gRPC`层会将其转换为适当的`RPC`状态，以在线上发送。

### 客户端流式`RPC`
现在让我们看一些更复杂的东西：客户端流方法`RecordRoute`，从客户端获取`Point`的流，并返回`RouteSummary`包含有关其行程的信息的单个流。如您所见，这一次该方法根本没有`request`参数。相反，它获得一个`RouteGuide_RecordRouteServer`流，服务器可以使用该流来读取和写入消息-它可以使用其`Recv()`方法接收客户端消息，并使用其方法返回其单个响应`SendAndClose()` 。
```Golang
func (s *routeGuideServer) RecordRoute(stream pb.RouteGuide_RecordRouteServer) error {
	var pointCount, featureCount, distance int32
	var lastPoint *pb.Point
	startTime := time.Now()
	for {
		point, err := stream.Recv()
		if err == io.EOF {
			endTime := time.Now()
			return stream.SendAndClose(&pb.RouteSummary{
				PointCount:   pointCount,
				FeatureCount: featureCount,
				Distance:     distance,
				ElapsedTime:  int32(endTime.Sub(startTime).Seconds()),
			})
		}
		if err != nil {
			return err
		}
		pointCount++
		for _, feature := range s.savedFeatures {
			if proto.Equal(feature.Location, point) {
				featureCount++
			}
		}
		if lastPoint != nil {
			distance += calcDistance(lastPoint, point)
		}
		lastPoint = point
	}
}
```
在方法主体中，我们使用`RouteGuide_RecordRouteServer`的`Recv()`方法重复读取客户机对请求对象的请求（在本例中为`Point`），直到没有更多消息为止：服务器需要检查`Read()`每次调用后返回的错误。如果是`nil`，则该流仍然良好，可以继续读取；如果`io.EOF`是消息流已结束，则服务器可以返回`RouteSummary`。如果它具有任何其他值，我们将按“原样”返回错误，以便`gRPC`层将其转换为`RPC`状态。

### 双向流式`RPC`
最后，让我们看一下双向流式`RPC` `RouteChat()`。
```Golang
func (s *routeGuideServer) RouteChat(stream pb.RouteGuide_RouteChatServer) error {
	for {
		in, err := stream.Recv()
		if err == io.EOF {
			return nil
		}
		if err != nil {
			return err
		}
		key := serialize(in.Location)
                ... // look for notes to be sent to client
		for _, note := range s.routeNotes[key] {
			if err := stream.Send(note); err != nil {
				return err
			}
		}
	}
}
```
这次，我们得到了一个`RouteGuide_RouteChatServer`流，就像在客户端流示例中那样，可用于读取和写入消息。然而，这一次，我们通过我们的方法的流返回值，而客户端仍在消息写入其消息流。

此处的读写语法与我们的客户端流方法非常相似，不同之处在于服务器使用流的`Send()`方法，而不是`SendAndClose()`因为服务器正在写多个响应。尽管双方总是会按照对方的编写顺序获得对方的消息，但是客户端和服务器都可以按照任何顺序进行读写–流完全独立地运行。

## 启动服务器
一旦实现了所有方法，我们还需要启动`gRPC`服务器，以便客户端可以实际使用我们的服务。以下代码段显示了我们如何为`RouteGuide`服务执行此操作：
```Golang
flag.Parse()
lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
if err != nil {
        log.Fatalf("failed to listen: %v", err)
}
grpcServer := grpc.NewServer()
pb.RegisterRouteGuideServer(grpcServer, &routeGuideServer{})
... // determine whether to use TLS
grpcServer.Serve(lis)
```
要构建和启动服务器，我们：

1. 使用`lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))`指定要用于侦听客户端请求的端口。
2. 使用`grpc.NewServer()`创建一个`gRPC`服务器实例。
3. 在`gRPC`服务器上注册我们的服务实现。
4. 调用`Serve()`与我们的端口细节做了阻塞等待在服务器上，直到进程被杀死或者`Stop()`被调用。

## 创建客户端
在本节中，我们将研究为我们的`RouteGuide`服务创建一个`Go`客户端。您可以在`grpc-go/examples/route_guide/client/client.go`中看到我们完整的示例客户端代码 。

### 创建`stub`
要调用服务方法，我们首先需要创建一个`gRPC`通道来与服务器通信。我们通过将服务器地址和端口号传递给`grpc.Dial()`以下内容来创建此文件 ：
```Golang
conn, err := grpc.Dial(*serverAddr)
if err != nil {
    ...
}
defer conn.Close()
```
如果您请求的服务需要，则可以在`grpc.Dial`使用`DialOptions`设置身份验证凭据（例如`TLS`，`GCE`凭据，`JWT`凭据-但是，我们不需要为我们的`RouteGuide`服务执行此操作。

设置`gRPC`通道后，我们需要一个客户端`stub`来执行`RPC`。我们使用从`.proto`生成的`pb`包中提供的`NewRouteGuideClient`方法来获得此信息。
```Golang
client := pb.NewRouteGuideClient(conn)
```
### 调用服务方法
现在让我们看看我们如何调用我们的服务方法。请注意，在`gRPC-Go`中，`RPC`在阻塞/同步模式下运行，这意味着`RPC`调用等待服务器响应，并且将返回响应或错误。

#### 简单的`RPC`
调用简单的`RPC GetFeature`与调用本地方法几乎一样简单。
```Golang
feature, err := client.GetFeature(context.Background(), &pb.Point{409146138, -746188906})
if err != nil {
        ...
}
```
如您所见，我们在之前获得的`stub`上调用该方法。在我们的方法参数中，我们创建并填充一个请求`protocol buffer`对象（在我们的示例中 `Point`）。我们还传递了一个`context.Context`对象，该对象可以根据需要更改`RPC`的行为，例如超时/取消运行中的`RPC`。如果调用没有返回错误，那么我们可以从服务器的第一个返回值中读取响应信息。
```Golang
log.Println(feature)
```

#### 服务器端流式`RPC`
在这里，我们称为服务器端流方法`ListFeatures`，该方法返回`geo`的流`Feature`。如果您已经阅读了创建服务器，其中的一些内容可能看起来非常熟悉-流式`RPC`在两侧都以类似的方式实现。
```Golang
rect := &pb.Rectangle{ ... }  // initialize a pb.Rectangle
stream, err := client.ListFeatures(context.Background(), rect)
if err != nil {
    ...
}
for {
    feature, err := stream.Recv()
    if err == io.EOF {
        break
    }
    if err != nil {
        log.Fatalf("%v.ListFeatures(_) = _, %v", client, err)
    }
    log.Println(feature)
}
```
就像在简单的`RPC`中一样，我们为该方法传递一个上下文和一个请求。但是，我们没有取回响应对象，而是取回的实例`RouteGuide_ListFeaturesClient`。客户端可以使用`RouteGuide_ListFeaturesClient`流来读取服务器的响应。

我们使用`RouteGuide_ListFeaturesClient`的`Recv()`方法重复读取服务器对响应`protocol buffer`对象（在本例中为`Feature`）的响应， 直到没有更多消息为止：客户端需要检查每次调用`Recv()`后`err`返回的错误 。如果为`nil`，则信息流仍然良好，可以继续阅读；如果是`io.EOF`则消息流已结束；否则，必须存在一个`RPC`错误，该错误会通过传递`err`。

#### 客户端流式`RPC`
客户端流方法`RecordRoute`与服务器端方法相似，不同之处在于，我们仅向该方法传递上下文并获取`RouteGuide_RecordRouteClient`流，该流可用于写入和 读取消息。
```Golang
// Create a random number of random points
r := rand.New(rand.NewSource(time.Now().UnixNano()))
pointCount := int(r.Int31n(100)) + 2 // Traverse at least two points
var points []*pb.Point
for i := 0; i < pointCount; i++ {
	points = append(points, randomPoint(r))
}
log.Printf("Traversing %d points.", len(points))
stream, err := client.RecordRoute(context.Background())
if err != nil {
	log.Fatalf("%v.RecordRoute(_) = _, %v", client, err)
}
for _, point := range points {
	if err := stream.Send(point); err != nil {
		if err == io.EOF {
			break
		}
		log.Fatalf("%v.Send(%v) = %v", stream, point, err)
	}
}
reply, err := stream.CloseAndRecv()
if err != nil {
	log.Fatalf("%v.CloseAndRecv() got error %v, want %v", stream, err, nil)
}
log.Printf("Route summary: %v", reply)
```
该`RouteGuide_RecordRouteClient`有一个`Send()`，我们可以用它来发送请求到服务器的方法。使用完将客户的请求写入流后`Send()`，我们需要调用`CloseAndRecv()`该流以让`gRPC`知道我们已经完成了写入并期望收到响应。我们从`CloseAndRecv()`返回的`err`中获取`RPC`状态。如果状态为`nil`，则`from`的第一个返回值`CloseAndRecv()`将是有效的服务器响应。

#### 双向流式`RPC`
最后，让我们看一下双向流式`RPC RouteChat()`。与`RecordRoute`的情况一样，我们仅向方法传递一个上下文对象，并获取一个可用于写入和读取消息的流。然而，这一次，我们通过我们的方法的流返回值，而服务器还在写邮件给他们的消息流。
```Golang
stream, err := client.RouteChat(context.Background())
waitc := make(chan struct{})
go func() {
	for {
		in, err := stream.Recv()
		if err == io.EOF {
			// read done.
			close(waitc)
			return
		}
		if err != nil {
			log.Fatalf("Failed to receive a note : %v", err)
		}
		log.Printf("Got message %s at point(%d, %d)", in.Message, in.Location.Latitude, in.Location.Longitude)
	}
}()
for _, note := range notes {
	if err := stream.Send(note); err != nil {
		log.Fatalf("Failed to send a note: %v", err)
	}
}
stream.CloseSend()
<-waitc
```
除了`CloseSend()`在完成调用后使用流的方法外，此处的读写语法与我们的客户端流方法非常相似。尽管双方总是会按照对方的编写顺序获得对方的消息，但是客户端和服务器都可以按照任何顺序进行读写–流完全独立地运行。

## 试试看！
从示例目录工作：
```bash
$ cd $GOPATH/src/google.golang.org/grpc/examples/route_guide
```
运行服务器：
```bash
$ go run server/server.go
```
从另一个终端，运行客户端：
```bash
$ go run client/client.go
```