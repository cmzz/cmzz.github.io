---
title: Go gRPC 客户端内存泄漏问题排查
tags: 
- gRPC
- golang
- 编程
date: 2022-11-15 08:00:00
---

近期对系统进行压力测试的过程发现随着请求的增加，程序占用内存会持续增长的情况，且增长没有上限，最高占用系统内存超过 90%。

在线系统增加 `pprof` 部署后，开始 debug 与问题排查。

## 从现象开始定位问题

很明确的问题，内存占用过高。因此更直接查看了内存分析。通过分析 `pprof/heap` 文件，得到了如下的调用堆栈，从图中可以看到， newBufWriter + NewReaderSize 共计占用了 2.5GB 内存，显得很不正常。

![](https://tva1.sinaimg.cn/large/008vxvgGgy1h85ywm8sntj30vm0u0tcm.jpg)

系统本身是一个 Web 应用，不过在其请求处理的过程中需要通过 gRPC 调用几个外部的服务，但即使是 500 的并发，占如如此多的内存也不是一个正常现象。

## 从问题开始开析原因

既然已经找到 gRPC 客户端占用了最多的内存的证据，那就开始从 gRPC 调用代码开始分析原因。

```go
conn, err = grpc.Dial( server, grpc.WithTransportCredentials(insecure.NewCredentials()), grpc.WithBlock())
```

这是程序中初始化 gRPC 客户端的代码，简单直接，从 gRPC 官方的文档上 copy 来的。

进入 Dail 内部：

```go
DialContext(context.Background(), target, opts...)
```

请注意，这直接使用了默认的 Background Context. 根据文档的介绍， ctx 参数可以控制连接的取消和**超时**。

如果要使 ctx 的超时生效，必须要同时使用 `grpc.WithBlock()` ，因为 gRPC 默认是使用非阻塞的 http2 客户端。

那 ctx 作用是啥呢？请看官方的说法。ctx 可以用来控制 pending 的超时时间。

```
// In the blocking case, ctx can be used to cancel or expire the pending  
// connection. Once this function returns, the cancellation and expiration of  
// ctx will be noop. Users should call ClientConn.Close to terminate all the// pending operations after this function returns.
```

那是不是并发大太，外部的服务承受不了如大的流量导致了大量 penging 状态的请求没有释放？

于是，我们将代码改成了：

```
ctx, cancel := context.WithTimeout(context.Background(), time.Second)  
defer cancel()
conn, err = grpc.DialContext(ctx, address, grpc.WithTransportCredentials(insecure.NewCredentials()), grpc.WithBlock())
```

增加了 timeout，但将新的代码上线后，内部占用的问题并没有解决！继续看文档，此 timeout 仅作用于链接建立 block 类型的连接建立阶段。

## 网上的答案

网上搜索的答案，几乎千篇一律的说是在 Server 端指定的 KeepAlive 参数，用于在客户端没有心跳时自动的关闭链接。

![](https://tva1.sinaimg.cn/large/008vxvgGgy1h86q1btnr8j316i0u045f.jpg)

服务端不是我能控制的啊，那不管服务端，可以直接在客户端直接应用 KeepAlive 吗？答案是不能。

`keepalive.ClientParameters` 是客户端的 keepalive 参数配饰的 grpc.option，其注释中有明确提示：

> // Make sure these parameters are set in  
   // coordination with the keepalive policy on the server, as incompatible  
   // settings can result in closing of connection.

`ClientParameters` 和`ServerParameters` 需要搭配使用，使用不当会导致链接错误的被关闭。

## ectd 如何使用 gRPC Client

etcd 在 v3 中全面使用了 gRPC，因此想看看在 etcd 中是如何去使用的，这里贴一下 ectd Client 初始化代码：

[etcd/client.go at bf5c936ff1de422b48cc313435aa40ef6f2057ac · etcd-io/etcd · GitHub](https://github.com/etcd-io/etcd/blob/bf5c936ff1de422b48cc313435aa40ef6f2057ac/client/v3/client.go#L289)

## 继续

etcd 在初始化 Connection 时考虑了如 TimeOut、KeepAlive 相关的可选项，可谓是使用的标杆。参照其代对程序 Connection 的建立部分做了一些完善，不过无法仅通过 gRPC Client 的连接配置来解决这个问题，但这个问题又确确实实的发生在 gRPC Client 上，那是不是我们代码对 gRPC 使用不当？

于是，把所有调用 gRPC 的代码都找出来，共有 10 来处，一处一处的排查。

系统因需要链接多个外部的 gRPC Server，应止在程序层面有一些封装用于获取客户端。

```
func GetAaaRPCClient() *grpc.Client
func GetBbbRPCClient() *grpc.Client
```

重新 Review 该部份代码，没有问题！且对客户端做了复用。继续 Review 余下部份，确实找到了 3 处不正确的使用：没用复用上面的 `GetClient`，而是在代码直接初始化客户端，且没有主动关闭。刚好，这三处代码码的 API 也在压测范围之内，那没错，问题就出在这了。

## 总结

gRPC 客户端在其内部做了连接的优化与管理，虽并不需要用户在程序中去管理连接池，但在使用时依然需要注意：

1. 建议只为每个 Service 建立一个客户端
2. 如果需要为每个请求建立连接，那么一定不要忘了关闭
3. Server 端建议配置 KeepAlive 参数，参考 [keepalive package - google.golang.org/grpc/keepalive - Go Packages](https://pkg.go.dev/google.golang.org/grpc/keepalive#ServerParameters) 并在文档中告知调用方
4. 如果 Server 明确说明了 KeepAlive，客户端在建立连接时，建议指定相关 Option，参考 [grpc package - google.golang.org/grpc - Go Packages](https://pkg.go.dev/google.golang.org/grpc#KeepaliveParams)

## 参考

- [Pooling gRPC Connections - My Code Smells!](https://mycodesmells.com/post/pooling-grpc-connections)
- [go - GRPC Connection Management in Golang - Stack Overflow](https://stackoverflow.com/questions/56067076/grpc-connection-management-in-golang)
- [transport.newBufWrite go grpc not releasing memory](https://groups.google.com/g/grpc-io/c/KGlqYrTOjqI)
