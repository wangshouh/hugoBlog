---
title: "以太坊机制详解:Gas Price计算"
date: 2022-08-24T15:47:33Z
tags: [ethereum,GAS,EIP-1559]
aliases: ["/2022/08/24/ethereum-gas"]
---
## 概述

在以太坊London升级后，以太坊启用了`EIP1559`进行`gas`计算。由于`EIP1559`引入的新的`gas`机制较为复杂，所以我写了此文介绍了以太坊的`gas`机制。

本文主要涉及以下内容:

- `EIP1559`引入的新的`gas price`设置方式
- 交易花费的具体计算方式

另，此文写作日期在以太坊即将进行合并时，所以我们在后文依旧使用了矿工这一称谓。

## 概念辨析

由于此篇是解析以太坊GAS机制的第一篇，所以我们首先在此处介绍`gas`与`gas price`的区别。

前者是以太坊转账或者合约操作的基准价值。你可以在[此网站](https://www.evm.codes/)查询到每一个操作码的最小GAS消费。如下图:

![OpCode GAS](https://acjgpfqbqr.cloudimg.io/_img1_/93707598f42c435125c76e597f71b8e9.png)

理论上，我们可以通过合约字节码判断出合约操作所需要的`gas`值。当然，如果读者使用了`Foundry`作为智能合约开发工具链，可以在合约代码根目录运行`forge test --gas-report`获得`gas`报告，如下图:

![Foundry Gas Report](https://acjgpfqbqr.cloudimg.io/_img1_/085268c1a57225df3cbee23429cff5ac.png)

上述表格也显示了合约部署消耗的`gas`值。当然，以太坊中也有一种不需要与智能合约交互的但非常重要的操作就是ETH转账，此操作被规定为`21,000`。可以参考[此交易](https://etherscan.io/tx/0xa0d39dbf2d4eff585699bbdf837ed6e0f58158cd2f7bdf4bdc3a94c43d9af5a5)，如下图:

![Transfer Gas](https://acjgpfqbqr.cloudimg.io/_img1_/1c369378906b4e986f439f7b626ee2b1.png)

如果你自定义交易的`gas`最大限额，但设置的数量小于合约操作所需要的`gas`，就会出现错误。比如[这个交易](https://etherscan.io/tx/0x3bc2af543fb45cddd2fc5efda785ab79b5246c7bed353fe57f7668a45a1ee432)，如下图:

![Gas Small Fail](https://acjgpfqbqr.cloudimg.io/_img1_/c60b75cf90f4a062cb65bab13ab111e2.png)

上图由红框框出的部分就是此交易的`gas`限制和`gas`实际用量。此操作实际的`gas`用量为`160,596`，此处的最大限额小于合约操作的用量，所以出现了错误。正常的合约操作可以参考[此交易](https://etherscan.io/tx/0x1acbc0f87338a39972964aee1c487ed7b2047a3ebb71d549e618687087091b2b)。当然此交易虽然失败了，但仍打包到区块内并收取交易手续费并奖励矿工。因为矿工在接受交易时并不清楚交易的`gas`用量，矿工会运行交易直至`gas`耗尽，此部分需要补偿矿工。

> 当Gas的实际用量小于Gas Limit时，剩余部分会退还给用户。

但`gas`并不代表着进行这一操作所消耗的ETH数量。以太坊中存在大量的交易，我们需要根据网络情况调整手续费，为了有效调整手续费，以太坊引入了`gas price`价值作为计算手续费的单位，具体计算公式为
`Transaction Fee = Gas * Gas Price`，其中`Transaction Fee`就是交易手续费的意思。在后文中，我们会详细分析`gas price`的计算方法。

## Gas Limit 的获取

对于Gas Limit的获取，以太坊客户端给出了一个专用的RPC API，被称为`eth_estimateGas`。

此API调用所需要的参数其实就是交易所需要的参数，我们在此处直接给出两个示例帮助大家使用。

在后文中，我们主要使用Cloudflare提供的公用以太坊网关作为RPC API服务商，其地址为`https://cloudflare-eth.com/v1/mainnet`。

为了方便读者学习，此处我们使用[以太坊官方文档](https://ethereum.github.io/execution-apis/api-documentation/)提供的线上测试功能。读者可以通过以下方法打开测试功能:

![Test Console](https://acjgpfqbqr.cloudimg.io/_img1_/7c5f169a91a6e6f6292382d3901df6ef.png)

首先，我们尝试获取转账交易的Gas消耗，在上图给出的测试栏的的左侧输入以下内容:
```json
{
    "jsonrpc": "2.0",
    "method": "eth_estimateGas",
    "params": [
        {
            "from": "0x8D97689C9818892B700e27F316cc3E41e17fBeb9",
            "to": "0xd3CdA913deB6f67967B99D67aCDFa1712C293601",
            "value": "0x186a0"
        }
    ],
    "id": 0
}
```
输入完成后点击运行按钮，我们可以在右侧获得以下返回:
```json
{
    "jsonrpc": "2.0",
    "result": "0x5208",
    "id": 0
}
```
其中，`result`就是此交易的`gas`，将其转为十进制，结果恰好为`21000`，与上文给出的结果相符。

当然，更常见的Gas估计是估计合约操作所消耗的Gas值，我们在此处以WETH合约(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2)为例获取存储`deposit()`操作的Gas消耗。

使用此API的具体参数可以参考以下
```json
{
    "jsonrpc": "2.0",
    "method": "eth_estimateGas",
    "params": [
        {
            "type": "2",
            "from": "0x8D97689C9818892B700e27F316cc3E41e17fBeb9",
            "to": "0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2",
            "value": "0x186a0",
            "input": "0xd0e30db0"
        }
    ],
    "id": 0
}
```
其中各个参数意义如下:

- from 调用合约的用户地址
- to 目标合约地址
- value 在调用合约时发送的ETH
- input 调用合约时发送的Calldata

`input`可以在[此网站](https://abi.hashex.org/)获得。获得`deposit()`函数调用Calldata的形式如下图:

![deposit CallData](https://acjgpfqbqr.cloudimg.io/_img1_/e63757e270f78ea02c0221a08fad722c.png)

> 由于此处`deposit()`没有参数，所以我们没有在此处使用`Add argument`增加参数。

发送上述请求，我们可以获得以下返回值:
```json
{
    "jsonrpc": "2.0",
    "result": "0xafee",
    "id": 0
}
```

将`result`转换为十进制得到`45038`，这与我们在[此页面](https://etherscan.io/tx/0xe433968b74209376c301904cd4c3bdb80afd11f59aa3322db548ae50374656c6)查询得到的结果一致。

对于获取`gas`的估计值，我们也可以使用`cast`获得，在此处，我们仍使用WETH合约。

在终端内输入

```bash
cast estimate 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2 \
	--value 1.1ether "deposit()" \
	--rpc-url https://cloudflare-eth.com/v1/mainnet
```

我们可以获得返回值为`27938`。读者可以发现[此交易](https://etherscan.io/tx/0x5b9b37e98571cf609858be986934f23e495f7766255ec39d9b6a88230e25c01d)的`gas`正是`27938`。

上述两者的不同原因是在[EIP2929](https://eips.ethereum.org/EIPS/eip-2929)。简单来说，`SSTORE`操作符的`gas`决定方式较为特殊。此操作符用于向合约特定的存储槽内写入数据。其`gas`决定方法如下:

- 当写入存储槽本来无数据时，使用`SSTORE`写入数据消耗`22100`。如果读者的地址未持有`WETH`时，我们需要消耗此数值的`gas`
- 当写入存储槽内存在非零数据时，使用`SSTORE`写入数据消耗`5000`。

当我们使用`cast estimate`评估`gas`时，默认使用的地址内存在WETH，而在我们上文使用的 RPC API 时，使用的地址内不持有WETH。更加详细的Gas分析我们会在后面几篇内给出。

## Gas Price计算

我们主要考虑在London升级后的符合`EIP1559`标准的交易，这些交易均被标记为`type 2`。

### 名词解释

在此处，我们给出一个交易的[实例](https://etherscan.io/tx/0xe433968b74209376c301904cd4c3bdb80afd11f59aa3322db548ae50374656c6):

![Tx Example](https://acjgpfqbqr.cloudimg.io/_img1_/d8efb34de3b43f5d6fabf22190108b78.png)

我们主要考察`Gas Price`这一栏。内部由以下构成:

- Gas Limit & Usage by Txn 我们在上文进行了解释，前者表示合约操作的Gas限额，后者表示本次交易的Gas用量
- Gas Fees 给出Gas Price的各个计算参数
    - Base 基础Gas Price
    - Max 最大Gas Price
    - Max Priority 支付给以太坊节点矿工的Gas Price
- Burnt & Txn Savings Fees 燃烧掉的手续费和给予矿工的手续费
    - Burnt 燃烧的手续费。`EIP1559`规定了每次交易的手续费部分进行燃烧，这一行为有效避免了ETH通货膨胀
    - Txn Savings 给予矿工的手续费

我们会在下文给出每个参数的计算方法。

### Base Fee

此参数由以太坊网络计算得到，在同一区块内是固定的。如果你设置的`Base Fee`小于当前网络的`Gas Fee`，则交易永远不会被打包。

我们在此处给出`go-ethereum`的[源代码](https://github.com/ethereum/go-ethereum/blob/master/consensus/misc/eip1559.go#L55):
```go
// CalcBaseFee calculates the basefee of the header.
func CalcBaseFee(config *params.ChainConfig, parent *types.Header) *big.Int {
	// If the current block is the first EIP-1559 block, return the InitialBaseFee.
	if !config.IsLondon(parent.Number) {
		return new(big.Int).SetUint64(params.InitialBaseFee)
	}

	parentGasTarget := parent.GasLimit / params.ElasticityMultiplier
	// If the parent gasUsed is the same as the target, the baseFee remains unchanged.
	if parent.GasUsed == parentGasTarget {
		return new(big.Int).Set(parent.BaseFee)
	}

	var (
		num   = new(big.Int)
		denom = new(big.Int)
	)

	if parent.GasUsed > parentGasTarget {
		// If the parent block used more gas than its target, the baseFee should increase.
		// max(1, parentBaseFee * gasUsedDelta / parentGasTarget / baseFeeChangeDenominator)
		num.SetUint64(parent.GasUsed - parentGasTarget)
		num.Mul(num, parent.BaseFee)
		num.Div(num, denom.SetUint64(parentGasTarget))
		num.Div(num, denom.SetUint64(params.BaseFeeChangeDenominator))
		baseFeeDelta := math.BigMax(num, common.Big1)

		return num.Add(parent.BaseFee, baseFeeDelta)
	} else {
		// Otherwise if the parent block used less gas than its target, the baseFee should decrease.
		// max(0, parentBaseFee * gasUsedDelta / parentGasTarget / baseFeeChangeDenominator)
		num.SetUint64(parentGasTarget - parent.GasUsed)
		num.Mul(num, parent.BaseFee)
		num.Div(num, denom.SetUint64(parentGasTarget))
		num.Div(num, denom.SetUint64(params.BaseFeeChangeDenominator))
		baseFee := num.Sub(parent.BaseFee, num)

		return math.BigMax(baseFee, common.Big0)
	}
}
```

其中`parent`为上一区块的区块头。我们在此处不再详细解释此结构体内的变量，读者可自行查找对应源代码。此处用到的一个重要参数为`parent.GasLimit`，含义为区块内各个交易的Gas累加最大值，读者可以通过[此网站](https://www.etherchain.org/charts/blockGasLimit)查看历史上的`GasLimit`变化。目前(2022年8月)，此值大概为3千万。
```go
Miner: miner.Config{
        GasCeil:  30000000,
        GasPrice: big.NewInt(params.GWei),
        Recommit: 3 * time.Second,
    }
```
当然，区块的`GasLimit`并不是固定不变的，会在小范围内波动，具体的计算逻辑位于`go-ethereum`内的[CalcGasLimit(parentGasLimit, desiredLimit uint64)](https://github.com/ethereum/go-ethereum/blob/master/core/block_validator.go#L108)函数，此函数使用的参数`desiredLimit`即为 3千万 。限于篇幅且此计算函数较为简单，我们不对计算函数进行详细解释，读者有兴趣可以自行研究此函数。

`params.ElasticityMultiplier`值已经在源代码进行了硬编码为`2`。通过`parentGasTarget := parent.GasLimit / params.ElasticityMultiplier`代码，我们可以计算出目前目标区块容量为1.5千万。

`params.InitialBaseFee`此值为`EIP1559`启动时区块的`baseFee`，从后文我们可以看到计算`baseFee`依赖于上一区块的`baseFee`，而初始区块的上一区块没有通过此属性，所以我们需要进行初始化。此变量被初始化为`const InitialBaseFee untyped int = 1000000000`，`1000000000`的单位为`wei`，即`1 gwei`。

```go
if parent.GasUsed == parentGasTarget {
    return new(big.Int).Set(parent.BaseFee)
}
```
此代码说明，当目前区块交易Gas累加值为1.5千万时，区块与上一区块的`Base Fee`相同。这也意味着当前Gas Price很好平衡了交易数量与交易费用，不需要进行调整。

除了这种相同的情况，还有大于和小于的情况，下面先展示上一区块没有大于目标Gas总量的情况。
```
// If the parent block used more gas than its target, the baseFee should increase.
// max(1, parentBaseFee * gasUsedDelta / parentGasTarget / baseFeeChangeDenominator)
num.SetUint64(parent.GasUsed - parentGasTarget)
num.Mul(num, parent.BaseFee)
num.Div(num, denom.SetUint64(parentGasTarget))
num.Div(num, denom.SetUint64(params.BaseFeeChangeDenominator))
baseFeeDelta := math.BigMax(num, common.Big1)

return num.Add(parent.BaseFee, baseFeeDelta)
```
在注释中，我们可以看到当前区块的`baseFee`的计算公式为
```
parent.BaseFee + 
max(1, parentBaseFee * gasUsedDelta / parentGasTarget / baseFeeChangeDenominator)
```
其中各个参数意义如下:

- `parentBaseFee`为`parent.BaseFee`，即上一区块的`baseFee`
- `gasUsedDelta`为`parent.GasUsed - parentGasTarget`，即上一区块的Gas总量与目标总量之间的差额
- `parentGasTarget`为上一区块的目标值，在一定时期内可以认为是常量，目前为1.5千万Gas
- `BaseFeeChangeDenominator`，定义为`const BaseFeeChangeDenominator untyped int = 8`

我们计算极限情况，即当前区块的上一区块的Gas总量到达限额3千万，此时`gasUsedDelta`为`1.5`，`parentGasTarget`为`1.5`，简单计算可以得出当前区块的`BaseFee`应为上一区块的`112.5 %`。

接下来我们使用[Etherscan Blocks](https://etherscan.io/blocks)提供的真实数据进行计算。

![Base Fee Calc](https://acjgpfqbqr.cloudimg.io/_img1_/1c74f515d06a6248dba96dddab985691.png)

我们计算`15406316`区块的`BaseFee`，我们需要参照该区块的上一区块`15406315`的参数进行计算，我们可以看到上一区块的`gasUsedDelta/parentGasTarget`为`+ 11%`，计算得到此时`15406316`的`BaseFee`的值应为`6.38 Gwei * 0.11 / 8`，计算得到`0.885225 gwei`，即`15406316`的`baseFee`应为`6.38 * 0.11 / 8 + 6.38`，计算得到结果为`6.467725`，与`etherscan`给出的相同。

以下给出上一区块Gas总量小于目标总量的代码:
```go
// Otherwise if the parent block used less gas than its target, the baseFee should decrease.
// max(0, parentBaseFee * gasUsedDelta / parentGasTarget / baseFeeChangeDenominator)
num.SetUint64(parentGasTarget - parent.GasUsed)
num.Mul(num, parent.BaseFee)
num.Div(num, denom.SetUint64(parentGasTarget))
num.Div(num, denom.SetUint64(params.BaseFeeChangeDenominator))
baseFee := num.Sub(parent.BaseFee, num)

return math.BigMax(baseFee, common.Big0)
```

根据代码，我们可以得出计算公式如下:
```
parent.BaseFee - 
max(1, parentBaseFee * gasUsedDelta / parentGasTarget / baseFeeChangeDenominator)
```
这意味着如果上一区块的Gas总量为`0`，则当前区块的`baseFee`为上一区块`baseFee`的`87.5 %`。我们不再给出具体的计算过程，读者可自行使用[Etherscan Blocks](https://etherscan.io/blocks)提供的数据进行验算。

`BaseFee`的动态调整可以很好平衡以太坊网络流量，一旦单一区块的交易Gas到达1.5千万，那么根据上述机制，下一区块就会提高`BaseFee`以增加用户的交易手续费，抑制用户交易。反之，当交易需求不足时，以太坊网络则会降低交易手续费以提高用户的交易欲望。

![BaseFee List](https://acjgpfqbqr.cloudimg.io/_img1_/da4f6d11775e878a9b248cb083a4852d.png)

在上图中，我们可以明显考到这一趋势。在`15406535`区块出现了交易Gas为`0`的情况，导致`BaseFee`下降，在下一区块`15406536`则出现了大量交易。

我使用了部分区块的数据绘制了以下图像:

![Gas Block Size](https://acjgpfqbqr.cloudimg.io/_img1_/4bcdd78419d10d5ee9644f20e6aa204c.png)

在此图像中，条形图展示了区块的大小，而折线图展示了`Base Fee`的变化，我们可以很明显的看出`Base Fee`对区块大小的调整作用。

> 此图主要使用了`eth_getBlockByNumber`方法获得区块数据。

根据`EIP1559`规定，`baseFee`不归属于矿工而会被直接燃烧。这种燃烧行为有效避免ETH通货膨胀。通过[Etherscan EIP1559 Dashboard](https://bi.etherscan.io/public/dashboards/ORfoxXZXVdCGQ4ShYL2Ndk7ji6n0hLy9RwSrvt4w)可以获得对应的数据，如下图:

![Gas Burn Chart](https://acjgpfqbqr.cloudimg.io/_img1_/9d921b07c858f62e3b34dcc9d642f6e4.png)

在作者写作此文的过程中，ETHW项目作为以太坊合并后的POS分支废除了EIP1559，很明显，EIP1559没有将所以的手续费分配给矿工的行为不被部分以太坊矿工认可。

### Max Priority Fee

在此交易的[实例](https://etherscan.io/tx/0xe433968b74209376c301904cd4c3bdb80afd11f59aa3322db548ae50374656c6)中，我们可以看到`Max Priority`为`1 Gwei`。相比于上文给出的`BaseFee`而言，此变量完全由交易者自己规定，而不涉及计算问题。`Max Priority Fee`与`Base Fee`不同，此手续费完全交给矿工。所以此值越高则意味着被提前打包的概率越大。

此数值可以通过交易内存池(`mempool`)中的交易数据进行推测，目前市面由很多网站提供`Max Priority Fee`的参考数值，比如:

- [Etherscan GasTracker](https://etherscan.io/gastracker)
- [BlockNative GasEsmator](https://www.blocknative.com/gas-estimator)

我们在此处以[BlockNative](https://www.blocknative.com/gas-estimator)提供的数据为例，如下图:

![BlockNative Gas](https://acjgpfqbqr.cloudimg.io/_img1_/4918fcc21584a8c41381008e5d62e1d5.png)

BlockNative显示了在当前区块确认交易所需要的`Priority Fee`和`Max Fee`以及当前区块的`Base Fee`。关于`Max Fee`的设置，我们会在下文进行介绍。

此处我们以`MetaMask`为例(版本为`10.18.3`)，给出`EIP1559`的设置方法。在进行转账或其他操作时，我们可以点击`编辑`，如下图:

![MetaMask Edit Gas](https://acjgpfqbqr.cloudimg.io/_img1_/3fb286a48816867f1c197a17e833988a.png)

在弹出页面内选择`高级选项`，我们就可以手动调整各个参数，如下图:

![MetaMask Advance](https://acjgpfqbqr.cloudimg.io/_img1_/208e9f86a2c2ff2c4a71faf234b05133.png)

由于此处为转账操作，所以`燃料限制`，即`Gas Limit`为`21000`。其他数值我们可以自行调整。一般来说，`MetaMask`填入的默认数值是可以直接使用的，但当遇到铸造NFT等场景时，我们可以手动调高`Max Priority Fee`以提高铸造成功率。

有了以上参数，我们可以计算具体的交易手续费。我们仍是使用[示例交易](https://etherscan.io/tx/0xe433968b74209376c301904cd4c3bdb80afd11f59aa3322db548ae50374656c6)为大家介绍。

我们可以看到此交易的`Base`为`7.326319867 Gwei`，而`Max Priority`为`1 Gwei`。将上述两个数累加即`gas price`，此处计算得到`8.326319867 Gwei`。然后我们将`gas price * gas`，即`8.326319867 * 45038`，得到此交易的手续费为`375000.79416994605 Gwei`，基本与`Transaction Fee`的值一致。

### Max Fee

我们最后介绍`Max Fee`。此数值规定交易的最大`gas price`。可能有读者会疑惑，我们已经设置了`Base Fee`和`Max Priority Fee`，为什么还需要`Max Fee`？

原因在于用户提交给以太坊节点的交易不一定在下一个区块内完成。如果读者还记得上文给出的`BaseFee`就知道此数值是随着区块Gas总量不断变化的。假如我们根据区块0计算出下一区块1的`BaseFee`为`7 Gwei`，同时手动设置了`Max Priority Fee`为`1 Gwei`，由于我们给出的矿工小费太少，我们的交易会进入打包序列但可能无法在区块1内打包。只能等待区块2进行打包，但极有可能出现区块2的`BaseFee`为`7.875 Gwei`高于区块1，我们给出的`BaseFee`小于区块2的`BaseFee`，此时交易会被直接抛弃，造成交易失败。

如果我们给出`Max Fee`参数为`9 Gwei`，当交易进入区块2时，区块2会根据`Max Fee`计算出我们可以承受的`Base Fee`为`Max Fee - Max Priority Fee`即`8 Gwei`，此数值大于区块2的`Base Fee`，交易仍会保存在序列中等待打包。

简单来说，`Max Fee`的设置可以保证交易不会在未来几个区块内因为`Base Fee`设置过低问题而被抛出打包序列。此数值设置越高，你的交易会在打包序列中保存的时间越长，避免因手续费问题而交易失败。

比如这个Binance的[交易](https://etherscan.io/tx/0x8d01809e07db59b680afc3e25efc97e635492cc9be4838e0e6931a0b39625859)给出了超高的`Max Fee`，彻底避免在因`Base Fee`而出现交易失败的问题。

读者可以估计以下自己目标交易在几个区块内完成，然后设置`Max Fee`。当然，`BlockNative`提供了一种简单的计算方法，公式如下:
```
Max Fee = (2 * Base Fee) + Max Priority Fee
```

这种计算方法可以保证即使用户遇到连续6个满区块(即区块Gas总额均达到3千万)仍可以保证交易不会被提出打包序列。

> 连续6个满区块会导致相对于当前的`BaseFee`的`(1.125)^6`，计算可知此倍数为`2.027`

读者可以根据自己的情况设置`Max Fee`。但不建议`Max Fee`与`Base Fee`的值差距较小，这可能会导致交易无法完成。

## 总结

本篇主要介绍了以下内容:

- 以太坊中的`Gas`、`Gas Price`、`Transaction Fee`之间的区别
- EIP1559 中各个参数的计算方法和功能

我们可以通过下图简单总结本文:

![Gas Picture](https://blogimage.4everland.store/gas.drawio.svg)

如果读者可以很好的理解本篇文章，且想进一步了解以太坊交易，可以参考[以太坊机制详解:交易与交易池]({{<ref "ethereum-transcation" >}})

