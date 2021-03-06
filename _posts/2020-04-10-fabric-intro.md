---
layout: post
title:  "初识联盟链1: Fabric 是什么"
categories: blockchain
tags: 原创 fabric 
excerpt: 本篇文章简单介绍了联盟链和联盟链的知名项目 Fabric 。这里是一个简单的介绍，因此读者不需要有联盟链甚至区块链的相关知识，但最好有区块链的基本概念。读完本篇后，读者能够了解 Fabric 的一些特点，并对 Fabric 网络形成一个具体的印象。
author: fatcat22
---

* content
{:toc}




> 本篇文章简单介绍了联盟链和联盟链的知名项目 Fabric 。这里是一个简单的介绍，因此读者不需要有联盟链甚至区块链的相关知识，但最好有区块链的基本概念。读完本篇后，读者能够了解 Fabric 的一些特点，并对 Fabric 网络形成一个具体的印象。


# 什么是联盟链

联盟链是区块链的一种。所谓联盟链，简单来说，是指由多个机构或组织组成的、并且共同控制的区块链。个人或组织想要接入某个联盟链，必须得到授权。

目前，在讨论区块链时，大家通常根据区块链的开放程度，将区块链公有链、私有链、联盟链三类。公有链就是比特币、以太坊那种，任何人都可以随时接入，获得链上的数据或向链上提交新的数据。当然，你也可以随时退出网络。私有链是指那些由单个机构或组织控制的区块链。这种区块链不开放给别人参与，只有自己内部使用。

而联盟链则介于公有链与私有链之间，它的成员由多个机构或组织组成。因此它不像私有链那样，由某个组织独自掌握控制权；但也不会像公有链那样，任何人都能访问。

联盟链的英文为「 Consortium Blockchain 」，我觉得「联盟」这个词翻译得非常好，即使不作任何解释，我们也能从「联盟链」这个名字里体会到它的特点。

（注意我们刚才在解释联盟链时，多次用到了「机构」、「组织」这样的词，后面的文章以及我们其它介绍联盟链的文章中，还会频繁用到这样的词汇。这两个词乍听上去比较虚、不知道是什么，其实你可以将其简单地理解为「公司」就好了）


# Fabric 又是什么

