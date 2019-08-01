---
layout: post
title:  "认识以太坊智能合约----我的认知和手工布署"
categories: ethereum
tags: 原创 ethereum contract
excerpt: 以太坊合约是以太坊区别于其它区块链项目的重要功能。本篇文章介绍了我对以太坊智能合约的认知，以及如何手工布署。
author: fatcat22
---

* content
{:toc}





# 引言

在目前以太坊之外的各个区块链项目中，虽然也是声称利用区块链技术解决了各种各样的问题，但在技术层面，多数项目本质上与比特币一样：Transaction 保证了代币可以在不同的人手中收发；打包 Block 以保存 Transaction；使用不同的共识将 Block 形成链，以保证不可篡改。这一套做法的重点在「交易」上。但以太坊不同，以太坊在区块链的特性（无中心、不可篡改）之上提供了智能合约的功能，区块链技术（如打包 Block 并使用共识形成链）只是为智能合约提供底层支持。以太坊的重点在智能合约，而不是「交易」。因此，学习以太坊不能不学习智能合约，这篇文章就是我个人对以太坊智能合约的一些认知和布署等问题的介绍。

需要注意的是，这篇文章并不会深入的讲解如何去写一个合约。如果你想了解这方面的知识，可以去看[官方文档](https://solidity.readthedocs.io)（或[中文翻译版](https://solidity-cn.readthedocs.io)，但翻译版有延迟，可能比英文版版本老一些）。

另外这篇文章也不会详细讲解各种智能合约的开发框架（如 Truffle 和 Embark）。这篇文章介绍的，仅仅是我对智能合约的思考，和一些很基本的了解。原因我会在最后提及。

# 智能合约：事物发展的约定

在没有接触以太坊和智能合约以前，我总是想不明白智能合约加上区块链会有怎样的应用：「合约」一词代表某一事物经由双方或多方确认并无法修改，区块链就可以做到，那么把一纸静态的「合约」放到区块链里，有什么特殊的呢？

简单学习了一下合约，才恍然大悟：现实中合约是一张纸，是静态的，但以太坊中合约是动态的、可编程的。这里的合约不仅仅是约定好了一件或几件事情（比如张三借了李四100块钱，一年以后还），更是约定好了事情可以怎样发展，甚至可以约定不按照即定情况发展会有怎样的措施。由于这些约定是在区块链的基础之上，因此可以不用担心会被某一方篡改。

不得不感慨大神们的思维真是活跃。

下面我们来看看一般情况下，合约是如何被使用的。以官方文档中的一个投票的例子为例：
```
pragma solidity >=0.4.22 <0.7.0;

/// @title Voting with delegation.
contract Ballot {
    // 投票者信息。weight 代表此人的投票权重，一般为 1；如果为 2，可以理解成投一票顶一般人的两票
    struct Voter {
        uint weight;
        bool voted;
        address delegate; // person delegated to
        uint vote;   // index of the voted proposal
    }

    // 被投票人信息，包含名字和票数
    struct Proposal {
        bytes32 name;
        uint voteCount;
    }

    // 发起此次投票的人，即 「主席」
    address public chairperson;

    // 一个 map，记录所有投票人信息
    mapping(address => Voter) public voters;

    // 一个数组，记录所有被投票人信息
    Proposal[] public proposals;

    // constructor 在合约创建时被调用，相当于 C++ 的构造函数
    // 这里主要保存了创建此合约的人（即「主席」）的地址，并初始化所有被投票人信息
    constructor(bytes32[] memory proposalNames) public {
        chairperson = msg.sender;
        voters[chairperson].weight = 1;

        for (uint i = 0; i < proposalNames.length; i++) {
            proposals.push(Proposal({
                name: proposalNames[i],
                voteCount: 0
            }));
        }
    }

    // 「主席」调用此方法，可以开放给某人投票权限。
    // 注意只有「主席」可以调用这个方法
    function giveRightToVote(address voter) public {
        require(
            msg.sender == chairperson,
            "Only chairperson can give right to vote."
        );
        require(
            !voters[voter].voted,
            "The voter already voted."
        );
        require(voters[voter].weight == 0);
        voters[voter].weight = 1;
    }

    // 将投票权交给别人代理。如果有 10 个人交给 A 代理投票，
    // 那么 A 的权重为 10。（相当于 1 票顶 10票）
    // 这个方法复杂在处理多层代理上，即 A 交给 B 代理，B 又交给 C 代理的情况。可以不用细看。
    function delegate(address to) public {
        // assigns reference
        Voter storage sender = voters[msg.sender];
        require(!sender.voted, "You already voted.");

        require(to != msg.sender, "Self-delegation is disallowed.");

        while (voters[to].delegate != address(0)) {
            to = voters[to].delegate;

            // We found a loop in the delegation, not allowed.
            require(to != msg.sender, "Found loop in delegation.");
        }

        // Since `sender` is a reference, this
        // modifies `voters[msg.sender].voted`
        sender.voted = true;
        sender.delegate = to;
        Voter storage delegate_ = voters[to];
        if (delegate_.voted) {
            // If the delegate already voted,
            // directly add to the number of votes
            proposals[delegate_.vote].voteCount += sender.weight;
        } else {
            // If the delegate did not vote yet,
            // add to her weight.
            delegate_.weight += sender.weight;
        }
    }

    // 给某人投票。投票人必须有投票权限且没有行使过投票权。
    function vote(uint proposal) public {
        Voter storage sender = voters[msg.sender];
        require(sender.weight != 0, "Has no right to vote");
        require(!sender.voted, "Already voted.");
        sender.voted = true;
        sender.vote = proposal;

        // If `proposal` is out of the range of the array,
        // this will throw automatically and revert all
        // changes.
        proposals[proposal].voteCount += sender.weight;
    }

    // 返回得票最高的人的信息
    function winningProposal() public view
            returns (uint winningProposal_)
    {
        uint winningVoteCount = 0;
        for (uint p = 0; p < proposals.length; p++) {
            if (proposals[p].voteCount > winningVoteCount) {
                winningVoteCount = proposals[p].voteCount;
                winningProposal_ = p;
            }
        }
    }

    // 返回得票最高的人的名字
    function winnerName() public view
            returns (bytes32 winnerName_)
    {
        winnerName_ = proposals[winningProposal()].name;
    }
}
```
这是一个用来投票的智能合约，合约比较简单，不用学 solidity 语言，估计也能看懂大体的意思。这里我们不在意合约的编写细节，而是要看这个合约是如何被用来起的。

这个合约创建了这样一个投票情景：投票这件事情需要某个人发起，这个人也是创建这个合约的人，被称为「主席」。合约创建时，通过「构造函数」记录了所有被投票人的信息。「主席」创建合约后，需要将投票权限开放给可以投票的人，这之后这个人才能投票。最后，任何一个人都可以调用相应的方法，获取投票的结果。

可以看到，与普通脚本不同，solidity 语言增加了一些权限管理手段，可以约束方法的调用者和做的事情：只有特定的人才能调用某个方法，只有特定的人才能执行某个动作。这样就可以保证在这个合约上，不是作何一个人想做什么事就可以做什么的，从而约束投票这件事按「规定」发展。

从这个例子中，我们可以总结出如下合约的应用场景：
- 任何一个合约，需要有人主动创建；创建时可以传入一些初始信息。
- 根据合约的需求，合约可以规定哪此人可以调用哪些方法，从而对方法的调用权进行控制（也就是对信息的修改权进行控制）。
- 有些人开始的时候可能没有权限作任何事情，只有等待被授权以后才行。

建议读者看一懂 solidity 官方的合约示例：[Solidity by Example](https://solidity.readthedocs.io/en/v0.5.10/solidity-by-example.html)，会对「动态的合约」有更好的理解。



# 手动布署合约

虽然这篇文章里不会讲合约的具体语法，但其实我还是看了一下官文文档、稍微学习了一下的。我觉得语法比较好学，并且官文文档已经很详细了（虽然感觉有点乱），但官方文档里并没有说如何手动布署合约（也可能是我没找到）。所以这里我要记录一下如何手动布署合约。

首先讲为什么要手动布署。其实官方已经给出了一些简便的布署和测试合约的方法（比如下一小节里要讲的 remix），但我的目的就是为了学习和调试以太坊的源代码，所以自己手动布署一个合约测试环境还是很有必要的。

这里假设你已经搭建好了一个本地测试环境（关于本地测试环境的搭建，可以参看[这篇文章](https://yangzhe.me/2019/07/29/ethereum-construct-test-envirment/)）。因此剩下的事情，就是要编写合约、编译合约、提交合约、调用合约。我们分别来看。

### 编写合约

这个其实没什么好讲的，纯粹是为了文章的完整性。只要随便打开一个编辑器，都可以用来编写合约（也可能有比较友好的编辑器可以高亮、代码提示等，我不知道。如果有可以留言告诉我）。

后面的介绍中，我们以这个简单的合约为例进行介绍：
```
pragma solidity >=0.4.4;

contract test { 
    uint256 x;

    function multiply(uint256 a) public view returns(uint256){
        return x * a;
    }

    function multiplyState(uint256 a) public returns(uint256){
        x = a * 7;
        return x;
    }
}
```


### 编译合约

写好合约源码以后，需要使用合约编译器进行编译。合约编译器也是以太坊的项目之一，地址在[这里](https://github.com/ethereum/solidity)。你可以直接从它的[ releases 项](https://github.com/ethereum/solidity/releases)里下载合约编译器的可执行程序，也可以 clone 源代码自己编译。这是一个 C++ 写的项目，需要用到 CMake 及 C++ 相关编译工具，以及 boost 库。合约编译器项目的 readme 文件中 [Build and Install](https://github.com/ethereum/solidity#build-and-install) 节详细记录了编译的步骤，这里就不再赘述了。按照步骤编译成功后，编译目录下的 solc/solc 程序就是合约编译器。

有了编译器以后，就可以编译我们写的合约了。假设合约源代码存放在 mycontract.sol 中，那么你可以使用如下命令编译：
```
solc --bin -o ./ ./mycontract.sol
```
其中 `--bin` 参数指定输出数据的格式； `-o` 参数指定输出文件所在目录，这里指定的是当前目录。因此运行此命令以后，会在当前目录生成一个 test.bin 文件，里面存放的是合约编译后的二进制的字符串表示形式。

`solc` 程序还可以输出其它格式的数据，比如 `--asm` 可以输出 solidity 语言的汇编指令，读者可以自已尝试一下。


### 提交合约

合约编译成功以后，就可以提交给以太坊节点了。这里我们使用以太坊的 rpc 命令提交，因此首先需要将之前搭建好的测试环境运行起来，保证有一个节点启动了 rpc 功能，且有节点可以正常挖矿。

现在我们假设 rpc 节点已经在本地正常运行，且使用默认端口，因此 rpc 地址为 `localhost:8545`。

我们知道，合约在以太坊的 EVM 中运行是需要 gas 的。gas 之于合约的运行，相当于汽油之于汽车的运行：汽车每跑一步，都需要消耗汽油；合约每执行一条汇编指令，都需要消耗 gas。（其实还有一点非常相似，gas 并不是以太币的计价单位，只是合约汇编指令的计价单位，因此 gas 的价格是可变的；同样汽油的价格也是浮动的、可变的。只是这一条与我们目前讨论的没有关系，因此不重点介绍）。

由于每执行一条合约的汇编指令，都需要消耗 gas，而一条合约在执行过程中如果 gas 耗尽，那么 EVM 将立即中止合约的执行，已经消耗的 gas 也不会退回。如果发生这种情况，相当于钱已经花了、但因为花得少而没办成事，很冤枉。所以我们要尽量避免这种情况的发生。那么我们如何知道我们写的合约需要多少 gas 呢？rpc 有一个 `eth_estimateGas` 方法，可以帮我们评估我们的合约需要多少 gas。示例如下：
```
curl -X POST -H 'Content-Type: application/json'  --data '{"jsonrpc":"2.0","method":"eth_estimateGas", "params":[{"from":"0x7656bde2ff583c0b0817386e7b88eb8991f4d90c", "data":"0x608060405234801561001057600080fd5b50610134806100206000396000f3fe6080604052348015600f57600080fd5b506004361060325760003560e01c80631122db9a146037578063c6888fa1146076575b600080fd5b606060048036036020811015604b57600080fd5b810190808035906020019092919050505060b5565b6040518082815260200191505060405180910390f35b609f60048036036020811015608a57600080fd5b810190808035906020019092919050505060cb565b6040518082815260200191505060405180910390f35b6000600782026000819055506000549050919050565b60008160005402905091905056fea265627a7a72315820fb8e1458bd3bdfe4c1385508d7a7e2ee7262d6d834c779a219a63886cf2e57cf64736f6c637828302e352e31312d646576656c6f702e323031392e372e32362b636f6d6d69742e34666137383030340058"}], "id":1}' localhost:8545

# output:
{"jsonrpc":"2.0","id":1,"result":"0x21667"}
```
`data` 参数就是刚才 `solc` 使用 `--bin` 命令输出的数据，但在加了 `0x` 前缀。后面所有方法用到编译后合约的 bin 数据时，都需要加 `0x` 前缀。可以看见在我的例子中，这个合约需要的 gas 值为 `0x21667`。

**这里需要注意， `eth_estimateGas` 估算的值并不保证准确，所以保险起见，最好在这个值的基础之上再写大一些（多余的 gas 用不完会退回来）。**

大体知道了需要用到的 gas 量，我们就可以提交合约了。合约的提交，也就是创建，是通过发送一个交易实现的，即使用 rpc 的 `eth_sendTransaction` 方法。以太坊规定在发送一笔交易时，如果接收者为空，则使用 `data` 参数中的数据创建一个新的合约。创建合约的示例如下：
```
curl -X POST -H 'Content-Type: application/json'  --data '{"jsonrpc":"2.0","method":"eth_sendTransaction", "params":[{"from":"0x7656bde2ff583c0b0817386e7b88eb8991f4d90c", "gas":"0x121667", "data":"0x608060405234801561001057600080fd5b50610134806100206000396000f3fe6080604052348015600f57600080fd5b506004361060325760003560e01c80631122db9a146037578063c6888fa1146076575b600080fd5b606060048036036020811015604b57600080fd5b810190808035906020019092919050505060b5565b6040518082815260200191505060405180910390f35b609f60048036036020811015608a57600080fd5b810190808035906020019092919050505060cb565b6040518082815260200191505060405180910390f35b6000600782026000819055506000549050919050565b60008160005402905091905056fea265627a7a72315820fb8e1458bd3bdfe4c1385508d7a7e2ee7262d6d834c779a219a63886cf2e57cf64736f6c637828302e352e31312d646576656c6f702e323031392e372e32362b636f6d6d69742e34666137383030340058"}], "id":1}' localhost:8545

# output:
{"jsonrpc":"2.0","id":1,"result":"0x050414e6fd576fdbdbb67962de784e9089adfc71f69a813b605f5d18c96cc9c7"}
```
在 `eth_sendTransaction` 的方法中， `gas` 参数的值比刚才通过 `eth_estimateGas` 评估得到的值大了一些；`data` 参数与刚才一样，是合约编译后的输出值（加 0x 前缀）。这个方法返回值为新创建的交易的哈希。

至此，合约已经提交了，但我们并不知道合约是否创建成功了，以及成功后合约的地址是多少。这些信息可以通过 `eth_getTransactionReceipt` 方法来获取（但需要在提交合约后稍等片刻，因为以太坊节点需要时间将新提交的交易打包到区块中）：
```
curl -X POST -H 'Content-Type: application/json'  --data '{"jsonrpc":"2.0","method":"eth_getTransactionReceipt", "params":["0x050414e6fd576fdbdbb67962de784e9089adfc71f69a813b605f5d18c96cc9c7"], "id":1}' localhost:8545

# output:
{"jsonrpc":"2.0","id":1,"result":{"blockHash":"0x19572a7448b4b0678c7e247d66d7bb7ec54ab69eb334adbc2c09ea8df2f9090d","blockNumber":"0x1f0","contractAddress":"0x55a4ccf5907c7f3891f77137ffb346a68a129516","cumulativeGasUsed":"0x1b2cd","from":"0x7656bde2ff583c0b0817386e7b88eb8991f4d90c","gasUsed":"0x1b2cd","logs":[],"logsBloom":"0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000","status":"0x1","to":null,"transactionHash":"0x2fc90503131ac2dbb1e7423700d8988a66ad02e4dbf5106162d6318e487ea9f5","transactionIndex":"0x0"}}
```
`eth_getTransactionReceipt` 方法的唯一一个参数，就是刚才 `eth_sendTransaction` 返回的交易哈希。这个方法会返回交易成功后的所有相关信息，其中有一项 `contractAddress`，就是我们新创建的合约的地址啦。另外一个需要重点注意的是 `status` 字段的状态，只有其值为「真」时，合约才真正创建成功。因为虽然 `eth_getTransactionReceipt` 显示提交的 Transaction 被成功打包，但并不代表合约创建的成功：创建合约的代码可能在 EVM 中执行失败，此时合约创建失败，但 Transaction 仍然可以是打包成功的。


### 调用合约

合约创建成功后，就可以调用合约的 public 方法了。有两个 rpc 的方法可以调用合约：`eth_call` 和 `eth_sendTransaction`。 这两个方法的参数是一样的：`to` 参数为将要调用的合约的地址；`data` 参数存放被调用的方法和参数信息。

这两个方法的区别在于，`eth_call` 仅可以调用不会改变合约状态信息的方法（其实即使改变了也不会被保存），且可以立即得到合约的方法的返回值；而 `eth_sendTransaction` 可以调用所有方法，但无法得到返回值。所以在我们的例子中，`eth_call` 用来调用 `test.multiply` 比较合适；而 `eth_sendTransaction` 用来调用 `test.multiplyState` 比较合适。

无论用哪个方法调用，我们首先要解决的是怎么填充 `data` 参数。调用合约，实际上就是调用合约的某个 public 方法（类似于C++对象的某个 public 方法），那么 `data` 里必须有数据能标识调用的是哪个方法；然后就是这个方法的参数数据，也需要在 `data` 参数中。

对于调用哪个方法的标识，官方文档中将其称为「函数选择子」（function selector）。实际上这个值是被调用的方法的方法名和参数声明的字符串（不带空格）的 Keccak 哈希的前 4 个字节。这句话很绕，看个例子就很简单了。例如对于文章前面布署的合约：
```
pragma solidity >=0.4.4;
  
contract test { 
    uint256 x;

    function multiply(uint256 a) public view returns(uint256){
        return x * a;
    }

    function multiplyState(uint256 a) public returns(uint256){
        x = a * 7;
        return x;
    }
}
```
如果我想调用 `multiply` 或 `multiplyState` 方法，那么可以这样得到函数选择子：
```
# KeccakHash 只是一个代号。具体计算哈希的方法下面会讲
KeccakHash("multiply(uint256)")[0:4] => c6888fa1
KeccakHash("multiplyState(uint256)")[0:4] => 1122db9a
```
所以这个合约的 `multiply` 方法的函数选择子为 `c6888fa1`，而 `multiplyState` 的函数选择子为 `1122db9a`。

但是这个例子中 `KeccakHash` 这个计算哈希的函数并不存在，那么该怎么计算哈希呢？简单一点，你可以使用 console 中的 `web3.sha3` 函数。例如你以以下命令启动某个节点的 console：
```
geth --datadir my/datadir console 2>xxx.log
```
接着就得到了以太坊控制台的输入界面。然后就可以计算哈希了：
```
> web3.sha3("multiply(uint256)")
"0xc6888fa159d67f77c2f3d7a402e199802766bd7e8d4d1ecd2274fc920265d56a"
> web3.sha3("multiplyState(uint256)")
"0x1122db9a73f12708d7674a9a5d0d5dd2bdb260193a8fcc233d1817ca8708fc93"
```
使用 `2>xxx.log` 是为了不让 `geth` 程序的 log 输出影响到 console 的使用。

或者你还可以使用 rpc 的 `web3_sha3` 方法。使用这个方法时注意参数是字符串的 16 进制形式，并在前面加 0x 前缀。比如对于 `multiply(uint256)` 这个字符串来说来说它的 16 进制形式为 `0x6d756c7469706c792875696e7432353629`。所以你可以使用如下命令得到 `multiply(uint256)` 这个字符串的哈希：
```
curl -X POST -H 'Content-Type: application/json'  --data '{"jsonrpc":"2.0","method":"web3_sha3", "params":["0x6d756c7469706c792875696e7432353629"], "id":1}' localhost:8545

# output:
{"jsonrpc":"2.0","id":1,"result":"0xc6888fa159d67f77c2f3d7a402e199802766bd7e8d4d1ecd2274fc920265d56a"}
```

有了函数选择子，乘下的就是参数的编码了。合约的方法的参数有多个，且类型也很多，这里就不一一介绍每种类型的编码方式了，具体细节可以参看[这篇官方文档](https://solidity.readthedocs.io/en/latest/abi-spec.html#formal-specification-of-the-encoding)。在我们的例子中，只有一个 `uint256` 类型的参数，它将被编码为一个 32 字节的 16 进制数字，所以如果我们想通过传入一个 10 进制值为 26 的整数，那么它的编码为：
> 0x000000000000000000000000000000000000000000000000000000000000001a

现在函数选择子和参数都有了，只需要将它们拼在一起就行了，所以我们得到了以 26 为参数调用 `multiplyState` 方法的数据：
> 0x1122db9a000000000000000000000000000000000000000000000000000000000000001a

和以 2 为参数调用 `multiply` 方法的数据：
> 0xc6888fa10000000000000000000000000000000000000000000000000000000000000002

现在我们可以开始调用合约的方法了。我们首先以参数 26 调用 `multiplyState`，根据代码，这会将合约的状态变量 `x` 设置为 182。根据刚才的说明，这里需要使用 `eth_sendTransaction` 进行调用（我们忽徊了使用 `eth_estimateGas` 评估 gas 值的步骤）：
```
curl -X POST -H 'Content-Type: application/json'  --data '{"jsonrpc":"2.0","method":"eth_sendTransaction", "params":[{"from":"0x7656bde2ff583c0b0817386e7b88eb8991f4d90c", "to":"0x55a4ccf5907c7f3891f77137ffb346a68a129516", "gas":"0x153d8", "data":"0x1122db9a000000000000000000000000000000000000000000000000000000000000001a"}], "id":1}' localhost:8545

# output:
{"jsonrpc":"2.0","id":1,"result":"0x3e7a15879105924589a45c5a83b8a8650d702a4c909ff982d94ec9f8814bbe42"}
```
虽然我们知道 `multiplyState` 方法有一个返回值，但我们是查不到这个返回值的。（你可以使用 `eth_getTransactionReceipt` 和上面返回的交易哈希，查询一下交易的数据和是否成功，但里面是没有返回值的）

虽然没有返回值，但如果方法执行成功，我们确信合约的状态变量 `x` 此时已经是 182 了。此时我们再使用 `eth_call` ，以参数 2 调用 `multiply`，应该返回 364（0x16c）：
```
curl -X POST -H 'Content-Type: application/json'  --data '{"jsonrpc":"2.0","method":"eth_call", "params":[{"from":"0x7656bde2ff583c0b0817386e7b88eb8991f4d90c", "to":"0x55a4ccf5907c7f3891f77137ffb346a68a129516", "data":"0xc6888fa10000000000000000000000000000000000000000000000000000000000000002"}, "latest"], "id":1}' localhost:8545

# output:
{"jsonrpc":"2.0","id":1,"result":"0x000000000000000000000000000000000000000000000000000000000000016c"}
```

可以看到 `eth_call` 直接返回了结果，`result` 字段里正是我们期望的值 0x16c。


# 使用 remix

虽然以调用以太坊 rpc 的方式与合约交互，可以达到我们调试代码、熟悉 EVM 的目的，但如果关心的是合约本身，那么这种方法就太麻烦了。以太坊官方实现了一个非常好用的在线合约 IDE：[remix](https://remix.ethereum.org)。它提供了代码高亮和自动提示等编辑器功能，还可以一键布署合约，一键调用，或者对合约进行单步调试，非常好用，强烈推荐大家试试。


# 总结 

这篇文章主要介绍了以太坊合约的两个方面，一是我自己对智能合约的认识：这是一种动态合约，是事物发展的约定，而不仅仅是我们日常所见的那种纸质的、静态的合同；二是从熟悉原理和源代码的目的入手，介绍了一下如何使用 rpc 方法这种「笨办法」与智能合约交互，包括创建和调用等方法。

最后说一点我自己对智能合约的看法吧。不可否认，以太坊智能合约是一个很棒的想法，但我觉得它的应用非常有限。首要的一个原因是它无法使用打补丁的方式升级，一旦布署，如果要修改只能重新布署一个新的合约。其次因为无法升级，加上以太坊去中心化的特性，这些原因导致对合约的编写要求非常高：写出来的合约必须毫无破绽，否则合约被恶意利用，有可能会造成很大损失。而作为程序员，我觉得很少有人能写出完全没有 bug 的代码。当然这个想法不一定对，如果有不同想法，欢迎留言讨论。

限于水平，文章可能有错误的地方。如果发现，感谢留言指正。