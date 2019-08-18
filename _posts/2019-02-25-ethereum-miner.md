---
layout: post
title:  "以太坊源码解析：miner"
categories: ethereum
tags: 原创 ethereum miner 源码解析
excerpt: miner模块是以太坊（ethereum）中对“挖矿”进行调度的模块。让我们来看看，miner模块是如何调度的。
author: fatcat22
---

* content
{:toc}





>本篇文章分析的源码地址为：[https://github.com/ethereum/go-ethereum](https://github.com/ethereum/go-ethereum)  
>分支：[master](https://github.com/ethereum/go-ethereum/tree/master)  
>commit id: [257bfff316e4efb8952fbeb67c91f86af579cb0a](https://github.com/ethereum/go-ethereum/tree/257bfff316e4efb8952fbeb67c91f86af579cb0a)  


# 引言
所有的区块链项目中，「挖矿」都是非常重要的一项功能。前面几篇文章中我们介绍了以太坊中挖矿的两种共识实现模块：clique 和 ethash。那么这篇文章里，我们介绍另一个与挖矿紧密相关的模块 miner。

不要被 miner 这个模块名字搞糊涂了：miner 是「矿工」的意思，但这个模块的功能，更像是现实中一个矿上的「调度室」，它管理着什么时候开始挖矿、什么时候开始挖一个新的区块等。

分析以太坊的 miner 模块的代码，我觉得最重要的是学习它作为一个「调度器」都实现了哪些功能，以便我们自己写一个类似的功能的时候，可以有的放矢，知道大约需要具备什么功能。因此本篇文章的思路是首先从整体上看一下整个模块的功能结构，然后再对一些较重要又较难理解的问题进行一下详细的说明。


# 主要流程

miner 模块只有两个重要的结构体：Miner 和 worker。这一小节我我们介绍一下这两个对象的主要工作流程。知道了这两个对象的工作流程，基本就了解了以太坊的出块流程了。


### Miner

Miner的功能比较简单，主要就是控制挖扩工作的开启（Start）和停止（Stop），以及设置一些数据和获取挖矿的状态。因此 Miner 对象的工作流程，主要体现在对挖矿工作的启停的控制上。

启停的控制本来很简单，但以太坊的挖矿有一个特性，就是在 downloader 模块同步区块时，会暂停挖矿；等区块同步完成以后，再恢复挖矿（不过这个过程只有一次）。这样一来就增加了一些复杂度。 `Miner` 对象是通过两个字段辅助实现这一特性的， 它们是：
- `Miner.canStart`
- `Miner.shouldStart`

这里 `canStart` 代表「可不可以启动」，而 `shouldStart` 代表「应不应该启动」。 `canStart` 是先决条件，如果它为 0，无论 `shouldStart` 是什么值都不能启动挖矿；如果 `canStart` 为非 0，才看 `shouldStart` 是否应该启动。

知道了这两个字段的意义，不用看代码我们也能想明白，什么时候它们的值会是 0，什么时候会是 1。在 `Miner` 对象创建之时，这时挖矿功能肯定是可以启动的，但还没有调用 `Miner.Start` 方法，所以不应该启动。所以此时 `canStart` 的值是 1，而 `shouldStart` 的值为 0：
```go
func New(eth Backend, config *params.ChainConfig, mux *event.TypeMux, engine consensus.Engine, recommit time.Duration, gasFloor, gasCeil uint64, isLocalBlock func(block *types.Block) bool) *Miner {
  miner := &Miner{
    ......
    canStart: 1, // shouldStart 的值没有被设置，默认值为 0
  }
  ......
}
```

在 `Miner` 对象创建后调用 `Miner.Start` 时， 代表「应该启动」挖矿功能了，至于「可不可以启动」，还要看 `canStart` 的值：
```go
func (self *Miner) Start(coinbase common.Address) {
  // 应该启动
  atomic.StoreInt32(&self.shouldStart, 1)
  ......

  // 如果不可以启动，就直接返回；否则调用 worker.start 启动
  if atomic.LoadInt32(&self.canStart) == 0 {
    return
  }
  self.worker.start()
}
```

类似的，在调用 `Miner.Stop` 时，代表「应该停止」挖矿了（但这里不关心「可不可以停止」，因为随时可以停止）：
```go
func (self *Miner) Stop() {
  self.worker.stop()
  atomic.StoreInt32(&self.shouldStart, 0)
}
```

以上是调用者对挖矿的控制。那么当 downloader 模块开始同步时，按约定要暂时停止挖矿。此时 `canStart` 应该为 0；而 `shouldStart` 则要看当前的状态，如果当前正在挖矿，那么 `shouldStart` 应被设置为 1 （因为按理说此时 `Miner` 「应该」是在挖矿的，只不过被暂时停止了）：
```go
func (self *Miner) update() {
  events := self.mux.Subscribe(
    downloader.StartEvent{}, downloader.DoneEvent{}, downloader.FailedEvent{})

  for {
    select {
    case ev := <-events.Chan():
      ......

      switch ev.Data.(type) {
      case downloader.StartEvent:
        atomic.StoreInt32(&self.canStart, 0)
        if self.Mining() {
          self.Stop()
          atomic.StoreInt32(&self.shouldStart, 1)
        }

      ......
      }

    ......
    }
  }
}
```
可以看到，在收到 downloader 开始同步的消息 `downloader.StartEvent` 后，会立即设置 `canStart` 为 0，即「不可以启动」；但如果当前正在挖矿，那么接着设置 `shouldStart` 为 1，即「应该启动」。

当 downloader 同步结束后，miner 模块又可以正常挖矿了。所以此时应设置 `canStart` 为 1；而如果 `shouldStart` 为 1，则说明挖矿功能被我们暂停了，现在应该恢复：
```go
func (self *Miner) update() {
  events := self.mux.Subscribe(
    downloader.StartEvent{}, downloader.DoneEvent{}, downloader.FailedEvent{})

  for {
    select {
    case ev := <-events.Chan():
      ......

      switch ev.Data.(type) {
        ......
      case downloader.DoneEvent, downloader.FailedEvent:
        shouldStart := atomic.LoadInt32(&self.shouldStart) == 1

        atomic.StoreInt32(&self.canStart, 1)
        atomic.StoreInt32(&self.shouldStart, 0)
        if shouldStart {
          self.Start(self.coinbase)
        }
        // stop immediately and ignore all further pending events
        return
      }
    ......
    }
  }
}
```
可以看到，在收到 downloader 同步结束的消息之后（无论成功或失败），会立即设置 `canStart` 为 1，即 「可以启动」；如果 `shouldStart` 为 1，则代表我们应该把挖矿功能启动起来，所以调用 `self.Start` 再次启动挖矿。

需要多说一句的是，这段代码中有个 return，即处理完直接返回了，所以 `Miner.update` 这个 go routine 也就退出了。以后 downloader 再同步区块，miner 也不会暂停了。

总得来说， `Miner` 的主要流程就是控制挖矿功能的启停。由于要在区块同步时暂停挖矿，所以使用 `canStart` 和 `shouldStart` 控制挖矿的启停。在任何时候，都要根据这两个字段判断「能不能启动挖矿」和「应不应该启动挖矿」，然后再作相应的操作启动挖矿（或不作任何操作）。

这里再多聊两句，解释一下为什么要在区块同步时暂停挖矿。miner 模块总是在本地主链上的最新的区块（也即「chain head」，）的基础之上出块，如果「chain head」区块发生变化了，那么就需要以新的「chain head」为基础出块，之前正在出的块就得作废。而在区块同步时，「chain head」变化是很频繁的，若此时仍在挖矿，只能让 miner 模块不断的作废正在计算的区块，这显然是一种浪费，所以干脆暂停算了。至于为什么只暂停一次，主要是因为防止 DOS 攻击。如果每次同步都要暂停挖矿，试想你搭建了一个挖矿节点开始挖矿，但总有别的节点告诉有区块需要同步，你就不得不总是暂停挖矿去同步区块；如果这个节点是恶意的一直让你同步区块（即使都同步失败了），你也一直也挖不了矿。


### worker

除了 `Miner` 对象外，miner 模块还有一个对象，就是 `worker`。我们可以理解它为「工人」，因为具体的「挖矿调度」工作都是由 `worker` 对象完成的。

`worker` 对象的每一项工作，都是由一些消息触发的，因此乍看这个对象的代码，会感觉比较混乱。不过只要把握了它的主要流程，其实也是挺清晰的。下面是我画的关于 `worker` 对象的工作流程图：
![](/pic/ethereum-miner/worker-flowchart.png)

从图中可以看到，出块的时机就是最顶上的几个 channel 接收到消息时。我们首先解释一下这几个 channel：
- startCh  
此消息在调用`newWorker`函数和调用`worker.start`方法时都被被触发。代表启动挖矿工作。收到此消息后，启动 timer 并发送消息给 `newWorkCh` 以发起一次新的出块任务。
- timer.C  
此消息是一个定时器消息，每隔一段时间都会发消息给 `newWorkCh` 发起一次出块任务。间隔时间由生成worker对象时的参数 `recommit` 决定（但 resubmitIntervalCh 和 resubmitAdjustCh 消息可以对其作出修改）。
- chainHeadCh  
此消息在有新的区块被加入到本地的区块数据库时被触发（此消息由 blockchain 模块发送）。收到消息后会发送消息给 `newWorkCh` 发起一次出块任务。
- chainSideCh  
此消息在本地有区块被加入到侧链时被触发。收到此消息后，将消息中的区块（即加入到侧链中的区块）加入到「叔块」列表中，并根据当前状态，可能选取两个叔块提交一个出块任务。（这个 channel 是以太坊鼓励出叔块的体现）
- txsCh  
此消息在收到新的 transaction 时被触发。一般情况下收到这个消息时不会提交新的出块任务，除非当前的共识算法是 clique 模块且配置里区块间隔值（Period）为 0（这种配置的意义就是收到交易时立即出块，这里算是对这一意义的体现）。

在出块的 channel 被触发以后，所有新的出块任务被发送给 `newWorkch`。在这个 channel 的处理中，会调用 `worker.commitNewWork` 收集新块的基本信息，并提交出块任务到 `taskCh` 中。

在 `taskCh` 的处理中，会首先中止之前的出块任务。因为既然新的出块任务到达了，那么旧的出块任务中的信息显示已经过时了。在中止旧的任务后，将任务信息发送给 `engine.Seal` 对区块进行签名（也就是真正的「挖矿计算」）。这个过程是异步的。当签名成功后，共识模块（engine）会发送消息给 `resultCh`。至此，我们就挖到了一个新的模块，因此在 `resultCh` 的处理中，将这个新区块加入本地库，并广播给别的人知道。

基本上 `worker` 的工作流程就是这样的。虽然不算复杂，但在流程中有一些细节的操作，比如将要提交新的出块任务时，如何废除旧的任务；比如如何在提交新的签名任务时，中止对之前区块的签名。在下一节里，我将 miner 模块的一具体功能做了总结，进行详细的说明。


# 主要功能


上一小节里梳理出了 miner 模块的主要工作流程。但还有很多细节，是难以在工作流程中说清楚的。因此这一小节我们对 miner 模块的一些主要功能作一个介绍，这样如果我们自己要实现一个类似的模块，在功能点上就可以有所参照了。

这里我总结了 miner 模块有以下一些功能特性：
1. 同步区块时暂停挖矿：在收到同步区块的消息后，要暂停挖矿（否则挖到的很可能不是主链上的区块），待同步成功后再继续。但这个过程中只做一次，以防止 DOS 攻击。
2. 多个出块时机：在收到 start 消息(`startCh`)、新区块插入的消息(`chainHeadCh`)、侧链上区块插入的消息(`chainSideCh`)、收到新的交易时(`txsCh`)，都会视情况着手生成区块。其中侧链上的区块不管有没有新的 transaction，都会生成一个区块。这是以太坊在鼓励矿工包含叔块；收到新的交易时，只有是 clique 共识且 period 为 0（clique的 period 为 0 代表只有有新交易时才生成区块，其它任何时候都不生成）时，才生成区块，否则只是将新的交易加入到待出块的信息中。
3. 随时准备出块：即使调用者没有调用 start 方法正式开始挖矿，只要创建了 `worker` 对象，就一直在准备挖矿。
4. 可以出空块：在某些情况下，即使没有新的 transaction ，也可以出块。
5. 叔块：在收到侧链的上区块时，鼓励矿工引用这些区块作为叔块。
6. 出块频率可动态调整：这主是要指 timer 中的 recommit 功能。
7. 出块动作序列化：虽然有多个源头可以发起一次生成区块的行为，但最后都会将这些行为按先后顺序一个一个进行，而且后来的会中断之前的。

其中前两个比较简单，我们在前面介绍流程时已经介绍过了，后面就不再提了。下面我们重点介绍后五个功能点。


### 随时准备出块
在 miner 模块的实现中，只要创建了 `Miner` 对象，就一直处于「随时准备着」的状态。

刚才我们已经分过析 `worker` 中有多个出块时机。这些时机的触发并不需要调用 `worker.start` 方法，只要生成了 `worker` 对象就可能被触发。因此即使没有调用 `worker.start` 方法正式挖矿，在那些出块时机到来时，也会将收集到的消息放到 `worker.current` 变量中。

随时准备出块的原因，一是因为在生成 `worker` 对象时，发送了消息给 `startCh`：
```go
func newWorker(config *params.ChainConfig, engine consensus.Engine, eth Backend, mux *event.TypeMux, recommit time.Duration, gasFloor, gasCeil uint64, isLocalBlock func(*types.Block) bool) *worker {
  ......

  // Submit first work to initialize pending state.
  worker.startCh <- struct{}{}
}
```

收到 `startCh` 消息后，会走一个出块的流程（虽然会忽略一些正式出块的情况下才会执行的代码），因此就把一些出块相关的字段初始化了（最重要的，比如 `worker.current` ）。处理 `startCh` 消息的代码如下：
```go
func (w *worker) newWorkLoop(recommit time.Duration) {
  ......

  commit := func(noempty bool, s int32) {
    ......  
    w.newWorkCh <- &newWorkReq{interrupt: interrupt, noempty: noempty, timestamp: timestamp}
    timer.Reset(recommit)
    ......
  }

  for {
    select {
    case <-w.startCh:
      clearPending(w.chain.CurrentBlock().NumberU64())
      timestamp = time.Now().Unix()
      commit(false, commitInterruptNewHead)

    ......
    }
  }
}
```

二是因为在监听三个「外部消息」：chainHeadCh、chainSideCh、txsCh，它们随时可能会被触发，触发后就马上需要作相应的处理进行出块或出块准备。

我们刚才提到「忽略一些正式出块的情况一下才会执行的代码」，那哪些是在「准备」阶段会被略过、而正式出块时才会执行的代码呢？

##### 「准备」时被略过的代码
调用 `worker.start` 与否的区别，在于 `worker.running` 的值不同。这个字段标志着是否在正式挖矿中，在代码中由 `worker.isRunning` 来读取 `worker.running` 字段并返回判断结果。因此我们来看一下调用 `worker.isRunning` 的几处代码。

首先是 `worker.commit` 中的代码：
```go
func (w *worker) commit(uncles []*types.Header, interval func(), update bool, start time.Time) error {
  if w.isRunning() {
    ......  
    case w.taskCh <- &task{receipts: receipts, state: s, block: block, createdAt: time.Now()}:
    ......
  }
}
```

这里的判断是最重要的，只有在正式挖矿时，才会提交任务给 `taskCh` 。在 `taskLoop` 中收到 `taskCh` 消息后才会调用共识接口的` Seal` 方法对区块进行签名。

然后是 `newWorkLoop` 中的代码：
```go
func (w *worker) newWorkLoop(recommit time.Duration) {
  ......
  case <-timer.C:
    if w.isRunning() && (w.config.Clique == nil || w.config.Clique.Period > 0) {
      if atomic.LoadInt32(&w.newTxs) == 0 {
        timer.Reset(recommit)
        continue
      }
      commit(true, commitInterruptResubmit)
    }
  ......
}
```

可以看到在定时器 `timer` 中，如果没有正式挖矿，是不会提交新的挖矿任务的。也就是说在调用 `worker.start` 之前，定时器是无效的。

接下来是 `mainLoop` 中的代码：
```go
func (w *worker) mainLoop() {
  ......
  case ev := <-w.chainSideCh:
    if w.isRunning() && w.current != nil && w.current.uncles.Cardinality() < 2 {
      w.commitUncle(w.current, ev.Block.Header())
      w.commit(uncles, nil, true, start)
    }  
  case ev := <-w.txsCh:
    if !w.isRunning() && w.current != nil {
      w.commitTransactions(txset, coinbase, nil)
    } else {
      if w.config.Clique != nil && w.config.Clique.Period == 0 {
        w.commitNewWork(nil, false, time.Now().Unix())
      }
    }
  ......
}
```

这里面 `chainSideCh` 和 `txsCh` 都会判断是否在正式挖矿。在收到侧链消息chinSideCh中，只有在正式挖矿时才会将侧链区块提交到 `current` 字段中，且调用 `worker.commit` 提交一个新的出块任务。

下面是 `commitNewWork` 中的代码：
```go
func (w *worker) commitNewWork(interrupt *int32, noempty bool, timestamp int64) {
  ......
  if w.isRunning() {
    header.Coinbase = w.coinbase
  }
  ......
}
```

可见只有在正式挖矿时，才会设置 `header.Coinbase` 字段（这个字段标识挖矿奖励该给谁）。

下面是 `commitTransactions` 方法的代码：
```go
func (w *worker) commitTransactions(txs *types.TransactionsByPriceAndNonce, coinbase common.Address, interrupt *int32) bool {
  ......
  if !w.isRunning() && len(coalescedLogs) > 0 {
    go w.mux.Post(core.PendingLogsEvent{Logs: cpy})
  }
  ......
}
```

这段代码其实与我们分析的问题关系不大，只是出于完整性将其放在这里。

总结一下上面的几处代码判断，可以看到：
- 如果没有正式挖矿，是无论如何不会调用共识接口的 `Seal` 方法的（`worker.commit`）
- 如果没有正式挖矿， timer 定时器是无效的（`worker.newWorkLoop`）
- 如果没有正式挖矿，在收到侧链区块时不会提交带叔块的当前区块（`worker.mainLoop`）
- 如果没有正式挖矿，在收到新的 transactions 时只会将其提交到 `worker.current` 中，不会提交出块任务；如果在正式挖矿，且 clique 共识设置为「只有有新交易时才出块」时，才会提交新的出块任务（`worker.mainLoop`）
- 如果没有正式挖矿，不会设置新区块的 `Header.Coinbase` 字段（`worker.commitNewWork`）



### 空块
miner模块是允许出空块的。可能提交空块的代码有两处，一处在 `worker.commitNewWork` 中：
```go
func (w *worker) commitNewWork(interrupt *int32, noempty bool, timestamp int64) {
  ......
  if !noempty {
    // Create an empty block based on temporary copied state for sealing in advance without waiting block
    // execution finished.
    w.commit(uncles, nil, false, tstart)
  }
  ......
}
````

另一处在处理 `chainSideCh` 消息时：
```go
case ev := <-w.chainSideCh:
  ......
  w.commit(uncles, nil, true, start)
```

处理 `chainSideCh` 消息这里之所以有可能提交空块，是因为在调用 `w.commit` 时可能没有新的 transactions 存在，但新的出块任务仍然会被直接提交。注意这里只是一种可能性，而上面 `worker.commitNewWork` 中提交的出块任务肯定是一个空块。

通过研究代码查找哪里使用 `noempty` 值为 false 作为参数调用了 `worker.commitNewWork` ，结合对 `chainSideCh` 的处理，我们可以总结出来，空块的来源有这几个地方：
- 处理 `chainSideCh` 时可能提交空块
- 处理 `startCh` 消息时
- 处理 `chainHeadCh` 消息时
- 处理 `txsCh` 消息时

需要说明一下的是，这里只是提交空块任务，不是肯定就能出一个空块。比如在 `worker.commitNewWork` 中，提交完空块任务后，会在函数结尾再次调用 `worker.commit` 提交一个出块任务，这个块肯定不是空块。如果此时刚才的空块还没有被签名，就会中断空块的签名，转而对当前这个非空的区块签名。

在知道了哪些情况下会提交空块任务后，这里我想多探讨一下，为什么要允许出空块呢？按说空块不包含任何交易数据，还浪费链的高度和磁盘的存储空间，这怎么看都很不合适。

这里要分两种情况讨论。第一种是处理 `chainSideCh` 时的情况。此时不检查是否有新的 transactions，而是直接提交出块任务。这里其实是为了鼓励矿工产出带有叔块的区块。此时可以不需要包含 transactions。（至于为什么鼓励产出带有叔块的区块，我们后面会讲到）

第二种是处理上面的另外三种消息时。此时在调用 `worker.commitNewWork` 的 `noempty` 参数都是 false，也就是说下面这个分支肯定会被执行：
```go
func (w *worker) commitNewWork(interrupt *int32, noempty bool, timestamp int64) {
  ......
  if !noempty {
    // Create an empty block based on temporary copied state for sealing in advance without waiting block
    // execution finished.
    w.commit(uncles, nil, false, tstart)
  }
  ......
}
````

而此时还没有统计 transactions 信息呢。根据代码里的注释和[这个提问](https://github.com/ethereum/go-ethereum/issues/18960)的后面的答案，原因应该是可以提高挖矿效率。当然[这里](https://ethereum.stackexchange.com/questions/9045/why-does-ethereum-creates-a-new-block-without-even-a-single-transaction/9058)还有一种观点是说允许出空块可以让区块链不断的增长，即使没有新增交易。这样可以维护区块链的安全，原因是如果区块链增长得很慢使得很长一段时间链的高度都较低，那么想要改变之前的区块只需较少的算力；而允许出空块后使区块链不断增长，高度越高就越难更改以前的区块。

### 叔块
读者可能已经注意到 `chainSideCh` 这个消息。在收到这个消息时，会将收到的区块加到 uncle 列表中，并从中选两个作为「叔块」，提交新的出块任务。那么什么是叔块，叔块的功能又是什么呢？

顾名思义，叔块就是当前区块的「叔叔辈」的区块，它被保存在 `Header.UncleHash` 这个字段中。

根据[这里](https://github.com/ethereum/wiki/wiki/Design-Rationale#uncle-incentivization)的描述，当我们试图减少 PoW 共识的出块间隔时，必然导致大量的区块冲突（即 A 与 B 同时挖到了同一高度的区块）。如果丢弃这些冲突的区块，就只能造成算力的浪费。另外这种情况也会导致算力的集中化：如果一个人有全网 10% 的算力，那么很有可能他有 90% 的时间是在浪费算力的（产生的块不被确认）；而另一个人有全网 30% 的算力，那么很有可能他有 70% 的时间是在浪费算力的，可见算力越大浪费越少，这相当于鼓励矿工加大自己的算力，最好自己拥有全网所有算力。

为了应对第一种情况，GHOST 协议将那些未进入主链的区块（stale blocks）也加入到「最长链」的计算当中；为了解决第二种情况，以太坊给那些未进入主链的区块的生产者一定的奖励，使它们的算力不至于被浪费那么多。

因此在以太坊中，「主链」的认定就不是哪个链的高度高，而是哪个链的「 Difficulty」 总值大，这其中当然包含链上的叔块的「 Difficulty 」值（具体的计算方法可以参看[这篇文章](https://yangzhe.me/2019/02/14/ethereum-ethash-theory/)和[这篇文章](https://yangzhe.me/2019/02/18/ethereum-ethash-code/)，或者可以自己查看 `HeaderChain.GetTd` 方法，和 `BlockChain.writeBlockWithState` 中关于 `externTd` 和 `localTd` 两个变量的代码）。并且以太坊给包含叔块的区块一些奖励，鼓励矿工将叔块包含进「主链」中，因此主链的「 Difficulty 」值会更大，更安全。

注意在以太坊中，一个区块的叔块与这个区块之间，高度差不能超过 7， miner 模块中的以下代码可以证明：
```go
func (w *worker) commitNewWork(interrupt *int32, noempty bool, timestamp int64) {
  ......

  uncles := make([]*types.Header, 0, 2)
  commitUncles := func(blocks map[common.Hash]*types.Block) {
    // Clean up stale uncle blocks first
    for hash, uncle := range blocks {
      if uncle.NumberU64()+staleThreshold <= header.Number.Uint64() {
        delete(blocks, hash)
      }
    }
    ......
  }
  commitUncles(w.localUncles)
  commitUncles(w.remoteUncles)

  ......
}
```
`commitUncles` 用来在生成新的区块时，选择有效的 uncle 区块，其中 `blocks` 参数就是备选的 uncle 区块。在这个函数里，首先做的就是将高度差大于 `staleThreshold` 的备选 uncle 区块删除掉。而 `staleThreshold` 的值就是 7。


### 动态调整出块频率
这里的出块频率，是指 `timer` 的频率，因此这里说的调整频率也是指调整 `timer` 的时间间隔。因为除了 `startCh` 事件，其它出块事件基本是当前模块无法控制的，因此也谈不上频率。

`timer` 的时间间隔是通过两种方式调整的：一种是通过 `resubmitIntervalCh` 消息；另一种是通过 `resubmitAdjustCh` 消息。

`resubmitIntervalCh` 是由 `Miner.SetRecommitInterval` 触发的，这是一个对外导出的方法，因此外面的模块可以通过这个方法调整 `timer` 的时间间隔。

`resubmitAdjustCh` 是由 `worker` 对象的代码自己触发的。在 `worker.newWorkLoop` 中， `timer` 定时器会定时提交一个出块任务，如果提交时有其它任务在执行，旧的任务就会因为 `timer` 提交的这个任务被中断（`commitInterruptResubmit`）。此时，被中断的任务就会认为 `timer` 的间隔时间太短了，从而通过 `resubmitAdjustCh` 来增加时间间隔。但如果旧任务一直没有被中断，那么它会认为timer时间足够长，可以缩短一些了。具体代码如下：
```go
func (w *worker) commitTransactions(txs *types.TransactionsByPriceAndNonce, coinbase common.Address, interrupt *int32) bool {
  ......

  //任务执行过程中被中断
  if interrupt != nil && atomic.LoadInt32(interrupt) != commitInterruptNone {
    // Notify resubmit loop to increase resubmitting interval due to too frequent commits.
    if atomic.LoadInt32(interrupt) == commitInterruptResubmit {
      ratio := float64(w.current.header.GasLimit-w.current.gasPool.Gas()) / float64(w.current.header.GasLimit)
      if ratio < 0.1 {
        ratio = 0.1
      }
      w.resubmitAdjustCh <- &intervalAdjust{
        ratio: ratio,
        inc:   true,
      }
    }
    return atomic.LoadInt32(interrupt) == commitInterruptNewHead
  }
  ......

 //函数最后，任务一直没有被中断，发通知可以缩短一下时间间隔
  if interrupt != nil {
    w.resubmitAdjustCh <- &intervalAdjust{inc: false}
  }
  return false
}
```


### 出块动作序列化
前面我们已经讨论过， `worker` 中有多个出块时机，且它们都是等待事件，随时可能同时发生。但即便如此，最后的签名任务却是一个一个进行的，而非同时签名多个区块。这里我们稍微讨论一下这个问题。

在所有的消息中，提交出块任务只有两种方式：调用 `worker.commitNewWork` 或 `worker.commit` 。且 `worker.commitNewWork` 最终也会调用 `worker.commit` 。因此我们只要看看这两个函数是怎么做的就可以了。

在 `worker.commitNewWork` 函数的一开始，就是加锁的代码，因此这个函数实际上是一个单入函数，不能并行执行。

在 `worker.commit` 中，所有的任务都是提交到 `taskCh` 这个 channel 中。而这个 channel 恰好是一次只能容纳一个对象，因此这里通过 channel 这个 go 语言里的对象实现了序列化执行。


# 其它实现细节

### 中断上一个出块任务
在处理`startCh`、`chainHeadCh`、`timer.C`这三个消息时，都会有一个 `interrupt` 变量，其可能取值为：
- commitInterruptNone
- commitInterruptNewHead
- commitInterruptResubmit

这个过程的简化代码如下：
```go
func (w *worker) newWorkLoop(recommit time.Duration) {
  var interrupt   *int32

  commit := func(noempty bool, s int32) {
    if interrupt != nil {
      atomic.StoreInt32(interrupt, s)
    }
    interrupt = new(int32)
    w.newWorkCh <- &newWorkReq{interrupt: interrupt, noempty: noempty, timestamp: timestamp}
  }


  for {
    select {
    case <-w.startCh:   
      commit(false, commitInterruptNewHead)
    case head := <-w.chainHeadCh:
      commit(false, commitInterruptNewHead)
    case <-timer.C:
      commit(true, commitInterruptResubmit)
    }
    .....
  }
}
```

由于 `commit` 是一个局部匿名函数，因此处理这三个消息时使用了同一个变量 `interrupt` ，注意它是一个指针类型的整数。每次调用 `commit` 时，如果之前的 `interrupt` 变量不为空，就设置相应的值；然后更新 `interrupt` 指向一个新的 int 对象（此时 `interrupt` 指向的值为 commitInterruptNone）。所以可以总结 `commit` 的行为是：修改旧对象的值，然后指向新的对象。

那么设置旧的对象的值有什么用呢？如果 `interrupt` 变量不为nil，那么一定有一个任务是在执行过程中的，我们看一下相关的代码：
```go
func (w *worker) commitTransactions(txs *types.TransactionsByPriceAndNonce, coinbase common.Address, interrupt *int32) bool {
  ......
  for {
    if interrupt != nil && atomic.loadint32(interrupt) != commitinterruptnone {
      // Notify resubmit loop to increase resubmitting interval due to too frequent commits.
      if atomic.LoadInt32(interrupt) == commitInterruptResubmit {
        ratio := float64(w.current.header.GasLimit-w.current.gasPool.Gas()) / float64(w.current.header.GasLimit)
        if ratio < 0.1 {
          ratio = 0.1
        }
        w.resubmitAdjustCh <- &intervalAdjust{
        ratio: ratio,
        inc:   true,
        }
      }
      return atomic.LoadInt32(interrupt) == commitInterruptNewHead
    }
    ......
  }
  ......
}
```

在上面的 for 循环里，如果 `interrupt` 被设置为除 commitInterruptnone 以外的值，就会中断当前函数的执行。且如果其值为 commitInterruptNewHead ，就会中断整个提交任务（参看 `worker.commitNewWork` 中对 `worker.commitTransactions` 返回值的处理）。因此这个变量的实际意义也就在此：通过指针方式共享同一个变量，然后判断其值来决定是否应该中断任务。

### 中断上一个签名任务
前面我们介绍过，区块的签名是一个一个进行的，并非多个区块同时进行。并且区块的签名是异步的（参看[介绍ethash的文章](https://yangzhe.me/2019/02/18/ethereum-ethash-code/)）。所以前一个区块正在异步进行签名时，来了一个新的区块需要签名，就需要中断之前区块的签名过程。

这一动作是和共识接口的 `Seal` 方法合作完成的。具体来说，在调用 `Seal` 时会传入一个 `stopCh` channel；而在 `Seal` 方法中，需要监听这个 channel，如果被关闭就立即退出签名。

而在当前的 worker 模块中，如果新来一个区块需要签名，就关闭 `stopCh` 这个 channel 就可以了。

具体代码如下：
```go
func (w *worker) taskLoop() {  
  var stopCh chan struct{}

  interrupt := func() {
    if stopCh != nil {
      close(stopCh)
      stopCh = nil
    }
  }

  ......

  for {
    select {
    case task := <-w.taskCh:
      ......

      interrupt()
      stopCh, prev = make(chan struct{}), sealHash

      ......
      if err := w.engine.Seal(w.chain, task.block, w.resultCh, stopCh); err != nil {
        log.Warn("Block sealing failed", "err", err)
      }
    }
  }
}
```

### gas
在 `worker.commitTransactions` 的代码中，我们可以看到 `gas` 相关的操作。本篇文章中其实不会涉及 gas 相关的细节，但为了文章的完整性，这里就对 gas 简单进行一个介绍。

在以太坊中，如果你想发起一个合约，那么矿工在执行你这个合约的过程中，是需要消耗资源的。而以太坊的合约又是图灵完备的，因此可能很短的语句需要消耗大量资源（比如一个死循环）。这种情况下每个合约使用固定的费用或根据合约字节大小来决定合约的费用，都是不合适的。因此在执行合约的过程中，每执行一步都要扣除合约发起者一定量的费用，这个费用就被称为 `gas`。

`gas` 是一个独立的计费单位，不是以太币的单位。但 `gas` 却是有价格的，每个合约中合约的发起者可以约定每个gas值多少个以太币。这就好像汽车要跑起来烧的是汽油却不是人民币，但汽油确是以人民币计价的，并且每个加油站的价格可能稍有不同。

如果一个合约在执行完些之前 gas 就被消耗完了，那么这个合约就会被立即中断执行。这会当然会导致合约完成不了，但已民经消耗的 gas 却不会被退回，因为矿工确实为了执行你的合约消耗了资源。

注意在每一个区块中可以消耗的gas总数是有限的，最大值为 `Header.GasLimit` 。计算这个字段的值的函数为 `core.CalcGasLimit` ，这里我们就不详细说明了。

# 总结
本篇文章中我们介绍了以太坊的 miner 模块。文章主要总结了以太坊的 miner 模块实现了哪功能，且这些功能是如体现在代码中的。我觉得关注的重在于一个 miner 模块所实现的功能，而不必纬结于以太坊是如何实现的。

如果您发现文章中有错误的地方，非常感谢能够指正。