虽然我们刚才介绍了联盟链，但联盟链只是这类区块链的统称。落实在具体的实现上，则有不同的项目，比如 [
Hyperledger Fabric](https://www.hyperledger.org/projects/fabric) (后面我们简单为 Fabric )。

Hyperledger 是由 linux 基金会创立的一个项目，此项目致力于推动跨行业的区块链技术的发展。而 Fabric 则是 Hyperledger 下的一个子项目，它实现了一个联盟链的**框架**，并提供了许多区块链技术：比如它会维护一个账本（区块数据）、可以使用智能合约、参与者的交易由他们共同管理。

Fabric 与其它区块链项目最大的区别是，Fabric 会保护数据的私密性，并通过权限控制其他人（成员和非成员）对数据的访问。这一特性非常适合企业级的应用，因为任何一家企业都不会想其它人随便访问自己的数据。

注意刚才我们特意强调了**框架**。是的，Fabric 不是一个具体的区块链项目，而只是一个框架。一些组织想要创建他们自己的联盟链，需要在 Fabric 的基本上，实现自己的业务逻辑；但也仅仅是业务相关的逻辑需要自己实现。（我想这也体现了 Hyperledger 基金会的目标，即推动跨行业区块链技术的发展）

所以总得来说，Fabric 是一个出色的联盟链框架。利用它，各组织机构可以快速的布署自己的联盟链项目。那 Fabric 作为一个框架，提供了哪些功能呢？下面我们就来详细看看。


# Fabric 有哪些特性

首先，Fabric 是一个区块链项目，因此它拥有一般区块链所拥有的特性，包括多用户共识、分布式共享账本、智能合约等。同时它又有许多自己独有的特性，这些特性可以保证它适用于当前的企业运营环境。

- 准入限制

联盟链与公有链最大最直接的不同，就是成员接入的限制。在联盟链中，想要接入某个联盟链网络中，必须获得网络管理员的批准认可。

Fabric 使用一种叫作 MSP（Membership Service Provider ）的服务实现这种准入限制。每一个批准加入当前网络的用户，都会有一对由非对称加密算法生成的公私钥对。而成功运行的联盟链的 MSP 中包含了所有批准加入的用户的公钥。当发送交易等数据时，用户需要使用私钥对交易信息进行签名，MSP 则使用存储的公钥验证这些签名数据。如果验证失败，则不会处理这些交易信息，从而达到了将未批准的用户「拒之门外」的目的。

通常在实际应用中，各组织会使用某些可信任的证书认证机构（比如 Symantec 、 Comodo 等）来生成密钥对，公钥信息就放在 MSP 服务的配置中。而在搭建测试网络时，也可以使用 `cryptogen` 工具生成，此时无需认证机构。


- Shared Ledger

和普通区块链一样，Fabric 中每一个参与者都有属于自己网络内的账本的拷贝，正常情况下，同一网络内的账本数据是一致的。这也是 Fabric 的区块链特性。

在 Fabric 内部，有一个由两部分组成的账本系统：world state 和 transaction log （为了表达准备，这两个名字没有进行翻译，后文为了方便，我们将其分别简单为 state 和 log）。state 是一个数据库，用于描述某一时刻下账本的状态，而 log 记录按顺序记录了所有交易（Transaction）信息。正如你所想的那样，正是 log 中的交易叠加在一起，最终生成了 state 数据库的状态。所以 state 的存在，其实是为了加快账本的访问速度，而 log 才是区块链中那个真正的「链」。

需要提一下的是，我们刚才提到参与者有「属于自己网络内」的账本拷贝。这是是因为 Fabric 的账本其实是和网络对应的，这个网络叫做 channel。简单来理解，channel 是 Fabric 用来连接和隔离组织的一种方式。在同一 channel 内，各组织共享同一账本；channel 外的组织则无法访问这个账本（虽然这些组织也是这个联盟链的参与者）。我们会在文章的后面介绍更多关于 channel 的信息。


- Smart Contracts

Fabric 的智能合约（Smart Contracts）又叫 chaincode。在 Fabric 中，智能合约用来和账本进行交互：直接查询账本信息，或者生成一条可以修改账本信息的交易。所有使用联盟链服务的应用程序，都是通过智能合约使用联盟链的。

智能合约也是参与合作的各组织实现业务逻辑的主要地方。当一个智能合约编写完成后，就可以在链上进行布署。但只有所有相关的参与组织都批准认可这个智能合约，它才能真正的生效。

顺便说一下，Fabric 的智能合约的编写支持使用当前流行的编程语言（而不像以太坊那样，自己创造一种新的语言），这显然对开发智能合约的程序员非常友好，因为不用再费劲学一门新的编程语言了。当前 Fabric 的版本是 2.0，官方支持 go 和 Node.js。


- 数据隐私

在商业环境中，数据隐私是一个基本需求。没有一个非公益性的企业，愿意别人随便访问自己的所有数据。Fabric 作为一个企业级联盟链框架，自然也考虑到了这一点。

Fabric 系统使用 channel 将不同的组织连接起来。在一个实际的 Fabric 系统中，channel 将不同的组织连接在一起。在同一个 channel 内部，组织之间共享同一个账本数据；但 channel 外的成员无法访问这个账本数据。通过这种方式，保证了账本数据的隐密性。根据实际应用需求，一个 Fabric 系统中可能存在多个 channel，同一个组织也可以加入多个 channel，这样样的设计，可以让组织成员按业务需求分享不同的数据，从而保证数据的隐密性。

另外 Fabric 还提供了一种叫做「 gossip 」的通信方式，它允许即使在同一个 channel 内，也只有指定的成员才能访问某些数据。比如一个 channel 内有一个卖方和两个买方，其中一个买方与卖方达成了一个协议，将以更低的价格买入商品；但卖方不想另一个买方知道这笔交易的价格细节。使用 gossip ，卖方可以在同一个 channel 内，即实现这笔交易，又不会让另一个买方知道价格详情。


- Consensus

对于区块链来说，账本的每个副本都分布存储于各个节点上。正常情况下，不同节点的账本数据必须是一致的，即所有交易必须以相同的顺序被执行。共识（Consensus）就是用来完成这份工作的。

我想接触区块链技术的人，听到最多的词就是「共识」了，像比特币的 PoW，还有[之前的文章](https://yangzhe.me/2019/11/25/pbft/)中我们介绍过的 PBFT，这些都是共识算法。在 Fabric 的早期版本中，就是使用 PBFT 作为共识算法。但在目前的版本 2.0 ，已经使用更简单的 Raft 算法作为默认的共识算法。当然了，作为一个框架来说，Fabric 的共识算法是可以配置的，除了 Raft，还可以选用 Kafka （官方不推荐），甚至你可以自己实现自己的共识算法。

与比特币这些区块链项目非常不同的一点是， Fabric 中有单独的结点用来运行共识算法，对交易进行排序。这些节点被称为「 orderer 」。这些 orderer 节点将交易排序打包、生成 block 以后，再将 block 发送给各组织的节点进行处理。详细情况我们在以后的文章中还会进行介绍。


- channel

channel 是 Fabric 内部成员之间组网的一种方式。初次接触这个概念可能有些不好理解。首先要知道的一点是，与普通区块链项目不同，Fabric 内部成员是可以任意组网的，即一个 Fabric 联盟链内部，可能存在多个区块链网络，这一网络的实现就叫做 channel。

正如名字所表示的，channel 就像是一根根管道，将多个用户连接在一起。管道内的用户共享同一个账本，参与同一个记账过程；管道外的用户与管道内的用户是完全隔离的，访问不到管道内部的账本数据。

需要特别提一下的是，正如刚才所说，每一个 channel 对应着一个账本。而一个 Fabric 联盟链内部是可能存在多个 channel 的，因此也就可能存在多个账本。这与普通的区块链有很大的不同。比如在比特币网络中，所有参与者维护的都是同一个账本。但在 Fabric 中不是这样子的。在 Fabric 中，有多少个 channel，就有多少个账本；一个组织如果加入了多个 channel，就会同时维护多个账本。

channel 是 Fabric 中实现数据隐私的重要手段。设想一个公司参与到了它所在行业的一个 Fabric 联盟链中。在这里面，他可以共享给合作伙伴的数据，肯定不同于可以共享给竞争对手的数据；然而他们都在同一个联盟链项目中，要如何给不同的人共享不同的数据呢？有了 channel，这家公司就可以以身份甲进入到由合作伙伴组成的 channel A 中；然后以身份乙进入竞争对手存在的 channel B 中。这样一来，在 channel A 中，这家公司可以放心地分享一些数据给合作伙伴，而竞争对手不在这个 channel 里，也就看不到这些数据；而在 channel B 中，这家公司可以只分享少量的必要数据，以防其他竞争对手得到自己的机密商业信息。


# 一个 Fabric 网络长什么样

在介绍完 Fabric 的一些主要特性后，相信你一定好奇，一个 Fabric 联盟链到底是什么样子的。在[这篇](https://hyperledger-fabric.readthedocs.io/en/latest/network/network.html)官方文档中，有一个非常好的图例，我把它「汉化」了一下：

![](/pic/fabric-intro/network.diagram.png)

乍看这张图，可能会有点蒙。不过仔细看看，还是比较好理解的。只要注意一下图中所示的联盟链中，不同的组织分别用不同的颜色表示（如图例所示）。同一组织的所有元素，都使用同样的颜色标识。

参与这个联盟链的共有 4 个组织，其中 组织4 并没有参与实际业务，而只是提供了排序服务节点 orderer4；其他 3 个组织都有实际的业务逻辑，并各自提供了一个节点，用于保存账本、布署智能合约。另外通过网络配置文件（network configuration）可以看到，组织1 和组织4 拥有这个联盟链的管理员权限。

在这个联盟链内部，一共创建了 3 个 channel。其中一个 system channel 由排序服务器节点 orderer4 使用。另外两个 channel 分别用于实际业务的处理。其中 channel1 连接了 组织1 和 组织2，以及排序节点；channel2 连接了 组织2 、 组织3 和排序节点。channel 可以连接哪些组织，是由 channel 的配置文件来决定的。另外除了 组织4 没有实际业务逻辑外，其他 3 个组织都有自己的客户端程序，它们也被 channel 连接到了不同的网络中。

对于存在于同一 channel 的节点来说，它们的账本数据、智能合约都是一样的。比如同处于 channel1 的 节点1 和 节点2 来说，它们都保存了账本1，并布署了智能合约5；而对于同处理 channel2 的 节点2 和 节点3 来说，它们也同时保存了账本2，并布署了智能合约6。 

值得注意的是，图中的示例显示了同一组织可以加入不同 channel 的能力。比如对于 组织2，他即存在于 channel1 中，也存在于 channel2 中。可以看到，组织2 的客户端同时接入了这两个网络，他的的节点「节点2」也同时接入了 channel1 和 channel2。所以理所当然地，节点2 中同时存在了两个不同的账本，并布署了两个不同的智能合约。但这并不会影响数据的私密性：虽然节点2接入了两个 channel 中，但属于 channel1 的智能合约5仍然无法访问属于 channel2 的账本2；智能合约5只能访问账本1中的数据。

最后，在示意图的左下角还标出了 4 个认证机构的图标。这里的「认证机构」就是指那些数字签书颁发机构，比如常见的 Symantec 、 Comodo 等。这些机构为联盟链成员提供身份信息（简单来理解，就是密钥对）。（所以图中这些机构虽然也使用了与组织相同的颜色，但并不为组织所拥有，只是表示这个组织使用了某个认证机构的服务）


# 总结

这篇文章里，我们简单介绍了联盟链和 Fabric。联盟链是一种部分妥协的公有链：以牺牲部分去中心化的特性为代价，换取更高的交易效率。但对于参与的各组织来说，仍然保留了去中心化这一典型的区块链特性：每个组织都保留账本的幅本，交易的达成也仍需要多人形成共识才可以。

在联盟链这个世界里，Fabric 项目相对运行的时候比较久了，也比较知名。后面我们会写多篇文章来对其进行学习。除了 Fabric，还有一个叫做 [FISCO BCOS](http://www.fisco-bcos.org) 也很不错。这是我们国内开源维护的联盟链项目，非常值得期待。


# 参考
- [官网文档对 Fabric 的介绍](https://hyperledger-fabric.readthedocs.io/en/latest/blockchain.html)

