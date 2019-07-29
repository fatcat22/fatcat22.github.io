---
layout: post
title:  "如何搭建以太坊测试环境"
categories: ethereum
tags: 原创 ethereum test-envirment 
excerpt: 如何在自己机器上搭建一个以太坊的测试环境呢？这篇文章手把手教给你。

author: fatcat22
---

* content
{:toc}





# 引言

本来没想写这么一篇的，但我自己在学习和调试以太坊代码和合约的过程中，在自己机制上搭建过好几次以太坊私有网络，每次都会忘记点什么；另外在有些文章中表达想要完整，必须加入测试环境搭建的说明。所以干脆就单独写一篇，详细描述一下搭建测试环境吧。

在这篇文章中，我把测试环境分成了两类，一类是加入官方的测试网络；另一类是在自己的机器上搭建多于两个节点的私有网络。下面我们分别说明。


# 编译 geth

由于我创建测试网络的主要目的是了解以太坊代码的运行机制，因此本篇文章里只介绍使用 geth 程序运行以太坊节点创建测试网络的情况。至于使用其它以太坊客户端加入官方测试网络的情况，本文不进行介绍。

geth 程序可以从以太坊源代码中编译得到。编译方法很简单，下载源码[go-ethereum](https://github.com/ethereum/go-ethereum)后，进入项目主目录，运行 `make` 成功后，就可以在 build/bin/geth 路径下得到 geth 程序。当然前提是你得安装好 go 语言和 make 相关程序。


# 加入官方测试网络

随着以太坊的发展，以太坊官方的测试网络有好几个版本，目前用得比较多的被称为「 Rinkeby 」。「 Rinkeby 」是一个使用 PoA 共识的测试网络，它的区块链浏览器网址为[https://rinkeby.etherscan.io](https://rinkeby.etherscan.io)。水龙头地址为[https://faucet.rinkeby.io](https://faucet.rinkeby.io)。以下是加入测试网络的详细步骤：

1. 创建以太坊账户  
首先得有自己的以太坊账户地址。这可以通过命令 `geth account new` 创建。如果你不想使用默认的数据目录，可以指定数据目录参数：`geth --datadir your/data/path account new`。注意记住自己创建过程中输入的密码。
2. 运行以太坊节点  
有了账户以后，就可以运行本地节点加入测试网络。运行的命令为：`geth --rinkeby --unlock 0`。运行后会提示让你输入第一步中创建账户时设置的密码。（如果创建账户时用的不是默认数据目录，运行时需要加上 `--datadir your/data/path` 参数）

加入 「Rinkeby」 网络就这么简单。但仅仅加入网络我们能做的事情很少，还需要我们自己的账户中存在一些以太币，以便我们可以测试转账、合约等。我们可以从前面提到的[官方水龙头](https://faucet.rinkeby.io)获取测试网的以太币，根据主页上的「 How does this work 」提示的操作即可。目前的提示里需要用到 Twitter 或 Facebook 账户，所以不能翻墙的小伙伴会麻烦些。（以前可以使用 github 账号，现在不能用了）

当你的账户里有币以后，你就可以随心所欲的测试你想测试的功能啦。


# 自建私有网络

加入官方的测试网络很方便，但也有可能满足不了我们的需求。比如我们想指定特定的共识算法，或者想让自己的账户有更多的以太币供使用，或者电脑不能联网，又或者觉得测试网同步区块太慢又占磁盘空间。自己搭建一个私有网络可以随心所欲的进行配置，满足自己的任意需求。

**注意我们并不需要多台机器来搭建一个以太坊私有网络，只需要在一台机器上，就可以运行以太坊多个节点，组成一个小的私有网络。**

以下是搭建私有网络的步骤（一般你的私有网络有三个节点就可以，其中至少两个启动挖矿功能（因为只有一个矿工时，PoA 无法正常出块））：

**1.创建各节点的 datadir**

由于我们要在同一台机器上运行多个节点，所以不能使用默认数据路径，需要为不同节点创建不同的数据路径。为了说明方便，我们假设要创建三个节点，那么就需要三个目录，比如：  
./node0  
./node1  
./node2

**2. 创建以太坊账户**

你想启动哪个节点，就需要创建几个账户。建议至少两个账户，因为在配置矿工账户时，最好配置两个或两个以上的矿工，这样在挖矿时由 clique 模块实现的 PoA 共识才能正常出块。（一般创建三个账户就能满足所有测试需求）

我们已经创建了各节点的数据目录，因此要将账户创建到各自的数据目录中。以之前示例的 node0、node1、node2 举例，你可以这样创建三个以太坊账户：
```
geth --datadir ./node0 account new
geth --datadir ./node1 account new
geth --datadir ./node2 account new
```

注意记住创建账户时你配置的账户密码。

**3. 创建私有网络配置**

使用 `puppeth` 命令创建私有网络配置（和 `geth` 在同一目录） 。运行此程序后会通过问题提示的方式让你一步步进行配置，过程中会用到前面创建的账户地址。过程如下：
> 运行 puppeth  
> ...... // 一些说明信息  
>  
> // 输入一个 network 名字，只是为了标识生成的配置信息所在的文件  
> // 带有 > 提示符开头的是我的输入内容  
> Please specify a network name to administer (no spaces, hyphens or capital letters please)  
> \> myconfig
>
> // 注意这两行 log 里显示的最后存储配置信息的位置，在 ~/.puppeth/myconfig 文件中  
> INFO [07-29|20:16:20.879] Administering Ethereum network           name=myconfig  
> WARN [07-29|20:16:20.882] No previous configurations found         path=/Users/usrname/.puppeth/myconfig  
>
> // 选择配置新的 genesis 信息  
> What would you like to do? (default = stats)  
> 1 . Show network stats  
> 2 . Configure new genesis  
> 3 . Track new remote server  
> 4 . Deploy network components  
> \> 2  
>
> What would you like to do? (default = create)  
> 1 . Create new genesis from scratch  
> 2 . Import already existing genesis  
> \> 1  
>
> // 选择使用的共识机制。如果没有特殊要求非得用 Ethash，建议选择 Clique，出块更规律、更可控一点。  
> Which consensus engine to use? (default = clique)  
> 1 . Ethash - proof-of-work  
> 2 . Clique - proof-of-authority  
> \> 2  
>
> // PoA 共识的出块时间间隔，这里我选择 5 秒  
> How many seconds should blocks take? (default = 15)  
> \> 5
>
> // 允许哪些账户可以挖矿。至少选择一个前面步骤中创建的账户地址（我的建议至少两个，理由前面说了）  
> // 这里我将三个刚才创建的账户地址全输入了，这样当节点运行时，这些账户都有权限进行挖矿。  
> // 输入完毕后，直接按回车键，进入下一项  
> Which accounts are allowed to seal? (mandatory at least one)  
> \> 0x7656bde2ff583c0b0817386e7b88eb8991f4d90c  
> \> 0x4b40b728ab138dcfd641aa4592fb58349c4bd0c3  
> \> 0xfdabd4d5728b0203ac028e32dc540a85ba99fe4d   
> \> 0x
>
> // 这里输入的账户里一开始就有很多以太币  
> // 至少输入一个前面步骤中创建的新账户。这里我将三个账户都输入了  
> // 输入完毕后，直接按回车键，进入下一项  
> Which accounts should be pre-funded? (advisable at least one)  
> \> 0x7656bde2ff583c0b0817386e7b88eb8991f4d90c  
> \> 0x4b40b728ab138dcfd641aa4592fb58349c4bd0c3  
> \> 0xfdabd4d5728b0203ac028e32dc540a85ba99fe4d   
> \> 0x
>
> // 如果选 yes，那么从 0x0000000000000000000000000000000000000000 到  
> // 0x00000000000000000000000000000000000000ff 这些地址都会有 1wei 的以太币。  
> // 这个看情况选择，用得着就选 yes  
> Should the precompile-addresses (0x1 .. 0xff) be pre-funded with 1 wei? (advisable yes)  
> \> no
>
> // 指定 network ID。直接按回车键随机生成  
> Specify your chain/network ID if you want an explicit one (default = random)  
> \>   
> INFO [07-29|20:31:57.518] Configured new genesis block
>   
> // 生成成功，此时所有配置已写入 ~/.puppeth/myconfig 中。  
> // 按 Ctrl+C 退出程序  
> What would you like to do? (default = stats)  
> 1 . Show network stats  
> 2 . Manage existing genesis  
> 3 . Track new remote server  
> 4 . Deploy network components  
> \> ^C

`puppeth` 程序运行完成后，生成的是一个 json 格式的文件，文件如下所示：
```
{  
  "genesis": {  
    "config": {  
      "chainId": 56680,  
      "homesteadBlock": 1,  
      ......  
    }  
    ......  
    "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000"  
  }  
}
```

但是 `geth` 在使用这个配置文件时，只使用「 “genesis” 」这个字段，所以你应该将 "genesis" 字段的内容作为这个 json 文件的内容（相当于删除`{    "genesis": `这段字符和文件末尾最后一个`}`）。修改后如下：
```
{
  "config": {
    "chainId": 56680,
    "homesteadBlock": 1,
    ......
  }
  ......
  "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000"
}
```

**4.初始化节点**

现在有了私有网络的配置文件，就可以利用这个配置对每个节点进行初始化了。假设这个配置文件就是之前举例的 ~/.puppeth/myconfig，而数据目录也是之前举例的 node0、node1、node2，那么可以使用以下命令初始化节点：
```
geth --datadir ./node0 init ~/.puppeth/myconfig
geth --datadir ./node1 init ~/.puppeth/myconfig
geth --datadir ./node2 init ~/.puppeth/myconfig
```

**5. 配置 p2p 发现网络**

每个节点的 p2p 模块在启动时，需要知道一个或几个初始的 p2p 地址，p2p 模块会先连接这些地址，然后从这些地址获取到其它节点的地址并再次连接，这样所有的节点才能加入到同一个网络中，并组成一个大的 p2p 网络。

继续沿用之前的例子，假设我们要创建的三个节点为 node0、node1、node2。我们可以任选一个或多个节点作为主节点。比如我们选 node0 节点，那么我们首先要知道 node0 节点的 p2p 地址，这可以通过运行一下这个节点得到：
```
geth --datadir ./node0
```
运行后可以按 ctrl+C 退出程序。在输出 log 中，有这样一行：
```
INFO [07-29|20:42:37.353] Started P2P networking                   self=enode://1cd68168360270a7ff43075bca80074156b080b37dc0a7be0ed4e6f73d8b6dacd39ef2a77fbaf1ce90e376de6b2d5cf776bb186460a01d2d6bce42e3995d572c@127.0.0.1:30303
```

其中 self 后面的地址，就是当前节点的 p2p 地址，此例中是 "enode://......@127.0.0.1:30303"

我们需要将这个地址写入每个节点的数据目录中的 static-nodes.json 文件内，这是一个 json 文件，节点地址组成了一个数组。以刚才的地址为例：
```
[  
"enode://1cd68168360270a7ff43075bca80074156b080b37dc0a7be0ed4e6f73d8b6dacd39ef2a77fbaf1ce90e376de6b2d5cf776bb186460a01d2d6bce42e3995d572c@127.0.0.1:30303"  
]
```

当然你也可以将多个地址写入这个文件中。比如我们将三个节点的地址都写入这个文件中：
```
[  
"enode://1cd68168360270a7ff43075bca80074156b080b37dc0a7be0ed4e6f73d8b6dacd39ef2a77fbaf1ce90e376de6b2d5cf776bb186460a01d2d6bce42e3995d572c@127.0.0.1:30303",  
"enode://7022084f144acb9845199310f5d89c68fcab35885ca2fec96239ecc018d675171fd7d1d7dfb99efbb9f6eec1d0711d5e36ba76010f0d5d3b3ebdc498cca690f5@127.0.0.1:30304",  
"enode://efab560571e54235a696680fc2387bce028d4cb4ce51b844e38d78b5eee490fc3fe8f51377492f57a4c1fb8220c89f61874e08fee7a96e2570d46ff9e4501204@127.0.0.1:30305"
]
```

需要注意的是，当你写多个地址时，地址最后面的端口号不能重复，并且要与启动 `geth` 时的 `--port` 参数一致（后面会讲到）。

另外，这些内容必须保存在每个节点的数据目录的 static-nodes.json 文件中，不可以使用其它文件名和路径。在我们的三个节点的例子中，这个此内容必须被保存在 ./node0/static-nodes.json、./node1/static-nodes.json、./node2/static-nodes.json 这三个文件中，文件内容可以一样。

**6.启动节点**

最后，我们只要启动各个节点就可以了。使用以下命令和参数启动节点：
```
geth --datadir ./node0 --unlock 0 --mine --rpc 
geth --datadir ./node1 --unlock 0 --mine --port 30304
geth --datadir ./node2 --unlock 0 --mine --port 30305
```

注意除了第一个节点，其它节点都没有启动 rpc 功能。一是因为 rpc 功能有一个节点启动就够用了；二是因为多个节点启动 rpc 功能，rpc 端口被第一个启动的节点占用，后面的启动就会报失败。

如果你不想某个节点进行挖矿，就去掉启动参数中的 `--mine` 参数。

最后要注意的是 `--port` 参数，这个参数指定了 p2p 连接的端口，需要与 static-nodes.json 中的一致，且不同节点不能重复。


# 总结

本篇文章没有什么太多内容，只是一个搭建测试网络的步骤说明。如果有错误的地方，感谢指正。