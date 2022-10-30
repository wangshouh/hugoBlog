---
title: "AAVE交互指南"
date: 2022-10-15T11:47:33Z
tags: [aave,defi]
aliases: ["/2022/10/09/aave-interactive/"]
---

{{< math.inline >}}

<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/katex@0.16.2/dist/katex.min.css" integrity="sha384-bYdxxUwYipFNohQlHt0bjN/LCpueqWz13HufFEV1SUatKs1cm4L6fFgCi1jT643X" crossorigin="anonymous">
<script defer src="https://cdn.jsdelivr.net/npm/katex@0.16.2/dist/katex.min.js" integrity="sha384-Qsn9KnoKISj6dI8g7p1HBlNpVx0I8p1SvlwOldgi3IorMle61nQy4zEahWYtljaz" crossorigin="anonymous"></script>
<script defer src="https://cdn.jsdelivr.net/npm/katex@0.16.2/dist/contrib/auto-render.min.js" integrity="sha384-+VBxd3r6XgURycqtZ117nYw44OOcIax56Z4dCRWbxyPt0Koah1uHoK0o4+/RRE05" crossorigin="anonymous"></script>
<script>
    document.addEventListener("DOMContentLoaded", function() {
        renderMathInElement(document.body, {
          // customised options
          // • auto-render specific keys, e.g.:
          delimiters: [
              {left: '$$', right: '$$', display: true},
              {left: '$', right: '$', display: false},
              {left: '\\(', right: '\\)', display: false},
              {left: '\\[', right: '\\]', display: true}
          ],
          // • rendering keys, e.g.:
          throwOnError : false
        });
    });
</script>

{{</ math.inline >}}

## 概述

本文主要介绍如何在`AAVE`内进行存款、借贷等基本操作，为读者进一步研究`AAVE`源代码奠定坚实的基础。

本文要求读者具有以下条件:

