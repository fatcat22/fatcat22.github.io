---
layout: post
title:  "以太坊源码解析：rpc"
categories: ethereum
tags: 原创 ethereum rpc 源码解析
excerpt: 以太坊的 rpc 模块为访问以太坊的服务和数据提供了一个接口。这篇文章我们来看看以太坊的 rpc 是如何实现的。
author: fatcat22
---

* content
{:toc}





>本篇文章分析的源码地址为：[https://github.com/ethereum/go-ethereum](https://github.com/ethereum/go-ethereum)  
>分支：[master](https://github.com/ethereum/go-ethereum/tree/master)  
>commit id: [257bfff316e4efb8952fbeb67c91f86af579cb0a](https://github.com/ethereum/go-ethereum/tree/257bfff316e4efb8952fbeb67c91f86af579cb0a)  


# 引言

在几乎所有区块链项目中，都会提供 RPC 功能供其它工具和程序调用。我觉得这是因为我们不仅要求区块链的核心服务可以正常运行，还需要在核心服务运行时需取到一些当前数据、状态，或给核心服务发送数据（比如交易信息）。那么 RPC 就是实现这种需要的最好的方法了。以太坊也不例外，对外提供了 RPC 服务供其它工具调用。这篇文章里，我们就来看一下以太坊的 RPC 服务是如何实现的。


# 什么是 RPC

在介绍以太坊的 rpc 模块时，我想先介绍一下「 RPC 」这个概念。根据[维基百科上的解释](https://zh.wikipedia.org/wiki/%E9%81%A0%E7%A8%8B%E9%81%8E%E7%A8%8B%E8%AA%BF%E7%94%A8)，RPC 的英文全称为「 Remote Procedure Call 」，中文翻译为 「远程过程调用」。它是一种协议，这种协议可以让一台计算机上的程序调用另一台计算机上的程序的接口，而无需额外的为这个交互功能而编程。

简单来说， RPC 主要用来方便地在不同计算机之间调用程序的接口。我们都知道，一般情况下接口的调用都是进程内调用，即提供接口的模块与调用接口的模块需要在同一进程中。RPC 释放了这一要求，使得不同机器、不同进程之间也可以相互调用。

了解了 RPC 的概念，我们接下来就看看以太坊的 rpc 模块是如何实现的。


# 实现架构

rpc 模块中的代码分成两部分：client 和 server。其中 client 实现了与 server 通信的接口，由 `Client` 对象实现，启动代码在 client.go 中； server 实现了服务端的功能，由 `Server` 对象实现，启动代码在 endpoints.go 中。我们分别对其进行说明。


### client

下面是 client 部分的实现结构示意图：
![rpc client 示意图](/pic/ethereum-rpc/rpc-client.png)

从图中可以看到， client 部分的主要对象是 `rpc.Client`。如果要连接某个以太坊的服务，那么你可以调用 `Dial` 函数，在这个函数的内部会根据参数中的路径格式，决定不同的连接方法，并最终返回一个 `rpc.Client` 对象。这个对象导出了 `Call`、`EthSubscribe` 等方法，可以用它们来调用某个 rpc 的 API，或对某些信息进行订阅（后面会对订阅机制进行详细说明）。

在 `Dial` 方法内部，会调用 `Client.dispatch` 创建一个 goroutine。这是 `rpc.Client` 进行数据接收与处理的一个「内部引擎」。

`Dial` 提供了四种连接 server 的方法：HTTP、websocket、stdio、IPC。它们的名字基本都很明显的说明了它们的实现方法，其中 IPC 是通过管道实现的。

可以看到，实现整个 client 功能的思路是比较简单的清淅的：通过 `Dial` 函数连接服务，启动 `Client.dispatch` 「引擎」并返回 `rpc.Client` 对象；在调用  `Call` 等方法时，将参数进行编码发送后，等待一个 channel 事件；如果服务端有回复，那么 `Client.dispatch` 就会对其进行通知。（注意 `Client.dispatch` 起作用的情况是非 HTTP 连接的情况。如果使用 HTTP 方式连接，那么根据 HTTP 连接的特性，一次请求后数据是立刻返回的，不需要 `Client.dispatch` 的功能）

### server

下面是 server 部分的实现结构示意图：
![rpc server 示意图](/pic/ethereum-rpc/rpc-server.png)

从图中也可以看出，以太坊 rpc 的 server 端提供了三种连接方法：HTTP、websocket、IPC，这三种方式的启动函数分别是 `StartHTTPEndpoint`、`StartWSEndpoint`、`StartIPCEndpoint`。这三种方式是可以同时存在的，在 node 模块中，当启动以太坊节点时，我们可以看到三种方式都会被尝试启动：
```go
func (n *Node) startRPC(services map[reflect.Type]Service) error {
    ......
    if err := n.startIPC(apis); err != nil {
        n.stopInProc()
        return err
    }
    if err := n.startHTTP(n.httpEndpoint, apis, n.config.HTTPModules, n.config.HTTPCors, n.config.HTTPVirtualHosts, n.config.HTTPTimeouts); err != nil {
        n.stopIPC()
        n.stopInProc()
        return err
    }
    if err := n.startWS(n.wsEndpoint, apis, n.config.WSModules, n.config.WSOrigins, n.config.WSExposeAll); err != nil {
        n.stopHTTP()
        n.stopIPC()
        n.stopInProc()
        return err
    }
    ......
}
```

任何一种服务方式，都使用 `http.Server` 代表，但它们的 handler 是各不相同的，信息的编码方式也不太相同，但最终它们都由 `rpc.Server.serveRequest` 进行处理。

注意这个图中并没有体现提供 rpc 服务的 API 的注册和调用。一般在提供 rpc 服务时，相关的功能肯定是由其它模块提供接口实现的，服务启动时会将这些接口注册给 rpc 模块。文章后面会专门对 API 的注册和调用进行说明。


### API 的注册与调用

前面我们说过，一般在提供 rpc 服务时，相关功能肯定是由其它模块提供接口并实现，然后在 rpc 服务启动时会将这些接口注册给 rpc 模块。以太坊的 rpc 模块也是这么做的，但不同的是我们看不到通常的一大串等待注册的方法的数组，因为以太坊利用了 go 的反射机制。下面我们对注册和调用分别进行一个详细的说明。


### 注册

API 的注册发生在启动某一连接方式的服务时，比如调用 `StartHTTPEndpoint` 时，第二个参数就是需要注册的 API 信息。这是一个数组，它的每个元素都由一个 `rpc.API` 结构代表，这个结构体的定义为：
```go
type API struct {
    Namespace string      // namespace under which the rpc methods of Service are exposed
    Version   string      // api version for DApp's
    Service   interface{} // receiver instance which holds the methods
    Public    bool        // indication if the methods must be considered safe for public use
}
```

每一个 `rpc.API` 结构体其实代表的是一组 API，这组 API 由 `Service` 字段提供，这个字段保存的是某个真实的对象，对象的导出方法（方法的第一个字母是大写）就是这个对象需要注册到 rpc 模块的 API。另外 `Namespace` 字段定义了 `Service` 字段提供的这组 API 的命名空间。比如在 `Ethash.APIs` 这个方法中，定义了下面一个 `rpc.API` 结构：
```go
[]rpc.API{
    {
        Namespace: "eth",
        Version:   "1.0",
        Service:   &API{ethash},
        Public:    true,
    }
    ......
}
```
其中 `Service` 字段是一个 `ethash.API` 对象，它有一个名为 `GetWork` 的方法，因此使用这个结构体注册 API 以后，rpc 模块就提供了一个名为 `eth_getWork` 的 API，其中「 eth 」是命令空间，「 getWork」 是导出方法的第一个字母变小写，「 _ 」是连接符。

在了解了 `rpc.API` 结构体以后，我们来看看 API 注册到哪了，以及是如何注册的。


### 注册到哪

API 的注册是通过 `Server.RegisterName` 方法实现的，稍微看一下这个方法就可以知道，所有 API 的注册信息都保存到了 `Server.services` 字段中。这个字段的类型为 `serviceRegistry`，它的定义如下：
```go
type serviceRegistry map[string]*service
```
可见它是一个 `map` 类型，map 的 key 为命名空间，也就是 `rpc.API.Namespace` 这个字段；map 的 value 是一个 `service` 类型的结构体，里面的信息对应着 `rpc.API.Service` 字段的相关信息。在上面的例子中，`Server.services` 字段肯定有一项 key 为「 eth 」的 service 对象。

我们继续看一下 `service` 这个结构体的定义如下：
```go
type service struct {
    name          string        // name for service
    typ           reflect.Type  // receiver type
    callbacks     callbacks     // registered handlers
    subscriptions subscriptions // available subscriptions/notifications
}

type callbacks map[string]*callback      // collection of RPC callbacks
type subscriptions map[string]*callback  // collection of subscription callbacks

type callback struct {
    rcvr        reflect.Value  // receiver of method
    method      reflect.Method // callback
    argTypes    []reflect.Type // input argument types
    hasCtx      bool           // method's first argument is a context (not included in argTypes)
    errPos      int            // err return idx, of -1 when method cannot return error
    isSubscribe bool           // indication if the callback is a subscription
}
```
`service` 中的关键字段是 `callbacks` 和 `subscriptions` 字段，这两个字段的内容是从 `rpc.API.Service` 对象中解析出来的、对象的导出方法的相关信息，也就是具体 api 的相关信息。这两个字段的类型都是 `map`，map 的 key 为方法的名字，map 的 value 是一个 `callback` 对象，它存放着方法的具体信息，可以使用这些信息对方法进行调用。还以上面我们说过的例子为例，此时 `service.name` 仍是命名空间的名字「 eth 」，而 `service.callbacks` 中肯定有一项 key 为「 getWork 」的 `callback` 对象。

这里稍微解释一下 subscription 的概念，在本文中我们将其称为「订阅」。这种机制允许客户端订阅某些消息，在有消息时服务端会主动推送这些消息到客户端，而不需要客户端不停的查询。用来进行消息订阅的 API 与普通 API 基本没什么区别，但如果某个方法的第一个参数是 `context` 类型、第一个返回值的类型是 `*Subscription`、第二个返回值类型是 `error`，那么 rpc 模块就会把这个方法当作一个订阅方法，并将这个方法的信息放到 `service.subscriptions` 字段中。（后面我们还会详细说明订阅机制）


### 如何注册

下面我们再来看看注册的实现代码。API 的注册是由 `Server.RegisterName` 实现的，所以我们就从这个方法说起吧。

##### Server.RegisterName

```go
func (s *Server) RegisterName(name string, rcvr interface{}) error {
    if s.services == nil {
        s.services = make(serviceRegistry)
    }

    svc := new(service)
    svc.typ = reflect.TypeOf(rcvr)
    rcvrVal := reflect.ValueOf(rcvr)

    if name == "" {
        return fmt.Errorf("no service name for type %s", svc.typ.String())
    }
    if !isExported(reflect.Indirect(rcvrVal).Type().Name()) {
        return fmt.Errorf("%s is not exported", reflect.Indirect(rcvrVal).Type().Name())
    }

    ......
}
```
这个方法的参数分别是前面我们提到过的 `rpc.API.Namespace` 和 `rpc.API.Service` 字段。方法的一开始是对 `Server.services` 字段的检查。然后生成一个新的 `service` 变量，后续都是对这个变续的填充，并在最后将其放入 `Server.services` 字段。

接下来这段代码通过反射库提供的方法，获取到了参数 `rcvr` 的 Type 值和 Value 值，也就是提供 API 的对象的 Type 和 Value。后面提取相关的 API 名字等信息，都是通过这两个值来提取的。

这段代码里最后通过 `isExported` 方法检查提供 api 的对象自身是否是导出的，这里要求这个对象必须是导出的，否则后面就无法调用这个对象的所有方法。

我们继续看后面的代码：
```go
func (s *Server) RegisterName(name string, rcvr interface{}) error {
    ......

    methods, subscriptions := suitableCallbacks(rcvrVal, svc.typ)

    if len(methods) == 0 && len(subscriptions) == 0 {
        return fmt.Errorf("Service %T doesn't have any suitable methods/subscriptions to expose", rcvr)
    }

    ......
}
```
这段代码的重点是 `suitableCallbacks` 函数，这个函数用来将参数 `rcvr` 的所有导出方法的相关信息提取出来，并将它们返回。这些方法分两类：普通方法和订阅方法，分别返回在 `methods` 和 `subscriptions` 中。

我们暂时不看 `suitableCallbacks` 是如何提取这些方法数据的，继续往下看 `Server.RegisterName` 方法：
```go
func (s *Server) RegisterName(name string, rcvr interface{}) error {
    ......

    if regsvc, present := s.services[name]; present {
        for _, m := range methods {
            regsvc.callbacks[formatName(m.method.Name)] = m
        }
        for _, s := range subscriptions {
            regsvc.subscriptions[formatName(s.method.Name)] = s
        }
        return nil
    }

    svc.name = name
    svc.callbacks, svc.subscriptions = methods, subscriptions

    s.services[svc.name] = svc
    return nil
}
```
这是 `Server.RegisterName` 的最后一段代码，这里将刚才提取到的 api 信息 `mehtods` 和 `subscriptions` 加入到 `svc` 变量的 `svc.callbacks` 和 `svc.subscriptions` 字段中，并最终将 `svc` 加入到 `Server.services` 字段中。

这里注意会判断一下 `name` 所代表的命名空间是否在 `Server.services` 中存在，因为不同对象可能使用同一命名空间的进行注册，所以这里要区别处理。

到这里我们就分析完了 `Server.RegisterName` 这个方法，无非就是解析对象的方法信息，并将这些信息存储到 `Server.services` 中。那么我们现在看看是如何解析对象的方法信息的，这个功能在 `suitableCallbacks` 中。

##### suitableCallbacks

```go
func suitableCallbacks(rcvr reflect.Value, typ reflect.Type) (callbacks, subscriptions) {
    callbacks := make(callbacks)
    subscriptions := make(subscriptions)

METHODS:
    for m := 0; m < typ.NumMethod(); m++ {
        ......
    }

    return callbacks, subscriptions
```
方法一开始就定义了两个要返回的变量：`callbacks` 和 `subscriptions`，然后通过一个 for 循环，对每一个方法进行处理，从而填充 `callbacks` 和 `subscriptions`。注意这里 `typ` 参数就是通过 reflect 库获取到的 `rpc.API.Service` 的 Type 值，rcvr 则是 `rpc.API.Service` 的 Value 值。

我们下面一点点详细看一下 for 循环中是如何填充 `callbacks` 和 `subscriptions` 这两个变量的：
```go
func suitableCallbacks(rcvr reflect.Value, typ reflect.Type) (callbacks, subscriptions) {
    ......

METHODS:
    for m := 0; m < typ.NumMethod(); m++ {
        method := typ.Method(m)
        mtype := method.Type
        mname := formatName(method.Name)
        if method.PkgPath != "" { // method must be exported
            continue
        }

        ......
    }

    return callbacks, subscriptions
}
```
这里第一步是获取当前方法的基本信息，包括代表方法本身的 `method` 变量，它的类型 `mtype`，以及它的名字 `mname`。 `formatName` 实际上将第一个字符变为小写字符，所以在前面的例子中，有一个名字为「GetWork」的方法，但 rpc 的 API 名字为 「eth_getWork」。

这段代码还会判断方法是否是导出的，只有导出的方法才能成为 rpc 的 API。

我们继续看后面的处理代码：
```go
func suitableCallbacks(rcvr reflect.Value, typ reflect.Type) (callbacks, subscriptions) {
    ......

METHODS:
    for m := 0; m < typ.NumMethod(); m++ {
        ......

        var h callback
        h.isSubscribe = isPubSub(mtype)
        h.rcvr = rcvr
        h.method = method
        h.errPos = -1

        firstArg := 1
        numIn := mtype.NumIn()
        if numIn >= 2 && mtype.In(1) == contextType {
            h.hasCtx = true
            firstArg = 2
        }

        if h.isSubscribe {
            h.argTypes = make([]reflect.Type, numIn-firstArg) // skip rcvr type
            for i := firstArg; i < numIn; i++ {
                argType := mtype.In(i)
                if isExportedOrBuiltinType(argType) {
                    h.argTypes[i-firstArg] = argType
                } else {
                    continue METHODS
                }
            }

            subscriptions[mname] = &h
            continue METHODS
        }

        ......
    }

    return callbacks, subscriptions
}
```
这一段代码开始定义了一个 `callback` 类型的变量 `h`，其实后面所有的内容都是填充这个 `h` 变量，然后将其放入到 `callback` 或 `subscriptions` 中。

`firstArg` 定义了第一个非 `context.Context` 类型的参数的索引，因为后面会填充 `callback.argTypes` 字段，而第一个参数如果是 `context.Context` 类型，则不会记录在这个字段里，而是记录在 `callback.hasCtx` 字段里。

这段代码还使用 `isPubSub` 函数来判断注册的是否是一个「订阅」方法，这个函数的判断方法很简单，这里就不上它的代码，直接给出它的判断逻辑。如果一个方法是「订阅」方法，它必须满足：：
1. 至少有一个参数，且第一个参数的类型是 `context.Context`
2. 有两个返回值，第一个返回值的类型必须为 `*Subscription`，第二个返回值的类型必须实现了接口 `error`

比如这两个方法： `PrivateDebugAPI.TraceChain` 和 `PublicFilterAPI.NewHeads`，它们的声明分别如下：
```go
func (api *PrivateDebugAPI) TraceChain(ctx context.Context, start, end rpc.BlockNumber, config *TraceConfig) (*rpc.Subscription, error)

func (api *PublicFilterAPI) NewHeads(ctx context.Context) (*rpc.Subscription, error)
```
可以看到这两个方法都满足 `isPubSub` 函数的要求，所以这俩个是「订阅」方法。

如果要注册的方法是一个「订阅」方法，那么我们就将第一个参数之后的所有参数信息，保存到 `callback.argTypes` 中（第一个参数肯定是 `context.Context` 类型，且已被标记在 `callback.hasCtx` 中），然后将 `h` 变量加入到 `subscriptions` 中，并处理下一个方法。

下面我们再来看看如果注册的不是「订阅」方法，而是一个普通方法，是如何处理的：
```go
func suitableCallbacks(rcvr reflect.Value, typ reflect.Type) (callbacks, subscriptions) {
    ......

METHODS:
    for m := 0; m < typ.NumMethod(); m++ {
        ......

        h.argTypes = make([]reflect.Type, numIn-firstArg)
        for i := firstArg; i < numIn; i++ {
            argType := mtype.In(i)
            if !isExportedOrBuiltinType(argType) {
                continue METHODS
            }
            h.argTypes[i-firstArg] = argType
        }

        // check that all returned values are exported or builtin types
        for i := 0; i < mtype.NumOut(); i++ {
            if !isExportedOrBuiltinType(mtype.Out(i)) {
                continue METHODS
            }
        }

        // when a method returns an error it must be the last returned value
        h.errPos = -1
        for i := 0; i < mtype.NumOut(); i++ {
            if isErrorType(mtype.Out(i)) {
                h.errPos = i
                break
            }
        }

        if h.errPos >= 0 && h.errPos != mtype.NumOut()-1 {
            continue METHODS
        }

        switch mtype.NumOut() {
        case 0, 1, 2:
            if mtype.NumOut() == 2 && h.errPos == -1 { // method must one return value and 1 error
            continue METHODS
        }
        callbacks[mname] = &h
        }
    }
    return callbacks, subscriptions
}
```

这段代码虽然比较长，但逻辑很简单。一开始也是保存方法的参数信息到 `callback.argTypes` 字段中。剩下的代码其实几乎都是在检查返回值信息（「订阅」方法的返回值数量和类型是固定的，在 `isPubSub` 中就算是检查过了）。总得来说，对返回值有如下要求：
1. 所有返回值类型必须是已导出的或内置类型
2. 返回值数量必须小于等于 2 个
3. 如果有两个返回值，则第二个必须是 `error` 类型

（你可能会觉得上面第 3 条没有与之对应的代码片段。其实这一条是结合 `if h.errPos >= 0 && h.errPos != mtype.NumOut()-1` 与 `if mtype.NumOut() == 2 && h.errPos == -1` 得出的，即「如果存在 `error` 类型的返回值，那么它必须是最后一个」 与 「如果存在 2 个返回值，那么必须有一个 `error` 类型的返回值」）

到此，`suitableCallbacks` 函数就完成了。这个函数就是遍历提供服务的对象的所有导出方法，并将它们记录到 `callbacks` 和 `subscriptions` 中返回。回想整个注册的过程，其实也就是解析提供服务的对象的导出方法，然后保存这些信息的过程。


### 调用

在前面的 server 结构图中，我们可以看到服务端所有对消息的处理，最后都汇集到了 `Server.serveRequest` 这个方法中。对请求的解析、和解析后调用相应的注册过的 API，也是从这个方法出发完成的。因此我们从这个方法入手，看看请求是如何被处理的。

##### Server.serveRequest

我们先看一下这个方法的关键代码组成的一个框架，如下：
```go
func (s *Server) serveRequest(ctx context.Context, codec ServerCodec, singleShot bool, options CodecOption) error {
    ......
    for atomic.LoadInt32(&s.run) == 1 {
        reqs, batch, err := s.readRequest(codec)

        ......

        if singleShot {
            if batch {
                s.execBatch(ctx, codec, reqs)
            } else {
                s.exec(ctx, codec, reqs[0])
            }
            return nil
        }

        // For multi-shot connections, start a goroutine to serve and loop back
        pend.Add(1)
        go func(reqs []*serverRequest, batch bool) {
            defer pend.Done()
            if batch {
                s.execBatch(ctx, codec, reqs)
            } else {
                s.exec(ctx, codec, reqs[0])
            }
        }(reqs, batch)
    }
}
```
这里首先调用 `Server.readRequest` 处理发送过来的请求数据，将这些请求数据组织到 `reqs` 变量里。然后调用 `Server.exec` 或 `Server.execBatch` 执行请求。这里根据 `singleShot` 值的不同来决定当前请求是在新的 goroutine 中执行，还是直接执行请求并返回。当连接方式是 HTTP 时，`singleShot` 的值为 true，因为对于 http 连接，一次连接就是一次请求，请求执行完就直接返回结果，不需要使用 gotoutine。

从上面这段代码中可以看到，关键方法是 `Server.readRequest` 和 `Server.exec`，所以下面我们就来详细看看这两个方法。（`Server.execBatch` 用来执行一次请求中包含多个 API 调用的情况，我们这里暂不关心这种情况，其实处理方式都是一样的，无非就是多个循环而已）

我们先来看看 `Server.readRequest` 方法。

##### Server.readRequest

```go
func (s *Server) readRequest(codec ServerCodec) ([]*serverRequest, bool, Error) {
    reqs, batch, err := codec.ReadRequestHeaders()

    requests := make([]*serverRequest, len(reqs))

    for i, r := range reqs {
        ......
    }

    return requests, batch, nil
}
```
这里首先调用 `Servercodec.ReadRequestHeaders` 读取请求数据，然后进一步解析这些数据，并填充到 `requests` 变量中返回。 `requests` 变量的类型 `serverRequest` 的定义如下：
```go
type serverRequest struct {
    id            interface{}
    svcname       string
    callb         *callback
    args          []reflect.Value
    isUnsubscribe bool
    err           Error
}
```
注意这个结构体中的两个字段 `callb` 和 `args`，它们分别代表了将要调用的 API 的信息和参数信息。也就是说调用 `Servercodec.ReadRequestHeaders` 后，就已经将以后需要调用的方法信息获取到了。

下面我们看看 for 循环是如何进一步解析 `reqs` 的：
```go
func (s *Server) readRequest(codec ServerCodec) ([]*serverRequest, bool, Error) {
    ......

    for i, r := range reqs {
        ......

        if r.isPubSub && strings.HasSuffix(r.method, unsubscribeMethodSuffix) {
            requests[i] = &serverRequest{id: r.id, isUnsubscribe: true}
            argTypes := []reflect.Type{reflect.TypeOf("")} // expect subscription id as first arg
            if args, err := codec.ParseRequestArguments(argTypes, r.params); err == nil {
                requests[i].args = args
            } else {
                requests[i].err = &invalidParamsError{err.Error()}
            }
            continue
        }
        ......
    }
}
```
首先处理的是「退订」的请求，如果 `r.isPubSub` 的值为 true（在 `ServerCodec.ReadRequestHeaders` 中被设置，后面会讲到），且 API 的名字后缀为「 _unsubscribe 」，就认为这是一个「退订」的请求。这种情况的 `serverRequest` 变量很简单，就是设置 `id` 和 `isUnsubscribe` 字段就可以了。然后调用 `ServerCodec.ParseRequestArguments` 从请求数据中解析参数信息，存储到 `serverRequest.args` 字段中。

如果不是「退订」请求，则继续执行下面的代码：
```go
func (s *Server) readRequest(codec ServerCodec) ([]*serverRequest, bool, Error) {
    ......

    for i, r := range reqs {
        ......

        if r.isPubSub { // eth_subscribe, r.method contains the subscription method name
            if callb, ok := svc.subscriptions[r.method]; ok {
                requests[i] = &serverRequest{id: r.id, svcname: svc.name, callb: callb}
                if r.params != nil && len(callb.argTypes) > 0 {
                    argTypes := []reflect.Type{reflect.TypeOf("")}
                    argTypes = append(argTypes, callb.argTypes...)
                    if args, err := codec.ParseRequestArguments(argTypes, r.params); err == nil {
                        requests[i].args = args[1:] // first one is service.method name which isn't an actual argument
                    } else {
                        requests[i].err = &invalidParamsError{err.Error()}
                    }
                }
            } else {
                requests[i] = &serverRequest{id: r.id, err: &methodNotFoundError{r.service, r.method}}
            }
            continue
        }
        ......
    }
}
```
这里判断是否是「订阅」请求。如果是，则通过请求的 API 的名字，从 `svc.subscriptions` 中拿到对应的 API 信息，并存储在 `callb` 变量中。这个变量接着被转存到了 `serverRequest.callb` 字段中。与刚才类似，请求中的参数信息也是存储到了 `serverRequest.args` 中。

如果判断不是「订阅」信息，那么应该是一个普通的 API 请求了。我们继续看下面的处理代码：
```go
func (s *Server) readRequest(codec ServerCodec) ([]*serverRequest, bool, Error) {
    ......

    for i, r := range reqs {
        ......

        if callb, ok := svc.callbacks[r.method]; ok { // lookup RPC method
            requests[i] = &serverRequest{id: r.id, svcname: svc.name, callb: callb}
            if r.params != nil && len(callb.argTypes) > 0 {
                if args, err := codec.ParseRequestArguments(callb.argTypes, r.params); err == nil {
                    requests[i].args = args
                } else {
                    requests[i].err = &invalidParamsError{err.Error()}
                }
            }
            continue
        }

        requests[i] = &serverRequest{id: r.id, err: &methodNotFoundError{r.service, r.method}}
    }

    return requests, batch, nil
}
```
这段代码的逻辑与前面类似，先是从 `svc.callbacks` 中取出 API 的信息到 `callb` 变量中，与「订阅」消息区别的是这是一个普通 API 请求，因此 API 信息是从 `svc.callbacks` 中获取而非 `svc.subscriptions` 中。请求的参数信息也是存储在 `serverRequest.args` 中。

可以看到，整个 `Server.readRequest` 虽然很长，但逻辑却很简单，就是通过请求数据，获取 API 的信息并存放到 `serverRequest.callb` 字段中；从请求数据中解析参数信息，并存储到 `serverRequest.args` 中。


##### ServerCodec.ReadRequestHeaders

我们再来看看 `ServerCodec.ReadRequestHeaders` 是如何解析请求数据的。`ServerCodec` 是一个接口类型，实现它的结构体是 `jsonCodec`：
```go
func (c *jsonCodec) ReadRequestHeaders() ([]rpcRequest, bool, Error) {
    ......

    var incomingMsg json.RawMessage
    if err := c.decode(&incomingMsg); err != nil {
        return nil, false, &invalidRequestError{err.Error()}
    }
    if isBatch(incomingMsg) {
        return parseBatchRequest(incomingMsg)
    }
    return parseRequest(incomingMsg)
}
```
这个方法很简短，就是解析出 `json.RawMessage` 并调用 `parseRequest` 解析。我们重点看一下 `parseRequest` 函数。

##### parseRequest

```go
func parseRequest(incomingMsg json.RawMessage) ([]rpcRequest, bool, Error) {
    var in jsonRequest
    if err := json.Unmarshal(incomingMsg, &in); err != nil {
        return nil, false, &invalidMessageError{err.Error()}
    }

    if err := checkReqId(in.Id); err != nil {
        return nil, false, &invalidMessageError{err.Error()}
    }

    ......
}
```
这个方法一开始主要是把 json 数据解析成 `jsonRequest` 结构。我们看后面的实现：
```go
func parseRequest(incomingMsg json.RawMessage) ([]rpcRequest, bool, Error) {
    ......

    if strings.HasSuffix(in.Method, subscribeMethodSuffix) {
        reqs := []rpcRequest{{id: &in.Id, isPubSub: true}}
        if len(in.Payload) > 0 {
            // first param must be subscription name
            var subscribeMethod [1]string
            if err := json.Unmarshal(in.Payload, &subscribeMethod); err != nil {
                log.Debug(fmt.Sprintf("Unable to parse subscription method: %v\n", err))
                return nil, false, &invalidRequestError{"Unable to parse subscription request"}
            }

            reqs[0].service, reqs[0].method = strings.TrimSuffix(in.Method, subscribeMethodSuffix), subscribeMethod[0]
            reqs[0].params = in.Payload
            return reqs, false, nil
        }
        return nil, false, &invalidRequestError{"Unable to parse subscription request"}
    }
    ......
}
```
这里首先判断请求的 API 名字是否有「 _subscribe 」后缀，如果是，则代表这是一个「订阅」请求（所有的「订阅」请求方法名都为「 eth_subscribe 」，第一个参数为要订阅的 API 名字，详细信息可参考[这篇文章](https://github.com/ethereum/go-ethereum/wiki/RPC-PUB-SUB)）。因此这里设置 `rpcRequest.isPubSub` 为 true，并解析第一个参数，作为要「订阅」的 API 的名字。

如果不是「订阅」请求，则继续向下执行：
```go
func parseRequest(incomingMsg json.RawMessage) ([]rpcRequest, bool, Error) {
    ......

    if strings.HasSuffix(in.Method, unsubscribeMethodSuffix) {
        return []rpcRequest{{id: &in.Id, isPubSub: true,
            method: in.Method, params: in.Payload}}, false, nil
    }
    ......
}
```
这段代码判断请求的 API 名字是否有「 _unsubscribe 」后缀，如果是，则代表这是一个「退订」请求。（所有的「退订」请求方法名都为「 eth_unsubscribe 」，唯一的一个参数是 ID 信息，详细信息可参考[这篇文章](https://github.com/ethereum/go-ethereum/wiki/RPC-PUB-SUB)）。这里同样将相关信息存储到 `rpcRequest` 中，并返回。

如果也不是「退订」请求，那么就是一个普通的 rpc 请求了，我们看后面的代码是如何处理的：
```go
func parseRequest(incomingMsg json.RawMessage) ([]rpcRequest, bool, Error) {
    ......

    elems := strings.Split(in.Method, serviceMethodSeparator)
    if len(elems) != 2 {
        return nil, false, &methodNotFoundError{in.Method, ""}
    }

    // regular RPC call
    if len(in.Payload) == 0 {
        return []rpcRequest{{service: elems[0], method: elems[1], id: &in.Id}}, false, nil
    }

    return []rpcRequest{{service: elems[0], method: elems[1], id: &in.Id, params: in.Payload}}, false, nil
}
```
这段代码中首先使用分隔符「 _ 」将请求的方法名切分，得到的第一个元素是命名空间的名字，第二个元素是请求的 API 的名字。这些信息同样也会用来构造一个 `rpcRequest` 对象并返回。

可以看到，整个 `parseRequet` 函数就是分别判断是否是「订阅」、「退订」和普通请求，然后依据这三种不同的请求类型、按不同的格式对请求数据进行处理，并将这些数据填充到 `rpcRequest` 对象返回。

这里需要注意一点的是，对于「订阅」和「退订」请求，它们的名字分别为「 eth_subscribe 」和「 eth_unsubscribe 」。然而这两个方法根据不是通过 API 注册提供的服务，而是解析请求的方法名时，对这两个名字进行了特殊处理。（我自己在查看源代码的时候，花了一些时间去查找这两个方法是在哪里注册的，但一直没找到。后来经过调试才发现，原来这两名字只是在解析请求数据的时候进行了特殊对待而已）


##### Server.exec

前面我们一直在说如何解析请求数据的，现在我们看看数据解析后，是如何调用对应的 API 提供服务的。调用对应的 API 是从 `Server.exec` 开始的：
```go
func (s *Server) exec(ctx context.Context, codec ServerCodec, req *serverRequest) {
    var response interface{}
    var callback func()
    if req.err != nil {
        response = codec.CreateErrorResponse(&req.id, req.err)
    } else {
        response, callback = s.handle(ctx, codec, req)
    }

    if err := codec.Write(response); err != nil {
        log.Error(fmt.Sprintf("%v\n", err))
        codec.Close()
    }

    // when request was a subscribe request this allows these subscriptions to be actived
    if callback != nil {
        callback()
    }
}
```
这个方法很简单，主要功能在被调用的 `Server.handle` 这个方法上，它会调用相应的 API，并将结果返回到 `response` 变量中。我们下面看看 `Server.handle` 是怎么实现的。


##### Server.handle

```go
func (s *Server) handle(ctx context.Context, codec ServerCodec, req *serverRequest) (interface{}, func()) {
    ......

    if req.isUnsubscribe { // cancel subscription, first param must be the subscription id
        if len(req.args) >= 1 && req.args[0].Kind() == reflect.String {
            notifier, supported := NotifierFromContext(ctx)
            if !supported { // interface doesn't support subscriptions (e.g. http)
                return codec.CreateErrorResponse(&req.id, 
                    &callbackError{ErrNotificationsUnsupported.Error()}), nil
            }

            subid := ID(req.args[0].String())
            if err := notifier.unsubscribe(subid); err != nil {
                return codec.CreateErrorResponse(&req.id, &callbackError{err.Error()}), nil
            }

            return codec.CreateResponse(req.id, true), nil
        }
        return codec.CreateErrorResponse(&req.id, 
            &invalidParamsError{"Expected subscription id as first argument"}), nil
    }
}
```
与前面介绍的解析请求数据的方法类似，`Server.handle` 这个方法也是先对「退订」这种情况进行处理。如果是「退订」请求，它首先从 `ctx` 取得类型为 `Notifier` 的变量。然后将参数列表中的第一个参数作为 ID，并调用 `Notifier.unsubscribe` 退订指定的消息。（「退订」消息只有一个参数，就是要退订的 ID，详细信息可以参考[这篇文章](https://github.com/ethereum/go-ethereum/wiki/RPC-PUB-SUB)）

我自己一开始不明白 `NotifierFromContext` 是如何从 `ctx` 中取得 `Notifier` 的，因为 `ctx` 的 `Notifier` 是在 websocket 的 handler 的一开始被新创建的：
```go
func (s *Server) serveRequest(ctx context.Context, codec ServerCodec, singleShot bool, options CodecOption) error {
    ......

    if options&OptionSubscriptions == OptionSubscriptions {
        ctx = context.WithValue(ctx, notifierKey{}, newNotifier(codec))
    }
    ......
}
```
既然是一个新创建的 notifier，为什么退订的时候可以取得一个有有效 ID 的 notifier 呢？

后来发现是我自己没有理解 websocket 的 handler 的调用方式，以为和 HTTP 一样，每次有请求时都调用一次。后来经过调试才发现， websocket 的 handler 在某个连接连接成功以后被调用，其后一直运行，直到连接断开。因此刚才调用 `newNotifier` 的代码类似于一个初始化的代码，只要连接不断开，使用此连接订阅的消息都会记录在这个 notifier 中。换句话说，「订阅」和「退订」请求中的 ID，只有在同一连接中有效。

关于订阅和退订的具体实现机机制，请参见后面的「订阅与推送」小节。

我们继续对 `Server.handle` 的分析。如果不是「退订」消息，就继续往后面的代码执行：
```go
func (s *Server) handle(ctx context.Context, codec ServerCodec, req *serverRequest) (interface{}, func()) {
    ......

    if req.callb.isSubscribe {
        subid, err := s.createSubscription(ctx, codec, req)
        if err != nil {
            return codec.CreateErrorResponse(&req.id, &callbackError{err.Error()}), nil
        }

        // active the subscription after the sub id was successfully sent to the client
        activateSub := func() {
            notifier, _ := NotifierFromContext(ctx)
            notifier.activate(subid, req.svcname)
        }

        return codec.CreateResponse(req.id, subid), activateSub
    }

    ......
}
```
这段是处理「订阅」消息的代码。如果是「订阅」请求，那么调用 `Server.createSubscription` 响应这一请求，并返回本次订阅的 ID。我们马上看一下 `Server.createSubscription` 是如何实现的：
```go
func (s *Server) createSubscription(ctx context.Context, c ServerCodec, req *serverRequest) (ID, error) {
    // subscription have as first argument the context following optional arguments
    args := []reflect.Value{req.callb.rcvr, reflect.ValueOf(ctx)}
    args = append(args, req.args...)
    reply := req.callb.method.Func.Call(args)

    if !reply[1].IsNil() { // subscription creation failed
        return "", reply[1].Interface().(error)
    }

    return reply[0].Interface().(*Subscription).ID, nil
}
````
这个方法很简单，就是调用 `req` 参数中的 `callb` 字段记录的方法。前面我们已经说过，「订阅」 API 有固定的格式，第一个参数的类型必须是 `context.Context` 类型。所以这个方法里第一行将当前的 `ctx` 变量作为参数加入了进来。

我们继续看如果不是「订阅」请求， `Server.handle` 的下一步处理：
```go
func (s *Server) handle(ctx context.Context, codec ServerCodec, req *serverRequest) (interface{}, func()) {
    ......

    // regular RPC call, prepare arguments
    if len(req.args) != len(req.callb.argTypes) {
        rpcErr := &invalidParamsError{fmt.Sprintf("%s%s%s expects %d parameters, got %d",
            req.svcname, serviceMethodSeparator, req.callb.method.Name,
            len(req.callb.argTypes), len(req.args))}
        return codec.CreateErrorResponse(&req.id, rpcErr), nil
    }

    arguments := []reflect.Value{req.callb.rcvr}
    if req.callb.hasCtx {
        arguments = append(arguments, reflect.ValueOf(ctx))
    }
    if len(req.args) > 0 {
        arguments = append(arguments, req.args...)
    }

    // execute RPC method and return result
    reply := req.callb.method.Func.Call(arguments)
    if len(reply) == 0 {
        return codec.CreateResponse(req.id, nil), nil
    }
    if req.callb.errPos >= 0 { // test if method returned an error
        if !reply[req.callb.errPos].IsNil() {
            e := reply[req.callb.errPos].Interface().(error)
            res := codec.CreateErrorResponse(&req.id, &callbackError{e.Error()})
            return res, nil
        }
    }
    return codec.CreateResponse(req.id, reply[0].Interface()), nil
}
```
这段代码处理普通的 API 请求。这里首先检查请求数据中的参数个数与注册时定义的参数个数是否一致，如果不一致肯定是无法调用的。然后就是构造参数列表，并通过 `req.callb.method.Func.Call` 调用。这与「订阅」请法时的调用方式一样，用的都是 go 语言反射库的特性。

到此，整个请求的调用过程我们就分析完了。代码在一开始解析请求数据的阶段就已经准备好了当前是哪种类型的调用，以及该调用哪个方法（`req.callb`）；然后根据不同类型的请求，使用 go 的反射库特性，对方法进行调用。
 

# 订阅与推送

以太坊的 rpc 模块提供了消息订阅功能，当客户端订阅了某一消息之后，如果服务端产生了这个消息的数据，就会主动将数据推送给客户端，而不需要客户端不断的查询新数据。这样可以有效的提高效率。但只有使用 websocket 或 ipc 的方式连接服务端时，才可以使用这个功能；使用 HTTP 连接时是无法使用的。在这一小节里我们看看 rpc 模块是如何实现这一功能的。

在[这篇文章](https://github.com/ethereum/go-ethereum/wiki/RPC-PUB-SUB)里详细介绍了如何使用订阅与推送的机制。如果要订阅某一事件消息，那么你可以发送一个方法名为「 eth_subscribe 」的请求，它的第一个参数是要订阅的 API 的名字，其后的参数是提供给这个 API 的参数。例如你要订阅 「newHeads」这个消息，你可以发送如下请求数据：
> {"id": 1, "method": "eth_subscribe", "params": ["newHeads"]}

如果订阅成功，服务端会返回此次订阅的 ID，以后每次推送消息，或取消订阅时，都会使用这个 ID。例如：
> {"jsonrpc":"2.0","id":1,"result":"0xcd0c3e8af590364c09d0fa6a1210faf5"}

随后当有数据产生时，就会将数据推送到客户端，例如：
> {"jsonrpc":"2.0","method":"eth_subscription","params":{"subscription":"0xcd0c3e8af590364c09d0fa6a1210faf5","result":{"difficulty":"0xd9263f42a87",<...>, "uncles":[]}}}

可以看到，第一个参数 `subscription` 就是此次订阅的 ID。

当你想退订某个消息时，只需要使用 `eth_unsubscribe` 方法，加上消息的 ID 就可以了，比如：
> {"id": 1, "method": "eth_unsubscribe", "params": ["0xcd0c3e8af590364c09d0fa6a1210faf5"]}

大体了解了这整个过程以后，我们就从代码的层面来看一下这这个过程是如何实现的。

前面我们已经说过，只有使用 websocket 或 ipc 的方式连接服务端时，才可以使用订阅的功能。因为这两种连接机制是持久连接且双向通信，即连接建立以后可以一直保持，并可以主动发数据给客户端。这两种连接的消息处理函数也是在连接建立以后一直运行，直到连接中断后才会退出。所以在这里我们首先要注意的是在消息处理函数运行之初，建立的一个 notifier 对象：
```go
func (s *Server) serveRequest(ctx context.Context, codec ServerCodec, singleShot bool, options CodecOption) error {
    ......
    if options&OptionSubscriptions == OptionSubscriptions {
        ctx = context.WithValue(ctx, notifierKey{}, newNotifier(codec))
    }
    ......
}
```
websocket 和 ipc 方式的连接，参数 `options` 都会带有 `OptionSubscriptions` 标志位，因此也一定会调用 `newNotifier` 在 `ctx` 中创建一个 `Notifier` 类型的变量。这个变量是消息订阅的关键，也是中枢。所有本次连接的消息订阅事件都是记录在这个对象中的。

前面我们提到过，在调用 `Server.RegisterName` 注册 API 时，一个提供订阅功能的 API 必须满足以下条件：
1. 至少有一个参数，且第一个参数的类型是 `context.Context`
2. 有两个返回值，第一个返回值的类型必须为 `*Subscription`，第二个返回值的类型必须实现了接口 `error`

我搜索了一下代码，发现目前代码中的订阅 API 有这些：
```go
func (api *PrivateDebugAPI) TraceChain(ctx context.Context, 
    start, end rpc.BlockNumber, config *TraceConfig) (*rpc.Subscription, error)
func (api *PublicDownloaderAPI) Syncing(ctx context.Context) (*rpc.Subscription, error)
func (api *PublicFilterAPI) NewPendingTransactions(ctx context.Context) (*rpc.Subscription, error)
func (api *PublicFilterAPI) NewHeads(ctx context.Context) (*rpc.Subscription, error)
func (api *PublicFilterAPI) Logs(ctx context.Context, crit FilterCriteria) (*rpc.Subscription, error)
func (api *PrivateAdminAPI) PeerEvents(ctx context.Context) (*rpc.Subscription, error)
```

它们在注册时，全部注册到了 `Server.services` 中每个 Value 类型的 `service.subscriptions` 字段中。

我们接下来看看当收到一个「订阅」消息时，代码是如何进行处理的。其实在读取和解析请求数据的方法 `Server.readRequest` 中，就已经对订阅消息进行了处理。比如在 `parseRequest` 中，会特殊处理方法名为 `eth_subscribe` 的请求：
```go
func parseRequest(incomingMsg json.RawMessage) ([]rpcRequest, bool, Error) {
    ......

    if strings.HasSuffix(in.Method, subscribeMethodSuffix) {
        reqs := []rpcRequest{{id: &in.Id, isPubSub: true}}
        ......
    }
    ......
}
```
在 `parseRequest` 中如果发现请求的方法名的后缀为 `subscribeMethodSuffix` 定义的值（即 「 _subscribe 」），则将 `rpcRequest.isPubSub` 字段设置为 true。

而在 `Server.readRequest` 中，当 `rpcRequest.isPubSub` 的值为 true 时，代表这是订阅或退订请求，因此也会对其进行特殊处理：
```go
func (s *Server) readRequest(codec ServerCodec) ([]*serverRequest, bool, Error) {
    reqs, batch, err := codec.ReadRequestHeaders()
    ......

    for i, r := range reqs {
        ......

        if r.isPubSub && strings.HasSuffix(r.method, unsubscribeMethodSuffix) {
            // 退订请求
            requests[i] = &serverRequest{id: r.id, isUnsubscribe: true}
            argTypes := []reflect.Type{reflect.TypeOf("")} // expect subscription id as first arg
            if args, err := codec.ParseRequestArguments(argTypes, r.params); err == nil {
                requests[i].args = args
            } else {
                requests[i].err = &invalidParamsError{err.Error()}
            }
            continue
        }

        if r.isPubSub { 
            // eth_subscribe, 订阅请求，退订请求前面已经处理过了
            if callb, ok := svc.subscriptions[r.method]; ok {
                requests[i] = &serverRequest{id: r.id, svcname: svc.name, callb: callb}
                ......
            }
        }
    }
}
```
这里首先处理的是退订请求，重点是保存参数，因为要退订的消息的 ID 在参数中；其次处理订阅请求，可以看到这里是从 `service.subscriptions` 中获取到 `callback` 类型的值，并将其存储到 `serverRequest.callb` 字段中。这个值代表了提供服务的 API，后面将会对它进行调用。

对请求数据解析完成以后，就要对相应的提供服务的 API 进行调用了。下面我们看看是如何调用，以及如何完成订阅功能的。

对于 API 的调用，我们在「Server.handle」小节里已经讲过了，这里再简单说一下。我们先看订阅消息的调用，代码如下：
```go
func (s *Server) handle(ctx context.Context, codec ServerCodec, req *serverRequest) (interface{}, func()) {
    ......

    if req.callb.isSubscribe {
        subid, err := s.createSubscription(ctx, codec, req)
        ......
        activateSub := func() {
            notifier, _ := NotifierFromContext(ctx)
            notifier.activate(subid, req.svcname)
        }

        return codec.CreateResponse(req.id, subid), activateSub
    }
}
```
这里有两个重点，一是调用 `Server.createSubscription` 创建一个订阅对象；二是返回 `activateSub` 函数。我们一个一个看，先看 `Server.createSubscription` 是如何完成订阅的，它的代码如下；
```go
func (s *Server) createSubscription(ctx context.Context, c ServerCodec, req *serverRequest) (ID, error) {
    args := []reflect.Value{req.callb.rcvr, reflect.ValueOf(ctx)}
    args = append(args, req.args...)
    reply := req.callb.method.Func.Call(args)

    if !reply[1].IsNil() { // subscription creation failed
        return "", reply[1].Interface().(error)
    }

    return reply[0].Interface().(*Subscription).ID, nil
}
```
代码很简单，首先构造参数，将 `ctx` 加到参数列表中，然后通过 `req.callb.method.Func.Call(args)` 调用对应的提供订阅服务的 API，这个 API 的第一个返回值是一个 `*Subscription` 类型，因此这里将其 `Subscription.ID` 返回，这是本次订阅的 ID。

其实从这里好像还看不太出是如何完成消息订阅的，因为具体的订阅功能需要提供服务的 API 来实现，rpc 模块中的相关代码只是一个管理的功能。所以我们以 `PublicFilterAPI.NewHeads` 为例，看看实现订阅功能的 API 是如何实现的：
```go
func (api *PublicFilterAPI) NewHeads(ctx context.Context) (*rpc.Subscription, error) {
    notifier, supported := rpc.NotifierFromContext(ctx)
    if !supported {
        return &rpc.Subscription{}, rpc.ErrNotificationsUnsupported
    }

    rpcSub := notifier.CreateSubscription()

    go func() {
        headers := make(chan *types.Header)
        headersSub := api.events.SubscribeNewHeads(headers)

        for {
            select {
            case h := <-headers:
                notifier.Notify(rpcSub.ID, h)
            case <-rpcSub.Err():
                headersSub.Unsubscribe()
                return
            case <-notifier.Closed():
                headersSub.Unsubscribe()
                return
            }
        }
    }()

    return rpcSub, nil
}
```
这个方法很简单，主要分两块：一是调用 `rpc.NotifierFromContext` 获取 `Notifier` 变量，然后调用 `Notifier.CreateSubscription` 生成一个 `Subscription` 对象；二是创建一个 goroutine，用来不停地监听 header 创建的消息，并使用 `Subscription.Notify` 进行通知。

我们先来看 `Notifier.CreateSubscription` 的调用。这里关键之一是获取 `notifier` 对象。这个 `notifier` 就是我们前面提到过的，消息处理函数刚开始运行时调用 `newNotifier` 设置到 `ctx` 中的对象。那么 `Notifier.CreateSubscription` 都做了什么呢？我们来看一下它的代码：
```go
func (n *Notifier) CreateSubscription() *Subscription {
    s := &Subscription{ID: NewID(), err: make(chan error)}
    n.subMu.Lock()
    n.inactive[s.ID] = s
    n.subMu.Unlock()
    return s
}
```
很简单，生成一个新的 `Subscription` 对象，并把它加入到 `Notifier.inactive` 中。这里另一个重点是调用 `NewID` 生成一个新的 ID。

刚才我们提到过，在有数据到来时，调用 `Subscription.Notify` 进行通知。我们现在就来看看这个方法是如何实现的：
```go
func (n *Notifier) Notify(id ID, data interface{}) error {
    n.subMu.Lock()
    defer n.subMu.Unlock()

    if sub, active := n.active[id]; active {
        n.send(sub, data)
    } else {
        n.buffer[id] = append(n.buffer[id], data)
    }
    return nil
}
```
可以看到，如果参数中指定的 id 记录在 `Notifier.active` 字段中，就直接调用 `Notifier.send` 发送要通知的数据；如果不在 `Notifier.active` 中，就将其暂存到 `Notifier.buffer` 中（至于原因我们一会再说）。`Notifier.send` 很简单，它的代码如下；
```go
func (n *Notifier) send(sub *Subscription, data interface{}) error {
    notification := n.codec.CreateNotification(string(sub.ID), sub.namespace, data)
    err := n.codec.Write(notification)
    if err != nil {
        n.codec.Close()
    }
    return err
}
```
可以看到，它直接把数据发送给了客户端。

到目前为止，整个订阅过程基本都走通了，但这个过程中还有几个「奇怪」的问题：一是 `Server.handle` 中处理订阅请求时，还会返回一个 `activateSub` 函数，它的定义如下：
```go
activateSub := func() {
    notifier, _ := NotifierFromContext(ctx)
    notifier.activate(subid, req.svcname)
}
```
二是在调用 `Subscription.Notify` 对新产生的数据进行通知时，消息 ID 可能并不在 `Notifier.active` 中。
三是在调用 `Notifier.CreateSubscription` 方法时，新生成的 `Subscription` 对象加入到了 `Notifier.inactive` 字段中，而不是 `Subscription.Notify` 中使用的 `Notifier.active` 字段。

其实将这三点放在一起，整个逻辑就通了，它们各自也就不那么奇怪了。在调用 API 订阅消息成功以后，并不能说整个订阅成功了，而是在将订阅 ID 和订阅成功的信息发到客户端以后，才算成功，此时才能将订阅数据发送给客户端。否则客户端还没收到 ID，却已经收到了订阅数据，会莫名其妙。这一逻辑在体现在代码中，则要从 `Server.exec` 看起：
```go
func (s *Server) exec(ctx context.Context, codec ServerCodec, req *serverRequest) {
    ......

    else {
        response, callback = s.handle(ctx, codec, req)
    }
    if err := codec.Write(response); err != nil {
        log.Error(fmt.Sprintf("%v\n", err))
        codec.Close()
    }

    // when request was a subscribe request this allows these subscriptions to be actived
    if callback != nil {
        callback()
    }
}
```
可以看到，当 `Server.handle` 返回后，只有将 `response` 成功发送给客户端之后，才会调用 `callback`，这个变量代表的函数是由 `Server.handle` 返回的，查看代码可以很容易的看到它的定义：
```go
activateSub := func() {
    notifier, _ := NotifierFromContext(ctx)
    notifier.activate(subid, req.svcname)
}
```
显然，在这个函数中使用本次订阅的 ID 作为参数，调用了 `Notifier.activate` 方法。我们继续看这个方法的实现：
```go
func (n *Notifier) activate(id ID, namespace string) {
    n.subMu.Lock()
    defer n.subMu.Unlock()

    if sub, found := n.inactive[id]; found {
        sub.namespace = namespace
        n.active[id] = sub
        delete(n.inactive, id)
        // Send buffered notifications.
        for _, data := range n.buffer[id] {
            n.send(sub, data)
        }
        delete(n.buffer, id)
    }
}
```
很显然，这个方法将存储于 `Notifier.inactive` 中的 ID 相关的信息，转移到 `Notifier.active` 中，并将存储于 `Notifier.buffer` 中的数据，通过 `Notifier.send` 发送给客户端。联系我们之前说过的 `Notifier.Notify` 方法，以及创建订阅时的 `Notifier.CreateSubscription` 方法，整个逻辑就很清楚了：服务端的创建订阅消息成功后，如果有数据产生，并不能立即发送给客户端，而是要等到将订阅 ID 发给客户端以后才能发送为些数据，因此代码在实现时先将订阅相关信息保存到 `Notifier.inactive` 中。在将 ID 成功发送给客户端以后，再将其转移到 `Notifier.active` 中保存，在这之前如果有数据要发送，全部保存在 `Notifier.buffer` 中，直到将 ID 转移到 `Notifier.active` 后，再真正将这些数据发送给客户端。

**整个订阅框架其实并不复杂，它以 `ctx` 中的 `Notifier` 变量为中心，每当有订阅请求时，在相应的服务 API 中（如 `PublicFilterAPI.NewHeads`）调用 `Notifier.CreateSubscription` 创建一个订阅对象，并启动一个 goroutine ，监听和发送订阅的数据和订阅 ID。 rpc 模块的代码则会将服务 API 中创建的订阅对象的 ID 发送给客户端，这样客户端收到数据时就能知道是哪次订阅发送的数据。一个小 trick 就是，在未将 ID 发送给客户端之前，订阅产生的数据不会发送给客户端，而是缓存在 `Notifier.buffer` 中。**


# 连接协议

以太坊 rpc 模块的服务端提供了三种连接方式：HTTP、websocket 和 IPC。其中 IPC 使用 pipe 的方式，比较简单我们就不讨论了，这一小节主要讨论一下 HTTP 和 websocket 这两个方式中的一些关键点。


### HTTP

使用 HTTP 的方式进行连接很常见，一般 RPC 服务都会提供这种方式。在 go 语言中使用 HTTP 实现一个 server 端也非常方便。在 `StartHTTPEndpoint` 函数中是这样实现的：
```go
func StartHTTPEndpoint(endpoint string, apis []API, modules []string, cors []string, vhosts []string, timeouts HTTPTimeouts) (net.Listener, *Server, error) {
    ......

    if listener, err = net.Listen("tcp", endpoint); err != nil {
        return nil, nil, err
    }
    go NewHTTPServer(cors, vhosts, timeouts, handler).Serve(listener)
    ......
}
```
这里调用 `net.Listen` 创建一个 `listener`，然后利用 `NewHTTPServer` 返回的 `http.Server` 对象，调用其 `Serve` 方法就可以构造一个 HTTP 服务端。这个服务端的消息处理函数就是 `http.Server.Handler` 字段代表的对象。

其实在 go 的 http 包里，有更简单的方法创建一个 HTTP 服务，但这不是本文的重点，这里就不展开讲了。

细心的你是否发现上面的代码中，`NewHTTPServer` 有两个参数 `cors` 和 `vhosts`。一般来说要创建一个 HTTP 服务只要有地址和端口就可以了（即 `endpoint` 变量），这两个参数是用来做什么的呢？


##### virtual host

在 geth 程序的命令行参数中，有一个 `--rpcvhosts` 的选项，它用来指定可以正常访问 rpc 服务的「 virtual hostnames」。在这个选项中指定的值，最后就会存储在 `vhosts` 变量中传给 `NewHTTPServer` 函数。

这里有一个[pull request](https://github.com/ethereum/go-ethereum/pull/15962)，详细说明了 `--rpcvhosts` 选项的意义。在那篇文章里，作者说明加这个参数主要是为了阻止绕过同源策略（Same Origin Policy，简称 SOP）的攻击，这种攻击会伪装成同源的形式。比如 DNS 重绑定攻击（DNS rebinding）。因此要理解 `--rpcvhosts` 选项及 `vhosts` 在 `NewHTTPServer` 函数中的意义，我们首先要了解一下什么是 DNS 重绑定攻击。

要理解 DNS 重绑定攻击，要从「同源策略」说起。在目前所有的浏览器中，都允许你同时打开多个网页，但不同网页之间的资源（无论是本地还是服务端的资源）是不可以相互访问的，除非这些网页是「同源」的。这里的同源，是指协议、域名、端口都相同。比如你同时打开了京东和淘宝，那么京东的网页代码是无法访问淘宝生成在本地的 Cookie 的，也不能向淘宝的服务器请求数据；但你在浏览器某个 tab 页中登录京东后，在另一个 tab 页中再次打开京东的网页，也是已登录的状态，这是因为这俩网页是同源的。

可以看出，「同源策略」是浏览器安全的基础。但有一些方式可以绕过这种策略，DNS 重绑定就是一种。攻击者首先申请一个域名并布署一个恶意网站，比如「 http://www.attack.com 」，同时将这个域名的 TTL 设置得特别小（ TTL 规定了域名解析结果在本地的缓存时间，如果 TTL 特别小，本地缓存会很快失效，因此在访问这个域名时几乎每次都需要重新发起 DNS 请求解析域名）。这个恶意网站的某个脚本中可能会实现这样一个逻辑：首先修改「 http://attack.com 」的 DNS 信息，使其对应一个想攻击的服务器资源；然后再次访问这个域名。运行此脚本时访问恶意域名时，由于 TTL 值特别小，会再次发起 DNS 解析请求，但域名对应的 IP 已被修改，因此这时请求到的 IP 地址已经是修改后的要攻击的服务器，而非原来自己的服务器。但浏览器依然认为这次访问是同源的，就会允许这次访问，从而使攻击成功。

比如有人想攻击某公司的内部服务器。由于这个服务器的只能在公司内网使用，因此直接外网访问是访问不到的。攻击者可以诱使某内部员工访问自己的恶意网站，然后使用网站的脚本先修改自己的恶意网站的 DNS 记录，使其指向内网服务器，然后再访问自己的恶意网站。这时访问到的，就是内网的服务器了。

在这个例子中，如果这个内网的服务正是你的私有以太坊网络，你本意不想对外公开这个网络，但由于重绑定攻击的存在，你是无法完全避免数据泄露的。

使用 `--rpcvhosts` 就可以阻止这种情况。这里需要了解一点的是，即使重绑定可以访问到没权限访问的服务，但服务程序在接收到请求后获取请求的「 Host 」字段时，获取到的是恶意域名，如果服务程序去比对这个 Host 数据发现不是自己认可的域名，就可以拒绝访问。`--rpcvhosts` 就是利用这一点防止 DNS 重绑定攻击的。

我们具体看看代码是如何实现的。假如在启动以太坊服务时，我们传入了 `--rpcvhosts http://myservice.com`，那么最终这个值会传到 `NewHTTPServer` 的 `vhosts` 参数中：
```go
func NewHTTPServer(cors []string, vhosts []string, timeouts HTTPTimeouts, srv *Server) *http.Server {
    ......
    handler = newVHostHandler(vhosts, handler)
    ......

    return &http.Server{
        Handler:      handler,
        ......
    }
}
```
在这段代码里又将 `vhosts` 传给了 `newHostHandler`。我们再来看看这个函数：
```go
func newVHostHandler(vhosts []string, next http.Handler) http.Handler {
    vhostMap := make(map[string]struct{})
    for _, allowedHost := range vhosts {
        vhostMap[strings.ToLower(allowedHost)] = struct{}{}
    }
    return &virtualHostHandler{vhostMap, next}
}

type virtualHostHandler struct {
    vhosts map[string]struct{}
    next   http.Handler
}
```
在 `newVHostHandler` 中，创建了一个 `virtualHostHandler` 对象，并将 `vhosts` 的信息存储到了 `virtualHostHandler.vhosts` 字段中。也就是说，`--rpcvhosts` 中的值最终将作为 `virtualHostHandler.vhosts` 的 Key 存储起来。

`virtualHostHandler` 对象作为填充 `http.Server.Handler` 字段的值，`virtualHostHandler.ServeHTTP` 方法是处理 HTTP 请求的唯一方法。我们看一下它的代码是如何阻止攻击的：
```go
func (h *virtualHostHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // if r.Host is not set, we can continue serving since a browser would set the Host header
    if r.Host == "" {
        h.next.ServeHTTP(w, r)
        return
    }
    host, _, err := net.SplitHostPort(r.Host)
    if err != nil {
        // Either invalid (too many colons) or no port specified
        host = r.Host
    }
    if ipAddr := net.ParseIP(host); ipAddr != nil {
        // It's an IP address, we can serve that
        h.next.ServeHTTP(w, r)
        return
    }

    // Not an ip address, but a hostname. Need to validate
    if _, exist := h.vhosts["*"]; exist {
        h.next.ServeHTTP(w, r)
        return
    }
    if _, exist := h.vhosts[host]; exist {
        h.next.ServeHTTP(w, r)
        return
    }
    http.Error(w, "invalid host specified", http.StatusForbidden)
}
```
这个方法虽然有点长，但很简单。它首先从请求数据中拿到请求时的 host 地址，然后看这是否是一个 IP，如果是，说明发送请求时没有使用域名，而是直接使用的 IP，因此允许访问；接着看 `virtualHostHandler.vhosts` 中是否存在「 * 」作为 Key，如果有则不关心使用的域名是什么，都可以访问；最后如果前面的检查都没通过，那么就会查看 `virtualHostHandler.vhosts` 中是否存在解析出来的 host 作为 Key，如果不存在，说明客户端使用的不是一个我们认可的域名，就会拒绝访问，返回「 invalid host specified 」的错误。


##### CORS

上一小节里，我们提到过「同源策略」，即在浏览器中只允许网页代码访问与自身同源的资源。但有时我们又有同一网页访问不同服务器资源的情况，比如一个展示图片的网页，但图片很可能来自第三方的、专门提供图片存储服务的服务器。基于这种实际需求，就有了「 CORS 」（Cross-origin resource sharing，跨域资源共享）。

所以事实上「 CORS 」是一种机制，它需要客户端和服务端共同配合和支持。由于本篇文章的重点不在于此，因此不展开介绍，感兴趣的朋友可以自己搜索相关文章进行了解。这里我们只是说明一下以太坊的 rpc 模块是如何支持 CORS 的。

你可以使用参数 `--rpccorsdomain` 配置允许访问的其它的域名。比如你有一个来自「 www.myservice.com 」网页，而你的以太坊运行在「www.my-eth.com」这台服务器上。你想在你的网页代码中通过 rpc 访问你的以太坊服务，那么你的以太坊服务在启动时就需要加上如下参数：「 --rpccorsdomain www.myservice.com 」。如果不指定，你可能会得到类似于「 can't execute CORS 」的错误。

在代码中，`--rpccorsdomain` 中指定的数据最终将通过 `cors` 参数传给 `NewHTTPServer` 函数：
```go
func NewHTTPServer(cors []string, vhosts []string, timeouts HTTPTimeouts, srv *Server) *http.Server {
    // Wrap the CORS-handler within a host-handler
    handler := newCorsHandler(srv, cors)
    handler = newVHostHandler(vhosts, handler)
    ......
```
在 `NewHTTPServer` 中将调用 `newCorsHandler` 将 `cors` 参数传给它。我们再来看看这个函数：
```go
func newCorsHandler(srv *Server, allowedOrigins []string) http.Handler {
    // disable CORS support if user has not specified a custom CORS configuration
    if len(allowedOrigins) == 0 {
        return srv
    }
    c := cors.New(cors.Options{
        AllowedOrigins: allowedOrigins,
        AllowedMethods: []string{http.MethodPost, http.MethodGet},
        MaxAge:         600,
        AllowedHeaders: []string{"*"},
    })
    return c.Handler(srv)
}
```
在这里如果 `allowedOrigins` 里的内容为空，则不使用 cors 服务，否则利用 `allowedOrigins` 中的数据，使用 cors 库创建一个 `http.Handler` 对象。


### websocket

websocket  是一种网络通信协议，与 HTTP 显著不同的是，它可以主动发数据给客户端，这个特性在 web 页面中实现一些高级功能时特别好用。本篇文章的重点不在于介绍 websocket，所以想了解更多详细内容的朋友可以自己搜索相关文章。这一小节里，我们主要介绍一下以太坊的 rpc 模块是如何创建一个 websocket 服务的。

在启动 websocket 服务的函数 `StartWSEndpoint` 中，代码调用了 `NewWSServer` 创建一个 websocket 服务对象，我们看看 `NewWSServer` 是如何实现的：
```go
func NewWSServer(allowedOrigins []string, srv *Server) *http.Server {
    return &http.Server{Handler: srv.WebsocketHandler(allowedOrigins)}
}
```
可以看到，websocket 服务对象其实也是一个 `http.Server` 对象，这与创建 HTTP 服务时是一样的。不同的是这里的 handler 是 `Server.WebsocketHandler` 返回的值。所以我们看看它的实现：
```go
func (srv *Server) WebsocketHandler(allowedOrigins []string) http.Handler {
    return websocket.Server{
        Handshake: wsHandshakeValidator(allowedOrigins),
        Handler: func(conn *websocket.Conn) {
            // Create a custom encode/decode pair to enforce payload size and number encoding
            conn.MaxPayloadBytes = maxRequestContentLength

            encoder := func(v interface{}) error {
                return websocketJSONCodec.Send(conn, v)
            }
            decoder := func(v interface{}) error {
                return websocketJSONCodec.Receive(conn, v)
            }
            srv.ServeCodec(NewCodec(conn, encoder, decoder), 
                OptionMethodInvocation|OptionSubscriptions)
        },
    }
}
```
从这里可以看到，websocket 服务的消息处理对象，其实是利用现成的库 websocket 创建的。这里创建了一个 `websocket.Server` 结构体作为消息请求的处理对象，它的 Handler 字段是一个匿名函数，最终调用的是 `Server.ServeCodec` 方法。注意它们的编码和解码方法也与 HTTP 的不同。

另外 websocket 建立的连接是持久连接，因此一旦连接建立起来，它的 handler 就会一直运行，直到连接中断。


# 总结

在这篇文章里，我们详细分析了以太坊的 rpc 模块的各项功能和代码。以太坊的 rpc 模块提供了三种连接方式：HTTP、websocket 和 IPC，其中 IPC 使用的 pipe 通信方式。除了 HTTP 连接以外，其它两种连接都可以使用「订阅」功能。另外，以太坊的 rpc 模块充分利用了 go 语言的反射特性，各个提供服务的模块只需将包含服务功能的对象提交到 rpc 模块中，它就能解析各个对象的导出方法，作为服务的 API。

需要强调一点的是，以太坊的 rpc 模块的「订阅」功能非常的好用，我看了一下 go 语言自己的 rpc 库，好像还没有这种「订阅」功能。

以上就是对 rpc 模块的所有分析。水平有限，如果有错误还请留言或邮件指出，非常感谢。