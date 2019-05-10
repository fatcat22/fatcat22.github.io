---
layout: post
title:  "以太坊源码解析：downloader/queue"
categories: ethereum
tags: 原创 ethereum downloader queue 源码解析
excerpt: downloader/queue 对象是以太坊区块同步模块的一个对象，想要完全理解以太坊的区块同步，downloader/queue 是一个关键点。
author: fatcat22
---

* content
{:toc}





>本篇文章分析的源码地址为：[https://github.com/ethereum/go-ethereum](https://github.com/ethereum/go-ethereum)  
>分支：[master](https://github.com/ethereum/go-ethereum/tree/master)  
>commit id: [257bfff316e4efb8952fbeb67c91f86af579cb0a](https://github.com/ethereum/go-ethereum/tree/257bfff316e4efb8952fbeb67c91f86af579cb0a)  


# 引言
`queue` 对象是 downlaoder 模块的一个辅助对象，它的主要目的，是记录所需下载区块的各种信息，以及将分开下载的各区块信息（header，body，receipt等）组成完整的区块。`queue` 对象与 `Downloader`  对象紧密关联，共同完成了区块下载的逻辑。它也是完全理解 downloader 模块、尤其是 `Downloader.fetchParts` 的关键。因此我们专门用一篇文章来详细分析一下 `queue` 对象，

`queue` 对象是 downloader 模块的一个内部对象，只有 `Downloader` 对象使用了它。下面我们会先从整体上了解一下 `queue` 提供的功能，然后详细分析一下 `queue` 的内部实现。


# 有哪些功能

根据 `queue` 对象的使用方式，我将它的功能分为三类，因此下面我们按大的类型分别看一下 `queue` 对象的所有功能。

在下载正式发起之前，以及数据真正下载之前，`Downloader` 对象会调用 `queue` 的一些方法，对其进行初始化，或将需要下载的数据信息告诉 `queue`。我们先来看一下这类功能：
- Prepare  
`queue.Prepare` 方法用来在下载开始之前，告诉 `queue` 对象将要下载的一系列区块的起始高度和下载模式（fast 或 full 模式）。
- ScheduleSkeleton  
`queue.ScheduleSkeleton` 用来在填充 skeleton 之前，使用 skeketon 的信息对 `queue` 对象进行初始化。
- Schedule  
`queue.Schedule` 方法用来准备对一些 body 和 receipt 数据的下载。在 `Downloader.processHeaders` 中处理下载成功的 header 时，使用这些 header 调用 `queue.Schedule` 方法，以便 `queue` 对象可以开始对这些 header 对应的 body 和 receipt 开始下载调度。

在数据的下载过程中，`Downloader` 对象会使用 `queue` 提供的一些信息来决定和判断数据的下载状态等信息。下面就是 `queue` 提供的这类功能：
- pending  
pending 功能用来告诉调用者还有多少条数据需要下载。提供此功能的方法有：`queue.PendingHeaders`、`queue.PendingBlocks`、`queue.PendingReceipts`
- inflight  
inflight 功能用来告诉调用者当前是否有数据正在被下载。提供此功能的方法有：`queue.InFlightHeaders`、`queue.InFlightBlocks`、`queue.InFlightReceipts`
- shouldThrottle  
shouldThrottle 功能用来告诉调用者是否该限制（或称为暂停）一下某类数据的下载，其目的是为了防止下载过程中本地内存占用过大。在 `Downloader.fetchParts`中向某节点发起获取数据请求之前，会进行这种判断。提供此功能的方法有：`queue.ShouldThrottleBlocks`、`queue.ShouldThrottleReceipts`
- reserve  
reserve 功能通过构造一个 `fetchRequest` 结构并返回，向调用者提供指定数量的待下载的数据的信息（`queue` 内部会将这些数据标记为「正在下载」）。调用者使用返回的 `fetchRequest` 数据向远程节点发起新的获取数据的请求。提供此功能的方法有：`queue.ReserveHeaders`、`queue.ReserveBodies`、`queue.ReserveReceipts`
- cancel  
cancel 功能与 reserve 相反，用来撤消对 `fetchRequest` 结构中的数据的下载（`queue` 内部会将这些数据重新从「正在下载」的状态更改为「等待下载」）。提供此功能的方法有：`queue.CancelHeaders`、`queue.CancelBodies`、`queue.CancelReceipts`
- expire  
通过在参数中指定一个时间段，expire 用来告诉调用者下载时间已经超过指定时间的节点 id 和超时的数据条数。提供此功能的方法有：`queue.ExpireHeaders`、`queue.ExpireBodies`、`queue.ExpireReceipts`
- deliver  
当有数据下载成功时，调用者会使用 deliver 功能用来通知 `queue` 对象。提供此功能的方法有：`queue.DeliverHeaders`、`queue.DeliverBodies`、`queue.DeliverReceipts`

在数据下载完成后，`Downloader` 对象会调用 `queue` 中的一些方法，获取下载并组装好的区块数据。这类功能有下面几个：
- RetrieveHeaders  
在填充 skeleton 完成后，`queue.RetrieveHeaders` 用来获取整个 skeleton 中的所有 header。
- Results  
`queue.Results` 用来获取当前的 header、body 和 receipt（只在 fast 模式下） 都已下载成功的区块（并将这些区块从 `queue` 内部移除）

上面的分类按从前到后的顺序，也基本是区块同步的流程和顺序。可以看出来，`queue` 是一个真正的工具类，它的方法的调用方式和顺序完全依赖于下载流程。清楚了 `queue` 各方法的调用时机，那么它的详细实现也就很好理解了。


# 实现分析

在了解了 `queue` 提供了功能以后，我们现在来看一下它内部的详细实现。注意在下面的分析中，我们是顺着区块同步过程中， `queue` 对象的各功能被调用的先后逻辑来进行的。

在下载开始时，`queue.Prepare` 首先被调用，用来设置下载模式和起始区块的高度。它的实现非常简单，为了完整性，只是在这里简单提一下。


### ScheduleSkeleton

接下来 `Downloader` 对象该开始填充 skeleton 了，因此在 `Downloader.fillHeaderSkeleton` 中，`queue.ScheduleSkeleton` 被调用。之前我们说过，以太坊在同步区块时，先确定要下载的区块的高度区间，然后将这个区间按 `MaxHeaderFetch` 切分成很多组，每一组的最后一个区块组成了 「skeleton」（最后一组不满 `MaxHeaderFetch` 个区块不算作一组，详细信息请参看[这篇文章](http://yangzhe.me/2019/05/09/ethereum-downloader/)中的「header 的下载和 skeleton」小节）。

例如，假设已确定需要下载的区块高度区间是从 10 到 46，`MaxHeaderFetch` 的值为 10，那么这个高度区块就会被分成 3 组：10 - 19，20 - 29，30 - 39，而 skeleton 则分别由高度为 19、29、39 的区块头组成。下面的分析中我们会继续使用这个例子中的数值，但不会重新说明，请读者注意。

我们继续看 `queue.ScheduleSkeleton`的实现。它的参数是要下载区块的起始高度和所有 skeleton 区块头。再次强调一下这些区块头是每一组的最后一个区块。此方法中的代码是对各个字段的初始化，非常简单，这里我们只说明一下最后的 for 循环：
```go
func (q *queue) ScheduleSkeleton(from uint64, skeleton []*types.Header) {
    ......
    for i, header := range skeleton {
        index := from + uint64(i*y)

        q.headerTaskPool[index] = header
        q.headerTaskQueue.Push(index, -int64(index))
    }
}
```
我们知道skeleton中每一组区块的个数是 `MaxHeaderFetch`，因此循环中的 `index` 变量实际上是每一组区块中的第一个区块的高度（比如 10、20、30），而我们刚才提到过 `skeleton` 参数中的区块头是每一组区块中的最后一个区块，因此 `queue.headerTaskPool` 实际上是一个**每一组区块中第一个区块的高度到最后一个区块的 header 的映射**。比如它的值看上去是这样的：
> headerTaskPool = {  
>   10: headerOf_19,  
>   20: headerOf_20,  
>   30: headerOf_39,  
> }

`queue.headerTaskQueue` 是一个优先级队列，每一项的值为每一组的第一个区块的高度，优先级为此高度的负数。因此在这个队列里，高度值越小优先级越高。这个字段里的值看上去是这样的：
> headerTaskQueue = {  
>   -10: 10,  
>   -20: 20,  
>   -30: 30,  
> }

接下来 `Downloader` 对象就要调用 `fetchParts` 填充 skeleton 了。`fetchParts` 的参数与 `queue` 对象结合非常紧密，几乎所有参数代表的回调函数都用到了 `queue` 的功能。而在 `fetchParts` 中获取数据主要依赖于 `capacity`、`reserve`、`fetch` 和 `deliver` 这四个参数，所以我们这里主要分析一下这几个回调函数。

`capacity` 参数代表的是某次下载可以请求的数据条数；`fetch` 参数则向远程节点发起数据请求；这俩参数的实现都与 `peerConnection` 对象有关，因此我们这里不进行详细分析，而只是关注 `reserve` 和 `deliver`。


### reserve header

`reserve` 参数用来获取可下载的数据，在 `Downloader.fillHeaderSkeleton` 中它的实现如下：
```go
reserve  = func(p *peerConnection, count int) (*fetchRequest, bool, error) {
    return d.queue.ReserveHeaders(p, count), false, nil
}
```
可见它是调用了 `queue.ReserveHeaders` 来实现这功能的，我们来看看这个方法的实现。但我们首先展示一下这个方法中用到的 `queue.headerPendPool` 字段的用处：
```go
func (q *queue) ReserveHeaders(p *peerConnection, count int) *fetchRequest {
    if _, ok := q.headerPendPool[p.id]; ok {
        return nil
    }

    ......

    q.headerPendPool[p.id] = request
    return request
}
```
`queue.headerPendPool` 这个字段用来记录正在下载的节点和数据信息。方法首先从这个字段中判断指定的远程节点（使用 id 标识）是否正在下载，如果是直接返回。而在方法的最后，则将新构造的 `request` 和 节点 id 写入 `headerPendPool` 中。

然后我们详细看一下 `queue.ReserveHeaders` 的主要代码：
```go
func (q *queue) ReserveHeaders(p *peerConnection, count int) *fetchRequest {
    ......
    //从 task queue 中选择本次请求的起始高度（跳过已经失败了的）
    send, skip := uint64(0), []uint64{}
    for send == 0 && !q.headerTaskQueue.Empty() {
        from, _ := q.headerTaskQueue.Pop()
        if q.headerPeerMiss[p.id] != nil {
            if _, ok := q.headerPeerMiss[p.id][from.(uint64)]; ok {
                skip = append(skip, from.(uint64))
                continue
            }
        }
        send = from.(uint64)
    }
    // 将跳过的（失败的）任务重新写回 task queue
    for _, from := range skip {
        q.headerTaskQueue.Push(from, -int64(from))
    }
    //构造 request 对象
    if send == 0 {
        return nil
    }
    request := &fetchRequest{
        Peer: p,
        From: send,
        Time: time.Now(),
    }
    ......
    return request
}
```
代码首先从 `queue.headerTaskQueue` 中取出一个值，刚才我们说过，这是一个优先级队列，因此这里最先被 「pop」 出来的是最小的高度值（在我们的例子中，如果这是第一次调用，则 pop 出的是 10）。`queue.headerPeerMiss` 中记录着节点下载数据失败的信息，通过这个字段 `queue.ReserveHeaders`避免了让远程节点重复下载它曾经下载失败的数据：如果某个高度曾经被当前的节点下载过，并且下载失败了，那么就跳过并重新 pop 一个值。注意这段代码虽然是一个 for 循环，但只要取到一个有效的 `send` 值，就会停止循环。

由于 `q.headerTaskQueue.Pop()` 这一条语句会将值从队列中删除，但我们可能没有使用这个值而是记录到了 `skip` 中，因此接下来的一个 for 循环需要将刚才跳过的值重新写回 `queue.headerTaskQueue` 任务队列，以便之后使用其它节点调用 `queue.ReserveHeaders` 时，这些高度可以被其它节点下载。

（从这里也可以看出，一旦某个节点下载失败，就再也不会让这个节点下载相同的数据）

接下来就是利用 `send` 构造 `fetchRequest` 结构了。注意当前我们处在填充 skeleton 的阶段，因此这里的 `fetchRequest` 中的 `From` 字段只标记了 skeleton 中当前这一组的起始高度，而下载数量默认就是 `MaxHeaderFetch` 了。这一点我们也可以从 `Downloader.fillHeaderSkeleton` 中的 `fetch` 回调函数看出来（事实上 `queue.ReserveHeaders` 的返回值正是传给了 `fetch`）：
```go
fetch = func(p *peerConnection, req *fetchRequest) error { 
    return p.FetchHeaders(req.From, MaxHeaderFetch) 
}
```

至此 `queue.ReserveHeaders` 就分析完了。这个方法只有在填充 skeleton 时才会被用到，它的功能就是从任务队列中选一个值最小的、且指定节点没有失败过的起始高度，构造一个 `fetchRequest` 结构并返回。`Downloader.fetchParts` 会将这个结构传给 `fetch` 获取数据。


### deliver header

说完了`reserve`，我们再来看看 `deliver`。在 `Downloader.fillHeaderSkeleton` 中这个变量的实现为：
```go
deliver = func(packet dataPack) (int, error) {
    pack := packet.(*headerPack)
    return d.queue.DeliverHeaders(pack.peerID, pack.headers, d.headerProcCh)
}
```
可见它其实是调用了 `queue.DeliverHeaders` 方法。我们现在就来分析一下这个方法：
```go
func (q *queue) DeliverHeaders(id string, headers []*types.Header, headerProcCh chan []*types.Header) (int, error) {
    request := q.headerPendPool[id]
    if request == nil {
        return 0, errNoFetchesPending
    }
    headerReqTimer.UpdateSince(request.Time)
    delete(q.headerPendPool, id)
}
```
代码一开始是对 `queue.headerPendPool` 的处理。我们刚才介绍 `queue.ReserveHeaders` 时提到过，这个字段用来记录某个节点是否正在下载数据。因此这里如果发现用来下载数据的节点没有在 `queue.headerPendPool` 中，就直接返回错误；否则就继续处理，并将节点记录从 `queue.headerPendPool` 中删除。

我们继续看接下来的代码：
```go
func (q *queue) DeliverHeaders(id string, headers []*types.Header, headerProcCh chan []*types.Header) (int, error) {
    ......

    target := q.headerTaskPool[request.From].Hash()

    accepted := len(headers) == MaxHeaderFetch
    if accepted {
        //检查起始区块的高度和哈希
        if headers[0].Number.Uint64() != request.From {
            accepted = false
        } else if headers[len(headers)-1].Hash() != target {
            accepted = false
        }
    }
    if accepted {
        for i, header := range headers[1:] {
            hash := header.Hash()
            //检查高度的连接性
            if want := request.From + 1 + uint64(i); header.Number.Uint64() != want {
                accepted = false
                break
            }
            //检查哈希的连接性
            if headers[i].Hash() != header.ParentHash {
                accepted = false
                break
            }
        }
    }

    if !accepted {
        miss := q.headerPeerMiss[id]
        if miss == nil {
            q.headerPeerMiss[id] = make(map[uint64]struct{})
            miss = q.headerPeerMiss[id]
        }
        miss[request.From] = struct{}{}

        q.headerTaskQueue.Push(request.From, -int64(request.From))
        return 0, errors.New("delivery not accepted")
    }
    ......
}
```
这代码代码的关键是 `accepted` 变量，也就是决定我们当前的 queue 对象是否接受传送过来的 header 链（即参数 `headers`）。接受的第一个条件是 header 链的长度必须为 `MaxHeaderFetch`，也就是 skeleton 中每一组的区块数量。因为我们当前正处在填充 skeleton 的阶段，且 `queue.DeliverHeaders` 方法也只有在填充 skeleton 时才会被调用，因此此时传递过来的 `headers` 如果有效，数量肯定是 `MaxHeaderFetch`。

长度上满足还不行，代码还会对区块的高度和哈希进行验证，这段验证方法虽然啰嗦，但逻辑很简单，就不细说了。

如果最后发现数据是无效的，也就是「不接受」这些区块，则将这一信息记录到 `queue.headerPeerMiss` 中，并将这组区块起始高度重新放入任务队列 `queue.headerTaskQueue` 中。我们刚才介绍 `queue.ReserveHeaders` 时已经提到过，`queue.headerPeerMiss` 中的信息避免了让远程节点重复下载它曾经下载失败的数据。

我们继续看接受了这些数据的处理代码：
```go
func (q *queue) DeliverHeaders(id string, headers []*types.Header, headerProcCh chan []*types.Header) (int, error) {
    ......

    // Clean up a successful fetch and try to deliver any sub-results
    copy(q.headerResults[request.From-q.headerOffset:], headers)
    delete(q.headerTaskPool, request.From)

    ready := 0
    for q.headerProced+ready < len(q.headerResults) && 
    q.headerResults[q.headerProced+ready] != nil {
        ready += MaxHeaderFetch
    }
    if ready > 0 {
        // Headers are ready for delivery, gather them and push forward (non blocking)
        process := make([]*types.Header, ready)
        copy(process, q.headerResults[q.headerProced:q.headerProced+ready])

        select {
        case headerProcCh <- process:
            q.headerProced += len(process)
        default:
        }
    }
    ......
}
```
这一段代码看上去多，其实主要就是保存数据，以及通知 `headerProcCh` 有新的 header 可以处理了。这里的关键在于理解 `queue.headerResults` 这个字段。这是一个 header 数组（go语言中的切片），它在 `queue.ScheduleSkeleton` 中就初始化好了，其长度为整个 skeleton 的大小（我们的例子中其值为 30）。在下载到数据之前，每一项的值为 `nil`。接受一些数据后，相应的项保存的就是接受的 header。这就是上面代码第一行 `copy` 的功能。而 `headerOffset` 是 `queue.Prepare` 中初始化的将要下载的区块的起始高度，所以 `request.From - q.headerOffset` 自然就是 `headers` 应该在 `queue.headerResults` 的起始位置啦。

在 `queue.DeliverHeaders` 中还有一个功能就是要通知参数 `headerProcCh` 接受了哪些 header。由于所有的 header 都是按顺序存放在 `queue.headerResults` 中，需要有字段记录哪些通知了哪些没通知，所以代码使用了 `queue.headerProced` 这个字段记录已经通知了的 header 的最大索引：在这个索引之前的 header 都已经通知过了，在这之后的都没有通知过。而 `ready` 变量就是找出来的那些还没通知过的 header 的数量。

你可能会想，这里提到的通知是通知给谁呢？这需要看参数 `headerProcCh` 是谁：
```go
func (d *Downloader) fillHeaderSkeleton(from uint64, skeleton []*types.Header) ([]*types.Header, int, error) {
    ......
    deliver = func(packet dataPack) (int, error) {
        pack := packet.(*headerPack)
        return d.queue.DeliverHeaders(pack.peerID, pack.headers, d.headerProcCh)
    }
    ......
}
```
很容易发现它的值就是 `Downlaoder.headerProcCh`，而这个值正是 `Downloader.processHeaders` 等待的 channel。

我们继续看看最后一点代码：
```go
func (q *queue) DeliverHeaders(id string, headers []*types.Header, headerProcCh chan []*types.Header) (int, error) {
    ......
    // Check for termination and return
    if len(q.headerTaskPool) == 0 {
        q.headerContCh <- false
    }
    return len(headers), nil
}
```
这里就很简单了，如果 `queue.headerTaskPool` 为空，说明 skeleton 中所有组都被下载完了，因此发送消息给 `queue.headerContCh`。这个 channel 在 `Downloader.fillHeaderSkeleton` 中是作为 `wakeCh` 传给 `Downloader.fetchParts` 的，用来通知 header 数据已经下载完成了（详细信息参看 [这篇文章](http://yangzhe.me/2019/05/09/ethereum-downloader/)中的「fetchParts」小节）。

总得来看，`queue.DeliverHeaders` 用来处理下载成功的 header 数据，它会对数据进行检验和保存，并发送 channel 消息给 `Downloader.processHeaders` 和 `Downloader.fetchParts` 的 `wakeCh` 参数。


### Schedule

刚才我们说过，在 `queue.DeliverHeaders` 的处理过程中，会发消息给 `Downloader.processHeaders` 对下载成功的 header 进行处理。这个处理的步骤之一就是要根据 header 发起对应的 body 或 receipt 的下载，在这个过程中会调用 `queue.Schedule` 为下载 body 和 receipt 作准备。我们现在来看一下 `queue.Schedule` 的实现：
```go
func (q *queue) Schedule(headers []*types.Header, from uint64) []*types.Header {
    inserts := make([]*types.Header, 0, len(headers))
    for _, header := range headers {
        //一些有效性判断
        ......

        //将信息写入 body 和 receipt 队列
        q.blockTaskPool[hash] = header
        q.blockTaskQueue.Push(header, -int64(header.Number.Uint64()))

        if q.mode == FastSync {
            q.receiptTaskPool[hash] = header
            q.receiptTaskQueue.Push(header, -int64(header.Number.Uint64()))
        }
        inserts = append(inserts, header)
        q.headerHead = hash
        from++
    }
    return inserts
}
```
上面的代码被我删减过，将 for 循环中的四个 if 判断去掉了，这样就显得这个方法简单很多（去掉的四个 if 判断主要是检查 header 是否正确，以及数据是否已经在队列中）。所以我们可以清淅的看到，参数 `headers` 中的区块头的哈希和高度被写到了 body 和 receipt 队列中，等待调度。其中 `queue.blockTaskQueue` 和 `queue.receiptTaskQueue` 都是一个优先级队列，它们与 `queue.headerTaskQueue` 是类似的，都存放着将要下载的数据信息。`queue.BlockTaskPool` 和 `queue.receiptTaskPool` 仅仅是为了记录数据已经被 queue 对象处理了，除此之外并没有什么用处。

注意这里会对下载模式进行判断，因为在 fast 模式下要下载 receipt 数据，而其它模式下 `queue.receiptTaskQueue` 是空的。


### reserve body & receipt

在 `queue` 中准备好了 body 和 receipt 相关的数据队列，`Downloader.processHeaders` 就会通知 `Downloader.fetchBodies` 和 `Downloader.fetchReceipts` 可以对各自的数据进行下载了。这俩方法都调用了 `Downloader.fetchParts`，因此它们的逻辑和填充 skeleton 时的差不多，都是主要使用 `reserve` 和 `deliver` 参数。所不同的是参数的实现方式不一样罢了。查看一下代码就可以知道，对于 body 和 receipt 数据来说，`reserve` 参数最终调用的都是 `queue.reserveHeaders`；而 `deliver` 最终调用的都是 `queue.deliver`。所以我们接下来看一下这两个方法的实现。

`queue.reserveHeaders` 是被 `queue.ReserveBodies` 和 `queue.ReserveReceipts` 共同调用的，只不过使用的参数不同，即用到的「task queue」和「task pool」等数据不同，其它逻辑都是一样的。因此我们通过分析 `queue.reserveHeaders` 来看一下 queue 对象是如何实现 body 和 receipt 的 reserve 功能的：
```go
func (q *queue) reserveHeaders(p *peerConnection, count int, taskPool map[common.Hash]*types.Header, taskQueue *prque.Prque,
	pendPool map[string]*fetchRequest, donePool map[common.Hash]struct{}, isNoop func(*types.Header) bool) (*fetchRequest, bool, error) {
    //如果没有可处理的任务，直接返因
    if taskQueue.Empty() {
        return nil, false, nil
    }
    //如果参数给定的节点正在正在相关数据，直接近回
    if _, ok := pendPool[p.id]; ok {
        return nil, false, nil
    }
    //resultSlots 计算 queue 对象中的缓存空间还可以容纳多少条数据
    space := q.resultSlots(pendPool, donePool)

    // Retrieve a batch of tasks, skipping previously failed ones
    send := make([]*types.Header, 0, count)
    skip := make([]*types.Header, 0)

    ......
}
```
在方法的一开始是一些有效性检查，需要稍作说明的是 `space` 变量，它是 `queue.resultSlots` 的返回值，代表本次 「reserve」 操作最多可以选取多少数据。`queue.resultSlots` 也是 `queue` 对象的 shouldThrottle 方法的内部实现。

我们继续看下面的代码：
```go
func (q *queue) reserveHeaders(p *peerConnection, count int, taskPool map[common.Hash]*types.Header, taskQueue *prque.Prque,
	pendPool map[string]*fetchRequest, donePool map[common.Hash]struct{}, isNoop func(*types.Header) bool) (*fetchRequest, bool, error) {
    ......

    progress := false
    for proc := 0; proc < space && len(send) < count && !taskQueue.Empty(); proc++ {
        header := taskQueue.PopItem().(*types.Header)
        hash := header.Hash()

        //计算当前 header 在 resultCache 中的位置
        index := int(header.Number.Int64() - int64(q.resultOffset))
        if index >= len(q.resultCache) || index < 0 {
            common.Report("index allocation went beyond available resultCache space")
            return nil, false, errInvalidChain
        }
        if q.resultCache[index] == nil {
            components := 1
            if q.mode == FastSync {
                components = 2
            }
            q.resultCache[index] = &fetchResult{
                Pending: components,
                Hash:    hash,
                Header:  header,
            }
        }
        //如果 header 代表的区块是一个空区块（也就是 body 或 receipt 的数据为空），
        //那么直接设置这块数据为下载完成，并标记 progress 为 true
        if isNoop(header) {
            donePool[hash] = struct{}{}
            delete(taskPool, hash)

            space, proc = space-1, proc-1
            q.resultCache[index].Pending--
            progress = true
            continue
        }
        //如果 peerConnection 对象已明确记录了它代表的节点没有这个数据，
        //则将数据放到 skip 中。
        if p.Lacks(hash) {
            skip = append(skip, header)
        } else {
            send = append(send, header)
        }
    }

    ......
}
```
这一段代码是一个 for 循环，它根据 `space` 和 `count` 等变量的限制，从 「task queue」 中依次取出一个任务进行处理。注意对于 body 和 receipt 数据来说，它们的 「task queue」 中的数据依然是 header 对象，因为 body 和 receipt 是依赖于 header 进行下载的。

在 for 循环中主要有三块功能。第一块功能是计算当前 header 在 `queue.resultCache` 中的位置，然后填充 `queue.resultCache` 中相应位置的元素。`queue.resultCache` 字段用来记录所有正在被处理的数据的处理结果，它的元素类型是 `fetchResult` 结构。这里注意它的 `Pending` 字段，它代表当前区块还有几类数据需要下载。这里需要下载的数据最多有两类：body 和 receipt，full 模式下只需要下载 body 数据，而 fast 模式要多下载一个 receipt 数据，因此这里的 `components` 变量初始值为 1，如果是 fast 模式则改成 2。每当有数据下载成功时，`Pending` 字段就会减 1，因此如果它的值为 0，代表所有数据都下载成功了。

for 循环中的第二块功能是处理空区块的情况（`if isNoop(header)` 这个代码分支）。`isNoop` 用来判断将要下载的数据是否为空，如果为空那么是没有必要下载的。这个分支中将 `progress` 设置为 true。这个变量只是用来返回的。在 `Downloader.fetchParts` 中会用到这个变量。

for 循环的第三块功能是处理远程节点缺少这个当前区块数据的情况。在 `peerConnection` 对象中记录着下载失败的区块的数据，因此这里如果发现这个节点曾经下载当前数据失败过，就不再让它下载了。

这个 for 循环结束后，功能基本也就完成了，我们再看看剩下的代码：
```go
func (q *queue) reserveHeaders(p *peerConnection, count int, taskPool map[common.Hash]*types.Header, taskQueue *prque.Prque,
	pendPool map[string]*fetchRequest, donePool map[common.Hash]struct{}, isNoop func(*types.Header) bool) (*fetchRequest, bool, error) {
    ......

    // Merge all the skipped headers back
    for _, header := range skip {
        taskQueue.Push(header, -int64(header.Number.Uint64()))
    }
    if progress {
        // Wake Results, resultCache was modified
        q.active.Signal()
    }
    // Assemble and return the block download request
    if len(send) == 0 {
        return nil, progress, nil
    }
    request := &fetchRequest{
        Peer:    p,
        Headers: send,
        Time:    time.Now(),
    }
    pendPool[p.id] = request

    return request, progress, nil
}
```
最后的代码将跳过的数据（`skip` 变量）再次加入到「task queue」中。如果 `progress` 变量为true，也就是说有区块数据下载成功了（其实是空数据），则设置 `queue.active` 进行通知（`queue.Results` 可能会在等待这个信号）。接下来就是构造 `fetchRequest` 结构并返回了。注意这里的 `request` 变量与 `queue.ReserveHeaders` 中的不同，这里没有用到 `From` 字段（这个字段是下载 header 时才用的）。

可以看到在 `queue` 对象中对于 body 和 receipt 的 reserve 操作，就是从各自的 「task queue」 中选取一定数量的任务数据，写入 `queue.resultCache`中并构造 `fetchRequest` 结构并返回。这个过程中会对空区块和曾经下载失败的区块进行特殊处理。


### deliver body & receipt

现在在 `Downloader.fetchParts` 内部，body 或 receipt 数据都已经通过 reserve 操作构造了 `fetchRequest` 结构并传给 `fetch`，接下来就是等待数据的到达啦。数据下载成功后，会调用 `queue` 对象的 deliver 方法进行传递，这包括 `queue.DeliverBodies` 和 `queue.DeliverReceipts`。这两个方法都以不同的参数调用了 `queue.deliver` 方法，因此接下来我们就看看这个方法：
```go
func (q *queue) deliver(id string, taskPool map[common.Hash]*types.Header, taskQueue *prque.Prque,
	pendPool map[string]*fetchRequest, donePool map[common.Hash]struct{}, reqTimer metrics.Timer,
	results int, reconstruct func(header *types.Header, index int, result *fetchResult) error) (int, error) {

    // Short circuit if the data was never requested
    request := pendPool[id]
    if request == nil {
        return 0, errNoFetchesPending
    }
    reqTimer.UpdateSince(request.Time)
    delete(pendPool, id)

    // If no data items were retrieved, mark them as unavailable for the origin peer
    if results == 0 {
        for _, header := range request.Headers {
            request.Peer.MarkLacking(header.Hash())
        }
    }

    ......
}
```
此方法前面的代码也是一些有效性判断。首先是针对 「pending pool」的操作；其次是如果下载的数据数量为0，则把所有此节点此次下载的数据标记为「缺失」（还记得刚才 `queue.reserveHeaders` 中的 「isLacks」调用吗？它与 「MarkLacking」 是配合使用的）。

我们继续看接下来的代码：
```go
func (q *queue) deliver(id string, taskPool map[common.Hash]*types.Header, taskQueue *prque.Prque,
	pendPool map[string]*fetchRequest, donePool map[common.Hash]struct{}, reqTimer metrics.Timer,
	results int, reconstruct func(header *types.Header, index int, result *fetchResult) error) (int, error) {
    ......

    var (
        accepted int
        failure  error
        useful   bool
    )
    for i, header := range request.Headers {
        // Short circuit assembly if no more fetch results are found
        if i >= results {
            break
        }

        index := int(header.Number.Int64() - int64(q.resultOffset))
        if index >= len(q.resultCache) || index < 0 || q.resultCache[index] == nil {
            failure = errInvalidChain
            break
        }

        //reconstruct 填充 resultCache[index] 中的相应的字段
        if err := reconstruct(header, i, q.resultCache[index]); err != nil {
            failure = err
            break
        }
        hash := header.Hash()

        donePool[hash] = struct{}{}
        q.resultCache[index].Pending--
        useful = true
        accepted++

        // Clean up a successful fetch
        request.Headers[i] = nil
        delete(taskPool, hash)
    }

    ......
}
```
这个 for 循环处理下载成功的数据。注意 `request.Headers` 仍然是 reserve 操作时生成的数据，那么这里下载到的数据在哪呢？由于代码中把 body 和 receipt 的 deliver 操作都抽象到了 `queue.deliver` 这个方法中，所以这些不同类型的数据无法直接传递到这个方法中进行处理，而是使用了 `reconstruct` 这样一个函数。在 `queue.DeliverBodies` 中这个函数是这样实现的：
```go
    reconstruct := func(header *types.Header, index int, result *fetchResult) error {
        if types.DeriveSha(types.Transactions(txLists[index])) != header.TxHash || 
            types.CalcUncleHash(uncleLists[index]) != header.UncleHash {
            return errInvalidBody
        }
        result.Transactions = txLists[index]
        result.Uncles = uncleLists[index]
        return nil
    }
```
而在 `queue.DeliverReceipts` 中它是这样实现的：
```go
    reconstruct := func(header *types.Header, index int, result *fetchResult) error {
        if types.DeriveSha(types.Receipts(receiptList[index])) != header.ReceiptHash {
            return errInvalidReceipt
        }
        result.Receipts = receiptList[index]
        return nil
    }
```
不管对于 body 还是 receipt，它都是先检查其哈希值是否正确，然后将其写入 `fetchResult` 结构中相应的字段中。而 `result` 参数的值正是 `queue.resultCache[index]`。

将数据写入 `queue.resultCache[index]` 中对应的字段后，代码还会处理 「done pool」 和 「task pool」，以及将 `queue.resultCache[index].Pending` 字段的值减 1。

我们继续看下面的代码：
```go
func (q *queue) deliver(id string, taskPool map[common.Hash]*types.Header, taskQueue *prque.Prque,
	pendPool map[string]*fetchRequest, donePool map[common.Hash]struct{}, reqTimer metrics.Timer,
	results int, reconstruct func(header *types.Header, index int, result *fetchResult) error) (int, error) {
    ......

    for _, header := range request.Headers {
        if header != nil {
            taskQueue.Push(header, -int64(header.Number.Uint64()))
        }
    }
    // Wake up Results
    if accepted > 0 {
        q.active.Signal()
    }
    // If none of the data was good, it's a stale delivery
    switch {
    case failure == nil || failure == errInvalidChain:
        return accepted, failure
    case useful:
        return accepted, fmt.Errorf("partial failure: %v", failure)
    default:
        return accepted, errStaleDelivery
    }
}
```
在第一个 for 循环之后，所有被验证通过且写入 `queue.resultCache` 中的数据，其对应的 `request.Headers` 中的 header 都应为 nil。如果不是，说明这个数据没有被验证通过，需要将其加入 「task queue」 重新下载。

如果有数据被验证通过且写入 `queue.resultCache` 中了（`accepted` > 0），那么就要发送 `queue.active` 消息了。我们前面提到过，`queue.Results` 可能会等待这这个信号。


### RetrieveHeaders

对于整个区块同步的流程来说，我们现在处在不断填充 skeleton、同时不断下载 body 和 receipt 数据的过程。如果 skeleton 填充完成，就会调用 `queue.RetrieveHeaders` 获取下载的 header 信息（在 `Downloader.fillHeaderSkeleton` 中）。我们看看这个方法是如何实现的 ：
```go
func (q *queue) RetrieveHeaders() ([]*types.Header, int) {
    q.lock.Lock()
    defer q.lock.Unlock()

    headers, proced := q.headerResults, q.headerProced
    q.headerResults, q.headerProced = nil, 0

    return headers, proced
}
```
这个方法的实现很简单，就是直接返回 `queue.headerResults` 和 `queue.headerProced` 两个字段的数据。那么这两个字段存储了什么数据呢？这就需要再看一下 `queue.DeliverHeaders` 的部分实现了：
```go
func (q *queue) DeliverHeaders(id string, headers []*types.Header, headerProcCh chan []*types.Header) (int, error) {
    ......
    copy(q.headerResults[request.From-q.headerOffset:], headers)
    ......
    if ready > 0 {
        ......
        select {
        case headerProcCh <- process:
            q.headerProced += len(process)
        default:
        }
    }
    ......
}
```
可见俩字段是在 `queue.DeliverHeaders` 中填充的。当有 header 下载成功后，就会将这些 header 存储在 `queue.headerResults` 中；并且将发送给参数 `headerProcCh` 的 header 的数量存储在 `queue.headerProced` 中。（前面我们在介绍 `queue.DeliverHeaders` 时提到过，`headerProcCh` 参数实际上就是 `Downloader.headerProcCh`， `downloader.processHeaders` 会等待这个 channel）


### Results

目前为止，整个流程处在 header、body 等数据不断在同时下载的过程中。那么当一个区块的数据（header、body 和 receipt）都下载完成时，`Downloader` 对象就要获取这些区块并将其写入数据库了。`queue.Results` 就是用来返回所有目前已经下载完成的数据，它在 `Downloader.processFullSyncContent` 和 `Downloader.processFastSyncContent` 中被调用。我们下面就看一下它的实现：
```go
func (q *queue) Results(block bool) []*fetchResult {
    q.lock.Lock()
    defer q.lock.Unlock()

    // countProcessableItems 返回 queue.resultCache 中已经下载完成的数据的数量
    nproc := q.countProcessableItems()
    for nproc == 0 && !q.closed {
        if !block {
            return nil
        }
        q.active.Wait()
        nproc = q.countProcessableItems()
    }
    // Since we have a batch limit, don't pull more into "dangling" memory
    if nproc > maxResultsProcess {
        nproc = maxResultsProcess
    }
    results := make([]*fetchResult, nproc)
    copy(results, q.resultCache[:nproc])
    ......
}
```
这个方法的首先获取已经完全下载完成的数据的数量，如果为 0，则根据参数的指示来决定是否进行等待。如果参数为 true 就会等待 `queue.active` 这个信号（还记得我们前面我们几次提过 `queue.active` 消息，说它可能在 `queue.Results` 中被等待吗）。

在有数据的情况下，会将这些数据拷贝到待返回的 `results` 中。注意这里是从 `queue.resultCache` 中的第 0 个元素开始拷贝的，这可能会让你产生疑惑，为什么完全下载完成的数据一定是从第 0 个到第 `nproc-1` 个呢？这是由代码的实现方式决定的，一会下面的代码就会看到，将数据拷贝到 `results` 中以后，就会删除这些数据。这样做可以清除那些被取走的数据，从而为新的数据腾出内存。这也是 `Downloader.fetchParts` 中调用 throttle 参数有意义的关键：如果不清除已经被取走的数据，那么内存占用就会一直增加，当达到 shouldThrottle 的阈值以后即使进行等待，也等不到内存占用减小了。

下面我们看看 `queue.Results` 后面的代码：
```go
func (q *queue) Results(block bool) []*fetchResult {
    ......
    if len(results) > 0 {
        // Mark results as done before dropping them from the cache.
        for _, result := range results {
            hash := result.Header.Hash()
            delete(q.blockDonePool, hash)
            delete(q.receiptDonePool, hash)
        }
        //将已经拷贝到 results 中的数据从 resultCache 中清除，同时修正 resultOffset 的值
        copy(q.resultCache, q.resultCache[nproc:])
        for i := len(q.resultCache) - nproc; i < len(q.resultCache); i++ {
            q.resultCache[i] = nil
        }
        // Advance the expected block number of the first cache entry.
        q.resultOffset += uint64(nproc)

        // Recalculate the result item weights to prevent memory exhaustion
        for _, result := range results {
            size := result.Header.Size()
            for _, uncle := range result.Uncles {
                size += uncle.Size()
            }
            for _, receipt := range result.Receipts {
                size += receipt.Size()
            }
            for _, tx := range result.Transactions {
                size += tx.Size()
            }
            q.resultSize = common.StorageSize(blockCacheSizeWeight)*size + (1-common.StorageSize(blockCacheSizeWeight))*q.resultSize
        }
    }
    return results
}
```
这些代码是对一些记录的修正，比如将已经拷贝到 `results` 中的数据从 `queue.resultCache` 中清除；修正 `queue.resultSize` 的值等，都比较简单，就不再多说了。


### 其它
上以关于 `queue` 对象的一些功能的介绍，基本上走完了一个区块下载的主要流程。其实在区块同步的过程中，`Downloader` 对象还依赖于 `queue` 对象提供的一些其它的功能，比如 inflight、shouldThrottle、cancel、expire 等。这些功能的实现比较简单，基本都是对 `queue` 对象内部记录的数据的一些计算和判断，这里就不再细说了。


# 总结

本篇文章我们以区块同步的流程为线索，介绍了 `queue` 对象的一些重要的且又较难理解的内容。`queue` 是 downloader 模块的一个辅助对象，用来帮助 `Downloader` 对象记录区块的下载状态。文章中并没有分析到 `queue` 对象的每一个实现细节，如果有文章中未提及到的内容的疑问，欢迎讨论。另外限于个人能力，文章中难免有失误的地方，如果有问题，也感谢留言或邮件讨论。