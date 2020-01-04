---
layout: post
title:  "PBFT代码篇：fabric 中的 PBFT 实现"
categories: blockchain
tags: 原创 consensus PBFT fabric
excerpt: 之前的文章我们学习了 PBFT 算法的三阶段和 ViewChange 等关键要素，但在实际的项目中，PBFT 是如何被实现的呢？这篇文章，我们就从 fabric 中找找答案。
author: fatcat22
mathjax: true
---

* content
{:toc}





>本篇文章分析的源码地址为：[https://github.com/hyperledger/fabric/tree/v0.6.0-preview](https://github.com/hyperledger/fabric/tree/v0.6.0-preview)  
>分支：[tag: v0.6.0-preview](https://github.com/hyperledger/fabric/tree/v0.6.0-preview)  
>commit id: [d9fb2195ceb198efe8714a480c6b1afd08694406](https://github.com/hyperledger/fabric/commit/d9fb2195ceb198efe8714a480c6b1afd08694406)  


# 引言

在[上一篇文章](https://yangzhe.me/2019/11/25/pbft/)里，我们学习了 PBFT 算法，其中主要包括 PBFT 的三个阶段（pre-prepare、prepare、commit）、View Change 以及 checkpoint 等。只学完理论毕竟感觉不踏实，所以这篇文章，我们从源代码出发，来学习一下 PBFT 的相关实现。

使用 PBFT 作为共识算法的区块链项目不多，Hyperledger Fabric 算是其中一个。不过 Fabric 项目也只是在比较早的版本 v0.6.0-preview 中使用了 PBFT，在那之后就不用了。所以本篇文章中我们使用的源代码，是 Fabric 项目的 0.6.0 版本。以下再次提到「Fabric」时，一般都是指它的 0.6 版本。

在下面的介绍中，我们会经常提到[上一篇介绍 PBFT 理论的文章]((https://yangzhe.me/2019/11/25/pbft/))，后面我们将其称为「理论篇」。

# Fabric 是什么

虽然这篇文章的主题是 PBFT 的实现，但我们学习的是 Hyperledger Fabric 项目的源码，所以有必要先简单介绍一下它。

根据官网的介绍，Fabric 是一个企业级的、跨行业的、有准入功能的分布式账本框架，企业之间可以使用它快速方便地搭建分布式的、可信的账本平台。其实这个项目还是很有名气的，相信很多小伙伴都听说过它，不过我们通常将其作为「联盟链」的代表项目。

所谓联盟链，是相对于开放式区块链来说的。在普通区块链如以太坊中，任何人随时都可以进入或退出区块链网络、不需经过任何许可就可获得所有区块链上的数据。而联盟链由于提供了准入功能，因此只有参与合作的企业才能进入到链网络中、拿到链上的数据，这就是「联盟」这个名字的由来。

不过 Fabric 不是一个具体的联盟链项目，而是一个框架，企业想要实现自己的联盟链，需要在这个框架上加上自己的业务逻辑。Fabric 框架提供了一些联盟链上的基本功能，共识算法作为区块链的标配，当然也是由这个框架提供了。Fabric 在 v0.6 版本上使用的是 PBFT（最新的版本已经不是了），我们的这篇文章就是对这块代码进行分析学习。


# pbft 模块目录结构

在正式学习代码之前，我们先介绍一下 pbft 模块源码的目录结构，以对相关功能和实现有个大体的了解。

Fabric 中关于 PBFT 共识的所有源码都位于目录 consensus/pbft 目录下。下面我们分别介绍一下每个源码文件中的代码的大体功能。

- batch.go  
这个文件中主要实现了 `obcBatch` 对象其及方法。这个对象在稍高的层次上包装了 PBFT，它包含了 PBFT 执行的主要流程，也是共识接口（`consensus.Consenter`）的直接实现。但 PBFT 算法却并不在这个对象中直接实现的，而是在 `pbftCore` 中。

- broadcast.go  
这个文件中的代码实现了信息的发送和广播功能。在介绍 PBFT 时我们说过，PBFT 中的每一个结点都有一个编号。这个文件中的代码，就是根据结点编号来区分不同结点、从而知道消息该发给谁。

- config.yaml  
这不是源代码，而是一个 PBFT 的配置文件。但这个配置文件很值得看一下，从配置的字段中可以体现很多 PBFT 的特性。

- deduplicator.go  
这个文件中的代码实现了 `deduplicator` 对象其及方法。正如它的名字所示，这个对象通过记录其它结点的请求时间戳和执行消息的时间戳，可以对重复或无效的消息进行处理。

- external.go  
这个文件中的代码定义了一些事件对象和 `externalEventReceiver` 对象。如文件名和对象名字所示，这里实现的代码主要是接收 pbft 模块外部传送的消息，并将其提交到内部消息循环中进行处理。`externalEventReceiver` 对象是对外接口 `ExecutionConsumer` 的具体实现。

- messages.proto  
Fabric 中的 PBFT 共识使用 protobuf 协议传递消息，这个文件就是对 PBFT 中用到的消息的定义。由这个文件、使用 protoc-gen-go 程序，就可以生成 go 源码文件 messages.pb.go。

- pbft-core.go  
这个文件中的代码主要实现了 `pbftCore` 对象及其方法。从名字也能看出来，这个文件中的代码是 PBFT 算法的核心实现，也是我们要学习的重点。但 `pbftCore` 对象的实现不仅限于这个文件中，还包含 pbft-persist.go、sign.go、viewchange.go。

- pbft-persist.go  
这个文件中的代码实现了 `pbftCore` 对象的关于 PBFT 算法中数据的存储和恢复相关的功能。其实数据的存储和恢复不是 PBFT 的主要功能，也不是共识算法的功能，因此这里其实是对 `StatePersistor` 接口的封装。具体功能的实现，需要用户根据实际业务逻辑，实现 `StatePersistor` 接口。

- pbft.go  
这个源文件中的代码主要实现了 PBFT 对外的创建方法：`New` 和 `GetPlugin`。

- persist-forward.go  
这个文件中实现了 `persistForward` 对象，这是对 `consensus.StatePersistor` 接口的封装。但实际上代码中好像并没有地方使用 `persistForward` 对象，我想应该是被 pbft-persist.go 中的实现替代了。

- requeststore.go  
这个文件主要实现了 `requestStore` 对象。这个对象实现了内存中的请求的管理。

- sign.go  
这个文件中实现了 `pbftCore` 对象中关于消息签名和验证方面的功能，以及 `ViewChange` 对象中关于 id、设置签名及验证签名的功能。

- viewchange.go  
这个文件中实现了 `pbftCore` 对象中关于域转换的相关逻辑，也是我们学习的重点之一。


以上就是 pbft 目录下的源文件的简单说明。另外，由于 Fabric 作为一个框架，很多东西是插件式的，所以理论上不仅可以使用 PBFT 这一个共识算法。consensus 目录下的除 pbft 之外的代码基本上就是用来完成这一「框架」的，这些代码并不多，也不复杂，我们这里就不多作介绍了，后面用到的时候再说。


# 预热：一些概念的解释

为了后面更容易弄明白 Fabric 中 PBFT 的实现，我们需要对一些知识提前作一些说明。这样不管是自己去学习相关代码，还是看下面的文章，我觉得会更容易理解些。


## 插件式

首先，我们要了解一下 PBFT 共识在整个 Fabric 中所处的位置。首先很清楚的是，它处在 Fabric 的共识模块中，也就是 consensus 目录下。Fabric 的共识使用的接口是 `consensus.Consenter`，之前我们说过， Fabric 的设计上很多功能是「插件式」的，共识也不例外，只要实现了 `consensus.Consenter` 接口，就可以作为 Fabric 的共识。这从 consensus/controller/controller.go 中的代码中可以看出来：
```go
func NewConsenter(stack consensus.Stack) consensus.Consenter {
    plugin := strings.ToLower(viper.GetString("peer.validator.consensus.plugin"))
    if plugin == "pbft" {
        logger.Infof("Creating consensus plugin %s", plugin)
        return pbft.GetPlugin(stack)
    }
    logger.Info("Creating default consensus plugin (noops)")
    return noops.GetNoops(stack)
}
```
可以看到，PBFT 作为一个共识算法，被配置在配置文件的 `peer.validator.consensus.plugin` 字段中。如果这个字段是 `pbft`，就调用 `pbft.GetPlugin` 使用 PBFT 算法作为共识算法；如果不是，就使用默认的共识算法 noops （这是一个很简单的对象，我猜应该只能用来调试和测试）。

所以可以看到，PBFT 是 Fabric 的共识算法的一种，只要实现了 `consensus.Consenter` 接口，并稍加修改 `NewConsenter` 函数，都可以作为 Fabric 的共识算法。

那既然 pbft 作为共识算法的实现，只实现了 PBFT 算法的核心功能，那它与 Fabric 的其它模块的交互怎么办呢？这是通过接口实现的。共识的对外接口是 `consensus.Consenter`，在这个接口，有一个方法 `RecvMsg`，当网络模块收到消息需要共识模块处理时，就会调用这个方法将消息传递给共识模块，接口中的其它方法的使用情景与之类似。


## 消息循环

刚才我们提到了在接收到新消息时，共识模块接口 `consensus.Consenter` 的 `RecvMsg` 会被调用。那么在这一小节里，我们就看看消息是如何在 pbft 模块中流传的。因为在 Fabric 的实现中，pbft 的整体是基于消息驱动的，把握了 pbft 的消息流动框架，才不会在分析代码时迷失在各种各样的消息处理中。

之前我们已经提到过，`obcBatch` 对象是在一个稍高的层次上实现了 pbft 模块的共识接口 `consensus.Consenter`，所以 `RecvMsg` 肯定也是它实现的。不过具体的实现不是 `obcBatch`，而是 `externalEventReceiver`（因为 `obcBatch` 中嵌套了 `externalEventReceiver` 对象）：
```go
func (eer *externalEventReceiver) RecvMsg(ocMsg *pb.Message, senderHandle *pb.PeerID) error {
    eer.manager.Queue() <- batchMessageEvent{
        msg:    ocMsg,
        sender: senderHandle,
    }
    return nil
}
```

可见在方法被调用后，会向字段 manager 代表的消息队列中发送 `batchMessageEvent` 消息。那么 `externalEventReceiver.manager` 是一个怎样的实现呢？它是一个 `events.Manager` 接口，这个接口由 `managerImpl` 实现，它位于 consensus/util/events/events.go 中，我们直接看一下代码，或许会更清楚刚才 `RecvMsg` 中发生了什么：
```go
func (em *managerImpl) Start() {
    go em.eventLoop()
}

func (em *managerImpl) Queue() chan<- Event {
    return em.events
}

func (em *managerImpl) eventLoop() {
    for {
        select {
        case next := <-em.events:
            em.Inject(next)
        case <-em.exit:
            logger.Debug("eventLoop told to exit")
            return
        }
    }
}
```
`managerImpl` 对象在启动后，会在 `managerImpl.eventLoop` 中不停地监听 `managerImpl.events` 消息，如果收到消息就调用 `managerImpl.Inject` 方法。我们再来看看这个方法是如何实现的：
```go
func (em *managerImpl) Inject(event Event) {
    if em.receiver != nil {
        SendEvent(em.receiver, event)
    }
}

func SendEvent(receiver Receiver, event Event) {
    next := event
    for {
        // If an event returns something non-nil, then process it as a new event
        next = receiver.ProcessEvent(next)
        if next == nil {
            break
        }
    }
}
```
`managerImpl.Inject` 调用了 `SendEvent` 函数。在这个函数中，将事件消息发送给 `receiver.ProcessEvent`；如果它返回另一个事件，则一直循环调用，直到它返回空。

那么 `receiver` 又是哪个对象呢？我们看一下下面的代码：
```go
func (em *managerImpl) SetReceiver(receiver Receiver) {
    em.receiver = receiver
}

func newObcBatch(id uint64, config *viper.Viper, stack consensus.Stack) *obcBatch {
    op := &obcBatch{
        ......
    }

    op.manager = events.NewManagerImpl() // TODO, this is hacky, eventually rip it out
    op.manager.SetReceiver(op)

    ......
}
```
可见，`managerImpl.receiver` 其实就是 `obcBatch` 对象，在 `SendEvent` 中循环调用的 `receiver.ProcessEvent` 其实就是 `obcBatch.ProcessEvent` 方法。我们粗略地看一眼这个方法：
```go
func (op *obcBatch) ProcessEvent(event events.Event) events.Event {
    switch et := event.(type) {
    case batchMessageEvent:
        ocMsg := et
        return op.processMessage(ocMsg.msg, ocMsg.sender)
    case executedEvent:
        ......
    case committedEvent:
        ......

    ......

    default:
        return op.pbft.ProcessEvent(event)
    }

    return nil
}
```
可以看到 `obcBatch.ProcessEvent` 中是一个大的 switch/case，用来处理各种各样的消息。

至此，我们可以总结出 pbft 中的消息循坏框架了。当 Fabric 的网络模块接收到新消息时，共识模块接口 `consensus.Consenter` 的 `RecvMsg` 方法就会被调用。在 pbft 中，实现这个方法的是 `obcBatch` 对象（实际上是 `externalEventReceiver`，`obcBatch` 嵌套了它）；在 `externalEventReceiver.RecvMsg` 中将消息发给了 `managerImpl` 对象的消息队列；然后 `managerImpl` 对象内部会循环调用 `obcBatch.ProcessEvent` 方法，直到此方法返回 `nil`。


## batch

最后，我们要了解一下 Fabric 的 pbft 模块中关于「batch」的概念。这里的 batch，主要是针对客户端请求（即 messages.pb.go 中的 `Request` 对象）。在 Fabric 中，发送给共识的请求可以是单个请求，也可以是一批请求。单个请求就是发送 `Request` 对象，一批请求就发送 `RequestBatch` 对象（都在 messages.pb.go 中定义）。

在 pbft 内部，请求的处理也是按「批」进行的，而不是按「个」。如果 pbft 收到了代表单个请求的 `Request` 对象，它会将期保存在 `obcBatch.batchStore` 字段中，直到这个字段中的请求数量超过设置的最大值、或计时器超时，才会将这些请求组成一个 `RequestBatch` 对象进行处理。

在 pbft 的配置文件中，有一个字段为 `general.mode`，它的值只有一个，就是 `batch`，也就是 pbft 的 「batch」 模式。我想 pbft 处理请求的这种方式，也是 batch 这个模式名字的由来。


# PBFT 实现

下面我们就来详细看一看 Fabric 中是如何实现 PBFT 的。我们依然按照「理论篇」中介绍 PBFT 理论的思路，先按正常流程进行分析，再看看 超时、 checkpoint 和域转换是如何处理的。


## 接收请求

这一小节里，我们来介绍一下 pbft 是如何接收请求的。这里的「请求」不是泛指，而是特指的是「理论篇」中提到的请求，即客户端发来的消息。主结点收到这些消息后，是要发送 pre-prepare 消息并进入三阶段的处理流程的。

pbft 中代表请求的对象是 `Request`，它的定义如下：
```go
type Request struct {
    Timestamp *google_protobuf.Timestamp 
    Payload   []byte
    ReplicaId uint64
    Signature []byte
}
```
和「理论篇」中介绍得一样，请求中包含了时间戳（Timestamp）、请求操作（Payload）、发起请求的结点（ReplicaId）、签名（Signature）。

但我们在「batch」小节中也提到过，pbft 内部处理请求时，是按「批」处理的。代表此种方式的对象是 `RequestBatch`：
```go
type RequestBatch struct {
    Batch []*Request 
}
```
这个对象非常简单，就是一些 `Request` 对象的集合。

前面我们也介绍过，有新消息时接口 `consensus.Consenter` 的 `RecvMsg` 方法会被调用，最终进入到 `obcBatch.ProcessEvent` 被处理。`RecvMsg` 的实现如下：
```go
func (eer *externalEventReceiver) RecvMsg(ocMsg *pb.Message, senderHandle *pb.PeerID) error {
    eer.manager.Queue() <- batchMessageEvent{
        msg:    ocMsg,
        sender: senderHandle,
    }
    return nil
}
```
可以看到所有消息都被转转成了 `batchMessageEvent` 进入到了 `obcBatch.ProcessEvent` 中。所以我们下面的分析只要关注 `obcBatch.ProcessEvent` 中对这个消息事件的处理，以及处理这个消息事件时返回的事件（消息循环会使用 `obcBatch.ProcessEvent` 的返回值作为参数，不断调用它，直到其返回 `nil`）就可以了。

这里我们先说结论。总得来说，pbft 接收到的请求有三种：类型为 `Message_CHAIN_TRANSACTION` 的消息、类型为 `Request` 的请求、类型为 `RequestBatch` 的请求。处理过程中 `Message_CHAIN_TRANSACTION` 消息会转变成 `Request` 请求，而多个 `Request` 请求会集合在一起转变成 `RequestBatch`，最终由 `pbftCore` 对 `RequestBatch` 进行处理。下面我们详细看一下每种消息的接收和处理流程。


### Message_CHAIN_TRANSACTION

刚才说过，所有消息都首先作为 `batchMessageEvent` 事件进入 `obcBatch.ProcessEvent` 进行处理，代码如下：
```go
func (op *obcBatch) ProcessEvent(event events.Event) events.Event {
    switch et := event.(type) {
    case batchMessageEvent:
        ocMsg := et
        return op.processMessage(ocMsg.msg, ocMsg.sender)
    ......
    }

    return nil
}
```
我们直接看一下 `obcBatch.processMessage` 中与 `Message_CHAIN_TRANSACTION` 类型的消息有关的实现：
```go
func (op *obcBatch) processMessage(ocMsg *pb.Message, senderHandle *pb.PeerID) events.Event {
    if ocMsg.Type == pb.Message_CHAIN_TRANSACTION {
        req := op.txToReq(ocMsg.Payload)
        return op.submitToLeader(req)
    }

    ......
}

func (op *obcBatch) txToReq(tx []byte) *Request {
    now := time.Now()
    req := &Request{
        Timestamp: &timestamp.Timestamp{
            Seconds: now.Unix(),
            Nanos:   int32(now.UnixNano() % 1000000000),
        },
        Payload:   tx,
        ReplicaId: op.pbft.id,
    }
    // XXX sign req
    return req
}
```

从上面的代码可以看到，`obcBatch.txToReq` 其实是将一条 Transaction （交易，特指区块链中的一笔交易）转变成了 `Request` 请求对象，然后调用 `obcBatch.submitToLeader` 提交这个请求给主结点：
```go
func (op *obcBatch) submitToLeader(req *Request) events.Event {
    // Broadcast the request to the network, in case we're in the wrong view
    op.broadcastMsg(&BatchMessage{Payload: &BatchMessage_Request{Request: req}})

    ......

    if op.pbft.primary(op.pbft.view) == op.pbft.id && op.pbft.activeView {
        return op.leaderProcReq(req)
    }
    return nil
```
这个方法首先不会去判断自己是不是主结点，而是一律广播这个请求给所有结点，根据注释我猜是为了保险起见：万一自己认为自己是主结点，但其实不是，那这条请求就不会被执行了。接下来才会去判断自己是不是主结点，如果是，就调用 `obcBatch.leaderProcReq` 处理这个请求对象。

`obcBatch.leaderProcReq` 如何处理这个请求呢？注意此时 `Message_CHAIN_TRANSACTION` 类型的消息也即一条 Transaction 已经完全变成了一个 `Request` 请求，所以后续的处理方式应该与直接收到 `Request` 请求对象相同了。这一过程我们下一小节再介绍。

（虽然我没有完整看过 Fabric 的代码，不过从这里也可以看出，在 Fabric 中把一条 Transaction 也当作请求来处理了；从 `obcBatch.execute` 、 `Executor.Execute` 等方法中也可以看出，貌似 `Request` 请求对象中**默认**只包含 Transaction 一种请求方式）

所以总得来说，当 pbft 模块收到 `Message_CHAIN_TRANSACTION` 类型的消息后，会将其转换成 `Request` 请求对象，并提交主结点进行处理。


### Request

与 `Message_CHAIN_TRANSACTION` 类型的消息类似，`Request` 消息也是通过 `obcBatch.ProcessEvent` 并调用 `obcBatch.processMessage` 进行处理。我们直接看一下 `obcBatch.processMessage` 中与 `Request` 请求对象有关的代码：
```go
func (op *obcBatch) processMessage(ocMsg *pb.Message, senderHandle *pb.PeerID) events.Event {
    ......

    batchMsg := &BatchMessage{}
    err := proto.Unmarshal(ocMsg.Payload, batchMsg)

    if req := batchMsg.GetRequest(); req != nil {
        if !op.deduplicator.IsNew(req) {
            return nil
        }

        if (op.pbft.primary(op.pbft.view) == op.pbft.id) && op.pbft.activeView {
            return op.leaderProcReq(req)
        }
        op.startTimerIfOutstandingRequests()
        return nil
    } else if ...... {
        ......
    }

    return nil
}
```

在上面的这段代码中，首先将请求消息反序列化为一个 `BatchMessage` 对象，这个对象只是一个「壳」，它可能是四种类型的对象之一，这从 `BatchMessage` 的定义的注释就能看出来：
```go
type BatchMessage struct {
    // Types that are valid to be assigned to Payload:
    //	*BatchMessage_Request
    //	*BatchMessage_RequestBatch
    //	*BatchMessage_PbftMessage
    //	*BatchMessage_Complaint
    Payload isBatchMessage_Payload 
}

type BatchMessage_Request struct {
    Request *Request 
}
```
目前我们关心的是 `*BatchMessage_Request` 这种类型。所以随后的代码尝试调用 `BatchMessage.GetRequest` 来获取 `Request` 对象。

如果 `BatchMessage.GetRequest` 成功，说明刚才的 `BatchMessage` 真正代表的是一个 `Request` 请求。那么接下来代码首先利用 `deduplicator` 对象判断这个 `Request` 请求是否是重复的（`deduplicator` 对象比较简单，这里就不展开说了）。如果是一个新收到的请求，且自己就是主结点，那么就会调用 `obcBatch.leaderProcReq` 处理这个请求。

（上一小节介绍 `Message_CHAIN_TRANSACTION` 类型的消息时，代码将 Transaction 转换成一个 `Request` 对象后，也是调用了 `obcBatch.leaderProcReq` 进行处理。所以这两种类型的消息的处理方式在这里汇合了。）

我们接下来看看 `obcBatch.leaderProcReq` 的实现：
```go
func (op *obcBatch) leaderProcReq(req *Request) events.Event {
    op.batchStore = append(op.batchStore, req)

    if !op.batchTimerActive {
        op.startBatchTimer()
    }

    if len(op.batchStore) >= op.batchSize {
        return op.sendBatch()
    }

    return nil
}

func (op *obcBatch) sendBatch() events.Event {
    op.stopBatchTimer()
    if len(op.batchStore) == 0 {
        return nil
    }

    reqBatch := &RequestBatch{Batch: op.batchStore}
    op.batchStore = nil
    return reqBatch
}
```

在 `obcBatch.leaderProcReq` 中，首先将 `Request` 对象存储到 `obcBatch.batchStore` 字段中；然后确保启动 batch timer （用来每隔一段时间调用 `obcBatch.sendBatch`，后面会有关于计时器的集中介绍）；最后如果 `obcBatch.batchStore` 中请求的数量大于某一数值，则调用 `obcBatch.sendBatch` 「发送」这些请求。

`obcBatch.sendBatch` 的代码非常简单，在其中我们并没有看到任何「send」有关的代码。它只是简单地使用 `obcBatch.batchStore` 中的所有 `Request` 对象构造一个 `*RequestBatch` 对象，然后将其返回而已。

仔细看一下代码会发现，`obcBatch.sendBatch` 中返回的 `*RequestBatch` 对象，最终会让 `obcBatch.ProcessEvent` 返回相同的值。前面我们提到过，消息循环的处理方式是不停地以 `obcBatch.ProcessEvent` 的返回值作为新的参数调用这个方法，直到其返回 `nil`。所以这里返回的 `*RequestBatch` 对象后，又会重新以这个对象为参数再次调用 `obcBatch.ProcessEvent` 方法。而此时不会再进入 `batchMessageEvent` 这个 case 分支了，而是进入处理 `*RequestBatch` 的分支（`obcBatch.ProcessEvent` 并没有 `*RequestBatch` 分支，而是在 `pbftCore.ProcessEvent` 中）。具体情况我们下一小节进行分析。

到这里我们可以看到，pbft 虽然也可以直接接收 `Request` 对象，但内部仍然是将这些对象「攒一起」，然后作为 `RequestBatch` 来处理的。


### RequestBatch

前面提到过，所有请求消息都进入到 `obcBatch.ProcessEvent` 中的 `batchMessageEvent` 分支、调用 `obcBatch.processMessage` 进行处理。但在 `obcBatch.processMessage` 方法中并没有处理 `RequestBatch` 请求。事实上，此时的 `RequestBatch` 请求是包含在 `BatchMessage_PbftMessage` 消息中的，而 `obcBatch.processMessage` 中恰好有针对这个消息的处理，所以我们先看一下相关代码：
```go
func (op *obcBatch) processMessage(ocMsg *pb.Message, senderHandle *pb.PeerID) events.Event {
    ......

    else if pbftMsg := batchMsg.GetPbftMessage(); pbftMsg != nil {
        senderID, err := getValidatorID(senderHandle) // who sent this?
        msg := &Message{}
        err = proto.Unmarshal(pbftMsg, msg)
        return pbftMessageEvent{
            msg:    msg,
            sender: senderID,
        }
    }

    return nil
}
```
这里的重点是，代码将接收到的消息转变成了一个 `pbftMessageEvent` 对象并返回了。然后消息循环会再次使用 `pbftMessageEvent` 作为参数调用 `obcBatch.ProcessEvent`，这时又会在哪个分支进行处理呢？

查看 `obcBatch.ProcessEvent` 方法的代码，没有直接处理 `pbftMessageEvent` 事件的代码，只能进入到 default 分支，调用 `pbftCore.ProcessEvent` 进行处理了。我们看一下这个方法只关于 `pbftMessageEvent` 的处理：
```go
func (instance *pbftCore) ProcessEvent(e events.Event) events.Event {
    switch et := e.(type) {
    ......

    case pbftMessageEvent:
        msg := et
        next, err := instance.recvMsg(msg.msg, msg.sender)
        if err != nil {
            break
        }
        return next
    ......
    }
}

func (instance *pbftCore) recvMsg(msg *Message, senderID uint64) (interface{}, error) {
    if reqBatch := msg.GetRequestBatch(); reqBatch != nil {
        return reqBatch, nil
    } else if ...
    .......
}
```

这里调用 `pbftCore.recvMsg` 处理 `pbftMessageEvent` 事件。而 `pbftCore.recvMsg` 的逻辑也非常简单，就是判断并转换参数 `msg` 为真正的类型，并返回。这里我们主要关心 `*RequestBatch` 类型，可以看到如果 `msg` 是这一类型，就会将其直接返回。消息循环会再次利用这个 `*RequestBatch` 类型的变量为参数，调用 `pbftCore.ProcessEvent` 方法。

下面我们看看 `pbftCore.ProcessEvent` 中关于 `*RequestBatch` 分支的处理：
```go
func (instance *pbftCore) ProcessEvent(e events.Event) events.Event {
    switch et := e.(type) {
    ......

    case *RequestBatch:
        err = instance.recvRequestBatch(et)

    ......

    }
}
```
这个方法里对 `*RequestBatch` 分支的处理特别简单，就是直接调用 `pbftCore.recvRequestBatch` 进行处理。

我们重点看一下 `pbftCore.recvRequestBatch` 是如何处理的：
```go
func (instance *pbftCore) recvRequestBatch(reqBatch *RequestBatch) error {
    digest := hash(reqBatch)

    // 保存 *RequestBatch 对象
    instance.reqBatchStore[digest] = reqBatch
    instance.outstandingReqBatches[digest] = reqBatch
    instance.persistRequestBatch(digest)

    // 收到请求后启动计时器。如果超时代表请求处理失败，要发起 View Change。
    if instance.activeView {
        instance.softStartTimer(
            instance.requestTimeout, 
            fmt.Sprintf("new request batch %s", digest))
    }

    // 如果自己是主结点，主要马上发送 pre-prepare 消息
    if instance.primary(instance.view) == instance.id && instance.activeView {
        instance.nullRequestTimer.Stop()
        instance.sendPrePrepare(reqBatch, digest)
    } else {
        logger.Debugf(...)
    }
    return nil
}
```
可以看到这里的主要处理方法与「理论篇」中的介绍基本一致：启动计时器，并立即调用 `pbftCore.sendPrePrepare` 发送 pre-prepare 消息。另外需要注意一下的是，这里还将 `RequestBatch` 请求写入了 `pbftCore.outstandingReqBatches` 这个字段。这个字段用来存储还未执行完的请求，我们在后面的分析中还会遇到这个字段。

到此，我们就完全分析完了请求的接收过程。总结起来就是，请求可能是 `Message_CHAIN_TRANSACTION` 类型的消息，也可能是 `Request` 对象，也可能是 `RequestBatch` 对象；但它们最终都会被转变成 `RequestBatch` 对象，由 `pbftCore.recvRequestBatch` 进行处理。主要的处理过程就是存储对象，启动计时器，并发送 pre-prepare 消息（如果自己是主结点）。


## pre-prepare

这一小节里，我们先介绍收到请求后发送 pre-prepare 的流程，然后再介绍收到 pre-prepare 消息时是如何进行处理的。


### 发送

接收到请求 `RequestBatch` 以后，主结点就要发送 pre-prepare 消息响应请求了。前面我们分析请求的处理时，最后发现会调用 `pbftCore.sendPrePrepare` 方法发送 pre-prepare 消息。下面我们就来看看具体的实现。

我们先来看实现的第一部分：
```go
func (instance *pbftCore) sendPrePrepare(reqBatch *RequestBatch, digest string) {
    n := instance.seqNo + 1
    // 检查是否存在 RequestBatch 的哈希相同、但 SequenceNumber 不同的 pre-prepare 消息
    for _, cert := range instance.certStore { 
        if p := cert.prePrepare; p != nil {
            if p.View == instance.view && p.SequenceNumber != n 
                && p.BatchDigest == digest && digest != "" {
                return
            }
        }
    }

    ......
}
```
这里首先要做的事就是为新的 pre-prepare 消息分配一个新的 Sequence Number，然后检查请求对象 `RequestBatch` 是否存在序号冲突（已经为其构造了 pre-prepare 消息、但与当前的序号值不同）。如果有冲突，说明发生了错误，就不会处理这个请求。

如果序号值没有冲突，则继续进行处理：
```go
func (instance *pbftCore) sendPrePrepare(reqBatch *RequestBatch, digest string) {
    ......

    if !instance.inWV(instance.view, n) || n > instance.h+instance.L/2 {
        // We don't have the necessary stable certificates to advance our watermarks
        logger.Warningf(...j)
        return
    }

    if n > instance.viewChangeSeqNo {
        logger.Info(...)
        return
    }

    ......
}
```
这里主要判断新的序号和域编号是否合规。在「理论篇」我们提到过，每个节点有一个区间 $[h, H]$，主要用来防止恶意主结点在构造 pre-prepare 消息时滥用序号值，这里的第一个 if 就是在判断新的 pre-prepare 消息的序号是否在 $[h, H]$ 内。（至于 $h$ 和 $H$ 是如何设置的，以及这里为什么是 `instance.L` 和 `instance.L / 2`，我们将在「 watermark 」小节中介绍）

第一个 if 判断新消息的序号是否超过了 `pbftCore.viewChangeSeqNo`。Fabric 的 pbft 与论文中不同的是，除了在异常情况下发起域转换（View Change），它还会设定一个序号间隔，即使一直没有发生异常，也会在每处理一定数量的请求后，主动发起域转换。`pbftCore.viewChangeSeqNo` 这个字段就是用来标记这种情况下应该发起域转换的序号值。关于这一点，我们在「view change」小节里再详细说明。

第二个 if 中如果新的序号超过 `pbftCore.viewChangeSeqNo`，即此时应该马上进行域转换了，所以暂不处理这个请求，直接返回了。你可能会想那这个请求什么时候处理呢？难道就这样丢弃了吗？应该不是的，但这里我没法给出一个确定的答案。由于此时还处于发送 pre-prepare 消息的阶段，所以只有这个主结点记录了这个请求信息，如果这个主结点此时直接返回，确实没发现有其它地方再次处理这个请求（虽然请求记录在了 `pbftCore.outstandingReqBatches` 中，但这个字段记录的请求是需要在域转换之后（`pbftCore.processNewView2`）由新主结点进行处理的。这种情况下当前的主结点已经不再是主结点了）。那么有可能处理这个消息的地方就是 `obcBatch.resubmitOutstandingReqs` 方法了，此方法用来在结点变成主结点后，处理之前接收到的未被处理的消息（具体参看「 outstanding 」小节）。如果这个猜测成立，那么 Fabric 项目中客户端发送请求的方式肯定是向所有结点广播的方式，而不是论文中所说的那样只向主结点发送请求。

如果刚才的检查都成功，那么继续进行处理，发送 pre-prepare 消息：
```go
func (instance *pbftCore) sendPrePrepare(reqBatch *RequestBatch, digest string) {
    ......

    instance.seqNo = n
    preprep := &PrePrepare{
        View:           instance.view,
        SequenceNumber: n,
        BatchDigest:    digest,
        RequestBatch:   reqBatch,
        ReplicaId:      instance.id,
    }
    cert := instance.getCert(instance.view, n)
    cert.prePrepare = preprep
    cert.digest = digest
    instance.persistQSet()
    instance.innerBroadcast(&Message{Payload: &Message_PrePrepare{PrePrepare: preprep}})
    instance.maybeSendCommit(digest, instance.view, n)
}
```
可以看到这里更新了结点的 Sequence Number，并构造了一个 `PrePrepare` 对象，然后调用 `pbftCore.innerBroadcast` 将其广播出去。紧接着调用 `pbftCore.maybeSendCommit` 方法来判断这个请求是否已于处理 prepared 状态，如果是，则继续发送 commit 消息。

可以看到，整个发送 pre-prepare 消息的过程，与我们在「理论篇」中描述得基本一致：分配序号；检查序号的正确性；构造 pre-prepare 消息并广播；检查这个请求是否处于 prepared 状态。不太一样的是加了一个每隔固定请求数量就会发起域转换，不管是不是有异常，此时有可能导致请求暂不会被处理。

**但是，这里并没有给 `PrePrepare` 对象签名。**在「理论篇」中，每一个消息都是经过签名的。为什么这里可以不用签名呢？如果没有签名，那不成了拜占庭将军问题论文中的「口头消息」了吗？

需要签名的原因主要是防止恶意结点假冒别人发送假的数据。而这里不需要签名的原因我想主要有三个：一是 pre-prepare 消息不需要转发，每个人都是从消息的发送者那里直接接收；二是 p2p 网络模块可以正确的识别出这个消息来自于哪个结点；三是对象内部有一个 `ReplicaId` 字段字录这条消息是由谁发送的。由于不存在转发 `PrePrepare` 对象的情况，所以正常情况下每个结点在发送此对象之前，都要将 `PrePrepare.ReplicaId` 字段设置成自己的 ID；其它结点收到 `PrePrepare` 对象以后，它可以通过 p2p 网络模块获得此对象的发送者的 ID，然后将其与接收到的 `PrePrepare.ReplicaId` 进行比较；如果相同，则说明消息是真实的，没有被篡改。如果某结点想要冒充别人发送 `PrePrepare` 对象，那么它需要将 `PrePrepare.ReplicaId` 设置成别人的 ID，但 p2p 网络中的 ID 是改不了的，所以别人收到消息后，发现两个 ID 不一致，那肯定是不对的。

在接收 pre-prepare 消息的方法 `pbftCore.recvMsg` 中，有一段这样的判断代码：
```go
func (instance *pbftCore) recvMsg(msg *Message, senderID uint64) (interface{}, error) {
    ......
    else if preprep := msg.GetPrePrepare(); preprep != nil {
        if senderID != preprep.ReplicaId {
            return nil, fmt.Errorf(
                "Sender ID included in pre-prepare message (%v) 
                doesn't match ID corresponding to the receiving stream (%v)", 
                preprep.ReplicaId, senderID)
        }
        return preprep, nil
    } else if ...
    ......
}
```
可以看到，这里会判断发送者的 ID `senderID` 是否与 `PrePrepare.ReplicaId` 一致。其实在这个方法里针对其它消息如 prepare 或 commit 消息进行接收时，也会进行类似的判断。

但是你可能会发现对于 ViewChange 消息是有签名的。这是为什么呢？我想这是因为 ViewChange 消息是需要转发的。在新的主结点发送 NewView 消息中，需要携带其它结点发送的 ViewChange 消息，这时刚才判断 ID 是否一致的方法就无效了。如果没有签名，新的主结点完全可以自己构造足够多的、假的 ViewChange 消息，然后生成 NewView 消息广播出去。

这里还要提一下上面代码中的 `pbftCore.getCert` 及其用到的字段 `pbftCore.certStore`。这个字段存储了所有最近处理的请求的信息（无论成功的或进行中的），它的类型为 `map[msgID]*msgCert`，其中 `msgID` 是由 View Number 和 Sequence Number 组成的，`msgCert` 的定义如下：
```go
type msgCert struct {
    digest      string
    prePrepare  *PrePrepare
    sentPrepare bool
    prepare     []*Prepare
    sentCommit  bool
    commit      []*Commit
}
```
可见 `msgCert` 中存储的正是 PBFT 共识三阶段的所有消息。`pbftCore.certStore` 字段和 `pbftCore.getCert` 立法会在后面的分析中多次被用到，因为每一阶段的消息都存储在它这里。


### 接收

刚才的内容都是发送 pre-prepare 的流程。那我们如何接收 pre-prepare 消息？接收到又是如何处理的呢？

其实 pre-prepare 消息是属于「pbft message」中的一种，它包含在 `pbftMessageEvent` 这个消息事件中：
```go
func (instance *pbftCore) ProcessEvent(e events.Event) events.Event {
    switch et := e.(type) {
    ......

    case pbftMessageEvent:
        next, err := instance.recvMsg(msg.msg, msg.sender)
        if err != nil {
            break
        }
        return next

    ......
    }
}

func (instance *pbftCore) recvMsg(msg *Message, senderID uint64) (interface{}, error) {
    ......

    else if preprep := msg.GetPrePrepare(); preprep != nil {
        if senderID != preprep.ReplicaId {
            return nil, fmt.Errorf(...)
        }
        return preprep, nil
    } else if ...

    ......
}
```
可以看到，在收到 `pbftMessageEvent` 消息事件以后调用 `pbftCore.recvMsg` 进行处理时，会判断这个消息是否是 pre-prepare 消息。如果是，则返回 `*PrePrepare` 类型的变量。并且这里也会判断一下发送者的 ID 和 pre-prepare 消息中的发送者 ID 是否一致。如果不一致，则不认为这是一个正常的消息（防止恶意结点冒充主结点发送 pre-prepare 消息）。

`PrePrepare` 对象的定义如下：
```go
type PrePrepare struct {
    View           uint64
    SequenceNumber uint64
    BatchDigest    string
    RequestBatch   *RequestBatch
    ReplicaId      uint64
}
```
基本与「理论篇」的介绍一致，只不过这里包含了请求对象本身，而论文中只携带了请求的哈希。

`pbftCore.recvMsg` 返回 `*PrePrepare` 类型的变量后，又会进入到 `pbftCore.ProcessEvent` 的处理中。这次进入的是 `*PrePrepare` 分支。这个分支直接调用 `pbftCore.recvPrePrepare` 处理接收到的 pre-prepare 消息，所以我们直接看一下这个方法的第一部分：
```go
func (instance *pbftCore) recvPrePrepare(preprep *PrePrepare) error {
    if !instance.activeView {
        return nil
    }

    // 自已当前的主结点 ID 与 pre-prepare 中发送者 ID 不同，则不处理
    if instance.primary(instance.view) != preprep.ReplicaId {
        return nil
    }

    // 如果 pre-prepare 中的域编号与自己当前的不一致，
    // 或者 pre-prepare 中的序号不在 [h, H] 区间内，
    // 则不处理
    if !instance.inWV(preprep.View, preprep.SequenceNumber) {
        if preprep.SequenceNumber != instance.h && !instance.skipInProgress {
            logger.Warningf(...)
        } else {
            // This is perfectly normal
            logger.Debugf(...)
        }

        return nil
    }

    // 如果马上要进行域转换了，也不处理当前消息
    if preprep.SequenceNumber > instance.viewChangeSeqNo {
        instance.sendViewChange()
        return nil
    }

    // 根据域编号和序号获取相应的共识信息，
    // 判断存在请求哈希一致、但序号和域编号不一致的情况。如果存在则发起域转换。
    cert := instance.getCert(preprep.View, preprep.SequenceNumber)
    if cert.digest != "" && cert.digest != preprep.BatchDigest {
        logger.Warningf("Pre-prepare found for same view/seqNo but different digest: ...", ...)
        instance.sendViewChange()
        return nil
    }
}
```
这一部分的代码比较简单，都是对 pre-prepare 一些字段进行检查，已经在代码中填加了注释进行说明，就不再多说了。

我们继续看下一部分的代码：
```go
func (instance *pbftCore) recvPrePrepare(preprep *PrePrepare) error {
    ......

    cert.prePrepare = preprep
    cert.digest = preprep.BatchDigest

    // Store the request batch if, 
    // for whatever reason, we haven't received it from an earlier broadcast
    if _, ok := instance.reqBatchStore[preprep.BatchDigest]; !ok && preprep.BatchDigest != "" {
        digest := hash(preprep.GetRequestBatch())
        if digest != preprep.BatchDigest {
            return nil
        }
        instance.reqBatchStore[digest] = preprep.GetRequestBatch()
        instance.outstandingReqBatches[digest] = preprep.GetRequestBatch()
        instance.persistRequestBatch(digest)
    }

    ......
}
```
这一段代码主要是将 pre-prepare 消息存储到 `msgCert` 结构（`pbftCore.certStore` 字段中的 value）中。如果自己没有保存请求数据，则现在保存一份。

最后一部分代码是最主要的部分：
```go
func (instance *pbftCore) recvPrePrepare(preprep *PrePrepare) error {
    ......

    instance.softStartTimer(instance.requestTimeout, 
    instance.nullRequestTimer.Stop()

    // if 判断：
    //   1. 自己不是主结点
    //   2. 我们已经接受了当前请求的 pre-prepare 消息
    //   3. 还未发送过 prepare 消息
    if instance.primary(instance.view) != instance.id && 
       instance.prePrepared(preprep.BatchDigest, preprep.View, preprep.SequenceNumber) && 
       !cert.sentPrepare {
        prep := &Prepare{
            View:           preprep.View,
            SequenceNumber: preprep.SequenceNumber,
            BatchDigest:    preprep.BatchDigest,
            ReplicaId:      instance.id,
        }
        cert.sentPrepare = true
        instance.persistQSet()
        instance.recvPrepare(prep)
        return instance.innerBroadcast(&Message{Payload: &Message_Prepare{Prepare: prep}})
    }

    return nil
```
这里首先启动计时器，然后在满足三个条件的情况下，构造 `Prepare` 对象，并将其广播出去。判断的条件分别为：自己不是主结点、已接受当前请求的 pre-prepared 消息、还没广播过 prepare 消息。

从这里也可以看出来，主结点是不发送 prepare 消息的（后面在分析接收到 prepare 消息的处理时，也会看到忽略主结点发送的 prepare 消息）。 但这并不代表主结点不参与共识的过程。至于为什么，我们会在「commit」小节里详细介绍。

至此，pre-prepare 消息从发送要接收处理的过程我们就分析完了。简单来说在收到 `RequestBatch` 对象后，就会构造 `PrePrepare` 对象并广播；在收到 `PrePrepare` 对象后，又会尝试构造 `Prepare` 对象并广播。整个过程中，会判断域编号和序号的合规性等问题。


## prepare

前面的小节里，我们分析到了收到 pre-prepare 消息后进行处理，并构造发送 prepare 消息的过程。这一小节我们就看一下 prepare 是如何接收和处理的。

在 pbft 模块中，使用 `Prepare` 对象代表 prepare 消息，它的定义如下：
```go
type Prepare struct {
    View           uint64
    SequenceNumber uint64
    BatchDigest    string
    ReplicaId      uint64
}
```
`Prepare` 中保存的信息与「理论篇」介绍的基本一致，包含域编号、请求序号、请求哈希、发送者 ID，并且没有签名数据（至于原因我们在 pre-prepare 消息的「发送」小节已经介绍过了）。

当有 prepare 消息时，会调用 `pbftCore.recvPrepare` 进行处理。这个方法有两个调用者：一是自己接收到 pre-prepare 消息并处理，然后发送 prepare 消息后，这在上一小节里已看过；另一个是从外界接收到 prepare 消息后，我们简单看一下这个情况。

与 pre-prepare 类似，外界来的 prepare 消息也被归类到 pbft message 中，在 `pbftCore.ProcessEvent` 中的 `pbftMessageEvent` 分支进行处理：
```go
func (instance *pbftCore) ProcessEvent(e events.Event) events.Event {
    switch et := e.(type) {
    ......

    case pbftMessageEvent:
        next, err := instance.recvMsg(msg.msg, msg.sender)
        if err != nil {
            break
        }
        return next

    ......
    }
}

func (instance *pbftCore) recvMsg(msg *Message, senderID uint64) (interface{}, error) {
    ......

    else if prep := msg.GetPrepare(); prep != nil {
        // 对发送者和 Prepare 中保存的 ID 进行比较。
        // 前面我们说过这是在无需签名的情况下确保数据真实性的重要手段
        if senderID != prep.ReplicaId {
            return nil, fmt.Errorf(...)
        }
        return prep, nil
    } else if ...

    ......
}
```
最终 `pbftCore.recvMsg` 返回了 `*Prepare` 对象，并继续由 `pbftCore.ProcessEvent` 的 `*Prepare` 分支处理。在这个分支中最终调用了 `pbftCore.recvPrepare` 方法。

下面我们看一下 `pbftCore.recvPrepare` 方法是如何处理 prepare 消息的。第一部分对 `Prepare` 对象中域编号和请求序号的检查与处理 `PrePrepare` 类似，这里就不再啰嗦了（但保留了第一个判断）：
```go
func (instance *pbftCore) recvPrepare(preprep *PrePrepare) error {
    // 如果是主结点发送的 prpare 消息，则忽略它
    if instance.primary(prep.View) == prep.ReplicaId {
        logger.Warningf("Replica %d received prepare from primary, ignoring", instance.id)
        return nil
    }
    ......

    cert := instance.getCert(prep.View, prep.SequenceNumber)

    for _, prevPrep := range cert.prepare {
        if prevPrep.ReplicaId == prep.ReplicaId {
            return nil
        }
    }
    cert.prepare = append(cert.prepare, prep)
    instance.persistPSet()

    return instance.maybeSendCommit(prep.BatchDigest, prep.View, prep.SequenceNumber)
}
```
方法的后半部分也比较简单，首先仍是从 `pbftCore.certStore` 中获取请求的共识相关信息；然后判断是否重复。最后将 `Prepare` 对象记录在 `msgCert.prepare` 中。最后调用 `pbftCore.maybeSendCommit`，看是否可以已经处于 prepared 状态，如果是则发送 commit 消息。

我们顺便看一下 `pbftCore.maybeSendCommit` 是如何实现的：
```go
func (instance *pbftCore) maybeSendCommit(digest string, v uint64, n uint64) error {
    cert := instance.getCert(v, n)
    if instance.prepared(digest, v, n) && !cert.sentCommit {
        commit := &Commit{
            View:           v,
            SequenceNumber: n,
            BatchDigest:    digest,
            ReplicaId:      instance.id,
        }
        cert.sentCommit = true
        instance.recvCommit(commit)
        return instance.innerBroadcast(&Message{&Message_Commit{commit}})
    }
    return nil
}
```
所有共识阶段的信息都存储在 `pbftCore.certStore` 字段中，所以这里再次从中取出相关信息，并调用 `pbftCore.prepared` 方法判断当前请求是否处于 prepared 状态。如果是则构造 commit 消息并广播。

这里我们主要关心一下 `pbftCore.prepared` 方法是如何判断请求处于 prepared 状态的：
```go
func (instance *pbftCore) prepared(digest string, v uint64, n uint64) bool {
    if !instance.prePrepared(digest, v, n) {
        return false
    }

    if p, ok := instance.pset[n]; ok && p.View == v && p.BatchDigest == digest {
        return true
    }

    quorum := 0
    cert := instance.certStore[msgID{v, n}]
    if cert == nil {
        return false
    }

    for _, p := range cert.prepare {
        if p.View == v && p.SequenceNumber == n && p.BatchDigest == digest {
            quorum++
        }
    }

    return quorum >= instance.intersectionQuorum()-1
}
```
此方法的逻辑还是很简单的，基本和「理论篇」中介绍的一致。首先这个请求得是 pre-prepared 状态，也就是接受了请求的 pre-prepare 消息；然后如果 `pbftCore.pset` 字段存在这个请求，也算是处于 prepared 状态了；最后就是从 `msgCert.prepare` 找出符合要求的 prepare 消息的数量，如果这个数量大于等于 `pbftCore.intersectionQuorum() - 1`，则认为处于 prepared 状态了。

这里最后数量的判断与「理论篇」中的要求是基本一致的。`pbftCore.intersectionQuorum` 方法最终返回值为 $2f+1$（$f$ 为「理论篇」中提到了系统能容忍的最大恶意结点数量），但又减去了 1，变成了 $2f$。也就是说最终 prepare 的数量大于等于 $2f$ 就可以了。而「理论篇」中介绍的需要大于等于 $2f+1$的，这是什么情况呢？刚才在 `pbftCore.recvPrepare` 的代码中，第一个判断的作用是忽略主结点发送的 prepare 消息。也就是在 Fabric 中的 pbft 实现中，主结点的 prepare 消息是被忽略的（前面介绍 pre-prepare 消息的处理时，也提到过忽略了主结点的消息），而「理论篇」中是包含主结点的，如果去掉主结点，数量要求也就变成了 $2f$，如果一来就一致了。

虽然这里忽略了主结点的 prepare 消息，但与在「pre-prepare」小节中提到的一样，这并不代表主结点不最终参与共识。相反 Fabric 的实现使用了一个取巧的方式，即所有结点默认主结点的 prepare 消息肯定是正确的。详细分析我们将放在「commit」小节里进行介绍。

这里还需要稍微提一下为什么请求在 `pbftCore.pset` 中也可以认为处于 prepared 状态了。这里的「pset」指的就是论文中 ViewChange 消息中的 $P$ 集合，这个字段在发送 ViewChange 消息时（`pbftCore.sendViewChange`），调用 `pbftCore.calcPSet` 进行填充。在 `pbftCore.calcPSet` 中其实主要也是计算 `pbftCore.certStore` 中符合要求的 prepare 的数量，与刚才 `pbftCore.prepared` 后半部分的代码是类似的。所以 `pbftCore.prepared` 中对其进行判断，应该算是一种「快捷」方式，因为如果已经存在就不用再一个一个遍历 `msgCert.prepare` 字段了。

总得来说，prepare 消息的处理与「理论篇」中介绍的也基本一致，在最开始（`pbftCore.recvMsg` 中）会判断消息是否真实；在处理过程中会判断消息中的域编号和请求序号是否合规；最后如果有 $2f$ 条合规则的 prepare 消息，则认为处于 prepared 状态，从而发送 commit 消息。


## commit

刚才我们提到如果处理 prepared 状态，则结点会发送 commit 消息。这一小节我们来看看 commit 消息是如何发送和处理的。

在 pbft 模块中，使用 `Commit` 对象代表 comit 消息，它的定义如下：
```go
type Commit struct {
    View           uint64
    SequenceNumber uint64
    BatchDigest    string
    ReplicaId      uint64
}
```
`Commit` 中保存的信息与「理论篇」介绍的基本一致，包含域编号、请求序号、请求哈希、发送者 ID，并且没有签名数据（至于原因我们在 pre-prepare 消息的「发送」小节已经介绍过了）。

当有 commit 消息时，会调用 `pbftCore.recvCommit` 进行处理。这个方法有两个调用者：一是自己接收到 prepare 消息并处理后，发现已处于 prepared 状态，这在上一小节里已看过；另一个是从外界接收到 commit 消息后，这一小节主要关注这种情况。

与 pre-prepare 类似，外界来的 commit 消息也被归类到 pbft message 中，在 `pbftCore.ProcessEvent` 中的 `pbftMessageEvent` 分支进行处理：
```go
func (instance *pbftCore) ProcessEvent(e events.Event) events.Event {
    switch et := e.(type) {
    ......

    case pbftMessageEvent:
        next, err := instance.recvMsg(msg.msg, msg.sender)
        if err != nil {
            break
        }
        return next

    ......
    }
}

func (instance *pbftCore) recvMsg(msg *Message, senderID uint64) (interface{}, error) {
    ......

    else if commit := msg.GetCommit(); commit != nil {
        if senderID != commit.ReplicaId {
            return nil, fmt.Errorf(...)
        }
        return commit, nil
    } else if ...

    ......
}
```
最终 `pbftCore.recvMsg` 返回了 `*Commit` 对象，并继续由 `pbftCore.ProcessEvent` 的 `*Commit` 分支处理。在这个分支中最终调用了 `pbftCore.recvCommit` 方法。

下面我们看一下 `pbftCore.recvCommit` 方法是如何处理 commit 消息的。第一部分的对 `Commit` 对象中域编号和请求序号的检查与处理 `PrePrepare` 类似，这里就不再啰嗦了：
```go
func (instance *pbftCore) recvCommit(commit *Commit) error {
    ......

    cert := instance.getCert(commit.View, commit.SequenceNumber)
    // 去掉重复
    for _, prevCommit := range cert.commit {
        if prevCommit.ReplicaId == commit.ReplicaId {
            return nil
        }
    }
    cert.commit = append(cert.commit, commit)

    if instance.committed(commit.BatchDigest, commit.View, commit.SequenceNumber) {
        instance.stopTimer()
        ......
        // 已经 commited，因此将从请从「未解决」的 outstandingReqBatches 中删掉
        delete(instance.outstandingReqBatches, commit.BatchDigest)

        instance.executeOutstanding()

        // 到了该进行域转换序号了
        if commit.SequenceNumber == instance.viewChangeSeqNo {
            instance.sendViewChange()
        }
    }

    return nil
}
```
这段代码并不复杂，首先是从 `pbftCore.certStore` 中获取此请求的共识过程信息，然后判断 commit 消息是否重复。最后如果 commit 消息有效，则将其加入到 commit 消息列表中，并马上判断是否处于 committed 状态。如果是，则将请求从代表「未解决」请求列表的 `pbftCore.outstandingReqBatches` 中删掉，然后调用 `pbftCore.executeOutstanding` 执行请求。

下面我们来看看判断是否处于 committed 状态的 `pbftCore.committed` 是如何实现的：
```go
func (instance *pbftCore) committed(digest string, v uint64, n uint64) bool {
    if !instance.prepared(digest, v, n) {
        return false
    }

    quorum := 0
    cert := instance.certStore[msgID{v, n}]
    if cert == nil {
        return false
    }

    for _, p := range cert.commit {
        if p.View == v && p.SequenceNumber == n {
            quorum++
        }
    }

    return quorum >= instance.intersectionQuorum()
}
```
首先请求必须已经处于 prepared 状态，然后后面的代码与 `pbftCore.prepared` 方法最后的代码类似，都是统计当前请求的有效的 commit 消息的数量，如果大于或等于 `pbftCore.intersectionQuorum` 方法的返回值（实际为 $2f+1$），则说明已处于 committed 状态。这与「理论篇」中介绍的是一样的。

### intersectionQuorum() or intersectionQuorum()-1

前面我们介绍过判断 prepared 状态满足的条件是 prepare 消息数量大于等于 `pbftCore.intersectionQuorum() - 1`，减 1 是因为主结点的 prepare 消息是被所有结点忽略的；而这里判断 committed 状态满足的条件却是 `pbftCore.intersectionQuorum()`。这是怎么回事呢？

在 Fabric 的实现中，所有结点默认主结点的 prepare 消息是有效的、正确的，因此在实现时都不需要主结点发送 prepare 消息，其它结点判断只要有 $2f$ 条（`pbftCore.intersectionQuorum() - 1`） prepare 消息，就认为达到 prepared 状态了。主结点自己也是这么做的，它在发送 pre-prepare 消息后，并不发送 prepare 消息。

当所有结点（包括主结点）收到 $2f$ 条 prepare 消息后，它判断处于 prepared 状态，并构造和发送 commit 消息。由于这一过程是由包括主结点在内的所有结点参与的，所以需要 $2f+1$ 条（`pbftCore.intersectionQuorum()`） commit 消息，才认为处于 committed 状态。

总得来说，Fabric 的这个实现默认了主结点的 prepare 消息总是正确的，继而直接省去了主结点发送 prepare 消息这一步。这是一种比较取巧的办法，但也很符合逻辑，我们下面简单分析一下。

我们假设不忽略主结点的 prepare 消息。然后分两种情况进行讨论：主结点是正常的，和主结点是恶意的。

如果主结点是正常结点，那么它会发送正确的 pre-prepare 消息，也会发送正确的 prepare 消息，共识可以正常达成。可以看到在主结点是正常的情况下，我们可以放心的忽略它的 prepare 消息，因为它总是正确的。

如果主结点是恶意结点，也存在两种情况：它发送的 pre-prepare 是错误的或正常的。如果它发送错误的 pre-prepare 消息，此时无论主结点的 prepare 消息正确与否，都不会达成共识；如果它发送正确的 pre-prepare 消息，此时也无论主结点的 prepare 消息正确与否，总能达成共识。可以看到在主结点是恶意的情况下，无论主结点发送的 prepare 消息正确与否，都不会影响最终结果的达成，所以我们也可以放心的忽略它的 prepare 消息。

所以总得来说，无论哪种情况，都可以放心地忽略主结点的 prepare 消息。


## 执行请求

如果某个请求处于 committed 状态，就可以执行这个请求了。在代码中也是如此实现的：
```go
func (instance *pbftCore) recvCommit(commit *Commit) error {
    ......

    if instance.committed(commit.BatchDigest, commit.View, commit.SequenceNumber) {
        ......
        instance.executeOutstanding()
        ......
    }

    return nil
}
```
可以看到每次收到 commit 消息以后，都会判断一下是否处于 committed 状态，如果是则调用 `pbftCore.executeOutstanding` 执行请求。我们看一下这个方法的实现：
```go
func (instance *pbftCore) executeOutstanding() {
    if instance.currentExec != nil {
        return
    }

    for idx := range instance.certStore {
        if instance.executeOne(idx) {
            break
        }
    }

    instance.startTimerIfOutstandingRequests()
}
```
这个方法很简单，一开始的 if 用来判断当前是否有请求在被执行，如果有就直接返回。`pbftCore.currentExec` 为当前正在执行的请求序号的变量指针，它会在 `pbftCore.executeOne` 中被设置，一旦被设置，就代表就请求正在被执行。

接下来一个 for 循环遍历 `pbftCore.certStore` 中保存的共识信息，对每一项调用 `pbftCore.executeOne` 尝试进行执行。从名字也能看出， `pbftCore.executeOne` 只会执行一个请求，如果执行成功就会返回 `true`，那么这个循环也就中断了。这么看来，调用一次 `pbftCore.Outstanding` 只会执行一个请求，并不会执行所有 committed 状态的请求。

下面我们看看 `pbftCore.executeOne` 是如何实现的。先来看第一部分的代码：
```go
func (instance *pbftCore) executeOne(idx msgID) bool {
    cert := instance.certStore[idx]

    if idx.n != instance.lastExec+1 || cert == nil || cert.prePrepare == nil {
        return false
    }

    if instance.skipInProgress {
        return false
    }

    // we now have the right sequence number that doesn't create holes

    digest := cert.digest
    reqBatch := instance.reqBatchStore[digest]

    if !instance.committed(digest, idx.v, idx.n) {
        return false
    }

    ......
}
```
这里第一个 if 判断中比较重要的是，当前要执行的请求的序号是否是最近一次执行的请求的序号加 1（instance.lastExec+1）。如果不是则返回 `false`。结合刚才调用 `pbftCore.executeOne` 的代码，这一块实现了按序号递增的方式执行请求，保证了请求的顺序。这样一来，即使序号较大的请求率先达成共识，也必须等序号较小的请求成达共识并执行完成后，才会被执行。

代码中第二个 if 判断 `pbftCore.skipInProgress` 的值。这个值在进行状态同步的时候会设置为 `true`，用来暂停请求的执行。

代码接下来判断要执行的请求是否处于 committed 状态。如果不是则返回 `false`，挑选下一个请求进行执行。

如果请求处于 committed 状态，则进行下面的处理：
```go
func (instance *pbftCore) executeOne(idx msgID) bool {
    ......

    // we have a commit certificate for this request batch
    currentExec := idx.n
    instance.currentExec = &currentExec

    // null request
    if digest == "" {
        instance.execDoneSync()
    } else {
        // synchronously execute, it is the other side's 
        // responsibility to execute in the background if needed
        instance.consumer.execute(idx.n, reqBatch)
    }
    return true
```
经过前面的判断，已经确定可以执行这个请求了。因此这部分代码主要就是对请求进行执行。这里又分两种情况，如果是一个空请求，就什么也不做，直接调用 `pbftCore.execDoneSync` 处理请求已执行完的情况；如果不是空请求，则调用 `pbftCore.consumer` 中的 `execute` 方法执行请求。其中 `pbftCore.consumer` 实际上指向的是 `obcBatch` 对象，因此这里实际上调用的是 `obcBatch.execute` 方法。

`obcBatch.execute` 的实现如下：
```go
func (op *obcBatch) execute(seqNo uint64, reqBatch *RequestBatch) {
    var txs []*pb.Transaction
    for _, req := range reqBatch.GetBatch() {
        tx := &pb.Transaction{}
        if err := proto.Unmarshal(req.Payload, tx); err != nil {
            continue
        }

        if outstanding, pending := op.reqStore.remove(req); !outstanding || !pending {
            logger.Debugf(...)
        }
        txs = append(txs, tx)
        op.deduplicator.Execute(req)
    }
    meta, _ := proto.Marshal(&Metadata{seqNo})
    // This executes in the background, 
    // we will receive an executedEvent once it completes
    op.stack.Execute(meta, txs) 
}
```
在这个方法中，主要是将请求转换成 `Transaction` 对象，最后调用 `Executor` 接口的 `Execute` 方法执行请求。其中 `Executor` 接口是由 Fabric 其它模块实现的。（我们前面说过 Fabric 是一个框架，业务逻辑需要企业自己实现。实现 `Execute` 接口应该就是业务逻辑之一）

至此一个请求算是被提交执行了，但还没完。根据上面代码的注释，其它模块执行完请求后还会调用 `ExecutionConsumer` 接口的 `Executed` 方法通知请求执行完成；实现这一接口的是 `externalEventReceiver.Executed` 方法（也即 `obcBatch.Executed`），它会将一个 `executedEvent` 消息事件插入消息队列中；最终的处理会进入到 `pbftCore.ProcessEvent` 方法中的 `execDoneEvent` 分支中，这个分支中目前我们关心的是它调用 `pbftCore.execDoneSync` 方法，所以我们来看一下这个方法：
```go
func (instance *pbftCore) execDoneSync() {
    if instance.currentExec != nil {
        instance.lastExec = *instance.currentExec
        if instance.lastExec%instance.K == 0 {
            instance.Checkpoint(instance.lastExec, instance.consumer.getState())
        }

    } else {
        // XXX This masks a bug, this should not be called when currentExec is nil
        logger.Warningf("Replica %d had execDoneSync called, flagging ourselves as out of date", instance.id)
        instance.skipInProgress = true
    }
    instance.currentExec = nil

    instance.executeOutstanding()
}
```

这个方法中首先更新了 `pbftCore.lastExec` 字段，并判断是否应该生成 checkpoint，如果是则调用 `pbftcore.Checkpoint` 生成。方法的最后清空 `pbftCore.currentExec`（代表目前没有请求被执行了），然后再次调用 `pbftCore.executeOutstanding` 方法。还记得我们分析请求的执行最开始就是从这个方法执行的吗？这里再次调用这个方法，就可以在一次只执行一个请求的情况下，将 `pbftCore.certStore` 中所有可以执行的请求都执行了。


## checkpoint

在「理论篇」中我们提过 checkpoint（检查点），主要是为了避免历史数据太多；另一方面移定的检查点也可以让其它结点信任并同步，从而在状态不一致的时候快速趋于一致。这一小节我们就聊聊 checkpoint。

在 Fabric 中，使用 `Checkpoint` 对象代表一个 checkpoint 消息，它的定义如下：
```go
type Checkpoint struct {
    SequenceNumber uint64
    ReplicaId      uint64
    Id             string
}
```
其中 `Id` 代表的是当前数据状态的 id（可能是状态的哈希）。与前面介绍的其它消息一样，`Checkpoint` 对象也没有签名信息。

checkpoint 的处理起源和前面分析的消息一样，也有两种：一是结点自己发现需要生成 checkpoint 了，主动生成 checkpoint 并广播；二是从别处接收到了 checkpoint 消息。另外 checkpoint 还会在发送 view change 消息时被用到。所以我们下面从发送、接收、应用三个方法看一下 Fabric 是如何实现 checkpoint 的。


### 主动生成 checkpoint

在「理论篇」中我们介绍过，每间隔一定数量的请求，就会生成一个 checkpoint。Fabric 的实现与我们介绍的一致。在这里，`pbftCore.K` 字段保存了生成 checkpoint 的间隔，它的值来自于 pbft 目录下的配置文件 `config.yaml` 中的 `general.K` 字段。

生成 checkpoint 的时机有两个：一个是 state 数据同步完成以后；另一个是某个请求执行完成以后。生成 checkpoint 的功能是由 `pbftCore.Checkpont` 完成的。

对于第一种情况，当 state 数据同步完成以后，会立即针对获取到的数据，调用 `pbftCore.Checkpoint` 生成一个 checkpoint，如下代码如示：
```go
func (instance *pbftCore) ProcessEvent(e events.Event) events.Event {
    switch et := e.(type) {
    ......
    case stateUpdatedEvent:
        update := et.chkpt
        ......
        instance.Checkpoint(update.seqNo, update.id)
    ......
    }
}
```
其中 `update` 变量代表了同步到的 checkpoint 数据，它包含了 checkpoint 的序号和数据 id。`Checkpoint` 对象是不能转发的（它没有签名数据），所以这里同步的数据并不包含其它结点中的 `Checkpoint` 对象。又因为数据是我们从其它结点中同步过来的而非自己一个一个执行请求得到的，所以肯定没有没有生成 checkpoint ，所以这里利用这两个数据，立即生成自己的 checkpoint。

对于第二种情况，每次执行完一个请求后，我们都要检查是不是应该生成 checkpoint 了。如果是则调用 `pbftCore.Checkpoint` 生成。代码如下：
```go
func (instance *pbftCore) execDoneSync() {
    if instance.currentExec != nil {
        instance.lastExec = *instance.currentExec
        if instance.lastExec%instance.K == 0 {
            instance.Checkpoint(instance.lastExec, instance.consumer.getState())
        }

    } 

    ......
}
```
可以看到这里判断「是不是应该生成 checkpoint 」的方法，就是看刚才执行成功的请求的序号是否是 `pbftCore.K` 的整数倍。

下面我们具体看看 `pbftCore.Checkpoint` 是如何生成 checkpoint 的：
```go
func (instance *pbftCore) Checkpoint(seqNo uint64, id []byte) {
    if seqNo%instance.K != 0 {
        logger.Errorf(...)
        return
    }

    idAsString := base64.StdEncoding.EncodeToString(id)

    chkpt := &Checkpoint{
        SequenceNumber: seqNo,
        ReplicaId:      instance.id,
        Id:             idAsString,
    }
    instance.chkpts[seqNo] = idAsString

    instance.persistCheckpoint(seqNo, id)
    instance.recvCheckpoint(chkpt)
    instance.innerBroadcast(&Message{Payload: &Message_Checkpoint{Checkpoint: chkpt}})
}
```
这个方法的实现非常简单。首先方法会检查一下是否应该生成 checkpoint （生成的序号是否是 `pbftCore.K` 的整数倍），如果检查通过才会继续。接下来就是构造一个 `Checkpoint` 对象，并把 checkpoint 的序号和对应的数据 ID 保存在 `pbftCore.chkpts` 中。最后方法调用 `pbftCore.recvCheckpoint` 处理这个 checkpoint；并调用 `pbftCore.innerBroadcast` 广播我们自己生成的 checkpoint。

可以看到主动生成 checkpoint 的逻辑非常简单，就是达到请求间隔时（`pbftCore.K`）构造一个 `Checkpoint` 并广播。但构造完成以后我们自己还要对这个 checkpoint 进行处理，这一进程与从别的结点接收到 checkpoint 消息的过程是一致的，因此我们在下一小节讨论 `pbftCore.recvCheckpoint` 方法。


### 接收 checkpoint

checkpoint 消息与前面提到的其它消息类型类似，也是被归类到「pbft message」中。消息的接收过程如下：
```go
func (instance *pbftCore) ProcessEvent(e events.Event) events.Event {
    switch et := e.(type) {
    ......

    case pbftMessageEvent:
        msg := et
        next, err := instance.recvMsg(msg.msg, msg.sender)
        if err != nil {
            break
        }
        return next

    ......

    case *Checkpoint:
        return instance.recvCheckpoint(et)

    ......
    }
}

func (instance *pbftCore) recvMsg(msg *Message, senderID uint64) (interface{}, error) {
    ......
    else if chkpt := msg.GetCheckpoint(); chkpt != nil {
        if senderID != chkpt.ReplicaId {
            return nil, fmt.Errorf(...)
        }
        return chkpt, nil
    } else if ...
    ......
}
```
这一过程与其它消息的接收过程类似，这里就不多说了。

下面我们重点看一下 `pbftCore.recvCheckpoint` 的实现，首先看第一部分的代码：
```go
func (instance *pbftCore) recvCheckpoint(chkpt *Checkpoint) events.Event {
    if instance.weakCheckpointSetOutOfRange(chkpt) {
        return nil
    }

    if !instance.inW(chkpt.SequenceNumber) {
        return nil
    }

    ......
}
```
方法的一开始是两个if 判断。第一个 if 判断结点自己是否已经落后太多，以至于新的 checkpoint 的序号大于我们目前 $[h, H]$ 区间的上限 $H$（即 `pbftCore.h + pbftCore.L`）；第二个判断新的 checkpoint 的序号是否在当前的域中。

这里第一个 if 调用的 `pbftCore.weakCheckpointSetOutOfRange` 的实现稍显复杂，我们来简单说一下。这个方法的作用就是判断我们是否已经落后了，如果是则设置会触发 state 数据的同步。我们先看它的第一部分实现：
```go
func (instance *pbftCore) weakCheckpointSetOutOfRange(chkpt *Checkpoint) bool {
    H := instance.h + instance.L

    if chkpt.SequenceNumber < H {
        delete(instance.hChkpts, chkpt.ReplicaId)
    } else {
        ......
    }

    return false
```
这个方法首先判断新的 checkpoint 中的序号是否超过我们目前的上限 $H$，如果没超过则将新的 checkpoint 的发送者信息从 `pbftCore.hChkpts` 中删除。`pbftCore.hChkpts` 这个字段记录了所有我们收集到的、序号超过我们上限 $H$ 的 checkpoint 信息。它的作用体现在上面省略的 else 代码中：
```go
func (instance *pbftCore) weakCheckpointSetOutOfRange(chkpt *Checkpoint) bool {
    ......
    else {
        instance.hChkpts[chkpt.ReplicaId] = chkpt.SequenceNumber

        if len(instance.hChkpts) >= instance.f+1 {
            // 把 pbftCore.hChkpts 中记录的序号全部移到 
            /// chkptSeqNumArray 中并将其从低到高排序
            chkptSeqNumArray := make([]uint64, len(instance.hChkpts))
            index := 0
            for replicaID, hChkpt := range instance.hChkpts {
                chkptSeqNumArray[index] = hChkpt
                index++
                if hChkpt < H {
                    delete(instance.hChkpts, replicaID)
                }
            }
            sort.Sort(sortableUint64Slice(chkptSeqNumArray))

            // 现在 chkptSeqNumArray 中的序号已由低到高排序。
            // 如果 chkptSeqNumArray 中的后 f+1 个序号中最小的值大于 H，
            // 则可以说明 chkptSeqNumArray 中至少有 f+1 个序号是大于 H 的。
            if m := chkptSeqNumArray[len(chkptSeqNumArray)-(instance.f+1)]; m > H {
                // 清空一些数据，准备同步 state 数据
                instance.reqBatchStore = make(map[string]*RequestBatch) 
                instance.persistDelAllRequestBatches()
                instance.moveWatermarks(m)
                instance.outstandingReqBatches = make(map[string]*RequestBatch)
                instance.skipInProgress = true
                instance.consumer.invalidateState()
                instance.stopTimer()

                return true
            }
        }
    }

    return false
}
```
这段代码看一去很复杂，实际上很简单。如果 `pbftCore.hChkpts` 的记录的数量大于 $f+1$，则说明有多于 `f+1` 个结点都向我们发送了序号超过我们上限的 checkpoint，因此我们有理由认为自己落后了，需要去主动同步数据。

但代码并没有马上同步，而是又筛查了一遍是否有 $f+1$ 个结点的 checkpoint 的序号大于 $H$。这是因为如果上一次走过这个流程，`pbftCore.hChkpts` 中的记录数量已大于 $f+1$，但在数据同步之前并没有被清空。等数据同步完以后再次进入到这个流程中时，`pbftCore.hChkpts` 是仍然会保留着原来那些多数据，其数量仍然多于 $f+1$ 个。但此时由于已经同步过数据，里面记录的序号未必仍会大于 $H$。所以需要再做一次筛查。筛查的方法是将这些序号排序，然后看后 $f+1$ 个序号中的最小值是否大于 $H$，如果是，则说明我们仍然处于落后的状态。

（不过我觉得这段代码真够别扭的）

下面我们继续回到 `pbftCore.recvCheckpoint` 方法，看看在两个 if 判断成功以后，是如何处理的：
```go
func (instance *pbftCore) recvCheckpoint(chkpt *Checkpoint) events.Event {
    ......

    instance.checkpointStore[*chkpt] = true

    // Track how many different checkpoint values we have for the seqNo in question
    diffValues := make(map[string]struct{})
    diffValues[chkpt.Id] = struct{}{}

    // 从 pbftCore.checkpointStore 中
    //   找出所有序号相同且数据 ID 相同的 checkpoint 的数量，
    // 同时找出所有序号相同但数据 ID 不同的 ID。
    matching := 0
    for testChkpt := range instance.checkpointStore {
        if testChkpt.SequenceNumber == chkpt.SequenceNumber {
            if testChkpt.Id == chkpt.Id {
                matching++
            } else {
                if _, ok := diffValues[testChkpt.Id]; !ok {
                    diffValues[testChkpt.Id] = struct{}{}
                }
            }
        }
    }

    // If f+2 different values have been observed, we'll never be able to get a stable cert for this seqNo
    if count := len(diffValues); count > instance.f+1 {
        logger.Panicf(...)
    }

    ......
}
```
这段代码的主要作用有两个：一是设置 `matching` 变量；二是设置 `diffValues` 变量。`matching` 变量代表当前保存的 checkpoint 消息中，与新接收到的 checkpoint 中的 Sequence Number 和 ID 都相同的 checkpoint 的数量。`diffValues` 则代表序号相同但 ID 不同的 checkpoint 信息。如果 `diffValues` 的长度大于 $f+1$，也即序号相同、但 state 数据的 ID 不同的 checkpoint 个数大于 $f+1$，说明结点间的状态并不一致，这是一个严重的错误，因此立即调用 panic 函数停止运行。

如果通过了对 `diffValues` 的检查，下面就要检查 `matching` 的值了，代码如下：
```go
func (instance *pbftCore) recvCheckpoint(chkpt *Checkpoint) events.Event {
    ......

    // 如果有 f+1 个结点的 checkpoint 的序号和 ID 是相同的，
    // 但自己产生的、相同序号的 checkpoint 却有不同的 ID，
    // 那么肯定是自己搞错了。
    if matching == instance.f+1 {
        if ownChkptID, ok := instance.chkpts[chkpt.SequenceNumber]; ok {
            if ownChkptID != chkpt.Id {
                logger.Panicf(...)
            }
        }
        instance.witnessCheckpointWeakCert(chkpt)
    }

    if matching < instance.intersectionQuorum() {
        // We do not have a quorum yet
        return nil
    }

    ......
}
```
这里的两个判断也很好理解，第一个判断自己产生的 checkpoint 是否与其它 $f+1$ 个结点一致。因为已经超过了 $f$ 个结点有相同的结果，所以如果自己与这个结果不一致，肯定是自己错了，所以调用 panic 停止运行；但如果一致，则调用 `pbftCore.witnessCheckpointWeakCert` **在已经是同步 state 数据的状态时**，再次重新同步 state 数据，因为既然有 $f+1$ 个结点生成了相同的 checkpoint，基本可以认为这是一个稳定检查点，既然目前已经在同步数据了，那么就更新一下同步目标，使其可以更新到目前最新的状态。第二个判断看是否有超过 `pbftCore.intersectionQuorum()`（$2f+1$）个结点产生了相同的 checkpoint，如果是则可以认为这是一个「稳定检查点」了。

当获得一个稳定检查点时，`pbftCore.recvCheckpoint` 运行下面的代码，作最后的处理：
```go
func (instance *pbftCore) recvCheckpoint(chkpt *Checkpoint) events.Event {
    ......

    // 已经认证了一个稳定检查点，但如果自己还没生成这样一个检查点也没关系，
    // 随着自己不断处理请求，正常也会生成这样一个 Checkpoint 的。
    if _, ok := instance.chkpts[chkpt.SequenceNumber]; !ok {
        if instance.skipInProgress {
            logSafetyBound := instance.h + instance.L/2
            if chkpt.SequenceNumber >= logSafetyBound {
                instance.moveWatermarks(chkpt.SequenceNumber)
            }
        }
        return nil
    }

    instance.moveWatermarks(chkpt.SequenceNumber)

    return instance.processNewView()
}
```
最后这段代码中，首先处理自己没有生成这个稳定检查点的情况。前面的代码已经确认过 `chkpt` 变量代表了一个稳定检查点。但如果我们自己还没生成，说明我们暂时落后了，不过这里并不十分关心，因为我们是有可能暂时落后，后面可能会及时赶上来，并生成相同的 `Checkpoint` 对象的。

但如果我们落后了且已经在同步 state 数据了，那么作为一个小优化，在稳定检查点的序号大于自己的 $[h, H]$ 区间的中间值时，提高自己的 $h$ 为稳定检查点的序号。这样可以在 state 数据同步完以后，及时发现还有更新的数据，并立即再次同步。

这里的逻辑是这样的：如果最新的检查点已经大于我们 $[h, H]$ 区间的中间值，说明我们落后得比较远了，所以调用 `pbftCore.moveWatermarks` 将 `pbftCore.h` 设置为最新的稳定检查点的序号。注意我们当前的状态是在同步 state 数据，所以更新完 `pbftCore.h` 后且此次数据更新完成后，会检查检查最新的数据是不是小于 `pbftCore.h` ，如果是则继续更新，这段代码如下：
```go
func (instance *pbftCore) ProcessEvent(e events.Event) events.Event {
    switch et := e.(type) {
    ......
    // 收到 state 同步完成事件
    case stateUpdatedEvent:
        update := et.chkpt
        // update.seqNo < instance.h，即刚同步完的序号
        // 依然小于 pbftCore.h
        if et.target == nil || update.seqNo < instance.h {
            ......
            else if update.seqNo < instance.highStateTarget.seqNo {
                instance.retryStateTransfer(nil)
            } else ......

            return nil
        }
        ......

    ......
    }
}
```

在 `pbftCore.recvCheckpoint` 的最后，既然我们已经得到了一个新的稳定检查点，因此调用 `pbftCore.moveWatermarks` 更新 `pbftCore.h` 为稳定检查点的序号。

总得来说，在收到新的 `Checkpoint` 对象时，一方面我们利用它和之前收到的 `Checkpoint` 对象，检查自己是否落后；另一方面，在确认这是一个稳定检查点以后，更新自己区间 $[h, H]$ 中的 $h$ 值。

这里你可能会有一个疑问，在验证通过 $2f+1$ 条 `Checkpoint` 消息确认稳定检查点后，代码并没有对这 $2f+1$ 条稳定检查点的「证明」作特殊的处理：即没有把它们特别地存起来，也没有通知谁。而在发送 View Change 消息时，我们知道是需要用到稳定检查点的证明的。这是因为 Fabric 的实现使用了变通的方法，之前我们就说过，`Checkpoint` 对象没有签名信息，是不能被转发的；如果 View Change 的实现如「理论篇」中说的那样，直接带上 $2f+1$ 条 `Checkpoint` 对象作为稳定检查点的证明，那就相当于转发 `Checkpoint` 对象了。至于 View Change 消息是如何实现的，我们在下一小节进行讨论。


### 使用 checkpoint

在 Fabric 的实现中，checkpoint 的作用主要有两个：一是用来在同步 state 数据时作为一个「里程碑」，确认数据同步的终点或阶段；二是如「理论篇」中提到的那样，在进行域转换时提供稳定检查点，以辅助完成域转换过程。下面我们分别介绍一下这两种情况下是如何使用 checkpoint 的，重点介绍第二种情况。

对于同步 state 的情况，我们在上一小节里已经基本介绍了。在收到新的 `Checkpoint` 对象时，会作两个检查。一是利用新收到的 checkpoint 消息，可以发现自己是否落后于其它结点，从而及时发起 state 数据同步；二是如果最新的检查点是（或可能是）稳定检查点，且自己目前正处在同步数据的状态，那么就更新一下同步目标，使我们可以及时同步到最新的状态。详细请参看上一小节，这里不再重复解释了。

现在我们重点关注一下 checkpoint 是如何在域转换时提供稳定检查点证明的。

首先，在发送 view change 消息时，pbft 将所有自己生成的 checkpoint 加入到了 view change 对象中，如下代码所示：
```go
func (instance *pbftCore) sendViewChange() events.Event {
    ......

    vc := &ViewChange{
        View:      instance.view,
        H:         instance.h,
        ReplicaId: instance.id,
    }

    for n, id := range instance.chkpts {
        vc.Cset = append(vc.Cset, &ViewChange_C{
            SequenceNumber: n,
            Id:             id,
        })
    }

    ......
}
```
可以看到，这段代码首先构造了一个 `ViewChange`  对象，然后将所有自己生成的 checkpoint 信息填充到 `ViewChange.Cset` 字段中。这里的 `Cset` 即是论文中 $\langle VIEW$-$CHANGE\rangle$ 消息中的集合 $C$ 。（需要提醒一下的是，`pbftCore.chkpts` 中保存的只有自己生成的 checkpoint 对象，没有从其它结点中接收的对象。从其它结点接收到的存储在 `pbftCore.checkpointStore` 中）

你会发现这里并没有像「理论篇」中介绍的那样，将 $2f+1$ 条 checkpoint 消息放入到 `Cset` 中，而只是将自己生成的 checkpoint 对象放入。这样如何能达到证明稳定检查点的目的呢？那么我们需要看看收到 view-change 消息时是如何处理的。

这里我们不深入讨论收到 view-change 以后的处理流程，总之就是正常情况下，最终新的主结点会发送 new-view 消息，其它节点也会接收到 new-view 消息并进行处理。在发送和处理 new-view 消息时，就会使用 view-change 中的 checkpoint 信息：
```go
func (instance *pbftCore) sendNewView() events.Event {
    ......

    cp, ok, _ := instance.selectInitialCheckpoint(vset)

    ......
}

func (instance *pbftCore) processNewView() events.Event {
    ......

    cp, ok, replicas := instance.selectInitialCheckpoint(nv.Vset)

    ......
}
```
可以看到，无论是发送 new-view 还是接收处理 new-view，都会调用 `pbftCore.selectInitialCheckpoint` 从所有 view-change 消息中（即代码中的 `vset` 或 `nv.Vset`）选择一个合适的、稳定的检查点。

稳定的检查点我们理解，那么这里「适合的」是指什么呢？当然是稳定检查点的序号越高越合适啦。下面我们看看 `pbftCore.selectInitialCheckpoint` 方法的第一部分实现：
```go
func (instance *pbftCore) selectInitialCheckpoint(vset []*ViewChange) (checkpoint ViewChange_C, ok bool, replicas []uint64) {
    checkpoints := make(map[ViewChange_C][]*ViewChange)
    for _, vc := range vset {
        for _, c := range vc.Cset { 
            checkpoints[*c] = append(checkpoints[*c], vc)
        }
    }

    if len(checkpoints) == 0 {
        return
    }

    ......
}
```
这里首先构造了一个 `checkpoints` 变量，它是一个 `map` 类型，从所有 view-change 消息中提取所有 checkpoint 信息，将某个 checkpoint 信息对应着所有使用这个 checkpont 的 view-change，如下图所示：
![](/pic/fabric-pbft/checkpoints-map.png)

构造完了 `checkpoints` 变量以后，我们再看后面的处理：
```go
func (instance *pbftCore) selectInitialCheckpoint(vset []*ViewChange) (checkpoint ViewChange_C, ok bool, replicas []uint64) {
    ......

    // idx 为 checkpoint 信息，vcList 为包含了 idx 的所有 view change 列表
    for idx, vcList := range checkpoints {
        // 某个 checkpoint 信息必须被多于 f 个 view change 包含才可使用
        if len(vcList) <= instance.f {
            continue
        }

        // vc.H 代表了发送 view change 时结点的 [h, H] 区间中的 h.
        quorum := 0
        for _, vc := range vset {
            if vc.H <= idx.SequenceNumber {
                quorum++
            }
        }

        if quorum < instance.intersectionQuorum() {
            logger.Debugf("Replica %d has no quorum for n:%d", instance.id, idx.SequenceNumber)
            continue
        }

        // 现在 idx 代表的 checkpoint 信息可以认为是稳定的，
        // 但我们要找出序号最大的那个。
        replicas = make([]uint64, len(vcList))
        for i, vc := range vcList {
            replicas[i] = vc.ReplicaId
        }

        if checkpoint.SequenceNumber <= idx.SequenceNumber {
            checkpoint = idx
            ok = true
        }
    }

    return
}
```

这段代码有点长，我们一块一块解释。整体上来说，这里是一个 for 循环，目的是选择出 `checkpoints` 中是稳定检查点、且序号值最高的那个 checkpoint。判断是否稳定检查点的地方就是 for 循环最开始的 if 语句，这里查看包含这个 checkpoint 的 `ViewChange` 对象的数量，如果超过 $f$ 个，则可以认为是稳定的（弱条件）。判断「适合的」是 for 循环最后的 if 语句，这里通过遍历 `checkponts` 中所有元素，找出序号最大的检查点，作为最终的返回结果。

正规来说，for 循环的第一个判断应该去判断是否大于等于 $2f+1$，而不是大于 $f$，论文中就是这么介绍的。这里这么做的原因，我觉得是由于 Fabric 在发送 `ViewChange` 时，`Cset` 中只包含自己创建的 checkpoint 对象。这可能产生一种情况，就是部分稍稍落后的结点还没有创建比较新的 checkpoint，此时发生了域转换，在创建新的 checkpoint 之前，构造并发送了 `ViewChange` 对象。因此这里不需要严格的规定必须是 $2f+1$。当然如果使用 $2f+1$ 也没有问题，只不过进行域转换时使用 checkpoint 序号较低，会使域转换的过程稍慢一些。

另外在这个 for 循环的中间一段代码，还加了一个处理和判断。这段代码要求检查点的序号，必须不小于 $2f+1$ 个 `ViewChange` 对象的 `H` 字段。其中 `ViewChange.H` 是结点创建 `ViewChange` 对象时 $[h, H]$ 区间的 $h$ 值。这里的逻辑有两方面，一方面是如果检查点的序号比 $h$ 小，说明这个检查点过时了，比较旧，所以不采用；另一方面，结合 for 循环第一个 if 判断，既然不要求 $2f+1$，但至少要保证这个 checkpoint 是绝大多数（$2f+1$个）结点中比较新的，所以有了 checkpoint 的序号不能小于 `viewChange.H` 这个要求。

所以总得来看，Fabric 的实现中不允许对 `Checkpoint` 转发，所以在构造 `ViewChange` 时，就只能使用自己曾经创造的 `Checkpoint` 对象，而不能使用从网络中接收的他人的对象，但这也包含了足够的信息，可以从这些 `Checkpoint` 中找出最新的稳定检查点。所以在对 `NewView` 对象进行处理时，就是将这些 `Checkpoint` 进行汇总，找出重复数量超过 $f$ 个、在超过 $2f+1$ 个结点中都是比较新的、序号最高的那个 `Checkpoint`，作为新域的稳定检查点。


## 计时器与超时

在整个 PBFT 算法过程中，计时器和超时也是不可或缺的一项功能，它让 PBFT 算法及时发现异常并进行处理。在「理论篇」中，我们介绍了有两个重要的计时器：一个是三阶段共识时的计时器；一个是域转换时的计时器。在三阶段共识时，如果结点在收到请求或 pre-prepare 后，超过了预定时间请求仍没有达成 committed 状态，则认为共识失败，发起域转换；在域转换过程中，如果发送 view-change 消息后，在预定时间内域没有转换成功，则需要再次发起域转换，直到转换成功。

但「理论篇」中对这两个计时器的认识还是比较冷统的。在 Fabric 的实现中也使用了两个重要的计时器，但是从稍微不同的角度区分的：一个是超时以后，要发起一个新的域转换；另一个是超时以后，要重新发送 view-change 消息。注意这两种情况是不同的，发起一个新的域转换需要将编编号加 1，然后发送 view-change 消息；而重新发送 view-change 消息不会改变域编号，只是将刚才发送过的 view-change 消息再发送一次而已。Fabric 中的这种计时器的功能设置更加合理和可行（毕竟是实现而不只是理论）。

在详细分析这两类计时器的实现之前，我们要先简单了解一下 pbft 模块中 Timer 对象的实现。在 pbft 模块中，计时器的实现没有使用 go 语言中公共库中的对象，而是利用公共库的 Timer 对象，自己封装实现的。具体的代码位于 consensus/util/events/events.go 文件中，这里的实现将 Timer 与 pbft 中的消息循环系统集成在了一起。也就是说，当某个 pbft 的 Timer 对象超过，就会向消息循环系统中插入一些事件。这些事件就跟我们前面看到的所有事件一样，消息循环系统对它们同等对待。具体的实现比较简单，我们就不细说了。

下面我们就来主要分析详细分析一下这两类计时器的实现。另外 pbft 模块中还有其它几个计时器，我们也会简单介绍一下。

### 发起域转换的计时器

在 pbft 的实现中，超时后会发起域转换的计时器对象是 `pbftCore.newViewTimer`。这时我们不看代码中的实现，先自己想一下，有些哪些情况需要发起域转换呢？我觉得最容易想到的就是在三阶段共识过程中，请求没能及时达到 committed 状态。其实还有一种情况就是某结点自己发现已经收到了 $2f+1$ 条 view-change 消息，达到了域转换的条件，但迟迟没有收到新主结点的 new-view 消息。这说明新主结点出了问题，因此要再次发起域转换，跳过这个新的主结点。

下面我们就分别来看一下这两种情况。首先要说明的是，`pbftCore.newViewTimer` 的启动与停止是通过 `pbftCore.softStartTimer` 、 `pbftCore.startTimer` 和 `pbftCore.stopTimer` 三个方法实现的，因此后面的代码片段中不会直接出现操作 `pbftCore.newViewTimer` 对象的代码，而是调用这三个方法。其中 `pbftCore.softStartTimer` 与 `pbftCore.startTimer` 在区别在于，前者在 timer 已经在计时时，不会重置计时器；而后者无论 timer 是否在计时，都会进行重置。

我们先来看看三阶段共识过程中 `pbftCore.newViewTimer` 的设置：
```go
func (instance *pbftCore) recvRequestBatch(reqBatch *RequestBatch) error {
    ......
    if instance.activeView {
        instance.softStartTimer(
            instance.requestTimeout, 
            fmt.Sprintf("new request batch %s", digest))
    }
    ......
}

func (instance *pbftCore) recvPrePrepare(preprep *PrePrepare) error {
    ......
    instance.softStartTimer(
        instance.requestTimeout, 
        fmt.Sprintf("new pre-prepare for request batch %s", preprep.BatchDigest))
    ......
}
```
在收到请求时，`pbftCore.recvRequestBatch` 会被调用；在收到 pre-prepare 消息时，`pbftCore.recvPrePrepare` 会被调用。在这两个方法中，都会调用 `pbftCore.softStartTimer` 启动 `pbftCore.newViewTimer`。

最后，直到请求变成了 committed 状态，`pbftCore.newViewTimer` 才会被停止：
```go
func (instance *pbftCore) recvCommit(commit *Commit) error {
    ......
    if instance.committed(commit.BatchDigest, commit.View, commit.SequenceNumber) {
        instance.stopTimer()
        ......
    }
    ......
}
```

如果在这一过程中，最终请求没能达到 committed 状态，那么 `pbftCore.newViewTimer` 肯定会超时。它超时后，会向消息循环中插入 `viewChangeTimerEvent` 事件。以下是此事件的处理代码：
```go
func (instance *pbftCore) ProcessEvent(e events.Event) events.Event {
    switch et := e.(type) {
    case viewChangeTimerEvent:
        instance.timerActive = false
        instance.sendViewChange()
    ......
    }
}
```
此事件的处理非常简单，就是调用 `pbftCore.sendViewChange` 构造并发送 view-change 消息。由此开启了域转换的过程。

以上过程还是比较简单易懂的，不过还是要稍微解释一下为什么在收到 `RequestBatch` 和 pre-prepare 消息后，都会启到一次计时器。这是因为对于主结点来说，三阶段共识从接收到请求就开始了；而对于其它从结点来说，三阶段共识是从接收到 pre-prepare 消息才开始的。因此这两类结点对计时器的启动时机是不同的，只有主结点会收到并处理 `RequestBatch` ，其它结点实际上是从收到 pre-prepare 消息的 `pbftCore.recvPrePrepare` 方法开始这一过程中。

上面我们分析的是三阶段的主要过程中，对 `pbftCore.newViewTimer` 的启停操作。还有一种情况会启动这个计时器，也需要我们重点关注一下，那就是收到了足够多有效的 view-change 消息、却没收到新主结点发送的 new-view 消息的情况。

设想我们现在接收到了 $2f+1$ 条 view-change 消息，并全部验证通知了。因此我们知道域转换成功了，现在就等新的主结点发送 new-view 消息了。这时我们也要设置一个计时器，如果超时没有收到新主结点的 new-view 消息，说明新主结点有问题，因此要发起新的域转换，跳过当前的主结点：
```go
func (instance *pbftCore) recvViewChange(vc *ViewChange) events.Event {
    ......

    if !instance.activeView && 
       vc.View == instance.view && 
       quorum >= instance.allCorrectReplicasQuorum() {
        ......
        instance.startTimer(instance.lastNewViewTimeout, "new view change")
        instance.lastNewViewTimeout = 2 * instance.lastNewViewTimeout
        return viewChangeQuorumEvent{}
    }
    return nil
}
```
这里要解释一下 `pbftCore.lastNewViewTimeout` 会翻倍的情况。在「理论篇」中我们也提到过，如果没能在规定时间内域转换成功、收到 new-view 消息，那么进行新一轮的域转换时，会将计时器的时间设置得长一些（如 $2T$）。这里 `pbftCore.lastNewViewTimeout` 乘以 2 的处理，就是对应的实现。这主要是为了防止网络问题导致的消息收发过慢的情况。

如果正常收到了 new-view 消息，当然要及时停止这里设置的计时器啦：
```go
func (instance *pbftCore) processNewView2(nv *NewView) events.Event {
    instance.stopTimer()

    ......
}
```
`pbftCore.processNewView2` 处在 new-view 消息处理的后期，是基本确定 new-view 消息的有效性并将要更新到新的域中。所以这里第一件事就是停止 `pbftCore.newViewTimer`。

除了上面提到的情况，其实还有一些流程，会启动或停止这个计时器，下面我们简单看一下。

其它启动 `pbftCore.newViewTimer` 的情况：

- 情况1：自己不是主结点的情况下收到请求

我们再来想一下这样一个情景：自己不是主结点，但也收到了请求消息。正常情况下我们肯定不会去处理这个消息（只有主结点才去处理）。但如果主结点是恶意结点且不响应这一消息呢？此时如果我们不作任何处理，我们就不知道实际主结点不对消息进行响应，也就无法发起域转换了。

在「理论篇」中我们也讨论过这种情况，即在主结点不影响的情况下，客户端会发送请求给所有结点；之后从结点会等待主结点的 pre-prepare 消息，如果超时没等到，就要发起域转换。所以任何一个结点，如果它收到了请求，即使自己不是主结点，也要设置计时器，以便在预定时间内没收到 pre-prepare 消息时，发起域转换。

这一功能是由 `obcBatch.startTimerIfOutstandingRequests` 方法完成的。在 `obcBatch.reqStore` 字段中，有两个队列存储着两种请求：一种是自己是主结点的情况下，此时接收到的请示是该由自己处理的，这种请求存储在 「pending」 队列中；另一种是无论自己是不是主结点，只要收到的请求就存储起来，这种请求存储在 「outstanding」队列中：
```go
func (op *obcBatch) processMessage(ocMsg *pb.Message, senderHandle *pb.PeerID) events.Event {
    if ocMsg.Type == pb.Message_CHAIN_TRANSACTION {
        ......
        return op.submitToLeader(req)
    }

    ......

    if req := batchMsg.GetRequest(); req != nil {
        // 首先存储到 outstanding 中
        op.reqStore.storeOutstanding(req)
        if (op.pbft.primary(op.pbft.view) == op.pbft.id) && op.pbft.activeView {
            // 如果是主结点，才调用 leaderProcReq
            return op.leaderProcReq(req)
        }
        op.startTimerIfOutstandingRequests()
        return nil
    }
    ......
}

func (op *obcBatch) submitToLeader(req *Request) events.Event {
    ......

    // 首先存储到 outstanding 中
    op.reqStore.storeOutstanding(req)
    op.startTimerIfOutstandingRequests()
    if op.pbft.primary(op.pbft.view) == op.pbft.id && op.pbft.activeView {
        // 如果是主结点，才调用 leaderProcReq
        return op.leaderProcReq(req)
    }
}
```
可以看到，无论是收到了 Transaction 后调用 `obcBatch.submitToLeader` ，还是收到 `Request` 的处理，都是首先将请求存储到 outstanding 队列中，然后如果自己是主结点，才调用 `obcBatch.leaderProcReq` 处理请求。但无论如何，都会在将请求放入 outstanding 队列后紧接着调用 `obcBatch.startTimerIfOutstandingRequests`，启动 `pbftCore.newViewTimer` 。

```go
func (op *obcBatch) startTimerIfOutstandingRequests() {
    ......

    // 判断是否存在非 pending 的请求，
    // 也就是不该自己处理的请求
    if !op.reqStore.hasNonPending() {
        // Only start a timer if we are aware of outstanding requests
        return
    }
    op.pbft.softStartTimer(op.pbft.requestTimeout, "Batch outstanding requests")
}
```

其实 `obcBatch.startTimerIfOutstandingRequests` 被调用的地方不止上面提到的两处，当域转换成功时（`viewChangedEvent` 事件），或接收到 commit 消息后，或执行完某请求后，都会调用这个方法。这里的共同逻辑是，这些事件发生后，有可能 `pbftCore.newViewTimer` 是停止的状态，但 outstanding 队列中仍然可能会有一些消息等待被处理。尤其是 `viewChangedEvent` 事件后，结点需要判断自己是不是变成了主结点，如果是则需要处理所有的 outstanding 状态的请求：
```go
func (op *obcBatch) ProcessEvent(event events.Event) events.Event {
    switch et := event.(type) {
    ......
    case execDoneEvent:
        if res := op.pbft.ProcessEvent(event); res != nil {
            return res
        }
        return op.resubmitOutstandingReqs()
    case *Commit:
        res := op.pbft.ProcessEvent(event)
        op.startTimerIfOutstandingRequests()
        return res
    case viewChangedEvent:
        ......
        return op.resubmitOutstandingReqs()
    }
}

func (op *obcBatch) resubmitOutstandingReqs() events.Event {
    op.startTimerIfOutstandingRequests()

    // 如果自己是主结点，对 outstanding 中的消息进行处理
    if op.pbft.primary(op.pbft.view) == op.pbft.id && 
        op.pbft.activeView && op.pbft.currentExec == nil {
        ......
    }
}
```

所以总结一下，在收到请求时，需要对那些不该由自己处理的请求进行计时，如果发现超时后这些消息还没有被处理，说明当前的主结点有问题，要立刻发起域转换。启动计时的时机有：收到新请求时、收到并处理完「请求执行完成事件」时、收到并处理完 commit 消息时、收到并处理完「域转换完成事件」时。其中后三种情况的逻辑是，此时计时器可能是被停止的，但仍可能有不该由自己处理的消息存在、仍有对其计时的需求。


- 情况2：某请求已处于 committed 状态并被执行，但还有其它未处于 committed 状态的请求

刚才我们提到，请求变成了 committed 状态， `pbftCore.neViewTimer` 会被停止。但如果此时还有其它请求正在处理之中，还没有达到 committed 状态呢？

在 `pbftCore` 对象中，当收到请求或 pre-prepare 时，会将请求存储到 `pbftCore.outstandingReqBatches` 字段中。然后在请求达到 committed 状态后，再次请求从这个字段中删除：
```go
func (instance *pbftCore) recvRequestBatch(reqBatch *RequestBatch) error {
    ......
    instance.outstandingReqBatches[digest] = reqBatch
    ......
}

func (instance *pbftCore) recvPrePrepare(preprep *PrePrepare) error {
    ......
    if _, ok := instance.reqBatchStore[preprep.BatchDigest]; !ok && preprep.BatchDigest != "" {
        ......
        instance.outstandingReqBatches[digest] = preprep.GetRequestBatch()
    }
    ......
}

func (instance *pbftCore) recvCommit(commit *Commit) error {
    if instance.committed(commit.BatchDigest, commit.View, commit.SequenceNumber) {
        // 停止 pbftCore.newViewTimer 计时器
        instance.stopTimer()
        ......
        // 请求处于 committed 状态，将其从 pbftCore.outstandingReqBatches 中删除，
        // 并执行请求
        delete(instance.outstandingReqBatches, commit.BatchDigest)
        instance.executeOutstanding()
    }
}
```
可以看到，这个字段就是用来保存接收到了、但还未处于 committed 状态的请求。

如上代码所示，当某个请求处于 committed 状态时，会首先停止 `pbftCore.newViewTimer` 计时器，然后调用 `pbftCore.executeOutstanding` 执行这个请求。注意此时计时器是停止的，但如果执行完这个请求以后，还有其它请求未达到 committed 状态，而计时器又处于停止状态，那就无法在那些未处于 committed 状态的请求超时以后，及时发起域转换。所以执行完一个请求后，如果还有未达到 committed 状态的请求，肯定还是要重新启动计时器的：
```go
func (instance *pbftCore) executeOutstanding() {
    ......
    instance.startTimerIfOutstandingRequests()
}

func (instance *pbftCore) startTimerIfOutstandingRequests() {
    ......
    if len(instance.outstandingReqBatches) > 0 {
        ......
        // 重新启动 pbftCore.newViewTimer
        instance.softStartTimer(......)
    }
}
```


以上是其它启动 `pbftCore.newViewTimer` 的情况。下面我们再看看其它停止 `pbftCore.newViewTimer` 的情况：

- 情况1：发现自己落后、需要进行数据同步

在发现自己落后、需要进行数据同步时，显示不再需要对当前处理的消息进行计时了，代码如下：
```go
func (instance *pbftCore) weakCheckpointSetOutOfRange(chkpt *Checkpoint) bool {   
    H := instance.h + instance.L

    if chkpt.SequenceNumber < H {
        ......
        else {
            ......

            if m := chkptSeqNumArray[len(chkptSeqNumArray)-(instance.f+1)]; m > H {
                ......
                instance.consumer.invalidateState()
                instance.stopTimer()
            }
        }
    }
}
```
这段代码的逻辑是，如果有 $f+1$ 个结点生成的 checkpoint 的序号都超过我们的区间最大值 $H$，那么可以认为我们落后了。因此要开始同步数据，停止计时器。

- 情况2：开始发送 viewchange 消息

有一个细节在「理论篇」中我们也提到过，就是在发送 view-change 消息时，停止接收其它消息。显然在这个时间，请求的处理超时也没必要关心了：
```go
func (instance *pbftCore) sendViewChange() events.Event {
    instance.stopTimer()

    ......
}
```


### 重新发送 view-change 的计时器

在发送 view-change 消息之后，我们要设置一个计时器，监控是否在预定时间内收到了多于 $2f+1$ 条 view-change 消息。这个计时器的功能是由 `pbftCore.vcResendTimer` 实现的：
```go
func (instance *pbftCore) sendViewChange() events.Event {
    ......
    instance.vcResendTimer.Reset(instance.vcResendTimeout, viewChangeResendTimerEvent{})
    ......k
}
```
注意这个计时器关联的事件是 `viewChangeResendTimerEvent`。

如果在预定时间内收到了多于 $2f+1$ 条 view-change 消息，则要停止这个计时器，然后等待 new-view 消息：
```go
func (instance *pbftCore) recvViewChange(vc *ViewChange) events.Event {
    ......
    if !instance.activeView && 
       vc.View == instance.view && 
       quorum >= instance.allCorrectReplicasQuorum() {
        instance.vcResendTimer.Stop()
        ......
    }
```

如果这个计时器超时了，就会将 `viewChangeResendTimerEvent` 事件加入到消息循环中。在这个事件的处理中，会重新发送一遍 view-change 消息：
```go
func (instance *pbftCore) ProcessEvent(e events.Event) events.Event {
    switch et := e.(type) {
    ......

    case viewChangeResendTimerEvent:
        ......
        instance.view-- // sending the view change increments this
        return instance.sendViewChange()

    ......
    }
}k
```
注意这里在调用 `pbftCore.sendViewChange` 方法重新发送 view-change 消息时，会先将 view 编号减 1。这是因为目前是「重新发送」view-change 的状态，也就是说之前已经发送过一次了，那么域编号已经加过 1 了。现在要重新发送，需要先将域编号减 1，然后随后调用的 `pbftCore.sendViewChange` 会将域编号加 1，以完成「重新发送」的目的。


### 其它计时器

在 pbft 模块中还有几个计时器不如前面介绍的重要，但也值得一说。这一小节我们就介绍一下这些计时器。

第一个要介绍的是 `obcBatch.batchTimer` 。前面我们提到过，在 Fabric 中各结点可以接收 Transaction 、 `Request` 、 `RequestBatch` 三种对象作为请求，但最终 pbft 内部都会将其转换为 `RequestBatch` 进行处理。所谓的 `RequestBatch` 其实就是多个 `Request` 。所以在 pbft 中，当收到 Transaction 或 `Request` 类请求时，并不马上提交处理，而是「攒」在一起，等达到一定数量再组成一个 `RequestBatch` 提交处理。但也有可能「攒」了好久都达不到预定的数量，因此还需要一种机制，在「攒」了一定时间之后，不管是否达到预定数量，都会提交处理。 `obcBatch.batchTimer` 就是用来实现这一功能的。

在 `obcBatch` 对象中，`obcBatch.batchTimer` 的启停是由 `obcBatch.startBatchTimer` 和 `obcBatch.stopBatchTimer` 两个方法来完成的：
```go
func (op *obcBatch) startBatchTimer() {
    op.batchTimer.Reset(op.batchTimeout, batchTimerEvent{})
    op.batchTimerActive = true
}

func (op *obcBatch) stopBatchTimer() {
    op.batchTimer.Stop()
    op.batchTimerActive = false
}
```

当自己作为主结点收到 Transaction 或 `Request` 时，就会启动这个计时器：
```go
func (op *obcBatch) leaderProcReq(req *Request) events.Event {
    ......
    if !op.batchTimerActive {
        op.startBatchTimer()
    }
    ......
}
```
其中 `obcBatch.leaderProcReq` 的调用时机是自己是主结点且要把请求加入处理队列等待处理时。

如果 `obcBatch.batchTimer` 超时，它会将 `batchTimerEvent` 事件加入到消息循环中。此事件的处理方式，就是调用 `obcBatch.sendBatch` 提交请求：
```go
func (op *obcBatch) ProcessEvent(event events.Event) events.Event {
    switch et := event.(type) {
    ......
    case batchTimerEvent:
        if op.pbft.activeView && (len(op.batchStore) > 0) {
            return op.sendBatch()
        }
    ......
    }
}
```

停止 `obcBatch.batchTimer` 的时机是域转换成功（`viewChangedEvent`） 和 `obcBatch.sendBatch` 中将要提交请求。刚才说过只有当自己是主结点并且要将这些请求加入队列等待处理时，才会启动这个计时器；那么域转换成功以后，自己就不再是主结点了，因此也就没必要再提交这些请求了。至于 `obcBatch.sendBatch` 中停止此计时器是非常好理解的。停止的代码如下：
```go
func (op *obcBatch) ProcessEvent(event events.Event) events.Event {
    switch et := event.(type) {
    ......
    case viewChangedEvent:
        ......
        if op.batchTimerActive {
            op.stopBatchTimer()
        }
        ......
    ......
    }
}

func (op *obcBatch) sendBatch() events.Event {
    op.stopBatchTimer()

    ......
}
```

第二个要介绍的计时器是 `pbftCore.nullRequestTimer` 。这个计时器主要被主结点用来实现类似于网络联接中的「keep alive」的功能，即每隔一定时间发送一段消息，证明主结点还是活跃的。这个 timer 的超时时间由 consensus/pbft/config.yaml 文件中的「general.timeout.nullrequest」字段设置，如果为 0，则可以禁用这个功能。

每当调用 `pbftCore.startTimerIfOutstandingRequests` 时，如果发现没有请求可以处理了，就会启动这个 timer (配置为可用的情况下)：
```go
func (instance *pbftCore) startTimerIfOutstandingRequests() {
    if len(instance.outstandingReqBatches) > 0 {
        // 有请求等待处理
        ......
    } else if instance.nullRequestTimeout > 0 {
        // 无请求等待处理，且超时时间配置非 0
        timeout := instance.nullRequestTimeout
        // 由于主结点在超时后才会发送 null request，
        // 所以给主结点多一点时间，多等它一会。
        if instance.primary(instance.view) != instance.id {
            timeout += instance.requestTimeout
        }
        instance.nullRequestTimer.Reset(timeout, nullRequestEvent{})
    }
}
```

`pbftCore.nullRequestTimer` 超时后，会将 `nullRequestEvent` 事件加入到消息循环中。在这个事件的处理中，如果自己是主结点，则会发送一个空请求；如果自己是从结点，说明主结点一直没有发送请求（包括空请求和非空请求），因此认为主结点出现了问题，所以调用 `pbftCore.sendViewChange` 发起域转换：
```go
func (instance *pbftCore) ProcessEvent(e events.Event) events.Event {
    switch et := e.(type) {
    ......
    case nullRequestEvent:
        instance.nullRequestHandler()
    ......
    }
}

func (instance *pbftCore) nullRequestHandler() {
    if !instance.activeView {
        return
    }

    if instance.primary(instance.view) != instance.id {
        // 从结点，发送 view-change
        instance.sendViewChange()
    } else {
        // 主结点，发送 null request
        instance.sendPrePrepare(nil, "")
    }
}
```

如果在超时之前，主结点收到了正常的请求，那么正常情况下主结点肯定会将这个请求的 pre-prepare 消息发给其它从结点，因此也就不用特意发个空请求证明自己了，所以主结点在收到请求时，就将这个计时器停止了；作为一个从结点如果在超时之前收到了主结点的 pre-prepare 消息，那么不管此消息是否包含的是空请求，都代表主结点是活跃的，所以此时也将这个计时器停止。这两个停止逻辑的代码如下：
```go
func (instance *pbftCore) recvRequestBatch(reqBatch *RequestBatch) error {
    ......

    if instance.primary(instance.view) == instance.id && instance.activeView {
        // 主结点收到请求，停止 nullRequestTimer
        instance.nullRequestTimer.Stop()
        ......
    }
    ......
}

func (instance *pbftCore) recvPrePrepare(preprep *PrePrepare) error {
    ......
    // 从结点收到了从主结点发来的 pre-prepare，
    // 也停止 nullRequestTimer
    instance.nullRequestTimer.Stop()
    ......
}
```

另外一个停止时机是在 `pbftCore.processNewView2` 中。此时马上要域转换成功了，如果要等待空请求，也应该等待新的主结点的空请求，而不再去等待旧主结点的：
```go
func (instance *pbftCore) processNewView2(nv *NewView) events.Event {
    ......
    instance.nullRequestTimer.Stop()
    ......
}
```


## view change

这一小节时，我们详细分析一下 pbft 模块中关于域转换的实现。我们将按照域转换流程，先介绍发起域转换的时机；然后依次介绍 view-change 的构造和发送、new-view 的构造和发送。


### 发起时机

现在我们来看看在 pbft 模块中，在什么情况下会发起域转换。总得来说，只要是发现有异常，就会发起域转换，因为在 PBFT 算法中，域转换是保证活性和纠正错误的唯一手段。总结 pbft 模块中发起域转换的时机，可以有以下几种：
- 三阶段共识过程中
  - pre-prepare 消息冲突：收到的 pre-prepare 消息中的域编号和请求序号相同，但请求哈希不同
  - 三阶段共识处理超时，触发 `viewChangeTimerEvent` 事件
- 域转换过程中
  - 发送 view-change 消息后，未能在预定时间内收到多于 $2f+1$ 个其它结点发送的 view-change 消息，触发 `viewChangeResendTimerEvent` 事件
  - 接收到 view-change 消息后，发现有多于 $f$ 个结点发送的 view-change 消息中的域编号各不相同且都大于自己当前的域编号，因而挑选里面最小的域编号作为自己的域编号，并发送 view-change 消息
  - 接收到 new-view 消息后，但从消息中找不出合规的 checkpoint 集合
  - 收到 new-view 消息后，从消息中计算出的重新进行共识处理的集合为空（`pbftCore.assignSequenceNumbers` 返回 nil）
  - 收到 new-ivew 消白后，从消息中计算出的重新进行共识处理的集合（调用 `pbftCore.assignSequenceNumbers`  获得）与 new-view 消息中的 Xset（即「理论篇」中 new-view 消息中的集合 $O$） 不相同
- 其它情况
  - 预定时间内未收到主结点的 pre-prepare 消息（包括空请求）
  - 收到的 pre-prepare 消息中的请求序号大于 `pbftCore.viewChangeSeqNo` （即应该发起域转换了）
  - 收到 commit 消息时，发现请求处于 committed 状态，且执行请求提交后，发现 commit.SequenceNumber == instance.viewChangeSeqNo （即执行完这一请求，就应该发起域转换了）

下面我们详细看一下 pbft 模块中域转换的过程。


### 构造发送 view-change 消息

构造并发送 view-change 消息这一过程是在 `pbftCore.sendViewChange` 中完成的。下面我们一部分一部分地看一下这个方法：
```go
func (instance *pbftCore) sendViewChange() events.Event {
    // 停止 pbftCore.newViewTimer 计时器
    // 此计时器在超时时会调用 pbftCore.sendViewChange
    instance.stopTimer()

    // 将当前的 new-view 消息从 pbftCore.newViewStore 中删除
    // 此字段记录了自己发送的或接收的、最新的、但还没处理完的 new-view 消息。
    // 在接收到 new-view 消息时，用这个字段判断相同域编号的 new-view 消息是否已经在处理中
    delete(instance.newViewStore, instance.view)
    // 增加 view 域号
    instance.view++
    instance.activeView = false

    ......
}
```
第一部分的代码可以理解成是做准备工作，比较简单就不多介绍了。

下面是接下来的一部分代码：
```go
func (instance *pbftCore) sendViewChange() events.Event {
    ......

    instance.pset = instance.calcPSet()
    instance.qset = instance.calcQSet()

    ......
}
```
这两行代码用来计算两个集合：pset 和 qset。其中 pset 包含了当前结点中最新的 checkpoint 之后的所有 prepared 状态的请求的信息；qset 包含了当前结点中最新的 checkpoint 之后的所有 pre-prepare 消息的信息。这两个集合在「理论篇」中没有明确的对应，若非要对应起来，「理论篇」中 view-change 消息中的集合 $P$ 应该包含了这里的 pset 和 qset，它包含的内容仍然是不相同的。这一点我们在介绍完 pset 和 qset 的计算方法再来说。

下面是计算 pset 和 qset 的代码：
```go
func (instance *pbftCore) calcPSet() map[uint64]*ViewChange_PQ {
    pset := make(map[uint64]*ViewChange_PQ)

    for n, p := range instance.pset {
        pset[n] = p
    }

    for idx, cert := range instance.certStore {
        if cert.prePrepare == nil {
            continue
        }

        digest := cert.digest
        if !instance.prepared(digest, idx.v, idx.n) {
            continue
        }

        if p, ok := pset[idx.n]; ok && p.View > idx.v {
            continue
        }

        pset[idx.n] = &ViewChange_PQ{
            SequenceNumber: idx.n,
            BatchDigest:    digest,
            View:           idx.v,
        }
    }

    return pset
}

func (instance *pbftCore) calcQSet() map[qidx]*ViewChange_PQ {
    qset := make(map[qidx]*ViewChange_PQ)

    for n, q := range instance.qset {
        qset[n] = q
    }

    for idx, cert := range instance.certStore {
        if cert.prePrepare == nil {
            continue
        }

        digest := cert.digest
        if !instance.prePrepared(digest, idx.v, idx.n) {
            continue
        }

        qi := qidx{digest, idx.n}
        if q, ok := qset[qi]; ok && q.View > idx.v {
            continue
        }

        qset[qi] = &ViewChange_PQ{
            SequenceNumber: idx.n,
            BatchDigest:    digest,
            View:           idx.v,
        }
    }

    return qset
}
```
这两个方法的代码是类似的，一开始都是生成一个新的集合对象，并将 `pbftCore.pset` 或 `pbftCore.qset` 中的数据加入到新创建的集合对象中。我想这一是因为在结点启动时会从数据库中 restore 加载以前的 pset 和 qset；二是因为上次的 view-change 消息可能发送失败了，需要重发。无论哪种情况，此时的 `pbftCore.pset` 和 `pbftCore.qset` 中都已经存储了一些集合数据，因此需要将它们加入到当前的集合中。

这两个方法下面的代码也很类似，对于 `pbftCore.calcPSet` 来说，要选取 `pbftCore.certStore` 中处于 prepared 状态的请求加入到 pset 中；对于 `pbftCore.calcQSet` 来说，是选取 `pbftCore.certStore` 中所有 pre-prepare 消息的信息加入到 qset 中。注意 `pbftCore.certStore` 中只存储上次生成 checkpoint 以后的信息，所以这里 pset 和 qset 中的信息都是「最新的 checkpoint 之后的」信息。

刚才我们提至过，虽然 pset 和 qset 是 view-change 消息的组成部分，但它与「理论篇」中介绍的集合 $P$ 还是不一样的。由于「理论篇」中，所有消息都是有签名的（因此可以转发收到的消息），所以集合 $P$ 中直接包含了所有针对某个请求的、其它结点发送的 pre-prepare 和 prepare 消息。而 Fabric 的实现中，pre-prepare 和 prepare 消息都是没有签名的，因而在发送 view-change 时不能直接携带这些消息（携带了也不可信）。所以这里 pset 和 qset 只能携带结点自己的信息（即自己记录的达到 prepared 状态的请求信息和 pre-prepare 信息）。

下面我们继续看 `pbftCore.sendViewChange` 的代码：
```go
func (instance *pbftCore) sendViewChange() events.Event {
    ......

    for idx := range instance.certStore {
        if idx.v < instance.view {
            delete(instance.certStore, idx)
        }
    }
    for idx := range instance.viewChangeStore {
        if idx.v < instance.view {
            delete(instance.viewChangeStore, idx)
        }
    }

    ......
}
```
这段代码特别简单，由于我们已经将域编号加 1 了，并且也从 `pbftCore.certStore` 中计算出了 pset 和 qset，因此这里将 `pbftCore.certStore` 和 `pbftCore.viewChangeStore` 中旧的域中的信息删除。其中 `pbftCore.viewChangeStore` 用来记录本次域转换过程中接收到的 view-change 消息，后面很多逻辑都要基于这个字段做判断，因此在发起域转换时需要将旧的 view-change 消息删除。（但我不太明白为什么要删除 `pbftCore.certStore` 中的旧信息）

我们继续看后面的代码：
```go
func (instance *pbftCore) sendViewChange() events.Event {
    ......
    vc := &ViewChange{
        View:      instance.view,
        H:         instance.h,
        ReplicaId: instance.id,
    }

    for n, id := range instance.chkpts {
        vc.Cset = append(vc.Cset, &ViewChange_C{
            SequenceNumber: n,
            Id:             id,
        })
    }

    for _, p := range instance.pset {
        if p.SequenceNumber < instance.h {
            logger.Errorf("BUG! ......")
        }
        vc.Pset = append(vc.Pset, p)
    }

    for _, q := range instance.qset {
        if q.SequenceNumber < instance.h {
            logger.Errorf("BUG! ......")
        }
        vc.Qset = append(vc.Qset, q)
    }

    instance.sign(vc)
}
```
这段代码就是构造 `ViewChange` 对象，并将前面得到的 pset 和 qset 分别拷贝到 `ViewChange.Pset` 和 `ViewChange.Qset` 中。最后对 `ViewChange` 对象签名（是的， view-change 消息是有签名的）。

`ViewChange.Cset` 这个字段在「使用 checkpoint」小节中已介绍过，这里再简单提一下。这个字段大体上与「理论篇」中 view-change 消息中的集合 $C$ 相对应。之所以说「大体上」，还是因为 checkpoint 消息在 Fabric 的实现中是没有签名的，而「理论篇」中是有签名的，因此这个字段只能携带自己曾经创建过的 checkpoint 信息（`pbftCore.chkpts` 只保存了自己创建的 checkpoint 消息，没有从别处接收的消息）。

我们继续看最后一部分代码：
```go
func (instance *pbftCore) sendViewChange() events.Event {
    ......

    instance.innerBroadcast(&Message{Payload: &Message_ViewChange{ViewChange: vc}})

    instance.vcResendTimer.Reset(instance.vcResendTimeout, viewChangeResendTimerEvent{})

    return instance.recvViewChange(vc)
}
```
最后也是比较简单的：将刚才构造的 `ViewChange` 对象广播，然后启动 `pbftCore.vcResendTimer` 计时器（此计时器超时后会重新发送 view-change 消息，且域编号和刚才发送的 view-change 消息一样）。最后调用 `pbftCore.recvViewChange` ，即自己率先处理自己广播的 view-change 消息。

至此发送 view-change 消息的动作就算完成了。可以看到发送 view-change 并不复杂，稍微复杂一点的地方就是理解和计算 pset 和 qset。


### 接收处理 view-change 消息

下面我们来看看在收到 view-change 消息是如何处理的。不过为了完整性，我们先看看在消息循环中是如何接收到 view-change 消息的。

与共识三阶段的消息类似，view-change 消息也被归到 `pbftMessage` 这一大类消息中：
```go
func (instance *pbftCore) ProcessEvent(e events.Event) events.Event {
    switch et := e.(type) {
    ......
    case pbftMessageEvent:
        msg := et
        next, err := instance.recvMsg(msg.msg, msg.sender)
        if err != nil {
        break
        }
        return next     
    case *ViewChange:
        return instance.recvViewChange(et)
        ......
    ......
    }
}

func (instance *pbftCore) recvMsg(msg *Message, senderID uint64) (interface{}, error) {
    ......
    else if vc := msg.GetViewChange(); vc != nil {
        if senderID != vc.ReplicaId {
            return nil, fmt.Errorf(...)
        }
        return vc, nil
    }
    ......
}
```
可见最后 `pbftCore.recvMsg` 返回了 `*ViewChange` 对象，并最终调用了 `pbftCore.recvViewChnage` 进行处理。

下面我们来看看 `pbftCore.recvViewChange` 是如何处理 view-change 消息的。先看第一部分代码：
```go
func (instance *pbftCore) recvViewChange(vc *ViewChange) events.Event {
    if err := instance.verify(vc); err != nil {
        logger.Warningf(...)
        return nil
    }

    if vc.View < instance.view {
        logger.Warningf(...j)
        return nil
    }

    if !instance.correctViewChange(vc) {
        logger.Warningf(...)
        return nil
    }

    if _, ok := instance.viewChangeStore[vcidx{vc.View, vc.ReplicaId}]; ok {
        logger.Warningf(...)
        return nil
    }

    ......
}
```
第一部分代码是对新收到的 view-change 消息的有效性的判断。其中 `pbftCore.verify` 用来验证 view-change 消息是否有效；第二个 if 用来判断新的 view-change 中的域编号是否比结点自己的大；`pbftcore.correctViewChange` 用来判断 view-change 消息中的一些数据是否正确，包含 pset 和 qset 中的域编号是否比 view-change 中新的域编号小、pset 和 qset 中的请求序号是否位于合理区间内、cset 中的数据是否正确，代码比较简单，就不一一说明了。最后一个 if 用来判断是否已接收过同一结点发送的域编号相同的 view-change 消息。

我们继续看后面的代码：
```go
func (instance *pbftCore) recvViewChange(vc *ViewChange) events.Event {
    ......

    instance.viewChangeStore[vcidx{vc.View, vc.ReplicaId}] = vc

    // PBFT TOCS 4.5.2 Liveness: ...... 
    // 统计总共收到了多少个结点发送了
    // 域编号比结点自己的大的 view-change 消息
    replicas := make(map[uint64]bool)
    minView := uint64(0)
    for idx := range instance.viewChangeStore {
        if idx.v <= instance.view {
            continue
        }

        replicas[idx.id] = true
        if minView == 0 || idx.v < minView {
            minView = idx.v
        }
    }

    if len(replicas) >= instance.f+1 {
        logger.Infof(...)
        // subtract one, because sendViewChange() increments
        instance.view = minView - 1
        return instance.sendViewChange()
    }

    ......
}
```
这段代码的第一行很简单，就是存储这个 view-change 消息。下面的两段代码实现了论文中提到的算法内容（论文的 4.5.2 小节，代码的注释写错了），这一内容我们在「理论篇」中也有提到（「安全性（Safety）和活性（Liveness）」小节中的「第二个细节...」），即一个结点如果收到了多于 $f+1$ 个其它结点发送的 view-change 消息，且这些消息的域编号都比自己当前的大，那么就立即从这些消息中选一个最小的域编号，发送 view-change 消息。这样可以节省域转换的时间。这里的两段代码就是实现这个内容的。

还需要说明的是，在调用 `pbftCore.recvViewChange` 接收新的 view-change 消息时，结点可能还处在旧的域中。所以很自然的，接收到的 view-change 消息的域编号在正常情况下，都会比自己当前的域编号大。

这会产生两种情况：一种情况是在接收到 $f+1$ 条域编号比自己大的 view-change 之前，上面的最后一个 if 分支不会执行，而是直接执行后面的（此处被省略的）代码；另一种情况是累计收到 $f+1$ 条域编号比自己大的 view-change 之后，就会执行上面的代码段的最后的 if 分支，选择一个最小的域编号并发送 view-change 消息，因而当再次接收到 view-change 消息时，也不会再进入这里最后的 if 分支，而是直接执行后面的（此处被省略的）代码。

所以可以得出，后面的被省略的代码可能会在两种情况下执行：一是结点并不曾发送过 view-change 消息， `pbftCore.view` 还是旧的域编号；二是结点已送过 view-change 消息， `pbftCore.view` 已被设置为新的域编号。我们来看下这段代码：
```go
func (instance *pbftCore) recvViewChange(vc *ViewChange) events.Event {
    ......

    quorum := 0
    for idx := range instance.viewChangeStore {
        if idx.v == instance.view {
            quorum++
        }
    }

    if !instance.activeView && 
       vc.View == instance.view && 
       quorum >= instance.allCorrectReplicasQuorum() {
        instance.vcResendTimer.Stop()
        instance.startTimer(instance.lastNewViewTimeout, "new view change")
        instance.lastNewViewTimeout = 2 * instance.lastNewViewTimeout
        return viewChangeQuorumEvent{}
    }

    return nil
}
```

这段代码主要是统计有多少 view-chnage 消息的域编号与结点自己的相同；如果此数量达到要求（`pbftCore.allCorrectReplicasQuorum()`，即 $2f+1$），则返回 `viewChangeQuorumEvent` 对象。

你可能会疑惑为什么会判断 view-change 中的域编号与结点当前自己的域编号是否相等呢？正常情况下 view-change 的域编号不应该比结点当前的小吗？

前面我们解释过，执行这段代码时有可能有两种情况：一种是结点的域编号还是旧的，因此 `pbftCore.viewChangeStore` 中存储的 view-change 消息中的域编号正常情况下肯定比结点自己的大，所以此时第一个 for 循环中的统计值肯定无法满足大于 `pbftCore.allCorrectReplicasQuorum()` 的要求；另一种情况是结点的域编号为新的，即结点自己已发送过 view-change 。这种情况下就有可能有多于或等于 `pbftCore.allCorrectReplicasQuorum()` 条 view-change 消息的域编号与结点自己的域编号相同，因而使 `pbftCore.recvViewChange` 方法返回 `viewChangeQuorumEvent` 对象。

这里返回的 `viewChangeQuorumEvent` 对象会被加入到消息循环中。我们看一下这一事件消息是如何处理的：
```go
func (instance *pbftCore) ProcessEvent(e events.Event) events.Event {
    switch et := e.(type) {
    ......
    case viewChangeQuorumEvent:
        if instance.primary(instance.view) == instance.id {
            return instance.sendNewView()
        }
        return instance.processNewView()
    ......
    }
}
```
这里的处理很简单，如果自己是新的主结点，则调用 `pbftCore.sendNewView` 发送 new-view 消息；如果不是，则调用 `pbftCore.processNewView` 尝试处理 new-view 消息。

所以总得来说，接收处理 view-change 消息也是比较简单的。代码首先作有效性检查；然后最重要的是在接收到多于 $f+1$ 个不同结点收到的 view-change 消息时，自己也从中选一个域编号最小的值，作为自己新的域编号发送 view-change 消息；最后如果有多于 $2f+1$ 条 view-change 消息的域编号与自己的相同，则认为可以进行域转换了，因此返回 `viewChangeQuorumEvent` 对象进行处理。


### 构造发送 new-view

下面我们来看看 `pbftCore.sendNewView` 是如何构造和发送 new-view 消息的。
```go
func (instance *pbftCore) sendNewView() events.Event {
    if _, ok := instance.newViewStore[instance.view]; ok {
        return nil
    }

    vset := instance.getViewChanges()

    cp, ok, _ := instance.selectInitialCheckpoint(vset)
    if !ok {
        return nil
    }

    msgList := instance.assignSequenceNumbers(vset, cp.SequenceNumber)
    if msgList == nil {
        return nil
    }

    nv := &NewView{
        View:      instance.view,
        Vset:      vset,
        Xset:      msgList,
        ReplicaId: instance.id,
    }

    ......
}
```
这里首先检查是否已经发送过当前域编号的 view-change 消息，如果已发送过就不再重复发送了。接下来就是构造 `NewView` 对象了，也是这里我们要重点介绍的。

这里我们先介绍一下 `NewView` 结构体，它的定义如下：
```go
type NewView struct {
    View      uint64
    Vset      []*ViewChange
    Xset      map[uint64]string
    ReplicaId uint64
}
```
这个结构体的字段多数比较好理解，其中 `NewView.Vset` 是所有 `ViewChange` 消息的集合，它对应了「理论篇」中 new-view 消息中的集合 $V$； `NewView.Xset` 则是所有在收到 new-view 后要重新进行共识处理的请求的集合，其中 key 是请求的序号，value 是请求的哈希。这个字段与「理论篇」中 new-view 消息中的集合 $O$ 作用类似，但内容却是完全不同的。

这里我们重点介绍一下如何生成 `NewView.Xset`。从代码上也可以看出，生成这个集合之前，会先调用 `pbftCore.selectInitialCheckpoint` 方法。我们在前面的「使用 checkpoint」小节中已经介绍过这个方法，它的功能就是从所有 view-change 消息中选出一个序号值最高的稳定检查点。`pbftCore.assignSequenceNumbers` 从这个稳定检查点的序号值开始处理 view-change 集合中的数据。

`NewView.Xset` 的生成是由 `pbftCore.assignSequenceNumbers` 完成的。但这个方法写得太复杂，所以这里我不想一一解释其中的每一行代码，而是换一种方式，首先了解一下这个方法应该返回哪些数据，然后从方法上解释一下应该如何做才能返回这些数据，最后放上我对这个方法的改写的代码。如果你想认真了解 `pbftCore.assignSequenceNumbers` 的实现，可以对照着这里的解释和代码看，或许会有帮助。

首先解释一下 `pbftCore.assignSequenceNumbers` 应该返回哪些数据，也即 `NewView.Xset` 的作用是什么。在「理论篇」中介绍域转换时，有一个集合 $O$，它是一个 pre-prepare 消息的集合，所有收到 new-view 消息的结点，也需要像正常收到 pre-prepare 消息一样，处理集合 $O$ 中的 pre-prepare 消息。`NewView.Xset` 中的数据与此类似，只不过简单一些，只是包含了请求的哈希和序号，而不是完整的 pre-prepare 消息。结点在收到 new-view 消息后，需要使用 `NewView.Xset` 中的请求哈希和序号自己构造 pre-prepare 消息并进行处理。

那 `pbftCore.assignSequenceNumbers` 该如何生成这些序号和哈希的对应值呢？在「理论篇」中我们也详细说明了集合 $O$ 的生成方法，简单来说就是利用所有 view-change 消息中的稳定检查点和集合 $P$（处于 prepared 状态还没有 commit 的请求的集合） 确定一个请求序号区间 $[min$-$s, max$-$s]$，然后针对这一区间内的所有序号，如果序号出现在集合 $P$ 中则为此序号和对应请求创建一个 pre-prepare 消息；如果序号没在集合 $P$ 中出现，则代表这一序号代表的请求已经被所有结点 commit 了，因此用此序号和一个空请求创建 pre-prepare 消息。

但 Fabric 的实现与论文不同，它的很多消息是没有签名的，因此实现方式必然不一样。比如由于 pre-prepare 和 prepare 消息都没有签名，因此 `ViewChange.Pset` 都不可能像「理论篇」中介绍的那样，包含 $2f+1$ 条完整的 prepare 消息来证明某一请求确实处于 prepared 状态，而只能包含结点自己记录的处于 prepared 状态的请求的信息。但这种在 `ViewChange.Pset` 中的信息是不可靠的，因为结点完全可以自由伪造。

因此，想要构造 `NewView.Xset` ，即想要知道哪些请求需要在新的域中重新处理，新的主结点需要对所有 view-change 消息中的 pset 和 qset 进行汇总。比如如果有 $2f+1$ 条 view-change 中的 pset 都显示某一请求处于 prepared 状态了，那么新主结点主可以认为此请求真的处于 prepared 状态了。

所以 `pbftCore.assignSequenceNumbers` 的思路可以总结为：以 `pbftCore.selectInitialCheckpoint` 方法确定的稳定检查点的序号为起始序号，以下一个 checkpoint 的序号为结束序号；在这个序号区间内针对每一个序号，统计所有 view-change 中的 pset 和 qset ，如果有多于 $2f+1$ 条 view-change 中的 pset 都没有出现此序号对应的请求，则认为此序号没被使用过，因此只将此序号对应一个空请求；如果有多于 $f+1$ 条 view-change 中的 qset 都出现了此序号对应的请求，且在 pset 中显示是可能达成共识的（参考下面的代码中的 `couldPrepared`），则认为这一请求已被 pre-prepare 消息处理过，但最终还未达到 prepared 状态，因此将序号和请求哈希加入到 `NewView.Xset` 中，以期在新的域中重新处理；如果以上两种情况都不属于，则认为出现了问题，因此结束这一过程。

（其实我对 Fabric 的这一实现是有疑问的。在原始论文中，这一过程的目的是为了重新处理已处于 prepared 状态、但还未达到 committed 状态的请求，因为 view-change 消息中的集合 $P$ 记录的是结点中已处于 prepared 状态但不是 committed 状态的请求信息。而 Fabric 中 `pbftCore.assignSequenceNumbers` 的实现方法，是无法分辨请求是否处于 committed 状态的，它只能知道某请求是否处于 prepared 状态，并且如果超过 $2f+1$ 个结点报告某请求已处于 prepared 状态，它就不会再去处理（`NewView.Xset` 中序号对就的是一个空请求），这会导致未处于 committed 状态、但处于 prepared 状态的请求不会再次被执行。）

解释完了如何做，我们来看看我对这个方法的改写：
```go
func (instance *pbftCore) assignSequenceNumbers(vset []*ViewChange, h uint64) (msgList map[uint64]string) {
    msgList = make(map[uint64]string)

    maxN := h + 1

    for n := h + 1; n <= h+instance.L; n++ {
        if isSeqNumNotUsed(vset, n, instance) {
            msgList[n] = ""
            continue
        }

        // 运行至此，说明序号 n 被使用过。

        // preprep_list 中可能有多个元素，
        // 但所有元素的序号都等于 n。
        preprep_list = getPrePreparedListForSeqNum(vset, n)

        // 虽然 preprep_list 中可能有多个元素，
        // 但只要有一个可能可以达到 prepared 状态，就不关心其它的了。
        found := false
        for _, preprep := range preprep_list {
            if couldPrepared(vset, preprep, instance) {
                msgList[n] = d
                maxN = n
                found = true
                break
            }
        }

        if !found {
            return nil
        }
    }

    // prune top null requests
    for n, msg := range msgList {
        if n > maxN && msg == "" {
            delete(msgList, n)
        }
    }

    return msgList
}

// 判断序号 n 是否被使用了。
// 判断依据是如果有多于 2f+1 个结点的 pset 中没出现过序号 n, 则认为 n 没被使用；
// 否则认为使用过了。
func isSeqNumNotUsed(vset []*ViewChange, n uint64, pbft *pbftCore) bool {
    notUsedQuorum := 0

    for _, m := range vset {
        if ! hasSeqNum(m.Pset, n) {
            notUsedQuorum++
        }
    }

    if notUsedQuorum >= instance.intersectionQuorum() {
        return true
    } else {
        return false
    }
}

// 获取所有有效的、序号为 n 的 pre-prepare 消息。
// 所谓有效的，除了 pre-prepare 消息的序号为 n 外，
// 必须有多于 f+1 个结点的 qset 中出现过这一 pre-prepare 消息。
func getPrePreparedListForSeqNum(vset []*ViewChange, n uint64, pbft *pbftCore) []*ViewChange_PQ {
    results := make([]*viewChange_PQ)

    all_preprep := getAllPreprepareForSeqNum(vset, n)

    for pre, _ := range all_preprep {
        quorum := getPreprepareQuorum(vset, pre)
        if quorum >= pbft.f + 1 {
            tmp := pre
            results = append(results, &tmp)
        }
    }

    return results
}

// 获取所有结点的所有 qset 中序号为 n 的 pre-prepare 信息。
// 返回结点已去重。
func getAllPreprepareForSeqNum(vset []*ViewChange, n uint64) map[viewChange_PQ]struct{} {
    results := make(map[viewChange_PQ]struct{})

    for _, v := range vset {
        for _, pre := range v.Qset {
            if pre.SequenceNumber == n {
                results[*pre] = struct{}{}
            }
        }
    }

    return results
}

// 获取包含指定 pre-prepare 消息的结点数量。
// 若某 pre-prepare 消息的 view 大于参数 pre 的 view、其它数据相等，
// 也认为它们是相等的。
func getPreprepareQuorum(vset []*ViewChange, pre ViewChange_PQ) uint64 {
    quorum := 0

    for _, v := range vset {
        if hasPreprepare(v, pre) {
            quorum++
        }
    }

    return quorum
}

// 计算某 pre-prepare 消息（即参数 pq ）对应的请求是否有可能达到 prepared 状态。
// 如果满足以下至少一条的结点总数超过 2f+1，则 pq 是有可能达成 prepared 状态的：
//   1. 结点中没有与 pq 的序号相同的 prepare 消息
//   2. 结点中存在与 pq 对应的 prepare 消息（「对应」是指序号、请求哈希相同，域编号大于或等于 pq 的域编号）
func couldPrepared(vset []*ViewChange, pq *ViewChange_PQ, pbft *pbftCore) bool {
    quorum := 0

    for _, v := range vset {
        if ! hasSeqNum(v.Pset, pq.SequenceNumber) {
            quorum++
        } else if hasPrepare(v, pq) {
            quorum++
        }
    }

    return quorum >= pbft.intersectionQuorum()
}

func hasPreprepare(v *ViewChange, pre ViewChange_PQ) bool {
    return hasPQ(v.Qset, &pre)
}

func hasPrepare(v *ViewChange, pq *ViewChange_PQ) bool {
    return hasPQ(v.Pset, pq)
}

func hasPQ(pqset []*ViewChange_PQ, pq *ViewChange_PQ) bool {
    for _, eum := range pqset {
        if pq.SequenceNumber == eum.SequenceNumber  &&
           pq.view <= eum.View &&
           pq.BatchDigest == eum.BatchDigest {
            return true
           }
    }

    return false
}

func hasSeqNum(set []*ViewChange_PQ, n uint64) bool {
    for _, s := range set {
        if s.SequenceNumber == n {
            return true
        }
    }

    return false
}
```

这里稍微解释一下上面的代码与原代码的异同。在 Fabric 的实现中，在锁定序号 n 的情况下，循环遍历每一个 prepare 消息 `em`，首先看序号 n 是否可能达到 prepare 状态（此时还与 `em` 没有关系），然后再看 `em` 是否有对应的（判断 BatchDigest 和 View）、序号为 n 的 、在多于 $f+1$ 个结点都再现过的 pre-prepare 消息，如果有才将 em 对应的信息记录下来。如果最终在锁定 n 的情况下找不到满足刚才条件的 `em`，再去判断 n 是否被用过。此时正常应该不会被用过，否则这个方法就出错返回空值了。

在我自己的实现中，首先判断这个序号 n  是否被用过，如果没有则记录一个空请求。如果被用过，再去找用过这个序号的所有 pre-prepare 消息。然后从这些 pre-prepare 消息中，找一个可能可以达到 prepare 状态的，记录下来。


### 接收处理 new-view

下面我们再来看看接收到 new-view 消息，结点是如何处理的。

与其它消息类似，new-view 消息也被归到 `pbftMessage` 这一大类中处理的：
```go
func (instance *pbftCore) ProcessEvent(e events.Event) events.Event {
    switch et := e.(type) {
    ......
    case pbftMessageEvent:
        msg := et
        next, err := instance.recvMsg(msg.msg, msg.sender)
        if err != nil {
            break
        }
        return next
    case *NewView:
        return instance.recvNewView(et)
    ......
    }
}

func (instance *pbftCore) recvMsg(msg *Message, senderID uint64) (interface{}, error) {
    ......
    else if nv := msg.GetNewView(); nv != nil {
        if senderID != nv.ReplicaId {
            return nil, fmt.Errorf(...)
        }
        return nv, nil
    }
    ......
}
```

所以接收到 new-view 消息以后，是从 `pbftCore.recvNewView` 方法开始处理的。下面我们看看这个方法的代码：
```go
func (instance *pbftCore) recvNewView(nv *NewView) events.Event {
    if !( nv.View > 0 && 
       nv.View >= instance.view && 
       instance.primary(nv.View) == nv.ReplicaId && 
       instance.newViewStore[nv.View] == nil ) {
        logger.Infof(...)
        return nil
    }

    for _, vc := range nv.Vset {
        if err := instance.verify(vc); err != nil {
            logger.Warningf(...)
            return nil
        }
    }

    instance.newViewStore[nv.View] = nv
    return instance.processNewView()
}
```
在这个方法里，主要是对 new-view 消息中的数据有效性的验证，比较好理解这里不多作解释了。如果验证通过，则会将 new-view 消息存储到 `pbftCore.newViewStore` 中，后面 `pbftCore.processNewView` 会使用这个字段再次获取到这个 new-view 消息。

下面我们分析一下 `pbftCore.ProcessNewView` 这个方法。先看第一部分：
```go
func (instance *pbftCore) processNewView() events.Event {
    var newReqBatchMissing bool
    nv, ok := instance.newViewStore[instance.view]
    if !ok {
        logger.Debugf(...)
        return nil
    }

    if instance.activeView {
        logger.Infof(...)
        return nil
    }

    cp, ok, replicas := instance.selectInitialCheckpoint(nv.Vset)
    if !ok {
        logger.Warningf(...)
        return instance.sendViewChange()
    }

    ......
}
```
第一部分代码首先判断 `pbftCore.newViewStore` 中是否有 new-view 消息需要处理；然后判断当前域是否是活跃的（`pbftCore.activeView` 这个字段在发起域转换的方法 `pbftCore.sendViewChange` 中被设置为 true，域转换成功后，会再次设置成 false）；最后代码调用 `pbftCore.selectInitialCheckpoint` 获取 new-view 消息中序号最高的、稳定的检查点。后面的代码以这个稳定检查点的序号为基础进行处理，后面我们就称呼它为 `cp.SequenceNumber`。

我们继续看下面的代码：
```go
func (instance *pbftCore) processNewView() events.Event {
    ......

    speculativeLastExec := instance.lastExec
    if instance.currentExec != nil {
        speculativeLastExec = *instance.currentExec
    }

    // If we have not reached the sequence number, check to see if we can reach it without state transfer
    // In general, executions are better than state transfer
    if speculativeLastExec < cp.SequenceNumber {
        canExecuteToTarget := true
        ......
        if canExecuteToTarget {
            logger.Debugf(...)
            return nil
        }
    }

    ......
}
```
由于上面这部分代码逻辑简单但代码繁锁，因此这里并未展示全部代码，而只显示了这部分的头尾。这一部分的代码主要是判断当前结点的最新序号小于 `cp.SequenceNumber` 时，是否可以自己执行请求到序号等于 `cp.SequenceNumber` 的情况。当请求执行得太慢，而这些未执行的请求其实都已经处于 committed 状态时，就会发生这种情况。如果可以执行到 `cp.SequenceNumber` ，就直接返回等待请求的执行。

这里你可能会疑惑这里直接返回，那 new-view 消息还处理吗？答案当然是还是要处理的。在每执行完一个请求后，会再次调用 `pbftCore.processNewView` 进行处理：
```go
func (instance *pbftCore) ProcessEvent(e events.Event) events.Event {
    switch et := e.(type) {
    ......
    case execDoneEvent:
        ......
        return instance.processNewView()
    ......
    }
}
```
这也是要把 new-view 消息存储到 `pbftCore.newViewStore` 字段中的原因吧。

如果刚才的代码判断无法自己执行到 `cp.SequenceNumber`  ，就会继续向下执行，发起数据同步。这部分逻辑在后面，我们一会就会看到。

我们继续看后面的部分代码：
```go
func (instance *pbftCore) processNewView() events.Event {
    ......

    msgList := instance.assignSequenceNumbers(nv.Vset, cp.SequenceNumber)
    if msgList == nil {
        logger.Warningf(...)
        return instance.sendViewChange()
    }

    if !(len(msgList) == 0 && len(nv.Xset) == 0) && !reflect.DeepEqual(msgList, nv.Xset) {
        logger.Warningf(...)
        return instance.sendViewChange()
    }

    ......
}
```
这部分代码主要是判断 new-view 消息中的 xset 是否正确。在「理论篇」中我们介绍过，各从结点在收到 new-view 消息时，会使用和新主结点相同的方法计算集合 $O$ ，从而判断 new-view 消息中集合 $O$ 是否正确。这里的意思是一样的，各从结点使用和新主结点相同的方法 `pbftCore.assignSequenceNumbers` 计算集合 `NewView.Xset`，并判断自己的计算结果与收到的 new-view 中的 xset 数据是否一致。

我们继续看下面的代码：
```go
func (instance *pbftCore) processNewView() events.Event {
    ......

    if instance.h < cp.SequenceNumber {
        instance.moveWatermarks(cp.SequenceNumber)
    }

    if speculativeLastExec < cp.SequenceNumber {
        snapshotID, err := base64.StdEncoding.DecodeString(cp.Id)

        target := &stateUpdateTarget{
            checkpointMessage: checkpointMessage{
                seqNo: cp.SequenceNumber,
                id:    snapshotID,
            },
            replicas: replicas,
        }

        instance.updateHighStateTarget(target)
        instance.stateTransfer(target)
    }

    ......
}
```
这里首先提高自己的 watermark 即 `pbftCore.h` 的值，然后在结点最大请求序号小于 `cp.SequenceNumber` 时，发起数据同步。前面我们说过，如果结点最大请求序号小于 `cp.SequenceNumber` 时，会判断是否可以自己就能执行到序号 `cp.SequenceNumber` 代表的请求。如果不能，就会继续执行代码到这里。

我们继续看后面的代码：
```go
func (instance *pbftCore) processNewView() events.Event {
    ......

    for n, d := range nv.Xset {
        if n <= instance.h {
            continue
        } else {
            if d == "" {
                continue
            }

            if _, ok := instance.reqBatchStore[d]; !ok {
                if _, ok := instance.missingReqBatches[d]; !ok {
                    newReqBatchMissing = true
                    instance.missingReqBatches[d] = true
                }
            }
        }
    }

    if len(instance.missingReqBatches) == 0 {
        return instance.processNewView2(nv)
    } else if newReqBatchMissing {
        instance.fetchRequestBatches()
    }

    return nil
}
```
这是 `pbftCore.processNewView` 的最后一部分代码。由于后面结点要把 `NewView.Xset` 中的信息作为 pre-prepare 消息进行处理，因此这里首先判断自己是否存储了 xset 是的请求。如果未存储某请求，则将其记录到 `pbftCore.missingReqBatches` 中，并调用 `pbftCore.fetchRequestBatches` 同步这些请求；如果已全部存储了这些请求，则调用 `pbftCore.ProcessNewView2` 继续处理。

在发起同步请求后，结点最终会在消息循环中收到 `returnRequestBatchEvent` 消息。处理此消息后会再次调用 `pbftCore.processNewView` 处理 new-view 消息。

下面我们看看 `pbftCore.processNewView2` 是如何处理的。先看第一部分代码：
```go
func (instance *pbftCore) processNewView2(nv *NewView) events.Event {
    instance.stopTimer()
    instance.nullRequestTimer.Stop()

    instance.activeView = true
    delete(instance.newViewStore, instance.view-1)

    ......
}
```
从代码的一开始可以看到，执行到 `pbftCore.processNewView2` ，这个 new-view 消息基本算是处理完了。在这之前排除了所有异常，所以这里将旧的 new-view 删除，并设置 `pbftCore.activeView` 为 true 激活当前的域。

我们来看下一部分的代码：
```go
func (instance *pbftCore) processNewView2(nv *NewView) events.Event {
    ......

    instance.seqNo = instance.h
    for n, d := range nv.Xset {
        if n <= instance.h {
            continue
        }

        reqBatch, ok := instance.reqBatchStore[d]
        if !ok && d != "" {
            logger.Criticalf()
        }
        preprep := &PrePrepare{
            View:           instance.view,
            SequenceNumber: n,
            BatchDigest:    d,
            RequestBatch:   reqBatch,
            ReplicaId:      instance.id,
        }
        cert := instance.getCert(instance.view, n)
        cert.prePrepare = preprep
        cert.digest = d
        if n > instance.seqNo {
            instance.seqNo = n
        }
        instance.persistQSet()
    }

    instance.updateViewChangeSeqNo()

    ......
}
```
这段代码逻辑比较简单，但很重要，它使用 `NewView.Xset` 中的每一个元素构造 pre-prepare消息，并存储到 `pbftCore.certStore` 中。在这个过程中还不断更新自己的序号值，并最后使用这个序号值更新 `pbftCore.viewChangeSeqNo` 字段（当处理的请求达到此字段代表应该进行 view-change 了）。

我们继续看后面的代码：
```go
func (instance *pbftCore) processNewView2(nv *NewView) events.Event {
    ......

    if instance.primary(instance.view) != instance.id {
        for n, d := range nv.Xset {
            prep := &Prepare{
                View:           instance.view,
                SequenceNumber: n,
                BatchDigest:    d,
                ReplicaId:      instance.id,
            }
            if n > instance.h {
                cert := instance.getCert(instance.view, n)
                cert.sentPrepare = true
                instance.recvPrepare(prep)
            }
            instance.innerBroadcast(&Message{Payload: &Message_Prepare{Prepare: prep}})
        }
    } else {
        instance.resubmitRequestBatches()
    }

    instance.startTimerIfOutstandingRequests()
    return viewChangedEvent{}
}
```
这段代码里，如果自己不是新的主结点，则继续处理 new-view 消息中的 xset 数据，为 xset 中的每一项生成一个 prepare 消息，并处理和广播；如果自己是新的主结点，则调用 `pbftCore.resubmitRequestBatches` 将之前收到的、还未处理的请求进行处理。

这段代码结合上一段代码，其实就是实现了 xset 中的消息的重新处理。每个结点在收到 new-view 消息并检查 xset 通过以后，首先使用 xsest 中的信息生成 pre-prepare 消息并存储（代表接受 pre-prepare 消息）；然后继续生成 prepare 消息并广播和处理。这一过程和正常收到请求然后进行共识三阶段的处理是一样的。所以这里也就实现了「理论篇」中介绍的处理集合 $O$ 的过程。

`pbftcore.processNewView2` 最终返回 `viewChangedEvent` 对象。`obcBatch` 对象的消息循环中在收到这个事件时，会做一些事情来应对域发生转变的这种情况，比如清空一些数据等。这些应对中比较重要的是检查自己是不是新的主结点，如果是则要调用 `obcBatch.resubmitOutstandingReqs` 将自己之前收到的、没有被处理的请求进行处理。

总得来说，new-view 消息的处理比较简单。在收到 new-view 消息以后，首先是对消息中的数据进行有效性判断；然后在必要的情况要会发起数据同步；最后也是最重要的，就是重新处理 xset 中的请求。在 new-view 消息处理成功以后，结点还要判断自己是不是成了新的主结点，如果是则要马上处理之前收到的、但还未被处理的请求。


# 其它细节

这一小节我们来看一下代码中的其它细节。这些细节不是 PBFT 算法重点，但我觉得也需要稍微了解一下，因此在这里啰嗦一下。


## watermark

在「理论篇」和本篇文章中，我们多次提到过 $[h, H]$ 这个区间。这个区间被称为 watermark，它限定了一个 checkpoint 内序号的最大值和最小值。下面我们来看一下 pbft 的实现中是如何设置这个区间值的。

在 pbft 的实现中，字段 `pbftCore.h` 代表了区间的下限值 $h$；上限值 $H$ 没有一个直接字段代表，而是通过 `pbftCore.h` 和 `pbftCore.L` 计算出来的：$H =$ `pbftCore.h + pbfftCore.L`。可以看出，`pbftCore.L` 代表了这个区间的大小。

这里我们先看 `pbftCore.L` 是如何设置的。这个值在 `pbftCore` 对象创建之初就设置好了，之后不会再变：
```go
func newPbftCore(id uint64, config *viper.Viper, consumer innerStack, etf events.TimerFactory) *pbftCore {
    ......

    instance.K = uint64(config.GetInt("general.K"))

    instance.logMultiplier = uint64(config.GetInt("general.logmultiplier"))
    if instance.logMultiplier < 2 {
        panic("Log multiplier must be greater than or equal to 2")
    }
    instance.L = instance.logMultiplier * instance.K // log size

    ......
}
```
可以看到 `pbftCore.L` 是通过从配置文件中读取 `pbftCore.K` 和 `pbftCore.logMultiplier` ，然后相乘计算出来的。其中 `pbftCore.K` （也即配置文件中的 `general.K` 字段）代表 checkpoint 的序号的区间大小。所以这里可以看出，`pbftCore.L` 所代表的 watermark 的区间大小会是 checkpoint 的区间大小的整数值，至少两倍。

正是由于 `pbftCore.L` 至少为 checkpoint 区间大小的两倍，所以超过 $h$ + instance.L/2 的序号肯定是在下一个 checkpoint 中了。因此才有下面代码中 `n > instance.h+instance.L/2` 的判断：
```go
func (instance *pbftCore) sendPrePrepare(reqBatch *RequestBatch, digest string) {
    ......

    if !instance.inWV(instance.view, n) || n > instance.h+instance.L/2 {
        // We don't have the necessary stable certificates to advance our watermarks
        logger.Warningf(...j)
        return
    }

    ......
}
```

我们再来看看 `pbftCore.h` 是如何被设置的。设置这个字段的方法是 `pbftCore.moveWatermarks`。这个方法很简单，就是将一些字段中的请求序号小于新的 $h$ 的数据删除，并设置 `pbftcore.h` 为新的值，比较简单就不多说了。


## 同步状态数据

虽然「理论篇」中没有涉及如何进行状态数据的同步，甚至这都不算是 PBFT 算法范畴内的东西。但这里我们还是简单来看一下 pbft 模块是如何处理状态数据的同步的吧。

首先我们要介绍字段 `pbftCore.skipInProgress`。当结点发现自己已经落后别人了，就会标记这个字段为 `true`。因而以后的处理中，在合适的时机如果这个字段为 `true`，就会发起状态同步。总共有三个地方设置这个字段为 `true` ：
```go
func (instance *pbftCore) stateTransfer(optional *stateUpdateTarget) {
    if !instance.skipInProgress {
        instance.skipInProgress = true
        ......
    }
    ......
}

func (instance *pbftCore) execDoneSync() {
    if instance.currentExec != nil {
    ......
    } else {
        // XXX This masks a bug, this should not be called when currentExec is nil
        instance.skipInProgress = true
    }
    ......
}

func (instance *pbftCore) weakCheckpointSetOutOfRange(chkpt *Checkpoint) bool {
    ......
    if chkpt.SequenceNumber < H {
        ......
    } else {
        ......
        instance.skipInProgress = true
        ......
    }
}
```
其中第一个方法 `pbftCore.stateTransfer` 的功能就是发起数据同步，因此它会首先设置 `pbftCore.skipInProgress` 字段；第二个方法如注释所示，出现这种情况应该是一个 bug，所以我们忽略它；第三种情况的逻辑是，在收到一个 checkpoint 时，如果我们发现有超过 $f+1$ 个节点向自己发送了序号超过自己当前 watermark 上限的 checkpint，那么就认为自己已经落后了，因此设置 `pbftCore.skipInProgress` 字段，在后面适当的时候进行状态数据同步。

你可能会问为什么要有 `pbftCore.skipInProgress` 这么一个字段，为什么不在想要同步数据的时候立即进行同步呢？我想这是为了保持数据的完整性吧。设想如果某一时间正在执行一个请求，这个请求肯定是基于当前的数据状态执行的；如果此时进行同步，肯定会改变当前的数据状态，从而造成混乱。

因此正确的开始同步状态数据的时机，是执行完某个请求以后：
```go
func (instance *pbftCore) ProcessEvent(e events.Event) events.Event {
    switch et := e.(type) {
    ......
    case stateUpdatedEvent:
        update := et.chkpt
        if et.target == nil || update.seqNo < instance.h {
            ......
            instance.retryStateTransfer(nil)
            ......
        }
    case execDoneEvent:
        ......
        if instance.skipInProgress {
            instance.retryStateTransfer(nil)
        }
        ......
    ......
    }
}

func (instance *pbftCore) witnessCheckpointWeakCert(chkpt *Checkpoint) {
    ......

    if instance.skipInProgress {
        instance.retryStateTransfer(target)
    }
}

func (instance *pbftCore) processNewView() events.Event {
    ......

    if speculativeLastExec < cp.SequenceNumber {
        ......
        instance.stateTransfer(target)
    }

    ......
}
```
从以上节选的代码可以看出，在以下情况下才会发起数据同步：
- 当某个请求执行成功后（`execDoneEvent`）
- 或状态数据同步完成但仍然落后时，
- `pbftCore.witnessCheckpointWeakCert` 中发现已经处于数据同步的状态时
- 处理 new-view 消息时发现当前结点落后于 new-view 消息中的 checkpiont 时

这四种情况中，共同点在于此时肯定不会有请求正在执行。其中前两种情况比较好理解，第四种情况我们在「接收处理 new-view」小节中也讨论过。这里稍微说一下第三种情况，`pbftCore.witnessCheckpointWeakCert` 是在收到 checkpoint 消息时调用的，其逻辑是发现已经有超过 $f+1$ 个节点广播了同一个 checkpoint ，因此基本认为这个 checkpoint 是稳定的。所以这个方法中会判断如果已经在进行数据同步，就马上更新一下同步目标到最新的。

最后，`pbftCore.stateTransfer` 和 `pbftCore.retryStateTransfer` 用来执行状态同步操作，由于 pbft 只是一个共识模块，状态数据的管理不属于这个模块的功能，所以这两个方法最终调用其它模块提供的 `Executor.UpdateState` 接口进行状态同步。


## outstanding

在 `obcBatch` 和 `pbftCore` 两个对象中，都有「 outstanding 请求」的概念，比如 `obcBatch.resubmitOutstandingReqs` 方法，再比如 `pbftCore.executeOutstanding` 方法和 `pbftCore.outstandingReqBatches` 字段 。在学习源代码时，这两个「 outstanding 」 一度让我混乱，因此这里简单说一下。

我们先说 `obcBatch` 中的「 outstanding 请求」的概念。如果一个节点不是主结点，它仍有可能直接收到请求。比如我们在「理论篇」中提到的，如果当前主结点不响应，客户端会将请求发给所有结点；或者 Fabric 中的客户端本来就是用广播的方式发送请求（还没学习这方面的东西，只是说一种可能性）。那么从结点收到这些请求后，不会去处理，但会记录下来。这些请求就是 `obcBatch` 中的「 outstanding 请求」。即这些请求是等待主结点来处理的请求。

既然从结点不主动处理这些请求，那么它们记录这些请求有什么用呢？作用在于，当发生域转换时，原来的从结点变成了新的主结点，此时它可以检查自己记录的这些未被旧主结点处理的请求，并构造 pre-prepare 消息对它们进行处理。如此一来就可以避免某一请求不被旧的主结点处理、又被新的主结点忽略的情况。

`pbftCore` 中的「outstanding 请求」又是另一种情况。总得来说，在 `pbftCore` 中「outstanding 请求」是指那些处于共识三阶段的、但还未被执行的请求。具体来说，`pbftcore.executeOutstanding` 方法用来执行那些已处于 committed 状态、但还示被执行的请求；而 `pbftCore.outstandingReqBatches` 字段则代表处于共识三阶段过程中、但还未达到 committed 状态的请求，对于这些请求，每次域转换成功后新主结点会调用 `pbftCore.resubmitRequestBatches` 对它们重新进行处理。


# 总结

在这篇文章里，我们详细分析了 Fabric 项目中的 pbft 模块的实现，从代码的角度重新学习了 PBFT 算法的细节。可以看到，Fabric 的实现与原始论文中的算法基本相同，但也有一些很重要的不同，具体表现在：
1. 除了 view-change 消息外，其它消息都没有签名。
2. 基于上面这一点，各消息的实现和处理方法肯定与论文中的不同（这里的不同点很大）。
3. 主结点的 prepare 消息是被忽略的。
4. 请求可以成批发送和处理（`RequestBatch`），而不像论文中一次只能发送一个请求。'
5. 在共识三阶段中，默认主结点的 prepare 消息永远正确；因此主结点的 prepare 消息被忽略，只要有 $2f$ 条 prepare 消息就可以达到 prepared 状态
6. 请求是按序号大小顺序，从小到达依次执行，即使某序号大的请求先达到 committed 状态。
7. 即使没发生过异常，也会每间隔一定数量的请求就发起域转换。

以上就是我对 pbft 模块的分析。水平有限，文章中难免有错误的地方，如果发现感谢您能不吝指正。