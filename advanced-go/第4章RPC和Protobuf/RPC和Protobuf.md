# 4.1 RPC入门

RPC是远程过程调用的简称，是分布式系统中不同节点间流行的通信方式。在互联网时代，RPC已经和IPC一样成为一个不可或缺的基础构件。

本节书中相关代码 https://github.com/StormSpirit22/code/tree/main/advanced-go/chapter4/4.1

## 4.1.3 跨语言的RPC



使用 tcp 服务而不是 Go 的 rpc 服务（即停掉 server)，命令行里输入 nc -l 1234

在 client 里运行 go run main.go

nc 命令行输出：

```json
{"method":"HelloService.Hello","params":["hello"],"id":0}
```

这是一个 clientRequest。服务端是serverRequest。clientRequest和serverRequest结构体的内容基本是一致的：

```go
type clientRequest struct {
    Method string         `json:"method"`
    Params [1]interface{} `json:"params"`
    Id     uint64         `json:"id"`
}

type serverRequest struct {
    Method string           `json:"method"`
    Params *json.RawMessage `json:"params"`
    Id     *json.RawMessage `json:"id"`
}
```

然后在 server 里运行 go run main.go，在命令行里输入

```shell
 echo -e '{"method":"HelloService.Hello","params":["hello"],"id":1}' | nc localhost 1234
```

返回的结果也是一个json格式的数据：

```json
{"id":1,"result":"hello:hello","error":null}
```

其中id对应输入的id参数，result为返回的结果，error部分在出问题时表示错误信息。对于顺序调用来说，id不是必须的。但是Go语言的RPC框架支持异步调用，当返回结果的顺序和调用的顺序不一致时，可以通过id来识别对应的调用。

返回的json数据也是对应内部的两个结构体：客户端是clientResponse，服务端是serverResponse。两个结构体的内容同样也是类似的：

```go
type clientResponse struct {
    Id     uint64           `json:"id"`
    Result *json.RawMessage `json:"result"`
    Error  interface{}      `json:"error"`
}

type serverResponse struct {
    Id     *json.RawMessage `json:"id"`
    Result interface{}      `json:"result"`
    Error  interface{}      `json:"error"`
}
```

因此无论采用何种语言，只要遵循同样的json结构，以同样的流程就可以和Go语言编写的RPC服务进行通信。这样我们就实现了跨语言的RPC。



## 4.3.1 客户端RPC的实现原理

一些铺垫，主要解释了 RPC client.Dial 的原理，说明了`client.send`方法调用是线程安全的，因此**可以从多个Goroutine同时向同一个RPC链接发送调用指令**。

## 4.3.2 基于RPC实现Watch功能

