---
title: "以太坊机制详解:交易与交易池"
date: 2022-10-14T23:47:33Z
tags: [ethereum,geth]
aliases: ["/2022/09/26/ethereum-transcation/"]
---

## 概述

我们在上一篇[以太坊机制详解:Gas Price计算](https://hugo.wongssh.cf/posts/ethereum-gas/)文章内介绍了以太坊中交易的具体`gas`计算相关规则。本篇文章将在此基础上介绍以下内容:

1. 客户端如何构建一笔交易
1. 服务端如何处理交易并进行打包

本文以一笔交易的生命周期为主线安排全文，从交易池初始化、交易构造到交易执行，尽可能为读者全景展示以太坊中的交易所涉及的方方面面。

![State with Tx](https://img-blog.csdnimg.cn/img_convert/1186f0862f7dfe370a4f20b152a472e4.png)

对于交易的打包涉及状态树、数据库等内容，为强化文章专题性，我们将这一部分内容放到下一篇文章内。

## 交易池初始化

在交易进行之前，节点首先需要完成节点初始化，本节内容主要聚焦于此。

节点需要对以下数据进行初始化:
```go
type TxPoolConfig struct {
	Locals    []common.Address // Addresses that should be treated by default as local
	NoLocals  bool             // Whether local transaction handling should be disabled
	Journal   string           // Journal of local transactions to survive node restarts
	Rejournal time.Duration    // Time interval to regenerate the local transaction journal

	PriceLimit uint64 // Minimum gas price to enforce for acceptance into the pool
	PriceBump  uint64 // Minimum price bump percentage to replace an already existing transaction (nonce)

	AccountSlots uint64 // Number of executable transaction slots guaranteed per account
	GlobalSlots  uint64 // Maximum number of executable transaction slots for all accounts
	AccountQueue uint64 // Maximum number of non-executable transaction slots permitted per account
	GlobalQueue  uint64 // Maximum number of non-executable transaction slots for all accounts

	Lifetime time.Duration // Maximum amount of time non-executable transaction are queued
}
```

各参数含义如下:

- Locals 本地账户地址列表。在以太坊执行层节点内，本地交易拥有一系列特权，节点会监控交易来源地址并确定是否将其列为本地交易
- `NoLocals` 禁止本地交易处理，将所有交易均列为远程交易(即无特权交易)
- `Journal` 对交易池内的本地交易进行数据持久化的文件名
- `Rejournal` 进行数据持久化的时间间隔
- `PriceLimit` 交易池接受到最小`gas`价格，在`EIP1559`交易中代表最小`Priority Fee`价格
- `PriceBump` 交易`gas`价格提升的最小百分比
- `AccountSlots` 单个账户可执行交易的最大容量
- `GlobalSlots` 所有账户可执行交易的最大容量
- `AccountQueue` 最大单个账户非可执行交易容量
- `GlobalQueue` 全局最大非可执行交易容量
- `Lifetime` 非可执行交易的存活时间

我们在上文中使用了**交易容量**而非**交易数量**，两者具有一定的区别，一般来说，一笔交易占用 1 单位交易容量，具体计算公式如下:
```go
func numSlots(tx *types.Transaction) int {
	return int((tx.Size() + txSlotSize - 1) / txSlotSize)
}
```
其中`txSlotSize`为常量，其值为`32768`。`tx.Size()`代表一笔交易经过`RLP`编码后的体积，单位为`bytes`。正常交易都仅占用 1 单位交易容量。

> 设置上述规则的原因在于避免部分用户构建超大交易占用交易池资源，正常来说一笔交易仅占用`2000 bytes`左右。

上述各值的初始化如下:
```go
var DefaultTxPoolConfig = TxPoolConfig{
	Journal:   "transactions.rlp",
	Rejournal: time.Hour,

	PriceLimit: 1,
	PriceBump:  10,

	AccountSlots: 16,
	GlobalSlots:  4096 + 1024, // urgent + floating queue capacity with 4:1 ratio
	AccountQueue: 64,
	GlobalQueue:  1024,

	Lifetime: 3 * time.Hour,
}
```

在交易池中，我们把交易分为远程交易(`remote transactions`)和本地交易(`local transaction`)。其中，后者具有以下优先级:

1. 写入`Journal`文件中，在节点启动时直接载入交易池
1. 不受交易池中的`PriceLimit`等限制
1. 在交易排序时优先级最高
1. 不会因为交易队列已满等原因被交易队列剔除

我们使用`RPC API`向节点发送的交易也属于`local transactions`(前提为节点将`NoLocals`设置为`False`)。

> 此处的`PriceLimit`参数对应`EIP1559`交易中的[Max Fee](https://hugo.wongssh.cf/posts/ethereum-gas#max-fee)参数，这意味着只要`Max Fee`大于`PriceLimit`即可进入交易池

在此处也出现了两种交易类型，如下:
1. 可执行交易(executable transaction)，此交易位于`pending`队列中，极有可能被节点封装进入下一个区块
1. 不可执行交易(non-executable transaction)，此交易位于`queued`队列中，不太可能被节点封装进入下一个区块

> 封装(`seal`)是目前以太坊对于区块打包的描述

我们通过[NewTxPool](https://github.com/ethereum/go-ethereum/blob/052c634/core/tx_pool.go#L279)函数使用上述初始化配置实现交易池的初始化。由于篇幅限制，我们不对其进行详细介绍。

初始化完成后，我们通过[SubscribeNewTxsEvent](https://github.com/ethereum/go-ethereum/blob/052c634/core/tx_pool.go#L430)函数订阅在以太坊网络中广播的新的交易项目。

## 交易构造

本节内容主要介绍客户端如何构造一笔以太坊交易。鉴于本文的很多读者并没有独立运行的以太坊节点，我们在此处主要介绍通过以太坊的`API`构建交易。

在一笔交易的最初阶段，用户需要完成交易的初始化，设置一笔交易的各个参数。在此处，我们以以太坊标准API`eth_sendTransaction`为例向大家介绍一笔交易的具体构成，具体构成如下:

- `type` 交易类型，如果使用`EIP1559`类型交易，则设置为`0x02`
- `nonce` 用户的`nonce`，此数值会在用户完成每一笔交易后增加`1`
- `to` 交易目标地址
- `from` 交易来源地址
- `gas` 即`gas limit`，具体参考[Gas Limit 的获取](https://hugo.wongssh.cf/posts/ethereum-gas#gas-limit-%E7%9A%84%E8%8E%B7%E5%8F%96)
- `value` 交易转账的`ETH`数量(单位为`wei`)
- `input` 交易包含的合约运行数据，如果交互对象不是合约，可置为`0x`
- `gasPrice` 如果使用`EIP1559`，此项可置为空
- `maxPriorityFeePerGas` 设置的[Max Priority Fee](https://hugo.wongssh.cf/posts/ethereum-gas#max-priority-fee)
- `maxFeePerGas` 设置[Max Fee](https://hugo.wongssh.cf/posts/ethereum-gas#max-fee)
- `accessList` 由`EIP2930`进行了一些规定，由于目前使用较少，我们不进行介绍
- `chainID` 链ID，可通过[ChainList](https://chainlist.org)获得相关数据

接下来，我们尝试使用`MetaMask`提供的API构建一笔交易。

首先，我们需要任一已被`MetaMask`授权的网站进行测试。在此处我们以`MetaMask`的演示网站[MetaMask Test Dapp](https://metamask.github.io/test-dapp/)为例。点击`CONNECT`进行账户授权，授权完成后，可以点击`ETH_ACCOUNTS`与自己的地址进行对比。完成上述准备后，点击`F12`进入开发者模式后选择`Console`进入`Javascript`终端。

输入以下内容初始化交易参数:
```javascript
const transactionParameters = {
  to: '0x0000000000000000000000000000000000000000', 
  from: ethereum.selectedAddress, 
  value: '0x00', 
  data:
    '0x7f7465737432000000000000000000000000000000000000000000000000000000600057'
};
```
我们在此处省略了很多字段，这些字段会被`MetaMask`自动补齐。

输入内容并运行:
```javascript
const txHash = await ethereum.request({
  method: 'eth_sendTransaction',
  params: [transactionParameters],
});
```
如果一切顺利，读者可以看到如下内容:
![MetaMask API](https://img.gejiba.com/images/62fefc63e6ff21d1a32e1588b27a3861.png)

> 如果读者需要设置`gas`、`value`等参数，需要注意这些参数均使用`wei`作为单位，同时使用 16 进制进行编码。1 wei 为 `0.000000000000000001 eth`。

更加详细的对于此API的说明，读者可以自行参考[文档](https://docs.metamask.io/guide/sending-transactions.html)或者前往[MetaMask JSON-RPC API Reference](https://metamask.github.io/api-playground/api-documentation/)

值得注意的是，大部分RPC服务商均不支持此API。读者可以发现上述交易中不包含签名，但由于RPC服务商不托管用户私钥，不能对交易进行签名，所以不能进行交易提交。`MetaMask`钱包中包含用户私钥所以可以调用此函数。

> RPC服务商一般允许使用`eth_sendRawTransaction`接口，此接口需要提交已经完成签名的并使用RLP编码的交易。本质上，`MetaMask`也调用了此接口。

如果读者希望通过命令行提交交易，可以使用`Foundry`提供的`cast`命令，具体可以参考[cast send](https://book.getfoundry.sh/reference/cast/cast-send)命令，支持上述所有参数。一个最简单的案例如下:
```bash
cast send 0x11475691C2CAA465E19F99c445abB31A4a64955C --value 0.001ether --gas-limit 21000 --gas-price 5gwei --priority-gas-price 1.5gwei --private-key $pk --rpc-url https://goerli.infura.io/v3/9aa3d95b3bc440fa88ea12eaa4456161
```
其中`$pk`需要替换为用户自己的私钥。`--gas-price`的含义为`Max Fee`，`--priority-gas-price`含义为`Max priority fee`，详细介绍请参考上文给出的[文档](https://book.getfoundry.sh/reference/cast/cast-send)。

由于后文会使用到以太坊内的交易类型，在此处，我们一并给出交易在`go-ethereum`中的接口，如下:
```go
type TxData interface {
	txType() byte // returns the type ID
	copy() TxData // creates a deep copy and initializes all fields

	chainID() *big.Int
	accessList() AccessList
	data() []byte
	gas() uint64
	gasPrice() *big.Int
	gasTipCap() *big.Int
	gasFeeCap() *big.Int
	value() *big.Int
	nonce() uint64
	to() *common.Address

	rawSignatureValues() (v, r, s *big.Int)
	setSignatureValues(chainID, v, r, s *big.Int)
}
```
各个参数的含义如下:

- `txType` 返回交易的类型，对应`type`参数
- `gasTipCap` 对应交易设置中的`maxPriorityFeePerGas`，即`Max Priority Fee`
- `gasFeeCap` 对应交易设置中的`maxFeePerGas`，即`Max Fee`

其他参数较为简单，读者可以直接通过名称推断其含义，故不再进行介绍。

## 交易池添加交易

当交易完成设置并通过API发送后，节点中的交易池会接受到此交易，并将其纳入自己的交易队列中。在详细分析交易进入队列之前，我们首先讨论一下以太坊中交易队列的类型。

在以太坊中，我们可以将交易队列使用以下`Venn`图表示:

![Tx Types](https://img.gejiba.com/images/f0f90a3e1b3781c822373519ab230ac5.png)

所有交易可以根据来源首先被划分为两类:

1. 本地交易 `local transaction`
1. 远程交易 `remote transaction`

正如前文所言，前者在优先级上高于后者所有交易，所以在以太坊交易池中属于最高等级，需要独立对待。

而远程交易`remote transaction`则被细分为了以下两个队列:

1. `pending`队列 - 此队列数据基本可以保证会被纳入下一个区块，而在交易广播时也只广播此队列内的交易
1. `queued`队列(亦称`queue`队列) - 此队列内的数据只能在交易池刷新队列时可能被纳入`pending`队列，我们会在后文进行介绍具体的更新规则

> 但在`go-ethereum`中，将`local transaction`保存在了`pending`队列中，但另一方面保证了`local transaction`不会被`pending`队列剔除

在`go-ethereum`中，上述队列定义如下:
```go
pending map[common.Address]*txList   // All currently processable transactions
queue   map[common.Address]*txList   // Queued but non-processable transactions
```

在正常情况下，当一个新的交易进入节点交易池后，此交易有以下去向:

1. 大部分情况下直接进入`queue`队列等待刷新
1. 少部分用于增加`gas`费用的交易替换`pending`中的原有交易被纳入`pending`队列
1. 因不满足交易条件而被删除

我们首先分析用于在交易池中加入单个交易的`add`函数，其代码非常长，我们将逐块分析:
```go
func (pool *TxPool) add(tx *types.Transaction, local bool) (replaced bool, err error)
```
函数定义说明，此函数接受交易和标识交易是否为本地交易的`local`标识符作为输入，返回代表此交易是否替换了其他交易的`replaced`标识和错误`err`

整个流程可以使用以下流程图说明:

![Tx Add Flow](https://s-bj-3358-blog.oss.dogecdn.com/svg/txAdd.drawio.svg)

```go
hash := tx.Hash()
if pool.all.Get(hash) != nil {
	log.Trace("Discarding already known transaction", "hash", hash)
	knownTxMeter.Mark(1)
	return false, ErrAlreadyKnown
}
```
首先获得交易的哈希值，如果交易与交易池内的任何交易的哈希值相同，我们则丢弃此交易，并抛出异常

```go
isLocal := local || pool.locals.containsTx(tx)
```
此代码判断交易是否为`local transaction`。如果满足`local`标识符为`True`或交易发送者位于`Locals`地址列表内条件，则认定此交易为`local transaction`

```go
if err := pool.validateTx(tx, isLocal); err != nil {
	log.Trace("Discarding invalid transaction", "hash", hash, "err", err)
	invalidTxMeter.Mark(1)
	return false, err
}
```
使用`validateTx`函数验证交易是否符合交易池的要求，限于篇幅，我们在此处直接给出`validateTx`函数验证的内容，具体代码实现请自行查找。具体验证内容如下:

1. 交易进行`RLP`编码后体积不大于`131072 bytes`
1. 交易的`Gas Limit`不大于 3000 万(即当前**区块**的`GasLimit`)
1. 交易的`Max Fee`和`Max Priority Fee`不大于`2 ^ 256`
1. 交易的`Max Priority Fee`小于`Max Fee`
1. 交易签名正确
1. 交易的`Max Priority Fee`大于**交易池**设置的`PriceLimit`
1. 交易的`nonce`大于交易者当前的`nonce`
1. 交易者账户余额可以支付交易的`Gas`费用
1. 满足`AccessList`的一些`gas`要求

如果用户提交给节点的交易无法满足上述条件，则直接被丢弃。

当交易经过校验后，交易或被纳入`queued`或`pending`队列中，这一部分逻辑较为复杂。

首先，我们分析交易池容量已满的情况，我们使用以下的代码判定此情况:
```go
uint64(pool.all.Slots()+numSlots(tx)) > pool.config.GlobalSlots+pool.config.GlobalQueue
```
其中`pool.all.Slots()`会返回目前交易池内所有交易所占用的交易容量，`numSlots(tx)`计算准备进入交易池的交易的所占用的交易容量，如果两者之和大于`GlobalSlots`(所有账户可执行交易的最大容量)和`GlobalQueue`(全局最大非可执行交易容量)，我们可以判断交易池已满。

```go
if !isLocal && pool.priced.Underpriced(tx) {
	log.Trace("Discarding underpriced transaction", "hash", hash, "gasTipCap", tx.GasTipCap(), "gasFeeCap", tx.GasFeeCap())
	underpricedTxMeter.Mark(1)
	return false, ErrUnderpriced
}
```
当交易不是优先级最高的本地交易，且交易的`gasTipCap`(`Max Priority Fee`)低于交易池内最低`gasTipCap`时，我们直接丢弃此交易。

> 交易池内维护有一个由交易价格构成的堆`heap`，可以快速查找价格最低的交易

```go
if pool.changesSinceReorg > int(pool.config.GlobalSlots/4) {
	throttleTxMeter.Mark(1)
	return false, ErrTxPoolOverflow
}
```
此处涉及到一个名词`Reorg`，此名词表示交易池重排。每当一个新的区块被生成，交易池会根据区块中的交易信息对交易池内的交易进行重组，包括在交易池内删除已被打包的交易(这部分交易往往位于`pending`队列中)、升级符合条件的`queued`队列中的交易、在已满的队列中删除交易以及广播交易。该部分核心实现为`runReorg`函数，此函数会在后文多次出现。我们一般使用`channel`这种特殊的`go`数据类型与作为单独线程的`runReorg`函数进行通信。

在此代码中`changesSinceReorg`代表现在需要重组的交易数量，如果此交易数量大于`GlobalQueue`(所有账户可执行交易的最大容量)的`1 / 4`，我们则认为交易池非常拥挤，直接丢弃新的交易。

> 在交易池启动后，`runReorg`函数会自动清除已满队列中的交易，开发者认为通过`add`函数删除太多交易并不合适，具体可参考[#23095](https://github.com/ethereum/go-ethereum/commit/d705f5a5543402a1d505bdc411d264cd4f883402)

```go
drop, success := pool.priced.Discard(pool.all.Slots()-int(pool.config.GlobalSlots+pool.config.GlobalQueue)+numSlots(tx), isLocal)

// Special case, we still can't make the room for the new remote one.
if !isLocal && !success {
	log.Trace("Discarding overflown transaction", "hash", hash)
	overflowedTxMeter.Mark(1)
	return false, ErrTxPoolOverflow
}
// Bump the counter of rejections-since-reorg
pool.changesSinceReorg += len(drop)
// Kick out the underpriced remote transactions.
for _, tx := range drop {
	log.Trace("Discarding freshly underpriced transaction", "hash", tx.Hash(), "gasTipCap", tx.GasTipCap(), "gasFeeCap", tx.GasFeeCap())
	underpricedTxMeter.Mark(1)
	pool.removeTx(tx.Hash(), false)
}
```
当我们认为交易可以被删除，我们则进行真正的交易删除步骤。首先通过`pool.priced.Discard`函数移除占用`pool.all.Slots()-int(pool.config.GlobalSlots+pool.config.GlobalQueue)+numSlots(tx)`单位的交易容量的交易。此函数不会真正删除交易，而是会返回待删除交易的列表，待删除交易的筛选规则为交易由高到低进行排序，优先删除价格最低的交易(本地交易不会被删除)。

如果无法获得待删除列表，且交易不是本地交易，则返回错误。如果获得待删除交易列表，我们会更新`changesSinceReorg`变量。然后使用`pool.removeTx`真正执行删除步骤。

当我们完成交易池容量方面的处理后，我们接下来处理一部分特殊的交易，即用于替换交易池`pending`队列的交易。这种替换交易往往用于增加已经在队列中的交易的`gas`，保证交易可以尽快完成。

我们可以通过以下代码判断此交易是否为替换交易:
```go
from, _ := types.Sender(pool.signer, tx) // already validated
if list := pool.pending[from]; list != nil && list.Overlaps(tx)
```
首先使用`types.Sender(pool.signer, tx)`获得交易的具体签名人，然后前往`pool.pending`队列中查询此用户名下的所有交易，并使用`Overlaps`函数判断交易是否存在重复。

> `pool.pending`是一个映射`pending map[common.Address]*txList`，我们可以通过用户地址在其内部快速检索相关交易

当我们发现交易池内已包含此笔交易后，我们会尝试将交易加入交易池，代码如下:
```go
inserted, old := list.Add(tx, pool.config.PriceBump)
if !inserted {
	pendingDiscardMeter.Mark(1)
	return false, ErrReplaceUnderpriced
}
```
此处使用到了`Add`函数，此函数接受交易`tx`后会在交易队列中查询与`tx`的`nonce`相同的交易。获得交易列表内的旧交易后，函数会校验`gasTipCap`和`gasFeeCap`相较于旧交易的增加幅度是否符合要求，如果满足上述条件，则直接在交易列表内替换旧的交易。同时返回需要替换的旧交易，以满足后续处理流程。当然，如果此处发现`Add`函数返回替换失败的标识，我们直接放弃替换。

关于`Add`函数，其具体代码如下:
```go
func (l *txList) Add(tx *types.Transaction, priceBump uint64) (bool, *types.Transaction) {
	// 获取交易列表内 Nonce 相同的旧交易
	old := l.txs.Get(tx.Nonce())
	if old != nil {
		// 要求新交易的 GasFeeCap 和 GasTipCap 大于旧交易
		if old.GasFeeCapCmp(tx) >= 0 || old.GasTipCapCmp(tx) >= 0 {
			return false, nil
		}
		// thresholdFeeCap = oldFC  * (100 + priceBump) / 100
		a := big.NewInt(100 + int64(priceBump))
		aFeeCap := new(big.Int).Mul(a, old.GasFeeCap())
		aTip := a.Mul(a, old.GasTipCap())

		// thresholdTip    = oldTip * (100 + priceBump) / 100
		b := big.NewInt(100)
		thresholdFeeCap := aFeeCap.Div(aFeeCap, b)
		thresholdTip := aTip.Div(aTip, b)

		// 要求新交易的 GasFeeCap 和 GasTipCapCmp 增加幅度大于 PriceBump
		if tx.GasFeeCapIntCmp(thresholdFeeCap) < 0 || tx.GasTipCapIntCmp(thresholdTip) < 0 {
			return false, nil
		}
	}
	// 使用新交易覆盖旧交易
	l.txs.Put(tx)

	// 以下内容为刷新参数
	if cost := tx.Cost(); l.costcap.Cmp(cost) < 0 {
		l.costcap = cost
	}
	if gas := tx.Gas(); l.gascap < gas {
		l.gascap = gas
	}
	return true, old
}
```

> 读者可自行查阅上述代码及其注释理解实现过程

读者可能发现上述流程仅对交易列表进行了替换，而不是对`pool`交易池的其他参数进行同步更新，所以我们使用以下代码更新交易池内的其他参数:
```go
if old != nil {
	pool.all.Remove(old.Hash())
	pool.priced.Removed(1)
	pendingReplaceMeter.Mark(1)
}
pool.all.Add(tx, isLocal)
pool.priced.Put(tx, isLocal)
pool.journalTx(from, tx)
pool.queueTxEvent(tx)
log.Trace("Pooled new executable transaction", "hash", hash, "from", from, "to", tx.To())
```
首先在用于维护交易池内所有交易的`pool.all`队列中删除此交易，同时在更新交易价格构成的`pool.priced`队列。使用`pool.journalTx`方法**尝试**将此替换交易纳入用于交易数据存储`Journal`内。

> 此处使用了**尝试**是因为在`journalTx`函数中会对交易的`from`进行审查，如果交易不来自`local`地址，则不会进行存储。

在此处，使用了一个较为特殊的函数`queueTxEvent`，此函数会将交易推送给`pool.queueTxEventCh`通道，此通道的最终目的地为`runReorg`函数，并对`queueTxEvent`队列进行修改。
```go
case tx := <-pool.queueTxEventCh:
	addr, _ := types.Sender(pool.signer, tx)
	if _, ok := queuedEvents[addr]; !ok {
		queuedEvents[addr] = newTxSortedMap()
	}
	queuedEvents[addr].Put(tx)
```

> 此处没有设置`scheduleReorgLoop`中的一个重要参数`launchNextRun`，此参数用于判断`runReorg`，即重排过程是否立即执行，若不设置，则意味着重排过程不会立即进行。

而`queueTxEvents`定义如下:
```go
queuedEvents  = make(map[common.Address]*txSortedMap)
```
其中`txSortedMap`是一个`nonce`到交易`transaction`的堆(`heap`)。最终，`queuedEvents`映射会被用于交易广播，以下给出的代码摘自`runReorg`函数的最后，其中`events`即此处的`queuedEvents`。
```go
if len(events) > 0 {
	var txs []*types.Transaction
	for _, set := range events {
		txs = append(txs, set.Flatten()...)
	}
	pool.txFeed.Send(NewTxsEvent{txs})
}
```

完成上述步骤后，我们会更新账户最新的活动时间，完成整个替换交易流程。代码如下:
```go
pool.beats[from] = time.Now()
return old != nil, nil
```

如果一笔交易既不是对现有交易的替换，我们会使用使用以下代码直接将其推入`queue`队列中，代码如下:
```go
replaced, err = pool.enqueueTx(hash, tx, isLocal, true)
if err != nil {
	return false, err
}
```
此处的`enqueueTx`会将交易推送进入`queue`队列，限于篇幅，我们在此处以注释的形式解释此函数的源代码:
```go
func (pool *TxPool) enqueueTx(hash common.Hash, tx *types.Transaction, local bool, addAll bool) (bool, error) {
	from, _ := types.Sender(pool.signer, tx) // already validated
	// 如果 queue 队列内没有此地址记录，则创建一个新的映射
	// queue 的定义为 map[common.Address]*txList
	if pool.queue[from] == nil {
		pool.queue[from] = newTxList(false)
	}
	// 使用 (l *txList) Add 直接加入交易
	// 详情请参考上文给出的代码解析
	inserted, old := pool.queue[from].Add(tx, pool.config.PriceBump)

	// 处理插入失败的情况
	if !inserted {
		queuedDiscardMeter.Mark(1)
		return false, ErrReplaceUnderpriced
	}

	if old != nil {
		// 发生替换交易情况，刷新 pool 参数
		// 类似上文提到的 替换交易 的后续处理
		pool.all.Remove(old.Hash())
		pool.priced.Removed(1)
		queuedReplaceMeter.Mark(1)
	} else {
		// 没有发生替换交易的情况，增加计数器
		// 计数器用于评估等功能
		queuedGauge.Inc(1)
	}
	// If the transaction isn't in lookup set but it's expected to be there,
	// show the error log.
	if pool.all.Get(hash) == nil && !addAll {
		log.Error("Missing transaction in lookup set, please report the issue", "hash", hash)
	}
	// 刷新 pool 中的 all 和 priced 变量
	if addAll {
		pool.all.Add(tx, local)
		pool.priced.Put(tx, local)
	}
	// 刷新交易账户的生命周期
	if _, exist := pool.beats[from]; !exist {
		pool.beats[from] = time.Now()
	}
	return old != nil, nil
}
```

完成上述重要任务后，最后我们处理本地交易的问题，代码如下:
```go
if local && !pool.locals.contains(from) {
	log.Info("Setting new local account", "address", from)
	pool.locals.add(from)
	pool.priced.Removed(pool.all.RemoteToLocals(pool.locals)) // Migrate the remotes if it's marked as local first time.
}
if isLocal {
	localGauge.Inc(1)
}
pool.journalTx(from, tx)

log.Trace("Pooled new future transaction", "hash", hash, "from", from, "to", tx.To())
return replaced, nil
```
如果发现交易被标记为`local`，但账户没有被标记为`local`，则直接将交易发生账户列入`locals`名单内并对此地址下的所有交易进行提权至本地交易(`local transaction`)，最终使用`journalTx`存储本地交易。

当然，正常情况下我们更有可能一次性增加大量交易，所以在源代码中，我们可以看到大量函数都使用了`addTxs`函数，而此函数中的一个核心部分是`addTxsLocked`函数，我们首先介绍此函数。

代码如下:
```go
func (pool *TxPool) addTxsLocked(txs []*types.Transaction, local bool) ([]error, *accountSet) {
	dirty := newAccountSet(pool.signer)
	errs := make([]error, len(txs))
	for i, tx := range txs {
		replaced, err := pool.add(tx, local)
		errs[i] = err
		if err == nil && !replaced {
			dirty.addTx(tx)
		}
	}
	validTxMeter.Mark(int64(len(dirty.accounts)))
	return errs, dirty
}
```
此处使用到了`pool.add`函数用于向交易池内增加交易，但为了方便使用`runReorg`函数进行交易重排，在此处定义了一个较为特殊的`dirty`变量，此变量为一个交易发送者地址列表，我们使用了`dirty.addTx(tx)`向此地址列表内增加新的交易发送者信息。

> 此函数名内包含`Locked`字样，这意味着此函数必须在交易池拿到线程锁时才能使用

我们也对此函数进行分析。
```go
func (pool *TxPool) addTxs(txs []*types.Transaction, local, sync bool) []error
```
在此函数中，各参数含义如下:

- `txs`代表需要加入交易池的交易集合
- `local`用于标识此交易集合内的交易是否为本地交易
- `sync`用于标识此交易集合内的交易是否立即用于提权，用于测试

> 正如前文所述，一般来说交易会被直接推入`queue`队列内，而函数`runReorg`会定期运行对交易进行提权至`pending`队列内。如果将`addTxs`中的`sync`设置为`True`，则意味着在下一次`runReorg`运行时会直接进行提权

此函数的流程图如下:

![AddTxs](https://s-bj-3358-blog.oss.dogecdn.com/svg/addTxs.drawio.svg)

`addTxs`函数首先对交易进行了一个简单的验证，具体代码如下:
```go
var (
	errs = make([]error, len(txs))
	news = make([]*types.Transaction, 0, len(txs))
)
for i, tx := range txs {
	// 验证交易是否已经存在在交易池内
	if pool.all.Get(tx.Hash()) != nil {
		errs[i] = ErrAlreadyKnown
		knownTxMeter.Mark(1)
		continue
	}
	// 验证交易签名是否正确
	_, err := types.Sender(pool.signer, tx)
	if err != nil {
		errs[i] = ErrInvalidSender
		invalidTxMeter.Mark(1)
		continue
	}
	// 将交易添加到 news 列表内
	news = append(news, tx)
}
if len(news) == 0 {
	return errs
}
```

在完成基本的交易校验后，使用`pool.addTxsLocked`函数将交易加入交易池内，代码如下:
```go
pool.mu.Lock()
newErrs, dirtyAddrs := pool.addTxsLocked(news, local)
pool.mu.Unlock()
```
此处使用了`pool.mu.Lock()`及`pool.mu.Unlock()`实现交易池线程锁定和解锁。

```go
var nilSlot = 0
for _, err := range newErrs {
	for errs[nilSlot] != nil {
		nilSlot++
	}
	errs[nilSlot] = err
	nilSlot++
}
```
一个简单的`for`循环实现`newErrs`中的错误到`err`的转移。

```go
done := pool.requestPromoteExecutables(dirtyAddrs)
if sync {
	<-done
}
return errs
```
此函数实际上实现了将交易通过`channel`推送给`runReorg`的作用。其中`requestPromoteExecutables`的定义如下:
```go
func (pool *TxPool) requestPromoteExecutables(set *accountSet) chan struct{} {
	select {
	case pool.reqPromoteCh <- set:
		return <-pool.reorgDoneCh
	case <-pool.reorgShutdownCh:
		return pool.reorgShutdownCh
	}
}
```
对于具体的`channel`作用，我们会在下一节进行介绍。

## 交易重排

我们在上一节内大量提到了`runReorg`函数及其作用，在本节我们将介绍`runReorg`实现交易重排的具体原理及其实现。

`runReorg`函数是由`scheduleReorgLoop`函数启动，而`scheduleReorgLoop`函数在交易池初始化时就被调用，具体可以参考`NewTxPool`函数中的下述代码:
```go
go pool.scheduleReorgLoop()
```
而在`scheduleReorgLoop`函数内，我们可以看到大量的`channel`的使用。我对于`golang`语言中的`channel`使用并不是非常熟悉。所以在后文内可能出现错误，发现错误的读者可以通过[我的博客](https://hugo.wongssh.cf)中给出的邮箱地址向我反馈。

我们首先给出一系列的`channel`定义:
```go
reqResetCh      chan *txpoolResetRequest
reqPromoteCh    chan *accountSet
queueTxEventCh  chan *types.Transaction
reorgDoneCh     chan chan struct{}
reorgShutdownCh chan struct{}  // requests shutdown of scheduleReorgLoop
```
其传输的信息主要为:

- `reqResetCh` 传输用于区块更新的相关信息
- `reqPromoteCh` 传输用于更新的指定地址集合
- `queueTxEventCh` 传输用于加入`queued`队列交易的信息
- `reorgDoneCh` 传输由空结构体构成的`channel`
- `reorgShutdownCh` 传输`reorg`停止信号

一个简单的示例图，如下:

![scheduleReorgLoop](https://s-bj-3358-blog.oss.dogecdn.com/svg/scheduleReorgLoop.drawio.svg)

在这些`channel`中，较难理解的是`reorgDoneCh`和`reorgShutdownCh`，这两个变量的设计是为了保证并发的正确性。我们首先介绍`reorgDoneCh`变量，此变量非常奇怪属于`chan chan struct{}`类型。

对于任何一个`channel`，分析其作用的最好方法就是分析其数据发送者和数据接收者。`reorgDoneCh`的数据发送者代码如下:
```go
case req := <-pool.reqResetCh:
	// Reset request: update head if request is already pending.
	if reset == nil {
		reset = req
	} else {
		reset.newHead = req.newHead
	}
	launchNextRun = true
	pool.reorgDoneCh <- nextDone

case req := <-pool.reqPromoteCh:
	// Promote request: update address set if request is already pending.
	if dirtyAccounts == nil {
		dirtyAccounts = req
	} else {
		dirtyAccounts.merge(req)
	}
	launchNextRun = true
	pool.reorgDoneCh <- nextDone
```
上述代码均来自`scheduleReorgLoop`函数内，我们可以看到都是在进行一系列数据处理后在进行推送`nextDone`。其中`nextDone`的定义为`make(chan struct{})`，是符合`channel`的类型要求的。

> 上文给出的`case`代码块内，我们可以看到`launchNextRun`被设置为`true`，这意味着重排会立即进行。当然，此时进行的重排也会对上文介绍的`queuedEvents`中的内容一并进行重排。

我们进一步分析数据接收者，代码如下:
```go
func (pool *TxPool) requestReset(oldHead *types.Header, newHead *types.Header) chan struct{} {
	select {
	case pool.reqResetCh <- &txpoolResetRequest{oldHead, newHead}:
		return <-pool.reorgDoneCh
	case <-pool.reorgShutdownCh:
		return pool.reorgShutdownCh
	}
}

func (pool *TxPool) requestPromoteExecutables(set *accountSet) chan struct{} {
	select {
	case pool.reqPromoteCh <- set:
		return <-pool.reorgDoneCh
	case <-pool.reorgShutdownCh:
		return pool.reorgShutdownCh
	}
}
```
这些接收函数均直接选择将`pool.reorgDoneCh`内的空`channel`作为`return`返回，如果读者进一步研究这两个函数的应用会发现函数的`return`值并没有被具体的运行逻辑使用。

造成这种情况的原因是`reorgDoneCh`的目的仅是保证`reqResetCh`和`reqPromoteCh`函数发送给`reqResetCh`和`reqPromoteCh`的数据会被`scheduleReorgLoop`正确处理后关闭。更加详细的解释是当我们通过`return <-pool.reorgDoneCh`获得一个`channel`(即`nextDone`)时，由于`channel`自身具有阻塞性，主函数只有在`scheduleReorgLoop`进行完数据处理(即上文给出的`case`块)运行后退出。这一行为有效保障函数运行的同步。这种运行逻辑与`async/await`类似，在`golang`中，类似`reorgDoneCh`的`chan chan struct{}`是一种重要的无锁队列结构，

> 假如我们不进行`reorgDoneCh`队列操作，那么使用`requestPromoteExecutables`的`addTxs`函数就可以无视`scheduleReorgLoop`的数据处理流程而自行工作，这可能导致数据在`scheduleReorgLoop`进行数据处理操作时被推入函数，造成并发冲突。

> `reorgDoneCh`代表的`chan chan struct{}`是 无锁 Channel 的重要实现方式，其他实现方式可以参考[Go channels on steroids](https://docs.google.com/document/d/1yIAYmbvL3JxOKOjuCyon7JhW4cSv1wy5hC0ApeGMV9s/pub)。如果读者想进一步深入学习，建议阅读[Go语言设计与实现](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/#%E6%97%A0%E9%94%81%E7%AE%A1%E9%81%93)。


当然，读者可以分析`nextDone`也是一个`channel`，在`addTxs`函数中，我们使用了以下代码:
```go
done := pool.requestPromoteExecutables(dirtyAddrs)
if sync {
	<-done
}
```
此代码从`done`中获取成员，可以保证在`addTxs`一定在`runReorg`运行完后结束。

> 上述流程都使用了`channel`的阻塞特性，如果读者对于`go`语言的此部分不熟悉，可以参考[Go Channel 详解](https://colobu.com/2016/04/14/Golang-Channels/)

另一个较难理解的队列为`reorgShutdownCh`队列，此队列用于系统关闭。但我们分析此队列没有数据发送者，但存在大量的数据接收者，如下:
```go
case <-pool.reorgShutdownCh:
	// Wait for current run to finish.
	if curDone != nil {
		<-curDone
	}
	close(nextDone)
	return
```
以上代码块来自`scheduleReorgLoop`函数，此函数尝试早`reorgShutdownCh`队列中获得内容，但由于没有数据发送者，这会导致阻塞情况的发送，此`case`不会被激活。继续查找相关代码，我们可以查询到以下代码:
```go
case <-pool.chainHeadSub.Err():
	close(pool.reorgShutdownCh)
	return
```
此代码的关键在于`close(pool.reorgShutdownCh)`，在`go`语言中，当我们关闭`channel`时，`channel`的阻塞状态消失，接收者可以在`channel`中获得`nil`类型的成员。这意味着当我们关闭`reorgShutdownCh`时，上文给出的`case <-pool.reorgShutdownCh`会被激活，在此处依旧使用了`channel`的阻塞特性，通过阻塞`curDone`空队列保证了`runReorg`运行结束后，再进行`return`操作。

在上文中，我们讨论了`scheduleReorgLoop`与一系列`channel`之间的相互关系和数据流动。简单来说，所有对于交易重排的数据都会使用`channel`推送给的`scheduleReorgLoop`函数，此函数通过`select`结构进行分类处理，这些处理后的数据最终都会被用于一个关键函数，也是我们下文主要介绍的函数，即`runReorg`函数。

在`runReorg`的开始设定了一些用于记录和关闭`channel`的`defer`部分，由于这些内容并不重要且主要涉及`golang`语言特性，所以在此处为我们直接跳过此部分。同时，在下文中我们认为`reset`结构体为空，因为此结构体主要用于接受新的区块后进行交易池更新，与我们此处设定的单笔交易进入交易池流程不符，我们会在本文的在介绍交易池更新时给出对于此部分的解析。

我们首先给出函数的定义，代码如下:
```go
func (pool *TxPool) runReorg(done chan struct{}, reset *txpoolResetRequest, dirtyAccounts *accountSet, events map[common.Address]*txSortedMap)
```
各参数含义如下:

1. `done` 用于阻塞状态的`channel`，实现`addTxs`中的`sync`作用
1. `reset` 用于区块更新后刷新交易池的结构体，在本节我们不会详细说明
1. `dirtyAccounts` 此变量为需要提权交易的账户集合，由上文给出的`addTxs`函数增加账户
1. `events` 用于广播的替换交易，其成员主要来自上文给出的`add`函数中的替换交易池内已有交易部分

我们首先处理对账户集合内的账户进行提权的过程。代码如下:
```go
var promoteAddrs []common.Address
if dirtyAccounts != nil && reset == nil {
	promoteAddrs = dirtyAccounts.flatten()
}
pool.mu.Lock()
promoted := pool.promoteExecutables(promoteAddrs)
```
我们在上述代码中省略了`reset`部分。其中的核心函数是`promoteExecutables`，其代码及对应的注释如下:
```go
func (pool *TxPool) promoteExecutables(accounts []common.Address) []*types.Transaction {
	var promoted []*types.Transaction

	// 循环 accounts 并寻找可提权交易
	for _, addr := range accounts {
		list := pool.queue[addr]
		if list == nil {
			continue // Just in case someone calls with a non existing account
		}
		// 丢弃所有低于当前状态账户 nonce 的交易
		forwards := list.Forward(pool.currentState.GetNonce(addr))
		for _, tx := range forwards {
			hash := tx.Hash()
			pool.all.Remove(hash)
		}
		log.Trace("Removed old queued transactions", "count", len(forwards))
		// 丢弃所有账户余额无法满足 gas 消耗的交易
		drops, _ := list.Filter(pool.currentState.GetBalance(addr), pool.currentMaxGas)
		for _, tx := range drops {
			hash := tx.Hash()
			pool.all.Remove(hash)
		}
		log.Trace("Removed unpayable queued transactions", "count", len(drops))
		queuedNofundsMeter.Mark(int64(len(drops)))

		// 使用 pool.pendingNonces.get(addr) 获得 pending 队列中的最大 nonce
		// 使用 list.Ready 在 quenue 队列中获得上述 nonce 后的交易
		readies := list.Ready(pool.pendingNonces.get(addr))
		for _, tx := range readies {
			hash := tx.Hash()
			// 使用 pool.promoteTx 函数对交易进行提权
			// 即将交易自 quenue 转移到 pending
			if pool.promoteTx(addr, hash, tx) {
				promoted = append(promoted, tx)
			}
		}
		log.Trace("Promoted queued transactions", "count", len(promoted))
		queuedGauge.Dec(int64(len(readies)))

		// 判断用户的 quenue 队列中交易数量是否符合需求
		// 如不符合则删除部分交易
		var caps types.Transactions
		if !pool.locals.contains(addr) {
			caps = list.Cap(int(pool.config.AccountQueue))
			for _, tx := range caps {
				hash := tx.Hash()
				pool.all.Remove(hash)
				log.Trace("Removed cap-exceeding queued transaction", "hash", hash)
			}
			queuedRateLimitMeter.Mark(int64(len(caps)))
		}
		// Mark all the items dropped as removed
		pool.priced.Removed(len(forwards) + len(drops) + len(caps))
		queuedGauge.Dec(int64(len(forwards) + len(drops) + len(caps)))
		if pool.locals.contains(addr) {
			localGauge.Dec(int64(len(forwards) + len(drops) + len(caps)))
		}
		// Delete the entire queue entry if it became empty.
		if list.Empty() {
			delete(pool.queue, addr)
			delete(pool.beats, addr)
		}
	}
	return promoted
}
```

综上所述，对于交易在`quenue`到`pending`的转换并没有及其严格的审查，更不会校验交易中的内容是否可以正常运行。

可能有读者发现在`promoteExecutables`函数内，我们没有对交易池`pending`队列长度等内容进行检测，原因在于这一部分是通过`truncate`系列函数实现的，代码如下:
```go
pool.truncatePending()
pool.truncateQueue()
```
限于篇幅，我们无法具体分析这两个函数的代码实现，但我们仍会给出相关的逻辑实现。

`truncatePending`会将所有交易数量超过交易池限制`AccountSlots`且不在本地账户的账户地址构成一个以交易数量为权重的`prque`。完成队列构建后，代码会对此队列按权重进行循环，即权重较大者首先进入循环，并在每一次循环中，将权重较大的部分交易数量依次减去 1 ，直至总的交易数量满足交易池要求为止。值得注意的是，此流程没有考虑账户本身的交易数量限制。

如果经过上述循环依旧不满足交易池要求，则以账户最大可执行交易数量为限制进行循环，直至满足要求。

> 建议阅读[源代码](https://github.dev/ethereum/go-ethereum/blob/master/core/tx_pool.go#L1398)理解部分

`truncatePending`函数删除账户交易时，首先按账户进入交易池的顺序进行排列，较晚进入交易池的交易被优先清理。如果此地址内的交易小于需要丢弃的交易总量，则删除此账户下的所有交易。否则，则仅删除满足要求的交易。

> 关于以上内容，一个较好的参考资料是[以太坊技术与实现](https://learnblockchain.cn/books/geth/part2/txpool/txpromote.html)，读者可以自行参考阅读

完成上述流程后，我们进行数据统计和变量重置工作，代码如下:
```go
dropBetweenReorgHistogram.Update(int64(pool.changesSinceReorg))
pool.changesSinceReorg = 0 // Reset change counter
pool.mu.Unlock()
```
`dropBetweenReorgHistogram`用于统计在两次`reorg`之间升级的交易数量。统计后重置`changesSinceReorg`变量，并释放线程锁。

上述步骤完成了对于`dirtyAccounts`中的交易的提权，我们还需要对`events`中的单个交易进行广播。此过程的代码我们已在上文解释`add`函数中的交易替换部分时给出，此处不再赘述。

## 区块打包

在完成交易进入交易池、交易提升至`pending`队列后，我们需要处理区块打包问题，考虑到文章的专题性，本节不会讨论以下问题:

1. 区块的具体结构和生成方法
1. `PoS`共识算法

本节仅关注交易池与区块打包的对接部分。此部分主要位于`miner/worker.go`文件内，我们所需要介绍内容的核心函数为`fillTransactions`，代码如下:
```go
func (w *worker) fillTransactions(interrupt *int32, env *environment) error {
	// Split the pending transactions into locals and remotes
	// Fill the block with all available pending transactions.
	pending := w.eth.TxPool().Pending(true)
	localTxs, remoteTxs := make(map[common.Address]types.Transactions), pending
	for _, account := range w.eth.TxPool().Locals() {
		if txs := remoteTxs[account]; len(txs) > 0 {
			delete(remoteTxs, account)
			localTxs[account] = txs
		}
	}
	if len(localTxs) > 0 {
		txs := types.NewTransactionsByPriceAndNonce(env.signer, localTxs, env.header.BaseFee)
		if err := w.commitTransactions(env, txs, interrupt); err != nil {
			return err
		}
	}
	if len(remoteTxs) > 0 {
		txs := types.NewTransactionsByPriceAndNonce(env.signer, remoteTxs, env.header.BaseFee)
		if err := w.commitTransactions(env, txs, interrupt); err != nil {
			return err
		}
	}
	return nil
}
```

这一部分代码中使用了`types.NewTransactionsByPriceAndNonce`函数。调用此函数会构造以下类型:
```go
type TransactionsByPriceAndNonce struct {
	txs     map[common.Address]Transactions // Per account nonce-sorted list of transactions
	heads   TxByPriceAndTime                // Next transaction for each unique account (price heap)
	signer  Signer                          // Signer for the set of transactions
	baseFee *big.Int                        // Current base fee
}
```
其中，`heads`属于`TxByPriceAndTime`类型构成堆(`heap`)，其具体定义为`[]*TxWithMinerFee`，进一步`TxWithMinerFee`的定义如下:
```go
type TxWithMinerFee struct {
	tx       *Transaction
	minerFee *big.Int
}
```
此处出现了一个变量`minerFee`，此变量与`gas`费用有关，其具体的计算公式如下:
```
min(GasTipCap, gasFeeCap-baseFee)
等价于
min(maxPriorityFeePerGas, maxFeePerGas-BaseFee)
```

在后文中，我们经常使用`func (*TransactionsByPriceAndNonce).Pop()`函数，此函数会返回交易队列中`minerFee`最大的交易，如果`minerFee`相同则返回发现时间较早的交易。

在完成`NewTransactionsByPriceAndNonce`函数后，我们将使用`commitTransactions`函数，此函数在本节中属于核心地位，我们将着重介绍。

此函数的定义如下:
```go
func (w *worker) commitTransactions(env *environment, txs *types.TransactionsByPriceAndNonce, interrupt *int32) error
```
在此处，我们忽略用于用于传输中断信息的`interrupt`变量，而另一个变量`env`则存储有封装区块所需要的一系列其他参数，在此次我们仍将省略不谈。

接下来，我们分析其代码构成:

第一步，初始化`gas pool`，并限定区块可用`gas`，代码如下:
```go
gasLimit := env.header.GasLimit
if env.gasPool == nil {
	env.gasPool = new(core.GasPool).AddGas(gasLimit)
}
```
其中，函数`AddGas`的功能是提供`gas`限额。关于区块的`GasLimit`的讨论，可以参考[以太坊机制详解:Gas Price计算](https://hugo.wongssh.cf/posts/ethereum-gas#base-fee)中的内容。

接下来，我们会进入到一个`for`循环，此循环没有限定条件，仅能依靠循环体内的`break`跳出。

在此循环内首先检查了`interrupt`变量，我们跳过此部分。然后，检查了`gasPool`的余额，代码如下:
```go
if env.gasPool.Gas() < params.TxGas {
	log.Trace("Not enough gas for further transactions", "have", env.gasPool, "want", params.TxGas)
	break
}
```
如果检测到`gasPool`内的`gas`剩余小于`21000`，我们认为已达到区块的`gasLimit`，跳出循环。

检测完`gas`限制后，我们进一步检测交易队列的情况，若此笔交易位于队列最后，则退出循环。代码如下:
```go
tx := txs.Peek()
if tx == nil {
	break
}
```
其中，`tx.Peek()`会返回`TransactionsByPriceAndNonce`堆中的下一个元素，但与`pop`不同的是此操作不会影响堆的结构。

在完成上述步骤后，我们对一项非常重要的参数进行校验，即判断交易的签名是否符合`EIP155`的规定，关于`EIP155`签名的详细内容，可以参考[基于链下链上双视角深入解析以太坊签名与验证](https://hugo.wongssh.cf/posts/ecsda-sign-chain#%E7%AD%BE%E5%90%8D)。代码如下:
```go
if tx.Protected() && !w.chainConfig.IsEIP155(env.header.Number) {
	log.Trace("Ignoring reply protected transaction", "hash", tx.Hash(), "eip155", w.chainConfig.EIP155Block)

	txs.Pop()
	continue
}
```
如果交易不符合`EIP155`的规定，交易会不配剔除打包序列。

完成基本的校验后，我们准备执行交易，执行交易的代码如下:
```go
env.state.Prepare(tx.Hash(), env.tcount)

logs, err := w.commitTransaction(env, tx)
```
在此段代码内，我们首先初始化`state`，即用来记录状态变化的数据库。在第二行里，我们通过`commitTransaction`正式提交交易。运行交易步骤包含大量的函数调用，限于篇幅，我们无法完整介绍。在此处，我们仅给出一系列函数调用中最核心的代码，如下:
```go
if contractCreation {
	ret, _, st.gas, vmerr = st.evm.Create(sender, st.data, st.gas, st.value)
} else {
	// Increment the nonce for the next transaction
	st.state.SetNonce(msg.From(), st.state.GetNonce(sender.Address())+1)
	ret, st.gas, vmerr = st.evm.Call(sender, st.to(), st.data, st.gas, st.value)
}
```
如果交易涉及合约创建，则调用`st.evm.Create`，否则则调用`st.evm.Call`函数，所以即使交易仅是一笔转账交易，以太坊节点依旧会调用`EVM`，这与一般的认识是不相符的。

完成交易提交后，我们通过`switch`语句处理交易运行失败的各种情况，读者可以阅读相关代码。如果一切正常，我们运行以下代码块:
```go
case errors.Is(err, nil):
	// Everything ok, collect the logs and shift in the next transaction from the same account
	coalescedLogs = append(coalescedLogs, logs...)
	env.tcount++
	txs.Shift()
```
此代码会将日志推送到`coalescedLogs`日志中，然后运行`tx.Shift()`执行下一个交易。

在完成交易的执行后，我们使用以下代码将运行日志作为订阅源供用户使用:
```go
if !w.isRunning() && len(coalescedLogs) > 0 {
	cpy := make([]*types.Log, len(coalescedLogs))
	for i, l := range coalescedLogs {
		cpy[i] = new(types.Log)
		*cpy[i] = *l
	}
	w.pendingLogsFeed.Send(cpy)
}
return nil
```

我们首先对`coalescedLogs`进行复制，然后直接使用`Send`发送订阅源。此订阅源并不是在广播交易，而只是供终端用户使用。

在进行使用前，请确保拥有一个`ws`以太坊节点，在此处，我们使用了`infura`提供的服务。除此之外，读者应安装`ws`的客户端，在此处，我使用了`utws`作为客户端。输入以下命令:
```bash
uwsc wss://mainnet.infura.io/ws/v3/YOUR_API_KEY
```
回车后，在`>`后键入以下内容：
```json
{"jsonrpc":"2.0", "id": 1, "method": "eth_subscribe", "params": ["newPendingTransactions"]}
```
输入如下图:
![utws input](https://img.gejiba.com/images/16165ef03287c0a1f62ed2fa56d147ed.png)

输出如下图:
![utws output](https://img.gejiba.com/images/546871eeed9c3dc3e6785630836f2156.png)

在输出中，`result`代表交易的哈希值，读者可`https://etherscan.io/tx/{result}`形式的网址访问到交易详情。

最后，我们介绍用于区块密封的函数，代码如下:
```go
func (w *worker) generateWork(params *generateParams) (*types.Block, error) {
	work, err := w.prepareWork(params)
	if err != nil {
		return nil, err
	}
	defer work.discard()

	if !params.noTxs {
		interrupt := new(int32)
		timer := time.AfterFunc(w.newpayloadTimeout, func() {
			atomic.StoreInt32(interrupt, commitInterruptTimeout)
		})
		defer timer.Stop()

		err := w.fillTransactions(interrupt, work)
		if errors.Is(err, errBlockInterruptedByTimeout) {
			log.Warn("Block building is interrupted", "allowance", common.PrettyDuration(w.newpayloadTimeout))
		}
	}
	return w.engine.FinalizeAndAssemble(w.chain, work.header, work.state, work.txs, work.unclelist(), work.receipts)
}
```
目前以太坊已经完成了合并，所以此处最终的出块是由`engine`完成，即共识引擎。

简单给出一个当前以太坊的节点架构图:
![Ethereum Node Chart](https://img.gejiba.com/images/83779fa7aa66b0a16853c39d8cf281a6.png)

简单来说，当共识客户端(`Consensus Client`)被选为区块提案者(`proposer`)后，会在执行客户端(`Exection Client`)的交易池内筛选交易，并由执行客户端运行交易。最终，共识客户端将打包后的交易进行广播，由其他节点进行投票，最终确定一个区块。

> 目前对于合并后的基础架构，概览性资料较少，我目前仍在研究相关内容，上述的流程可能有错误。如果您发现错误，请通过[我的博客](https://hugo.wongssh.cf)给出的邮箱与我联系。

在此处，我们基本完成了一笔交易在以太坊执行层内的完整流程，接下来，我们介绍当区块到达交易池后，交易池重构的相关内容。

## 交易池重构

我们在上文仅考虑了节点直接生产区块的区块，但在实际情况中，节点更有可能无法生产区块，而仅仅作为区块的接受方，接受其他节点生产的区块。我们势必限需要讨论节点在接受到其他节点发送的区块时如何进行交易池重构的问题 。

在交易池主循环内，我们可以找到如下代码:
```go
case ev := <-pool.chainHeadCh:
	if ev.Block != nil {
		pool.requestReset(head.Header(), ev.Block.Header())
		head = ev.Block
	}
```
当交易池在`chainHeadCh`内获得新的区块头后，交易池会启动`requestReset`函数，此函数我们在上文已有所介绍，`requestReset`函数会将请求发送到`reqResetCh`的通道内，最终由`scheduleReorgLoop`函数接受，代码如下:
```go
case req := <-pool.reqResetCh:
	// Reset request: update head if request is already pending.
	if reset == nil {
		reset = req
	} else {
		reset.newHead = req.newHead
	}
	launchNextRun = true
	pool.reorgDoneCh <- nextDone
```

实际最终还是由`runReorg`运行，我们在此处仅接受上文未介绍的`Reset`部分，第一部分的代码如下:
```go
if reset != nil {
	// Reset from the old head to the new, rescheduling any reorged transactions
	pool.reset(reset.oldHead, reset.newHead)

	// Nonces were reset, discard any events that became stale
	for addr := range events {
		events[addr].Forward(pool.pendingNonces.get(addr))
		if events[addr].Len() == 0 {
			delete(events, addr)
		}
	}
	// Reset needs promote for all addresses
	promoteAddrs = make([]common.Address, 0, len(pool.queue))
	for addr := range pool.queue {
		promoteAddrs = append(promoteAddrs, addr)
	}
}
```
我们首先介绍除了`reset`外的其他部分，在完成`reset`函数后，我们首先删除`events`(内部含有准备加入`quenue`队列的交易)内所有小于当前状态数据内的`nonce`的交易。小于当前状态数据库内的`nonce`意味着此交易已被打包，不需要进行进一步处理。

然后，我们将当前交易池内`quenue`队列中的所有交易列入`promoteAddrs`中，这意味着在之后的代码运行中，这些交易均会被升级为`pending`队列内的交易。

最后，我们介绍较为复杂的`reset`函数，此函数负责向交易池内提交差异交易，而不负责删除交易，删除交易的代码我们会在第二部分进行介绍。

我们首先介绍第一个`if`语句内的交易:
```go
if oldHead != nil && oldHead.Hash() != newHead.ParentHash {
	// 获得区块编号
	oldNum := oldHead.Number.Uint64()
	newNum := newHead.Number.Uint64()
	// 当新旧区块差异过大时，不进行处理
	// 此种区块一般发生在节点同步数据时
	if depth := uint64(math.Abs(float64(oldNum) - float64(newNum))); depth > 64 {
		log.Debug("Skipping deep transaction reorg", "depth", depth)
	} else {
		// 声明需要丢弃和包含的交易
		var discarded, included types.Transactions
		var (
			// 获取区块完整数据
			rem = pool.chain.GetBlock(oldHead.Hash(), oldHead.Number.Uint64())
			add = pool.chain.GetBlock(newHead.Hash(), newHead.Number.Uint64())
		)
		if rem == nil {
			// 此情况属于特殊情况，即旧区块无法检索
			if newNum >= oldNum {
				log.Warn("Transaction pool reset with missing oldhead",
					"old", oldHead.Hash(), "oldnum", oldNum, "new", newHead.Hash(), "newnum", newNum)
				return
			}
			log.Debug("Skipping transaction reset caused by setHead",
				"old", oldHead.Hash(), "oldnum", oldNum, "new", newHead.Hash(), "newnum", newNum)
		} else {
			for rem.NumberU64() > add.NumberU64() {
				discarded = append(discarded, rem.Transactions()...)
				if rem = pool.chain.GetBlock(rem.ParentHash(), rem.NumberU64()-1); rem == nil {
					log.Error("Unrooted old chain seen by tx pool", "block", oldHead.Number, "hash", oldHead.Hash())
					return
				}
			}
			for add.NumberU64() > rem.NumberU64() {
				included = append(included, add.Transactions()...)
				if add = pool.chain.GetBlock(add.ParentHash(), add.NumberU64()-1); add == nil {
					log.Error("Unrooted new chain seen by tx pool", "block", newHead.Number, "hash", newHead.Hash())
					return
				}
			}
			for rem.Hash() != add.Hash() {
				discarded = append(discarded, rem.Transactions()...)
				if rem = pool.chain.GetBlock(rem.ParentHash(), rem.NumberU64()-1); rem == nil {
					log.Error("Unrooted old chain seen by tx pool", "block", oldHead.Number, "hash", oldHead.Hash())
					return
				}
				included = append(included, add.Transactions()...)
				if add = pool.chain.GetBlock(add.ParentHash(), add.NumberU64()-1); add == nil {
					log.Error("Unrooted new chain seen by tx pool", "block", newHead.Number, "hash", newHead.Hash())
					return
				}
			}
			reinject = types.TxDifference(discarded, included)
		}
	}
}
```

我们在源代码中给出了部分注释，但对于一些特殊情况，我们在此进行解释。

`rem == nil`情况，此情况较为特殊，我们无法在数据库内检索到旧的区块，此种情况可细分:

1. `newNum >= oldNum` 新区块编号大于或等于旧区块，此种情况下意味着我们之前使用的链并不位于主链上，而是位于分支链上。此种情况下，我们不需要对交易池进行特别处理，等待再获得一个区块重置变量即可。
1. 其他情况，这些情况都较为玄学，可能是函数运行出现问题，也可以通过等待区块重置变量解决问题

除了上面这种情况，我们还会遇到以下情况:

1. 旧区块大于新区块 这意味着节点获得了一个可能来自分支链的块广播，我们将旧区块的交易列入`discarded`序列内。正如上文所述，此函数其实不会丢弃交易，`discarded`仅作为交易序列名存在。我们认为旧区块内的交易都应该删除，无论旧区块是否位于主分支等情况

1. 新区块大于旧区块 正常情况，将新区块内的交易列入`included`列表内

> 在当前以太坊`PoS`情况下，很难出现分支链等情况

当然，只要新区块与旧区块的哈希值不同，我们就需要处理其内部的交易，即`rem.Hash() != add.Hash()`情况，在这种情况下，我们将旧区块内的交易列入`discarded`，并将新区块内的交易列入`included`。我们也做了两个校验，代码如下:
```go
rem = pool.chain.GetBlock(rem.ParentHash(), rem.NumberU64()-1); rem == nil
```
上述代码用于判断旧区块是否存在上一个区块，如果没有，则说明此区块不可信，应该直接抛弃。

还有一种极其特殊的情况，代码如下:
```go
if newHead == nil {
	newHead = pool.chain.CurrentBlock().Header() // Special case during testing
}
```
正如注释，此情况仅用于测试。

在完成上述步骤后，我们通过`reinject = types.TxDifference(discarded, included)`获得两个区块交易集合之间的差集。这也是我们需要补充到交易池内的交易，并同时根据新的区块更新部分设置，代码如下:
```go
statedb, err := pool.chain.StateAt(newHead.Root)
if err != nil {
	log.Error("Failed to reset txpool state", "err", err)
	return
}
pool.currentState = statedb
pool.pendingNonces = newTxNoncer(statedb)
pool.currentMaxGas = newHead.GasLimit

// Inject any transactions discarded due to reorgs
log.Debug("Reinjecting stale transactions", "count", len(reinject))
senderCacher.recover(pool.signer, reinject)
pool.addTxsLocked(reinject, false)

// Update all fork indicator by next pending block number.
next := new(big.Int).Add(newHead.Number, big.NewInt(1))
pool.istanbul = pool.chainconfig.IsIstanbul(next)
pool.eip2718 = pool.chainconfig.IsBerlin(next)
pool.eip1559 = pool.chainconfig.IsLondon(next)
```

上述内容基本就是根据新区块对各个变量进行重新设置，较为简单，不再赘述。

> 代码中出现的`statedb`是以太坊的状态数据库，存储有账户等信息，我们会在未来介绍

接下来，我们分析`runReorg`的第二部分，此部分会删除部分交易。代码如下:
```go
if reset != nil {
	pool.demoteUnexecutables()
	if reset.newHead != nil && pool.chainconfig.IsLondon(new(big.Int).Add(reset.newHead.Number, big.NewInt(1))) {
		pendingBaseFee := misc.CalcBaseFee(pool.chainconfig, reset.newHead)
		pool.priced.SetBaseFee(pendingBaseFee)
	}
	// Update all accounts to the latest known pending nonce
	nonces := make(map[common.Address]uint64, len(pool.pending))
	for addr, list := range pool.pending {
		highestPending := list.LastElement()
		nonces[addr] = highestPending.Nonce() + 1
	}
	pool.pendingNonces.setAll(nonces)
}
```
我们首先分析`demoteUnexecutables`函数，代码如下:
```go
func (pool *TxPool) demoteUnexecutables() {
	// 迭代交易池内的 pending 队列
	for addr, list := range pool.pending {
		nonce := pool.currentState.GetNonce(addr)

		// 删除交易中 nonce 较低的交易
		olds := list.Forward(nonce)
		for _, tx := range olds {
			hash := tx.Hash()
			pool.all.Remove(hash)
			log.Trace("Removed old pending transaction", "hash", hash)
		}
		// 丢弃到所有账户无法支付 gas 费用的交易
		// drops 是所有无法支付 gas 费用的交易
		// invalids 是指低于 drops 中最低 nonce 交易的列表
		drops, invalids := list.Filter(pool.currentState.GetBalance(addr), pool.currentMaxGas)
		for _, tx := range drops {
			hash := tx.Hash()
			log.Trace("Removed unpayable pending transaction", "hash", hash)
			pool.all.Remove(hash)
		}
		pendingNofundsMeter.Mark(int64(len(drops)))
		// invalids 内包含的交易可能符合要求，也有可能不符合要求
		// 所以交易应该进入 queued 队列
		for _, tx := range invalids {
			hash := tx.Hash()
			log.Trace("Demoting pending transaction", "hash", hash)

			// Internal shuffle shouldn't touch the lookup set.
			pool.enqueueTx(hash, tx, false, false)
		}
		// 更新计数器
		pendingGauge.Dec(int64(len(olds) + len(drops) + len(invalids)))
		if pool.locals.contains(addr) {
			localGauge.Dec(int64(len(olds) + len(drops) + len(invalids)))
		}
		// 交易列表长度大于 0 ，但交易列表内没有找到 nonce 为最新 nonce 的交易
		// 这意味着交易列表内的所有交易都是可能过时的
		if list.Len() > 0 && list.txs.Get(nonce) == nil {
			// 利用 Cap 返回交易列表内所有交易
			// Cap 的功能是限制列表内的交易数，此处限制为 0
			gapped := list.Cap(0)
			// 执行交易降级
			for _, tx := range gapped {
				hash := tx.Hash()
				log.Error("Demoting invalidated transaction", "hash", hash)

				// Internal shuffle shouldn't touch the lookup set.
				pool.enqueueTx(hash, tx, false, false)
			}
			// 重置计数器
			pendingGauge.Dec(int64(len(gapped)))
			blockReorgInvalidatedTx.Mark(int64(len(gapped)))
		}
		// 如果发现 list(账户地址对应的交易列表)为空，则直接在交易池内删除此账户
		if list.Empty() {
			delete(pool.pending, addr)
		}
	}
}
```
我们通过注释分析了以上代码，此代码的一大特点是在新区块到来时会对`pending`队列中不符合新区块要求的交易进行降级处理。

在完成交易降级流程后，我们使用以下代码进行`nonce`的重置:
```go
nonces := make(map[common.Address]uint64, len(pool.pending))
for addr, list := range pool.pending {
	highestPending := list.LastElement()
	nonces[addr] = highestPending.Nonce() + 1
}
pool.pendingNonces.setAll(nonces)
```
我们通过`list.LastElement()`获得最新的交易，然后将交易的`nonce + 1`作为我们目前跟踪的`nonce`。

> 此处使用的`nonce`是在区块生产期间内使用，所以我们无法通过`statedb`内的数据获得。

## 总结

本文主要介绍了一个交易从构建到打包进入区块的完整过程，基本分析了以太坊交易池的实现和构成，也设计了部分交易打包和执行的内容。主要内容列表如下:

1. 以太坊交易池的基本参数和初始化
1. 交易参数的含义与使用`MetaMask API`构建交易
1. 交易池内交易队列
1. 交易池增加交易使用的函数
1. 交易池内的`scheduleReorgLoop`调度函数及相关`channel`
1. `runReorg`函数实现交易提权的过程
1. 交易打包和执行的基本情况
1. `runReorg`在新区块到达情况下重置交易池状态的情况

考虑到读者可以希望自己阅读源代码，此处给出关于交易的核心函数流程图，为了简单，此流程图省略了部分数据结构，如下:

![Tx Function Flow](https://s-bj-3358-blog.oss.dogecdn.com/svg/txFunction.drawio.svg)
