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
所有的区块链项目中，“挖矿”都是非常重要的一项功能。前面几篇文章中我们介绍了以太坊中挖矿的两种共识实现模块：clique和ethash。那么这篇文章里，我们介绍另一个与挖矿紧密相关的模块miner。

不要被miner这个模块名字搞糊涂了：miner是“矿工”的意思，但这个模块的功能，更像是现实中一个矿上的“调度室”，它管理着什么时候开始挖矿、什么时候开始挖一个新的区块等。

分析以太坊的miner模块的代码，我觉得最重要的是学习它作为一个“调度器”都实现了哪些功能，以便我们自己写一个类似的功能的时候，可以有的放矢，知道大约需要具备什么功能。因此本篇文章的思路是首先从整体上看一下整个模块的功能结构，然后再对一些较重要又较难理解的问题进行一下详细的说明。


# 主要功能
miner模块只有两个重要的结构体：Miner和worker。

Miner的功能比较简单，主要就是控制挖扩工作的开启（Start）和停止（Stop）。但Miner有一个特性，会在同步区块时暂停挖矿，等区块同步完成后再视情况重新启动挖矿（后面会详细说明）。

worker对象可以理解成“工人”，具体的“调度”工作都是由worker对象完成的。worker中每一项工作的开始，都是由一些消息触发的，这些消息包含在几个消息循环函数中。下面我们简单说明一下这些消息。

- newWorkLoop
  - startCh  
  此消息在调用`newWorker`函数和调用`worker.start`方法时都被被触发。收到此消息后，发送消息给newWorkCh，以发起一次新的出块工作。
  - chainHeadCh  
  此消息在有新的区块被加入到本地的区块数据库时被触发。因此这个消息不是本模块触发的，而是blockchain模块触发的。此消息的处理与startCh类似，也主要是发送消息给newWorkCh。
  - timer.C  
  此消息是一个定时器消息，每隔一段时间都会发消息给newWorkCh。间隔时间由生成worker对象时的参数`recommit`决定（但resubmitIntervalCh和resubmitAdjustCh消息可以对其作出修改）。
  - resubmitIntervalCh  
  此消息由`Miner.SetRecommitInterval`方法触发。收到消息后，会更新timer的间隔时间。
  - resubmitAdjustCh  
  此消息由本模块其它地方触发。在出块过程中（具体是在commitTransactions时），可能会发送此消息，以要求调整`timer.C`的间隔时间。

这里需要注意一下的是，生成worker对象的时候发送了一个消息给startCh，因此即使没有start方法，worker也会开始工作，只是不会调用共识模块的`Seal`方法进行签名而已。

- mainLoop
  - newWorkCh  
  此消息由newWorkLoop中处理startCh、chainHeadCh、timer.C消息时触发。这个消息的处理很简单，就是调用worker.commitNewWork提交一个新的挖矿工作。
  - chainSideCh  
  此消息在有新的侧链上的区块被加入到本地区块数据库时被触发。收到此消息后，将消息中的区块加入到本地的“叔块”列表中，并可能选取两个叔块提交一个出块任务。
  - txsCh  
  此消息在收到新的transaction时被触发。收到此消息后，会根据当前状态，要么保存transactions，要么提交一个出块任务。

- taskLoop
  - taskCh  
  此消息在`worker.commit`中触发。收到此消息后，会调用共识接口中的`Seal`方法生成一个合法的区块。

- resultLoop
  - resultCh  
  当共识接口中的`Seal`方法成功生成一个合法区块时，就会将区块发送到这里。收到此消息后，会将区块写入本地数据库中，且发送`core.NewMinedBlockEvent`事件广播这个新挖到的区块。


