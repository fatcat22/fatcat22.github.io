---
layout: post
title:  "以太坊源码解析：evm"
categories: ethereum
tags: 原创 ethereum EVM 源码解析
excerpt: 学习以太坊不能不学习智能合约；学习智能合约不能不学一下 EVM。这篇文章我们就来了解一下 evm 模块。
author: fatcat22
---

* content
{:toc}





>本篇文章分析的源码地址为：[https://github.com/ethereum/go-ethereum](https://github.com/ethereum/go-ethereum)  
>分支：[master](https://github.com/ethereum/go-ethereum/tree/master)  
>commit id: [257bfff316e4efb8952fbeb67c91f86af579cb0a](https://github.com/ethereum/go-ethereum/tree/257bfff316e4efb8952fbeb67c91f86af579cb0a)  


# 引言

以太坊的智能合约是一个非常棒的想法，所以学习以太坊一定要学一下智能合约。而在以太坊源码里，evm 模块实现了执行智能合约的虚拟机，无论是合约的创建还是调用，都是由 evm 模块完成。所以学习以太坊也一定要学习 evm 模块。这篇文章里，我们通过源代码，看一下这个模块是如何实现的。

需要说明的是，如果想要完整了解以太坊智能合约的实现，达到自己也能写一个类似功能的程度，只学习 evm 模块是不够的，还需要学习智能合约的编译器 [solidity 项目](https://github.com/ethereum/solidity)。evm 只是实现了一个虚拟机，它在以太坊区块链环境中，逐条地解释执行智能合约的指令，而这是相对比较简单的一步。更重要且更有难度的是，如何设计一个智能合约语言，以及如何编译它。但我在编译原理方面懂得也不多，时间有限就先不深究 solidity 语言的设计和编译了。

（solidity 语言的设计很简单，但仍然是图灵完备的。因此如果要学习编译原理，拿它来作为一个实践项目学习起来应该会非常棒。）


# evm 实现结构

我们先看一下 evm 模块的整体实现结构的示意图：
![](/pic/ethereum-evm/evm.png)

其实 evm 的整体设计还是挺简单的。我们先简单说明一下示意图，后面各小节会针对各功能块进行详细的说明。

首先 evm 模块的核心对象是 `EVM`，它代表了一个以太坊虚拟机，用于创建或调用某个合约。每次处理一个交易对象时，都会创建一个 `EVM` 对象。

`EVM` 对象内部主要依赖三个对象：解释器 `Interpreter`、虚拟机相关配置对象 `vm.Config`、以太坊状态数据库 `StateDB`。

`StateDB` 主要的功能就是用来提供数据的永久存储和查询。关于这个对象的详细信息，可以查看我写的[这篇文章](https://yangzhe.me/2019/06/19/ethereum-state/)。

`Interpreter` 是一个接口，在代码中由 `EVMInterpreter` 实现具体功能。这是一个解释器对象，循环解释执行给定的合约指令，直接遇到退出指令。每执行一次指令前，都会做一些检查操作，确保 gas、栈空间等充足。但各指令真正的解释执行代码却不在这个对象中，而是记录在 `vm.Config` 的 `JumpTable` 字段中。

`vm.Config` 为虚拟机和解释器提供了配置信息，其中最重要的就是 `JumpTable`。`JumpTable` 是 `vm.Config` 的一个字段，它是一个由 256 个 `operation` 对象组成的数组。解释器每拿到一个准备执行的新指令时，就会从 `JumpTable` 中获取指令相关的信息，即 `operation` 对象。这个对象中包含了解释执行此条指令的函数、计算指令的 gas 消耗的函数等。

在代码中，根据以太坊的版本不同，`JumpTable` 可能指向四个不同的对象：`constantinopleInstructionSet`、`byzantiumInstructionSet`、`homesteadInstructionSet`、`frontierInstructionSet`。这四套指令集多数指令是相同的，只是随着版本的更新，新版本比旧版本支持更多的指令集。

大体了解了 evm 模块的设计结构以后，我们在后面的小节里，从源码的角度详细看一下各个功能块的实现。


# 以太坊虚拟机：EVM

`EVM` 对象是 evm 模块对外导出的最重要的对象，它代表了一个以太坊虚拟机。利用这个对象，以太坊就可以创建和调用合约了。下面我们从虚拟机的创建、合约的创建、合约的调用三个方面来详细介绍一下。


### 创建 EVM

前面我们提到过，每处理一笔交易，就要创建一个 `EVM` 来执行交易中的数据。这是在 `ApplyTransaction` 这个函数中体现的：
```go
func ApplyTransaction(config *params.ChainConfig, bc ChainContext, author *common.Address, gp *GasPool, statedb *state.StateDB, header *types.Header, tx *types.Transaction, usedGas *uint64, cfg vm.Config) (*types.Receipt, uint64, error) {
    msg, err := tx.AsMessage(types.MakeSigner(config, header.Number))
    if err != nil {
        return nil, 0, err
    }
    // Create a new context to be used in the EVM environment
    context := NewEVMContext(msg, header, bc, author)
    // Create a new environment which holds all relevant information
    // about the transaction and calling mechanisms.
    vmenv := vm.NewEVM(context, statedb, config, cfg)
    // Apply the transaction to the current state (included in the env)
    _, gas, failed, err := ApplyMessage(vmenv, msg, gp)
    if err != nil {
        return nil, 0, err
    }

    ......
}
```
`ApplyTransaction` 函数的功能就是将交易的信息记录到以太坊状态数据库（state.StateDB）中，这其中包括转账信息和执行合约信息（如果交易中有合约的话）。这个函数一开始调用 `Transaction.AsMessage` 将一个 `Transaction` 对象转换成 `Message` 对象，我觉得这个转换主要是为了语义上的一致，因为在合约中，`msg` 全局变量记录了附带当前合约的交易的信息，可能是为了一致，这里也将「交易」转变成「消息」传给 `EVM` 对象。

然后 `ApplyTransaction` 创建了 `Context` 对象，这个对象中主要包含了一些访问当前区块链数据的方法。

接下来 `ApplyTransaction` 利用这些参数，调用 `vm.NewEVM` 函数创建了以太坊虚拟机，通过 `ApplyMessage` 执行相关功能。`ApplyMessage` 只有一行代码，它最终调用了 `StateTransition.TransitionDb` 方法。我们来看一下这个方法：
```go
func (st *StateTransition) TransitionDb() (ret []byte, usedGas uint64, failed bool, err error) {
    // 执行一些检查
    if err = st.preCheck(); err != nil {
        return
    }

    ......

    // 判断是否是创建新合约
    contractCreation := msg.To() == nil

    ......

    if contractCreation {
        // 调用 EVM.Create 创建合约
        ret, _, st.gas, vmerr = evm.Create(sender, st.data, st.gas, st.value)
    } else {
        // 不是创建合约，则调用 EVM.Call 调用合约
        st.state.SetNonce(msg.From(), st.state.GetNonce(sender.Address())+1)
        ret, st.gas, vmerr = evm.Call(sender, st.to(), st.data, st.gas, st.value)
    }

    ......

    // 归还剩余的 gas，并将已消耗的 gas 计入矿工账户中
    st.refundGas()
    st.state.AddBalance(st.evm.Coinbase, new(big.Int).Mul(new(big.Int).SetUint64(st.gasUsed()), st.gasPrice))

    return ret, st.gasUsed(), vmerr != nil, err
}
```
`StateTransition` 对象记录了在处理一笔交易过程中的状态数据，比如 gas 的花费等。在 `StateTransition.TransitionDb` 这个方法中，首先调用 `StateTransaction.preCheck` 检查交易的 Nonce 值是否正确，并从交易的发送者账户中「购买」交易中规定数量的 gas，如果「购买」失败，那么此次的交易就失败了。

接下来 `StateTransaction.TransitionDb` 判断当前的交易是否创建合约，这是通过交易的接收者是否为空来判断的。如果需要创建合约，则调用 `EVM.Create` 进行创建；如果不是，则调用 `EVM.Call` 执行合约代码。（如果是一笔简单的转账交易，接收者肯定不为空，那么就会调用 `EVM.Call` 实现转账功能）。

`EVM` 对象执行完相关功能后，`StateTransaction.TransitionDb` 调用 `StateTransaction.refundGas` 将未用完的 gas 还给交易的发送者。然后将消耗的 gas 计入矿工账户中。

上面是创建 `EVM` 并调用其相关方法的环境说明。说了这么多「前戏」，我们现在正式介绍 `EVM` 的创建：
```go
func NewEVM(ctx Context, statedb StateDB, chainConfig *params.ChainConfig, vmConfig Config) *EVM {
    evm := &EVM{
        Context:      ctx,
        StateDB:      statedb,
        vmConfig:     vmConfig,
        chainConfig:  chainConfig,
        chainRules:   chainConfig.Rules(ctx.BlockNumber),
        interpreters: make([]Interpreter, 0, 1),
    }

    if chainConfig.IsEWASM(ctx.BlockNumber) {
        panic("No supported ewasm interpreter yet.")
    }

    // vmConfig.EVMInterpreter will be used by EVM-C, it won't be checked here
    // as we always want to have the built-in EVM as the failover option.
    evm.interpreters = append(evm.interpreters, NewEVMInterpreter(evm, vmConfig))
    evm.interpreter = evm.interpreters[0]

    return evm
}
```
`NewEVM` 函数用来创建一个新的虚拟机对象，它有四个参数，含义分别如下：
- ctx: 提供访问当前区块链数据和挖矿环境的函数和数据
- statedb: 以太坊状态数据库对象
- chainConfig: 当前节点的区块链配置信息
- vmConfig: 虚拟机配置信息

注意 `vmConfig` 参数中的数据可能是不全的，比如缺少 `Config.JumpTable` 数据（`Config.JumpTable` 会在 `NewEVMInterpreter` 中填充）。

`NewEVM` 函数的实现也相关简单，除了把参数记录下来以外，主要就是调用 `NewEVMInterpreter` 创建一个解释器对象。我们顺便看一下这个函数的实现：
```go
func NewEVMInterpreter(evm *EVM, cfg Config) *EVMInterpreter {
    // We use the STOP instruction whether to see
    // the jump table was initialised. If it was not
    // we'll set the default jump table.
    if !cfg.JumpTable[STOP].valid {
        switch {
        case evm.ChainConfig().IsConstantinople(evm.BlockNumber):
            cfg.JumpTable = constantinopleInstructionSet
        case evm.ChainConfig().IsByzantium(evm.BlockNumber):
            cfg.JumpTable = byzantiumInstructionSet
        case evm.ChainConfig().IsHomestead(evm.BlockNumber):
            cfg.JumpTable = homesteadInstructionSet
        default:
            cfg.JumpTable = frontierInstructionSet
        }
    }

    return &EVMInterpreter{
        evm:      evm,
        cfg:      cfg,
        gasTable: evm.ChainConfig().GasTable(evm.BlockNumber),
    }
}
```
这个方法在创建解释器之前，首先填充了 `Config.JumpTable` 字段。可以看到这里跟据以太坊版本不同，使用了不同的对象填充这个字段。然后函数生成一个 `EVMInterpreter` 对象并返回（这里还填充了 `gasTable` 这个字段，但此处我们先暂时忽略，在「gas 的消耗」小节中再详细说明）。

总之，以太坊在每处理一笔交易时，都会调用 `NewEVM` 函数创建 `EVM` 对象，哪怕不涉及合约、只是一笔简单的转账。 `NewEVM` 的实现也很简单，只是记录相关的参数，同时创建一个解释器对象。`Config.JumpTable` 字段在开始时是无效的，在创建解释器对象时对其进行了填充。


### 创建合约

上面我们已经看到，如果交易的接收者为空，则代表此条交易的目的是要创建一条合约，因此调用 `EVM.Create` 执行相关的功能。现在我们就来看看这个方法是如何实现合约的创建的。

先来看看 `EVM.Create` 的代码：
```go
func (evm *EVM) Create(caller ContractRef, code []byte, gas uint64, value *big.Int) (ret []byte, contractAddr common.Address, leftOverGas uint64, err error) {
    contractAddr = crypto.CreateAddress(
        caller.Address(), evm.StateDB.GetNonce(caller.Address()))
    return evm.create(caller, &codeAndHash{code: code}, gas, value, contractAddr)
}
```
`EVM.Create` 的代码很简单，就是通过当前合约的创建者地址和其账户中的 Nonce 值，计算出来一个地址值，作为合约的地址。然后将这个地址和其它参数传给 `EVM.create` 方法。这里唯一需要注意的是由于用到了账户的 Nonce 值，所以同一份合约代码，每次创建合约时得到的合约地址都是不一样的（因为合约是通过发送交易创建，而每发送一次交易 Nonce 值都会改变）。

创建合约的重点在 `EVM.create` 方法中，下面是这个方法的第一部分代码：
```go
func (evm *EVM) create(caller ContractRef, codeAndHash *codeAndHash, gas uint64, value *big.Int, address common.Address) ([]byte, common.Address, uint64, error) {
    // 检查合约创建的递归调用次数
    if evm.depth > int(params.CallCreateDepth) {
        return nil, common.Address{}, gas, ErrDepth
    }
    // 检查合约创建者是否有足够的以太币
    if !evm.CanTransfer(evm.StateDB, caller.Address(), value) {
        return nil, common.Address{}, gas, ErrInsufficientBalance
    }
    // 增加合约创建者的 Nonce 值
    nonce := evm.StateDB.GetNonce(caller.Address())
    evm.StateDB.SetNonce(caller.Address(), nonce+1)

    ......
}
```
第一段代码比较简单，主要是一些检查，并修改合约创建者的账户的 Nonce 值。

第一个 if 判断中的 `EVM.depth` 记录者合约的递归调用次数。在 solidity 语言中，允许在合约中通过 `new` 关键字创建新的合约对象，但这种「在合约中创建合约」的递归调用是有限制的，这也是这个 if 判断的意义。`EVM.depth` 在解释器对象开始运行时（即 `EVMInterpreter.Run` 中）加 1，解释器运行结束后减 1（不知道为什么不把这个加减操作放在 `EVM.create` 中进行）。

我们继续看 `EVM.create` 的下一段代码：
```go
func (evm *EVM) create(caller ContractRef, codeAndHash *codeAndHash, gas uint64, value *big.Int, address common.Address) ([]byte, common.Address, uint64, error) {
    ......

    contractHash := evm.StateDB.GetCodeHash(address)
    if evm.StateDB.GetNonce(address) != 0 || 
        (contractHash != (common.Hash{}) && contractHash != emptyCodeHash) {
        return nil, common.Address{}, 0, ErrContractAddressCollision
    }
    // Create a new account on the state
    snapshot := evm.StateDB.Snapshot()
    evm.StateDB.CreateAccount(address)
    if evm.ChainConfig().IsEIP158(evm.BlockNumber) {
        evm.StateDB.SetNonce(address, 1)
    }
    evm.Transfer(evm.StateDB, caller.Address(), address, value)

    ......
}
```

这段代码在状态数据库中创建了合约账户，并将交易中约定好的以太币数值（`value`）转到这个新创建的账户里。这里在创建新账户前，会先检查这个账户是否已经存在，如果已经存在就直接返回错误了。

我们继续下面的代码：
```go
func (evm *EVM) create(caller ContractRef, codeAndHash *codeAndHash, gas uint64, value *big.Int, address common.Address) ([]byte, common.Address, uint64, error) {
    ......
    contract := NewContract(caller, AccountRef(address), value, gas)
    contract.SetCodeOptionalHash(&address, codeAndHash)

    if evm.vmConfig.NoRecursion && evm.depth > 0 {
        return nil, address, gas, nil
    }

    ......
    ret, err := run(evm, contract, nil, false)

    ......
    if err == nil && !maxCodeSizeExceeded {
        createDataGas := uint64(len(ret)) * params.CreateDataGas
        if contract.UseGas(createDataGas) {
            evm.StateDB.SetCode(address, ret)
        } else {
            err = ErrCodeStoreOutOfGas
        }
    }
    ......

    return ret, address, contract.Gas, err
```
这段代码首先创建了一个 `Contract` 对象。一个 `Contract` 对象包含和维护了合约在执行过程中的必要信息，比如合约创建者、合约自身地址、合约剩余 gas、合约代码和代码的 jumpdests 记录（关于 jumpdests 记录请参看后面的「跳转指令」小节）。

创建 `Contract`对象以后，紧跟着的是一个 if 判断。如果以太坊虚拟机被配置成不可递归创建合约，而当前创建合约的过程正是在递归过程中，则直接返回成功，但并没有返回合约代码（第一个返回参数）。（我其实没太看懂这个判断和处理，如果不能递归，为什么直接返回成功呢？）

接着代码调用 `run` 函数运行合约的代码。我们一会再详细看这个 `run` 函数，先来看看 `run` 返回后的处理。如果运行成功，且合约代码没有超过长度限制（`maxCodeSizeExceeded` 为 false，一会再说这个变量），则调用 `StateDB.SetCode` 将合约代码存储到以太坊状态数据库的合约账户中，当然存储需要消耗一定数据的 gas。

你可能会奇怪为什么存储的合约代码是合约运行后的返回码，而不是原来交易中的数据（即 `Transaction.data.Payload`，或者说 `EVM.Create` 方法的 `code` 参数）。这是因为合约源代码在编译成二进制数据时，除了合约原有的代码外，编译器还另外插入了一些代码，以便执行相关的功能。对于创建来说，编译器插入了执行合约「构造函数」（即合约对象的 `constructor` 方法）的代码。因此在将编译器编译后的二进制提交以太坊节点创建合约时，`EVM` 执行这段二进制代码，实际上主要执行了合约的 `constructor` 方法，然后将合约的其它代码返回，所以才会有这里的 `ret` 变量作为合约的真正代码存储到状态数据库中。 

对于 `maxCodeSizeExceeded` 变量，它记录的是合约的代码（即 `ret` 变量）是否超过某一长度。前面的代码段中 `run` 函数后面省略的代码主要是处理与这个变量相关的逻辑。我觉得这个并不重要，所以就没有放上来。

下面我们就来看看 `run` 函数，看它是怎么执行一个合约创建的过程的：
```go
func run(evm *EVM, contract *Contract, input []byte, readOnly bool) ([]byte, error) {
    if contract.CodeAddr != nil {
        precompiles := PrecompiledContractsHomestead
        if evm.ChainConfig().IsByzantium(evm.BlockNumber) {
            precompiles = PrecompiledContractsByzantium
        }
        if p := precompiles[*contract.CodeAddr]; p != nil {
            return RunPrecompiledContract(p, input, contract)
        }
    }

    for _, interpreter := range evm.interpreters {
        if interpreter.CanRun(contract.Code) {
            if evm.interpreter != interpreter {
                // Ensure that the interpreter pointer is set back
                // to its current value upon return.
                defer func(i Interpreter) {
                    evm.interpreter = i
                }(evm.interpreter)
                evm.interpreter = interpreter
            }
            return interpreter.Run(contract, input, readOnly)
        }
    }
    return nil, ErrNoCompatibleInterpreter
}
```

这个函数不长，所以我一下子全放上来了。函数前半部分判断合约的地址是否是一些特殊地址，如果是则执行其对应的对象的 `Run` 方法。关于这些特殊地址的详细信息，请参看后面的「预定义合约」小节，这里就不展开细说了。

`run` 函数的后半部分代码是一个 for 循环，从当前 `EVM` 对象中选择一个可以运行的解释器，运行当前的合约并返回。当前源代码中只有一个版本的解释器，就是 `EVMInterpreter`，我们现在先大体上看一下这个对象的的 `Run` 方法：
```go
func (in *EVMInterpreter) Run(contract *Contract, input []byte, readOnly bool) (ret []byte, err error) {
    ......
    in.evm.depth++
    defer func() { in.evm.depth-- }()
    ......

    var (
        pc   = uint64(0) // program counter
    )
    for atomic.LoadInt32(&in.evm.abort) == 0 {
        ......
        op = contract.GetOp(pc)
        operation := in.cfg.JumpTable[op]
        ......

        res, err := operation.execute(&pc, in, contract, mem, stack)

        ......
        switch {
        case err != nil:
            return nil, err
        case operation.reverts:
            return res, errExecutionReverted
        case operation.halts:
            return res, nil
        case !operation.jumps:
            pc++
        }
    }
    return nil, nil
}
```
我将此方法中当前我们关心的代码在这里展示出来。可见这个方法就是从给定的代码的第 0 个字节开始执行，直到退出。

我们前面说过，编译合约时编译器会插入一些代码。在刚才展示的创建合约的过程中，执行的就是这些插入的代码。所以创建合约的过程并没有见到有「创建」的 go 代码，而是和调用合约一样，只是执行编译出来的指令。

为了更清楚的了解这一过程，我们来看一个实例。我使用 solidity 编译器将下面这个简单的合约编译成指令的汇编形式。合约源代码如下：
```
contract test { 
    uint256 x;

    constructor() public {
        x = 0xfafa;
    }

    function multiply(uint256 a) public view returns(uint256){
        return x * a;
    }
    function multiplyState(uint256 a) public returns(uint256){
        x = a * 7;
        return x;
    }
}
```
编译成汇编代码后，最前面一部分的指令如下：
```
    mstore(0x40, 0x80)
    callvalue
    dup1
    iszero
    tag_1
    jumpi /* 跳转到 tag_1 */
    0x00
    dup1
    revert

tag_1:
    /* "../test.sol":63:111  constructor() public {... */
    pop
    0xfafa  /* 0xfafa 关键数字 */
    0x00
    dup2
    swap1
    sstore
    pop
    dataSize(sub_0)
    dup1
    dataOffset(sub_0)
    0x00
    codecopy
    0x00
    return
stop

sub_0: assembly {
    ......
```
我们不需要看懂每条指令，也能明白这其中的意思。指令一开始作了一些操作，然后会跳转到 `tag_1` 处继续执行。在 `tag_1` 处的代码，我们可以看到 0xfafa 这个关键数字，这是我们在 `constructor` 方法中为 `x` 设置的初始值。所以我们可以知道 `tag_1` 处的代码执行了合约的初始化工作。最后通过 `codecopy` 指令，将 `sub_0` 处至最后的数据拷贝出来并返回。所以合约真正的代码是从 `sub_0` 开始到结尾处的数据。这也证实了我们前面说的，存储到状态数据库中的合约代码，是从 `run` 函数返回后的数据。


### 调用合约

`EVM` 对象有三个方法实现合约的调用，它们分别是：
- `EVM.Call`
- `EVM.CallCode`
- `EVM.DelegateCall`
- `EVM.StaticCall`

我们先说一下这四个调用合约的方法的差异，然后再通过 `EVM.Call` 看一下调用合约的代码是如何实现的。

`EVM.Call` 实现的基本的合约调用的功能，没什么特殊的。后面的三个调用方式都是与 `EVM.Call` 比较产生的差异。所以这里只介绍后面三个调用方式的特殊性。


##### EVM.CallCode 和 EVM.DelegateCall

首先需要了解的是，`EVM.CallCode` 和 `EVM.DelegateCall` 的存在是为了实现合约的「库」的特性。我们知道编程语言都有自己的库，比如 go 的标准库，C++ 的 STL 或 boost。作为合约的编写语言，solidity 也想有自己的库。但与普通语言的实现不同，solidity 写出来的代码要想作为库被调用，必须和普通合约一样，布署到区块链上取得一个固定的地址，其它合约才能调用这个「库合约」提供的方法。但合约又涉及到一些特有的属性，比如合约的调用者、自身地址、自身所拥有的以太币的数量等。如果我们直接去调用「库合约」的代码，用到这些属性时必然是「库合约」自己的属性，但这可能不是我们想要的。

例如，设想一个「库合约」的方法实现了一个这样的操作：从自已账户中给指定账户转一笔钱。如果这里的「自己账户」指的是「库合约」的账户，那肯定是不现实的（因为没有人会出钱布署一个有很多以太币的合约，并且让别人把这些币转走）。

此时 `EVM.DelegateCall` 就派上用场了。这个调用方式将「库合约」的调用者，设置成自己的调用者；将「库合约」的地址，设置成自己的地址（但代码还是「库合约」的代码）。如此一来，「库合约」的属性，就完全和自己的属性一样了，「库合约」的代码就像是自己的写的代码一样。

例如 A 账户调用了 B 合约，而在 B 合约中通过 `DelegateCall` 调用了 C 合约，那么 C 合约的调用者将被修改成 A ， C 合约的地址将被修改成 B 合约的地址。所以在刚才用来转账的「库合约」的例子中，「自己账户」指的不再是「库合约」的账户了，而是调用「库合约」的账户，转账也就可以按我们想要的方式进行了。

`EVM.CallCode` 与 `EVM.DelegateCall` 类似，不同的是 `EVM.CallCode` 不改变「库合约」的调用者，只是改变「库合约」的合约地址。也就是说，如果 A 通过 `CallCode` 的方式调用 B，那么 B 的调用者是 A，而 B 的账户地址也被改成了 A。

总结一下就是，`EVM.CallCode` 和 `EVM.DelegateCall` 修改了被调用合约的上下文环境，可以让被调用的合约代码就像自己写的代码一样，从而达到「库合约」的目的。具体来说，`EVM.DelegateCall` 会修改被调用合约的调用者和合约本身的地址，而 `EVM.CallCode` 只会修改被调用合约的本身的地址。

了解了这两个方法的目的和功能，我们来看看代码中它们是如何实现各自的功能的。对于 `EvM.CallCode` 来说，它通过下面展示的几行代码来修改被调用合约的地址：
```go
func (evm *EVM) CallCode(caller ContractRef, addr common.Address, input []byte, gas uint64, value *big.Int) (ret []byte, leftOverGas uint64, err error) {
    ......

    var (
        to       = AccountRef(caller.Address())
    )
    contract := NewContract(caller, to, value, gas)
    contract.SetCallCode(&addr, evm.StateDB.GetCodeHash(addr), evm.StateDB.GetCode(addr))

    ......
}
```
可以看到，在利用现有数据生成一个合约对象时，将合约对象的地址 `to` 变量设置成调用者，也就是 `caller` 的地址。（但在调用 `Contract.SetCallCode` 时， code 的地址还是 `addr`）

对于 `EVM.DelegateCall` 来说，它是通过下面几行代码修改被调用合约的调用者和自身地址的：
```go
func (evm *EVM) DelegateCall(caller ContractRef, addr common.Address, input []byte, gas uint64) (ret []byte, leftOverGas uint64, err error) {
    ......

    var (
        to       = AccountRef(caller.Address())
    )
    contract := NewContract(caller, to, nil, gas).AsDelegate()

    ......
}

func (c *Contract) AsDelegate() *Contract {
    parent := c.caller.(*Contract)
    c.CallerAddress = parent.CallerAddress
    c.value = parent.value

    return c
}
```
这里首先也是通过将 `to` 变量设置成调用者，也就是 `caller` 的地址，达到修改被调用合约的自身地址的目的。被调用合约的调用者是通过 `Contract.AsDelegate` 修改的。这个方法里，将合约的调用者地址 `CallerAddress` 设置成目前调用者的 `CallerAddress`，也即当前调用者的调用者的地址（有些绕，仔细看一下就能明白）。


##### EVM.StaticCall

`EVM.StaticCall` 与 `EVM.Call` 类似，唯一不同的是 `EVM.StaticCall` 不允许执行会修改永久存储的数据的指令。如果执行过程中遇到这样的指令，就会失败。比如对于以下合约：
```
contract test { 
    uint256 x;

    function multiplyState(uint256 a) public returns(uint256){
        x = a * 7;
        return x;
    }
}
```
就不能使用 `EVM.StaticCall` 调用 `multiplyState` 方法，因为它会修改合约的状态 `x` 变量，而 `x` 变量是需要在以太坊状态数据库永久存储的。

`EVM.StaticCall` 是如何实现拒绝执行会修改永久存储数据的指令的呢？在解释器的 `EVMInterpreter.Run` 方法以及 `EVMInterpreter.enforceRestrictions` 方法中，有几行这样的代码：
```go
func (in *EVMInterpreter) Run(contract *Contract, input []byte, readOnly bool) (ret []byte, err error) {
    ......
    if readOnly && !in.readOnly {
        in.readOnly = true
        defer func() { in.readOnly = false }()
    }
    ......

    for atomic.LoadInt32(&in.evm.abort) == 0 {
        ......
        operation := in.cfg.JumpTable[op]
        ......
        if err := in.enforceRestrictions(op, operation, stack); err != nil {
            return nil, err
        }
        ......
    }
}

func (in *EVMInterpreter) enforceRestrictions(op OpCode, operation operation, stack *Stack) error {
    if in.evm.chainRules.IsByzantium {
        if in.readOnly {
            if operation.writes || (op == CALL && stack.Back(2).BitLen() > 0) {
                return errWriteProtection
            }
        }
    }
    return nil
}
```
可以看到， `EVMInterpreter.Run` 有一个 `readOnly` 参数，如果为 true，就会设置 `EVMInterpreter.readOnly` 为 true；而在循环解释执行指令时，会在 `EVMInterpreter.enforceRestrictions` 方法中对其进行检查，如果是只读，但将要执行的指令有写操作，或者使用 CALL 指令向其它账户转账时，都会直接返回错误。

从上面的代码可以看出，`EVM.StaticCall` 的实现需要存储指令信息的 `operation` 对象配合，在其中记录指令是否有写操作。而 `readOnly` 参数正是来自于 `EVM.StaticCall`：
```go
func (evm *EVM) StaticCall(caller ContractRef, addr common.Address, input []byte, gas uint64) (ret []byte, leftOverGas uint64, err error) {
    ......
    // run 函数的最后一个参数 readOnly 为 true
    ret, err = run(evm, contract, input, true)
    ......
}

func run(evm *EVM, contract *Contract, input []byte, readOnly bool) ([]byte, error) {
    ......
    for _, interpreter := range evm.interpreters {
        if interpreter.CanRun(contract.Code) {
            ......
            // 调用 EVMInterrepter.Run，最后一个参数 readOnly 正是 EVM.StaticCall 传入的 true
            return interpreter.Run(contract, input, readOnly)
        }
    }
    ...... 
}
```


##### EVM.Call 的实现

现在我们来看一下最基本的 `EVM.Call` 的实现，它是如何调用合约的。代码较长，我们仍然分段来看：
```go
func (evm *EVM) Call(caller ContractRef, addr common.Address, input []byte, gas uint64, value *big.Int) (ret []byte, leftOverGas uint64, err error) {
    if evm.vmConfig.NoRecursion && evm.depth > 0 {
        return nil, gas, nil
    }

    // Fail if we're trying to execute above the call depth limit
    if evm.depth > int(params.CallCreateDepth) {
        return nil, gas, ErrDepth
    }
    // Fail if we're trying to transfer more than the available balance
    if !evm.Context.CanTransfer(evm.StateDB, caller.Address(), value) {
        return nil, gas, ErrInsufficientBalance
    }

    ......
}
```
方法开始的一部分代码比较简单，与 `EVM.create` 类似，也是判断递归层次和合约调用者是否有足够的交易中约定的以太币。需要注意的是 `input` 参数，这是调用合约的 public 方法的参数。

我们继续看下一部分代码：
```go
func (evm *EVM) Call(caller ContractRef, addr common.Address, input []byte, gas uint64, value *big.Int) (ret []byte, leftOverGas uint64, err error) {
    ......

    var (
        to       = AccountRef(addr)
        snapshot = evm.StateDB.Snapshot()
    )
    if !evm.StateDB.Exist(addr) {
        precompiles := PrecompiledContractsHomestead
        if evm.ChainConfig().IsByzantium(evm.BlockNumber) {
            precompiles = PrecompiledContractsByzantium
        }
        if precompiles[addr] == nil && 
            evm.ChainConfig().IsEIP158(evm.BlockNumber) && value.Sign() == 0 {
            ......
            return nil, gas, nil
        }
        evm.StateDB.CreateAccount(addr)
    }
    evm.Transfer(evm.StateDB, caller.Address(), to.Address(), value)
    // Initialise a new contract and set the code that is to be used by the EVM.
    // The contract is a scoped environment for this execution context only.
    contract := NewContract(caller, to, value, gas)
    contract.SetCallCode(&addr, evm.StateDB.GetCodeHash(addr), evm.StateDB.GetCode(addr))

    ......
}
```
这部分的代码做了两件事件。一是判断合约地址是否存在；二是使用当前的信息创建一个合约对象，其中 `StateDB.GetCode` 从状态数据库中获取合约的代码，填充到合约对象中。

一般情况下，被调用的合约地址应该存在于以太坊状态数据库中，也就是说合约已经创建成功了。否则就返回失败。但有一种例外，就是被调用的合约地址是预先定义的情况，此时即使地址不在状态数据库中，也要立即创建一个。（关于预定义的合约地址的详细信息，请参看后面的「预定义合约」小节）。

我们继续看后面的代码：
```go
func (evm *EVM) Call(caller ContractRef, addr common.Address, input []byte, gas uint64, value *big.Int) (ret []byte, leftOverGas uint64, err error) {
    ......

    ret, err = run(evm, contract, input, false)

    // When an error was returned by the EVM or when setting the creation code
    // above we revert to the snapshot and consume any gas remaining. Additionally
    // when we're in homestead this also counts for code storage gas errors.
    if err != nil {
        evm.StateDB.RevertToSnapshot(snapshot)
        if err != errExecutionReverted {
            contract.UseGas(contract.Gas)
        }
    }
    return ret, contract.Gas, err
}
```
最后这部分代码主要是对 `run` 函数的调用，然后处理其返回值并返回，方法结束。这里的 `run` 函数与创建合约时调用的是同一个函数，因此不再过多介绍这个函数，想要详细了解可以回看前面「创建合约」小节中对于 `run` 函数的介绍。

到这里你可能会有疑问：没看到哪里调用合约代码呀？只不过是使用 `StateDB.GetCode` 获取合约代码，然后使用 `input` 中的数据作为参数，调用解释器运行合约代码而已，哪里有调用合约 public 方法的影子？

这里确实与创建合约时类似，没有「调用」的影子。还记得前面我们介绍合约创建时，提到过合约编译器在编译时，会插入一些代码吗？在介绍合约创建时，我们只介绍了编译器插入的创建合约的代码，解释器执行这些代码，就可以将合约的真正代码返回。类似的，编译器还会插入一些调用合约的代码，只要使用正确的参数执行这些代码，就可以「调用」到我们想调用的合约的 public 方法。

想要了解这整个机制，我们需要先介绍一下「函数选择子」这个概念。在 solidity 的[这篇官方文档](https://solidity.readthedocs.io/en/v0.5.10/abi-spec.html?highlight=selector#function-selector)中，介绍了什么是「函数选择子」。简单来说，就是合约的 public 方法的声明字符串的 Keccak-256 哈希的前 4 个字节。（关于「函数选择子」的详细说明和计算方法，请参看[这篇文章](https://yangzhe.me/2019/08/01/ethereum-cognition-and-deployment/)）。

「函数选择子」告诉了以太坊虚拟机我们想要调用合约的哪个方法，它和参数数据一起，被编码到了交易的 data 数据中。但我们刚才通过对合约调用的分析，并没有发现有涉及到解析「函函数选择子」的地方呀？这是因为「函数选择子」和参数的解析功能并不是由以太坊虚拟机的 go 代码码完成的，而是由合约编译器在编译时插入的代码完成了。

我们还以「合约创建」小节中合约为例。我们在那里提到过，实际的合约代码是从 `sub_0` 这个标签处开始的，而当时这个标签后的指令我们并没有列出来。现在我们就把这些指令列出来。

合约的源代码如下：
```
    contract test { 
    uint256 x;

    constructor() public {
        x = 0xfafa;
    }

    function multiply(uint256 a) public view returns(uint256){
        return x * a;
    }
    function multiplyState(uint256 a) public returns(uint256){
        x = a * 7;
        return x;
    }
}
```
编译成汇编代码后，`sub_0` 标签后面的部分代码如下：
```
sub_0: assembly {
    mstore(0x40, 0x80)
    callvalue
    dup1
    iszero
    tag_1
    jumpi
    0x00
    dup1
    revert
tag_1:
    pop
    jumpi(tag_2, lt(calldatasize, 0x04))
    shr(0xe0, calldataload(0x00))
    dup1
    0x1122db9a  // 方法 multiplyState 的函数选择子的值
    eq
    tag_3
    jumpi
    dup1
    0xc6888fa1  // 方法 multiply 的函数选择子的值
    eq
    tag_4
    jumpi
tag_2:
    ......
tag_3:
    ......
tag_4:
    ......

    ......
```
我们不需要读懂所有指令，只需注意 `0x1122db9a` 和 `0xc6888fa1` 前后的一些指令就可以了。这两个值，正好分别是 `test.multiplyState` 和 `test.multiply` 的函数选择子的值，其实不用看其它指令，我们也能猜到这里是在比较参数中的函数选择子的值，如果找到了（`eq` 指令，表示相等），就跳到相应的代码（`tag_3` 或 `tag_4`）去执行。这样通过参数中的函数选择子，就达到了调用合约指定 public 方法的目的。（在这个例子中，`tag_3` 对应 `test.multiplyState`，`tag_4` 对应 `test.multiply`）

所以总得来看，调用合约与创建合约的 go 源码没有太大的差别，基本都是创建解释器对象，然后解释执行合约指令。通过执行不同的合约指令，达到创建或调用的目的。


# 解释器对象：EVMInterpreter

解释器对象 `EVMInterpreter` 用来解释执行指定的合约指令。不过需要说明一点的是，实际的指令解释执行并不真正由解释器对象完成的，而是由 `vm.Config.JumpTable` 中的 `operation` 对象完成的，解释器对象只是负责逐条解析指令码，然后获取相应的 `operation` 对象，并在调用真正解释指令的 `operation.execute` 函数之前检查堆栈等对象。也可以说，解释器对象只是负责解释的调度工作。

`EVMInterpreter` 关键方法是 `Run` 方法，我们在这一小节里主要介绍一下这个方法的实现。这个方法的代码比较长，我们分开来一部分一部分的来看。下面是第一部分的代码：
```go
func (in *EVMInterpreter) Run(contract *Contract, input []byte, readOnly bool) (ret []byte, err error) {
    if in.intPool == nil {
        in.intPool = poolOfIntPools.get()
        defer func() {
            poolOfIntPools.put(in.intPool)
            in.intPool = nil
        }()
    }

    // Increment the call depth which is restricted to 1024
    in.evm.depth++
    defer func() { in.evm.depth-- }()

    // Make sure the readOnly is only set if we aren't in readOnly yet.
    // This makes also sure that the readOnly flag isn't removed for child calls.
    if readOnly && !in.readOnly {
        in.readOnly = true
        defer func() { in.readOnly = false }()
    }

    // Reset the previous call's return data. It's unimportant to preserve the old buffer
    // as every returning call will return new data anyway.
    in.returnData = nil

    // Don't bother with the execution if there's no code.
    if len(contract.Code) == 0 {
        return nil, nil
    }

    ......
}
```
这一部分的代码比较简单，只是初始化 `EVMInterpreter` 对象的一些字段。`intPool` 代表的是 `big.Int` 对象的一个池子，这样可以节省频繁创建和销毁 `big.Int` 对象的开销。这里还增加了 `EVM.depth` 的值，这个值在「创建合约」小节提到过。 在一个合约中可以使用 `new` 创建另外一个合约，这种情况属于合约的递归创建，`EVM.depth` 记录了递归的层数。

我们继续看下面的代码：
```go
func (in *EVMInterpreter) Run(contract *Contract, input []byte, readOnly bool) (ret []byte, err error) {
    ......

    var (
        ......
        mem   = NewMemory() // bound memory
        stack = newstack()  // local stack
        // For optimisation reason we're using uint64 as the program counter.
        // It's theoretically possible to go above 2^64. The YP defines the PC
        // to be uint256. Practically much less so feasible.
        pc   = uint64(0) // program counter
        ......
    )
    ......
}
```
这一段是一些变量的声明。之所以把这一段单独拿出来，是想要强调 `mem` 和 `stack` 变量。在合约的执行过程中，有三种存储数据的方式，其中两种就是内存块和栈（详情请参看「存储」小节)，它们分别由 `mem` 和 `stack` 变量代表。`mem` 是一个 `*Memory` 类型的对象，它代表了一块平坦的内存空间；`stack` 是 `*Stack` 类型，它实现了一个有 push 和 pop 等典型栈操作的栈对象。

我们继续看后面的代码。后面的代码都在一个 for 循环里，这个 for 循环不断的执行合约的执行，直到执行被中断，或遇到错误，遇到停止执行的指令。这个 for 循环比较大，我们依然分开来看：
```go
func (in *EVMInterpreter) Run(contract *Contract, input []byte, readOnly bool) (ret []byte, err error) {
    ......

    for atomic.LoadInt32(&in.evm.abort) == 0 {
        ......
        op = contract.GetOp(pc)
        operation := in.cfg.JumpTable[op]

        // 检查指令的有效性
        if !operation.valid {
            return nil, fmt.Errorf("invalid opcode 0x%x", int(op))
        }
        // 检查栈空间是否充足
        if err := operation.validateStack(stack); err != nil {
            return nil, err
        }
        // 检查是否有写指令限制
        if err := in.enforceRestrictions(op, operation, stack); err != nil {
            return nil, err
        }

        ......
    }
}
```
for 循环一开始，就从合约代码中提取当前将要执行的指令的值（`pc` 跟踪记录了当前将要执行的指令的位置），然后从 `vm.Config.JumpTable` 中取出记录指令详细信息的 `operation` 对象。这个对象记录了指令的很多详细信息，比如解释指令的函数、指令消耗的 gas 的计算函数、指令是否是写操作等（详细请参看下面的「跳转表：vm.Config.JumpTable」小节）。紧接着的还有检查栈空间和是否有写指令限制的代码。

 我们接着看 for 循环的中间部分的代码：
```go
func (in *EVMInterpreter) Run(contract *Contract, input []byte, readOnly bool) (ret []byte, err error) {
    ......

    for atomic.LoadInt32(&in.evm.abort) == 0 {
        ......

        // 如果将要执行的指令需要用到内存存储空间，
        // 则计算所需要的空间大小
        var memorySize uint64
        if operation.memorySize != nil {
            memSize, overflow := bigUint64(operation.memorySize(stack))
            if overflow {
                return nil, errGasUintOverflow
            }
            if memorySize, overflow = math.SafeMul(toWordSize(memSize), 32); overflow {
                return nil, errGasUintOverflow
            }
        }

        // 计算将要执行的指令所需要的 gas 值并扣除。如果 gas 不足，就直接失败返回了
        cost, err = operation.gasCost(in.gasTable, in.evm, contract, stack, mem, memorySize)
        if err != nil || !contract.UseGas(cost) {
            return nil, ErrOutOfGas
        }
        // 保证足够的内存存储空间
        if memorySize > 0 {
            mem.Resize(memorySize)
        }

        ......
    }
}
```
在中间部分的代码中，主要做了两件事，一是计算将要执行的指令使用的内存存储空间，并保证 `mem` 对象中有足够的空间；二是计算将要消耗的 gas，并立即消耗这些 gas，如果不够用，则返回失败。

我们接下来看最后一部分代码：
```go
func (in *EVMInterpreter) Run(contract *Contract, input []byte, readOnly bool) (ret []byte, err error) {
    ......

    for atomic.LoadInt32(&in.evm.abort) == 0 {
        ......

        // 调用指令的解释执行函数
        res, err := operation.execute(&pc, in, contract, mem, stack)

        ......

        if operation.returns {
            in.returnData = res
        }

        // 退出，或向前移动指令指针
        switch {
        case err != nil:
            return nil, err
        case operation.reverts:
            return res, errExecutionReverted
        case operation.halts:
            return res, nil
        case !operation.jumps:
            pc++
        }
    }
    return nil, nil
}
```
这段代码调用 `operation.execute` 函数，用来真正的解释指令做的事情。在其返回后，通过 switch 判断错误值，或向前移动 `pc`。

总得来说，解释器对象 `EVMInterpreter` 并没有什么复杂的东西，无非解析合约代码获取指令码，并从 `vm.Config.JumpTable` 中取得指令信息，然后检查栈、内存存储、gas 等内容，最后调用解释指令的函数执行指令的功能，仅此而已。


### 预定义合约

解释器对象的 `Run` 方法是在 `run` 函数中被调用的。在被调用之前，`run` 还处理了一种情况，我们称之为「预定义合约」。如下代码所示：
```go
func run(evm *EVM, contract *Contract, input []byte, readOnly bool) ([]byte, error) {
    if contract.CodeAddr != nil {
        precompiles := PrecompiledContractsHomestead
        if evm.ChainConfig().IsByzantium(evm.BlockNumber) {
            precompiles = PrecompiledContractsByzantium
        }
        if p := precompiles[*contract.CodeAddr]; p != nil {
            return RunPrecompiledContract(p, input, contract)
        }
    }

    ......
}
```
这段代码查看合约的 `CodeAddr` 是否是预定义合约的地址，如果是就调用 `RunPrecompiledContract` 运行这些预定义合约，而不再启动解释器。（注意这里用的是 `CodeAddr`，因为 DelegateCall 等功能可能会改变合约地址，但 `CodeAddr` 是不会被改变的）

根据以太坊版本的不同，预定义合约的集合也不同，`PrecompiledContractsByzantium` 这个变量里的数据比较新，所以我们就选这个变量看一下：
```go
var PrecompiledContractsByzantium = map[common.Address]PrecompiledContract{
    common.BytesToAddress([]byte{1}): &ecrecover{},
    common.BytesToAddress([]byte{2}): &sha256hash{},
    common.BytesToAddress([]byte{3}): &ripemd160hash{},
    common.BytesToAddress([]byte{4}): &dataCopy{},
    common.BytesToAddress([]byte{5}): &bigModExp{},
    common.BytesToAddress([]byte{6}): &bn256Add{},
    common.BytesToAddress([]byte{7}): &bn256ScalarMul{},
    common.BytesToAddress([]byte{8}): &bn256Pairing{},
}
```
可以看到这些所谓的「预定义合约」的地址也是预先定义好的，它们从 1 至 8，类似于 `0x0000000000000000000000000000000000000008` 这样。与地址对应的对象也都有一个 `Run` 方法，用来执行各自合约的内容，但它们直接是用 go 代码实现的，而不是解释执行某一份合约。从对象的名字就可以看出来，这些对象都有各自特定的功能。它们是用来实现一些 solidity 的内置函数的。比如对于 solidity 的 `ecrecover` 这个内置函数来说，它通过编译器编译后，最终是通过使用 `STATICCALL` 这个指令实现的。`STATICCALL` 的调用方式如下：
> staticcall(gas, toAddr, inputOffset, inputSize, retOffset, retSize)

实现 `ecrecover` 时，其第二个参数 `toAddr` 就是 `0x0000000000000000000000000000000000000001`。最终执行 `ecrecover.Run` 方法实现此功能。

另外在使用 `EVM.Call` 调用合约时，如果被调用的合约地址是预定义合约的地址，且它不存在于状态数据库中，就使用预定义地址直接创建一个账户：
```go
func (evm *EVM) Call(caller ContractRef, addr common.Address, input []byte, gas uint64, value *big.Int) (ret []byte, leftOverGas uint64, err error) {
    ......
    if !evm.StateDB.Exist(addr) {
        precompiles := PrecompiledContractsHomestead
        if evm.ChainConfig().IsByzantium(evm.BlockNumber) {
            precompiles = PrecompiledContractsByzantium
        }
        if precompiles[addr] == nil && evm.ChainConfig().IsEIP158(evm.BlockNumber) && value.Sign() == 0 {
            ......
            return nil, gas, nil
        }
        evm.StateDB.CreateAccount(addr)
    }
}
```


### gas 的消耗

我们知道，合约中指令的执行、数据的存储都是需要消耗 gas 的。这一小节中我们集中看一下如何确定一个特定的动作需要消耗的 gas 值。

gas 值只是一个数值，它有单价，标记在 `Transaction.data.Price` 字段中。在合约执行开始之前，以太坊会根据交易中的 gas 值和单价，从交易发起人账户中扣除相应的以太币（所谓的「购买」gas）；合约执行完成后，再将未使用的 gas 退还能交易发起人。如下代码所示：
```go
func (st *StateTransition) TransitionDb() (ret []byte, usedGas uint64, failed bool, err error) {
    // 在 st.preCheck 中「购买」 gas
    if err = st.preCheck(); err != nil {
        return
    }

    // 执行合约
    ......

    // 退还没用完的 gas，并把已使用的 gas 计入矿工账户中
    st.refundGas()
    st.state.AddBalance(st.evm.Coinbase, new(big.Int).Mul(new(big.Int).SetUint64(st.gasUsed()), st.gasPrice))

    ......
}
```

这里也可以看出，在合约执行过程中，gas 就是一个整数值，不涉及到账户上以太币的变化。初始化给合约的是可以使用的 gas 的最大值，随着合约的执行， gas 逐渐被消耗，直到合约停止执行或 gas 被耗尽。

我们知道，智能合约中每执行一条指令都需要消耗 gas。实际上在指令在被解释执行之前，就已经计算好需要消耗的 gas ，并首先消耗掉这些 gas 后，才开始解释执行指令。如下代码所示：
```go
func (in *EVMInterpreter) Run(contract *Contract, input []byte, readOnly bool) (ret []byte, err error) {
    ......
    for atomic.LoadInt32(&in.evm.abort) == 0 {
        ......
        operation := in.cfg.JumpTable[op] 
        ......

        cost, err = operation.gasCost(in.gasTable, in.evm, contract, stack, mem, memorySize)
        if err != nil || !contract.UseGas(cost) {
            return nil, ErrOutOfGas
        }

        ......
        res, err := operation.execute(&pc, in, contract, mem, stack)
        ......
    }
}
```
可以看到，在每次执行 `operation.execute` 解释指令时，都会先调用 `operation.gasCost` 计算需要的 gas 值，并调用 `Contract.useGas` 消耗掉这些 gas。

你可能会注意到，上面的代码中调用 `operation.gasCost` 时第一个参数用到了 `EVMInterpreter.gasTable`。这个字段中的值定义了一些特殊指令在计算所需 gas 时用到的数值，它在解释器创建时被初始化：
```go
func NewEVMInterpreter(evm *EVM, cfg Config) *EVMInterpreter {
    ......
    return &EVMInterpreter{
        ......
        gasTable: evm.ChainConfig().GasTable(evm.BlockNumber),
    }
}
```
与 `vm.Config.JumpTable` 类似，根据以太坊版本的不同，这里初始化的对象也不同。我们挑一个比较高的版本，看一下 `EVMInterpreter.gasTable` 最终会被一些什么值填充：
```go
GasTableConstantinople = GasTable{
    ExtcodeSize: 700,
    ExtcodeCopy: 700,
    ExtcodeHash: 400,
    Balance:     400,
    SLoad:       200,
    Calls:       700,
    Suicide:     5000,
    ExpByte:     50,

    CreateBySuicide: 25000,
}
```

其实 `gasCost` 是一个函数类型的字段，根据指令的不同，这个字段指向的函数也不同，因而计算方式也不同。这些信息都是在 JumpTable 中配置好的（参见「跳转表：vm.Config.JumpTable」小节）。比如对于 ADD 指令，它的定义如下（下面的代码是函数 `newFrontierInstructionSet` 的片段）：
```go
    ADD: {
        execute:       opAdd,
        gasCost:       constGasFunc(GasFastestStep),
        validateStack: makeStackFunc(2, 1),
        valid:         true,
    },
```
而 `constGasFunc` 和 `GasFastestStep` 的定义如下：
```go
func constGasFunc(gas uint64) gasFunc {
    return 
        func(gt params.GasTable, 
                evm *EVM, 
                contract *Contract, 
                stack *Stack, 
                mem *Memory, 
                memorySize uint64) (uint64, error) 
        {
            return gas, nil
        }
}

const (
    GasFastestStep uint64 = 3
)
```
可见对于 ADD 指令来说，它的 gas 消耗是固定的，就是 3。但对于某些指令，gas 的计算就要复杂一些了，这里就不一一列举了，感兴趣的可以直接去源码中查看。

事实上，不仅指令的解释执行本身需要消耗 gas ，当使用内存存储（`Memory` 对象代表的存储空间，详见下面「Memory」小节）和 StateDB 永久存储时，都需要消耗 gas 。因此在所有 `operation.gasCost` 所指向的函数中，对 gas 的计算都需要考虑三方面的内容：解释指令本身需要的 gas 、使用内存存储需要的 gas 、使用 StateDB 存储需要的 gas 。对于大多数指令，后两项是用不到的，但对于某些指令（比如 CODECOPY 或 SSTORE），它们的 `gasCost` 函数会考虑到内存和 StateDB 使用的情况。

这里需要提一下内存存储的 gas 消耗。计算内存存储 gas 值的函数为 `memoryGasCost`。这个函数会根据所要求的空间大小，计算将要消耗的 gas 值。但只有新需求的空间大小超过当前空间大小时，超过的部分才需要消耗 gas 。

最后需要说明一点的是，除上上面提到的每次执行指令前消耗 gas ，还有两种情况情况会消耗 gas 。一种情况是在执行合约之前；另一种是在创建合约之后。

在执行合约之前消耗的 gas 被称之为 「intrinsic gas」，如下代码所示：
```go
func (st *StateTransition) TransitionDb() (ret []byte, usedGas uint64, failed bool, err error) {
    ......

    // Pay intrinsic gas
    gas, err := IntrinsicGas(st.data, contractCreation, homestead)
    if err != nil {
        return nil, 0, false, err
    }
    if err = st.useGas(gas); err != nil {
        return nil, 0, false, err
    }

    ......
    // 执行合约
    if contractCreation {
        ret, _, st.gas, vmerr = evm.Create(sender, st.data, st.gas, st.value)
    } else {
        // Increment the nonce for the next transaction
        st.state.SetNonce(msg.From(), st.state.GetNonce(sender.Address())+1)
        ret, st.gas, vmerr = evm.Call(sender, st.to(), st.data, st.gas, st.value)
    }

    ......
}
```
从代码中可以看出，在合约还没有执行之前，就要先 `useGas`。此时 gas 值是由 `IntrinsicGas` 计算出来的。这个函数根据合约代码中非 0 字节的数量来计算消耗的 gas 值。（感觉这里好霸道，啥也不干先交钱）

另一种情况在创建合约之后。由于合约创建成功后要把合约代码存储到状态数据库 StateDB 中，所以当然也需要消耗 gas。如下代码所示：
```go
func (evm *EVM) create(caller ContractRef, codeAndHash *codeAndHash, gas uint64, value *big.Int, address common.Address) ([]byte, common.Address, uint64, error) {
    ......

    // 创建合约。合约代码在返回值 ret 中
    ret, err := run(evm, contract, nil, false)

    maxCodeSizeExceeded := evm.ChainConfig().IsEIP158(evm.BlockNumber) && 
        len(ret) > params.MaxCodeSize
    if err == nil && !maxCodeSizeExceeded {
        // 合约创建成功，将合约保存到 StateDB 之前，先 useGas
        createDataGas := uint64(len(ret)) * params.CreateDataGas
        if contract.UseGas(createDataGas) {
            evm.StateDB.SetCode(address, ret)
        } else {
            err = ErrCodeStoreOutOfGas
        }
    }
}
```
可以看到在合约创建成功后，要调用 `StateDB.SetCode` 将合约代码保存到状态数据库中。但在保存之前，要先消耗一定数据的 gas 。如果此时 gas 不够用，合约没有被保存，也就创建失败啦。


# 跳转表：vm.Config.JumpTable

我们前面已经提到过，解释器不是真正解释指令。指令的真正解释函数，记录在 `vm.Config.JumpTable` 字段中。在 创建解释器的 `NewEVMInterpreter` 函数中，根据以太坊版本的不同，会使用不同的对象填充这个字段：
```go
func NewEVMInterpreter(evm *EVM, cfg Config) *EVMInterpreter {
    // We use the STOP instruction whether to see
    // the jump table was initialised. If it was not
    // we'll set the default jump table.
    if !cfg.JumpTable[STOP].valid {
        switch {
        case evm.ChainConfig().IsConstantinople(evm.BlockNumber):
            cfg.JumpTable = constantinopleInstructionSet
        case evm.ChainConfig().IsByzantium(evm.BlockNumber):
            cfg.JumpTable = byzantiumInstructionSet
        case evm.ChainConfig().IsHomestead(evm.BlockNumber):
            cfg.JumpTable = homesteadInstructionSet
        default:
            cfg.JumpTable = frontierInstructionSet
        }
    }

    return &EVMInterpreter{
        evm:      evm,
        cfg:      cfg,
        gasTable: evm.ChainConfig().GasTable(evm.BlockNumber),
    }
}
```
这里面共有四个集合，稍微看一下就能明白，`frontierInstructionSet` 这个对象包含了最基本的指令信息，其它三个是对这个集合的扩充，最全的一个是 `constantinopleInstructionSet`（这跟以太坊的升级历史是一样的）。

这些对象的类型都是 `[256]operation`，使用时以指令的 opcode 值为索引，从中取出存放指令详细信息的 `operation` 对象。所有 opcode 定义在 opcodes.go 文件中，这是就不一一列出了。

`operation` 对象详细记录了指令的所有信息，它的定义如下：
```go
type operation struct {
    execute executionFunc              // 指令的解释执行函数
    gasCost gasFunc                    // 计算指令将要消耗的 gas 值的函数
    validateStack stackValidationFunc  // 检查栈空间是否足够指令使用的函数
    memorySize memorySizeFunc          // 计算指令将要消耗的内存存储空间大小的函数

    halts   bool // 指令执行完成后是否停止解释器的执行
    jumps   bool // 是否是跳转指令（非跳转指令执行时不会修改 pc 变量的值）
    writes  bool // 是否是写指令（会修改 StatDB 中的数据）
    valid   bool // 是否是有效指令
    reverts bool // 指令指行完后是否中断执行并恢复状态数据库
    returns bool // 指令是否有返回值
}
``` 
这个结构体的字段几乎都比较好理解，这里就不多解释了。只有 `reverts` 字段需要多说一下。目前只有 `REVERT` 指令会设置这个字段为 true（ `REVERT` 指令应该对应了 solidity 语言的 `require` 和 `revert` 函数）。如果这个字段为 true，则当前指令执行完成后，`EVMInterpreter.Run` 立即返回错误 `errExecutionReverted`。这个错误值会一直传回给调用 `run` 函数的代码中，比如 `EVM.create` 或 `EVM.Call`。如果 `run` 返回的是这个错误值，那么将剩余的 gas 返还给调用者（出乎意料，如果是其它错误，所有 gas 将直接被耗尽），如下代码所示：
```go
func (evm *EVM) Call(caller ContractRef, addr common.Address, input []byte, gas uint64, value *big.Int) (ret []byte, leftOverGas uint64, err error) {
    ......

    ret, err = run(evm, contract, input, false)
    if err != nil {
        evm.StateDB.RevertToSnapshot(snapshot)
        if err != errExecutionReverted {
            contract.UseGas(contract.Gas)
        }
    }
}
```


### 跳转指令

跳转指令设计得稍微有点特殊，因此这里单独介绍一下。在合约的指令中，有两个跳转指令（不算 CALL 这类）：JUMP 和 JUMPI。它们的特殊的地方在于，跳转后的目标地址的第一条指令必须是 JUMPDEST。

我们以 JUMP 指令为例，看一下它的解释函数的代码：
```go
func opJump(pc *uint64, interpreter *EVMInterpreter, contract *Contract, memory *Memory, stack *Stack) ([]byte, error) {
    pos := stack.pop()
    if !contract.validJumpdest(pos) {
        nop := contract.GetOp(pos.Uint64())
        return nil, fmt.Errorf("invalid jump destination (%v) %v", nop, pos)
    }
    *pc = pos.Uint64()

    interpreter.intPool.put(pos)
    return nil, nil
}
```
这个函数解释执行了 JUMP 指令。代码首先从栈中取出一个值，作为跳转目的地。这个值其实就是相对于合约代码第 0 字段的偏移。然后代码会调用 `Contract.validJumpdest` 判断这个目的地的第一条指令是否是 JUMPDEST，如果不是这里就出错了。

现在的关键问题是如何判断目的地的第一条指令是 JUMPDEST。你可能会说很简单，看看目的地的第 0 个字节的值是否是 JUMPDEST 这个 opcode 码不是完了嘛。如果这个字节是一条指令，那确实是这样；但如果这个字节是数据，那就不对了。比如 JUMPDEST 这个 opcode 的值为 0x5b, 如果有下面一段指令：
> 60 5b 60 80 ......

把它翻译成指令的形式为：
```
PUSH1 0x5b
PUSH1 0x80
......
```
可以看到，这里的 0x5b 是作为数据出现的，而非指令。但如果某条 jump 指令跳到了第 1 个字节（从 0 开始计数），并把这个 0x5b 当成指令数据，得出跳转后第 0 条指令为 JUMPDEST，显然是不对的。

因此判断目的地第一条指令是否是 JUMPDEST，要保证两点：一是它的值为 JUMPDEST 指令的 opcode 的值；二是这是一条指令，而非普通数据。

下面我们介绍一下 `Contract.validJumpdest` 是如何做的。除了比对 opcode 外（这个很简单），`Contract` 还会创建一个位向量对象（即 `bitvec`，bit vector）。这个对象会从头至尾分析一遍合约指令，如果合约某一偏移处的字节属于普通数据，就将 `bitvec` 中这个偏移值对应的「位」设置为 1，如果是指令则设置为 0。在 `Contract.validJumpdest` 中，通过检查跳转目的地的偏移值在这个位向量对象中的「位」是否为 0，来判断此处是否是正常指令。

我们详细看一下 `Contract.validJumpdest` 的代码：
```go
func (c *Contract) validJumpdest(dest *big.Int) bool {
    udest := dest.Uint64()
    if dest.BitLen() >= 63 || udest >= uint64(len(c.Code)) {
        return false
    }

    // 检查目的地指令值是否为 JUMPDEST
    if OpCode(c.Code[udest]) != JUMPDEST {
        return false
    }

    // 下面的代码都是检查目的地的值是否是指令，而非普通数据

    // CodeHash 不为空的情况。一般调用合约时会进入这个分支
    if c.CodeHash != (common.Hash{}) {
        // Does parent context have the analysis?
        analysis, exist := c.jumpdests[c.CodeHash]
        if !exist {
            // Do the analysis and save in parent context
            // We do not need to store it in c.analysis
            analysis = codeBitmap(c.Code)
            c.jumpdests[c.CodeHash] = analysis
        }
        return analysis.codeSegment(udest)
    }

    // CodeHash 为空的情况，
    // 一般是因为新合约创建、还未将合约写入状态数据库
    if c.analysis == nil {
        c.analysis = codeBitmap(c.Code)
    }
    return c.analysis.codeSegment(udest)
}
```
代码的一开始检查目的地的偏移不能太大，因为合约代码不可能那么长。然后就是最简单的检查目的地第 0 个字节是否是 JUMPDEST 这个 opcode 值。当这些都符合要求时，就会继续检查目的地的值是否是指令而非数据。

检查是否为指令的代码有两个分支，但基本上它们做的事情是一样的。只不过 `Contract.jumpdests` 中包含了合约调用者的数据，而 `Contract.analysis` 只包含了当前合约的数据。这里的重点是 `codeBitmap` 函数生成的对象，以及这个对象的 `codeSegment` 方法。

我们先看一下 `codeBitmap` 函数：
```go
func codeBitmap(code []byte) bitvec {
    bits := make(bitvec, len(code)/8+1+4)
    for pc := uint64(0); pc < uint64(len(code)); {
        op := OpCode(code[pc])

        if op >= PUSH1 && op <= PUSH32 {
            // PUSH1 代表 push 指令后的 1 个字节都是用来入栈的数据
            // PUSH2 代表 push 指令后的 2 个字节都是用来入栈的数据
            // 以及类推，所以 numbits 为 op - PUSH1 + 1
            numbits := op - PUSH1 + 1 
            pc++
            for ; numbits >= 8; numbits -= 8 {
                bits.set8(pc) // 8
                pc += 8
            }
            for ; numbits > 0; numbits-- {
                bits.set(pc)
                pc++
            }
        } else {
            pc++
        }
    }
    return bits
}
```
这个函数用来生成位向量。位向量的类型为 `bitvec`，它的真正类型是 `[]byte`。在这个函数中，`code` 参数代表了某个合约的完整的合约代码，函数从头到尾扫描每一条指令，把普通数据的偏移值在位向量中设置成 1。`bits.set` 表示将指定偏移位设置成 1；`bits.set8` 表示将指定偏移位及其后面 7 个位都设置成 1。

（从这个函数上看，貌似只有 push 指令会在代码中夹杂数据，其它指令都不会如此。另外，我觉得这种从头到尾扫描的方式无法保证严格正确，即可能有些是数据但没有在位向量中置 1。也可能是合约编译器做了处理，保证这里可以得到正确结果）

有了 `bitvec` 对象，就可以通过 `bitvec.codeSegment` 判断某一个偏移位是否是数据了：
```go
func (bits *bitvec) codeSegment(pos uint64) bool {
    return ((*bits)[pos/8] & (0x80 >> (pos % 8))) == 0
}
```
这个方法只是查看指定偏移位的值是否为0，如果是，则代表这个偏移处的值是一条指令，而非数据。

总得来说，以太坊虚拟机中对跳转指令限制得比较严格，跳转目的地的第 0 条指令必须是 JUMPDEST，并且不光得 opcode 值相同，还得保证这确实是一条指令，而非普通数据。这种严格的限制可以保证虚拟机执行的正确性，也可能可以避免一些恶意攻击。


# 存储

以太坊虚拟机中有三个地方可以存储合约的数据：栈、内存块存储、状态数据库。其中只有状态数据库是永久性的，其它两个在合约运行结束后，就销毁了。

栈和内存块存储在解释器准备启动时被创建，如下所示：
```go
func (in *EVMInterpreter) Run(contract *Contract, input []byte, readOnly bool) (ret []byte, err error) {
    ......

    var (
        mem   = NewMemory() // bound memory
        stack = newstack()  // local stack
    )
    ......
}
```

下面我们分别详细介绍一下这几个存储对象。


### Stack

`Stack` 对象为合约的运行提供了栈功能，其中的数据先进后出。它的定义很简单：
```go
type Stack struct {
    data []*big.Int
}
```
它只有一个 `data` 字段，这是一个可变长数据，用来充当栈存储空间。可以看到它的元素的类型是 `big.Int`。

`newstack` 函数用来创建一个新的栈对象：
```go
func newstack() *Stack {
    return &Stack{data: make([]*big.Int, 0, 1024)}
}
```
这个函数也很简单，生成了一个预留 1024 个存储空间的数组填充 `data` 字段。

`Stack` 对象有几个操作方法，完全是按照「先进后出」的栈结构语义来实现的，比如 push 和 pop，再比如 swap 方法，其参数 `n` 也是指从栈顶（即数组最后一个元素）向前 n 个元素，而非从数组第 0 个元素开始算起。`Stack` 的方法实现都很简单，几乎都是一行代码搞定，因此这里就不再我说了。


### Memory

`Memory` 对象为合约的运行提供了一整块的平坦的存储空间。它的定义如下：
```go
type Memory struct {
    store       []byte
    lastGasCost uint64
}
```
可以看到这个对象的定义也很简单，`store` 字段用来提供一块平坦的内存作为存储空间。`lastGasCost` 用来在每次使用存储空间时，参与计算消耗的 gas。

`Memory` 对象提供了一些操作方法，如果 set 和 set32，用来将指定的数据存储到指定偏移中。`Memory.Resize` 方法接收一个数值，保证当前拥有的空间大小不小于这个数值。其它方法比较简单，就不一一介绍了。

需要注意的是，在合约中使用内存存储有可能是需要消耗 gas 的。消耗的大小由 `memoryGasCost` 计算确定。从这个函数的实现来看，只要所需空间超过当前空间的大小，超过部分就需要消耗 gas。


### 永久存储：StateDB

永久存储对象 `StateDB` 即是以太坊的状态数据库，这个数据库中包含了所有账户的信息，包括账户相关的合约代码和合约的存储数据。这个对象不是 evm 模块中的对象。想要详细了解状态数据库，可以参看[这篇文章](https://yangzhe.me/2019/06/19/ethereum-state/)。


# 其它辅助对象

为了文章的完整性，这里稍微提一下 evm 模块中的其它辅助对象。但其实这些对象不是 evm 的核心功能，无需花太多精力去了解。


### intPool

`intPool` 是 `big.Int` 类型的变量的一个缓存池。在 evm 模块的代码中，尤其是指令的解释函数中，当需要新申请或销毁一个 `big.Int` 变量时，都可以通过 `intPool` 来进行。

在 `intPool` 内部，这个对象使用 `Stack` 对象存储一些暂时不被使用的 `big.Int` 变量。当有人调用 `intPool.get` 申请一个 `big.Int` 变量时，它会从栈中取出一个，或栈为空时新创建一个；当有人调用 `intPool.put` 销毁一个 `big.Int` 变量时，它会将这个变量放到内部的栈中缓存起来（除非栈中缓存的数据已达到上限），供后续有人调用 `intPool.get` 时重复利用。


### logger

在 evm 模块中，log 对象有 `JSONLogger` 和 `StructLogger` 两种，它们都以一定的格式输出一些 log 数据，这些功能没有什么可详细说的，了解一下就行。

如果设置 `vm.Config.Debug` 字段为 true ，那么在虚拟机运行时，`StructLogger` 会有一些详细的 log 输出，方便进行调试。但我没有找到有哪个选项或配置文件可以设置这个字段，或许只是在调试代码时手动写一下这个值吧。


# 总结

evm 模块实现了运行以太坊智能合约的虚拟机，是一个非常重要的模块。在这篇文章里，我们详细了解了 evm 模块的实现，包括合约是如何创建和调用的；合约指令是如何被解释执行的等等。

我们在文章开始已经说过，以太坊智能合约是很重要也很复杂的实现，不能只靠 evm 模块了解智能合约的全部，evm 只提供了智能合约执行的虚拟机，是比较小的一部分。如果想要完整了解以太坊智能合约的实现，还需要学习智能合约的编译器 [solidity 项目](https://github.com/ethereum/solidity)。

水平有限，如果文章中有错误，还请不吝指正。