1. 熟悉以太坊钱包使用
1. 拥有一定的`GoerliETH`测试网`ETH`，可前往[Goerlin Faucet](https://goerlifaucet.com/)进行领取

上述条件可以保证读者可以理解本文的大部分内容，由于笔者本人具有较强的技术背景且未来准备编写`AAVE`智能合约解析系列文章，所以本文给出了部分合约裸交互(即直接与合约函数交互而并不通过网页)，理解这一部分内容需要读者了解智能合约开发和`cast`使用。如果您不是技术人员，可以酌情跳过此部分。考虑到安全性问题，本文提供的裸交互均为`eth_call`而不涉及真正的交易(需要签名和区块确认的交易)。

> 本文使用`AAVE V3`版本，此版本暂未在以太坊主网部署，在`L2`网络的详细部署地址可以参考[文档](https://docs.aave.com/developers/deployed-contracts/v3-mainnet)

## 初始准备

前往[AAVE DApp](https://app.aave.com/)官网，连结并授权钱包，点击右上角的齿轮符号，开启`Testnet mode`。如下图:

![AAVE Setting](https://img.gejiba.com/images/146138118a3f82db3195f20271276c41.png)

然后，进入[Faucet](https://app.aave.com/faucet/)页面，铸造一部分`Test Assets`，建议铸造`USDC`资产。

## 存入资产

### 存入步骤
完成初始测试资产的铸造后，我们返回到[Dashboard](https://app.aave.com/)页面，在`Assets to supply`栏目内选择`USDC`资产进行`Supply`操作，如图:

![USDC approve](https://img.gejiba.com/images/7e6b3ef60bf541f56e4dc4d6a1ea1ea8.png)

> 点[此](https://goerli.etherscan.io/tx/0xb998288a6d5f9c39d522f449e553fbac1736d39037c6228f32aa09a5f2a55a9c)查看我的交易

点击`Approve to continue`按钮，并在钱包内批准交易。值得注意的是，此笔交易完成后，并不以为这我们完成了`Supply`操作，这仅仅表示我们完成了`ERC-20`的`Approve`操作。我们需要再次点击`Supply USDC`，如图:

![USDC supply](https://img.gejiba.com/images/cdca80c660129dac7ccc92af74b19149.png)

> 点[此](https://goerli.etherscan.io/tx/0xf7bc3325b84af8b169167e9c26ab262ddfa34a15c2155f524a155c8ec3dffacc)查看我的交易

最终，完成后会出现以下对话框:

![Supply Finish](https://img.gejiba.com/images/7dbf1379b67fa25acc2049795eda2fb2.png)

### 存款利率计算

由于`AAVE`属于纯粹的`DeFi`，所有贷款获得盈利将分配给存款用户，所以我们可以通过以下公式进行计算:

$$ {LR}_t = \bar{R_t}{U_t} $$

$$ 
\bar{R_t} = \frac{variable\\ debt}{total\\ debt} * {R_{variable}} + \frac{stable\\ debt}{total\\ debt} * {R_{stable}}
$$

其中，各参数含义如下:

- $\bar{R_t}$ 浮动利率贷款与固定利率贷款的利率加权平均值(以占总贷款数量的比值为权数)
- $U_t$ 流动性资金利用率，即当前流动性池内贷出资产与总资产的比值

上述获得 ${LR}_t$ 实际上属于单利利率，在`AAVE`内，合约采用了每秒进行一次复利的方法进行计算利率，实际复利利率计算公式如下:
$$Actual\ APY = (1+Theoretical\ APY/secsperyear)^{secsperyear}-1$$

我们上文给出的 $R_t$ 即此处的 $Theoretical\ APY$ ，$secsperyear$ 含义为 每年的秒数，计算一下可以得到此值等于`31536000` 

> 如果读者具有金融背景，可以将此按秒复利的方法近似于`连续复利`，通过 
$e ^ {Theoretical\ APY} - 1$ 获得近似值。

> 事实上，贷款收益并不会完全分配给用户，有少部分的利息收入会被计入风险储备金，此部分比例一般较少，在上文中并没有给出相关参数。

事实上，我们可以通过与`AaveProtocolDataProvider`合约中的`getReserveData`函数获得相关数据，具体函数参数和返回值可参考[文档](https://docs.aave.com/developers/core-contracts/aaveprotocoldataprovider#getreservedata)。这里以`Arbitrum`上的`DAI`资产为例介绍参数获得。

直接前往[Arbiscan相关页面](https://arbiscan.io/address/0x69FA688f1Dc47d4B5d8029D5a35FB7a548310654#readContract#F12)操作，在`asset`内输入`DAI`的地址`0xda10009cbd5d07dd0cecc66161fc93d7c9000da1`，我们就可以索引到相关数据，如下图:

![getReverseData](https://img.gejiba.com/images/15ac4264b04fcd2ccab8d2faee35adb3.png)

返回的`liquidityRate`就是我们想要的 ${LR}_t$ ，但由于有限精度小数编码的方法，此处获得的数据需要与 ${10}^{27}$ 相除，即`0.00733376`。将此数值带入上文给出的实际利率计算公式，结果为 

$$ (1 + \frac{0.00733376}{31536000}) ^ {31536000} - 1 = 0.007365$$

> 实际上，为了减少`gas`消耗，`AAVE`合约采用了二项式近似的计算方法，可参考[calculateCompoundedInterest](https://github.com/aave/aave-v3-core/blob/master/contracts/protocol/libraries/math/MathUtils.sol#L51)函数。此处为了方便读者计算而直接采用了指数运算的方法。

上述数据与 `DAPP` 详情页显示一致，如下图:

![Supply APY](https://img.gejiba.com/images/e3302b24f635f12125f1c1605dc82cb2.png)

## 贷出资产

### 贷款目的

对于很多以不熟悉金融的读者来说，很难理解为什么有人以超额质押的形式贷出资产，而不是选择直接兑换资产。我个人总结了以下几种需要贷款的情形，希望帮助读者理解这一行为:

使用`AAVE`进行贷款最大的原因是获得流动性。我们首先给出一个传统金融的例子:假如我们个人维护有一个非常完美的由股票构成的资产组合，我们将所有流动性资产投资于此股票组合。但由于各种原因，我们需要一定量的现金，一种最简单的解决流动性问题的方案是直接出售股票组合内的部分资产，但这种行为意味着我们放弃了未来盈利，机会成本较高。另一种可靠的解决方案是通过券商进行股票的质押换取现金，待未来有了流动性资产后回购股票。通过质押回购股票的方式，我们即没有放弃资产组合，但也获得了流动性。这种交易被称为`融资`交易(与下文介绍的`融券`交易相对)。但质押回购股票要求投资者必须为合格投资者，门槛非常高。

使用`AAVE`的贷款功能可以很好的解决上述问题。我们仍假设维护有一个良好的资产组合，但这些组合由很多以太坊内的`ERC-20`代币构成且盈利稳定。我们遇到流动性危机时，仅需要将资产储蓄在`AAVE`的流动性池内，然后贷出稳定币就可以非常快速的解决流动性问题。且我们可以无许可地使用这套金融系统。

> 上述情况的另一个选择是使用`MarkDAO`通过铸造`DAI`解决问题，此种方式似乎更加常见，这也是为什么`MarkDAO`是目前以太坊内总锁仓量最大的`DeFi`的原因

使用`AAVE`进行贷款的第二个原因是可以进行投机。在传统金融中，尤其在没有做空机制的A股内，当我们需要做空一只股票，除了期货这类可做空的衍生品外，我们另一种途径是`融券`交易，即我们向券商质押一部分流动性较好的资产，然后券商会贷出一部分自己持有的股票。我们可以直接在股票市场上卖出这部分贷出的股票，获得现金。当股票价格下跌后，我们再买入股票归还给券商，并拿回质押品。这种融券交易实现了做空股票的功能。与上文介绍的融资交易相同，此交易也需要投资者具有合格投资者地位，门槛非常高。

在`AAVE`中，我们可以通过把一部分价值较为稳定的资产质押到流动性池内，贷出目标资产，获得目标资产后，我们直接卖出目标资产获得现金。等到目标资产价格下跌到一定水平后，我们使用之前卖出资产获得的现金购买目标资产，偿还贷款，解除质押。显然，这种方式更加优雅且有效。

> 如果读者想获得更多传统金融方面的知识，我建议读者可以尝试考取`证券从业资格证`，此证书可以较为容易地考取，并帮助读者获得金融领域的基础知识

综上所述，我个人认为`AAVE`不应该被称为`Web3`银行，而应该被称为`Web3`融资融券机构，这更符合`AAVE`的功能定位。银行最大的运行特点是`借长贷短`，但`AAVE`并没有设计锁仓机制实现`借长`的目的，在贷出资产方面也没有要求用户需要在指定期限内归还贷款，所以`AAVE`被称为`Web3`银行是一个非常不好的类比。

### 贷款参数
正如在现实世界内的贷款，我们需要认真研究贷款利率、当前资产可贷出的总金额等属性。但在`AAVE`内，由于其用户进行清算的属性，我们需要研究更多的金融属性。在此处，我们假设将`USDC`全部用于质押以贷出`USDT`。

首先，我们应该点击`Assets to borrow`栏目内的`Details`按钮以查询作为抵押品的`USDC`的相关属性。当然，读者可以通过直接点击[此链接](https://app.aave.com/reserve-overview/?underlyingAsset=0xa2025b15a1757311bfd68cb14eaefcc237af5b43&marketName=proto_goerli_v3)直接访问。如图:

![USDC Collateral](https://img.gejiba.com/images/d32de44af963c6dd56f1c0dfef197626.png)

其中，各参数含义如下:

- `APY` 年利率
- `Max LTV` 贷出资产价值与质押品价值的最大比值
- `Liquidation threshold` 流动性阈值，如果抵押资产与贷出资产的价值比率低于此值，会触发清算
- `Liquidation penalty` 资产被清算时，清算人使用被清算人贷出资产购买抵押品的折扣

在前文中，我们存入了 `1000 USDC` 资产，根据`Max LTV`计算出，我们最多可以贷出 `800` 美元的其他资产。由于`USDC`的`APY`过高，我目前持有`1000.21 USDC`，假如兑换`USDT`，根据`Max LTV`计算得到我可以借出最多`800.168 USDT`，查看[USDT Details](https://app.aave.com/reserve-overview/?underlyingAsset=0xc2c527c0cacf457746bd31b2a698fe89de2b6d49&marketName=proto_goerli_v3)页面，数值恰好相符。

![USDT Borrow](https://img.gejiba.com/images/15bcd4d7dbcccf64b58faa20b8bdef2f.png)

接下来，我们计算`Health factor`数值，此数值一旦低于`1`，就会触发清算程序。此数值的计算方法为`流动性阈值(Liquidation threshold) / 贷出资产价值与质押品价值比值`。在此处，我们可以计算出`贷出资产价值与质押品价值比值`为`80 %`(此处我们选择全部贷出，故而数值等同于`Max LTV`)，而`USDC`的`流动性阈值`为`85 %`，我们可以通过`85% / 80%`，结果正好为`1.06`。

一个更加权威的且适合于多种资产联合质押情况下`Health factor`计算公式如下:
$$H_f = \frac{\sum({Collateral_i\ in\ ETH} \times {Liquidation\ Threshold}_i)}{Total\ Borrows\ in\ ETH}$$

各参数含义如下:

- $H_f$ 健康因子，即`Health factor`
- $Collateral_i\ in\ ETH$ 资产 $i$ 以`ETH`计价的价值
- ${Liquidation\ Threshold}_i$ 资产 $i$ 对应的流动性阈值
- $Total\ Borrows$ 用户借出的总资产价值(以`ETH`计价)

上述公式可以计算以资产组合为抵押贷出一系列资产的`Health factor`。

在上文给出的参数中，较难理解的是`Liquidation penalty`的含义，我们通过一个例子向大家介绍此参数的含义。在现实世界，当我们无法支付贷款，那么银行会收走我们的抵押品进行拍卖以收回贷款。在`AAVE`内流程类似，当用户无法偿还贷款时，其抵押品会被出售，但为了激励清算人购买抵押品，抵押品会折价一定比例，此比例即`Liquidation penalty`。当然，这部分资产需要贷款人支付，相当于清算手续费。

> 清算人只能使用被清算人的贷出资产类别进行清算购买

> 贷款人也可以自己赎回质押品，但最多赎回 50%

假如某用户拥有质押资产`10 X`，贷出代币`5 Y`，当前贷款的`Health factor`已经小于`1`，进入清算阶段。假设当前有`1Y = 5X`且`Liquidation penalty`为`5%`，某清算人想购入该用户全部质押资产`X`，其需要支付`10 / 5 * (1 - 5%)`单位`Y`，而该用户需要支付`(1 - 5%)`单位`Y`的清算手续费。值得注意的是，清算人必须以被清算人贷出资产类别(即`Y`)支付购买费用。

在上述案例中，假设清算人决定购买 50% 抵押品。那么清算人需要支付等同于 50% 质押品价值的 95% 的`USDT`(减免的 5% 即`Liquidation penalty`，由质押者支付)。

我们看一笔发生在真实世界的[交易](https://etherscan.io/tx/0x11d0050b5040438b8f95b4a6d07b31656242f30405e5e931d75b2cca19dfc94e)，更加详细的交易调用可以参考[此网站](https://tx.eth.samczsun.com/#txhash=0x11d0050b5040438b8f95b4a6d07b31656242f30405e5e931d75b2cca19dfc94e)。简单来说，质押者在过去以`383.4730870573113 REN`以及其他资产作为质押品贷出了`114.5628671559096 DAI`。但目前，这笔交易已经到达清算阶段。上面给出的交易正是此交易的清算交易之一。清算人决定质押品内所有的`REN`代币，总计`383.4730870573113`。由于被清算人贷出了`DAI`，所以清算人需要支付`DAI`作为代币，且仅需要支付`383.4730870573113 * (1 - 0.075) REN`的`DAI`。

> 此处`0.075`即`Liquidation penalty`，这部分费用由被清算人支付

查阅交易调用记录，我们可以得到在交易发生时，`REN`、`DAI`与`ETH`的兑换比率如下：
``` 
1 REN = 94627327009678 wei ETH
1 DAI = 752403700000000 wei ETH
```
我们计算清算人所需要支付的`DAI`，计算公式如下:
```
383.4730870573113 * 94627327009678 / 752403700000000 * (1 - 0.075)
```
计算结果为`44.61103223941377`，基本与交易输入的`44.86338878495268 DAI`。

> 上述误差可能来自浮点数计算以及结算手续费等问题

读者可以在[eigenphi](https://eigenphi.io/ethereum/liquidation)网站内找到更多清算实例。如下图:

![Liquidation Overiew](https://img.gejiba.com/images/fc1e0d5f9da76eb16035646422edc43d.png)

但值得注意的是，此网站展示的结算过程较为会计化和复杂化，可能很难理解具体的清算流程。如下图:

![Tx Token Flow](https://img.gejiba.com/images/a72dd7fb8591ecf6199b09d7450518c1.png)

这是因为清算人在清算时往往大量使用`uniswap`等工具进行代币兑换，但此部分展示时并没有给出详细的交易执行顺序而仅仅给出交易运行结束后的代币流向，一个更好的理解清算过程的方法是使用`Etherscan`，如下图:

![Etherscan Liquidation](https://img.gejiba.com/images/f182baac558ad604f6a84faf0e90f49f.png)

读者可以通过在`eigenphi`交易详情页的最上方的按钮直接跳转到`Etherscan`，如下图:

![Etherscan Eigenphi](https://img.gejiba.com/images/d30b5fc7d7043692a34a364718fa60c5.png)

> 如果您是一位以太坊技术人员，更希望了解交易的本质，我建议您通过[此网站](https://tx.eth.samczsun.com/)查询清算交易的详细的函数调用情况。

我们在此处也给出不通过`AAVE`网页获得这些参数的方法。如果您不是技术人员，请不要阅读此部分。

在此处我们使用`Arbitrum`网络上的`DAI`流动性池作为案例，页面上的参数如图:

![DAI Args](https://img.gejiba.com/images/f76f7477c7f7c7d50d97da5b524aa3d2.png)

我们可以通过`Pool`合约的`getReserveData`函数获得压缩后的流动性池配置，使用以下命令获得相关数据:
```bash
cast call 0x794a61358D6845594F94dc1DB02A252b5b4814aD "getReserveData(address)(uint256)" 0xda10009cbd5d07dd0cecc66161fc93d7c9000da1 --rpc-url https://rpc.ankr.com/arbitrum
```

> `cast`命令为[Foundry](https://book.getfoundry.sh/)框架携带的，具体可参考[Foundry教程：编写测试部署ERC-20代币智能合约](https://hugo.wongssh.cf/posts/foundry-with-erc20/)。

获得的结果为`379853576081034459698860574047498960397610237566284`，此`uint256`数字中是由流动性池各个设置压缩产生的，我们在此处目标是提取`LTV`、`Liquidation threshold`和`Liquidation penalty`参数。读者可以打开任一程序语言的交互模式(我个人使用的是`Python`)，键入以下命令:
```python
>>> rep = 379853576081034459698860574047498960397610237566284
>>> rep & ~ 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF0000
7500
>>> (rep & ~ 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF0000FFFF) >> 16
8000
>>> (rep & ~ 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF0000FFFFFFFF) >> 32
10500
```

> 命令最开始的`>>>`均为命令提示符，不需要输入

上述输出依次为`LTV`、`Liquidation threshold`和`Liquidation bonus`。其中`Liquidation bonus`相当于`Liquidation penalty + 1`。由于以太坊不支持小数，所以上文的值其实都是设置值乘以`10000`的结果，换算后`LTV`为`0.75`(75%)，`Liquidation threshold`为`0.80`(80%)，`Liquidation bonus`为`1.05`(105%)。通过`Liquidation bonus`可以解算出`Liquidation penalty`为`5%`，这些数据均正确。

我们会在以后解析`AAVE`合约时说明上述位运算的原理。如果读者十分好奇，可以首先参考[此文档](https://docs.aave.com/developers/core-contracts/pool#getreservedata)获得各个二进制位与流动性池参数设置的对应关系可以参考。然后，读者可以直接阅读[此合约](https://github.com/aave/aave-v3-core/blob/master/contracts/protocol/libraries/configuration/ReserveConfiguration.sol#L12)。

### 贷款利率计算

此节主要关注于贷款利率的计算方面。在传统金融中，利率往往通过中央银行给出基准，商业银行在此利率基准上根据资金成本和风险等机制调整利率，较为详细的定价机制可以参考下图:

![Bank Interest rate](https://img.gejiba.com/images/b6ff62812bbc8603bee0bd20b416ef57.jpg)

上图来自`张建波,文竹.利率市场化改革与商业银行定价能力研究[J].金融监管研究,2012,(10):1-13.`

如果读者对此感兴趣，可以参考《商业银行产品定价理论综述》等论文。

但与银行的定价方式不同，在`AAVE`内，我们通过流动性池实现借贷等功能，利率需要尽可能调整借贷比例以实现流动性池的稳定。为了尽可能实现这一目的，`AAVE`设计了一个二阶段的利率曲线，具体图像如下:

![Interest Curve](https://img.gejiba.com/images/fd287385369cc076f547790258ce5d89.png)

在达成流动性池最佳利用率`Optimal`后，借贷利率会迅速上升以避免流动性枯竭。

上面的描述都是定性的，接下来我们进行定量化的分析。为了增强本文的实战性，我们以`AAVE V3`在`Arbitrum`网络上的`DAI`流动性池作为分析对象。

> 在本文编写时，`AAVE V3`暂未部署到以太坊主网上，而仅仅部署在`L2`网络上。

#### 利用率获取

正如上文所说，在资金利用率 $U$ 在`Optimal`之前与之后是两种不同的情况，我们首先计算 $U$ ，计算 $U$ 的公式如下:

$$ U = \frac{Total\ borrowed}{Total\ supplied}$$

其中，参数含义如下:

- `Total borrowed` 该资产总借出数量
- `Total supplied` 该资产总存入数量

这些变量的近似值都可以在[资产详情页](https://app.aave.com/reserve-overview/?underlyingAsset=0xda10009cbd5d07dd0cecc66161fc93d7c9000da1&marketName=proto_arbitrum_v3)内获得，如下图:

![DAI Data](https://img.gejiba.com/images/9666f566a15632f2530174f6826bf638.png)

计算得到的结果为`39.41%`，与页面给出的数据类似:

![DAI Utilization rate](https://img.gejiba.com/images/0a464777646d4b3f5c46eedb75a082c4.png)

此结果误差原因在于页面给出的贷出、存入资产数据精度不足，为了获得更加精确的数据，我们可以通过与合约进行交互。查询文档，我们可以发现`AaveProtocolDataProvider`合约内存在`getATokenTotalSupply`和`getTotalDebt(address)`函数。

我们可以通过以下命令获得`AToken`总量，正如上文所述，用户存入代币后获得`AToken`，所以此处给出的数据相等于用户存储资产总量。
```bash
cast call 0x69FA688f1Dc47d4B5d8029D5a35FB7a548310654 "getATokenTotalSupply(address)(uint256)" 0xda10009cbd5d07dd0cecc66161fc93d7c9000da1 --rpc-url https://rpc.ankr.com/arbitrum
```

> `0xda10009cbd5d07dd0cecc66161fc93d7c9000da1`是`Arbitrum`网络上的`DAI`代币地址

以上命令的返回值为`1411417181377440330599766`，实际等同于`1411417.1813774402`(18位精度)。

> 在智能合约内表示小数最简单的方法就是直接使用整数，并通过精度指定小数点的位置。将整数于精度相除即可获得结果。我们可以将`1411417181377440330599766`与 $10^{18}$ 相除即可获得小数。

我们通过以下命令获得总借出量，命令如下:
```bash
cast call 0x69FA688f1Dc47d4B5d8029D5a35FB7a548310654 "getTotalDebt(address)(uint256)" 0xda10009cbd5d07dd0cecc66161fc93d7c9000da1 --rpc-url https://rpc.ankr.com/arbitrum
```

返回值为`559994183809004825277047`，实际相当于`559994.1838090048`

将上述结果直接相除，获得 $U = 39.6760\% $

> 此值与页面显示的误差已经差别很少，误差可能来自智能合约计算除法时的误差。我们建议在后文使用此数值进行利率计算

#### 浮动利率计算
在具体利率计算中，浮动利率计算较为简单，而固定利率计算较为复杂且依赖于浮动利率计算，所有我们首先分析浮动利率的计算公式，而后分析固定利率计算公式。

我们分析浮动利率确定公式，首先给出流动性利用率 $U < U_{optimal}$ 的情况，具体计算公式如下:

$$R_t = R_0 + \frac{U_t}{U_{optimal}} \times R_{slope1}$$

各参数含义如下:

- $R_t$ 当前利用率 $U_t$ 下的贷款利率
- $R_0$ 初始利率
- $U_t$ 当前流动性池利用率
- $U_{optimal}$ 最佳利用率
- $R_{slope1}$ 预设参数

在 $U > U_{optimal}$ 的情况下，计算公式如下:

$$R_t = R_0 + R_{slope1} + \frac{U_t-U_{optimal}}{1-U_{optimal}} \times R_{slope2}$$

其中，$R_{slope2}$ 为预设参数

接下来，我们介绍各个参数的获取方法:

首先打开[流动性池详情页](https://app.aave.com/reserve-overview/?underlyingAsset=0xda10009cbd5d07dd0cecc66161fc93d7c9000da1&marketName=proto_arbitrum_v3)，将页面滚动到最下方，点击`Interest rate model`框内的`INTEREST RATE STRATEGY`按钮，如下图:

![INTEREST RATE STRATEGY](https://img.gejiba.com/images/4d31ebda9b1abc1f9b5676abe83a60be.png)

点击后会跳转到`Arbiscan`网站，点击`Contract`内的`Read Contract`，如下图:

![Read the Contract](https://img.gejiba.com/images/63f95365ab33038b6060612172b9e683.png)

我们可以在此页面内自由参考各个参数，对应关系如下:

- `OPTIMAL_USAGE_RATIO` 即最佳利用率 $U_{optimal}$ ，注意需要将返回值除以 $10^{27}$ (该数值被称为`RAY`，后文所有利率参数均需要除以此值)， 数值为`0.80`
- `getBaseVariableBorrowRate` 即 $R_0$ ，初始化利率，数值为`0`
- `getVariableRateSlope1` 即 $R_{slope1}$ ，数值为`0.04`
- `getVariableRateSlope2` 即 $R_{slope2}$ ，数值为`0.75`

上述计算的结果仅为单利利率，实际的复利利率计算公式参考[上文](#存款利率计算)

综合以上数据，我们计算一下当前`DAI`的利率，其中 $U_t = 0.3929 $ (该数值直接通过合约数据算出，比页面显示数据更加精确)。

> 注意由于我个人写文章时横跨了两天，此处给出的数据与上文图片给出数据不符是正常的。此处的 $U_t$ 是我根据写作时的最新数据计算出来的，获取方法为合约交互。

首先我们计算 $R_t$ 将数据代入后，结果如下:

$$0 + \frac{0.3929}{80\\%} \times 0.04 = 0.019645$$

这与此时页面显示的`variable APY`是一致的，如下图:

![AAVE APY Data](https://img.gejiba.com/images/bedd5d34dde6f9887c013325d09d1821.png)

读者可以自行验算当 $U > U_{optimal}$ 情况下的`APY`在于曲线数值进行核对，我们此处不再给出计算过程。

#### 固定利率计算
上文给出了浮动利率的计算，我们下文将给出固定利率的计算方法。首先我们需要明确一个观点，即固定利率对于流动性池而言风险更高，固定利率不随流动性池进行波动，导致无法对其进行调控，这会造成极大的风险，所以固定利率应该更高以弥补风险溢价。

> 关于固定利率的计算，在`AAVE`的治理论坛中有着一段[讨论](https://governance.aave.com/t/base-stable-rate-oracle-update-and-improvements-in-aave-v2/1879)。大意为，过于激进的稳定利率不能吸引新的流动性，这抑制了协议的增长，并带来更多的用户体验开销（由于利用率高，提取流动性更困难）。

> 注意此处给出的固定利率计算仅适用于`AAVE V3`版本，而目前仍常用的`AAVE V2`版本固定利率确定依赖于预言机。

为了尽可能减轻固定利率的风险，`aave`合约引入了`最佳固定利率负债总负债比率常数`($O_{ratio}$)进行计算。与 $U$ 类似，当流动性池的`固定利率负债总负债比率常数`($ratio$)大于 $O_{ratio}$ ，我们需要进一步提高利率。

固定利率的基础利率计算公式如下:

$$
R_{base} = \begin{cases}
   R_0 + \frac{U_t}{U_{optimal}} \times R_{slope1} &\text{if } U < U_{optimal} \\\
   R_0 + R_{slope1} + \frac{U_t-U_{optimal}}{1-U_{optimal}} \times R_{slope2} &\text{if } U \geq U_{optimal}
\end{cases}
$$

当获得基础利率 $R_{base}$ 后，我们考虑 最佳固定利率负债总负债比率常数 $O_{ratio}$ 的影响，公式如下:

$$
R_t = \begin{cases}
   R_{base} &\text{if } ratio < O_{ratio} \\\
   R_{base} + \frac{ratio - O_{ratio}}{1 - O_{ratio}} \times offset &\text{if } ratio \geq O_{ratio}
\end{cases}
$$

其中，各参数可通过与合约交互获得，对应如下:

- $ratio$ 固定利率负债与浮动利率负债之比，获取方法稍后给出
- $O_{ratio}$ 最佳比例，通过`OPTIMAL_STABLE_TO_TOTAL_DEBT_RATIO`获得
- $offset$ 标准偏差，通过`getStableRateExcessOffset`获得

> 由于`AAVE V3`白皮书与文档在固定利率计算方面给出的公式语焉不详，此处公式直接参考了[DefaultReserveInterestRateStrategy 合约](https://github.com/aave/aave-v3-core/blob/master/contracts/protocol/pool/DefaultReserveInterestRateStrategy.sol#L226)

获得 $ratio$ 的方法较为复杂，我们需要进行合约交互。查询文档发现，获得此数据需要与`AaveProtocolDataProvider`合约交互通过`getReserveData`获得相关数据，具体函数参数和返回值可参考[文档](https://docs.aave.com/developers/core-contracts/aaveprotocoldataprovider#getreservedata)。在`Arbitrum`网络上，此合约地址为`0x69FA688f1Dc47d4B5d8029D5a35FB7a548310654`。(部署地址可在[此页](https://docs.aave.com/developers/deployed-contracts/v3-mainnet/arbitrum)获得)。

直接前往[Arbiscan相关页面](https://arbiscan.io/address/0x69FA688f1Dc47d4B5d8029D5a35FB7a548310654#readContract#F12)操作，在`asset`内输入`DAI`的地址`0xda10009cbd5d07dd0cecc66161fc93d7c9000da1`，我们就可以索引到相关数据，如下图:

![getReverseData](https://img.gejiba.com/images/a7ece3a689482a7af4da6750e7a05694.png)

> 可以通过`cast call 0x69FA688f1Dc47d4B5d8029D5a35FB7a548310654 "getReserveData(address)(uint256,uint256,uint256,uint256,uint256,uint256,uint256,uint256,uint256,uint256,uint256,uint40)" 0xda10009cbd5d07dd0cecc66161fc93d7c9000da1 --rpc-url https://rpc.ankr.com/arbitrum` 命令获得相关数据。

使用`totalStableDebt`(固定利率债务总价值)与`totalVariableDebt`(浮动利率债务总价值)可以计算得到 $ratio$ ，计算结果为`0.01399`，小于 $O_{ratio}$ 的数值`0.2`，故而我们不用考虑 $ratio$ 对于利率的影响，可以直接使用以下公式计算:

$$ 0.05 + \frac{0.3941}{0.8000} \times 0.005 = 0.05246 $$

使用上述 $Actual\ APY$ 公式转换得到 实际利率为 $0.05386$，与页面显示一致，如下图:

![AAVE DAI APY](https://img.gejiba.com/images/5e0f2202f19a34838189b1ea41e4d46b.png)

其实在`getReserveData`函数的返回值内`stableBorrowRate`即代表当前资产流动性池的固定利率，此数值除以 $10^{27}$ 即可获得单利条件下的利率，即`0.05246`，此数值与我们的计算结果一致。同理，`variableBorrowRate`代表浮动利率，也可以通过除以 $10^{27}$ 的方式转换为单利。

## 其他模式
### isolation mode

我们可以在`Assets to supply`内发现部分资产的是否能成为贷款抵押资产`Can be collateral`栏目内填写有`Isolation`单词，这说明此资产属于`Isolation model`内的资产。

任何项目都可以将项目代币申请为`AAVE`交易对的成员，但过去由于所有资产都可以作为质押品贷出资产，这导致`AAVE`治理委员会要求申请`AAVE`交易对的代币必须满足一系列严格的要求。这使得`AAVE`的可交易的资产种类较少。

下图展示了对于一般代币进入`AAVE`的评级方法:
![Token Table](https://acjgpfqbqr.cloudimg.io/_cat_/hdd852.png)

为了解决这些问题，`AAVE`引入了`isolation mode`，进入`isolation mode`的代币位于一个特定的流动性池内，仅能用于贷出稳定币且具有债务上限。这使得部分早期项目的代币也可以进入`AAVE`的交易池，比如[这个](https://governance.aave.com/t/arc-add-tryb-to-aave-v3-on-avalanche-network-isolation-mode)。

下图展示了在`isolation mode`下，我们只能使用`$Token2`代币贷出稳定币:

![Isolation Mode Token2](https://img.gejiba.com/images/bf286978893868222a8ebaf4dd020810.png)


为了更加直观地理解此概念，我们为大家在测试网内展示如何使用`isolation mode`。我们首先前往[Faucet](https://app.aave.com/faucet/)领取`EURS`代币。在`EURS`代币内重复上述流程，最终结果如下图:

![AAVE Supply Example](https://img.gejiba.com/images/b46ec1cc28adef92c9ac70b10462b664.png)

为了进入`isolation mode`，我们需要首先偿还所有贷出代币，然后关闭`USDC`的`Collateral`功能，然后打开`EURS`的`Collateral`功能。通过上述操作，我们可以进入`isolation mode`，最终结果如下:

![Enter isolation mode](https://img.gejiba.com/images/f4c2fc069e3cef0498c8ed674b6568f0.png)

然后，我们可以贷出稳定币资产。退出`isolation mode`的步骤如下:

1. 偿还在`isolation mode`中贷出的资产
1. 关闭`EURS`的`Collateral`功能

> 关闭`Collateral`功能实质上是关闭了将此代币作为质押品的功能

可能有读者好奇，我可不可以即使用`isolation model`中的资产和正常资产混用进行贷款等操作？ 答案是不可以。可见`isolation model`具有一定缺陷，具体有以下几种:

1. 严重的流动性割裂。当流动性提供商面对大量的`isolation model`内的各类资产，基于逐利的本性，流动性提供商会进一步分割其资产以达到利益最大化，这导致部分流动性池内的资产可能较少。
1. 降低的用户体验。在正常模式中，用户的抵押品是汇总的，我们无需考虑可以直接使用质押品贷出资产。但对于`isolation`，我们需要为每一个分离的资产建立独立的头寸，并进行单独管理，这大大降低了用户的体验。

> 更加完整的讨论可以参考[AAVE V3 Technical Paper](https://github.com/aave/aave-v3-core/blob/master/techpaper/Aave_V3_Technical_Paper.pdf)

### E-Mode

此模式也是`AAVE V3`引入的，用于带有强相关性资产的抵押和贷款，比如`USDT`/`DAI`/`USDC`稳定币等，这些资产具有本质上的价值一致性，我们认为这些资产之间的抵押和贷款风险都较低。在正常模式下，使用`DAI`仅能贷出相当于`DAI`价值`80%`的其他稳定币，这显然不太合理，当我们启用`E-Mode`后，`DAI`可以借出`97%`价值的稳定币。

一个展示普通模式与`E-Mode`差别的示意图如下:

![E-Mode](https://img.gejiba.com/images/84c3e142e3120cd801834b29219b101b.png)

启用此模式时，我们需要点击`You Brrows`左侧的`DISABLED`，如下图:

![Enable button](https://img.gejiba.com/images/6284a3330415288f81745b309d499eb0.png)

点击后同意交易即可进入`E-Mode`模式享受对部分资产的超高`LTV`。推出此模式只需要在此点击`You Brrows`左侧的按钮即可，但需要注意`Health factor`，如下图:

![E-Mode Disable](https://img.gejiba.com/images/3b4fd2ea43f7196a23f16f90461d58b9.png)

读者可以注意到此处显示`Health factor`在退出`E-Mode`时已经小于`1`，所以此时如果我们一旦退出`E-Mode`会导致资产被立即清算。读者在退出`E-Mode`时需要注意此数值，可以在偿还部分贷款后再退出。

## 总结

本文基本介绍了正常情况下与`AAVE`交互所需要的基本知识，如下:

1. 存款的步骤与存款利率计算
1. 贷款步骤与贷款浮动利率、固定利率计算
1. `isolation mode`和`E-Mode`模式的进入和退出

本文没有涉及`$AAVE`的代币经济学和`AAVE`安全保险，这部分内容会在下一篇文章介绍。