以上就是`worker`对象中所有的重要的消息事件。其实通读`Miner`和`worker`对象的代码，我们可以总结出来，以太坊的这个“调度器”主要实现了这些功能：
1. 同步区块时暂停挖矿：在收到同步区块的消息后，要暂停挖矿（否则挖到的很可能不是主链上的区块），待同步成功后再继续。但这个过程中只做一次，以防止DOS攻击。
2. 多个出块时机：在收到start消息(startCh)、新区块插入的消息(chainHeadCh)、侧链上区块插入的消息(chainSideCh)、收到新的交易时(txsCh)，都会视情况着手生成区块。其中侧链上的区块不管有没有新的transaction，都会生成一个区块。这是以太坊在鼓励矿工包含叔块；收到新的交易时，只有是clique共识且period为0（clique的period为0代表只有有新交易时才生成区块，其它任何时候都不生成）时，才生成区块，否则只是将新的交易加入到待出块的信息中。
3. 随时准备出块：即使调用者没有调用start方法正式开始挖矿，只要创建了worker对象，就一直在准备挖矿。
4. 可以出空块：在某些情况下，即使没有新的transaction，也可以出块。
5. 叔块：在收到侧链的上区块时，鼓励矿工引用这些区块作为叔块。
6. 出块频率可动态调整：这主是要指timer中的recommit功能。
7. 出块动作序列化：虽然有多个源头可以发起一次生成区块的行为，但最后都会将这些行为按先后顺序一个一个进行，而且后来的会中断之前的。

下面我们就逐一详细说明一下这些个功能。

# 功能分析

### 同步区块时暂停挖矿
`Miner`对象代表了整个miner模块的功能。在创建`Miner`时（miner.New），会一起创建一个`Miner.update`线程。这个线程会等待区块更新的事件。如果更新模块开始对区块进行更新，则暂时停止挖矿；等待区块更新完成时（无论成功或失败），再视情况重新启动挖矿。

下面是`Miner`的功能图：
![](/pic/ethereum-miner/miner-flowchart.png)

在启动和停止挖矿的过程中，有两变量起到重要的协调作用：`canStart`和`shouldStart`。其中`canStart`用来控制是否能立即启动挖矿；而`shouldStart`用来控制在区块同步结果时，是否应该重新启动挖矿。

这里需要注意一下的是，在接收到区块同步完成的消息并处理后，Miner.update线程就退出了，虽然downloader模块还有可能再次抛出开始同步的消息。根据注释中的说明，这主要是为了防止DOS攻击：如果Miner.update一直在接收开始区块同步的消息并停止挖矿，那么别人就可以不停地给你发新的区块（不一进是合法的），造成你一直无法启动挖矿。

### 多个出块时机
多个出块时机，其实指的就是前面提到的各种随时监听的消息。这些消息主要分两类：内部的和外部的。这里我们把这些可以引发出块的消息和意义再罗列一下。
- 内部消息：
  - `startCh`: worker对象启动
  - `timer.C`: 内部定时器，每隔一定时间尝试提交一个出块任务
- 外部消息：
- `chainHeadCh`: 有新的区块加入主链
- `chainSideCh`: 有新的区块加入侧链
- `txsCh`: 有新的transactions

两个内部消息中，`startCh`在创建`worker`对象时就会被触发一次。而在处理`startCh`时，又会设置`timer`定时器（但是在调用`worker.start`正式启动挖矿之前，定时器实际上什么工作也不做的，详细参看后面的讨论）。

而外部消息就不受控制了，随时可能被触发。（后面说的“随时准备出块”其实也主要指这三个外部消息）


### 随时准备出块
在miner模块的实现中，只要创建了`Miner`对象，就一直处于“随时准备着”的状态。

刚才我们已经分过析`worker`中有多个出块时机。这些时机的触发并不需要调用`worker.start`方法，只要生成了`worker`对象就可能被触发。因此即使没有调用`worker.start`方法正式挖矿，在那些出块时机到来时，也会将收集到的消息放到`worker.current`变量中。

随时准备出块的原因，一是因为在生成`worker`对象时，发送了消息给`startCh`：
```go
func newWorker(config *params.ChainConfig, engine consensus.Engine, eth Backend, mux *event.TypeMux, recommit time.Duration, gasFloor, gasCeil uint64, isLocalBlock func(*types.Block) bool) *worker {
  ......

  // Submit first work to initialize pending state.
  worker.startCh <- struct{}{}
}
```

收到`startCh`消息后，会走一个出块的流程（虽然会忽略一些正式出块的情况下才会执行的代码），因此就把一些出块相关的字段初始化了（最重要的，比如`worker.current`）。处理`startCh`消息的代码如下：
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

二是因为在监听三个“外部消息”：chainHeadCh、chainSideCh、txsCh，它们随时可能会被触发，触发后就马上需要作相应的处理进行出块或出块准备。

