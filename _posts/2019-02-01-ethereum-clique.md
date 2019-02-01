---
layout: post
title:  "以太坊源码解析：共识算法之clique"
categories: ethereum
tags: 原创 ethereum consensus clique 源码解析
excerpt: clique模块是以太坊（ethereum）中PoA(权威证明，Proof of Authority)共识算法的实现。那么什么是PoA呢？clique又是如何实现的呢？本篇文章将会给你答案。相信仔细看过这篇文章以后，你就可以实现自己的PoA算法了。
author: fatcat22
---

* content
{:toc}





# 引言
共识算法是区块链项目的核心之一，每一个运行着的区块链都需要一个共识算法来保证出块的有效性和有序性。在以太坊的官方源码中，有两个共识算法：clique和ethash，它们都位于以太坊项目的consensus目录下。clique目录下的代码实现的是PoA(权威证明，Proof of Authority)共识，这是这篇文章要分析的代码；在ethash目录下实现的是PoW(工作量证明，Proof of Work)共识，下一篇文章将会对其进行介绍。

多数区块链项目都会根据项目需求选定一种共识算法，为什么以太坊会同时存在两种呢？这背后肯定是有故事的。

# clique诞生背景及其应用
clique代码的原作者式在[这里](https://github.com/ethereum/EIPs/issues/225)对clique进行了介绍。在"Background"这一节里，作者专门对诞生背景作了详细的说明，大体意思是说最开始官方的testnet是“Morden”，但随着时间的推移它的各种遗留问题和兼容问题越来越多，所以干脆推倒重来，创建了新的测试网络“Ropsten”。“Ropsten”和主网一样，使用的是PoW共识算法。但后来“Ropsten”受到了恶意攻击，主要原因是因为PoW共识算法的安全性受限于计算机的算力，而“Ropsten”作为测试网络对算力的要求比较低，导致攻击者对算力的滥用。虽然这仍然可以通过重启测试网络来修复攻击造成的不良影响，但以太坊团队选了一个一劳永逸的方式，那就是将共识算法换成PoA类型。这才有了clique模块。

从这里也可以看出，在以太坊中clique仅在测试网络里使用，真实的以太坊主网还是使用PoW算法（ethash模块实现）。但在自己组成私有网络时，你可以自由选择使用clique还是ethash。


# 什么是共识
本篇文章的主要目的是介绍以太坊的共识算法之一clique模块，无意深入介绍共识的所有知识，感兴趣的读者可以自己搜索相关知识，或以[这篇文章](https://yeasy.gitbooks.io/blockchain_guide/content/distribute_system/consensus.html)作为参考，对其中介绍到的知识点进行拓展学习。

区块链中的共识与我们日常所说的“共识”其实是一个概念。在现实世界里，每个人作为独立的个体，对事物的判断是不一致的，各有各的主观想法。但社会要正常运行，就需要所有人对某些事物有一致的认识（所谓达成共识），否则就会乱套。比如红绿灯，所有人都要达成遵守红绿灯规则的共识，否则就会出问题；比如各国人民对自己国家的货币其实都达成了共识：自己国家的货币是有购买力的（全世界的人都可以用美元交易，是因为全世界的人都对“美元具有购买力”达成了共识）；再比如消费，买东西要付钱，这是所有人必须达成一致的，如果有一帮人认为买东西要付钱、另一帮人认为买东西给块石头就行了，那就会出现混乱（一个人做出的决定，另一个人不认可）。

可以看到，共识其实是一个很基本的概念，广泛存在于各个方面，这一点从[维基百科](https://en.wikipedia.org/wiki/Consensus_(disambiguation))上对“共识”的解释也可以看出来。我们常说的“做人要讲道理”中的道理即是共识（所谓的道理是每个人都认可的基本观点，比如要孝顺、要有礼貌，如果每个人心中都有自己不同的一套道理，那跟没道理是一样的）。在计算机中，共识最早是分布式计算的一个概念，而区块链的世界也是一个分布式的世界。

在区块链中，每个节点和每个人类似，都是独立的个体。这些独立的节点想要顺利地共同完成一件事情（比如出块、记账，对链进行维护），就需要遵守相同的基本规则，否则一个节点产生的数据，别的节点也不会认可。这些基本规则，比如一个正常的块有哪些限制、链是如何组织的、如何判断一个块的正确性等等这些，共同组成了共识。

上面这些是我自己的理解，为了准确性，我这里摘录了[维基百科](https://zh.wikipedia.org/wiki/%E5%85%B1%E8%AD%98%E6%A9%9F%E5%88%B6)中对共识的解释，来结束我们对共识的讨论：
>由于加密货币多数采用去中心化的区块链设计，节点是各处分散且平行的，所以必须设计一套制度，来维护系统的运作顺序与公平性，统一区块链的版本，并奖励提供资源维护区块链的使用者，以及惩罚恶意的危害者。这样的制度，必须依赖某种方式来证明，是由谁取得了一个区块链的打包权（或称记账权），并且可以获取打包这一个区块的奖励；又或者是谁意图进行危害，就会获得一定的惩罚，这就是共识机制。


# 什么是PoA
PoA是共识算法的一种，它的全称为Proof of Authority，中文译为威权证明。PoA的基本思想应该也是来源于现实世界：在现实世界里，对很多事情我们往往“诉诸权威”，即我们相信专家。PoA的思想与此类似：授权一定数量的“专家”，由这些人相互合作打包区块并对区块链进行维护，其它人则无权打包区块，并且普通人相信成为专家的人会努力维护区块链的正常秩序。

当然事情没这么简单，不是说我们想当然的认为谁是专家他就是了。专家需要公开自己的身份。这也是PoA设计初衷的一部分：设计者认为每个人都是爱惜自己的声誉的，通过公开自己身份，专家会为了自己的声誉努力做正确的事，而不是作恶。

总得来说PoA共识中出块权掌握在部分专家手里，而普通人是无法参与的（无论你有多少算力、多少权益）。可见PoA共识牺牲了一部去中心化的特性，换来了一种可控性。但它同时有两个问题是必须解决的，我们逐个介绍。

注意，在后面的文章里，为了描述的一致性，我们将专家称为“签名者”，即有权生成新区块并签名的账号地址。

- **问题1：如何实现签名者的引进和踢出**  
PoA的第一个问题是需要解决签名者更换的问题。在PoA中，签名者必须保证多数情况下在线出块。然而随着项目的不断运行，不可能所有签名者都一直有资源、有意愿继续工作；另外偶尔签名者也会作恶，必须及时将作恶的人踢出。（作为对比，在PoW(工作量证明)中，任何一个人都可以随时接入区块链网络并尝试出块，也可以随时退出网络）
- **问题2：如何控制出块时机**  
首先要明确的是，出块时机由两方面决定：一是出块时间；二是由谁出块。在PoA中，签名者之间是合作关系，大家“和和气气”，什么时间出块、由谁出都要按规则来，不能争不能抢。因此需要有良好的规则控制出块时机。（作为对比，在PoW中，出块时间根据历史出块记录动态调整；由谁出块是由算力决定的：算力越强，越能获得出块权。可见在PoW中签名者之间是竞争的关系，出块时机由能力确定）

我认为一个好的PoA实现，必须要解决好这两个问题。下面我们看看clique是如何解决这些问题的。


# clique的设计概要
clique模块的原作者在[这篇文章](https://github.com/ethereum/EIPs/issues/225)里详细说明了clique的设计和背景。clique作为PoA的一个实现，自然要解决前面提到的两个问题。因此这里我们从这两个问题的解决方案的角度，汇总一下作者对clique的设计。

为了表达清晰，我们需要提先说明几个原文中的数据和名词的定义：
- **checkpoint**: 一个特殊的block，它的高度是EPOCH_LENGTH的整数倍，block中不包含投票信息但包含当时所有的签名者列表
- **SIGNER_COUNT**: 某一时刻签名者的数量
- **SIGNER_LIMIT**: 连续的块的数量，在这些连续的块中，某一签名者最多只能签一个块；同时也是投票生效的票数的最小值
- **BLOCK_PERIOD**: 两个相邻的块的Time字段的最小差值，也是出块周期
- **EPOCH_LENGTH**: 两个checkpoint之间的block的数量。达到这个数量后会生成checkpoint以及清除当前所有未生效的投票信息
- **DIFF_INTURN**: 出块状态（difficulty）之一，此状态代表“按道理已经轮到我出块”
- **DIFF_NOTURN**: 出块状态（difficulty）之一，此状态代表“按道理还没轮到我出块”

接下来我们看一下clique是如何解决上一节提到的两个问题的。

- **问题1：如何实现签名者的引进和踢出**  
clique中签名者的引进和踢出是通过已有签名者进行投票实现的，并且加入了更加详细的控制。下面我们看一下clique中的投票规则：
  - 投票信息保存在block中。一个block只有一个投票信息，且只能在自己生成的block上保存。
  - 针对某个被投人的票数超过SIGNER_LIMIT时，投票结果立即生效。
  - 投票生效后，立即清除所有被投人是当前生效人的投票信息。如果投的是踢出票，则被投人之前投出的、但还未生效的投票全部失效。
  - 踢出一个签名者以后，可能会导致原来不通过的投票理论上可以通过。clique不特意处理这种情况，等待下次统计时再判断。
  - 发起一个投票后，客户端不会被立即清除投票信息，而是在之后每次出块时都会选一个继续投票。因为区块链中的有效区块有重新调整的可能性，所以不能认为投票生效了之后就会一直生效。
  - 无效的投票：被投票人不在当前签名者列表中但投的是踢出票，或被投票人在当前签名列表中但投的是引进票
  - 为了使编码简单，无效的投票不会受到惩罚（其实我认为有些功能实现也依赖于无效的投票）
  - 在每个EPOCH_LENGTH内，一个签名者给同一个账号地址重复投票时，会先将上次的投票信息清除，然后再统计本次的投票信息（如果本次为无效的投票不会恢复已经清除的上次投票信息）
  - 每个checkpoint不进行投票，而只是包含当前签名者列表信息。对于其它区块，可以用来携带投票信息。

关于上面重复投票的处理需要多说一下，这种处理方式会产生两个结果（假设投票人是A，被投票人是B）：
1. 在当前EPOCH_LENGTH内，A给B只能投一票
2. 在当前EPOCH_LENGTH内，如果给B的投票未生效（总票数未超过SIGNER_LIMIT）时A想把投给B的票撤消，那么A可以投一次跟之前相反的票。因为新的投票会导致旧的投票信息清除，而如果旧的投票是有效的则新的投票必定是无效的，因而也不会进入投票统计。

- **问题2：如何控制出块时机**  
前面我们说过，出块时机由两方面决定：一是出块时间；二是由谁出块。下面我们看看clique是如何解决这些问题的。
  - 出块时间  
  在clique中，出块时间是固定的，由BLOCK_PERIOD决定。
  - 由谁出块  
  clique中出块权的确定稍微复杂，具体规则为：  
    - 签名者在签名者列表中且在SIGNER_LIMIT内没出过块
    - 如果签名者是DIFF_INTURN状态，则拥有较高出块权（等待出块时间到来，签名区块并立即广播出去）
    - 如果签名者是DIFF_NOTURN状态，则拥有较低出块权（等待出块时间到来，再延迟一下（延迟时间为rand(SIGNER_COUNT * 500ms)）  

可见出块权由两方面确定：一是最近是否出过块，如果出过则没有出块权；二是DIFF_INTURN / DIFF_NOTURN状态，DIFF_INTURN拥有较高出块权。

以上是我们对clique中实现PoA关键点的规则说明，如果目前看不太懂也没关系，后面还会有针对性的分析说明。相信理解这些规则以后，我们就可以自己实现一个PoA共识算法了。


# 使用clique搭建私有网络进行调试
使用以太坊搭建私有网络本来与这篇文章的内容没有关系，但我自己在学习源代码的过程中感觉只读代码有些难以理解，因此搭建了一个简单的有三个节点的私有网络，并使用clique作为共识模块，通过程序的实际运行加深和验证我对代码的理解。我觉得作为读者的你可能也会需要，因此在这里将搭建的过程简单说明一下。

注意我写这篇文章时使用的是[go-ethereum](https://github.com/ethereum/go-ethereum)项目上的master分支，commit id 为257bfff316e4efb8952fbeb67c91f86af579cb0a。

> 1. 执行`make all`进行编译。编译结果在build/bin目录下。这时我们只用到到geth和puppeth两个程序。
> 2. 进入build/bin目录。（后面会在这个目录下生成一些文件和文件夹。这里我为了运行geth和puppeh程序方便直接在build/bin目录下操作，你也可以换成其它目录）
> 3. 在当前目录创建node0、node1、node2三个文件夹。
> 4. 调用`geth --datadir node0 account new`在node0目录下创建新的账号。使用类似的命令为node1、node2创建一个账号，为了方便三个账号的密码最好都设置成一样的，并保存在./password文件中。记住这三个账号，下一步会用到。
> 5. 使用puppeth生成配置文件。puppeth是交互式的，你可以跟据提示进行操作，过程中会用到上一步生成的账号作为"allowed to seal"的账号。在选择"consensus engine"时记得使用clique。（假设最后生成的文件为当前目录的genesis.json）
> 6. puppeth生成的配置文件是json格式，但无法直接用于geth，需要去掉其它字段，而把"genesis"字段的内容作为整个json文件的内容。
> 7. 使用`geth --datadir node0 init genesis.json`初始化节点数据，使用类似的命令初始化node1和node2目录。
> 8. 使用`geth --datadir node0 --port 30000 --nodiscover --unlock '0' --password ./password console`启动节点。新打开两个终端，并使用类似的命令启动其它两个节点（注意修改datadir和port参数，每一个节点都不相同）。注意使用console参数后，可以通过以太坊的控制台对节点进行操作。
> 9. 在每个节点启动后的log中，有一条“Started P2P networking”的log，其self参数是当前节点的p2p地址。在console中，使用命令`admin.addPeer("nodeAddr")`添加节点，其中nodeAddr参数为节点p2p地址。为node0添加node1和node2的地址，为node1添加node2的地址。
> 10. 至此，这三个节点已经相互发现，组成了一个私有网络。在每个节点的console中调用`miner.start()`使当前节点开始出块。


# 代码分析
在这一小节里，我们将对clique模块的重要代码进行详细的分析和说明。我们首先从细节着手，对一些代码实现中的概念进行说明；然后从一个全局的视角，看一看clique是如何配合挖矿模块进行挖矿的。


### Header字段意义变化
在以太坊中存在两种共识，但显然不能用两套区块链上的数据结构。因此作为后来实现的clique只能复用原来Header的字段。下面我们对这些复用的字段一一说明一下。

- Header.Coinbase  
Coinbase字段在ethash中用来存放出块者的地址。在clique中用来保存投票时被投票人的地址。而出块者的地址通过签名数据计算得出（ecrecover）。
- Header.Nonce  
Nonce字段在ethash中用来作为一个变量调整Header的哈希。在clique中没有这个需求，因此直接用来保存投票目的。如果值为nonceAuthVote(0xffffffffffffffff)则代表这是一次授权投票；如果值为nonceDropVote(0x0000000000000000)则代表这是一次踢出投票。
- Header.Extra  
在clique中Extra除了依然保存vanity数据和Seal数据，还在checkpoint中增加了所有签名者地址数据。其结构为：  
vanityData(固定32字节)+signer1Address+signer2Address+...+SealData(固定65字节)


### 概念
读懂一段代码的关键，是要理解作者在代码中设立的概念。因此我们下面重点解释了一些clique中我认为比较重要和难以理解的概念。

##### epoch and checkpoint
在clique中，有一个值叫做"epoch"。当一个block的高度恰好是"epoch"值的整数倍时，这个block便不会包含任何投票信息，而是包含了当前所有的签名者列表。这个block被叫做checkpoint。可以看出，checkpoint类似于一个“里程碑”，可以用来表示“到目前为止，有效的签名者都记录在我这里了”；而epoch就是设立里程碑的距离。

为什么要设立epoch这个概念呢？在clique的[设计文档](https://github.com/ethereum/EIPs/issues/225)中，我找到了答案，摘录如下：
>To avoid having an infinite window to tally up votes in, and also to allow periodically flushing stale proposals, we can reuse the concept of an epoch from ethash, where every epoch transition flushes all pending votes. Furthermore, these epoch transitions can also act as stateless checkpoints containing the list of current authorized signers within the header extra-data. This permits clients to sync up based only on a checkpoint hash without having to replay all the voting that was done on the chain up to that point. It also allows the genesis header to fully define the chain, containing the list of initial signers.

简单来说，"epoch"的存在，是为了避免没有尽头的投票窗口，也是为了周期性的清除除旧的投票提案。更进一步地，在checkpoint中存在的签名者列表，可以让节点间基于中间某个checkpoint就可以同步到签名者列表，而不需要整个链上的数据。

下面我摘录了部分Clique.Prepare代码，可以看到如果将要生成的block是一个checkpoint时的特殊处理：
```go
func (c *Clique) Prepare(chain consensus.ChainReader, header *types.Header) error {
  //others code
  ......

  // number为将要生成的块的高度

  //如果number不是epoch的整数倍（不是checkpoint），则进行投票信息的填充
  if number%c.config.Epoch != 0 {
    ......

    //填写投票信息（投票信息存储在Coinbase和Nonce字段中）
    if len(addresses) > 0 {
      header.Coinbase = addresses[rand.Intn(len(addresses))]
      if c.proposals[header.Coinbase] {
        copy(header.Nonce[:], nonceAuthVote)
      } else {
        copy(header.Nonce[:], nonceDropVote)
      }
    }
  }

  ......

  //如果number是epoch的整数倍（将要生成一个checkpoint），则填充签名者列表
  if number%c.config.Epoch == 0 {
    for _, signer := range snap.signers() {
      header.Extra = append(header.Extra, signer[:]...)
    }
  }
}
```

在`Snapshot.apply`方法中的代码，则体现了“避免没有尽头的投票窗口，周期性的清除除旧的投票提案”的功能：
```go
func (s *Snapshot) apply(headers []*types.Header) (*Snapshot, error) {
  ......

  for _, header := range headers {
    // Remove any votes on checkpoint blocks
    number := header.Number.Uint64()
    //如果当前header的高度是epoch的整数倍（当前block是一个checkpoint），则清除所有投票信息和统计
    //Votes和Tally字段的详细信息参看对Snapshot的介绍。
    if number%s.config.Epoch == 0 {
      snap.Votes = nil
      snap.Tally = make(map[common.Address]Tally)
    }

    ......
  }
}
```

这里稍微说明一下为什么这一点代码实现了epoch的设计目的。一个Snapshot对象中保存了在某个高度上的投票信息，而创建一个Snapshot对象其实是分两步的，apply方法属于第二步。假如给apply传入10个header，其中第8个的高度恰好是epoch的整数倍，那么前面那7个的投票信息就被抹去了。


##### checkpointInterval
除了epoch，代码中还有一个值为checkpointInterval。当在某个block上创建一个Snapshot对象，且这个block的高度是checkpointInterval的整数倍时，则将这个Snapshot对象写入到数据库中永久存储；以后想到再次在这个block上生成Snapshot对象时，直接从数据库中读取就可以了。摘录代码如下：
```go
func (c *Clique) snapshot(chain consensus.ChainReader, number uint64, hash common.Hash, parents []*types.Header) (*Snapshot, error) {
  ......

  for snap == nil {
    ......

    //如果高度为checkpointInterval的整数倍，则直接尝试从数据库中读取Snapshot对象
    if number%checkpointInterval == 0 {
      if s, err := loadSnapshot(c.config, c.signatures, c.db, hash); err == nil {
        snap = s
        break
      }
    }

    ......
  }

  ......

  //如果高度为checkpointInterval的整数倍，则将Snapshot对象存储到数据库中
  if snap.Number%checkpointInterval == 0 && len(headers) > 0 {
    if err = snap.store(c.db); err != nil {
      return nil, err
    }
  }
  return snap, err
}
```

##### Snapshot
Snapshot对象是clique中比较重要的一个对象，它的作用是统计并保存链的某段高度区间的投票信息和签名者列表。这个统计区间是从某个checkpoint开始（包括genesis block），到某个更高高度的block。在Snapshot对象中用到了两个重要的结构体：Vote和Tally，我们先对它们进行一下说明，再来详细说一下Snapshot结构体。

- Vote struct

Vote代表的是一次投票的详细信息，包括谁给谁投的票、投的加入票还是踢出票等等。它的结构体定义如下：
```go
type Vote struct {
  Signer    common.Address // 此次投票是由谁投的
  Block     uint64         // 此次投票是在哪个高度的block上投的
  Address   common.Address // 此次投票是投给谁的
  Authorize bool           // 这是一个加入票（申请被投人成为签名者）还是踢出票（申请将被投人踢出签名者列表）
}
```

- Tally struct

Tally结构体是对所有被投人的投票结果统计。注意它与Vote结构体的区别：Vote是投票过程的记录（如A给B投了一个授权票），而Tally是对结果的统计（类似于选班长唱票时计票员在黑板上画的“正”字）。Tally的定义如下：
```go
type Tally struct {
  Authorize bool // 这是加入票的统计还是踢出票的统计
  Votes     int  // 目前为止累计的票数
}
```

如果只看这里你可能会意外这里并没有“针对谁进行的统计”的信息，这是因为Tally在Snapshot结构体是是作为map的一部分的，参看下面对Snapshot结构体字段的说明。

- Snapshot struct

前面我们已经说明了Snapshot结构体的意义，因此我们直接看一下结构体的定义：
```go
type Snapshot struct {
  config   *params.CliqueConfig
  sigcache *lru.ARCCache        

  Number  uint64                      // Block number where the snapshot was created
  Hash    common.Hash                 // Block hash where the snapshot was created
  Signers map[common.Address]struct{} // 当前的所有有效的签名者
  Recents map[uint64]common.Address   // 最近生成过block的签名者。其中map的key是生成的block的高度
  Votes   []*Vote                     // 按时间先后顺序保存的投票信息
  Tally   map[common.Address]Tally    // 目前为止累计各被投人的票数
}
```

除了Votes和Tally，比较重要的字段就是Signers和Recents了。Signers字段比较好理解，就是当前可以出块的所有的签名者。各个节点根据这个字段的信息来判断某个块的签名者是否真的有权出块。注意这个字段是可以不断动态变化的，当某个账号地址的累计投票数超过一半时，就会被加入到Signers中（或从Signers中移除），实现代码摘录如下：
```go
func (s *Snapshot) apply(headers []*types.Header) (*Snapshot, error) {
  for _, header := range headers {
    ......

    //如果累计投票数超过一半
    if tally := snap.Tally[header.Coinbase]; tally.Votes > len(snap.Signers)/2 {
      if tally.Authorize {
        //如果是加入票，则将被投人加入到Signers列表中
        snap.Signers[header.Coinbase] = struct{}{}
      } else {
        //如果是踢出票，则把被投人从Signers中删除
        delete(snap.Signers, header.Coinbase)

        //将被投人从Recents信息中删除
        if limit := uint64(len(snap.Signers)/2 + 1); number >= limit {
          delete(snap.Recents, number-limit)
        }

        //丢弃被投人之前投出的所有投票
        for i := 0; i < len(snap.Votes); i++ {
          if snap.Votes[i].Signer == header.Coinbase {
            snap.uncast(snap.Votes[i].Address, snap.Votes[i].Authorize)
            snap.Votes = append(snap.Votes[:i], snap.Votes[i+1:]...)
            i--
          }
        }
      }

      //清除所有被投人是当前生效人的投票信息
      for i := 0; i < len(snap.Votes); i++ {
        if snap.Votes[i].Address == header.Coinbase {
          snap.Votes = append(snap.Votes[:i], snap.Votes[i+1:]...)
          i--
        }
      }
      delete(snap.Tally, header.Coinbase)  //清空当前生效人的票数统计信息
    }
  }

  ......
}
```

`Snapshot.Recents`字段保存了最近出块的签名者和所出的块的高度。这个“最近”的定义是最新的`len(Snapshot.Signers)/2 + 1`个块。我们先看一下代码是如何操作这个字段的：
```go
func (s *Snapshot) apply(headers []*types.Header) (*Snapshot, error) {
  for _, header := range headers {
    ......

    //将高度与当前块相差limit的块从Recents中删除掉
    if limit := uint64(len(snap.Signers)/2 + 1); number >= limit {
      delete(snap.Recents, number-limit)
    }
    ......

    //将当前块的高度和签名者加入Recents中
    snap.Recents[number] = signer
  }
}
```

比如目前有6个签名者，当前块的高度是10，那么高度为"10 - (6/2 + 1) = 6"的块将从Recents中删除，然后将高度为10的块和其签名者加入Recents中；处理一下个块即高度为11时，高度为7的块又会从Recents中删除，然后高度为11的块会被加入。总之，这个"最近"的基点是当前块的高度，囊括的范围为`len(Snapshot.Signers)/2 + 1`。下面我们看看在出块时Recents字段是如何起作用的：
```go
func (c *Clique) Seal(chain consensus.ChainReader, block *types.Block, results chan<- *types.Block, stop <-chan struct{}) error {
  ......

  for seen, recent := range snap.Recents {
    if recent == signer {
      // Signer is among recents, only wait if the current block doesn't shift it out
      if limit := uint64(len(snap.Signers)/2 + 1); number < limit || seen > number-limit {
        log.Info("Signed recently, must wait for others")
        return nil
      }
    }
  }

  ......
}
```

在准备为新的block签名时，会判断当前的签名者是不是在Recents中，如果在则不再签名。（这里仍然有对limit的判断，但我觉得其实是没有必要了。也可能是我没看懂......）


最后，我们要说一下Snapshot的创建过程。前面其实提到过，Snapshot的创建分两步：一是在某个checkpoint(包括genesis block)上调用`newSnapshot`生成一个Snapshot对象，然后调用这个对象的apply方法。这两步被封装在了`Clique.snapshot`方法中，即`Clique.snapshot`才是正确生成Snapshot对象的方法。因此我们来详细看一下Clique的snapshot方法。下面是摘录的代码，为了更清楚的表达逻辑，对代码进行了简化，并加了一些伪代码：
```go
func (c *Clique) snapshot(chain consensus.ChainReader, number uint64, hash common.Hash, parents []*types.Header) (*Snapshot, error) {
  headers []*types.Header

  for snap == nil {
    //首先从缓存或数据库中查找
    if snapshot in cache or database {
      get and break
    }

    //既不在缓存中也不在数据库中，那么看是否是创世块或没有父块的checkpoint。
    //关于为什么要判断没有父块的checkpoint，后面有详细说明
    if number == 0 || (number%c.config.Epoch == 0 && chain.GetHeaderByNumber(number-1) == nil) {
      checkpoint := chain.GetHeaderByNumber(number)

      //从checkpoint中取出signers列表
      signers := make([]common.Address, (len(checkpoint.Extra)-extraVanity-extraSeal)/common.AddressLength)
      for i := 0; i < len(signers); i++ {
        copy(signers[i][:], checkpoint.Extra[extraVanity+i*common.AddressLength:])
      }

      //调用newSnapshot在checkpoint上创建Snapshot对象，并将其存入数据库中
      snap = newSnapshot(c.config, c.signatures, number, checkpoint.Hash(), signers)
      snap.store(c.db)
      break
    }

    //如果以上情况都不是，则往前回溯区块的链，并保存回溯过程中遇到的header
    header := get parent by number and hash
    headers = append(headers, header)
    number, hash = number-1, header.ParentHash
  }

  //把headers中保存的头从按高度从小到大排序。
  reverse(headers)

  //将回溯中遇到的headers传给apply方法，得到一个新的snap对象
  snap, err := snap.apply(headers)

  //保存snap对象
  store snap to cache
  if should store to database {
    snap.store(c.db)
  }
  return snap, err
}
```

从上面简化的代码中可以看到，想要创建一个Snapshot对象，需要从给定的block开始向前回溯，直到从缓存或数据库中找到对应的Snapshot，或者满足`number == 0 || (number%c.config.Epoch == 0 && chain.GetHeaderByNumber(number-1) == nil)`这个奇怪的条件。然后将回溯过程中遇到的header传给apply。在apply内部会根据传入的headers逐个统计投票等信息。

关于上面提到的这个奇怪的判断条件的分析，参看[奇怪的判断条件](#odd_if)这一小节里的说明。

##### inturn and noturn
前面说过，clique作为PoA的实现，挖矿的人之间是合作关系，因此需要有规则规定某一时刻应该由谁出块。在clique中，`inturn`状态代表的是“按道理轮到我出块了”，而`noturn`正好相反。

在代码中，inturn的值为`diffInTurn`，noturn的值为`diffNoTurn`。Header.Difficulty字段用来保存相应的值，它的计算方式非常简单，具体可以查看Snapshot.inturn方法，这里不再多说。

在`Clique.Seal`方法中，签名时会进行一定时间的等待。如果Header.Difficulty的值为`diffNoTurn`，则会比`diffInTurn`的块随机多等待一些时间，通过这种方式可以保证轮到出块的人可以优先出块。代码如下：
```go
func (c *Clique) Seal(chain consensus.ChainReader, block *types.Block, results chan<- *types.Block, stop <-chan struct{}) error {
  ......

  //计算正常的等待出块时间
  delay := time.Unix(header.Time.Int64(), 0).Sub(time.Now()) // nolint: gosimple
  if header.Difficulty.Cmp(diffNoTurn) == 0 {
    //没有轮到我们出块，多等一会
    // It's not our turn explicitly to sign, delay it a bit
    wiggle := time.Duration(len(snap.Signers)/2+1) * wiggleTime
    delay += time.Duration(rand.Int63n(int64(wiggle)))

    log.Trace("Out-of-turn signing requested", "wiggle", common.PrettyDuration(wiggle))
  }

  ......
}
```


### 出块流程
在理解了前面提到的一些概念以后，我觉得再加上一个整体的视图，clique模块就非常容易理解了，因此这里我们从一个更高的视角来看一下clique的工作流程。

Clique对象包含两大块功能：Header的有效性验证和生成新的Header。有效性验证的方法全都是以"Verify"开头，比较容易理解，就不多说了。这里主要介绍一下生成新的block的流程。我将关键信息整理成了一张流程图。由于Clique对象只是实现了以太坊中共识接口的方法，主要流程还是由miner模块控制，因此这里也加入了简单的miner模块的流程。

![](/pic/ethereum-clique/flowchart.png)

这张图里隐藏了Snapshot的功能。整个出块的功能主要由Prepare和Seal完成。在Prepare中准备一些与PoA相关的信息，在Seal中进行签名出块。需要特别注意的是，出块的时间是在Seal中控制的，而非miner中。


# 关键问题
前面的分析已经涉及了几乎clique模块的所有重点，但为了清晰，这里把一些需要重点关注的关键问题单独拿出来，再针对性的进行一次说明。

### 如何控制出块时机
所有的共识机制都需要有一套控制出块时机的方法，因为不可能在同一时间让所有人都出块。clique是通过两个方面对出块时机进行控制的：
1. recent列表
2. inturn或noturn

在clique中，不允许最近出过块的人再出块，必须间隔一定的高度，这是通过`Snapshot.Recents`字段控制的。在`Clique.Seal`方法中有如下一段代码：
```go
  for seen, recent := range snap.Recents {
    if recent == signer {
      // Signer is among recents, only wait if the current block doesn't shift it out
      if limit := uint64(len(snap.Signers)/2 + 1); number < limit || seen > number-limit {
        log.Info("Signed recently, must wait for others")
        return nil
      }
    }
  }
```

代码中`snap.Recents`保存的就是最近出的块的高度和块的签名者（signer），`seen`变量是块的高度，`recent`是这个块的签名者。这段代码表达的意思是，如果当前的签名者刚出过块（在Recents中），并且这个历史块的高度离新出块的高度相差在所有签名者数量的一半（严格来说是一半加1）以内，则不允许再出块。

比如目前共有7个签名者，将要新出的块高度为100，而我刚出过一个块的高度是97，那么这个高度为100的块我就不能再出了。（97 > 100 - (7/2 + 1)）

那么snap.Recents又是怎么来的呢？它是在`Snapshot.apply`中生成的：
```go
  // Delete the oldest signer from the recent list to allow it signing again
  if limit := uint64(len(snap.Signers)/2 + 1); number >= limit {
    delete(snap.Recents, number-limit)
  }
  // Resolve the authorization key and check against signers
  signer, err := ecrecover(header, s.sigcache)

  snap.Recents[number] = signer
```

可以看到在填充Recents时，会将高度比当前块小太多的块从Recents中踢出去（number - limit）。

从上面的分析中我们可以看到，在调用Seal时，其实是有大约一半数量的签名者都是可以出块的。那这一半的签名者都要出块吗？当然不是的。理想情况下，每一个时刻最好只有一个人出块。在clique中是使用inturn/noturn的机制实现的。

在前面的概念介绍时我们已经介绍过inturn概念，简单来说就是“当前是不是轮到我出块了”，判断方法是看当前块的高度是否和自己在签名者列表中的顺序一致。`Snapshot.inturn`方法实现了这一功能：
```go
func (s *Snapshot) inturn(number uint64, signer common.Address) bool {
  signers, offset := s.signers(), 0
  for offset < len(signers) && signers[offset] != signer {
    offset++
  }
  return (number % uint64(len(signers))) == uint64(offset)
}
```

在`Clique.Prepare`中会将“是否轮到自己”的值写到Header.Difficulty字段中：
```go
func (c *Clique) Prepare(chain consensus.ChainReader, header *types.Header) error {
  ......
  header.Difficulty = CalcDifficulty(snap, c.signer)
  ......
}

func CalcDifficulty(snap *Snapshot, signer common.Address) *big.Int {
  if snap.inturn(snap.Number+1, signer) {
    return new(big.Int).Set(diffInTurn)
  }
  return new(big.Int).Set(diffNoTurn)
}
```

然后在`Clique.Seal`中，会根据Header.Difficulty中的值判断“是否轮到自己”出块。如果不是，则要随机多等待点时间，这就给了该出块的人优先的出块权：
```go
func (c *Clique) Seal(chain consensus.ChainReader, block *types.Block, results chan<- *types.Block, stop <-chan struct{}) error {
  ......
  delay := time.Unix(header.Time.Int64(), 0).Sub(time.Now())
  if header.Difficulty.Cmp(diffNoTurn) == 0 {
    // It's not our turn explicitly to sign, delay it a bit
    wiggle := time.Duration(len(snap.Signers)/2+1) * wiggleTime
    delay += time.Duration(rand.Int63n(int64(wiggle)))
  }
}
```

所以，clique通过recent和inturn两种方式控制出块时机，最终给了该出块的人优先的出块权。


### 如何动态调整签名者列表
前面我们也说过，在项目运行过程中，签名者可能会发生变化，需要一种机制能动态的调整签名者名单。在clique中，这种调整是通过投票实现的。即对是否要加入或踢除某人，现有签名者可以发起投票，当投票超过半数时投票通过，被投人被自动加入或踢出。下面我们来看看这种投票机制是怎么实现的。

我们先来看看如何发起投票。你可以在console中调用`clique.propose`来进行一次投票，比如：
> clique.propose("0x8D5cC3e43CE479d81c7e1e4a6DebC8D6c126a9eF", true)

调用此方法以后，投票信息就会写入`Clique.proposals`字段中。在随后出块调用`Clique.Prepare`方法时，会从`Clique.proposals`中随机选择一条投票信息，放到Header.Coinbase和Header.Nonce中：
```go
func (c *Clique) Prepare(chain consensus.ChainReader, header *types.Header) error {
  ......

  //非checkpoint才可以携带投票信息
  if number%c.config.Epoch != 0 {
    //从c.proposals中收集所有有意义的被投地址
    addresses := make([]common.Address, 0, len(c.proposals))
    for address, authorize := range c.proposals {
      if snap.validVote(address, authorize) {
        addresses = append(addresses, address)
      }
    }

    //如果确实有有意义的被投地址，随机选一个填入Header中
    if len(addresses) > 0 {
      header.Coinbase = addresses[rand.Intn(len(addresses))]
      if c.proposals[header.Coinbase] {
        copy(header.Nonce[:], nonceAuthVote)
      } else {
        copy(header.Nonce[:], nonceDropVote)
      }
    }
  }

  ......
}
```

现在票已经投出去了，我们再来看看如何统计投票信息，以及投票通过后如何生效。这些功能都是在`Snapshot.apply`方法中实现的：
```go
func (s *Snapshot) apply(headers []*types.Header) (*Snapshot, error) {
  for _, header := range headers {
    number := header.Number.Uint64()

    //如果达到一个epoch，则清空当前所有的投票信息
    if number%s.config.Epoch == 0 {
      snap.Votes = nil
      snap.Tally = make(map[common.Address]Tally)
    }

    //如果当前的投票人已经给某人投过票，则清除之前的投票信息
    //这保证了一个epoch内一个signer只能给某人投一次票，或用来撤消投票
    for i, vote := range snap.Votes {
      if vote.Signer == signer && vote.Address == header.Coinbase {
        snap.uncast(vote.Address, vote.Authorize)
        snap.Votes = append(snap.Votes[:i], snap.Votes[i+1:]...)
        break
      }
    }

    //将本次的投票信息纳入统计
    var authorize bool
    switch {
      case bytes.Equal(header.Nonce[:], nonceAuthVote):
        authorize = true
      case bytes.Equal(header.Nonce[:], nonceDropVote):
        authorize = false
      default:
        return nil, errInvalidVote
    }
    if snap.cast(header.Coinbase, authorize) {
      snap.Votes = append(snap.Votes, &Vote{
        Signer:    signer,
        Block:     number,
        Address:   header.Coinbase,
        Authorize: authorize,
        })
    }

    //如果投票数量超过一半，则投票通过。
    if tally := snap.Tally[header.Coinbase]; tally.Votes > len(snap.Signers)/2 {
      //如果是加入票，则将被投人加入到Signers列表中
      //之后此人就可以出块了
      if tally.Authorize {
        snap.Signers[header.Coinbase] = struct{}{}
      } else {
        //如果是踢出票，则将被投人从Signers列表中踢出
        //之后这人就无法出块了
        delete(snap.Signers, header.Coinbase)

        //将被投人从Recents信息中删除
        if limit := uint64(len(snap.Signers)/2 + 1); number >= limit {
          delete(snap.Recents, number-limit)
        }

        //丢弃被投人之前投出的所有投票
        for i := 0; i < len(snap.Votes); i++ {
          if snap.Votes[i].Signer == header.Coinbase {
            snap.uncast(snap.Votes[i].Address, snap.Votes[i].Authorize)
            snap.Votes = append(snap.Votes[:i], snap.Votes[i+1:]...)
            i--
          }
        }
      }

      //清除所有被投人是当前生效人的投票信息
      for i := 0; i < len(snap.Votes); i++ {
        if snap.Votes[i].Address == header.Coinbase {
          snap.Votes = append(snap.Votes[:i], snap.Votes[i+1:]...)
          i--
        }
      }
      delete(snap.Tally, header.Coinbase) //清空当前生效人的票数统计信息
    }
  }
}
```

这段代码比较复杂，但通过代码中的中文注释，相信可以很容易弄明白其基本逻辑：当我们逐个遍历链上的block时，我们记录每个block上的投票信息，当投票总数达到半数以上时，对被投人作相应的操作------加入或踢除（踢除包含了更多操作）；如果遇到epoch，则之前统计的所有投票信息作废，全部清空（这在epoch概念中有详细讨论）。


### 作恶的代价和处理
PoA机制无法保证所有签名者都不作恶。那么万一有人作恶，会造成什么影响呢？

相信经过前面不断的介绍，读者已经能理解clique的基本原理。签名者在clique中无法连续出块（Recent机制限制），因此如果有人作恶，它最多也只能每隔“出块人数量一半”的高度出一个块，这可以保证多数块是正确的，这一点与PoW类似（只要作恶者的能力（对于PoA来说是作恶的人的数量，对于PoW来说是算力）不超过一半，就可以保证多数块是正常的）。

clique有投票机制，如果及时发现某人作恶，其他签名者可以快速反应，将其强制踢出签名者列表。所以总得来说，clique的实现可以保证少量人作恶的情况下只能造成很小的影响，并且可以及时强制制止。

<a id="odd_if" />
### 奇怪的判断条件
还记得在前面的分析Snapshot时，我们在`Clique.snapshot`方法中遇到了一个奇怪的判断条件吗？
```go
  if number == 0 || (number%c.config.Epoch == 0 && chain.GetHeaderByNumber(number-1) == nil) {
    新生成一个Snapshot对象
  }
```

`Clique.snapshot`是用来创建Snapshot对象的，有三种情况可以创建：要创建的Snapshot在缓存中，或者在数据库中，或者满足上面的判断条件。

这个判断条件从语法上来说不算复杂，它要表达的意思是：如果当前是一个创世块或是一个不存在父块的checkpoint（number是epoch的整数倍），则进行Snapshot的创建。

创世块的判断我能理解，但为什么非得是不存在父块的checkpoint才能创建呢？

其实这个地方的if语句最开始不是这样写的。从github上clique.go文件最早的提交记录（commit id [feeccdf4ec1084b38dac112ff4f86809efd7c0e5](https://github.com/ethereum/go-ethereum/commit/feeccdf4ec1084b38dac112ff4f86809efd7c0e5#diff-f3a3b461458aa6a0718cad17e056c5c4R373)）上可以看到，最初的代码是这样子的（线上代码在[这里](https://github.com/ethereum/go-ethereum/blob/feeccdf4ec1084b38dac112ff4f86809efd7c0e5/consensus/clique/clique.go#L373)）：
```go
  if number == 0 {
    新生成一个Snapshot对象
  }
```

最初只有创世块才进入到这个分支。这导致如果你的块都是从别的节点同步过来的（因而缓存和数据库中都没有Snapshot被保存过），那么想要在某个高度上创建一个Snapshot，就要把所有区块从头到尾都遍历一遍。

后来在为clique模块填加轻客户端同步支持的时候，作者希望创建Snapshot的时候不需要从头到尾将所有块遍历一遍去创建，而是直接信任某个checkpoint，从这个checkpoint创建就可以了，就像信任genesis一样，因此改成了这样（commit id [9f036647e4b3e7c3aa8941dc239f85326a5e5ecd](https://github.com/ethereum/go-ethereum/commit/9f036647e4b3e7c3aa8941dc239f85326a5e5ecd#diff-f3a3b461458aa6a0718cad17e056c5c4R391)，线上代码在[这里](https://github.com/ethereum/go-ethereum/blob/9f036647e4b3e7c3aa8941dc239f85326a5e5ecd/consensus/clique/clique.go#L391)）：
```go
  if number%c.config.Epoch == 0 {
    新生成一个Snapshot对象
  }
```

但这样修改以后出现了问题：如果从某个checkpoint直接创建Snapshot，那么Recents字段是空的，但实际情况是这个checkpoint前面肯定是有最近出过块的人（即Recents按道理肯定不为空。这与创世块不同，创世块前面确实没有出过块的人），也就是说这种情况下所有的签名者都可能创建一个块，并成功加入到链中，但很可能这个签名者刚刚已经创建过一个块。这违反了clique的原始设计（作者针针对这个问题的说明在[这里](https://github.com/ethereum/go-ethereum/pull/17620)）。在意识到这个问题以后，作者更加严格的限制了对checkpoint的信任：只有从区块链的中间某个位置同步区块时（这种情况下就可能会缺失父块），才使用这个checkpoint创建Snapshot，否则依然遍历整个链条创建Snapshot。所以代码又改成了现在的样子（commit id [bcfb7f58b93e6fb5f3da0000672adee80fd6a485](https://github.com/ethereum/go-ethereum/commit/bcfb7f58b93e6fb5f3da0000672adee80fd6a485#diff-f3a3b461458aa6a0718cad17e056c5c4R391)，线上代码在[这里](https://github.com/ethereum/go-ethereum/blob/bcfb7f58b93e6fb5f3da0000672adee80fd6a485/consensus/clique/clique.go#L391)）：
```go
  if number == 0 || (number%c.config.Epoch == 0 && chain.GetHeaderByNumber(number-1) == nil) {
    新生成一个Snapshot对象
  }
```

经过对改动过程的分析和作者对问题的说明，我们可以看到，问题主要出对区块同步的支持上。如果不是忽然同步大量区块，那么在缓存或数据库中肯定存在最近创建的Snapshot，无需遍历到创世块就可以创建Snapshot；但为了支持只同步部分区块（如从区块的中间某个checkpoint开始同步），就需要可以从任意一个checkpoint为基础创建Snapshot。在权衡之后，则只允许从没有父块的checkpoint为基础创建Snapshot，对于其它checkpoint仍然需要向前查找父块。

（这个改动进程产生了bug，有人使用稍旧的版本产生区块以后，使用最新的版本进行同步，产生了错误。详见[这里](https://github.com/ethereum/go-ethereum/issues/17849)）


# 总结
以太坊中clique模块作为PoA的一个实现，目前在官方应用上只用于[测试网络](https://faucet.rinkeby.io/)，但我们自己创建私人网络时也可以使用。本篇文章里，我们详细分析了clique代码中的一些基本概念和与PoA相关的一些关键问题，并描绘了一个大体的出块流程。

clique的代码虽然较少，但共识是区块链项目的核心之一，因此我在文章中尽量详细的对代码和原理进行了说明，有些地方甚至宁愿有重复。文章若有失误和我理解错误的地方，还望读者不吝指正。
