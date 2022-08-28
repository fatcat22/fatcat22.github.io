---
layout: post
title:  "关于 Beacon Chain 的那些事"
categories: blockchain
tags: 原创 eth2.0 BeaconChain 
excerpt: Beacon Chain 是以太坊 2.0 用于替换 PoW 共识的链，它实现了 PoS 共识，那么这个链与其它 PoS 共识的实现有什么不同的地方呢？
author: fatcat22
---

* content
{:toc}




> 在前面的文章里，我们通过简单介绍 eth2.0 ，了解了以太坊将要做哪些改进和升级。Beacon Chain 作为 eth2.0 已经迈出的第一步，有非常大的意义，也是后继改进和升级的关键。这篇文章里，我们就来了解一下 Beacon Chain 。
> 在阅读这篇文章之前，你需要对 PoS 共识和拜占庭容错算法（BFT）有所了解。如果不了解，可以先了解一下，或翻阅一下我之前写的文章。

# 什么是 Beacon Chain

其实我们在[介绍 eth2.0 ](https://yangzhe.me/2022/08/23/eth2-brief/)时已经提到了 Beacon Chain 的概念，很简单，[Beacon Chain](https://ethereum.org/en/upgrades/beacon-chain) 是一个独立的区块链，它使用 PoS 共识出块，首要目的就是为了替换以太坊当前的 PoW 共识。

在很多文章里，都会把 Beacon Chain 叫做以太坊的共识层（consensus layer，为了达成共识）；而把原以太坊称为执行层（execution layer，执行交易、合约等）。所以虽说是一条「独立的链」，但毕竟 Beacon Chain 是以太坊为了实现共识替换创建出来的，所以你要说它独立，其实也不完全准确，它有一些与以太坊相关的概念，比如它会通过 [Engine API](https://github.com/ethereum/execution-apis/blob/main/src/engine/specification.md) 与执行层通信；最终它的链也会与执行层的链合并，两者成为一条链。

Beacon Chain 并没有以太坊官方的实现，而只是出了一个[实现手册](https://github.com/ethereum/consensus-specs/blob/v1.2.0-rc.3/specs/phase0/beacon-chain.md)（为了简单起见，后面我们称它为「 Beacon 手册」），在这份手册里，会有一些 Python 写的示意代码；其他的团队通过这份手册实现了一些[客户端](https://ethereum.org/en/developers/docs/nodes-and-clients/client-diversity/#current-client-diversity)，通过 [clientdiversity](https://clientdiversity.org/) 的统计可以看出，Beacon Chain 的实现中 `Prysm` 和 `Lighthouse` 这俩的占有率是比较高的，它俩分别是用 Go 和 Rust 实现的。我非常喜欢 Rust 这门语言，所以在后面的分析中，多数情况会使用手册中的 Python 代码，偶尔会使用 `Lighthouse` 中的 Rust 代码段。（其实在手册的[目录](https://github.com/ethereum/consensus-specs/tree/v1.2.0-rc.3/specs/phase0)中，其它的文档也值得一读的，比如 validator.md）


# Beacon Chain 中的概念

Beacon Chain 目前（2022年8月）只是为了替换以太坊共识而存在的，它的首要目的是实现 PoS 共识，而实现的方式，依然是使用了某种拜占庭容错（BFT）算法。关于 BFT 我们已经讲了不少了，所以这篇文章里我们不再去详细分析它的共识算法，而是重点说明一下它的一些与众不同的概念。

## 质押

Beacon Chain 使用 PoS 共识出块，众所周知，PoS 是需要 validator 的。那么怎么能成为 validator 呢？别忘了它是为了以太坊而创建的链，所以要想成为 Beacon Chain 的 validator ，就需要向以太坊质押一定数量的 ETH 币。

具体来说，要想成为 Validator ，你需要向智能合约 [0x00000000219ab540356cBB839Cbe05303d7705Fa](https://etherscan.io/address/0x00000000219ab540356cbb839cbe05303d7705fa) 质押一定数量的币：至少 1 ETH 并且是 1 gwei 的整数倍。但只有满 32 ETH 时，才有资格成为 validator （也就是说虽然你可以质押 1 ETH ，但没什么用）。质押的越多，你就拥有越多的 validator ：比如你质押了 320 个 ETH ，那你就拥有了 10 个 validator 。

 注意：**这里我不对上面提到的合约地址的正确性负责。如果你真的要进行质押，请多方获取并对比质押的合约地址，不要只看我这里写的，因为这里的地址可能被恶意篡改**。当然，如果真的是成为 validator ，你不应该直接向刚才提到的智能合约转账，而应该按照 [launchpad](https://launchpad.ethereum.org) 的指引去操作。这个页面会引导你质押 32 ETH 的整数倍的币；在质押完成后，还会有如何部署的链接。由于我们的重点不是如何质押和部署，这里就不详细说了。

质押后的 ETH 是无法取回的，直到 The Merge 后的 [Shanghai](https://github.com/ethereum/pm/issues/450) 升级后才行。 [官网](https://ethereum.org/en/upgrades/merge/#merge-and-shanghai)有对这个问题的说明。 

另外需要提一下的是质押的安全问题。由于 validator 运行时需要持有私钥，以便在出块或投票时签名；同时又需要联网，以便和其它结点交流数据。因此有人可能会担心运行 validator 的结点被攻击后，盗走私钥将质押和赚到的 ETH 据为已有。这个担心是没必要的，因为在质押时需要指定一个退款的账户，无论是不是拥有私钥，将来退款时都会退到这个当初指定的账户。

下面我们从技术角度去看如何成为一个 validator 。通过 [validator 手册](https://github.com/ethereum/consensus-specs/blob/v1.2.0-rc.3/specs/phase0/validator.md#becoming-a-validator)和[智能合约的代码](https://github.com/ethereum/consensus-specs/blob/v1.2.0-rc.3/solidity_deposit_contract/deposit_contract.sol)可以看出来，每次质押都要要求至少 `MIN_DEPOSIT_AMOUNT` 个币：
```solidity
function deposit(
        bytes calldata pubkey,
        bytes calldata withdrawal_credentials,
        bytes calldata signature,
        bytes32 deposit_data_root
    ) override external payable {
        // hiden code ......

        // Check deposit amount
        require(msg.value >= 1 ether, "DepositContract: deposit value too low");
        require(msg.value % 1 gwei == 0, "DepositContract: deposit value not multiple of gwei");

        // hiden code ......
}
```
`MIN_DEPOSIT_AMOUNT` 这个值在 Beacon 手册中有说明，它的值正是 1_000_000_000 gwei。

另外从[合约手册](https://github.com/ethereum/consensus-specs/blob/v1.2.0-rc.3/specs/phase0/deposit-contract.md#depositevent-log) 和 合约代码中都可以看到，合约会通过 `DepositEvent` log 将数据提交：
```solidity
    function deposit(
        bytes calldata pubkey,
        bytes calldata withdrawal_credentials,
        bytes calldata signature,
        bytes32 deposit_data_root
    ) override external payable {
    // hiden code ......
    emit DepositEvent(
            pubkey,
            withdrawal_credentials,
            amount,
            signature,
            to_little_endian_64(uint64(deposit_count))
        );
    // hiden code ......
}
```
提交成功后，Beacon Chain 就可以拿到数据并对其进行处理：
```Rust
    impl Log {
        /// Attempts to parse a raw `Log` from the deposit contract into a `DepositLog`.
        pub fn to_deposit_log(&self, spec: &ChainSpec) -> Result<DepositLog, String> {
            /// 为了使代码更方便阅读，我删减了一些代码
            let bytes = &self.data;

            let pubkey = bytes
                .get(PUBKEY_START..PUBKEY_START + PUBKEY_LEN);
            let withdrawal_credentials = bytes
                .get(CREDS_START..CREDS_START + CREDS_LEN);
            let amount = bytes
                .get(AMOUNT_START..AMOUNT_START + AMOUNT_LEN);
            let signature = bytes
                .get(SIG_START..SIG_START + SIG_LEN);
            let index = bytes
                .get(INDEX_START..INDEX_START + INDEX_LEN);

            let deposit_data = DepositData {
                pubkey: PublicKeyBytes::from_ssz_bytes(pubkey),
                withdrawal_credentials: Hash256::from_ssz_bytes(withdrawal_credentials),
                amount: u64::from_ssz_bytes(amount),
                signature: SignatureBytes::from_ssz_bytes(signature),
            };

            /// hiden code ......
            Ok(DepositLog {
                deposit_data,
                block_number: self.block_number,
                index: u64::from_ssz_bytes(index),
                signature_is_valid,
            })
        }
    }
```
Beacon Chain 拿到 log 数据后，会处理其中的质押数据：
```Rust
pub fn process_deposit<T: EthSpec>(
    state: &mut BeaconState<T>,
    deposit: &Deposit,
    spec: &ChainSpec,
    verify_merkle_proof: bool,
) -> Result<(), BlockProcessingError> {
    /// hiden code ......

        let validator = Validator {
            pubkey: deposit.data.pubkey,
            withdrawal_credentials: deposit.data.withdrawal_credentials,
            activation_eligibility_epoch: spec.far_future_epoch,
            activation_epoch: spec.far_future_epoch,
            exit_epoch: spec.far_future_epoch,
            withdrawable_epoch: spec.far_future_epoch,
            effective_balance: std::cmp::min(
                amount.safe_sub(amount.safe_rem(spec.effective_balance_increment)?)?,
                spec.max_effective_balance,
            ),
            slashed: false,
        };
        state.validators_mut().push(validator)?;

    /// hiden code ......
}
```
上面的 `Validator` 结构其实就是 Beacon 手册中 `Validator` 的定义：
```Python
class Validator(Container):
    pubkey: BLSPubkey
    withdrawal_credentials: Bytes32  # Commitment to pubkey for withdrawals
    effective_balance: Gwei  # Balance at stake
    slashed: boolean
    # Status epochs
    activation_eligibility_epoch: Epoch  # When criteria for activation were met
    activation_epoch: Epoch
    exit_epoch: Epoch
    withdrawable_epoch: Epoch  # When validator can withdraw funds
```

其中有几个字段需要特别注意一下，一个是 `effective_balance` ，这个字段记录的是 validator 在整个过程中的 `balance` 数量（包括质押的和后来奖励的），上面的代码对其进行了初始化：
```Rust
            effective_balance: std::cmp::min(
                amount.safe_sub(amount.safe_rem(spec.effective_balance_increment)?)?,
                spec.max_effective_balance,
```
可见这个值最大就是 `spec.max_effective_balance` ，即 Beacon 手册中的 `MAX_EFFECTIVE_BALANCE` （32 ETH）；如果达不到这个值，则会将质押值按 `spec.effective_balance_increment` 取整，这个值是手册中的 `EFFECTIVE_BALANCE_INCREMENT` ，即 1 个 ETH 。

另一个需要注意的字段是 `activation_epoch` ，它记录了 validator 何时「正式工作」。上面初始化时它们都被初始化成 `spec.far_future_epoch` ，即 Beacon 手册中的 `FAR_FUTURE_EPOCH` ，它其实代表的是一个无效值，类似于编程语言中的 NULL 。也就是说，在创建一个 validator 对象后，它还不能开始出块。

上面的代码算是打通了从质押到成为 validator 的步骤。但此时 TA 还不能参与出块，一个原因是质押的币未必符合要求。合约要求只要质押数量大于 1 ETH 就可以，而要成为一个有效的 validator 则需要质押 `MAX_EFFECTIVE_BALANCE` 即 32  ETH：
```Python
# 此函数在 lighthouse 中有同名函数，实现也是类似的
def is_eligible_for_activation_queue(validator: Validator) -> bool:
    """
    Check if ``validator`` is eligible to be placed into the activation queue.
    """
    return (
        validator.activation_eligibility_epoch == FAR_FUTURE_EPOCH
        and validator.effective_balance == MAX_EFFECTIVE_BALANCE
    )
```
这个函数用于判断是不是可以「激活」一个 validator ，从实现可以看出来，需要满足 `validator.effective_balance == MAX_EFFECTIVE_BALANCE` 。

另一个原因是我们刚才也提到了，代表「何时可以出块」的 `activation_epoch` 的值无效。那它的值应该是什么呢？
```Python
# 此函数在 lighthouse 中有同名函数，实现也是类似的
def compute_activation_exit_epoch(epoch: Epoch) -> Epoch:
    """
    Return the epoch during which validator activations and exits initiated in ``epoch`` take effect.
    """
    return Epoch(epoch + 1 + MAX_SEED_LOOKAHEAD)
```
这个函数用于计算 validator 「正式参加工作」或「退出」的 epoch （关于 epoch 的概念，后面会详细聊，这里只要知道它代表一个出块的时间周期就行了，比如拿现实举例子，当前的 epoch 是 2022 年，下一个 epoch 是 2023 年）。

注意无论是加入还是退出，都是调用 `compute_activation_exit_epoch` 获取正式加入或正式退出的 epoch 值。在计算时除了在当前 epoch 基础上加 1 ，还加了一个值 `MAX_SEED_LOOKAHEAD` 。加 1 好理解，下一个 epoch 生效嘛，再加个 `MAX_SEED_LOOKAHEAD` 是为什么呢？

其实我也没太想明白，这个 [issue](https://github.com/ethereum/consensus-specs/issues/322) 里提到了加这个是为了延迟踢出，可以防止操纵 shuffle；[这篇文章](https://eth2book.info/altair/part3/config/preset#max_seed_lookahead)里解释得稍微清楚一些，为了防止链接失效，我把关键信息复制在这：
> The above notwithstanding, if an attacker has a large proportion of the stake, or is, for example, able to DoS block proposers for a while, then it might be possible for the attacker to predict the output of the RANDAO further ahead than MIN_SEED_LOOKAHEAD would normally allow. This might enable the attacker to manipulate committee memberships to their advantage by performing well-timed exits and activations of their validators.

>To prevent this, we assume a maximum feasible lookahead that an attacker might achieve (MAX_SEED_LOOKAHEAD) and delay all activations and exits by this amount, which allows new randomness to come in via block proposals from honest validators. With MAX_SEED_LOOKAHEAD set to 4, if only 10% of validators are online and honest, then the chance that an attacker can succeed in forecasting the seed beyond (MAX_SEED_LOOKAHEAD - MIN_SEED_LOOKAHEAD) = 3 epochs is 0.9^{3\times 32}0.9 
3×32, which is about 1 in 25,000.

这一段里开头提到的 `The above` 是对 `MIN_SEED_LOOKAHEAD` 的解释（在后面解释 `RANDAO` 时会出现这个值，它的意思是取 seed 时，向前（lookahead）多少个 epoch 取值）。上面大体的意思是，让新加入和退出的 validator 都再多延迟 `MAX_SEED_LOOKAHEAD` 个 epoch ，这样可以大大降低对 `RANDAO` 的预测（操控）的可能性，从而增加攻击难度。

这里不可避免的涉及了 `RANDAO` 这个概念，后面我们会详细介绍。这里需要知道的是，它是一种「随机数」产生的方式，用于「随机分配」 validator 进行出块和验证投票（即刚才两个链接里提到的 shuffle ）；它的机制是依赖于新产生的区块数据：每产生一个区块，就用区块数据更新 `RANDAO` 的核心值。因此，想要预测或操控 shuffle ，就是预测或操控 `RANDAO` 的核心值，也就是预测或操控区块的产生。既然能预测和操控区块的产生了，还是用得着去管 shuffle 吗？

不过上面复制的内容里提到的使用 DDoS 方式，倒是可以操控区块的产生，进而操控 shuffle 。比如在一个 epoch 内，让其它 Proposer 都出不了块，轮到自己出块时，出一个预先计算好的区块：通过这个区块计算出的 `RANDAO` 的核心值，可以产生自己想要的「随机值」，从而控制出块和投票。

但总而言之，我对这个问题确实不是很理解，如果您了解使用 `MAX_SEED_LOOKAHEAD` 的原因，非常感谢您能赐教。

激活 validator 还有一个重要的限制：为了安全起见，在每个 epoch 周期内，不会一次性激活太多 validator 。否则的话，有可能一下子次涌入超过 1/3 的 validator ，而这些 validator 如果属于某一个人或团体，那就区块链就被控制了。在 Beacon 手册中限制的代码如下：
```Python
def process_registry_updates(state: BeaconState) -> None:
    # hiden code ......
     for index in activation_queue[:get_validator_churn_limit(state)]:
        validator = state.validators[index]
        validator.activation_epoch = compute_activation_exit_epoch(get_current_epoch(state))

def get_validator_churn_limit(state: BeaconState) -> uint64:
    """
    Return the validator churn limit for the current epoch.
    """
    active_validator_indices = get_active_validator_indices(state, get_current_epoch(state))
    return max(MIN_PER_EPOCH_CHURN_LIMIT, uint64(len(active_validator_indices)) // CHURN_LIMIT_QUOTIENT)
```
计算限制的函数中 `get_validator_churn_limit` ，它的规则是计算当前 validator 的总数量是几倍于 `CHURN_LIMIT_QUOTIENT` 这个值，然后选取倍数与 `MIN_PER_EPOCH_CHURN_LIMIT` 中较大的那个，总得来说就是 validator 的数量越多，一次性激活的数量也就可以越多。


## 退出

有质押成为 validator 的想法，就有退出不想当 validator 的想法。退出其实分两种，一种是主动退出，一种是被动退出。主动退出好理解，就是自己不想干了；被动退出则是指 validator 出错太多，被罚了很多钱，系统主动踢出。

前面我们提到，成为一个正式的 validator 需要 32 ETH，这 32 个 ETH 连同被奖励的钱，都会记录在 `effective_balance` 中；在每个 epoch 开始的时候，系统会检查这个字段的值是否小于或等于 `EJECTION_BALANCE`，如果是，则会准备将这个 validator 踢出。
```Python
def process_registry_updates(state: BeaconState) -> None:
    # Process activation eligibility and ejections
    for index, validator in enumerate(state.validators):
        # hiden code ......

        if (
            is_active_validator(validator, get_current_epoch(state))
            and validator.effective_balance <= EJECTION_BALANCE
        ):
            initiate_validator_exit(state, ValidatorIndex(index))

    # hiden code ......

def initiate_validator_exit(state: BeaconState, index: ValidatorIndex) -> None:
    """
    Initiate the exit of the validator with index ``index``.
    """
    # hide code ......

    validator.exit_epoch = exit_queue_epoch
    validator.withdrawable_epoch = Epoch(validator.exit_epoch + MIN_VALIDATOR_WITHDRAWABILITY_DELAY)
```
退出的主要方式是设置 validator 的 `exit_epoch` 字段。当 epoch 超过这个值时，这个 validator 就不会被当作一个有效的 validator:
```Python
def is_active_validator(validator: Validator, epoch: Epoch) -> bool:
    """
    Check if ``validator`` is active.
    """
    return validator.activation_epoch <= epoch < validator.exit_epoch
```

如果是自愿退出，就要构造并签名一个 `SignedVoluntaryExit` 消息。在处理每个 Block 时，会发现并处理这个消息：
```Python
def process_block(state: BeaconState, block: BeaconBlock) -> None:
    # hiden code ......
    process_operations(state, block.body)

def process_operations(state: BeaconState, body: BeaconBlockBody) -> None:
    # hiden code ......
    for_ops(body.voluntary_exits, process_voluntary_exit)

def process_voluntary_exit(state: BeaconState, signed_voluntary_exit: SignedVoluntaryExit) -> None:
    voluntary_exit = signed_voluntary_exit.message
    validator = state.validators[voluntary_exit.validator_index]
    # Verify the validator is active
    assert is_active_validator(validator, get_current_epoch(state))
    # Verify exit has not been initiated
    assert validator.exit_epoch == FAR_FUTURE_EPOCH
    # Exits must specify an epoch when they become valid; they are not valid before then
    assert get_current_epoch(state) >= voluntary_exit.epoch
    # Verify the validator has been active long enough
    assert get_current_epoch(state) >= validator.activation_epoch + SHARD_COMMITTEE_PERIOD
    # Verify signature
    domain = get_domain(state, DOMAIN_VOLUNTARY_EXIT, voluntary_exit.epoch)
    signing_root = compute_signing_root(voluntary_exit, domain)
    assert bls.Verify(validator.pubkey, signing_root, signed_voluntary_exit.signature)
    # Initiate exit
    initiate_validator_exit(state, voluntary_exit.validator_index)
```
可以看到自愿退出的时候，最终调用的函数是 `process_voluntary_exit` ，这个函数主要检查参数和签名是否正常，然后同样调用 `initiate_validator_exit` 设置其退出。

同质押类似，退出时一个 epoch 内也不能一下子退出太多。这个逻辑是设计在 `initiate_validator_exit` 里的：
```Python
def initiate_validator_exit(state: BeaconState, index: ValidatorIndex) -> None:
    """
    Initiate the exit of the validator with index ``index``.
    """
    # hide code ......

    # Compute exit queue epoch
    exit_epochs = [v.exit_epoch for v in state.validators if v.exit_epoch != FAR_FUTURE_EPOCH]
    exit_queue_epoch = max(exit_epochs + [compute_activation_exit_epoch(get_current_epoch(state))])
    exit_queue_churn = len([v for v in state.validators if v.exit_epoch == exit_queue_epoch])
    if exit_queue_churn >= get_validator_churn_limit(state):
        exit_queue_epoch += Epoch(1)

    # hide code ......
```
这里会计算所有退出的 validator （包含当前要退出的）最大退出 epoch 值，然后判断在这个 epoch 退出的 validator 的数量，如果超过限制了，就把当前要退出的 validator 放在下一个 epoch 退出。


## Genesis

前面说过，Beacon Chain 并非与以太坊主链完全没有关系，这个关系体现之一就是，它不像其它完全独立的主链一样，随时可以启动（创建创世块），而是需要满足一定条件的。

首先，创世块的创建需要建立在原以太坊的 block 之上：
```Python
def initialize_beacon_state_from_eth1(eth1_block_hash: Hash32,
                                      eth1_timestamp: uint64,
                                      deposits: Sequence[Deposit]) -> BeaconState:
    # hiden code ......

    state = BeaconState(
        genesis_time=eth1_timestamp + GENESIS_DELAY,
        # hiden code ......
    )
    # hiden code ......
```
Beacon 手册中这个函数用来创建一个「候选的 state」（candidate state），从参数中就可以看出来，它需要原以太坊（eth1） 的 block 信息。

但并非这个函数创建的 state 就是创世 state 。要想成为创世 state 必须要满足以下两个条件：
1. 创世时间不能早于 `MIN_GENESIS_TIME`
2. validator 数量不能少于 `MIN_GENESIS_ACTIVE_VALIDATOR_COUNT`
所谓「创世时间」指的是 `genesis_time` 这个字段。

这个判断是在 `is_valid_genesis_state` 中完成的：
```Python
def is_valid_genesis_state(state: BeaconState) -> bool:
    if state.genesis_time < MIN_GENESIS_TIME:
        return False
    if len(get_active_validator_indices(state, GENESIS_EPOCH)) < MIN_GENESIS_ACTIVE_VALIDATOR_COUNT:
        return False
    return True
```

另外，即使 `is_valid_genesis_state` 返回 true，也不会立即启动，因为 `genesis_time` 的设置是有「机关」的：`genesis_time` 的值为 `eth1_timestamp + GENESIS_DELAY` ，即 eth1 区块的时间加上一个延迟 `GENESIS_DELAY` ，这个值 7 天。

所以总得来说，关于创世块的创建与 Beacon Chain 的正式运行，可以总结如下：
- 当第一个满足以下条件的 eth1 区块被发现时（为了方便，我们称它为区块 O），用它来创建创世 state ：
  - 区块 O 的时间晚于或等于 `MIN_GENESIS_TIME`
  - 截止（包含）区块 O 为止，统计到的 validator 数量不少于 `MIN_GENESIS_ACTIVE_VALIDATOR_COUNT` 个
- 当前时间为区块 O 的时间 + `GENESIS_DELAY` 时，开始正式运行。

上面几条中最后一个逻辑的代码如下：
```Rust
async fn wait_for_genesis<E: EthSpec>(
    beacon_nodes: &BeaconNodeFallback<SystemTimeSlotClock, E>,
    genesis_time: u64,
    context: &RuntimeContext<E>,
) -> Result<(), String> {
    let now = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .map_err(|e| format!("Unable to read system time: {:?}", e))?;
    let genesis_time = Duration::from_secs(genesis_time);

    // If the time now is less than (prior to) genesis, then delay until the
    // genesis instant.
    //
    // If the validator client starts before genesis, it will get errors from
    // the slot clock.
    if now < genesis_time {
        tokio::select! {
            result = poll_whilst_waiting_for_genesis(beacon_nodes, genesis_time, context.log()) => result?,
            () = sleep(genesis_time - now) => ()
        };
        /// hiden code .....
    }
    /// hiden code .....
}
```


## slot & epoch

每个区块链出块都有一个「节奏」：PoW 出块有「节凑」，尽量不要太快也不能太慢；PoS 也需要一个节奏，由于 PoS 不需要「挖矿」，节奏比 PoW 好控制多了，只要人为设置一个时间间隔就可以了。

Beacon Chain 中出块的时间间隔被称为 `slot` ，当前值为 12 秒（`SECONDS_PER_SLOT`）；每 32 个 slot 组成一个 epoch （`SLOTS_PER_EPOCH`）。每个 slot 和 epoch 都有一个编号，从创世开始从 0 按 1 递增。

显然，slot 和 epoch 这俩概念就是为了控制区块链的「节奏」：slot 好理解，就是为了控制出块节奏；epoch 又有什么用呢？

其实 epoch 控制的不是出块节奏，而是其它事情的「节奏」，这其中包括：
- 重新分配 validator 和创建 committees 的节奏
- 创建 checkpoint  的节奏
- 修改 checkpoint 的状态（`justified` 或 `finalized`） 的节奏
虽然这里面出现了很多概念我们还没涉及到，不过现在不用理会。你只要理解：slot 是出块的节奏，epoch 是做上面三件事的节奏。

当然，严格地说，epoch 不仅是上面三件事的节奏，还有好多事，这从 `process_epoch` 这个函数就能看出来：
```Python
def process_epoch(state: BeaconState) -> None:
    process_justification_and_finalization(state)
    process_rewards_and_penalties(state)
    process_registry_updates(state)
    process_slashings(state)
    process_eth1_data_reset(state)
    process_effective_balance_updates(state)
    process_slashings_reset(state)
    process_randao_mixes_reset(state)
    process_historical_roots_update(state)
    process_participation_record_updates(state)
```


## validator & committee

validator 指的是 PoS 共识中参与出块与投票的人，这个概念相信大家都比较理解。committee 是指由 128（`TARGET_COMMITTEE_SIZE`） 个 validator 组成的「委员会」，用于对 Proposer 出的块进行投票。注意 Proposer 是单独存在的，committee 中的成员只用来投票（虽然有可能在某个 slot 中，一个 Proposer 同时也是 committee 的成员）。每个 slot 可以有多个 committee ，最多有 64 个（`MAX_COMMITTEES_PER_SLOT`）。

总得来说，就是每个出块的 slot 都会有一个或多个 committee ，这些 committee 用于对这个块进行投票，决定这个块是否会被加到链上。每个 committee 由 128 个 validator 组成。

我们前面提过，epoch 是创建 committee 的节奏。也就是说，每个 epoch 周期开始时，此 epoch 内的每一个 solt 都有哪些 committee 、每个 committee 都由哪些 validator ，都是在每个 epoch 周期开始的时候，就设置好了。 这是如何分配的呢？

`get_beacon_committee` 函数用来分配指定 slot 和 index 的 committee 的成员：
```Python
def get_beacon_committee(state: BeaconState, slot: Slot, index: CommitteeIndex) -> Sequence[ValidatorIndex]:
    """
    Return the beacon committee at ``slot`` for ``index``.
    """
    epoch = compute_epoch_at_slot(slot)
    committees_per_slot = get_committee_count_per_slot(state, epoch)
    return compute_committee(
        indices=get_active_validator_indices(state, epoch),
        seed=get_seed(state, epoch, DOMAIN_BEACON_ATTESTER),
        index=(slot % SLOTS_PER_EPOCH) * committees_per_slot + index,
        count=committees_per_slot * SLOTS_PER_EPOCH,
    )
```
注意这里调用的参数需要 `slot` 和 `index` ，其中 `slot` 即前面我们提到的 slot 的值；`index` 是想要获取的 committee 的 index —— 前面我们说过，一个 slot 可能分配多个 committee 。

这个函数里调用了 `get_committee_count_per_slot` 来计算每个 slot 需要分配多少个 committee，它的实现如下：
```Python
def get_committee_count_per_slot(state: BeaconState, epoch: Epoch) -> uint64:
    """
    Return the number of committees in each slot for the given ``epoch``.
    """
    return max(uint64(1), min(
        MAX_COMMITTEES_PER_SLOT,
        uint64(len(get_active_validator_indices(state, epoch))) // SLOTS_PER_EPOCH // TARGET_COMMITTEE_SIZE,
    ))
```
计算方法就是根据当前有效的 validator 的总数量除以 epoch 周期中 slot 的数量，得到每个 slot 可以有多少个 validator ；然后将每个 slot 可以分配的 validator 数量除以 committee 数量（`TARGET_COMMITTEE_SIZE`），得到每个 slot 可以分配多少个 committee。

最后，`get_beacon_committee` 函数调用了 `compute_committee` 最终返回 committee 成员：
```Python
def compute_committee(indices: Sequence[ValidatorIndex],
                      seed: Bytes32,
                      index: uint64,
                      count: uint64) -> Sequence[ValidatorIndex]:
    """
    Return the committee corresponding to ``indices``, ``seed``, ``index``, and committee ``count``.
    """
    start = (len(indices) * index) // count
    end = (len(indices) * uint64(index + 1)) // count
    return [indices[compute_shuffled_index(uint64(i), uint64(len(indices)), seed)] for i in range(start, end)]
```
`start` 的计算方法我们可以不管（事实是也不唯一），但 `end` 的计算方式让我疑惑了一阵。前面我们提到过，一个 committee 的成员数量是 `TARGET_COMMITTEE_SIZE` ，所以这里 end - start 按道理应该等于这个值才对。根据算式推算，end - start == len(indices) // count ，那为什么 `len(indices) // count` 等于 `TARGET_COMMITTEE_SIZE` 呢？这里的关键是 `count` 参数，它的实参是 `committees_per_slot * SLOTS_PER_EPOCH` ，即一个 epoch 中一共有多少个 committee （注意 `committees_per_slot` 这个值可是根据 `TARGET_COMMITTEE_SIZE` 算出来的），而 `len(indices)` 则代表「一个 epoch 中一共有多少个 validator」，所以它俩相除，自然就是 `TARGET_COMMITTEE_SIZE` 。其实我觉得把 `count` 参数去掉，`end` 的计算直接改成 `end = start + TARGET_COMMITTEE_SIZE` 更好理解；但这样一来，这个函数就不是一个「真正的函数」了（真正的函数中指，每次给定相同的参数，它肯定会输出相同的结果），这个就看实现者的取舍了。

上面的函数中，其实还涉及到 `get_seed` 和 `compute_shuffled_index` 两个函数，它们的代码就不在这里展示了，但与它们相关的「随机」的概念，必须得说明一下。每个 slot 和 epoch ，Proposer 的选取和 committee 的组成都是「随机」的。如果不是这样，就可能导致恶意结点总是被分配到同一个或多个 committee 中，从而可以控制出块；「随机」后，可以有效打乱这些恶意结点的「团结」：
![](/pic/beaconchain/shuffle_compare.png)

而 Beacon Chain 中随机的关键被称为 `RANDAO` ，我们后会会详细解释。这里只需要了解：
- 它保证产生的随机值不会被预测到 
- 每个 Beacon 结点在同一个 epoch 周期中，得到的随机值是一样的

上面的 `get_seed` 就依赖于 `RANDAO` 的这些特性产生随机种子，从而保证每个 epoch 中分配的 validator 是随机的（但每个 Beacon 结点在相同的 epoch 中分配的 validator 又是一样的）

另一个需要稍微聊一下的点是，上面的代码中，都是使用索引 `ValidatorIndex`（这是个整数值）指代某个 validator ，并不是用一个地址指代。可能这样做的原因在于，validator 信息存在于 `BeaconState.validators` 数组中，在 Beacon 手册的示例代码中，这个数组只会在后面追加，不会删除中间的元素，也就是说，一个 validator 一旦被加入到这个数组中，就不会从这个数组中删除了，所以 Beacon 手册中的代码可以使用这个数组的索引来指代一个 validator 。（虽然这个数组设置得很大，但由于只能追加，总有用完的那天，到时候怎么办呢？或许因为 Beacon 手册里的代码都是示例代码，不需要考虑这些问题吧。）


## RANDAO

前面我们提过，每个 slot 中 Proposer 的选取和每个 epoch 开始时 committee 的分配都是「随机」的，`RANDAO` 则是对这个「随机」的底层支持，它有以下两个特点：
- 它保证产生的随机值不会被预测到 
- 每个 Beacon 结点在同一个 epoch 周期中，得到的随机值是一样的

虽然我觉得 `RANDAO` 这个名挺「高大上」的，但它的实现其实非常简单。这里我先把它的一些关键点列出来，最后将它们串起来，就很容易理解它了。

我们先来看 Beacon 手册中更新 `randao_mixes` 的函数：
```Python
def process_randao(state: BeaconState, body: BeaconBlockBody) -> None:
    epoch = get_current_epoch(state)
    # Verify RANDAO reveal
    proposer = state.validators[get_beacon_proposer_index(state)]
    signing_root = compute_signing_root(epoch, get_domain(state, DOMAIN_RANDAO))
    assert bls.Verify(proposer.pubkey, signing_root, body.randao_reveal)
    # Mix in RANDAO reveal
    mix = xor(get_randao_mix(state, epoch), hash(body.randao_reveal))
    state.randao_mixes[epoch % EPOCHS_PER_HISTORICAL_VECTOR] = mix
```
这个函数是 `RANDAO` 的一个核心，我们详细解释一下。这个函数的第二个参数是一个区块的 body ，在此函数中，先是保证区块的签名是正确的，然后最后关键的两行代码：
```Python
    mix = xor(get_randao_mix(state, epoch), hash(body.randao_reveal))
    state.randao_mixes[epoch % EPOCHS_PER_HISTORICAL_VECTOR] = mix
```
利用了区块的签名信息，与当前的 `randao_mixes` （其实就是 `BeaconState.randao_mixes[epoch % EPOCHS_PER_HISTORICAL_VECTOR]`） 异或，然后更新 `BeaconState.randao_mixes` 数组中的值。

这个函数是什么时候调用的呢？处理每个区块的时候（`process_block` 函数中调用）。也就是说，每生成一个区块，就会改变当前 epoch 的 `randao_mixes` 值。

`BeaconState.randao_mixes` 又是用来干什么的呢？它在 `get_seed` 中被调用：
```Python
def get_seed(state: BeaconState, epoch: Epoch, domain_type: DomainType) -> Bytes32:
    """
    Return the seed at ``epoch``.
    """
    mix = get_randao_mix(state, Epoch(epoch + EPOCHS_PER_HISTORICAL_VECTOR - MIN_SEED_LOOKAHEAD - 1))  # Avoid underflow
    return hash(domain_type + uint_to_bytes(epoch) + mix)
```
还记得前面我们提过，分配 committee 时会调用 `get_seed` 吗？不过这里需要注意的是，调用 `get_randao_mix` 时的 epoch 值，并不是当前 epoch ，而是当前 epoch 减去 `MIN_SEED_LOOKAHEAD` 。

最后一个重要的点，就是每个 epoch 开始前，会将当前 epoch 的 `randao_mixes` 的值，更新为即将到来的 epoch 的 `randao_mixes` 值:
```Python
def process_randao_mixes_reset(state: BeaconState) -> None:
    current_epoch = get_current_epoch(state)
    next_epoch = Epoch(current_epoch + 1)
    # Set randao mix
    state.randao_mixes[next_epoch % EPOCHS_PER_HISTORICAL_VECTOR] = get_randao_mix(state, current_epoch)
```
注意，调用 `process_randao_mixes_reset` 时还没有真正进入新的 epoch ，比如，调用此函数时 slot 值可能是 63 （32 个 slot 组成一个 epoch）。`get_randao_mix` 返回的是当前 epoch 的 `randao_mixes` 值，所以这个函数把当前 epoch 的 `randao_mixes` 值赋值给了下一个 epoch 。

上面给出了所有线索，现在我们来串一下。`BeaconState.randao_mixes` 是核心，有的地方设置它，有的地方使用它。设置它的地方是：
- 处理每个 block 时，更新当前 epoch 的 `randao_mixes` 值
- 新的 epoch 即将开始（还未开始）时，把当前 epoch 的 `randao_mixes` 值拷贝给下一个 epoch

使用它的地方是：
- `get_seed` ，使用的是之前的 epoch （当前 epoch - `MIN_SEED_LOOKAHEAD`） 的 `randao_mixes` 值。

所以，总得来说，随机数的来源是每当新产生的区块时更新当前 epoch 的 `randao_mixes` 值；每个 epoch 即将结束时，将当前 epoch 的 `randao_mixes` 值拷贝给下一个 epoch ，以便下一个 epoch 能在当前的基础上继续更新；而当进入到下一个 epoch 后，之前所有 epoch 的 `randao_mixes` 的值都不会再改变以了。所以用的时候，用的是之前 epoch （当前 epoch - `MIN_SEED_LOOKAHEAD`）的 mix 值。如此一来，由于新的区块是无法预测的，因此当前 epoch 的 `randao_mixes` 如何变化就无法预测了；而之前 epoch 的 `randao_mixes` 又是已经固定的、不变的，所以各个结点调用 `get_seed` 使用它时，得到的值又都是确定的。


## checkpoint, justified & finalized

Beacon Chain 中有 `checkpoint` 的概念，所谓 `checkpoint` ，是指每个 epoch 中第一个 slot 产生的区块。是的，与之前我们介绍 [PBFT](https://yangzhe.me/2019/11/25/pbft/) 的文章中的 checkpoint 不同，这里的 checkpoint 不需要投票产生，只要它是一个 epoch 中第一个 slot 产生的区块，它就是 checkpoint。

其实 checkpoint 这个概念，我觉得主要是为了定义区块的 `justified` 和 `finalized` 状态。如果一个区块有超过 2/3 的 balance 投票给它，那么就称它达到 `justified` 状态；如果一个 `justified` 的区块 A（假设高度为 H）的下一个区块 B（高度为 H+1）达到 `justified` 状态，那么区块 A 自动变为 `finalized` 状态。这个逻辑是在 Beacon 手册中的 `weigh_justification_and_finalization` 函数中实现的，这里就不展示了。

其实从手册里看不出来区块达到 `finalized` 状态有什么意义，有文章说 `finalized` 是为了在 Sharding 中使用的，这个概念有助于降低 shard 之间沟通的复杂度。不过 Sharding 还在研究没有正式定下来，我们这里就不多讨论了。


# 总结

这篇文章里，我们重点介绍了 Beacon Chain 中一些比较与众不同的概念与特性，主要参照 Beacon 手册中的示例代码，以及小部分 lighthouse 的代码。

Beacon Chain 作为以太坊从 PoW 向 PoS 转变的关键，承载了非常大的意义。不过虽然现在 Beacon Chain 已经运行，但生成的区块中除了保存质押的 validator 的信息，并不计算和保存以太坊上的交易、合约等信息。但在不久以后，Beacon Chain 就会与主链合并（即所谓的 [The Merge](https://ethereum.org/en/upgrades/merge), 也可以参看我[之前的文章的介绍](https://yangzhe.me/2022/08/23/eth2-brief/#beacon-chain)），届时 Beacon Chain 的链上也会记录主链上的信息。我们拭目以待吧。

限于作者水平，文章中难免有错误的地方，如果发现感谢您能不吝指正。