我们刚才提到“忽略一些正式出块的情况一下才会执行的代码”，那哪些是在“准备”阶段会被略过、而正式出块时才会执行的代码呢？

##### 准备出块时被略过的代码
调用`worker.start`与否的区别，在于`worker.running`的值不同。这个字段标志着是否在正式挖矿中，在代码中由`worker.isRunning`来读取`running`字段并返回判断结果。因此我们来看一下调用`worker.isRunning`的几处代码。

首先是`worker.commit`中的代码：
```go
func (w *worker) commit(uncles []*types.Header, interval func(), update bool, start time.Time) error {
  if w.isRunning() {
    ......  
    case w.taskCh <- &task{receipts: receipts, state: s, block: block, createdAt: time.Now()}:
    ......
  }
}
```

这里的判断是最重要的，只有在正式挖矿时，才会提交任务给`taskCh`。在`taskLoop`中收到`taskCh`消息后才会调用共识接口的`Seal`方法对区块进行签名。

然后是`newWorkLoop`中的代码：
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

可以看到在定时器`timer`中，如果没有正式挖矿，是不会提交新的挖矿任务的。也就是说在调用`worker.start`之前，定时器是无效的。

接下来是`mainLoop`中的代码：
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

这里面`chainSideCh`和`txsCh`都会判断是否在正式挖矿。在收到侧链消息chinSideCh中，只有在正式挖矿时才会将侧链区块提交到`current`字段中，且调用`worker.commit`提交一个新的出块任务。

下面是`commitNewWork`中的代码：
```go
func (w *worker) commitNewWork(interrupt *int32, noempty bool, timestamp int64) {
  ......
  if w.isRunning() {
    header.Coinbase = w.coinbase
  }
  ......
}
```

可见只有在正式挖矿时，才会设置`header.Coinbase`字段（这个字段标识挖矿奖励该给谁）。

下面是`commitTransactions`方法的代码：
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
- 如果没有正式挖矿，是无论如何不会调用共识接口的`Seal`方法的（`worker.commit`）
- 如果没有正式挖矿，timer定时器是无效的（`worker.newWorkLoop`）
- 如果没有正式挖矿，在收到侧链区块时不会提交带叔块的当前区块（`worker.mainLoop`）
- 如果没有正式挖矿，在收到新的transactions时只会将其提交到`worker.current`中，不会提交出块任务；如果在正式挖矿，且clique共识设置为“只有有新交易时才出块”时，才会提交新的出块任务（`worker.mainLoop`）
- 如果没有正式挖矿，不会设置新区块的`Header.Coinbase`字段（`worker.commitNewWork`）



### 可以出空块
miner模块是允许出空块的。可能提交空块的代码有两处，一处在`worker.commitNewWork`中：
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

另一处在处理`chainSideCh`消息时：
```go
case ev := <-w.chainSideCh:
  ......
  w.commit(uncles, nil, true, start)
```

处理`chainSideCh`消息这里之所以有可能提交空块，是因为在调用`w.commit`时可能没有新的transactions存在，但新的出块任务仍然会被直接提交。注意这里只是一种可能性，而上面`worker.commitNewWork`中提交的出块任务肯定是一个空块。

通过研究代码查找哪里使用`noempty`值为`false`作为参数调用了`worker.commitNewWork`，结合对`chainSideCh`的处理，我们可以总结出来，空块的来源有这几个地方：
- 处理`chainSideCh`时可能提交空块
- 处理`startCh`消息时
- 处理`chainHeadCh`消息时
- 处理`txsCh`消息时

需要说明一下的是，这里只是提交空块任务，不是肯定就能出一个空块。比如在`worker.commitNewWork`中，提交完空块任务后，会在函数结尾再次调用`worker.commit`提交一个出块任务，这个块肯定不是空块。如果此时刚才的空块还没有被签名，就会中断空块的签名，转而对当前这个非空的区块签名。

在知道了哪些情况下会提交空块任务后，这里我想多探讨一下，为什么要允许出空块呢？按说空块不包含任何交易数据，还浪费链的高度和磁盘的存储空间，这怎么看都很不合适。

