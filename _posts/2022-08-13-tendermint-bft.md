---
layout: post
title:  "Tendermint 中的拜占庭共识算法"
categories: blockchain
tags: 原创 tendermint consensus
excerpt: 博客终于又开始更新啦哈哈
author: fatcat22
---

* content
{:toc}




> 这篇文章，我将介绍一下 Tendermint 的共识算法和实现，其中会用到 PBFT 的一些知识，所以如果您对 PBFT 不太了解，最好能提前学习一下，比如读一下我之前写的[关于拜占庭将军问题的文章](https://yangzhe.me/2019/11/06/byzantine-generals-problem/)、[关于 PBFT 的文章](https://yangzhe.me/2019/11/25/pbft/)。

> 本文章分析的代码为[Tendermint 官方的 tag 号为 v0.35.9 的代码](https://github.com/tendermint/tendermint/tree/v0.35.9)。


# 什么是 Tendermint

[Tendermint](https://tendermint.com/) 是一个用于创建区块链的库，它提供了基于拜占庭容错的 PoS 共识、p2p 网络库的实现，以及一套称之为 ABCI(Application Blockchain Interface) 的接口，可以让开发者基于它提供的库和接口，使用任意语言快速开发自己的区块链。 [Cosmos](https://cosmos.network/) 应该是 Tendermit 支撑的最有名的区块链了。我们本篇文章的重点，就是 Tendermint 中的拜占庭容错算法。

其实我们之前的好几篇文章都分析过拜占庭问题了，但一方面 Tendermint 库确实是一个比较知名的库，另一方面它的实现与之前的经典 PBFT 确实有不同之处，可以说算法更简单了，所以我觉得还是有必要好好学习一下。


# Tendermint 中的 BFT 有什么不同？

在 [The latest gossip on BFT consensus](https://arxiv.org/pdf/1807.04938.pdf) 这篇论文里，作者描述了 Tendermint 的拜占庭容错算法的完整算法和证明。简单来说，Tendermint 的共识与之前我们介绍的 [PBFT](https://yangzhe.me/2019/11/25/pbft/) 最大的不同，是出错后的处理方式。

从 [PBFT](https://yangzhe.me/2019/11/25/pbft) 这篇文章里我们知道，那个 PBFT 的实现有「检查点」和「域转换」这两个概念。这两个概念的引入，主要是为了在错误出现时，系统仍能纠正错误并正常运转。但检查点这个概念的实现是相对低效的：一方面，每创建一个检查点，所有结点都要广播 CHECKPOINT 消息从而占用一部分网络资源；另一方面，当发生「域转换」时，$\langle VIEW$-$CHANGE \rangle$ 消息中会携带至少 $2f+1$ 条最新的检查点的证明消息，以及所有从这个检查点开始所有处于未完成的状态的 $\langlePRE$-$PREPARE\rangle$ 消息；因此 $\langle VIEW$-$CHANGE \rangle$ 消息不但很大消耗网络资源，并且每个结点在发生域转换时，都要从最新的检查点开始重新处理一遍这些处于 $\langlePRE$-$PREPARE\rangle$ 状态的消息，这个过程可能是会很慢的。

所以总得来说，那个 PBFT 的机制不但更多的占用网络资源，而且各结点在从错误中恢复、重新达成一致状态的过程也更慢。

 Tendermint 中 BFT 的实现则取消了「检查点」这个概念，取而代之的是 `ValidRound` 和 `ValidValue` （在代码中则为 `ValidRound` 和 `ValidBlock` 变量）。每个正确的结点都会保存这么一个变量，当结点发现一个 Proposal（提案, 即 PBFT 中的 $\langlePRE$-$PREPARE\rangle$ 消息，在区块链中即代表一个 Block）已经被至少 2/3 的结点认可时，它就会将这个 Proposal 记录到 ValidValue 中；如果因为某些原因这个 Proposal 提交（Commit）失败而发生了类似「域转换」这个的事情时，新的类似「主结点」的结点就会直接使用 `ValidValue` 中的值作为新的 Proposal，而不会重新计算，也不会有类似「检查点」同步和重新从「检查点」计算这样的过程。这就大大简化了这一过程。
 
 在 [论文](https://arxiv.org/pdf/1807.04938.pdf) 中有关于以上这个方式的正确性的证明。虽然我也没去细看证明过程，但我们可以从不那么学术的角度大体去想一下这个方法是否正确：当一个 Proposal 有至少 2/3 的结点认可时，其实我们可以认为这个 Proposal 肯定是正确的，它没有被最终提交（Commit）不是因为这个 Proposal 本身有问题，而应该是类似于网络不通这种问题导致的。所以在「下一轮」需要重新提交 Proposal 时，直接使用刚才已经被 2/3 的结点认可的数据（即 `ValidBlock`）是可以的。

刚才我们提到了「下一轮」这个词，这里需要解释一下。在 Tendermint 中没有「域转换」这个概念，取而代之的是 `Round` ，即「轮」。在每一个区块高度上，Proposal 总是从 `Round 0` 开始，即第 0 轮；如果这一轮数据没有被最终提交，则保持区块高度不变，进入到下一轮即 `Round 1`，直到成功，区块高度加 1 ，又从 `Round 0` 开始，周而复始。每进行一轮，Proposal 的提交者都会改变，这其实就是「域转换」，只不过这种方式更简单高效，不需要广播类似于 $\langle VIEW$-$CHANGE \rangle$ 这样的消息。

最后一点不太相同的是，PBFT 中把共识确认的三阶段叫做 PRE-PREPARE、PREPARE、COMMIT；而在 Tendermint 中，这三个阶段分别被叫做 Proposal、Prevote、Precommit。它们只是名字不一样，其实意义是一样的。

以上就是 Tendermint 的 BFT 实现与 PBFT 的主要不同点，也是 Tendermint 宣称更高效的主要原因。下面我们通过 Tendermint 的代码，详细看一看是如何实现的。（如果你只是关心算法上的差异，并且刚才说的已经完全理解的话，其实后面的内容也可以不看）

另外 Tendermint 也有[介绍共识的官方文档](https://docs.tendermint.com/master/spec/consensus/)，供感兴趣的朋友参考。


# 代码详解

从这里开始，我们就入代码入手，详细看一下 Tendermint 是如何实现这个共识算法的。


## 框架：数据是如何流转的

我作了两张图，希望能从全局的视角描绘出 Tendermint 的共识流程：

![共识流程汇总图](/pic/tenderminnt-bft/summary.png)

![超时流程](/pic/tenderminnt-bft/timeout.png)

但我自己都觉得这两张图画得很乱（尤其第一张），但这已经是我花了挺长时间整理的了，实在想不出在不缺少更多信息的情况下怎么能画得更清晰了。尽力了，大家凑合看吧。

不过细看这张图的话也能看出，整个流程是围绕 `receiveRoutine` 这个 go routine 循环展开的。`timeoutTicker` 负责处理超时的逻辑；`peerMsgQueue` 负责处理外部的消息事件（p2p 网络从其它结点接收到的消息）；`internalMsgQueue` 如其名所示，处理内部消息，所谓内部消息，就是自己发起的 Proposal 消息、Block 数据、投票消息等。

下面我们分别从正常流程和异常发生时的处理流程两个角度，去详细看一下代码的实现。


## 正常流程

我们先来看看正常流程是什么样的。前面我们提到过，Tendermint 共识达成也分三个阶段，分别为 Proposal 、Prevote、Precommit，共识达成后，就会将区块提交到链上，然后高度加 1 ，这一步被称为 `commit`。下面就分别看看这些阶段都是怎么做的。


### Proposal

达成一个共识的起始步聚是提出一个提案，即 Proposal。正常情况下每个 Proposal 的发起都是从 `scheduleRound0` 开始的，即某一高度的第 0 轮。这个方法的实现很简单，就是设置超时事件 `RoundStepNewHeight` 并等待触发：
```go
func (cs *State) scheduleRound0(rs *cstypes.RoundState) {
	sleepDuration := rs.StartTime.Sub(tmtime.Now())
	cs.scheduleTimeout(sleepDuration, rs.Height, 0, cstypes.RoundStepNewHeight)
}
```

这段代码中，`StartTime` 是本轮的开始时间；`scheduleTimeout` 则会将参数中的信息保存并设置一个 timer，当 timer 达到设定的时间后，将信息发送给一个特定的 channel，而在 `receiveRoutine` 中则会接收到这个信息并调用 `handleTimeout` 进行处理：
```go
func (cs *State) handleTimeout(ctx context.Context, ti timeoutInfo, rs cstypes.RoundState) {
    /// ...... previous code

	switch ti.Step {
	case cstypes.RoundStepNewHeight:
		cs.enterNewRound(ctx, ti.Height, 0)
    
    // ...... others code
    }

    /// ...... others code
}
```

可以看到在收到 `RoundStepNewHeight` 后，就会调用 `enterNewRound`，并且第 3 个参数直接为 0 。所以很容易看出来，从这里进入到了高度为 `ti.Height` 的第 0 轮提案。

在 `enterNewRound` 中，最主要的是做一些新一轮的初始化工作，比如设置当前轮的提案者（Proposer），初始化当前轮的投票数据，然后调有 `enterPropose` 深度发起提案：
```go
func (cs *State) enterNewRound(ctx context.Context, height int64, round int32) {
    /// hiden code ......

	validators := cs.Validators
	if cs.Round < round {
		validators = validators.Copy()
		r, err := tmmath.SafeSubInt32(round, cs.Round)
		if err != nil {
			panic(err)
		}
		validators.IncrementProposerPriority(r)
	}

	cs.updateRoundStep(round, cstypes.RoundStepNewRound)
	cs.Validators = validators

    /// hiden code ......

	cs.Votes.SetRound(r) // also track next round (round+1) to allow round-skipping

    /// hiden code ......

	cs.enterPropose(ctx, height, round)
}
```
在上面的代码中，如果 `cs.Round < round` 成立，则会调用 `IncrementProposerPriority` 生成新的 Proposer ，否则不会调用此方法，即仍使用之前的 Proposer。`cs.Round` 其实是 `cs.RoundState.Round`，它代表的是当前进在进行的是哪一轮。所以这个判断是要看一下即将开始的这一轮是否比当前正在进行的轮大，也即是否要开始新的一轮。

你可以会有疑问：方法名不就是 `enterNewRound` 吗？怎么还会判断「是否要开始新的一轮」。按道理来说是这样的，但代码的实现有两个特殊情况：
1. 如果是某一高度的第 0 轮，则 `cs.Round` 会被初始化为 0 (参数 `updateToState` 中对 `updateRoundStep` 的调用)，而参数 `round` 也是 0 （正如我们刚才提到的），所以此时「开始新的一轮」是指在新的高度上开始进入 round 0 ，而不是指「由 round X 进入到 round X+1」。
2. 为了确保万无一失，在共识达成过程中仍有很多地方会调用 `enterNewRound` ，但此时的参数 `round` 很可能与 `cs.Round` 相同甚至较小，所以仍要判断一下。

从这里我们也可以看出，只要某个 Proposer 一直成功，TA 就可以一直作为 Proposer 提案（即出块）。这是因为 「round」 在整个共识中起的作用与「域转换」相同，都是在失败时尝试重新达成共识的机制，所以当高度不变、重新开始新的一轮时，代表之前的「旧的那一轮」失败了，此时才会去更换 Proposer；当某一轮成功了，高度就会加 1 ，此时调用 `enterNewRound` 由于 `cs.Round` 和 `round` 都是 0 ，就不会更换 Proposer 了。（但是不知道这个机制合理吗？只要 TA 不失败就一直让 TA 出块，那挖矿的钱岂不是都让这个人挣去了哈哈）

初始化这些数据之后，接下来就进入到了 `enterPropose` 方法。我们看看 `enterPropose` 都做了些什么：
```go
func (cs *State) enterPropose(ctx context.Context, height int64, round int32) {
    /// hiden code ......

	// If this validator is the proposer of this round, and the previous block time is later than
	// our local clock time, wait to propose until our local clock time has passed the block time.
	if cs.privValidatorPubKey != nil && cs.isProposer(cs.privValidatorPubKey.Address()) {
		proposerWaitTime := proposerWaitTime(tmtime.DefaultSource{}, cs.state.LastBlockTime)
		if proposerWaitTime > 0 {
			cs.scheduleTimeout(proposerWaitTime, height, round, cstypes.RoundStepNewRound)
			return
		}
	}

    // hiden code ......
}
```
一开始此方法会判断自己是不是当前轮的 Proposer，如果是、并且之前的区块的时间比当前时间晚，那就需要等一会再提新的区块。否则继续：
```go
func (cs *State) enterPropose(ctx context.Context, height int64, round int32) {
    /// hiden code ......

	defer func() {
		// Done enterPropose:
		cs.updateRoundStep(round, cstypes.RoundStepPropose)
		cs.newStep()

		// If we have the whole proposal + POL, then goto Prevote now.
		// else, we'll enterPrevote when the rest of the proposal is received (in AddProposalBlockPart),
		// or else after timeoutPropose
		if cs.isProposalComplete() {
			cs.enterPrevote(ctx, height, cs.Round)
		}
	}()

	// If we don't get the proposal and all block parts quick enough, enterPrevote
	cs.scheduleTimeout(cs.proposeTimeout(round), height, round, cstypes.RoundStepPropose)

    /// hiden code ......
}
```
在这段代码中，先定义了一个 defer ，即在方法退出时，调用 `updateRoundStep` 更新 Step，并且如果退出时 Proposal 已经完成（`isProposalComplete`）则直接调用 `enterPrevote` 进入到 Prevote 阶段。然后插入了一个 `RoundStepPropose` 超时，即最晚在这段时间以后要进入到 Prevote 阶段，如果进入 Prevote 阶段 Proposal 数据还没有准备好，就会 prevote 给 nil 。

我们继续看 `enterPropose` 最后一部分代码：
```go
func (cs *State) enterPropose(ctx context.Context, height int64, round int32) {
    /// hiden code ......

	if cs.privValidator == nil {
		return
	}

	if cs.privValidatorPubKey == nil {
		return
	}

	addr := cs.privValidatorPubKey.Address()

	// if not a validator, we're done
	if !cs.Validators.HasAddress(addr) {
		return
	}

	if cs.isProposer(addr) {
		cs.decideProposal(ctx, height, round)
	} 
}
```
这段代码我删掉了所有 log ，让代码变得更清晰一些。很显然，这段代码在判断当前结点是否一个 validator 并且是当前轮的 Proposer ，如果是则调用 `decideProposal` 产生新的区块，如果不是就直接返回了。

下面我们再来看看 `decideProposal` 都做了什么。为了可以进行单元测试，代码中 `decideProposal` 并不是一个方法名，而是一个函数指针，正常情况下指向了 `decideProposal`：
```go
func (cs *State) defaultDecideProposal(ctx context.Context, height int64, round int32) {
	var block *types.Block
	var blockParts *types.PartSet

	// Decide on block
	if cs.ValidBlock != nil {
		// If there is valid block, choose that.
		block, blockParts = cs.ValidBlock, cs.ValidBlockParts
	} else {
		// Create a new proposal block from state/txs from the mempool.
		var err error
		block, err = cs.createProposalBlock(ctx)
		if err != nil {
			cs.logger.Error("unable to create proposal block", "error", err)
			return
		} else if block == nil {
			return
		}
		blockParts, err = block.MakePartSet(types.BlockPartSizeBytes)
		if err != nil {
			return
		}
	}

    /// hiden code ......

    proposal := types.NewProposal(height, round, cs.ValidRound, propBlockID, block.Header.Time)

    /// hiden code ......
}
```
此函数很直接如名字所说的，一开始就创建出一个提案（即区块）。正如我们前面所说，如果 `ValidBlock` 非空的话，就会直接使用这个值；否则才会调用 `createProposalBlock` 去创建一个新的区块。`ValidBlock` 代表的是已经有 2/3 的结点认可的区块了，所以虽然没有成功提交，但新的一轮里仍然直接提交这个区块而不用再去创建新的了。

```go
func (cs *State) defaultDecideProposal(ctx context.Context, height int64, round int32) {
    /// hiden code ......

	p := proposal.ToProto()

	if err := cs.privValidator.SignProposal(ctxto, cs.state.ChainID, p); err == nil {
		proposal.Signature = p.Signature

		// send proposal and block parts on internal msg queue
		cs.sendInternalMessage(ctx, msgInfo{&ProposalMessage{proposal}, "", tmtime.Now()})

		for i := 0; i < int(blockParts.Total()); i++ {
			part := blockParts.GetPart(i)
			cs.sendInternalMessage(ctx, msgInfo{&BlockPartMessage{cs.Height, cs.Round, part}, "", tmtime.Now()})
		}
    }
}
```
在这段代码里，Proposal 被签名以后，会调用 `sendInternalMessage` 将其发给 `internalMsgQueue` 这个 channel， Block 数据也同样会被分批发给它（`blockParts` 实际上就是把 Block 切成了一个个的小块数据，这些小块数据拼接在一起，仍然是一个完整的 Block 。之所以要切成小块，应该是因为可能一个 Block 会比较大，网络传输一整个 Block 会比较吃力失败的可能性更高）。

`internalMsgQueue` 收到一个 Proposal 数据后，会调用 `setProposal` 去处理，这个字段也是为了方便写测试用例而做成了一个指针，实际指向的是 `defaultSetProposal`：
```go
func (cs *State) defaultSetProposal(proposal *types.Proposal, recvTime time.Time) error {
    /// hiden code ......

	if !cs.Validators.GetProposer().PubKey.VerifySignature(
		types.ProposalSignBytes(cs.state.ChainID, p), proposal.Signature,
	) {
		return ErrInvalidProposalSignature
	}

	proposal.Signature = p.Signature
	cs.Proposal = proposal
	cs.ProposalReceiveTime = recvTime

    /// hiden code ......
}
```
这个方法一开始做了一些检查，但最重要的还是上面展示的这段代码，即检查 Proposal 签名的有效性，然后将其保存到 `cs.Proposal` 字段。

下面我们再看看 `internalMsgQueue` 收到 `BlockPartMessage` 消息是怎么处理的：
```go
func (cs *State) handleMsg(ctx context.Context, mi msgInfo) {
    /// hiden code ......

    added, err = cs.addProposalBlockPart(msg, peerID)

    cs.mtx.Unlock()

    cs.mtx.Lock()
    if added && cs.ProposalBlockParts.IsComplete() {
        cs.handleCompleteProposal(ctx, msg.Height)
    }

    /// hiden code
}
```
这里首先将数据保存到 `cs.ProposalBlockParts` 字段中，然后检查 `cs.ProposalBlockParts` 中的数据是否完整（「完整」代表它所包含的数据能够组成刚才提案的 Block），如果接收完整就调用 `handleCompleteProposal` 进行处理。

这里你可能会奇怪为什么会插入一个 `cs.mtx.Unlock` 和 `cs.mtx.Lock`。首先解释为什么可以先 Unlock 再 Lock ：这是因为在 `handleMsg` 这个方法中，一上来就先调用了 `cs.mtx.Lock` ，所以这里可以调用 Unlock 把锁打开一下，然后再调用 Lock 重新锁上了。其实再解释一下代码这样写的目的：从原代码的注释（为了节约篇幅已被我删掉）可以看出来，昨时 Unlock 一下是为了让其它地方可以获取到 `RoundState` ，所谓的 `RoundState` 一个结构体，是本轮共识的一些数据集合，前面提到过的 `cs.ValidBlock` 、 `cs.Proposal` 、`cs.ProposalBlockParts` 其实都是 `RoundState` 的字段（只不过用了 go 的语法糖看上去不明显罢了）。那什么地方会获取 `RoundState` 呢？经过一番探索我发现有一个 `GetRoundState` 方法，它是实现是这样的：
```go
func (cs *State) GetRoundState() *cstypes.RoundState {
	cs.mtx.RLock()
	defer cs.mtx.RUnlock()

	// NOTE: this might be dodgy, as RoundState itself isn't thread
	// safe as it contains a number of pointers and is explicitly
	// not thread safe.
	rs := cs.RoundState // copy
	return &rs
}
```
可以这里会先获取与刚才相同的锁，然后获取 `RoundState` 。那么谁会调用 `GetRoundState` 呢？经过搜索我发现 `reactor.go` 中的代码会调用它：
```go
func (r *Reactor) updateRoundStateRoutine(ctx context.Context) {
	t := time.NewTicker(100 * time.Microsecond)
	defer t.Stop()

	for {
		select {
		case <-ctx.Done():
			return
		case <-t.C:
			rs := r.state.GetRoundState()
			r.mtx.Lock()
			r.rs = rs
			r.mtx.Unlock()
		}
	}
}
```
可以看到每隔 100 毫秒就会调用 `GetRoundState` 。

至此我们基本就可以理解刚才的代码里为什么先 Unlock 再 Lock 了：当接收到了一个新的区块部分时，主动让出一下执行权，让其它代码可以尽到的拿到新接收到的区块数据。（忍不住吐槽一下，这机制和代码太差劲了....）

刚才在处理 `BlockPartMessage` 时，如果区块数据完整，就会调用 `handleCompleteProposal` 来处理这个提案，我们现在来看一下代码：
```go
func (cs *State) handleCompleteProposal(ctx context.Context, height int64) {
	// Update Valid* if we can.
	prevotes := cs.Votes.Prevotes(cs.Round)
	blockID, hasTwoThirds := prevotes.TwoThirdsMajority()
	if hasTwoThirds && !blockID.IsNil() && (cs.ValidRound < cs.Round) {
		if cs.ProposalBlock.HashesTo(blockID.Hash) {
			cs.ValidRound = cs.Round
			cs.ValidBlock = cs.ProposalBlock
			cs.ValidBlockParts = cs.ProposalBlockParts
		}
	}

	if cs.Step <= cstypes.RoundStepPropose && cs.isProposalComplete() {
		// Move onto the next step
		cs.enterPrevote(ctx, height, cs.Round)
		if hasTwoThirds { // this is optimisation as this will be triggered when prevote is added
			cs.enterPrecommit(ctx, height, cs.Round)
		}
	} else if cs.Step == cstypes.RoundStepCommit {
		// If we're waiting on the proposal block...
		cs.tryFinalizeCommit(ctx, height)
	}
}
```
从代码里可以看出来，处理这个提案首先是看 prevote 的投票数是不是超过 2/3 ，如果超过 2/3 则把相关信息记录到 `ValidRound` 、`ValidBlock` 、`ValidBlockParts` 中。前面我们提到过，如果提案最终提交失败，下一轮提案时如果 `ValidBlock` 有值，就使用这个值；而这个值就是在这里初始化的。

代码接下来就根据记录的步骤进入到不同阶段了，根据我们刚才的流程，当前应该是 `RoundStepPropose` 阶段，所以就会调用 `enterPrevote` 进入到 Prevote 阶段。

#### Propser 是如何更换的

前面我们分析进入「新一轮」提案时，会调用 `IncrementProposerPriority` 修改提案者（Proposer），这一小节我们看一下 Proposer 是如何确定的，虽然这与共识算法无关，但也 PoS 共识中比较重要的一部分了。

这里我这打算扣着代码一点一点介绍，只是根据 `IncrementProposerPriority` 的实现，把我的理解写入注释里解释一下就行了，至于详细的数学运算，相信较真的读者可以自己看懂。

有一点需要提前说明一下的是，PoS 共识里每个 validator 的投票权是不一样的，投票权越高的人，出块的机率也越大。「投票权」这个概念体现在当前的代码里就是 「voting power」，即 `Validator.VotingPower` 字段，这个值一般情况是不变的；而计算 Proposer 时也会用到 `Validator.ProposerPriority` ，这个值在每次计算时都会发生变化。

下面是 `IncrementProposerPriority` 的实现以及我自己添加的注释：
```go
func (vals *ValidatorSet) IncrementProposerPriority(times int32) {
    // PriorityWindowSizeFactor 的定义是常量 2 。
    // diffMax 的意思是，所有 Validator.ProposerPriority 的最大值和最小值之差，不能超过 diffMax。
    // 所以这里规定了所以 Validator.ProposerPriority 的最大值和最小值之差，不能超过 2 倍的 TotalVotingPower(所有 validators 的 voting power 加起来的总量)。
	diffMax := PriorityWindowSizeFactor * vals.TotalVotingPower()
    // RescalePriorities 用于确保上述的规则，即 所有 Validator.ProposerPriority 的最大值和最小值之差，不能超过 diffMax。
    // 它使用等比例缩小（除法）的方式来保证这一点。
	vals.RescalePriorities(diffMax)
    // shiftByAvgProposerPriority 用于保证让所有 ProposerPriority 的值分布于 0 的左右（有正的也有负的），
    // 它的做法是让每一个 ProposerPriority 减去所有 ProposerPriority 的平均值。
	vals.shiftByAvgProposerPriority()

	var proposer *Validator
    // incrementProposerPriority 将每一个 validator 的 ProposerPriority 值，加上它们自己的 VotingPower 值。
    // 最终选出 ProposerPriority 值最大的那个 validator 返回。但在返回之前，会将这个 validator 的 ProposerPriority
    // 减去 TotalVotingPower ，以减小它下次还最大的可能性。
	for i := int32(0); i < times; i++ {
		proposer = vals.incrementProposerPriority()
	}

	vals.Proposer = proposer
}
```
总得来说，调整及选择 Proposer 有以下几步：
1. 调整 `ProposerPriority`  的差距不要超过 2 倍的 TotalVotingPower
2. 将所有 `ProposerPriority` 的值向坐标轴的右侧推移，使得推移完的所有值的平均值是 0 。（所以到这一步，所有 `ProposerPriority` 值分布于 0 的左右两侧，最大值不超过正的 TotalVotingPower，最小值不超过负的 TotalVotingPower）
3. 将 `ProposerPriority` 加上各自的 `VotingPower` 值，并选择最大的那个作为 Proposer。
4. 将 Propser 的 `ProposerPriority` 减去 TotalVotingPower ，以此减少此 validator 下次被选为 Propser 的机会。

尤其是 3 、4 两步，可以保证已经被选过的往队列后面（坐标轴右边）移动，从而降低下次被选中的机会，同时又保证 Voting Power 比较大的人可以快速移动队列前面（坐标轴左边），根据 power 大小增加相应的选中机会。但另一方面，在即使有一个人 power 非常大、其它人非常小的情况时，这个方法也能保证虽然这个 power 非常大的人会连续多次都被选中（因为人家其实有很大的 power），但每被选中一次，第 4 步中他的位置就会比上次选中时更往队尾（坐标轴右边）一些，直到某次他的 power 不足以让他在第 3 步中再次变成最大，此时机会就轮到了别人。

我写了一个[测试程序](https://go.dev/play/p/Swg5XHvbYLb)，模拟 power 分别为 40 、4 、1 的三个 validator 在每次选举时的情况，最前面的部分输出如下：
```
ValidatorSet{vals: [{power: 40, priority: -5} {power: 4, priority: 4} {power: 1, priority: 1}]}; proposer: {power: 40, priority: -5}
ValidatorSet{vals: [{power: 40, priority: -10} {power: 4, priority: 8} {power: 1, priority: 2}]}; proposer: {power: 40, priority: -10}
ValidatorSet{vals: [{power: 40, priority: -15} {power: 4, priority: 12} {power: 1, priority: 3}]}; proposer: {power: 40, priority: -15}
ValidatorSet{vals: [{power: 40, priority: -20} {power: 4, priority: 16} {power: 1, priority: 4}]}; proposer: {power: 40, priority: -20}
ValidatorSet{vals: [{power: 40, priority: -25} {power: 4, priority: 20} {power: 1, priority: 5}]}; proposer: {power: 40, priority: -25}
ValidatorSet{vals: [{power: 40, priority: 15} {power: 4, priority: -21} {power: 1, priority: 6}]}; proposer: {power: 4, priority: -21}
ValidatorSet{vals: [{power: 40, priority: 10} {power: 4, priority: -17} {power: 1, priority: 7}]}; proposer: {power: 40, priority: 10}
ValidatorSet{vals: [{power: 40, priority: 5} {power: 4, priority: -13} {power: 1, priority: 8}]}; proposer: {power: 40, priority: 5}
.......
```
可以看到 power 为 40 的 validator 最选为 Proposer 的次数最多，但 5 次以后就会轮到 power 为 4 的人，更多次以后也可以轮到 power 为 1 的人。


#### Proposal 来自于其它结点

对于一个共识结点来说，除了自己作为 Proposer 提出 Proposal，多数情况下会从其它结点接收其它 Proposer 提出的 Proposal 并对其进行处理。其实其它结点的 Proposal 是从 `peerMsgQueue` 来的，向这个 channel 发消息的，是 `reactor.go` 中的 `Reactor.handleDataMessage`，这个方法的调用层次有点深，这里就不详细展示了，总之能理解 `Reactor.handleDataMessage` 是处理来自 p2p 的消息就行了。

`peerMsgQueue` 收到 Proposal 后，会调用 `handleMsg` 进行处理，此方法又会调用 `setProposal` （即 `defaultSetProposal`） 进行处理，而 `defaultSetProposal` 我们前面已经分析过了，所以流程就又接上了。


### Prevote

接下来我们看看 Prevote 阶段的处理。在上一阶段，代码进入了 `enterPrevote` ，很明显这个方法就是处理 Prevote 逻辑的。这个方法很简单，除了设置新的步骤 `RoundStepPrevote` ，主要工作是在 `doPrevote` 里完成的。这个字段为了方便写测试用例，实际上是一个指针，指向 `defaultDoPrevote`：
```go
func (cs *State) defaultDoPrevote(ctx context.Context, height int64, round int32) {
    /// hiden code ......

	// Validate proposal block, from Tendermint's perspective
	err := cs.blockExec.ValidateBlock(ctx, cs.state, cs.ProposalBlock)
	if err != nil {
		// ProposalBlock is invalid, prevote nil.
		cs.signAddVote(ctx, tmproto.PrevoteType, nil, types.PartSetHeader{})
		return
	}

	isAppValid, err := cs.blockExec.ProcessProposal(ctx, cs.ProposalBlock, cs.state)
	// Vote nil if the Application rejected the block
	if !isAppValid {
		cs.signAddVote(ctx, tmproto.PrevoteType, nil, types.PartSetHeader{})
		return
	}

    /// hiden code ......
}
```
`defaultDoPrevote` 最开始的一段代码主要是对重要字段的检查，这里就略过了。上面展示的代码则是对 Block 的检查，从原本的注释也能看出来，首先是 Tendermint 视角的检查来看看这是不是一个有效的 Block；然后是调用 App 的接口检查，由于 Tendermint 本质上是一个库，所以有了新的 Block 以后要通过 ABCI 让调用者去检查一下 Block 是否有效。如果无效，就会投票给 nil 。

我们继续看接下来的处理。接下来的两段代码都有很长的注释，其中有论文中的算法伪代码，也有对其的解释：
```go
func (cs *State) defaultDoPrevote(ctx context.Context, height int64, round int32) {
    /// hiden code ......

	/*
		22: upon <PROPOSAL, h_p, round_p, v, −1> from proposer(h_p, round_p) while step_p = propose do
		23: if valid(v) && (lockedRound_p = −1 || lockedValue_p = v) then
		24: broadcast <PREVOTE, h_p, round_p, id(v)>

		Here, cs.Proposal.POLRound corresponds to the -1 in the above algorithm rule.
		This means that the proposer is producing a new proposal that has not previously
		seen a 2/3 majority by the network.

		If we have already locked on a different value that is different from the proposed value,
		we prevote nil since we are locked on a different value. Otherwise, if we're not locked on a block
		or the proposal matches our locked block, we prevote the proposal.
	*/
	if cs.Proposal.POLRound == -1 {
		if cs.LockedRound == -1 {
			logger.Debug("prevote step: ProposalBlock is valid and there is no locked block; prevoting the proposal")
			cs.signAddVote(ctx, tmproto.PrevoteType, cs.ProposalBlock.Hash(), cs.ProposalBlockParts.Header())
			return
		}
		if cs.ProposalBlock.HashesTo(cs.LockedBlock.Hash()) {
			logger.Debug("prevote step: ProposalBlock is valid and matches our locked block; prevoting the proposal")
			cs.signAddVote(ctx, tmproto.PrevoteType, cs.ProposalBlock.Hash(), cs.ProposalBlockParts.Header())
			return
		}
	}

    /// hiden code ......
}
```
到这里区块的有效性都检查完了，如果没问题应该投票了。上面这段代码处理的就是这种情况：当前高度上不管之前有几轮，都没有达到过某区块在 Prevote 阶段有 2/3 的人投票的情况，更没有提交成功。这里 `cs.Proposal.POLRound == -1` 代表的就是这个意思（ `POLRound` 其实就是 `ValidRound` 的值）。`LockedRound` 和 `LockedBlock` 的值的意思是，在 `LockedRound` 这一轮中 `LockedBlock` 这个区块在 Prevote 阶段达到 2/3 的人投票数量。如注释中解释的那样，如果当前提案的区块与 `LockedBlock` 不同，那就不能给当前提案的区块投票（因为按道理 `LockedBlock` 既然 Prevote 已经有 2/3 的票，即使 `LockedRound` 那一轮没成功，下一轮仍然应该提案这个区块）。

上面的代码如果没被执行，那代表 `cs.Proposal.POLRound` 的值不是 -1 ，也即之前有某一轮某区块在 Prevote 阶段有 2/3 的投票，那就会来到下面的代码段：
```go
func (cs *State) defaultDoPrevote(ctx context.Context, height int64, round int32) {
    /// hiden code ......

	/*
		28: upon <PROPOSAL, h_p, round_p, v, v_r> from proposer(h_p, round_p) AND 2f + 1 <PREVOTE, h_p, v_r, id(v)> while
		step_p = propose && (v_r ≥ 0 && v_r < round_p) do
		29: if valid(v) && (lockedRound_p ≤ v_r || lockedValue_p = v) then
		30: broadcast <PREVOTE, h_p, round_p, id(v)>

		This rule is a bit confusing but breaks down as follows:

		If we see a proposal in the current round for value 'v' that lists its valid round as 'v_r'
		AND this validator saw a 2/3 majority of the voting power prevote 'v' in round 'v_r', then we will
		issue a prevote for 'v' in this round if 'v' is valid and either matches our locked value OR
		'v_r' is a round greater than or equal to our current locked round.

		'v_r' can be a round greater than to our current locked round if a 2/3 majority of
		the network prevoted a value in round 'v_r' but we did not lock on it, possibly because we
		missed the proposal in round 'v_r'.
	*/
	blockID, ok := cs.Votes.Prevotes(cs.Proposal.POLRound).TwoThirdsMajority()
	if ok && cs.ProposalBlock.HashesTo(blockID.Hash) && cs.Proposal.POLRound >= 0 && cs.Proposal.POLRound < cs.Round {
            // 这里其实有两个 if 判断，但仅仅是为了输出不同的 log ，所以为了简洁我就把它们去掉了。
			cs.signAddVote(ctx, tmproto.PrevoteType, cs.ProposalBlock.Hash(), cs.ProposalBlockParts.Header())
			return
	}

	cs.signAddVote(ctx, tmproto.PrevoteType, nil, types.PartSetHeader{})
}
```
上面的代码中，`POLRound` 即注释中的 `v_r`。这里首先检查 `POLRound` 轮中是不是真的有区块在 Prevote 投票中超过 2/3 ，并拿到这个区块的 id 。如果确实有这样的区块、并且这个区块就是当前处理的提案的区块、再并且 `POLRound` 所处理的轮数比当前正在进行的轮数小，就投票给这个区块，否则就投票给 nil。原文的注释也解释了为什么这里判断 `cs.Proposal.POLRound < cs.Round` ：其实 `POLRound` 也当然可以大于 `cs.Round` ，但那代表我们当前结点可能错过了某些轮的处理，当前我们可能正落后着呢。

从上面的代码中可以看到，实际投票的行为都是 `signAddVote` 来做的，从名字就可以看出来，此方法会生成一个投票并签名，然后将其填加到某地。下面我们简单来看一下它的实现：
```go
func (cs *State) signAddVote(
	ctx context.Context,
	msgType tmproto.SignedMsgType,
	hash []byte,
	header types.PartSetHeader,
) *types.Vote {
    /// hiden code ......

    vote, err := cs.signVote(ctx, msgType, hash, header)

    /// hiden code ......

	cs.sendInternalMessage(ctx, msgInfo{&VoteMessage{vote}, "", tmtime.Now()})
	return vote
}
```
这个方法其实挺简单，前面是一些检查（我们省略掉了），然后就是调用 `signVote` 生成并签名一个 `vote` 数据，最后将 `vote` 发送给 `internalMsgQueue` 处理。

在 `internalMsgQueue` 消息循环中，`VoteMessage` 是调有 `tryAddVote` 来处理的，而此方法主要调用 `addVote` 来处理，所以我们就看看这个方法的实现：
```go
func (cs *State) addVote(
	ctx context.Context,
	vote *types.Vote,
	peerID types.NodeID,
) (added bool, err error) {
    /// hiden code ......

	added, err = cs.Votes.AddVote(vote, peerID)
    /// hiden code ......
    cs.evsw.FireEvent(types.EventVoteValue, vote)

    /// hiden code ......
}
```
这个方法前在的一长串代码中，最重要的应该就是这两行。`AddVote` 将投票信息保存到 `Votes` 结构体中；`FireEvent` 将投票信息通过 p2p 广播出去。

接下来的代码会根据投票数据的类型处不同的处理（在代码中，不管是 Prevote 投票还是 Precommit 投票都是同一个结构体，只不过它们的 `Type` 不一样）：
```go
func (cs *State) addVote(
	ctx context.Context,
	vote *types.Vote,
	peerID types.NodeID,
) (added bool, err error) {
    /// hiden code ......

	switch vote.Type {
	case tmproto.PrevoteType:
		prevotes := cs.Votes.Prevotes(vote.Round)
		if blockID, ok := prevotes.TwoThirdsMajority(); ok && !blockID.IsNil() {
            // ValidRound < vote.Round == cs.Round
			if cs.ValidRound < vote.Round && vote.Round == cs.Round {
				if cs.ProposalBlock.HashesTo(blockID.Hash) {
					cs.logger.Debug("updating valid block because of POL", "valid_round", cs.ValidRound, "pol_round", vote.Round)
					cs.ValidRound = vote.Round
					cs.ValidBlock = cs.ProposalBlock
					cs.ValidBlockParts = cs.ProposalBlockParts
				} 
                /// hide code ......
            }
            /// hide code ......
        }

	case tmproto.PrecommitType:
        /// hiden code ......
    }
}
```
由于我们目前的流程处在 Prevote 阶段，所以我们暂时只关心类型为 `PrevoteType` 的投票类型。这里取出 prevote 的投票信息，并判断区块的投票是否超过 2/3 。如果超过了并且 round 和 block 检查都正确的情况下，则会记录 `ValidRound` 和 `ValidBlock` 信息。这也是 `ValidRound` 和 `ValidBlock` 这两个变量的初始化来源。

我们继续看下面的代码：
```go
func (cs *State) addVote(
	ctx context.Context,
	vote *types.Vote,
	peerID types.NodeID,
) (added bool, err error) {
    /// hiden code ......

	switch vote.Type {
	case tmproto.PrevoteType:
        /// hiden code ......

		switch {
        /// 当前结点的轮数已经落后了，已经有 2/3 的人已经处于 vote.Round 轮了
        /// 所以我们直接调用 enterNewRound 进入到 vote.Round 轮。
		case cs.Round < vote.Round && prevotes.HasTwoThirdsAny():
			// Round-skip if there is any 2/3+ of votes ahead of us
			cs.enterNewRound(ctx, height, vote.Round)

        /// vote 中记录的 round 正是当前结点的当前轮，且正处于 Prevote 或这个阶段后面，
        /// 那就正常处理。
		case cs.Round == vote.Round && cstypes.RoundStepPrevote <= cs.Step: // current round
			blockID, ok := prevotes.TwoThirdsMajority()
			if ok && (cs.isProposalComplete() || blockID.IsNil()) {
				cs.enterPrecommit(ctx, height, vote.Round)
			} else if prevotes.HasTwoThirdsAny() {
				cs.enterPrevoteWait(height, vote.Round)
			}

        /// 当前结点已经有一个提案，但投票信息是 POLRound（即 ValidRound）轮的投票，这说明这个投票已经过时了，
        /// 所以这里忽略了这个投票，直接针对已有提案（尝试）进入 Prevote 阶段。
		case cs.Proposal != nil && 0 <= cs.Proposal.POLRound && cs.Proposal.POLRound == vote.Round:
			// If the proposal is now complete, enter prevote of cs.Round.
			if cs.isProposalComplete() {
				cs.enterPrevote(ctx, height, cs.Round)
			}
		}
	case tmproto.PrecommitType:
        /// hiden code ......
    }
}
```
上面有三个 switch 分支，我已经在代码里加入了自己的注释，所以这里就不细说了。三个分支里中间那个是最正常的情况，从代码里可以看到如果这个区块的 prevote 投票已经达到 2/3 （`TwoThirdsMajority`） 并且 Block 已经准备就绪，就直接调用 `enterPrecommit` 进入 Precommit 阶段；否则如果仅仅是有 2/3 的人投票但还没选出一个区块（`HasTwoThirdsAny`），那么就调用 `enterPrevoteWait` 再等一会。（关于 `TwoThirdsMajority` 和 `HasTwoThirdsAny` 的区别，后面我们会单独说一下）

到此，Prevote 阶段已经完成了，接下来就进入到 Precommit 阶段了。


#### Block 来自其它结点

前面介绍的 Prevote 阶段的流程一直是我们自己产生区块的情况。但作为一个 validator ，也需要处理从其它节点接收到 Proposal 和 Block 。与 Proposal 类似，新的 Block 也是从 `peerMsgQueue` 来的，同样也是 `reactor.go` 中的 `Reactor.handleDataMessage` 向这个 channel 发消息。

`peerMsgQueue` 收到 BlockPart 后，会调用 `handleMsg` 进行处理。代码会先将收到的区块信息保存，并检查这个块是否接收完整了，如果是则调用 `handleCompleteProposal` 进行处理。其实这块逻辑我们在前面介绍生成 Proposal 时已经介绍过了，这里就不多说了。


#### HasTowThirdsAny 和 TwoThirdsMajority 的区别

前面介绍的时候，遇到好几次关于类似于「超过 2/3 的投票」这样的表述，这个表述来自于 `HasTowThirdsAny`  和 `TwoThirdsMajority` 这两个方法，但它们是有区别的，这里单独说明一下。

这两个方法都返回 bool 值， `TwoThirdsMajority` 用来判断某一轮的投票中，是否有某个区块已经有至少 2/3 的人投过票； 而 `HasTowThirdsAny` 用来判断在某一轮中，是否有超过 2/3 的人已经投过票，注意这里并没有要求「某个区块」，因为可能虽然已经有 2/3 的人投票，但这些人可能把票投给了不同的区块，所以此时可能还没有一个区块有超过 2/3 的人给它投票。


### Precommit

现在流程进入到了 Precommit 阶段，这一阶段的实现是在 `enterPrecommit` ，我们来看一下它的实现：
```go
func (cs *State) enterPrecommit(ctx context.Context, height int64, round int32) {
    /// hiden code ......

    defer func() {
		// Done enterPrecommit:
		cs.updateRoundStep(round, cstypes.RoundStepPrecommit)
		cs.newStep()
	}()

	blockID, ok := cs.Votes.Prevotes(round).TwoThirdsMajority()

	if !ok {
		cs.signAddVote(ctx, tmproto.PrecommitType, nil, types.PartSetHeader{})
		return
	}

    /// hiden code ......
}
```
这个方法一开始也是做一些检查，这里略过了；然后还会设置一个 defer ，在方法退出时设置 step 为 `RoundStepPrecommit` 。接着就要检查在 Prevote 阶段是否真的有 2/3 投票给了某个区块，如果不是，那么就直接投票给 nil 。

我们继续往下看：
```go
func (cs *State) enterPrecommit(ctx context.Context, height int64, round int32) {
    /// hiden code ......

	if cs.LockedBlock.HashesTo(blockID.Hash) {
		cs.LockedRound = round

		cs.signAddVote(ctx, tmproto.PrecommitType, blockID.Hash, blockID.PartSetHeader)
		return
	}

	if cs.ProposalBlock.HashesTo(blockID.Hash) {
		// Validate the block.
		if err := cs.blockExec.ValidateBlock(ctx, cs.state, cs.ProposalBlock); err != nil {
			panic(fmt.Sprintf("precommit step: +2/3 prevoted for an invalid block %v; relocking", err))
		}

		cs.LockedRound = round
		cs.LockedBlock = cs.ProposalBlock
		cs.LockedBlockParts = cs.ProposalBlockParts

		cs.signAddVote(ctx, tmproto.PrecommitType, blockID.Hash, blockID.PartSetHeader)
		return
	}

    /// hiden code ......
```
这里 prevote 的投票情况检查通过了，将并投出一区块 hash 放在了 `blockID` 中。这里如果有一个 `LockedBlock` 等于当前轮投出来的 block ，那就直接投给这个区块并返回。否则的话，会检查当前处理的提案的区块是否等于 `blockID` ，正常情况应该是的，所以调用 `ValidateBlock` 检查区块成功后，就投票给这个区块了。

注意这里还设置了 `LockedRound` 、 `LockedBlock` 等这些 lock 系统的变量，之前我们也提到了这些变量，它们的值就是在这里设置的。这里的情况是 prevote 已经成功了，即多数人（2/3）认可了这个区块，所以把它们「锁定」了。

正常情况下程序是会运行到这个逻辑并返回的，后面的代码是出错情况，我们就不细看了。

`signAddVote` 我们在前面分析过，它会生成并签名一个投票数据，然后通过 `VoteMessage` 消息将投票数据发送给 `internalMsgQueue` ；而在这个 channel 的处理中，最终会调用 `addVote` ，这个方法我们之前也分析过了，但有一段代码即投票类型是 `PrecommitType` 时我们并没有分析过，现在正是来看看这段代码的时候：
```go
func (cs *State) addVote(
	ctx context.Context,
	vote *types.Vote,
	peerID types.NodeID,
) (added bool, err error) {
    /// hiden code ......

	switch vote.Type {
	case tmproto.PrevoteType:
        /// hiden code ......

    case tmproto.PrecommitType:
		precommits := cs.Votes.Precommits(vote.Round)
		blockID, ok := precommits.TwoThirdsMajority()
		if ok {
			// Executed as TwoThirdsMajority could be from a higher round
			cs.enterNewRound(ctx, height, vote.Round)
			cs.enterPrecommit(ctx, height, vote.Round)

			if !blockID.IsNil() {
				cs.enterCommit(ctx, height, vote.Round)
				if cs.bypassCommitTimeout() && precommits.HasAll() {
					cs.enterNewRound(ctx, cs.Height, 0)
				}
			} else {
				cs.enterPrecommitWait(height, vote.Round)
			}
		} else if cs.Round <= vote.Round && precommits.HasTwoThirdsAny() {
			cs.enterNewRound(ctx, height, vote.Round)
			cs.enterPrecommitWait(height, vote.Round)
		}
}
```
这段代码的逻辑其实也不复杂，首先就是检查 precommit 的投票是否超过 2/3 ，如果是的话并且 `blockID` 有效，就调用 `enterCommit` 正式提交区块了；如果不是，并且当前结点的 round 并不落后且已经有 2/3 的人投过票了（`cs.Round <= vote.Round && precommits.HasTwoThirdsAny()`），那就调用 `enterPrecommitWait` 再等等看。

你可能会奇怪上面的代码为什么调用了好几次 `enterNewRound` 。我觉得可能是为了「保险起见」吧，比如在 `if ok` 后面接着调用 `enterNewRound` 但使用的参数是 `vote.Round` ，其潜在意义是说 `vote.Round` 可能比当前正在进行的轮（`cs.Round`）大（即当前结点已经落后了），所以直接尝试进入到 `vote.Round` 轮。在 `enterNewRound` 中一开始就有一个判断：
```go
func (cs *State) enterNewRound(ctx context.Context, height int64, round int32) {
	if cs.Height != height || round < cs.Round || (cs.Round == round && cs.Step != cstypes.RoundStepNewHeight) {
		logger.Debug("entering new round with invalid args",
			"height", cs.Height,
			"round", cs.Round,
			"step", cs.Step)
		return
	}

    /// hiden code ......
}
```
目前的步骤显然不是 `RoundStepNewHeight`，所以从这个判断可以看出来，如果当前结点确实落后了（ `vote.Round` > `cs.Round` ）那就正常开始新的一轮的流程；否则就直接返回了，不会真的开始新的一轮了。


### commit 

之前的分析已经调用 `enterCommit` 进入到 commit 阶段了。由于 commit 不是共识算法的关键，所以就简单看一下行了。

`enterCommit` 这个方法其实也不复杂，主要就是检查一下 precommit 的投票，如果没问题，就调用 `tryFinalizeCommit` ，最终调用到了 `finalizeCommit` 中：
```go
func (cs *State) finalizeCommit(ctx context.Context, height int64) {
    /// hiden code ......

	if cs.blockStore.Height() < block.Height {
		// NOTE: the seenCommit is local justification to commit this block,
		// but may differ from the LastCommit included in the next block
		seenExtendedCommit := cs.Votes.Precommits(cs.CommitRound).MakeExtendedCommit()
		if cs.state.ConsensusParams.ABCI.VoteExtensionsEnabled(block.Height) {
			cs.blockStore.SaveBlockWithExtendedCommit(block, blockParts, seenExtendedCommit)
		} else {
			cs.blockStore.SaveBlock(block, blockParts, seenExtendedCommit.ToCommit())
		}
    }

    /// hiden code ......

    stateCopy, err := cs.blockExec.ApplyBlock(ctx,
		stateCopy,
		types.BlockID{
			Hash:          block.Hash(),
			PartSetHeader: blockParts.Header(),
		},
		block,
	)

    /// hiden code ......

    cs.updateToState(stateCopy)

	cs.scheduleRound0(&cs.RoundState)
}
```
这个方法主要是先将 Block 保存起来，然后调用 ABCI 接口将这个 Block 通知 App 。最后调用 `updateToState` 和 `scheduleRound0` 开启新的一轮。这两个方法前面都介绍过了，这里就不再说了。

至此，整个过程的正常流程就串起来了。


## 失败了怎么办

前面我们关注的都是正常情况下的流程，但不正常也是很常见的，比如网络不通导致的收不到其它结点的 Proposal 信息和投票信息等数据，再比如结点收到的 Proposal 或投票信息的高度或 round 与自己的不符。这一小节里，我们就看看代码里是怎么应对不正常现象的。

其实我们可以自己先想一下有可能会面对哪些出错的地方，我觉得总得来说两种：网络出问题、结点作恶（主动或被动）。网络出问题会导致数据收发不畅，进而影响共识的达成；结点作恶会导致数据不是正常结点想要的。 Tendermint 代码中对这两个问题的解决方法也很直观，就是使用超时机制，和严谨的数据检查。

下面我们就分别看看代码中是如何实现的。不过有一点我觉得需要特别说明一下，就是在 Tendermint 中，当发现某一轮中数据不正确时，可能并不会发起新的一轮，而是尝试投票给 nil 对象（空对象）（不管是 prevote 还是 precommit），最终可能会形成的结果就是有 2/3 的人投票给 nil 对象，然后再开启下一轮。


### 数据检查

数据检查总得来说我觉得包括三方面：
1. 数据完整性
2. 签名检查
3. 数据有效性

前两个我觉得其实就不用看代码的，这个都很容易解决：数据完整性靠 hash 校验，签名检查也是现成机制。我们就重点说一下「数据有效性」。

这里的「数据有效性」是在数据完整、签名没问题的情况下，这个数据是不是一个正常结点想要的。这里的数据可以分成三种：
1. Proposal 数据
2. Block 数据
3. Vote 数据

下面我们分别来看一下代码是如何检查这些数据的。但总得来说，我们可以总结出以下数据检查的点：
- 高度是否与当前结点相同
- round 是否与当前结点相同
- 是否是一个真正的 validator 发送的数据
- 签名是否有效


#### Proposal 数据 

当结点从别的结点处收到 Proposal 数据时，会调用 `defaultSetProposal` 对其进行处理。我们看看这个方法对 Proposal 的检查：
```go
func (cs *State) defaultSetProposal(proposal *types.Proposal, recvTime time.Time) error {
	// Already have one
	// TODO: possibly catch double proposals
	if cs.Proposal != nil || proposal == nil {
		return nil
	}

	// Does not apply
	if proposal.Height != cs.Height || proposal.Round != cs.Round {
		return nil
	}

	// Verify POLRound, which must be -1 or in range [0, proposal.Round).
	if proposal.POLRound < -1 ||
		(proposal.POLRound >= 0 && proposal.POLRound >= proposal.Round) {
		return ErrInvalidProposalPOLRound
	}

	p := proposal.ToProto()
	// Verify signature
	if !cs.Validators.GetProposer().PubKey.VerifySignature(
		types.ProposalSignBytes(cs.state.ChainID, p), proposal.Signature,
	) {
		return ErrInvalidProposalSignature
	}

    /// hiden code ......
}
```
从上面的代码可以看出来，首先会检查是否已经有一个 Proposal 了，如果已经有一个了，就不处理新收到的这个。从 TODO 注释里也能看出来，其实收到多个 Proposal 的情况也是很可能的。那这种只是简单的拒绝第一个以后的处理会不会有问题呢？比如可能已经收到的这个 Proposal 可能是恶意的，反而被拒绝的那个才是正常的？有这个可能，但不会有问题，因为即使已经收到的是恶意的，那这个结点肯定无法达成共识，最终结点自己会进入到下一轮重新处理。

后面代码还会检查 Proposal 的高度和 round 是否和结点自己当前正在进行中的一致，如果不一致也是直接拒绝。高度这个好理解，为什么 round 不一致也要拒绝呢？我想这是为了防止某些恶意结点在出块时，直接把 round 上升到可以让自己出块的值（我们前面说过，在高度不变的情况下，每进行一轮就会变换 Proposer，并且只要成功了下次还是这个人出块），从而绕过 Voting Power 的限制。

不过这么处理的一个问题就是，如果当前结点因为某些原因，高度或 round 确实落后了，那就会真的会一直拒绝正常的 Proposal 。（这个我确实还不确定，暂时也没时间去调试，等有时间再确定吧......）

接下来的判断比较简单，就是判断 POLRound 是否合法，这里就说了。最后是检查签名，也就是说这个 Proposal 必须得是本地认为的那个 Proposer 正确签名才行。由于前面有 round 的检查，而 Proposer 又与 round 有关（高度不变的情况下，round 变化才会变更 Proposer），所以正常情况下此 Proposal 的签名确实应该是本地认为的那个 Proposer 。

以上就是对整个接收到的 Proposal 的检查。总得来说，结点不会让随便接收别人发的某高度、某轮的 Proposal ，这些信息必须本结点当前的值一致才行。


#### Block 数据

首先要说明一下的是，Tendermint 中传输 Block 的方式是把 Block 切成多块小的数据，对这些小的数据进行传输，然后结点再把这些接收到的小块数据接拼成一个完整的 Block ，所以代码中有 「BlockPart」 这个概念，就是指的 「Block 的一部分」。

当结点从别的结点处收到 Proposal 数据时，会调用 `addProposalBlockPart` 对其进行处理。我们看看这个方法都作了哪些检查：
```go
func (cs *State) addProposalBlockPart(
	msg *BlockPartMessage,
	peerID types.NodeID,
) (added bool, err error) {
	height, round, part := msg.Height, msg.Round, msg.Part

	// Blocks might be reused, so round mismatch is OK
	if cs.Height != height {
		return false, nil
	}

	// We're not expecting a block part.
	if cs.ProposalBlockParts == nil {
		return false, nil
	}

	added, err = cs.ProposalBlockParts.AddPart(part)

    /// hiden code ......
```
这里首先确认区块数据的高度与当前结点高度一致；然后又判断了 `cs.ProposalBlockParts` 是否为空，其实接收到的 BlockPart 是要放到这里面的（即接下来调用 `cs.ProposalBlockParts.AddPart`），它为空当然没法放进去；这里判断是否为空还有另外一层意思：在接收到 Proposal 进行处理时，如果 Proposal 被成功接受，就会初始化 `cs.ProposalBlockParts` 这个字段，准备接收 BlockPart，如果这个值为空，代表我们没有接收到一个有效的 Proposal ，当然也没准备要接收 BlockPart：
```go
func (cs *State) defaultSetProposal(proposal *types.Proposal, recvTime time.Time) error {
    /// hiden code ......

	if cs.ProposalBlockParts == nil {
		cs.ProposalBlockParts = types.NewPartSetFromHeader(proposal.BlockID.PartSetHeader)
	}
}
```

确认完 `cs.ProposalBlockParts` 后，接下来就是调用它的 `AddPart` 方法将新的 BlockPart 存进去了。`AddPart` 也有一些判断，但比较简单，这里就不多说了。


#### Vote 数据

当结点从别的结点处收到 Proposal 数据时，会调用 `tryAddVote` 对其进行处理，它会调用 `addVote`，我们就从 `addVote` 入手看一下：
```go
func (cs *State) addVote(
	ctx context.Context,
	vote *types.Vote,
	peerID types.NodeID,
) (added bool, err error) {
    /// hiden code ......
    
	if vote.Height != cs.Height {
		return
	}
    /// hiden code ......
	added, err = cs.Votes.AddVote(vote, peerID)

    /// hiden code ......
}
```
在这个方法里，只是检查了一下高度是否匹配，就调有 `AddVote` 了，这个方法其实最终会调用到 `VoteSet.addVote` 。我们再来看看 `VoteSet.addVote` 有哪些判断：
```go
func (voteSet *VoteSet) addVote(vote *Vote) (added bool, err error) {
	if vote == nil {
		return false, ErrVoteNil
	}
	valIndex := vote.ValidatorIndex
	valAddr := vote.ValidatorAddress
	blockKey := vote.BlockID.Key()

	// Ensure that validator index was set
	if valIndex < 0 {
		return false, fmt.Errorf("index < 0: %w", ErrVoteInvalidValidatorIndex)
	} else if len(valAddr) == 0 {
		return false, fmt.Errorf("empty address: %w", ErrVoteInvalidValidatorAddress)
	}

    /// hiden code ......
    /// 这里有一段关于 round 等数据的判断，但因为本对象就是使用这些数据初始化的，
    /// 这里判断只是为了程序的正确性，跟我们分析代码的目的不相关，所以就隐去了。

	// Ensure that signer is a validator.
	lookupAddr, val := voteSet.valSet.GetByIndex(valIndex)
	if val == nil {
		return false, ...
	}

	// Ensure that the signer has the right address.
	if !bytes.Equal(valAddr, lookupAddr) {
		return false, ...
	}

	// If we already know of this vote, return false.
	if existing, ok := voteSet.getVote(valIndex, blockKey); ok {
		if bytes.Equal(existing.Signature, vote.Signature) {
			return false, nil // duplicate
		}
		return false, fmt.Errorf("existing vote: %v; new vote: %v: %w", existing, vote, ErrVoteNonDeterministicSignature)
	}

	// Check signature.
    if err := vote.Verify(voteSet.chainID, val.PubKey); err != nil {
        return false, fmt.Errorf("failed to verify vote with ChainID %s and PubKey %s: %w", voteSet.chainID, val.PubKey, err)
    }

    /// hiden code ......
}
````
这里对 vote 信息的检查很多，主要是检查投票的人是否是合法的 validator 、签名是否正确等。


### 超时：预防失败的手段

如果因为网络问题或其它原因，一件原本该发生的事迟迟没有发生，那就会出现问题，就要想办法去处理这个问题了。超时机制就是这种问题的处理方案。

在 Tendermint 共识的代码中，处理超时的方法是 `State.handleTimeout` ，从它的代码中可以看出来，它处理的超时类型有以下几种： 
- `RoundStepNewHeight` 
- `RoundStepNewRound`
- `RoundStepPropose`
- `RoundStepPrevoteWait`
- `RoundStepPrecommitWait`

下面我们分别对这几种超时类型作一个说明。


#### `RoundStepNewHeight` 

`RoundStepNewHeight` 类型的意思是说：「是时候开始新的高度了」。这一类型只有在 `State.scheduleRound0` 方法中被使用，而这个方法的目的也很简单，正是为新的高度开启第 0 轮共识过程。


#### `RoundStepNewRound`

`RoundStepNewRound` 类型的意思是说：「是时候开始真的开启共识了」。这个类型的名字我感觉挺有歧义的，名字有 「NewRound」 ，但开始的永远是 round 0 ，因为处理这个消息的代码是这样的：
```go
	case cstypes.RoundStepNewRound:
		cs.enterPropose(ctx, ti.Height, 0)
```
这里 `enterPropose` 参 3 个参数就是 round 值，永远是 0 。

其实这个类型的超时只有在两种情况会被设置，一是在 `enterNewRound` 中，并且是第 0 轮，如果配置里配置的是 Proposal 时如果没有 transaction 就等一会，就会设置这个超时：
```go
func (cs *State) enterNewRound(ctx context.Context, height int64, round int32) {
    /// hiden code
	waitForTxs := cs.config.WaitForTxs() && round == 0 && !cs.needProofBlock(height)
	if waitForTxs {
		if cs.config.CreateEmptyBlocksInterval > 0 {
			cs.scheduleTimeout(cs.config.CreateEmptyBlocksInterval, height, round,
				cstypes.RoundStepNewRound)
		}
		return
	}
    /// hiden code
}
```

另一种情况是在 `enterPropose` 中，如果当前结点是一个 Proposer 但最新的区块的时间比本地时间还是晚，就设置这个超时类型，等一会再进行提案：
```go
func (cs *State) enterPropose(ctx context.Context, height int64, round int32) {
    /// hiden code ......

	if cs.privValidatorPubKey != nil && cs.isProposer(cs.privValidatorPubKey.Address()) {
		proposerWaitTime := proposerWaitTime(tmtime.DefaultSource{}, cs.state.LastBlockTime)
		if proposerWaitTime > 0 {
			cs.scheduleTimeout(proposerWaitTime, height, round, cstypes.RoundStepNewRound)
			return
		}
	}

    /// hiden code ......
}
```

可见上面无论哪种情况，前提都是要提案了，只不要要等一会而已，所以我把这个类型的意义解释为 「是时候开始真的开启共识了」 。

（但上面第二种情况，在处理此超时信息时，显然不应该用 0 调用 `enterPropose`，这觉得这是一个 bug ，已经经 tendermint 提了一个 [issue](https://github.com/tendermint/tendermint/issues/9229) , 截止文章发出来前，还没有回应）。


#### `RoundStepPropose`

`RoundStepPropose` 类型的意思是说：「是时候进入到 Prevote 阶段了」。这个类型只有在一个地方会被设置，即当前结点开始了新的一轮、开始提案了，会设置这个超时：
```go
func (cs *State) enterPropose(ctx context.Context, height int64, round int32) {
    /// hiden code ......

	// If we don't get the proposal and all block parts quick enough, enterPrevote
	cs.scheduleTimeout(cs.proposeTimeout(round), height, round, cstypes.RoundStepPropose)

    /// hiden code ......
}
```
注意上面的代码中设置这个超时的时候，新的提案还没有生成呢，结点也还没确定自己是不是当前的 Proposer。也就是说只要开始了提案，管它成功没成功，到了时间就得开始 Prevote 阶段（如果 Proposal 不成功，那就投票给 nil）。


#### `RoundStepPrevoteWait`

`RoundStepPrevoteWait` 类型的意思是说：「等待 Prevote 的时间已经够长了，是时候进入到 Precommit 了」。

这个类型只有在一个地方会被设置，即在处理 prevote 投票时，如果发现已经有 2/3 的人投票、但还没选出一个区块来时（`HasTwoThirdsAny`），就设置此超时，再等等看，说不定过会就先出来了呢：
```go
func (cs *State) addVote(
	ctx context.Context,
	vote *types.Vote,
	peerID types.NodeID,
) (added bool, err error) {
    /// hiden code ......

			blockID, ok := prevotes.TwoThirdsMajority()
			if ok && (cs.isProposalComplete() || blockID.IsNil()) {
                /// hiden code ......
			} else if prevotes.HasTwoThirdsAny() {
				cs.enterPrevoteWait(height, vote.Round)
			}

    /// hiden code ......
```

注意这里的实现其实有另外一个隐含意义：如果一票也没收到过，或者收到的票数不足 2/3 ，那么当前结点就一直处理 Prevote 阶段。


#### `RoundStepPrecommitWait`

`RoundStepPrecommitWait` 类型的意思是说：「等待 Precommit 阶段的时间已经够长了」。在处理这个类型的时候，会先调用 `enterPrecommit` 进行 Precommit 阶段的处理，无论成功与否都会调用 `enterNewRound` 尝试进入新的一轮（round + 1）：
```go
	case cstypes.RoundStepPrecommitWait:
		if err := cs.eventBus.PublishEventTimeoutWait(cs.RoundStateEvent()); err != nil {
			cs.logger.Error("failed publishing timeout wait", "err", err)
		}

		cs.enterPrecommit(ctx, ti.Height, ti.Round)
		cs.enterNewRound(ctx, ti.Height, ti.Round+1)
```
因为 `enterNewRound` 在开始的地方有检查，如果 `enterPrecommit` 成功，那么高度和 round 都已经变了，`enterNewRound` 就会无功而返；否则就真的进行新的一轮了：
```go
func (cs *State) enterNewRound(ctx context.Context, height int64, round int32) {
	if cs.Height != height || round < cs.Round || (cs.Round == round && cs.Step != cstypes.RoundStepNewHeight) {
		return
	}

    /// hiden code ......
}
```

有两个地方会设置这个超时，都是在处理 precommit 投票时设置的。一个是虽然已经有 2/3 的人投票选出了一个区块，但区块是 nil ，所以设置超时再等一下（有必要吗？）； 另一个是虽然有 2/3 的人投票了，但还没选出一个区块来，这时再等会，说不定过会就选出来了：
```go
func (cs *State) addVote(
	ctx context.Context,
	vote *types.Vote,
	peerID types.NodeID,
) (added bool, err error) {
    /// hiden code ......

		blockID, ok := precommits.TwoThirdsMajority()
		if ok {
			// Executed as TwoThirdsMajority could be from a higher round
			cs.enterNewRound(ctx, height, vote.Round)
			cs.enterPrecommit(ctx, height, vote.Round)

			if !blockID.IsNil() {
				cs.enterCommit(ctx, height, vote.Round)
				if cs.bypassCommitTimeout() && precommits.HasAll() {
					cs.enterNewRound(ctx, cs.Height, 0)
				}
			} else {
				cs.enterPrecommitWait(height, vote.Round)
			}
		} else if cs.Round <= vote.Round && precommits.HasTwoThirdsAny() {
			cs.enterNewRound(ctx, height, vote.Round)
			cs.enterPrecommitWait(height, vote.Round)
		}
```


# 总结

这篇文章介绍了 Tendermint 中的共识算法，详细分析了其代码实现。Tendermint 共识是拜占庭容错算法的一种，但它抛掉了之前 [PBFT](https://yangzhe.me/2019/11/25/pbft) 中检查点的概念，也简化了失败时更换出块结点的方法，使运行时效率更高、占用网络资源更少。

在分析代码的过程中，我觉得有些代码其实还是可以改进的（至少我自己感觉是这样的），比如 `State.mtx` 锁的使用粒度感觉有点大，可能还有优化空间；再比如如果是一些非正常情况的判断和处理，我习惯把这种处理放在 if 分支内部，处理完直接返回，例如：
```go
if some_thing_wrong {
    handle wrong thing 
    return
}

handle normal thing
```
但在 Tendermint 中很多代码不是这样的，如此一来在阅读代码时，就得小心关注每一个分支，因为正常流程可能在任何一个代码分支里。

不过总得来说 Tendermint 还是挺棒的，源代码的组织也比较清晰，作为一个很流行的库还是很值得学习的。

最后限于作者水平，文章中难免有错误的地方，如果发现感谢您能不吝指正。