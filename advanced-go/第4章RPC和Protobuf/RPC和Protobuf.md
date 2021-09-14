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