这里要分两种情况讨论。第一种是处理`chainSideCh`时的情况。此时不检查是否有新的transactions，而是直接提交出块任务。这里其实是为了鼓励矿工产出带有叔块的区块。此时可以不需要包含transactions。（至于为什么鼓励产出带有叔块的区块，我们后面会讲到）

第二种是处理上面的另外三种消息时。此时在调用`worker.commitNewWork`的`noempty`参数都是false，也就是说下面这个分支肯定会被执行：
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

而此时还没有统计transactions信息呢。根据代码里的注释和[这个提问](https://github.com/ethereum/go-ethereum/issues/18960)的后面的答案，原因应该是可以提高挖矿效率。当然[这里](https://ethereum.stackexchange.com/questions/9045/why-does-ethereum-creates-a-new-block-without-even-a-single-transaction/9058)还有一种观点是说允许出空块可以让区块链不断的增长，即使没有新增交易。这样可以维护区块链的安全，原因是如果区块链增长得很慢使得很长一段时间链的高度都较低，那么想要改变之前的区块只需较少的算力；而允许出空块后使区块链不断增长，高度越高就越难更改以前的区块。

### 叔块
读者可能已经注意到`chainSideCh`这个消息。在收到这个消息时，会将收到的区块加到uncle列表中，并从中选两个作为“叔块”，提交新的出块任务。那么什么是叔块，叔块的功能又是什么呢？

顾名思义，叔块就是当前区块的“叔叔辈”的区块，它被保存在`Header.UncleHash`这个字段中。

根据[这里](https://github.com/ethereum/wiki/wiki/Design-Rationale#uncle-incentivization)的描述，当我们试图减少PoW共识的出块间隔时，必然导致大量的区块冲突（即A与B同时挖到了同一高度的区块）。如果丢弃这些冲突的区块，就只能造成算力的浪费。另外这种情况也会导致算力的集中化：如果一个人有全网10%的算力，那么很有可能他有90%的时间是在浪费算力的（产生的块不被确认）；而另一个人有全网30%的算力，那么很有可能他有70%的时间是在浪费算力的，可见算力越大浪费越少，这相当于鼓励矿工加大自己的算力，最好自己拥有全网所有算力。

为了应对第一种情况，GHOST协议将那些未进入主链的区块（stale blocks）也加入到“最长链”的计算当中；为了解决第二种情况，以太坊给那些未进入主链的区块的生产者一定的奖励，使它们的算力不至于被浪费那么多。

因此在以太坊中，“主链”的认定就不是哪个链的高度高，而是哪个链的“Difficulty”总值大，这其中当然包含链上的叔块的“Difficulty”值（具体的计算方法我们以后的文章里再细说，有兴趣的读者可以自己查看`HeaderChain.GetTd`方法，和`BlockChain.writeBlockWithState`中关于`externTd`和`localTd`两个变量的代码）。并且以太坊给包含叔块的区块一些奖励，鼓励矿工将叔块包含进“主链”中，因此主链的“Difficulty”值会更大，更安全。

### 出块频率可动态调整
这里的出块频率，是指`timer`的频率，因此这里说的调整频率也是指调整`timer`的时间间隔。因为除了`startCh`事件，其它出块事件基本是当前模块无法控制的，因此也谈不上频率。

`timer`的时间间隔是通过两种方式调整的：一种是通过`resubmitIntervalCh`消息；另一种是通过`resubmitAdjustCh`消息。

`resubmitIntervalCh`是由`Miner.SetRecommitInterval`触发的，这是一个对外导出的方法，因此外面的模块可以通过这个方法调整`timer`的时间间隔。

`resubmitAdjustCh`是由`worker`对象的代码自己触发的。在`worker.newWorkLoop`中，`timer`定时器会定时提交一个出块任务，如果提交时有其它任务在执行，旧的任务就会因为`timer`提交的这个任务被中断（commitInterruptResubmit）。此时，被中断的任务就会认为`timer`的间隔时间太短了，从而通过`resubmitAdjustCh`来增加时间间隔。但如果旧任务一直没有被中断，那么它会认为timer时间足够长，可以缩短一些了。具体代码如下：
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
前面我们已经讨论过，`worker`中有多个出块时机，且它们都是等待事件，随时可能同时发生。但即便如此，最后的签名任务却是一个一个进行的，而非同时签名多个区块。这里我们稍微讨论一下这个问题。

在所有的消息中，提交出块任务只有两种方式：调用`worker.commitNewWork`或`worker.commit`。且`worker.commitNewWork`最终也会调用`worker.commit`。因此我们只要看看这两个函数是怎么做的就可以了。

在`worker.commitNewWork`函数的一开始，就是加锁的代码，因此这个函数实际上是一个单入函数，不能并行执行。

在`worker.commit`中，所有的任务都是提交到`taskCh`这个channel中。而这个channel恰好是一次只能容纳一个对象，因此这里通过channel这个go语言里的对象实现了序列化执行。


# 其它实现细节

### 中断上一个块的出块过程
在处理`startCh`、`chainHeadCh`、`timer.C`这三个消息时，都会有一个`interrupt`变量，其可能取值为：
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

由于`commit`是一个局部匿名函数，因此处理这三个消息时使用了同一个变量`interrupt`，注意它是一个指针类型的整数。每次调用`commit`时，如果之前的`interrupt`变量不为空，就设置相应的值；然后更新`interrupt`指向一个新的int对象（此时`interrupt`指向的值为commitInterruptNone）。所以可以总结`commit`的行为是：修改旧对象的值，然后指向新的对象。

那么设置旧的对象的值有什么用呢？如果`interrupt`变量不为nil，那么一定有一个任务是在执行过程中的，我们看一下相关的代码：
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

在上面的for循环里，如果`interrupt`被设置为除commitInterruptnone以外的值，就会中断当前函数的执行。且如果其值为commitInterruptNewHead，就会中断整个提交任务（参看`worker.commitNewWork`中对`worker.commitTransactions`返回值的处理）。因此这个变量的实际意义也就在此：通过指针方式共享同一个变量，然后判断其值来决定是否应该中断任务。

### 中断上一个块的签名过程
前面我们介绍过，区块的签名是一个一个进行的，并非多个区块同时进行。并且区块的签名是异步的（参看[介绍ethash的文章](https://yangzhe.me/2019/02/18/ethereum-ethash-code/)）。所以前一个区块正在异步进行签名时，来了一个新的区块需要签名，就需要中断之前区块的签名过程。

这一动作是和共识接口的`Seal`方法合作完成的。具体来说，在调用`Seal`时会传入一个`stopCh`channel；而在`Seal`方法中，需要监听这个channel，如果被关闭就立即退出签名。

而在当前的worker模块中，如果新来一个区块需要签名，就关闭`stopCh`这个channel就可以了。

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
在`worker.commitTransactions`的代码中，我们可以看到`gas`相关的操作。本篇文章中其实不会涉及gas相关的细节，但为了文章的完整性，这里就对gas简单进行一个介绍。

在以太坊中，如果你想发起一个合约，那么矿工在执行你这个合约的过程中，是需要消耗资源的。而以太坊的合约又是图灵完备的，因此可能很短的语句需要消耗大量资源（比如一个死循环）。这种情况下每个合约使用固定的费用或根据合约字节大小来决定合约的费用，都是不合适的。因此在执行合约的过程中，每执行一步都要扣除合约发起者一定量的费用，这个费用就被称为`gas`。

`gas`是一个独立的计费单位，不是以太币的单位。但`gas`却是有价格的，每个合约中合约的发起者可以约定每个gas值多少个以太币。这就好像汽车要跑起来烧的是汽油却不是人民币，但汽油确是以人民币计价的，并且每个加油站的价格可能稍有不同。

如果一个合约在执行完些之前gas就被消耗完了，那么这个合约就会被立即中断执行。这会当然会导致合约完成不了，但已民经消耗的gas却不会被退回，因为矿工确实为了执行你的合约消耗了资源。

注意在每一个区块中可以消耗的gas总数是有限的，最大值为`Header.GasLimit`。计算这个字段的值的函数为`core.CalcGasLimit`，这里我们就不详细说明了。

# 总结
本篇文章中我们介绍了以太坊的miner模块。文章主要总结了以太坊的miner模块实现了哪功能，且这些功能是如体现在代码中的。我觉得关注的重在于一个miner模块所实现的功能，而不必纬结于以太坊是如何实现的。

如果您发现文章中有错误的地方，非常感谢能够指正。