书中这部分代码在 [github](https://github.com/StormSpirit22/code/tree/main/advanced-go/chapter4/4.3/rpc_watch)

这一块代码基本功能就是观察 kv 存储是否改变。 rpc 服务端有一个  kv 存储 map 用于存储数据。 每次运行 watch 函数就会注册一个 watch id 和函数到服务的 filter 列表里。

```go
p.filter[id] = func(key string) { ch <- key }
```

在服务的  Set 函数被调用时，它会观察自己内部的一个的 key 的值是否改变，如果某个 key 的值改变了则会将该 key 遍历 filter 列表里每个函数进行调用：

```go
// 如果要修改某个 key，将对该 key 执行过滤器里的所有函数
if oldValue := p.m[key]; oldValue != value {
  for _, fn := range p.filter {
    fn(key)
  }
}
```

这样理论上每次修改 key 的值时，这个 key 就会经过通道通知到每个监听的 watch 函数，即返回到客户端输出。但事实上是如果先后运行 a, b 两个客户端，并且每次都修改 key  的值，那么 b 会等待 a  客户端调用的完成再开始监听，这样就不能实现同时监听的效果了。这是因为服务端的代码：

```go
for {
		conn, err := listener.Accept()
		if err != nil {
			log.Fatal("Accept error:", err)
		}
		fmt.Println("start conn ")

		// 这里会等到 conn 断掉再继续下一个连接
		rpc.ServeConn(conn)
	}
```

其实只要加个 go 就好了：

```go
go rpc.ServeConn(conn)
```

这样就能实现同时连接多个客户端，客户端同时监听了。其实之前的代码里也有。

## 4.3.3 反向RPC

反向RPC的内网服务将不再主动提供TCP监听服务，而是首先主动链接到对方的TCP服务器。然后基于每个建立的TCP链接向对方提供RPC服务。

而RPC客户端则需要在一个公共的地址提供一个TCP服务，用于接受RPC服务器的链接请求。

直接看代码 [反向rpc](https://github.com/StormSpirit22/code/tree/main/advanced-go/chapter4/4.3/reverse_rpc)



## 4.4.1 gRPC技术栈

Go语言的gRPC技术栈如图4-1所示：

![img](https://chai2010.cn/advanced-go-programming-book/images/ch4-1-grpc-go-stack.png)

*图4-1 gRPC技术栈*

最底层为TCP或Unix Socket协议，在此之上是HTTP/2协议的实现，然后在HTTP/2协议之上又构建了针对Go语言的gRPC核心库。应用程序通过gRPC插件生产的Stub代码和gRPC核心库通信，也可以直接和gRPC核心库通信。

## 4.4.2 gRPC入门

利用 Protobuf 的 grpc 插件，很容易可以通过一个 protobuf 代码生成服务端和客户端的 grpc 调用。

```go
type HelloServiceServer interface {
    Hello(context.Context, *String) (*String, error)
}

type HelloServiceClient interface {
    Hello(context.Context, *String, ...grpc.CallOption) (*String, error)
}
```

服务端代码：

```go
func main() {
	grpcServer := grpc.NewServer()
	proto.RegisterHelloServiceServer(grpcServer, new(HelloServiceImpl))

	lis, err := net.Listen("tcp", ":1234")
	if err != nil {
		log.Fatal(err)
	}
	grpcServer.Serve(lis)
}
```

客户端代码：

```go
func main() {
	conn, err := grpc.Dial("localhost:1234", grpc.WithInsecure())
	if err != nil {
		log.Fatal(err)
	}
	defer conn.Close()

	client := proto.NewHelloServiceClient(conn)
	reply, err := client.Hello(context.Background(), &proto.String{Value: "hello"})
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(reply.GetValue())
}
```

和普通的 rpc 代码非常相似。

如果从Protobuf的角度看，gRPC只不过是一个针对service接口生成代码的生成器。

## 4.4.3 gRPC流

`streaming rpc`相比于`simple rpc`来说可以很好的解决一个接口发送大量数据的场景。

比如一个订单导出的接口有20万条记录，如果使用`simple rpc`来实现的话。那么我们需要一次性接收到20万记录才能进行下一步的操作。但是如果我们使用`streaming rpc`那么我们就可以接收一条记录处理一条记录，直到所以的数据传输完毕。**这样可以较少服务器的瞬时压力，也更有及时性**。

而传统的RPC方法调用对于上传和下载较大数据量场景并不适合。同时传统RPC模式也不适用于对时间不确定的订阅和发布模式。

可以借助 protobuf 和 grpc 生成支持双向流的模式。

```protobuf
service HelloService {
    rpc Hello (String) returns (String);

    rpc Channel (stream String) returns (stream String);
}
```

关键字stream指定启用流特性，参数部分是接收客户端参数的流，返回值是返回给客户端的流。

重新生成代码可以看到接口中新增加的Channel方法的定义：

```go
type HelloServiceServer interface {
    Hello(context.Context, *String) (*String, error)
    Channel(HelloService_ChannelServer) error
}
type HelloServiceClient interface {
    Hello(ctx context.Context, in *String, opts ...grpc.CallOption) (
        *String, error,
    )
    Channel(ctx context.Context, opts ...grpc.CallOption) (
        HelloService_ChannelClient, error,
    )
}
```

在服务端的Channel方法参数是一个新的HelloService_ChannelServer类型的参数，可以用于和客户端双向通信。客户端的Channel方法返回一个HelloService_ChannelClient类型的返回值，可以用于和服务端进行双向通信。

HelloService_ChannelServer和HelloService_ChannelClient均为接口类型：

```go
type HelloService_ChannelServer interface {
    Send(*String) error
    Recv() (*String, error)
    grpc.ServerStream
}

type HelloService_ChannelClient interface {
    Send(*String) error
    Recv() (*String, error)
    grpc.ClientStream
}
```

可以发现服务端和客户端的流辅助接口均定义了Send和Recv方法用于流数据的双向通信。

具体代码在[grpc stream](https://github.com/StormSpirit22/code/tree/main/advanced-go/chapter4/4.4/grpc-stream)



## 4.4.4 发布和订阅模式

基于gRPC和pubsub包，提供一个跨网络的发布和订阅系统。

```protobuf
service PubsubService {
    rpc Publish (String) returns (String);
    rpc Subscribe (String) returns (stream String);
}
```

其中Publish是普通的RPC方法，Subscribe则是一个单向的流服务。

具体代码在[发布订阅](https://github.com/StormSpirit22/code/tree/main/advanced-go/chapter4/4.4/grpc-pub-sub)

代码主要实现了一个用于发布和订阅的服务器，客户端可以通过服务器发送消息和订阅。
