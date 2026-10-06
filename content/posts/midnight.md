---
title: "现代 DeFi: Midnight 固定利率市场"
date: 2026-10-06T12:52:13Z
tags: [DeFi]
math: true
---

## 概述

[Midnight](https://github.com/morpho-org/midnight) 是 Morpho 推出的基于 **订单簿** 的固定利率借贷协议。在 [Midnight whitepaper](https://morpho.org/whitepapers/midnight-whitepaper.pdf) 内，Morpho 指出传统的基于 Pool 的借贷协议往往面临启动时流动性缺失的情况，但使用订单簿模型可以一定程度的减轻该问题，因为资金的利用方可以通过订单表达自己的流动性需求。

基于订单簿的市场内，Morpho 主要讨论并解决了以下核心问题:

1. 如何处理固定利率债务的定价问题。对于该问题 Pendle 设计了复杂的 AMM 机制来处理固定利率定价，但是 Midnight 基本没有处理该问题。对于借款人而言，固定利率债务按照到期时的价值进行计价(类似传统金融内的零息债券)，假如担保品价值不足支撑到期债务，那么就会被清算。而存款人的报价也是直接按照贴现后的价格报价，固定利率可以使用简单的 $r = \frac{1}{P} - 1$ 计算。对于债务存续期的定价问题，Midnight 没有处理，用户可以通过订单成交提前清算债务，或者选择提前还款(提前还款也需要支付全部利息)
2. 如何处理单个流动性在多个市场内挂单的问题，比如用户提供 10 ETH 可能会在不同的市场内作为存款人等待借款人吃单
3. 如何设计订单簿的 tick 等，实现灵活挂单和流动性不被过度分散的平衡，我们会在后文详细讨论这一点

当然，除了上述问题外，借贷协议内最复杂的清算问题也是核心问题。另外，由于目前 Morpho 已经具有一个庞大的策展人市场，所以 Morpho 团队也非常重视 Midnight 与现有金库对接的问题，使用一系列 callback 函数来实现尽可能去中心化、无许可和模块化的外围系统接入。本文假设读者理解 Merkle Tree 的基础内容，特别是理解 merkle tree inclusion proof，本文不会对该部分展开详细介绍。

## 基础数据结构

在阅读具体的执行逻辑前，我们可以先阅读 `src/interfaces/IMidnight.sol` 内定义的 Midnight 所使用的数据结构。通过这些数据结构，我们可以知道一些核心信息。在本节中，我们会阅读 `IMidnight.sol` 内定义如下几个结构体:

1.  `Market` 定义了固定利率市场的基础参数
2.  `Position` 定义了用户进行固定利率借贷后的头寸信息
3. `Offer` 定义了用户如何进行挂单
4. `MarketState` 定义了市场内的一些其他状态，主要是协议手续费

### Market

显然，`Market` 结构体是核心数据类型，该结构体定义了 Midnight 内的固定利率市场。我们可以看到 `Market` 的基础核心参数包括该市场可以以固定利率借出的代币 `loanToken`、该市场接受的担保品及其参数 `CollateralParams`、固定利率市场到期日 `maturity` 以及一些特殊参数，这些参数基本都是用于清算逻辑的。更具体来说，在担保品参数 `CollateralParams` 内，我们可以看到 `lltv` 用于描述单位数量担保品可以借出资产的上限，这意味着担保品价值 * lltv > 债务价值，那么当前头寸就会被视为是健康的；`oracle` 是用于读取担保品价值的；`maxLif` 是一个与清算有关的参数，其中 `Lif` 是 **liquidation incentive factor** 缩写，代表清算者在清算过程中可以获得的激励。此处需要注意，`CollateralParams` 内的 `lltv` 不能随意设置，在 `src/libraries/ConstantsLib.sol`  代码内的 `isLltvAllowed` 函数约定了哪些 lltv 参数是合法的。

```solidity
struct Market {
    address loanToken;
    CollateralParams[] collateralParams;
    uint256 maturity;
    uint256 rcfThreshold;
    address enterGate;
    address liquidatorGate;
}

struct CollateralParams {
    address token;
    uint256 lltv;
    uint256 maxLif;
    address oracle;
}
```

此处我们特别介绍一下 Lif 的工作原理，在借贷协议中，当用户头寸不健康后就需要清算人介入处理用户头寸，清算人会接受头寸偿还债务，然后拿走债务底层的担保品，但是此处显然，假如偿还的债务价值和获得的担保品价值等值，那么清算人没有任何动机清算。所以一般来说，清算人会获得大于清算债务价值的担保品，而这个获得担保品机制与清算债务之间的比例就是 Lif。

最后，我们处理 `Market` 内的另一个与清算有关的数值参数 `rcfThreshold`。该参数是一个用于处理粉尘债务问题的参数。为了理解该参数的使用方法，我们需要引入一些 Midnight 内部清算机制运行的一些数学逻辑。在上文内，我们提到清算人可以清算债务然后获得大于债务价值的担保品。在借贷领域，一般存在两种清算模式:

1. 硬清算，清算人直接清算头寸的所有债务，然后拿走所有担保品，这种机制对于借款人不友好，但是对于协议开发者而言是最好实现的
2. 软清算，即清算人只能将用户头寸清算到健康即可，此处清算后目标监控度在 midnight 内被称为 RCF(**recovery close factor**)

首先，在 Midnight 内，一个重要的概念就是 max debt 即当前用户担保品可以承载的最大债务价值，我们可以使用 lltv 通过以下方法计算：
$$
\text{maxDebt} = \mathrm{sum}(\text{collateralValue} * \text{LLTV})
$$
假如出现 `debt > maxDebt` 的情况，那么就意味着该用户会被清算。那么清算者假如偿还 `repaidUnits` 价值债务，清算后的债务情况为:
$$
\text{newDebt} = \text{debtOf} - \text{repaidUnits}
$$
此处的 `newDebt` 代表清算后的债务情况，而 `debtOf` 代表清算前的债务情况。上述清算也会拿走被清算头寸的一定数量的担保品，此处清算者拿走担保品的数量为 `repaidUnits * LIF`。拿走这些数量的担保品显然会影响被清算头寸的 `maxDebt`，我们此处引入新变量 `newMaxDebt` 代表清算后按照新的担保品数量，当前头寸可以借出的最大资产情况，此处存在如下关系:
$$
\text{newMaxDebt} = \text{maxDebt} - \text{repaidUnits} * \text{LIF} * \text{LLTV}
$$
在上文中，我们提到 midnight 使用了软清算机制，且清算目标是 RCF。而 RCF 的实际条件则比较令人困惑，即 `newDebt >= newMaxDebt`。这是一个非常有趣的条件，在上文内我们已经提到 `debt > maxDebt` 的情况下，头寸依旧是不健康的，所以实际上 RCF 允许清算后的头寸处于不健康状态(当然，出于经济利益考虑，清算人往往不会让清算后的头寸仍保持不健康状态)，而清算人可以清算的极限是 `newDebt = newMaxDebt`，此时头寸刚刚回到健康状态。所以说，RCF 是一个更倾向借款人的条件，该条件限制了清算人可以清算的债务数量，清算人只能将被不健康头寸清算到健康，而不允许进一步清算。

此处，我们可以利用 RCF 计算出单次清算内 `repaidUnits` 的数值:

$$
\begin{aligned}
\text{newDebt} &\ge \text{newMaxDebt} \\\\
\Longleftrightarrow\quad
\text{debtOf} - \text{repaidUnits}
&\ge
\text{maxDebt}
\- \text{repaidUnits} \cdot \mathrm{LIF} \cdot \mathrm{LLTV} \\\\
\Longleftrightarrow\quad
\text{repaidUnits}
&\le
\frac{\text{debtOf} - \text{maxDebt}}
     {1 - \mathrm{LIF} \cdot \mathrm{LLTV}}
\end{aligned}
$$

Midnight 合约内使用了如下 solidity 进行计算:

```solidity
maxRepaid = (_position.debt - maxDebt).mulDivUp(WAD * WAD, WAD * WAD - lif * lltv)
```

此处使用 `WAD * WAD` 是为了提高计算过程内的精度。目前大部分借贷协议，比如 AAVE v4 都会额外考虑清算的另一个问题，即粉尘债务问题。所谓粉尘债务是指清算人由于上述清算数学规则的限制可能导致清算后的头寸内仍残留一部分债务，这部分债务数额微小，所以即使这部分债务可以被清算，往往也没有任何第三方有动机去清算。粉尘债务往往最终会变成协议的坏账，而且随着利息的累积逐渐扩大。对于庞大的协议，大量粉尘债务的累积可能产生不低的坏账影响协议的安全性。

对于粉尘债务问题，最简单的解决方案是当发现清算人清算后剩余的债务数量低于某一个限额，那么就允许清算人清算所有的债务。但是注意 midnight 是一个支持多担保品的系统，即头寸的债务是由一篮子担保品支撑的，而清算人在清算过程中需要输入此次清算要清算的担保品类型:

```solidity
function liquidate(
    Market calldata market,
    uint256 collateralIndex,
    uint256 seizedAssets,
    uint256 repaidUnits,
    address borrower,
    bool postMaturityMode,
    address receiver,
    address callback,
    bytes calldata data
) external returns (uint256, uint256) {
```

当用户清算后的担保品价值小于某一个阈值时，我们就允许清算者清算所有的债务。而这个阈值就是上文内的提到的 `rcfThreshold`，以下代码显示了 `rcfThreshold` 动作原理:

```solidity
require(
    repaidUnits <= maxRepaid
        || _position.collateral[collateralIndex].mulDivDown(liquidatedCollatPrice, ORACLE_PRICE_SCALE)
            .mulDivDown(WAD, lif).zeroFloorSub(maxRepaid) < market.rcfThreshold,
    RecoveryCloseFactorConditionsViolated()
);
```

上述代码可以简化为:

```
repaidUnits <= maxRepaid
    || collateral * price / LIF - maxRepaid < market.rcfThreshold
```

即我们首先计算待清算担保品可清算的债务数量，然后与目前最大计算出的最大债务可清算数量 `maxRepaid` 做差，如果获得差值小于 `market.rcfThreshold`，那么我就认为清算者可以清算所有债务。经过上文的分析，读者应该可以自行理解  `src/Midnight.sol` 内的以下注释:

```
/// @dev The RCF is deactivated for small collateral amount, essentially to mitigate issues with liquidations that are
/// too small compared to the gas cost. More precisely, it is deactivated if the liquidation could leave a collateral
/// with a value that would not be enough to repay rcfThreshold units. Which means (omitting scaling and roundings):
///   minNewCollateral * liquidatedCollatPrice / LIF < rcfThreshold
///     <=> (collateral - maxRepaid * LIF / liquidatedCollatPrice) * liquidatedCollatPrice / LIF < rcfThreshold
///     <=> collateral * liquidatedCollatPrice / LIF - maxRepaid < rcfThreshold
```

最后，我们处理 `enterGate` 和 `liquidatorGate` 的字段含义问题，这也是 Morpho 在 Midnight 引入的新机制，白皮书内的 **5 Access-control gates** 专门介绍了该机制。简单来说，所有的 `gate` 的目标都是进行访问权限控制，主要是进行一些白名单控制等。以下代码显示了 `liquidatorGate` 是如何工作的:

```solidity
require(
    market.liquidatorGate == address(0) || ILiquidatorGate(market.liquidatorGate).canLiquidate(msg.sender),
    LiquidatorGatedFromLiquidating()
);
```

### Position

在上文内，我们提到在 Midnight 内，用户可以使用多个担保品进行借贷，所以此处我们刚好可以看一下 `Position` 结构体的构成:

```solidity
struct Position {
    uint128 credit;
    uint128 pendingFee;
    uint128 lastLlossFactorossFactor;
    uint128 lastAccrual;
    uint128 debt;
    uint128 collateralBitmap;
    uint128[128] collateral;
}
```

与 AAVE v4 类似，Midnight 也使用了 bitmap 方案来存储用户存入的 collateral 情况。在 Midnight 内，每一个 collateral 都会被分配到一个 `id`，该 `id` 对应当前担保品在 `Position` 内的 `collateralParams` 数组内的位置，其实也是此处 `Position` 内的 `collateral` 数组位置。简单来说，假如 `id = 2` 的资产被用户存入，那么我们会在 `Position.collateral[2]` 内存储该资产被存入的数量。

此处就有一个简单问题，假如当前市场启用的资产种类很多，而用户真实提供的资产类型没有那么多，每次完整遍历 `collateral` 数组是一种 gas 浪费行为，所以我们需要一些其他方法减少遍历的次数。这种问题的通用解决方法就是使用 bitmap 进行存储，我们会在 bitmap 内利用 bit 为 `0` 或者 `1` 代表用户在此 `id` 内是否提供了资产。AAVE v4 和更早的 Uniswap v3 都使用了该技术方案。

另外比较重要的核心参数是 `credit` 和 `debt` 分别代表当前 Position 的债权数量和债务数量。其中前者主要用于借贷流动性提供方，而后者显然用于借贷流动性需求方。此处我们需要额外注意，`credit` 是会直接按照债务到期后的面值计算，我们假如存在一个 1 年期的 5% 年华的固定利率借贷，那么存款人提供 95 单位的资产实际上可以获得 100 单位的 `credit`。所以 `credit` 其实已经包含了存款人可以获得的利息收益。

DeFi 协议的运营商一般都是要分润手续费，而 `Position` 字段内的 `pendingFee` 就是用于协议方收取手续费的字段。协议管理员可以配置 `continuousFee` 数据，该数据是一个代表协议方收取手续费的比例情况，注意这部分手续费收取不是针对利息，而是针对 `credit` 全部，所以理论上可能出现 `continuousFee` 配置的费率大于当前收益率的情况，这种情况下向市场提供流动性会导致亏损。目前协议管理员不能配置大于 1% 年利率以上的 `continuousFee`。

```solidity
uint32 constant MAX_CONTINUOUS_FEE = uint32(uint256(0.01e18) / uint256(365 days));
```

`pendingFee` 是一个有趣且较为复杂的机制，我们会在后文详细讨论。目前我们只需要知道 `pendingFee` 是当前头寸从目前(更加具体的说是 `lastAccrual` 开始)到到期时间(即 `maturity` )整段时间内的手续费情况。`pendingFee` 会配合 `lastAccrual` 使用，我们会按照 `lastAccrual` 计算真实应该从 Position 内的拿走的 fee 情况:

```solidity
uint256 accrualEnd = UtilsLib.min(block.timestamp, market.maturity);
uint128 _lastAccrual = _position.lastAccrual;
// forge-lint: disable-next-item(unsafe-typecast) as fee <= pending <= credit which are uint128 position fields
uint128 fee = _lastAccrual < market.maturity
    ? uint128(postSlashPendingFee.mulDivDown(accrualEnd - _lastAccrual, market.maturity - _lastAccrual))
    : 0;
```

正如我们上文所述，`pendingFee` 内记录了 credit(我们会在后文详细介绍什么是 credit，目前可以理解为是用户的存款) 从 `lastAccrual` 到 `maturity` 内整段时间的手续费，所以计算 `fee` 的时候我们会按照时间比值计算 Position 真实累积的手续费。在上述代码片段内，我们可以看到 `postSlashPendingFee` 变量，该变量与 Midnight 内的清算机制有关，更直接来说，该变量计算也依赖于 `Position` 结构体内的 `lastLossFactor` 字段。

在 Morpho 内，我们知道 Morpho 实际上是直接通过社会化损失来处理借贷穿仓问题的，在 Midnight 内也是一样的。我们会在 `lossFactor` 内计入当前的穿仓情况，然后利用全局的 `lossFactor` 和用户 Position 内的 `lossFactor` 差值计算当前 Position 应该承担的坏账损失情况。

```solidity
uint128 _lastLossFactor = _position.lastLossFactor;
uint256 postSlashCredit = _lastLossFactor < type(uint128).max
    ? credit.mulDivDown(type(uint128).max - marketState[id].lossFactor, type(uint128).max - _lastLossFactor)
    : 0;
uint128 _pendingFee = _position.pendingFee;
uint256 postSlashPendingFee =
    credit > 0 ? _pendingFee - _pendingFee.mulDivUp(credit - postSlashCredit, credit) : 0;
```

注意，无论是全局的 `lastLossFactor` 还是 Position 内的 `lastLossFactor` 都是一个类似百分比的数字，该数值最大为 `type(uint128).max`，而发生坏账后，我们就会减少 `lastLossFactor` 的数值，所以上述计算中我们可以看到 `type(uint128).max - marketState[id].lossFactor` 类似的计算，实际上就是计算的坏账损失的情况。

### Offer

我们已经知道了 `Market` 和 `Position` 结构体内的具体内容，接下来我们讨论另一个重要的结构体 `Offer`。Midnight 是一个基于订单的固定利率借贷协议，那么订单所包含的内容也是协议的核心部分:

```solidity
struct Offer {
    Market market;
    bool buy;
    address maker;
    uint256 start;
    uint256 expiry;
    uint256 tick;
    bytes32 group;
    address callback;
    bytes callbackData;
    address receiverIfMakerIsSeller;
    address ratifier;
    bool reduceOnly;
    uint256 maxUnits;
    uint256 maxAssets; // buyerAssets if offer.buy else sellerAssets
}
```

与普通的订单簿类似，Midnight 的 `Offer` 也包含一些基本属性:

1. `buy` 代表订单方向，其中 `buy == true` 代表购买 `units`，会首先减少 `maker` 的 debt，假如此时 debt 已经为 0 那么就会增加 `maker` 的 credit。 `buy == false` 代表卖出 `units`，会首先减少 `maker` 的 credit 而假如此时 maker 的 credit 已经归零，那么就会增加 `maker` 的 debt。
2. `maker` 代表订单的挂单人，但是注意 `maker` 和 `seller` 以及 `buyer` 在概念上是不一致的，在代码内，我们使用 `(address buyer, address seller) = offer.buy ? (offer.maker, taker) : (taker, offer.maker);` 来使用 maker 字段获得 `buyer` 和 `seller` 的地址。`buyer` 总是买入 `units` 而 `seller` 总是卖出 `units`
3. `start` 和 `expiry` 分别代表订单的生效时间和过期时间
4. `tick` 代表订单的价格，我们会在后文介绍 Midnight 内的 tick 机制
5. `maxUnits` 和 `maxAssets` 代表订单规模，用户只能配置其中一个，`maxUnits` 是以到期债务或债权数量约束的，而 `maxAssets` 是以当前订单的代币数量约束的
6. `receiverIfMakerIsSeller` 代表部分订单成交后资产的流向，该字段只对 offer 的 maker 是 `seller` 时生效(即 `offer.buy == false` 的情况)。由于此时 offer 的 maker 是 seller 代表卖出 units，所以 seller 是可以获得 loanToken 的，此时 `receiverIfMakerIsSeller` 可以指定一个地址接受 loanToken。
7. `reduceOnly` 用于减仓，比如借款人可以使用 `buy == true` 的订单来减少自己的债务。对于一般的订单而言，假如借款人使用 `buy == true` 订单来减少债务。此处与 Midnight 的底层机制有关，我们会在后文介绍 credit 和 debt 关系时详细介绍

在上述基础字段后，我们可以关注一些有关新机制的字段。我们要分析的第一个字段就是出于优化流动性碎片化而引入的 `group`。Group 机制允许多个订单共用一个 `maxUnits` 或 `maxAssets`。在 Midnight 内部，存在如下存储变量定义:

```solidity
mapping(address user => mapping(bytes32 group => uint256)) public consumed;
```

显然，该变量实际上定义了当前 offer 已经被消耗的数量，我们上文介绍的 `maxUnits` 和 `maxAssets` 参数会配合 `consumed` 使用:

```solidity
uint256 newConsumed;
if (offer.maxAssets > 0) {
    newConsumed = consumed[offer.maker][offer.group] += offer.buy ? buyerAssets : sellerAssets;
    require(newConsumed <= offer.maxAssets, ConsumedAssets());
} else {
    newConsumed = consumed[offer.maker][offer.group] += units;
    require(newConsumed <= offer.maxUnits, ConsumedUnits());
}
```

我们注意到上述 `consumed` 字段只与 `offer.maker` 和 `offer.group` 有关，而与 `offer.market` 无关，所以 offer 提供者不难构建出跨多个市场但共用 `maxUnits` 或 `maxAssets` 的订单。但是需要注意的，目前合约内没有检查共用 group 字段的 offer 是否配置了同样的 `maxUnits` 或 `maxAssets` 字段，所以理论上 offer 提供者可以为同一个 group 下的订单设置不同的 `maxAssets` 或 `maxUnits`，但这会导致语义上的混乱，不建议开发者这样配置。

`consumed` 字段实际上还承载了撤销 offer 未成交部分的功能，我们可以使用以下函数直接修改 `consumed` 字段:

```solidity
/// @dev Passing type(uint256).max cancels all offers in the group (and never reverts).
function setConsumed(bytes32 group, uint256 amount, address onBehalf) external {
    require(onBehalf == msg.sender || isAuthorized[onBehalf][msg.sender], Unauthorized());
    require(amount >= consumed[onBehalf][group], AlreadyConsumed());
    consumed[onBehalf][group] = amount;
    emit EventsLib.SetConsumed(msg.sender, group, amount, onBehalf);
}
```

在此处，我们也可以一窥 midnight 内的 `onBehalf` 的鉴权机制。当然，阅读过 morpho 源代码的开发者对上述机制并不陌生。在 Midnight 内，用户可以使用 `setIsAuthorized` 给其他人权限，使其可以代表自己进行操作:

```solidity
/// @dev See AUTHORIZATIONS section above.
function setIsAuthorized(address authorized, bool newIsAuthorized, address onBehalf) external {
    require(onBehalf == msg.sender || isAuthorized[onBehalf][msg.sender], Unauthorized());
    isAuthorized[onBehalf][authorized] = newIsAuthorized;
    emit EventsLib.SetIsAuthorized(msg.sender, authorized, newIsAuthorized, onBehalf);
}
```

回到 `Offer` 结构体，对于结构体内的 `callback` / `callbackData` 和 `ratifier`，我们都还没有进行介绍。其中 `callback` 和 `callbackData` 类似 Uniswap v4 hook 的功能，在白皮书 3.2 Maker callbacks 一节介绍了 Midnight 设计者的目的。`callback` 主要服务于  maker 中的 `buyer` 情况。在上文中，我们提到 `buyer` 会买入 `units` ，所以 buyer 会向 receiver 转账 `loanToken`(具体 receiver 是那一个地址是需要根据 `receiverIfMakerIsSeller` 等决定，具体可以参考如下代码)，而这部分资金就是从 `buyerCallback` 内支付:

```solidity
(address buyer, address seller) = offer.buy ? (offer.maker, taker) : (taker, offer.maker);

...

address buyerCallback = offer.buy ? offer.callback : takerCallback;
address sellerCallback = offer.buy ? takerCallback : offer.callback;
address payer = buyerCallback != address(0) ? buyerCallback : (offer.buy ? buyer : msg.sender);
address receiver = offer.buy ? receiverIfTakerIsSeller : offer.receiverIfMakerIsSeller;

...

if (buyerCallback != address(0)) {
    bytes memory buyerCallbackData = offer.buy ? offer.callbackData : takerCallbackData;
    require(
        IBuyCallback(buyerCallback)
            .onBuy(id, offer.market, buyerAssets, units, buyerPendingFeeIncrease, buyer, buyerCallbackData)
        == CALLBACK_SUCCESS,
        WrongBuyCallbackReturnValue()
    );
}

SafeTransferLib.safeTransferFrom(offer.market.loanToken, payer, address(this), buyerAssets - sellerAssets);
SafeTransferLib.safeTransferFrom(offer.market.loanToken, payer, receiver, sellerAssets);
```

当然，上述代码还告诉我们，用户也可以将 offer 的 `callback` 配置为非零地址，此时就会使用 `offer.maker` 内的资金使用 callback 进行代币支付。这样做的好处主要是该机制允许用户将订单所需要的流动性先配置在其他地方，比如 morpho 的浮动利率借贷金库内，在需要时，再将资产转回到 callback 合约内，由此提高资本利用率。

上述 callback 处理了资金转移问题，最后我们该如何处理订单有效性问题。我们可以看到 `offer` 结构体内没有包含一些类似签名的字段，所以 Midnight 是如何处理订单有效性问题的？结论是使用 `ratifier` 字段。该字段用于检查 offer 是否是 maker 设置为有效的。以下两行代码来自 `take` 函数入口，用来检查 offer 有效性:

```solidity
require(isAuthorized[offer.maker][offer.ratifier], RatifierUnauthorized());
require(IRatifier(offer.ratifier).isRatified(offer, ratifierData) == CALLBACK_SUCCESS, RatifierFail());
```

首先 offer maker 需要使用 `setIsAuthorized` 将 `offer.ratifier` 配置为 `isAuthorized` 的对象，然后 Midnight 会调用 `offer.ratifier` 并传入订单信息确定当前订单是否有效。在 `src/ratifiers` 文件夹内，Midnight 官方给出了两个 ratifiers 的示例，这两个示例底层都依赖于 Merkle Tree，为什么依赖于 Merkle Tree? 是因为 Midnight 希望实现用户可以批量挂单，即我可以一次性将多个 offer 设置为有效(此处需要与之前的 group 机制区分，group 机制是指多个 offer 共用 `maxAssets` 或 `maxUnits`)。在密码学原语中，最简单的确定元素是否属于某一个集合的方法就是 Merkle Tree。

在 Midnight 给出的两个 ratifiers 示例内，`SetterRatifier` 较为简单，该合约允许 offers maker 使用 `setIsRootRatified` 函数将一批订单构建的 Merkle Tree root 设置到合约内:

```solidity
function setIsRootRatified(address maker, bytes32 root, bool newIsRootRatified) public {
    require(maker == msg.sender || IMidnight(MIDNIGHT).isAuthorized(maker, msg.sender), Unauthorized());
    isRootRatified[maker][root] = newIsRootRatified;
    emit SetIsRootRatified(msg.sender, maker, root, newIsRootRatified);
}
```

当 Midnight 调用 `isRatified` 时，`SetterRatifier` 会使用如下实现:

```solidity
function isRatified(Offer memory offer, bytes memory ratifierData) external view returns (bytes32) {
    require(msg.sender == MIDNIGHT, NotMidnight());
    (bytes32 root, uint256 leafIndex, bytes32[] memory proof) =
        abi.decode(ratifierData, (bytes32, uint256, bytes32[]));
    require(HashLib.isLeaf(root, HashLib.hashOffer(offer), leafIndex, proof), InvalidProof());
    require(isRootRatified[offer.maker][root], NotRatified());
    return CALLBACK_SUCCESS;
}
```

简单来说就是调用 `HashLib.isLeaf` 利用 merkle tree inclusion proof 判断订单是否在 root 内部。而另一个 `EcrecoverRatifier` 的原理类似，但是允许 offer maker 不进行在合约内进行预先的 merkle tree root 配置，而是在 `ratifierData` 内传入 `root` 和签名等信息:

```solidity
function isRatified(Offer memory offer, bytes memory ratifierData) external view returns (bytes32) {
    require(msg.sender == MIDNIGHT, NotMidnight());
    (Signature memory sig, bytes32 root, uint256 leafIndex, bytes32[] memory proof) =
        abi.decode(ratifierData, (Signature, bytes32, uint256, bytes32[]));
    require(HashLib.isLeaf(root, HashLib.hashOffer(offer), leafIndex, proof), InvalidProof());
    require(!isRootCanceled[offer.maker][root], RootCanceled());
    bytes32 structHash = keccak256(abi.encode(HashLib.offerTreeTypeHash(proof.length), root));
    bytes32 domainSeparator = keccak256(abi.encode(EIP712_DOMAIN_TYPEHASH, block.chainid, address(this)));
    bytes32 digest = keccak256(bytes.concat("\x19\x01", domainSeparator, structHash));
    address _signer = ecrecover(digest, sig.v, sig.r, sig.s);
    require(_signer != address(0), InvalidSignature());
    require(_signer == offer.maker || IMidnight(MIDNIGHT).isAuthorized(offer.maker, _signer), Unauthorized());
    return CALLBACK_SUCCESS;
}
```

相比于 `SetterRatifier`，`EcrecoverRatifier` 的 `isRatified` 函数复杂性主要在于检查签名部分，我们可以看到此处也使用了 EIP712 签名(假如读者不熟悉 EIP712 签名，可以参考笔者之前编写的 [博客](https://blog.wssh.dev/posts/ecsda-sign-chain/#eip712))。EIP712 的核心是将 typeHash 和编码后的数据拼接在一起，此处较为复杂的是计算 `OfferTree` 的 typehash，以下命令给出了 height == 0 情况下的 type hash 计算。

```bash
cast keccak 'OfferTree(Offer offerTree)CollateralParams(address token,uint256 lltv,uint256 maxLif,address oracle)Market(address loanToken,CollateralParams[] collateralParams,uint256 maturity,uint256 rcfThreshold,address enterGate,address liquidatorGate)Offer(Market market,bool buy,address maker,uint256 start,uint256 expiry,uint256 tick,bytes32 group,address callback,bytes callbackData,address receiverIfMakerIsSeller,address ratifier,bool reduceOnly,uint256 maxUnits,uint256 maxAssets)'
```

当然，实际上我们可以直接通过 `offerTreeTypeHash(uint256 height)` 函数获得 type hash。注意，在上文中，我们始终没有讨论如何使用 offers 列表获得 Merkle Tree root，读者不难借助 LLM 基于目前的代码自行获得相关信息。额外需要注意的是由于 Merkle Tree 内总是包含 $2^n$ 个 offer，假如用户手中的订单数量不足 $2^n$ 则需要使用空订单进行填充。

### MarketState

最后，我们介绍 `MarketState` 结构体，该结构体的定义如下:

```solidity
/// @dev Settlement fee cbp values and the continuous fee are 0 until the market is created, then set to the default
/// values.
struct MarketState {
    uint128 totalUnits;
    uint128 lossFactor;
    uint128 withdrawable;
    uint128 continuousFeeCredit;
    uint16 settlementFeeCbp0;
    uint16 settlementFeeCbp1;
    uint16 settlementFeeCbp2;
    uint16 settlementFeeCbp3;
    uint16 settlementFeeCbp4;
    uint16 settlementFeeCbp5;
    uint16 settlementFeeCbp6;
    uint32 continuousFee;
    uint8 tickSpacing;
}
```

首先，我们看到了 `totalUnits` 定义，在上文中，我们提到 Midnight 最底层的资产类型就是 `units`，当用户买入 `units` 后就会先减少 debt 然后增加 credit，反之，当用户卖出 `units` 时，就会先减少 credit 然后增加 debt。而 `totalUnits` 就是一个跟踪当前系统内存在 `units`。`totalUnits` 代表当前 market 的总未平仓 debt 数量。创造债务和消灭债务都会影响这个数值，而使用 `take` 函数进行单纯的 credit 换手或者 debt 换手则不会。我们在后文介绍 `take` 函数时会进一步介绍。

对于借款人还款的 `repay` 函数(当然，`liquidate` 其实也是一种特殊的 `repay` 函数)，`repay` 一定意味着 debt 的减少，但是并不会影响 `credit` ，所以 `repay` 函数不会影响 `totalUnits`。`repay` 函数整体较为简单，该函数也是涉及到了 `MarketState` 内的 `withdrawable` 变量:

```solidity
function repay(Market memory market, uint256 units, address onBehalf, address callback, bytes calldata data)
    external
{
    require(onBehalf == msg.sender || isAuthorized[onBehalf][msg.sender], Unauthorized());
    bytes32 id = touchMarket(market);

    position[id][onBehalf].debt -= UtilsLib.toUint128(units);
    marketState[id].withdrawable += UtilsLib.toUint128(units);

    address payer = callback != address(0) ? callback : msg.sender;
    emit EventsLib.Repay(msg.sender, id, units, onBehalf, payer);

    if (callback != address(0)) {
        require(
            IRepayCallback(callback).onRepay(id, market, units, onBehalf, data) == CALLBACK_SUCCESS,
            WrongRepayCallbackReturnValue()
        );
    }
    SafeTransferLib.safeTransferFrom(market.loanToken, payer, address(this), units);
}
```

在此处，我们注意到 `repay` 的时候我们会直接将 `units` 单位数量的 `loanToken` 直接转入 midnight 合约，这其实意味着在 `repay` 的时候，还款人需要偿付所有的债务，换言之，就算借款人在固定利率债务到期前还款也需要支付全额利息。当然，假如借款人希望只支付部分利息，可以通过提交 buy credit 订单的方式来进行操作。

在此处，我们可以看到 `repay` 增加 `marketState[id].withdrawable` 变量，该变量内存储着 midnight 合约内可以被存款人直接提取的资产数量。当然，这意味着以下不变量是恒成立的:

```
totalUnits = sum(position debt) + withdrawable
```

与 `withdrawable` 有关的另一个函数是 `withdraw` 函数，该函数允许持有 credit 的用户提取自己的收益:

```solidity
function withdraw(Market memory market, uint256 units, address onBehalf, address receiver) external {
    require(onBehalf == msg.sender || isAuthorized[onBehalf][msg.sender], Unauthorized());
    bytes32 id = touchMarket(market);
    MarketState storage _marketState = marketState[id];
    _updatePosition(market, id, onBehalf);

    Position storage _position = position[id][onBehalf];
    uint128 pendingFeeDecrease;
    if (_position.credit > 0) {
        pendingFeeDecrease = UtilsLib.toUint128(_position.pendingFee.mulDivUp(units, _position.credit));
        _position.pendingFee -= pendingFeeDecrease;
    }
    _position.credit -= UtilsLib.toUint128(units);
    _marketState.withdrawable -= UtilsLib.toUint128(units);
    _marketState.totalUnits -= UtilsLib.toUint128(units);

    emit EventsLib.Withdraw(msg.sender, id, units, onBehalf, receiver, pendingFeeDecrease);

    SafeTransferLib.safeTransfer(market.loanToken, receiver, units);
}
```

在上文中，我们简单介绍过 Position 内的 `pendingFee` 字段，在此处，我们可以由于 `withdraw` 相当于消灭了 credit 所以我们会将原本 `pendingFee` 内记录的从 credit 获取到到期日的 fee 减少 `pendingFeeDecrease`，因为用户在到期日来临前就已经退出了，所以要减少部分 fee。在 `MarketState` 结构体内的 `continuousFeeCredit` 和 `continuousFee` 字段也与协议手续费有关，在 `_updatePosition` 函数内，我们可以看到如下代码:

```solidity
Position storage _position = position[id][user];
(uint128 newCredit, uint128 newPendingFee, uint128 accruedFee) = updatePositionView(market, id, user);

...

marketState[id].continuousFeeCredit += UtilsLib.toUint128(accruedFee);
```

Midnight 内的 `feeClaimer` 角色可以调用 `claimContinuousFee(Market memory market, uint256 amount, address receiver)` 函数提取指定 market 内累积的 `continuousFeeCredit`。关于 `MarketState` 内的 `lossFactor` ，其功能在上文已有简单介绍，该字段用于 Position 穿仓后进行社会化损失。

在 `MarketState` 内，我们可以看到最有趣的就是 `settlementFeeCbp` 系列变量，存在从 `0` 到 `6` 共计 7 个变量。实际上`settlementFee` 与 `continuousFee` 类似，也是 midnight 协议运营方可以收取的一项协议手续费。在 Midnight 的变量内，存在 `mapping(address token => uint256) public claimableSettlementFee;` 记录相关费用。

与 `continuousFee` 不同，`settlementFee` 是对于 offer 的吃单人(即 taker) 直接收取的一项费用，会直接计入到 taker 的成交费用内。在 `take` 函数内，我们可以看到如下代码:

```solidity
uint256 offerPrice = TickLib.tickToPrice(offer.tick);
uint256 timeToMaturity = UtilsLib.zeroFloorSub(offer.market.maturity, block.timestamp);
uint256 _settlementFee = settlementFee(id, timeToMaturity);
uint256 sellerPrice = offer.buy ? offerPrice - _settlementFee : offerPrice;
uint256 buyerPrice = sellerPrice + _settlementFee;
uint256 buyerAssets = offer.buy ? units.mulDivDown(buyerPrice, WAD) : units.mulDivUp(buyerPrice, WAD);
uint256 sellerAssets = offer.buy ? units.mulDivDown(sellerPrice, WAD) : units.mulDivUp(sellerPrice, WAD);
```

此处的 `settlementFee` 就是使用了 `settlementFeeCbp` 和成交时距离固定利率到期日之间的时间差计算 `settlementFee` 的方法，计算获得的 `settlementFee` 单位是 WAD，与 `offerPrice` 的单位一致。此处我们通过直接将 `settlementFee` 作用到 `offerPrice` 上进行手续费收取，具体的收取方法如下:

1. `buy == true`，此时 `buyer`  是 `maker`，而 `buyerPrice = offerPrice - _settlementFee + _settlementFee = offerPrice`，而 `seller` 作为 taker，支付的 `sellerPrice = offerPrice - _settlementFee`。这意味着 `seller` 在更低价格内出售了 `units` 所以承担了 `settlementFee`
2. `buy == false`，此时 `seller` 作为 maker 其 `sellerPrice = offerPrice`，而 buyer 作为 taker 其支付的 `buyerPrice = sellerPrice + _settlementFee`，由于支付了更高价格买入，所以 buyer 作为 taker 承担了 `settlementFee`

> 在上述推导内，读者容易混乱 buyer / seller 和 maker  / taker 之间的关系，一个简单的判断方法是 `(address buyer, address seller) = offer.buy ? (offer.maker, taker) : (taker, offer.maker);`

那么具体 `settlementFee` 是如何计算的？此处我们可以直接展示 `settlementFee` 函数实现:

```solidity
/// @dev Returns the settlement fee using piecewise linear interpolation between breakpoints.
function settlementFee(bytes32 id, uint256 timeToMaturity) public view returns (uint256) {
    MarketState storage _marketState = marketState[id];
    require(_marketState.tickSpacing > 0, MarketNotCreated());

    if (timeToMaturity >= 360 days) return _marketState.settlementFeeCbp6 * CBP;

    // forgefmt: disable-start
    (uint256 start, uint256 end, uint256 feeLower, uint256 feeUpper) =
        timeToMaturity < 1 days   ? (  0 days,   1 days, _marketState.settlementFeeCbp0 * CBP, _marketState.settlementFeeCbp1 * CBP) :
        timeToMaturity < 7 days   ? (  1 days,   7 days, _marketState.settlementFeeCbp1 * CBP, _marketState.settlementFeeCbp2 * CBP) :
        timeToMaturity < 30 days  ? (  7 days,  30 days, _marketState.settlementFeeCbp2 * CBP, _marketState.settlementFeeCbp3 * CBP) :
        timeToMaturity < 90 days  ? ( 30 days,  90 days, _marketState.settlementFeeCbp3 * CBP, _marketState.settlementFeeCbp4 * CBP) :
        timeToMaturity < 180 days ? ( 90 days, 180 days, _marketState.settlementFeeCbp4 * CBP, _marketState.settlementFeeCbp5 * CBP) :
                                    (180 days, 360 days, _marketState.settlementFeeCbp5 * CBP, _marketState.settlementFeeCbp6 * CBP);
    // forgefmt: disable-end

    return (feeLower * (end - timeToMaturity) + feeUpper * (timeToMaturity - start)) / (end - start);
}
```

首先，我们需要理解为什么存在系列 `settlementFeeCbp` 变量，这是因为 Midnight 为距离到期日不同的市场设置了不同的参数，分别为 0d / 1d / 7d / 30d / 90d / 180d / 360d，这 7 个时间点对应 `settlementFeeCbp0 - 6` 变量。然后 `timeToMaturity` 会落到不同的时间点中，我们会进行插值处理，使用的公式是:
$$
\mathrm{settlementFee}(t) = F_{\text{lower}} + (F_{\text{upper}} - F_{\text{lower}}) \cdot \frac{t - T_{\text{start}}}{T_{\text{end}} - T_{\text{start}}}
$$
而代码内的 `(feeLower * (end - timeToMaturity) + feeUpper * (timeToMaturity - start)) / (end - start)` 实际上就是上述插值公式进行变化获得，变化过程如下:
$$
\begin{align}
\mathrm{settlementFee}(t) &= F_{\text{lower}} + (F_{\text{upper}} - F_{\text{lower}}) \cdot \frac{t - T_{\text{start}}}{T_{\text{end}} - T_{\text{start}}}\\\\
&= \frac{F_{\text{lower}} \cdot (T_{\text{end}} - T_{\text{start}}) +  (F_{\text{upper}} - F_{\text{lower}}) \cdot (t - T_{\text{start}})}{T_{\text{end}} - T_{\text{start}}}\\\\
&= \frac{F_{\text{lower}} \cdot T_{\text{end}} +  (F_{\text{upper}} - F_{\text{lower}}) \cdot t - F_{\text{upper}}\cdot T_{\text{start}}}{T_{\text{end}} - T_{\text{start}}}\\\\
&= \frac{F_{\text{lower}} \cdot (T_{\text{end}} - t) + F_{\text{upper}} \cdot (t - T_{\text{start}})}{T_{\text{end}} - T_{\text{start}}}
\end{align}
$$
上述变化大优势是可以增加分子的数值大小，减少一定程度的舍入误差。

## Tick 机制

在上文中，我们介绍的 `Offer` 内出现了 `tick` 字段，而 `MarketState` 内也出现 `tickSpacing` 字段。这些都是 midnight tick 机制所需的参数。在 Uniswap v3 和 v4 中，tick 与 price 之间的映射满足如下函数:
$$
p(i) = 1.0001^i
$$
当然，用户不能任意选择 tick 进行流动性放置，因为 Uniswap v3 规定了 `tickSpacing`，并且存在 ` if (tick % tickSpacing != 0) revert TickMisaligned(tick, tickSpacing);` 的约束。在 midnight 内，价格与 tick $n$ 之间的关系满足如下函数:
$$
p(n) = \frac{1}{1 + {1.005}^{N / 2 - n}}
$$
上述公式内出现了没有给出含义的字母 $N$，该字母的含义为 max tick，在合约内是一个常数，数值为 `5820`，也就是说在 Midnight 内部 tick 的取值范围是 $[0, 5820]$。所以我们可以使用上述公式计算一下 tick 范围对应的价格范围:
$$
\begin{align}
p(0) &= \frac{1}{1 + 1.005^{2910}} = 0.0000004 \approx 0\\\\
p(2910) &= \frac{1}{1 + 1.005^{0}} = 0.5\\\\
p(5820) &= \frac{1}{1 + 1.005^{-2910}} = 0.9999995 \approx 1
\end{align}
$$
上述的 $\approx$ 并不是随便约等于的，在合约内， `p(0)` 计算结果就是 `0` 而 `p(5820)` 的结果结果就是 `1`。这与合约底层的数值计算逻辑有关。合约内的 `tickToPrice` 的函数定义如下:

```solidity
/// @dev Returns x / d rounded to the nearest integer with ties rounded down, without checking for overflow.
function divHalfDownUnchecked(uint256 x, uint256 d) internal pure returns (uint256) {
    unchecked {
        return (x + (d - 1) / 2) / d;
    }
}

function tickToPrice(uint256 tick) internal pure returns (uint256) {
    require(tick <= MAX_TICK, TickOutOfRange());
    unchecked {
        // forge-lint: disable-next-item(unsafe-typecast)
        return uint256(1e36)
                .divHalfDownUnchecked(1e18 + wExp(LN_ONE_PLUS_DELTA * (int256(MAX_TICK / 2) - int256(tick))))
                .divHalfDownUnchecked(PRICE_ROUNDING_STEP) * PRICE_ROUNDING_STEP;
    }
}
```

其中 `divHalfDownUnchecked` 是一个四舍五入的算法，并且对于 `0.5` 数值会向下舍入。有了此知识，我们就可以看到 `uint256(1e36).divHalfDownUnchecked(1e18 + wExp(LN_ONE_PLUS_DELTA * (int256(MAX_TICK / 2) - int256(tick)))) `其实就是我们上文给出的计算公式。当然，稍有不同的是为了进行简化计算，solidity 内的实现其实是下面这个使用对数的计算公式:
$$
\begin{align}
p(n) &= \frac{1}{1 + {1.005}^{N / 2 - n}}\\\\
&= \frac{1}{1 + \exp((N / 2 - n) \times\ln 1.005)}
\end{align}
$$
上述代码内的 `LN_ONE_PLUS_DELTA` 其实是 `floor(ln(1.005) * 1e18)` 的值。完成 `uint256(1e36).divHalfDownUnchecked(1e18 + wExp(LN_ONE_PLUS_DELTA * (int256(MAX_TICK / 2) - int256(tick))))` 计算后，我们获得的是一个 `1e18` 精度的数值，在大部分协议中，我们就可以直接返回该数值。

但在 Midnight 内，我们虽然约定 price 的精度是 `1e18`，但是我们同时约定 price 使用 `1e12` 作为步长，换言之，price 虽然是 18 位定点小数，但其实只有 6 位有效精度。所以此处还需要进行一次有效数字调整，我们需要清理计算结果，这就是 `.divHalfDownUnchecked(PRICE_ROUNDING_STEP) * PRICE_ROUNDING_STEP;` 的计算原因。其中 `PRICE_ROUNDING_STEP = 1e12`。

上述讨论中，我们分析了计算 price 的方法，但是始终没有介绍上述公式的金融原理。在本文开篇时，我们介绍了在 Midnight 内，设计者没有对利率和价格的关系进行复杂设计，而是直接使用了经典的零息债券的利率计算方法，即:
$$
r = \frac{1}{p} - 1
$$
注意此处的 $p$ 其实是一个贴现因子。随着 p 增加，对应利率降低。然后，midnight 设计者规定利率从每一个 tick 之间的利率变动是 1.005，这是一个比值，所以我们需要一个初始值来构成等比数列。我们此处假设利率锚定点的利率为 $r\_0$ ，其对应的 tick 为 $t\_0$。所以我们得到如下公式(此处使用 $-n$ 的原因是我们需要构建一个随着 $n$ (此处的 $n$ 指的是 tick 数值)增加，而 p 逐渐增加的增函数):
$$
\begin{align}
r_01.005^{t_0-n} &= \frac{1}{p} - 1\\\\
\frac{1}{p} &= r_01.005^{t_0-n} + 1\\\\
p &= \frac{1}{r_01.005^{t_0-n} + 1}
\end{align}
$$
显然为了简化后续计算，我们可以将 $r\_0 = 1$ ，而 $t\_0 = N / 2$ 作为锚定，我们之所以这么设计参数，单纯是为了简化后续的计算。给定上述参数后，我们就可以获得如下结果:
$$
p(n) = \frac{1}{1 + {1.005}^{N / 2 - n}}
$$
至于后续如何获得 max tick。我们需要首选确定价格的有效精度误差 $\epsilon$ ，比如 midnight 选择 $\epsilon = 1e-6$，所以我们可以获得如下方程:
$$
p(0) = \frac{1}{1 + 1.005^{N/2}} < \frac{\epsilon}{2} = 5 \times 10^{-7}
$$
读者不难使用任何现代计算工具利用对数求解上述方程，结果为 $N = 5818$。但需要注意 midnight 内存在 `uint8 constant DEFAULT_TICK_SPACING = 4;` 默认配置，所以 $N$ 应该与默认的 tick spacing 配对，所以此处 midnight 选择了 `N = 5820`。

但与 Uniswap v4 等确定 tick spacing 就无法修改的情况不同，midnight 允许在市场存续期间对市场的 `tickSpacing` 进行修改。当然，不是所有人都有权修改 market 的 `tickSpacing`，而是存在一个专门的 `tickSpacingSetter` 角色。注意，这个角色不是市场创建者，而是 midnight 协议方设置的。

```solidity
/// @dev Refines the tick spacing of a market. Can not increase (more ticks become accessible).
function setMarketTickSpacing(bytes32 id, uint256 newTickSpacing) external {
    require(msg.sender == tickSpacingSetter, OnlyTickSpacingSetter());
    require(marketState[id].tickSpacing > 0, MarketNotCreated());
    require(newTickSpacing > 0 && marketState[id].tickSpacing % newTickSpacing == 0, InvalidTickSpacing());
    // forge-lint: disable-next-line(unsafe-typecast) as newTickSpacing <= DEFAULT_TICK_SPACING < type(uint8).max
    marketState[id].tickSpacing = uint8(newTickSpacing);
    emit EventsLib.SetMarketTickSpacing(id, newTickSpacing);
}
```

调整 `tickSpacing` 肯定是不能随意给定数值的，因为这可能导致之前的挂单的 tick 失效。我们可以看到上文代码内的 `marketState[id].tickSpacing % newTickSpacing == 0` 就是约束 `newTickSpacing` 设置的，这意味着 `newTickSpacing` 只能是原来的 `tickSpacing` 整数倍变化，这意味着使用旧的 `tickSpacing` 挂出的订单在 `newTickSpacing` 也不会失效。

额外的，实际上 market 创建者是不能在创建市场时给定 `tickSpacing` 的。创建市场需要使用如下函数:

```solidity
/// @dev Returns the market id and creates the market if it doesn't exist yet.
function touchMarket(Market memory market) public returns (bytes32) {
    bytes32 id = toId(market);
    if (marketState[id].tickSpacing == 0) {
    		MarketState storage _marketState = marketState[id];
    		_marketState.tickSpacing = DEFAULT_TICK_SPACING;
				...
				
        IdLib.storeInCode(market, INITIAL_CHAIN_ID);

        emit EventsLib.MarketCreated(market, id);
    }
    return id;
}
```

上述函数内的 `Market` 结构体在上文已经介绍过了，该结构体内实际上不包含 `tickSpacing` 字段。我们可以看到上文代码内直接将 `MarketState` 内的 `tickSpacing` 设置为了 `DEFAULT_TICK_SPACING`(即数值 `4`)。此处，我们还可以看到 `marketState[id].tickSpacing` 也作为判断 market 是否初始化的一个标志位。

额外的，我们可以注意到此处有一个 `IdLib.storeInCode(market, INITIAL_CHAIN_ID);` 函数，该函数是一个用于存储 market 配置的函数，该函数的作用是将 market 以 bytecode 的形式直接写入一个地址内部。此处开发者直接手动编写了一段字节码 `600b380380600b5f395ff3`，该字节码含义就是读取 `600b380380600b5f395ff3` 后续被追加的 `market` 数据，使用 `CODECOPY` 将其拷贝到内存中，然后使用 `RETURN` 方法返回。

```solidity
library IdLib {
    error SStore2DeploymentFailed();

    /// @dev Used as a prefix to some data, to give a creation code that deploys the data as runtime bytecode.
    /// @dev Explanation of the prefix:
    /// hex       opcode          stack              comments
    /// ------------------------------------------------------------------------------
    /// 60 0b     PUSH1 0x0b      [11]               11 = length(prefix)
    /// 38        CODESIZE        [codesize, 11]
    /// 03        SUB             [len]              with len = codesize - 11
    /// 80        DUP1            [len, len]
    /// 60 0b     PUSH1 0x0b      [11, len, len]     code offset = 11
    /// 5f        PUSH0           [0, 11, len, len]  mem offset = 0
    /// 39        CODECOPY        [len]              mem[0:len] <- code[11:11+len]
    /// 5f        PUSH0           [0, len]           return offset = 0
    /// f3        RETURN          []                 mem[0:len] is returned
    bytes constant SSTORE2_PREFIX = hex"600b380380600b5f395ff3";

    function toId(Market memory market, uint256 chainId, address midnight) internal pure returns (bytes32) {
        return keccak256(
            abi.encodePacked(
                uint8(0xff), midnight, chainId, keccak256(abi.encodePacked(SSTORE2_PREFIX, abi.encode(market)))
            )
        );
    }

    /// @dev Stores the data in the code of the contract at the given address.
    /// @dev Uses the given chain id as salt.
    function storeInCode(Market memory market, uint256 chainId) internal returns (address create2Address) {
        bytes memory creationCode = abi.encodePacked(SSTORE2_PREFIX, abi.encode(market));
        assembly ("memory-safe") {
            create2Address := create2(0, add(creationCode, 0x20), mload(creationCode), chainId)
        }
        require(create2Address != address(0), SStore2DeploymentFailed());
    }
}

```

此处我们也可以注意到 `toId` 与 `storeInCode` 之间的关系。在过去，Morpho 都是直接使用结构体的哈希值作为 `id`，但是这对于用户并不友好。给定一个 `id`，用户在不检索 event 的情况下并不能知道 `id` 对应的市场是什么。在 Midnight 内，开发者注意到了这个问题，所以目前的 `toId` 实际上计算的就是 `storeInCode` 函数存储 `market` 数据的合约地址(`toId` 内部其实就是 `create2` 地址计算)。更加严谨地说，应该是 `toId` 返回的 256 bit 值的后 160 bit 对应 market 结构体在链上存储的地址。这样用户看到一个 `marketId` 就可以通过读取合约字节码知道该市场对应的结构体。

## Fee 机制

在上文中，我们其实已经对 Midnight 的 Fee 机制有了大概介绍，我们主要介绍了两种 Fee 机制:

1. CONTINUOUS FEES，作用于 credit 的 fee 机制，用于协议运营方从 credit 存款利息内提取收入，我们主要围绕着 `pendingFee` 介绍该机制。在 credit 出现后，比如通过 `buy` 买入 units 后，`pendingFee` 内就会计入买入数量的 credit 的从当前到到期日的 continuous fee，后续 update position 时将其计提到协议方账本 `continuousFeeCredit` 内部。额外需要注意的
2. SETTLEMENT FEES，该机制作用于 take 函数，在上文内，我们也介绍过该机制，该机制本子上就是在 take 过程中制造 buy price 和 sell price 之间的 gap，该 gap 就是协议运营方的收入，该收入位于 `claimableSettlementFee` 账本内

我们首先介绍 continuous fee 机制。协议方可以使用 `setDefaultContinuousFee(address loanToken, uint256 newContinuousFee)` 函数规定某种可借资产的的默认费率。我们可以在 `touchMarket` 函数看到 `defaultContinuousFee` 的作用：
```solidity
/// @dev Returns the market id and creates the market if it doesn't exist yet.
function touchMarket(Market memory market) public returns (bytes32) {
    bytes32 id = toId(market);
    if (marketState[id].tickSpacing == 0) {
        require(market.maturity <= block.timestamp + 100 * 365 days, MaturityTooFar());
        require(market.collateralParams.length > 0, NoCollateralParams());
        require(market.collateralParams.length <= MAX_COLLATERALS, TooManyCollateralParams());
        address previousCollateralToken;
        for (uint256 i = 0; i < market.collateralParams.length; i++) {
            address collateralToken = market.collateralParams[i].token;
            require(collateralToken > previousCollateralToken, CollateralParamsNotSorted());
            uint256 lltv = market.collateralParams[i].lltv;
            require(isLltvAllowed(lltv), LltvNotAllowed());
            require(
                market.collateralParams[i].maxLif == maxLif(lltv, LIQUIDATION_CURSOR_LOW)
                    || market.collateralParams[i].maxLif == maxLif(lltv, LIQUIDATION_CURSOR_HIGH),
                InvalidMaxLif()
            );
            previousCollateralToken = collateralToken;
        }

        MarketState storage _marketState = marketState[id];
        _marketState.tickSpacing = DEFAULT_TICK_SPACING;
        uint16[7] memory _defaultSettlementFeeCbp = defaultSettlementFeeCbp[market.loanToken];
        _marketState.settlementFeeCbp0 = _defaultSettlementFeeCbp[0];
        _marketState.settlementFeeCbp1 = _defaultSettlementFeeCbp[1];
        _marketState.settlementFeeCbp2 = _defaultSettlementFeeCbp[2];
        _marketState.settlementFeeCbp3 = _defaultSettlementFeeCbp[3];
        _marketState.settlementFeeCbp4 = _defaultSettlementFeeCbp[4];
        _marketState.settlementFeeCbp5 = _defaultSettlementFeeCbp[5];
        _marketState.settlementFeeCbp6 = _defaultSettlementFeeCbp[6];
        _marketState.continuousFee = defaultContinuousFee[market.loanToken];
        IdLib.storeInCode(market, INITIAL_CHAIN_ID);

        emit EventsLib.MarketCreated(market, id);
    }
    return id;
}
```

在此处，我们直接给出 `touchMarket` 的完整代码。我们可以看到用户在创建 Market 时 `_marketState.continuousFee` 会被自动设置为 `defaultContinuousFee[market.loanToken]`。此处我们可以看到一个上文内没有介绍的额外细节，即 `market.collateralParams` 内的 token 被要求按照顺序排列。这是一种数据规范化的行为，保证了对于同一个担保品集合，各方计算的 MarketId 一致，同时也可以保证 BitMap 被正确使用。如果不进行排序，那么 bitmap 与担保品集合之间就无法建立稳定的映射关系。

> 额外的，此处我们可以看到 `maxLif` 不是可以随便设置的，而是必须要等于 `maxLif` 函数基于 `lltv`  和 `LIQUIDATION_CURSOR_LOW / LIQUIDATION_CURSOR_HIGH` 的计算结果

除了 `setDefaultContinuousFee` 外，协议方有权更新某一个市场内的 continuous fee，具体函数如下:

```solidity
function setMarketContinuousFee(bytes32 id, uint256 newContinuousFee) external {
    MarketState storage _marketState = marketState[id];
    require(msg.sender == feeSetter, OnlyFeeSetter());
    require(newContinuousFee <= MAX_CONTINUOUS_FEE, ContinuousFeeTooHigh());
    require(_marketState.tickSpacing > 0, MarketNotCreated());
    // forge-lint: disable-next-line(unsafe-typecast) as newContinuousFee <= MAX_CONTINUOUS_FEE < type(uint32).max
    _marketState.continuousFee = uint32(newContinuousFee);
    emit EventsLib.SetMarketContinuousFee(id, newContinuousFee);
}
```

当然，此处我们需要注意，对于已经存在的 Position 而言，Position 内使用的 continuous fee 费率不会有任何变化。这是因为在 Postion 内的 `pendingFee` ，我们已经记录了按照过去费率计算的从 Position 内的 `credit` 从 `lastAccrual` 到 maturity 内 continuous fee 数值，而新的费率不会影响 `pendingFee` 内的已有数值。但是假如 Position 的持有者增加 Position 内的 credit，那么此时新的 credit 就会使用新的费率。简单来说，Position 内的 credit 是按照增加时刻的 continuous fee 费率进行计算的，后续任何的 continuous fee 调整都不会溯及过往。比如，我们首先在 0.03% 费率市场条件下建立 100 credit，那么这 100 credit 就按照 0.03% 费率收取费用。假如此时协议方将费率修改为 0.05% 并不会影响按照  0.03% 费率的 100 credit。假如此时我们希望增加一部分头寸，我们又增加了 150 credit，那么这 150 credit 会按照 0.05% 费率收取，此时我们的 200 credit 的头寸的费率收取情况如下:

1. 100 credit 头寸按照 0.03% 费率收取
2. 150 credit 头寸按照 0.05% 费率收取

上述机制的原理本质上还是之前一直提及的 `pendingFee` 的原理，由于开仓时，我们就记录了开仓 credit 从开仓时刻到 maturity 的手续费，所以市场的  continuous fee 调整不会影响已开仓头寸。我们可以先看一下由于更新头寸的 `pendingFee` 的 `updatePositionView` 函数:

```solidity
function updatePositionView(Market memory market, bytes32 id, address user)
    public
    view
    returns (uint128, uint128, uint128)
{
    Position storage _position = position[id][user];
    uint128 credit = _position.credit;
    uint128 _lastLossFactor = _position.lastLossFactor;
    uint256 postSlashCredit = _lastLossFactor < type(uint128).max
        ? credit.mulDivDown(type(uint128).max - marketState[id].lossFactor, type(uint128).max - _lastLossFactor)
        : 0;
    uint128 _pendingFee = _position.pendingFee;
    uint256 postSlashPendingFee =
        credit > 0 ? _pendingFee - _pendingFee.mulDivUp(credit - postSlashCredit, credit) : 0;
    uint256 accrualEnd = UtilsLib.min(block.timestamp, market.maturity);
    uint128 _lastAccrual = _position.lastAccrual;
    // forge-lint: disable-next-item(unsafe-typecast) as fee <= pending <= credit which are uint128 position fields
    uint128 fee = _lastAccrual < market.maturity
        ? uint128(postSlashPendingFee.mulDivDown(accrualEnd - _lastAccrual, market.maturity - _lastAccrual))
        : 0;
    // forge-lint: disable-next-item(unsafe-typecast) as credit and pending are <= uint128 position fields
    return (uint128(postSlashCredit) - fee, uint128(postSlashPendingFee) - fee, fee);
}
```

其实在上文中，我们已经介绍过 `updatePositionView` 的工作原理，核心逻辑是首先基于 Position 内的 `lastLossFactor` 和目前的 `marketState[id].lossFactor` 内的数值计算当前 Position 需要承担的坏帐损失，然后计算坏账损失后的 credit 数值 `postSlashCredit`。基于 `postSlashCredit` ，我们计算出 pending fee 部分需要承担的损失。这是因为协议方收取的 continuous fee 也是需要承担坏账的，所以此处我们需要根据坏账调整 pendingFee 的数值。最后，根据时间差值计算用户的 `pendingFee` 需要调整的数值。

> 为了方便用户查询数据，在 Midnight 内引入了一种特殊的机制，即对于存在状态更新的函数，midnight 会提供一个无状态的版本用于外部第三方读取数据而不修改头寸信息，比如上文的 `updatePositionView` ，这个函数只负责读取信息进行计算，但是不会进行数据更新操作。而真实的数据更新会在 `_updatePosition` 函数内执行，这个函数负责将 `updatePositionView` 返回的信息真正写入 storage。这种编程范式存在很多好处，值的学习

在 `_updatePosition` 函数内，我们可以看到最后一步的 Position 更新过程:

```solidity
function _updatePosition(Market memory market, bytes32 id, address user)
    internal
    returns (uint128, uint128, uint128)
{
    Position storage _position = position[id][user];
    (uint128 newCredit, uint128 newPendingFee, uint128 accruedFee) = updatePositionView(market, id, user);

    uint128 creditDecrease = _position.credit - newCredit;
    uint128 pendingFeeDecrease = _position.pendingFee - newPendingFee;

    _position.credit = newCredit;
    _position.lastLossFactor = marketState[id].lossFactor;
    _position.pendingFee = newPendingFee;
    _position.lastAccrual = uint128(block.timestamp);
    marketState[id].continuousFeeCredit += UtilsLib.toUint128(accruedFee);

    emit EventsLib.UpdatePosition(id, user, creditDecrease, pendingFeeDecrease, accruedFee);

    return (newCredit, newPendingFee, accruedFee);
}
```

此处，我们可以看到每一次更新就是将 `pendingFee` 内的一些金额移动到 `continuousFeeCredit` 内部。在 continuous fee 的生命周期的最后一步就是协议方调用函数提取这部分收益，具体函数如下:

```solidity
function claimContinuousFee(Market memory market, uint256 amount, address receiver) external {
    bytes32 id = toId(market);
    MarketState storage _marketState = marketState[id];
    require(msg.sender == feeClaimer, OnlyFeeClaimer());
    require(_marketState.tickSpacing > 0, MarketNotCreated());

    _marketState.continuousFeeCredit -= UtilsLib.toUint128(amount);
    _marketState.totalUnits -= UtilsLib.toUint128(amount);
    _marketState.withdrawable -= UtilsLib.toUint128(amount);

    emit EventsLib.ClaimContinuousFee(msg.sender, id, amount, receiver);

    SafeTransferLib.safeTransfer(market.loanToken, receiver, amount);
}
```

与大家想象的直接提取资产不同，`claimContinuousFee` 函数需要考虑 midnight 内的不变量问题。因为 `claimContinuousFee` 行为其实和用户的 `withdraw` 行为类似，都涉及到 units 的消灭问题，此处可以可以看到除了 ` _marketState.continuousFeeCredit` 需要减少外，`_marketState.totalUnits` 因为 units 消灭也需要减少。而 `_marketState.withdrawable` 则记录了当前系统内可以提取的 loan token 数量。在 repay 以及 liquidate(可以视为一种特殊的 repay) 内会增加 `_marketState.withdrawable` 的数值。

那接下来，我们要介绍最后一种涉及 continuous fee 的最终情况，即存款人调用 `withdraw` 函数的情况。`withdraw` 函数是存款人退出协议的最终方法，我们可以看到此处对于 `pendingFeeDecrease` 的计算是直接使用用户 withdraw 的 units 数量占 credit 的比例进行计算的。

```solidity
function withdraw(Market memory market, uint256 units, address onBehalf, address receiver) external {
    require(onBehalf == msg.sender || isAuthorized[onBehalf][msg.sender], Unauthorized());
    bytes32 id = touchMarket(market);
    MarketState storage _marketState = marketState[id];
    _updatePosition(market, id, onBehalf);

    Position storage _position = position[id][onBehalf];
    uint128 pendingFeeDecrease;
    if (_position.credit > 0) {
        pendingFeeDecrease = UtilsLib.toUint128(_position.pendingFee.mulDivUp(units, _position.credit));
        _position.pendingFee -= pendingFeeDecrease;
    }
    _position.credit -= UtilsLib.toUint128(units);
    _marketState.withdrawable -= UtilsLib.toUint128(units);
    _marketState.totalUnits -= UtilsLib.toUint128(units);

    emit EventsLib.Withdraw(msg.sender, id, units, onBehalf, receiver, pendingFeeDecrease);

    SafeTransferLib.safeTransfer(market.loanToken, receiver, units);
}
```

此处需要注意由于使用了零息债券的机制，所以用户 withdraw 是可以直接获取全部利息的，无论是否真正到期，当然，相反的，借款人提取使用 repay 还款也需要支付全部利息。所以假如借款人提前还款，那么存款人其实也可以提前 withdraw 离开。当然这种情况只有接近到期日或者到期日后才会出现。假如还没有到达到期日，借款人使用 take 方法转移债务进行还款时更加合适的。

然后，我们介绍一次性的收费 settlement fee。在上文中，我们其实已经介绍了 settlement fee 的工作原理，这是一个在订单成交时收取的一次性费用，费用以交易双方的成交价差体现，且只对 taker 收取。我们首先关注 `settlementFeeCbp` 系列函数是如何设置的。在刚刚给出的 `touchMarket` 完整代码内，我们可以看到 `_marketState.settlementFeeCbp0 = _defaultSettlementFeeCbp[0];` 类似代码，市场创建时，我们会直接使用 `_defaultSettlementFeeCbp` 内的数值。

那 `_defaultSettlementFeeCbp` 是如何初始化的，我们可以看到如下函数:
```solidity
function setDefaultSettlementFee(address loanToken, uint256 index, uint256 newSettlementFee) external {
    require(msg.sender == feeSetter, OnlyFeeSetter());
    require(index <= 6, InvalidFeeIndex());
    require(newSettlementFee <= maxSettlementFee(index), SettlementFeeTooHigh());
    require(newSettlementFee % CBP == 0, FeeNotMultipleOfFeeCbp());
    // forge-lint: disable-next-item(unsafe-typecast) as newSettlementFee <= maxSettlementFee <= uint16.max * CBP
    defaultSettlementFeeCbp[loanToken][index] = uint16(newSettlementFee / CBP);
    emit EventsLib.SetDefaultSettlementFee(loanToken, index, newSettlementFee);
}
```

这里我们可以看到一个精度调整问题，在上文内，我们分析 tick 机制时提到 tick 机制下，price 虽然是 `1e18` (即 `uint256 constant WAD = 1e18;`)的精度，但是实际精度其实是 `1e12`(即 `uint256 constant CBP = 1e12;`)。在这里，我们可以看到设置 `SettlementFee` 时，此处也使用了类似的精度，`feeSetter` 调用函数时写入的是 `WAD` 精度的 `newSettlementFee`，但是内部会使用 `newSettlementFee % CBP == 0` 检查实际精度是不是 CBP。最后写入的时候，我们会直接将其写入为 CBP 精度 `defaultSettlementFeeCbp[loanToken][index] = uint16(newSettlementFee / CBP);`。此处需要注意，在用于计算 `settlementFee` 的函数内，我们会进行如下精度还原:
```solidity
/// @dev Returns the settlement fee using piecewise linear interpolation between breakpoints.
function settlementFee(bytes32 id, uint256 timeToMaturity) public view returns (uint256) {
    MarketState storage _marketState = marketState[id];
    require(_marketState.tickSpacing > 0, MarketNotCreated());

    if (timeToMaturity >= 360 days) return _marketState.settlementFeeCbp6 * CBP;

    // forgefmt: disable-start
    (uint256 start, uint256 end, uint256 feeLower, uint256 feeUpper) =
        timeToMaturity < 1 days   ? (  0 days,   1 days, _marketState.settlementFeeCbp0 * CBP, _marketState.settlementFeeCbp1 * CBP) :
        timeToMaturity < 7 days   ? (  1 days,   7 days, _marketState.settlementFeeCbp1 * CBP, _marketState.settlementFeeCbp2 * CBP) :
        timeToMaturity < 30 days  ? (  7 days,  30 days, _marketState.settlementFeeCbp2 * CBP, _marketState.settlementFeeCbp3 * CBP) :
        timeToMaturity < 90 days  ? ( 30 days,  90 days, _marketState.settlementFeeCbp3 * CBP, _marketState.settlementFeeCbp4 * CBP) :
        timeToMaturity < 180 days ? ( 90 days, 180 days, _marketState.settlementFeeCbp4 * CBP, _marketState.settlementFeeCbp5 * CBP) :
                                    (180 days, 360 days, _marketState.settlementFeeCbp5 * CBP, _marketState.settlementFeeCbp6 * CBP);
    // forgefmt: disable-end

    return (feeLower * (end - timeToMaturity) + feeUpper * (timeToMaturity - start)) / (end - start);
}
```

但需要额外注意，此处的 `settlementFee` 没有最终强制将输出的结果转化为 CBP 精度，所以此处返回的结果仍是 WAD 精度。

在 `setDefaultSettlementFee` 内也出现了  `maxSettlementFee` 函数，这个函数是用来检查设置的费率是否超过预期，具体最大费率数值如下:
```solidity
uint256 constant MAX_SETTLEMENT_FEE_0_DAYS = 0.000014e18;
uint256 constant MAX_SETTLEMENT_FEE_1_DAY = 0.000014e18;
uint256 constant MAX_SETTLEMENT_FEE_7_DAYS = 0.000098e18;
uint256 constant MAX_SETTLEMENT_FEE_30_DAYS = 0.000417e18;
uint256 constant MAX_SETTLEMENT_FEE_90_DAYS = 0.00125e18;
uint256 constant MAX_SETTLEMENT_FEE_180_DAYS = 0.0025e18;
uint256 constant MAX_SETTLEMENT_FEE_360_DAYS = 0.005e18;
```

我们可以看到随着时间增加，Settlement Fee 上限也不断增加，这个符合我们的经济学常识的，因为对于短期的固定利率债券交易而言，如果征收过高的费率，那么交易费率很有可能会占据债券交易的大部分利润。

在市场初始化后，协议方实际上还是可以使用 `setMarketSettlementFee` 调整市场的 `settlementFee` 数值的，这个函数实现也很简单:

```solidity
function setMarketSettlementFee(bytes32 id, uint256 index, uint256 newSettlementFee) external {
    MarketState storage _marketState = marketState[id];
    require(msg.sender == feeSetter, OnlyFeeSetter());
    require(index <= 6, InvalidFeeIndex());
    require(newSettlementFee <= maxSettlementFee(index), SettlementFeeTooHigh());
    require(newSettlementFee % CBP == 0, FeeNotMultipleOfFeeCbp());
    require(_marketState.tickSpacing > 0, MarketNotCreated());
    // forge-lint: disable-next-item(unsafe-typecast) as newSettlementFee <= maxSettlementFee <= uint16.max * CBP
    uint16 newSettlementFeeCbp = uint16(newSettlementFee / CBP);
    if (index == 0) _marketState.settlementFeeCbp0 = newSettlementFeeCbp;
    else if (index == 1) _marketState.settlementFeeCbp1 = newSettlementFeeCbp;
    else if (index == 2) _marketState.settlementFeeCbp2 = newSettlementFeeCbp;
    else if (index == 3) _marketState.settlementFeeCbp3 = newSettlementFeeCbp;
    else if (index == 4) _marketState.settlementFeeCbp4 = newSettlementFeeCbp;
    else if (index == 5) _marketState.settlementFeeCbp5 = newSettlementFeeCbp;
    else if (index == 6) _marketState.settlementFeeCbp6 = newSettlementFeeCbp;
    emit EventsLib.SetMarketSettlementFee(id, index, newSettlementFee);
}
```

在 `take` 函数内，计算完成了 settlement fee 后，`claimableSettlementFee[offer.market.loanToken] += buyerAssets - sellerAssets;` 会将其记录一下，后续可以通过 `claimSettlementFee` 进行提取。

```solidity
function claimSettlementFee(address token, uint256 amount, address receiver) external {
    require(msg.sender == feeClaimer, OnlyFeeClaimer());
    claimableSettlementFee[token] -= amount;
    emit EventsLib.ClaimSettlementFee(msg.sender, token, amount, receiver);
    SafeTransferLib.safeTransfer(token, receiver, amount);
}
```

至此，我们就再次介绍了 Midnight 内的 Fee 机制，主要分为用于作用于利息收入的 continous fee 和单笔交易中直接作用于 taker 的 settlement fee。

## Take 函数

`take` 函数内的大部分代码在上文已经零零散散的进行了介绍，`take` 函数作为 midnight 内最核心的函数主要用于订单成交。在本节中，我们将逐行介绍 `take` 函数的具体内容。

首先我们介绍一下 `take` 函数的定义:

```solidity
function take(
    Offer memory offer,
    bytes memory ratifierData,
    uint256 units,
    address taker,
    address receiverIfTakerIsSeller,
    address takerCallback,
    bytes memory takerCallbackData
) external returns (uint256, uint256) {
```

`take` 函数会使用如下几个参数:

1. 我们上文介绍的 `Offer` 结构体
2. 用于辅助证明 `Offer` 确实存在的 `ratifierData` 字段(用于调用 Offer 内的 `ratifier` 字段确定 Offer 存在)
3. `units` 代表交易的数量，可以部分成交订单
4. `taker` 代表吃单者，此处 `take` 函数要求 `require(taker == msg.sender || isAuthorized[taker][msg.sender], TakerUnauthorized());`
5. 在上文内介绍过的 `receiverIfTakerIsSeller` 的作用，该参数决定了 loan token 的转入地址，具体作用机制是 `address receiver = offer.buy ? receiverIfTakerIsSeller : offer.receiverIfMakerIsSeller;`
6. `takerCallback` 用于指定 take 完成后调用的合约地址，视成交情况会调用该地址的 `onBuy` 或 `onSell` 函数
7. `takerCallbackData` 用于配合 `takerCallback` 工作，指定调用 `onBuy` 或 `onSell` 函数时的 calldata

首先，我们分析 `take` 函数的第一部分，该部分主要检查入参是否合理：

```solidity
require(taker == msg.sender || isAuthorized[taker][msg.sender], TakerUnauthorized());
bytes32 id = touchMarket(offer.market);
MarketState storage _marketState = marketState[id];
require(_marketState.lossFactor < type(uint128).max, MarketLossFactorMaxedOut());
require(UtilsLib.atMostOneNonZero(offer.maxAssets, offer.maxUnits), MultipleNonZero());
require(offer.tick % _marketState.tickSpacing == 0, TickNotAccessible());
require(block.timestamp >= offer.start, OfferNotStarted());
require(block.timestamp <= offer.expiry, OfferExpired());
require(offer.maker != taker, SelfTake());
require(isAuthorized[offer.maker][offer.ratifier], RatifierUnauthorized());
require(IRatifier(offer.ratifier).isRatified(offer, ratifierData) == CALLBACK_SUCCESS, RatifierFail());
```

这些检查的作用分别是:

1. 检查 `take` 函数内的 `taker` 参数是否与交易发起者存在授权关系
2. 避免向 `_marketState.lossFactor == type(uint128).max` 的市场继续交易，这种市场代表 `credit` 已经归零，或者说所有的资产都被用于社会化坏账。在这种极端市场内，存款者在 `withdraw` 时无法获得任何资产
3. `UtilsLib.atMostOneNonZero(offer.maxAssets, offer.maxUnits)` 用于 Offer 结构限制，我们在上文介绍 Offer 时提到 `offer.maxAssets` 和 `offer.maxUnits` 只能设置一个
4. `offer.tick % _marketState.tickSpacing == 0` 是为了避免用户在不符合 `tickSpacing` 的 tick 上进心挂单和吃单操作
5. `offer.start` 和 `offer.expiry` 操作是为了保证参数正确
6. `offer.maker != taker` 是为了避免自成交。我们会在后文内看到 `credit` 和 `debt` 在 Position 内只能有一个非零，在这个前提下，很多代码可以被简化，所以此处的 `offer.maker != taker` 就是避免  `credit` 和 `debt` 出现都非零的情况
7. `isAuthorized[offer.maker][offer.ratifier]` 检查 offer 内的 maker 和 `ratifier` 是否匹配
8. 最后的 `IRatifier(offer.ratifier).isRatified(offer, ratifierData) == CALLBACK_SUCCESS` 就是最终的 offer 有效性检查，确保 `take` 函数内用户传入的 `Offer` 是真实存在的

接下来，我们将 `take` 内输入 `uints` 转化为 assets 并处理 settlement fee 问题。在 `offer` 内，价格是使用 `tick` 定义的，所以第一步我们需要使用 `tickToPrice` 将 tick 转化为 price，然后我们可以使用 `timeToMaturity` 计算 `settlementFee`，此处的 `id` 是指 market id。计算完成后，正如上文所述，我们会将 `settlementFee` 作用在 taker 的成交价格上，对于 `offer.buy = true` 的情况，taker 是卖方(seller)，所以此处需要 `offerPrice - _settlementFee` 使得卖方以更低价格买出，此时买入价格(`buyerPrice = sellerPrice + _settlementFee = offerPrice`)和卖出价格的差价就是 settlemet fee。对于 `offer.buy =  false` 的情况，也是类似的。

```solidity
uint256 offerPrice = TickLib.tickToPrice(offer.tick);
uint256 timeToMaturity = UtilsLib.zeroFloorSub(offer.market.maturity, block.timestamp);
uint256 _settlementFee = settlementFee(id, timeToMaturity);
uint256 sellerPrice = offer.buy ? offerPrice - _settlementFee : offerPrice;
uint256 buyerPrice = sellerPrice + _settlementFee;
uint256 buyerAssets = offer.buy ? units.mulDivDown(buyerPrice, WAD) : units.mulDivUp(buyerPrice, WAD);
uint256 sellerAssets = offer.buy ? units.mulDivDown(sellerPrice, WAD) : units.mulDivUp(sellerPrice, WAD);

uint256 newConsumed;
if (offer.maxAssets > 0) {
    newConsumed = consumed[offer.maker][offer.group] += offer.buy ? buyerAssets : sellerAssets;
    require(newConsumed <= offer.maxAssets, ConsumedAssets());
} else {
    newConsumed = consumed[offer.maker][offer.group] += units;
    require(newConsumed <= offer.maxUnits, ConsumedUnits());
}
```

当我们获得价格后，我们就可以计算 `uints` 对应的 assets 数量，此处涉及到舍入问题，在 midnight 内，舍入的预期是有利于 maker 而不利于 taker。以 `offer.buy = true` 为例，`buyerAssets` 和 `sellerAssets` 都是使用的向下取整的 `mulDivDown` 计算。这意味挂单的 maker 是少支付资金的，而吃单的 taker 也少收取了资金，但是作为购买者的 maker 收到的 uints 的数量没有改变。在 `offer.buy = sell` 的 maker 卖出情况也类似，即作为卖方的 maker 相同数量的 uints 多卖出了资金，而 taker 也为此多支付了资金。获得 `buyerAssets` 和 `sellerAssets` 后，我们可以调整 `consumed` 内的记录的 `offer.group` 避免超过最高限额。

获得上述数据后，我们进一步通过 uints 计算 credit 或者 debt 的调整问题。对于 buyer 而言，uints 首先会抵消 buyer 的 debt 进一步会增加 buyer 的 credit，对于 seller 而言，uints 首先会减少 credit 进一步增加 debts。在进行具体的 position 内的 credit 或 debt 调整前，我们首先需要使用 `_updatePosition` 更新头寸。`_updatePosition` 的完整函数已经在上文出现过了，该函数的作用是:

1. 处理 continous fee 问题，特别是更新 pending fee
2. 处理 loss factor 带来的坏账问题，将坏账从 position 内的 credit 内扣除，并将 `lastLossFactor` 和 `lastAccrual` 更新到头寸内

基于 `_updatePosition` 的作用，此处我们需要在如下两种情况下进行更新:

1. 无论买方还是卖方，只要 position 内存在 credit，我们都需要调用 `_updatePosition` 处理 continous fee
2. 特殊的对于买方买入大量 credit 使得自己的 debt 清零而 credit 产生的情况，我们也需要先进行 `_updatePosition` 再进行 position 内的 credit 和 debt 更新，这样是为了避免历史坏账影响到新的 credit

```solidity
(address buyer, address seller) = offer.buy ? (offer.maker, taker) : (taker, offer.maker);
Position storage buyerPos = position[id][buyer];
Position storage sellerPos = position[id][seller];

if (hasCredit(id, buyer) || units > buyerPos.debt) _updatePosition(offer.market, id, buyer);
if (hasCredit(id, seller)) _updatePosition(offer.market, id, seller);

uint256 buyerCreditIncrease = UtilsLib.zeroFloorSub(units, buyerPos.debt);
uint256 sellerCreditDecrease = UtilsLib.min(units, sellerPos.credit);
uint256 sellerDebtIncrease = units - sellerCreditDecrease;
uint128 buyerPendingFeeIncrease =
    UtilsLib.toUint128(buyerCreditIncrease.mulDivDown(_marketState.continuousFee * timeToMaturity, WAD));
uint128 sellerPendingFeeDecrease = sellerPos.credit > 0
    ? UtilsLib.toUint128(sellerPos.pendingFee.mulDivUp(sellerCreditDecrease, sellerPos.credit))
    : 0;
```

完成头寸更新后，我们就可以计算一下 credit 和 debt 的增加和减少情况。此处我们也计算 pending fee 的调整。在上文内，我们提到为了实现 pending fee 与历史的费率同步，我们使用了一次性记录 credit 从 position 调整到最终到期日到所有费用。此处假如 `buyerCreditIncrease > 0`，那就意味着 buyer 获得了 `buyerCreditIncrease` 数量的 credit，我们要将这部分 credit 从当前到到期日的 pending fee 计算出来。而对于 seller 而言，假如 seller 本身存在 credit(即 `sellerPos.credit > 0` 的情况)，此时需要减少 `sellerCreditDecrease` 部分的 pending fee。

完成上述计算后，我们需要验证一些条件:

1. `CannotIncreaseDebtPostMaturity` 当前市场是否到期，假如市场到期了，我们则不允许 seller 的 debt 增加，换言之，市场到期后，就不能增加债务
2. `MakerCreditOrDebtIncreased` 处理 offer 内的 `reduceOnly` 参数，假如存在 `reduceOnly`，那么就不允许 credit 和 debt 的反转，即对于 buyer 而言，只允许 debt 减少，不允许 credit 增加；对于 seller 而言，只允许 credit 减少，不允许 debt 增加
3. `BuyerGatedFromIncreasingCredit` 处理 buyer 增加 credit 的情况，调用 `canIncreaseCredit` 判断 buyer 是否允许增加 credit。我们在上文聊到 midnight 增加了更加动态的访问控制，对于某些市场，可能不允许随意存入资产，比如某些市场可能只允许 KYC 的用户存入资产，所以此处在 `buyerCreditIncrease > 0` 的情况下进行 `IEnterGate(offer.market.enterGate).canIncreaseCredit(buyer)` 检查
4. `SellerGatedFromIncreasingDebt` 类似刚刚的 `BuyerGatedFromIncreasingCredit` 用于处理 debt 增加

```solidity
require(block.timestamp <= offer.market.maturity || sellerDebtIncrease == 0, CannotIncreaseDebtPostMaturity());
require(
    !offer.reduceOnly || (offer.buy ? buyerCreditIncrease == 0 : sellerDebtIncrease == 0),
    MakerCreditOrDebtIncreased()
);

require(
    offer.market.enterGate == address(0) || buyerCreditIncrease == 0
        || IEnterGate(offer.market.enterGate).canIncreaseCredit(buyer),
    BuyerGatedFromIncreasingCredit()
);
require(
    offer.market.enterGate == address(0) || sellerDebtIncrease == 0
        || IEnterGate(offer.market.enterGate).canIncreaseDebt(seller),
    SellerGatedFromIncreasingDebt()
);

buyerPos.debt -= UtilsLib.toUint128(units - buyerCreditIncrease);
buyerPos.pendingFee += buyerPendingFeeIncrease;
buyerPos.credit += UtilsLib.toUint128(buyerCreditIncrease);

sellerPos.pendingFee -= sellerPendingFeeDecrease;
sellerPos.credit -= UtilsLib.toUint128(sellerCreditDecrease);
sellerPos.debt += UtilsLib.toUint128(sellerDebtIncrease);

_marketState.totalUnits =
    UtilsLib.toUint128(_marketState.totalUnits + buyerCreditIncrease - sellerCreditDecrease);
claimableSettlementFee[offer.market.loanToken] += buyerAssets - sellerAssets;
```

完成上述对交易双方的 credit 和 debt 情况的校验后，我们就可以直接更新到状态内部。此处需要更新 buyer 和 seller 的 position 外，我们还需要更新 `_marketState` 内的信息，此处的核心需要更新 `totalUnits`。在上文，我们已经介绍过 `totalUnits` 其实代表 `totalUnits = sum(position debt) + withdrawable`。所以此处我们的预期是计算出 `debt` 的变化，最简单的方法是计算 `sellerDebtIncrease - buyerDebtDecrease`，即卖方 debt 增加与买方 debt 减少做差，刚好获得总体的债务变化。但是此处我们其实没有在 `take` 函数计算 `buyerDebtDecrease` 变量，所以我们此处可以换一种计算方法，我们知道:

```
buyerDebtDecrease + buyerCreditIncrease = take uints
sellerCreditDecrease + sellerDebtIncrease = take uints
```

所以存在 `buyerCreditIncrease - sellerCreditDecrease = sellerDebtIncrease - buyerDebtDecrease` 的关系，此处我们就可以使用 `buyerCreditIncrease - sellerCreditDecrease` 替代原有的计算。经过上述介绍，读者应该可以理解 midnight 内的如下图片:

![Minight Uints](https://img.gopic.xyz/midnightPositionUints.png)

左上角代表开仓，`uints` 卖方需要获得 loan token，而买方则是希望将存入的 loan token 借出去，该操作会直接创造 u 单位的净债务，增加 `totalUnits`。而右下角则代表平仓，`uints` 买方希望减少自己的 debt 而卖方希望减少自己的 credit，此时买方归还债务而卖方获得利息，这会减少 `totalUnits`。剩下两种情况则代表债务转移和债权转移，只是将 credit 或者 debt 在不同用户之间移动。当然，上述图像其实默认了 `reduceOnly`，假如不使用 `reduceOnly`，会出现上文介绍的反转情况，比如债务人还款时还了更多的资金会使得债务人持有 credit。

接下来，我们会在 `take` 函数内处理 loan token 的转移，我们第一步获得 `buyerCallback` 和 `sellerCallback`。在上文中，我们已经介绍过 `buyerCallback` 被设置后会作为 loan token 的 payer。而 `sellerCallback` 主要用于调用 `onSell` 进行通知。`payer` 可以依靠如下方法确定:

1. 存在 `buyerCallback`，那么 `payer` 就会等于 `buyerCallback`
2. 不存在 `buyerCallback`，假如 `offer.buy` 成立，即当前吃单者是 seller，那么我们就使用 `offer.maker` 作为 payer，否则就说明吃单者是 buyer，则直接使用吃单者 `msg.sender` 的资金

对于 `receiver` 而言则比较简单，如果 `offer.buy` 则直接使用 `take` 函数的入参 `receiverIfTakerIsSeller`，这是因为 taker 是吃单者，所以可以直接使用 `take` 内的参数，否则使用 `offer` 内的 `offer.receiverIfMakerIsSeller`

完成上述操作中，我们首先进行 `buyerCallback` 的 `onBuy` 调用，然后调用 `safeTransferFrom` 将 `loanToken` 从 `payer` 转移给 `receiver`。我们注意到此处进行了两次转账，第一次向 `address(this)` 的转账其实转入的是 settlement fee，而第二次转账则是真正的 buyer 和 seller 之间的结算。

```solidity
address buyerCallback = offer.buy ? offer.callback : takerCallback;
address sellerCallback = offer.buy ? takerCallback : offer.callback;
address payer = buyerCallback != address(0) ? buyerCallback : (offer.buy ? buyer : msg.sender);
address receiver = offer.buy ? receiverIfTakerIsSeller : offer.receiverIfMakerIsSeller;

bool wasLocked = UtilsLib.tExchange(LIQUIDATION_LOCK_SLOT, id, seller, true);
if (buyerCallback != address(0)) {
    bytes memory buyerCallbackData = offer.buy ? offer.callbackData : takerCallbackData;
    require(
        IBuyCallback(buyerCallback)
            .onBuy(id, offer.market, buyerAssets, units, buyerPendingFeeIncrease, buyer, buyerCallbackData)
        == CALLBACK_SUCCESS,
        WrongBuyCallbackReturnValue()
    );
}

SafeTransferLib.safeTransferFrom(offer.market.loanToken, payer, address(this), buyerAssets - sellerAssets);
SafeTransferLib.safeTransferFrom(offer.market.loanToken, payer, receiver, sellerAssets);

if (sellerCallback != address(0)) {
    bytes memory sellerCallbackData = offer.buy ? takerCallbackData : offer.callbackData;
    require(
        ISellCallback(sellerCallback)
            .onSell(
                id,
                offer.market,
                sellerAssets,
                units,
                sellerPendingFeeDecrease,
                seller,
                receiver,
                sellerCallbackData
            ) == CALLBACK_SUCCESS,
        WrongSellCallbackReturnValue()
    );
}
if (!wasLocked) UtilsLib.tExchange(LIQUIDATION_LOCK_SLOT, id, seller, false);
```

上述代码中出现了 `bool wasLocked = UtilsLib.tExchange(LIQUIDATION_LOCK_SLOT, id, seller, true);` 和 `if (!wasLocked) UtilsLib.tExchange(LIQUIDATION_LOCK_SLOT, id, seller, false);`，这两段段代码构成了一个精细化的重入锁，避免了 `onBuy` / `onSell` 以及 transfer 中可能的重入。具体的工作原理与此处的 `UtilsLib.tExchange` 的实现有关，该函数实现如下:

```solidity
/// @dev Sets a boolean in transient storage keyed by a (bytes32, address) pair.
/// @dev Returns the previous value at the written slot.
function tExchange(uint256 baseSlot, bytes32 key1, address key2, bool value) internal returns (bool previous) {
    uint256 slot = uint256(keccak256(abi.encode(key1, key2, baseSlot)));
    assembly ("memory-safe") {
        previous := tload(slot)
        tstore(slot, value)
    }
}
```

我们可以看到 `tExchange` 不单单是写入 `value`，同时也会返回该 slot 上一次的数值。对于正常的不重入的情况来说，`wasLocked` 在 `bool wasLocked = UtilsLib.tExchange(LIQUIDATION_LOCK_SLOT, id, seller, true);` 内设置为 `wasLocked = false` 而实际 transit storage 内记录的数值为 `true`。假如用户进行重入，那么读取到的 `wasLocked = true`，这会导致后续检查失败。

此处为什么需要这个部分重入锁，因为要避免借贷协议中最有趣的一种攻击——自清算(特别是原子自清算)。所谓的自清算是指用户自己创建头寸后，然后手动使得头寸不健康，然后自己清算自己，在大部分协议中，都存在清算激励以及 midnight 还存在社会化损失，所以自清算等同于人工制造穿仓，然后获得穿仓损失的钱。

在 midnight 内的 `take` 函数内，我们还没有看到健康度检查，因为健康度检查是在最后完成的，midnight 允许借款人在 `onBuy` 或者 `onSell` callback 内原子性补充担保品，而不需要在挂单时一直占用担保品。所以在这个情况下，用户的头寸本身就是不健康的，即本身就存在大量债务但是可能没有担保品，这是自清算的最佳场景。

为了避免自清算问题，midnight 引入了 `wasLocked` 进行重入锁定，避免自清算发生。最后，我们可以看到 `take` 函数内一段代码:

```solidity
require(liquidationLocked(id, seller) || isHealthy(offer.market, id, seller), SellerIsLiquidatable());

return (buyerAssets, sellerAssets);
```

其中 `liquidationLocked(id, seller)` 就是读取 slot 判断用户是否重入，假如重入，那么这里会抛出 `SellerIsLiquidatable` 错误，而 `isHealthy` 则适用于检查健康度的。这是一个并不复杂的函数:

```solidity
function isHealthy(Market memory market, bytes32 id, address borrower) public view returns (bool) {
    Position storage _position = position[id][borrower];
    uint256 debt = _position.debt;
    uint256 maxDebt;
    if (debt > 0) {
        uint128 _collateralBitmap = _position.collateralBitmap;
        while (_collateralBitmap != 0) {
            uint256 i = UtilsLib.msb(_collateralBitmap);
            CollateralParams memory collateralParam = market.collateralParams[i];
            uint256 price = IOracle(collateralParam.oracle).price();
            maxDebt += _position.collateral[i].mulDivDown(price, ORACLE_PRICE_SCALE)
                .mulDivDown(collateralParam.lltv, WAD);
            _collateralBitmap = _collateralBitmap.clearBit(i);
        }
    }
    return maxDebt >= debt;
}
```

本质上该函数会遍历用户的 bitmap 找到所有被设置为 true 的担保品，然后去检查该担保品的价格，此处使用了 `msb` 算法，具体使用了 EVM 较新引入的 `clz` 操作码。`clz` 的作用是进行前缀零计数，本质上与 `msb` 语义是一致的，具体可以参考 [现代 DeFi: Uniswap V4 数学库分析](https://blog.wssh.dev/posts/uniswap-math/#mostsignificantbit)(在这篇文章编写时，`clz` 还不是 EVM opcode，所以介绍了一些复杂方法，但是 clz 出现后，我们只需要进行 `res := sub(255, clz(bitmap))` 就可以获得 `msb`)。

为了使用 `msb` 完成逐一遍历，每访问一个 bit 后会执行 `_collateralBitmap = _collateralBitmap.clearBit(i);` 操作来避免再次访问，注意此处的 `_collateralBitmap` 并不是 storage 而是位于栈上，所以可以随意修改。

## 清算函数

在任何借贷协议内，清算往往是最复杂且最容易出现问题的部分，在本节中，我们将分析 midnight 内的 `liquidate` 函数。我们还是首先阅读 `liquidate` 函数的入参情况:

```solidity
function liquidate(
    Market calldata market,
    uint256 collateralIndex,
    uint256 seizedAssets,
    uint256 repaidUnits,
    address borrower,
    bool postMaturityMode,
    address receiver,
    address callback,
    bytes calldata data
) external returns (uint256, uint256) {
```

上述参数的含义是:

1. `market` 用于传入需要清算的市场的信息
2. `collateralIndex` 用于指定清算的担保品的 index，由于 midnight 支持多种担保品，所以需要指定清算某种担保品
3. `seizedAssets` / `repaidUnits` 类似 `take` 函数，两者用于指定清算的债务的 asset 或 uints 数量，只能指定其中一个
4. `borrower` 用于指定被清算者的地址
5. `postMaturityMode` 用于启动到期后清算模式。在 midnight 内存在两种不同的清算模式，一种是常规的在到期前由于担保品价格变动而导致的清算，另一种则是到期后清算，到期后清算允许直接清算健康头寸，但是  LIF 会根据清算距离到期时间的时间差决定
6. `receiver` 是清算后担保品的接收地址
7. `callback` 和 `take` 函数内的类似，假如设置了 callback，那么债务清偿的资金会在 callback 地址内获取，而 `liquidate` 函数的 `data` 参数则是用于调用 `callback`

在 `liquidate` 函数的第一步就是进行参数检查，这部分检查较为简单，只是判断一下 `repaidUnits` 和 `seizedAssets` 是不是只有一个非零，以及检查待清算的 `borrower` 确实存在债务，以及假如存在特殊的 `liquidatorGate` 权限控制，那么检查一下 `msg.sender` 是否有权限。

```solidity
bytes32 id = touchMarket(market);
MarketState storage _marketState = marketState[id];
Position storage _position = position[id][borrower];
require(UtilsLib.atMostOneNonZero(repaidUnits, seizedAssets), InconsistentInput());
require(_position.debt > 0, NotBorrower()); // to avoid no-op liquidations of non borrower positions.
require(
    market.liquidatorGate == address(0) || ILiquidatorGate(market.liquidatorGate).canLiquidate(msg.sender),
    LiquidatorGatedFromLiquidating()
);
```

完成上述检查后，下一步我们的核心是判断当前头寸是可以被清算的。此处我们需要计算 `maxDebt`和 `badDebt`。所以 `maxDebt` 是指当前担保品可以偿付的最大债务，我们可以简单的使用担保品价值(即担保品数量与担保品价格的乘积)与 LLTV 相乘获得。而对于 `badDebt` 而言，由于存在  LIF 清算激励，所以计算方法稍有不同，我们需要计算担保品在包含清算激励情况下可以覆盖的债务额度。在此处我们会在 `badDebt` 初始化为 `originalDebt`，然后不断减去每一种担保品在激励情况下可以覆盖的债务，假如 `badDebt` 被减至 `0`，那么说明不存在坏账，否则意味着存在坏账。

```solidity
uint256 maxDebt;
uint256 liquidatedCollatPrice;
uint256 originalDebt = _position.debt;
uint256 badDebt = originalDebt;
uint128 _collateralBitmap = _position.collateralBitmap;
while (_collateralBitmap != 0) {
    uint256 i = UtilsLib.msb(_collateralBitmap);
    CollateralParams memory _collateralParam = market.collateralParams[i];
    uint256 price = IOracle(_collateralParam.oracle).price();
    if (i == collateralIndex) liquidatedCollatPrice = price;
    uint256 _collateral = _position.collateral[i];
    maxDebt += _collateral.mulDivDown(price, ORACLE_PRICE_SCALE).mulDivDown(_collateralParam.lltv, WAD);
    badDebt = badDebt.zeroFloorSub(
        _collateral.mulDivUp(price, ORACLE_PRICE_SCALE).mulDivUp(WAD, _collateralParam.maxLif)
    );
    _collateralBitmap = _collateralBitmap.clearBit(i);
}
```

此处可能有读者问，那么是否存在 `lltv` 和 `maxLif`  设置的不一致的情况，即按照 `lltv` 计算用户担保品是可以覆盖债务的，但是从 `maxLif` 视角看，反而存在坏账？为了避免这个问题，我们一直没有介绍 `maxLif` 的由来，实际上 `maxLif` 是通过 LLTV 计算出来的，在上文介绍 Fee 机制中的 `touchMarket` 函数时，我们就可以看到这样一段代码：

```solidity
require(
    market.collateralParams[i].maxLif == maxLif(lltv, LIQUIDATION_CURSOR_LOW)
        || market.collateralParams[i].maxLif == maxLif(lltv, LIQUIDATION_CURSOR_HIGH),
    InvalidMaxLif()
);
```

其中的 `LIQUIDATION_CURSOR_LOW` 和 `LIQUIDATION_CURSOR_HIGH` 是常量，分别是 `0.25e18` 和 `0.5e18` 而 `maxLif` 函数的定义如下:

```solidity
function maxLif(uint256 lltv, uint256 cursor) pure returns (uint256) {
    return UtilsLib.mulDivDown(WAD, WAD, WAD - UtilsLib.mulDivDown(cursor, WAD - lltv, WAD));
}
```

实际对应 `maxLif = 1 / (1 − cursor × (1 − lltv))` 数学表达式。为什么这么计算 Lif? 首先，我们知道 `1 - lltv` 的经济学含义是担保品与债务的缓冲空间，当债务价值大于担保品价值与 lltv 乘积时，此时担保品仍可以覆盖债务，但是我们认为此时应该清算用户的债务。而 `maxLif` 则利用 `cursor` 变量从这部分缓冲空间(即 `1 - lltv`) 内划拨了一部分给清算激励，具体划拨的比例是 `1 − cursor × (1 − lltv)`，我们可以通过担保品除以 `1 − cursor × (1 − lltv)` 获得含有激励的应该给清算者的担保品价值，但是除法总是没有乘法便捷，所以此处我们使用 `1 / (1 − cursor × (1 − lltv))` 获得一个可以直接被乘法计算的 `maxLif`。

完成上述计算后，我们在 `liquidate` 函数内需要确定当前的 `borrower` 是否可以被清算，我们使用如下方法判断:

```solidity
require(
    !liquidationLocked(id, borrower)
        && (postMaturityMode ? block.timestamp > market.maturity : originalDebt > maxDebt),
    NotLiquidatable()
);
```

在 `take` 函数内，我们介绍过 `take` 函数存在部分重入锁避免订单成交过程中被清算，而此处也读取了这个锁，假如处于锁定阶段，那么 `borrower` 无法被清算。接下来，我们就会使用 `postMaturityMode` 结合目前是否处于到期日后进行判断，有两种情况:

1. `postMaturityMode = true` 且 `block.timestamp > market.maturity` 此时清算者其实可以清算任何还存在的头寸
2. `postMaturityMode = false` 且 `block.timestamp > market.maturity` ，这是一种特殊情况，虽然目前已经到期，但是清算者仍可以使用 `postMaturityMode = false` 进行清算，但此时清算者只能清算 `originalDebt > maxDebt` 的不健康头寸。为什么存在这种情况？是因为清算到期健康头寸的 Lif 是一个随着距离到期时间越远而逐渐变大的数值，而清算不健康头寸可以直接获得 maxLif 激励而不需要考虑时间，所以为了利益最大化，存在清算者在到期日后清算不健康头寸时使用 `postMaturityMode = false` 的情况
3. `postMaturityMode = false` 且 `block.timestamp <= market.maturity` 的情况，此时就是最常见的普通清算，只能清算不健康头寸

假如我们确定清算者可以清算后，我们将处理坏账问题，假如我们计算出的 `badDebt > 0` 就意味着我们需要社会化额外的穿仓损失。此处，我们回直接在 `_position.debt` 内减去穿仓的损失，相当于协议使用社会资金填补了穿仓，然后将这部分损失计入到 `lossFactor` 内部

```solidity
if (badDebt > 0) {
    // forge-lint: disable-next-item(unsafe-typecast) as badDebt <= _position.debt
    _position.debt -= uint128(badDebt);
    uint256 _totalUnits = _marketState.totalUnits;
    uint256 _lossFactor = _marketState.lossFactor;
    _marketState.lossFactor = UtilsLib.toUint128(
        type(uint128).max - (type(uint128).max - _lossFactor).mulDivDown(_totalUnits - badDebt, _totalUnits)
    );
    _marketState.totalUnits -= UtilsLib.toUint128(badDebt);
    _marketState.continuousFeeCredit = _lossFactor < type(uint128).max
        ? UtilsLib.toUint128(
            _marketState.continuousFeeCredit
                .mulDivDown(type(uint128).max - _marketState.lossFactor, type(uint128).max - _lossFactor)
        )
        : 0;
}
```

这里的 `lossFactor` 的数学计算并不复杂，读者可以自行理解。而记录 `lossFactor` 后，我们也需要同步进行其他操作，由于 `badDebt` 偿还其实也会减少整体的债务，所以此处需要调整 `totalUnits` 以及调整 `continuousFeeCredit`。由于 `continuousFeeCredit` 可以视为协议方在协议内的存款，所以这部分也需要按照 loss factor 进行调整。

在完成坏账清理后，此时头寸的债务就可以交给清算人进行清算。清算的第一个环节是计算 lif。在上文中，我们提到对于清算不健康的头寸，lif 会直接被设置为 `_maxLif` ，但是假如清算 `postMaturityMode` 的头寸，即清算到期日后的头寸，那么我们会使用 `UtilsLib.min(_maxLif, WAD + (_maxLif - WAD) * (block.timestamp - market.maturity) / TIME_TO_MAX_LIF)` 进行计算，这个代码等价于 $\min\left(L\_{\max},\;1+(L\_{\max}-1)\frac{\Delta t}{T}\right)$，其中 $\Delta t$ 代表目前距离到期日过去的时间，而 $T$ 则是 `uint256 constant TIME_TO_MAX_LIF = 15 minutes;`。上述 LIF 计算的含义是从到期日开始到到期后 15 分钟，LIF 会线性增加直到 `maxLif`

```solidity
if (repaidUnits > 0 || seizedAssets > 0) {
    uint256 _maxLif = market.collateralParams[collateralIndex].maxLif;
    uint256 lif = postMaturityMode
        ? UtilsLib.min(_maxLif, WAD + (_maxLif - WAD) * (block.timestamp - market.maturity) / TIME_TO_MAX_LIF)
        : _maxLif;

    if (seizedAssets > 0) {
        repaidUnits = seizedAssets.mulDivUp(liquidatedCollatPrice, ORACLE_PRICE_SCALE).mulDivUp(WAD, lif);
    } else {
        seizedAssets = repaidUnits.mulDivDown(lif, WAD).mulDivDown(ORACLE_PRICE_SCALE, liquidatedCollatPrice);
    }
```

当然，此处也会进行一次根据 `seizedAssets` 计算 `repaidUnits` 或者根据 `repaidUnits` 计算 `seizedAssets` 的环节。此处，我们要注意取整问题。`repaidUnits` 向上取整意味着在担保品减少一致的情况下，债务偿还的更多一点；而 `seizedAssets` 向下取整意味着偿还债务一致的情况下，担保品支付更少。这两种取整方向都是有利于被清算者的。

继续阅读代码，我们会涉及到 RCF 部分。在上文中，我们已经介绍过 RCF 的具体工作原理。当然，假如清算后存在小于 `market.rcfThreshold` 的担保品，那么我就允许清算者清算所有的债务。如下这段代码的解析实际上在本文最开始出现过，为了减少篇幅，此处不再进行介绍。

```solidity
if (!postMaturityMode) {
    uint256 lltv = market.collateralParams[collateralIndex].lltv;
    // Note that debt >= maxDebt in this branch.
    // The imprecision in this computation is at most a few hundreds collateral or loan token assets.
    uint256 maxRepaid = lltv < WAD
        ? (_position.debt - maxDebt).mulDivUp(WAD * WAD, WAD * WAD - lif * lltv)
        : type(uint256).max;
    require(
        repaidUnits <= maxRepaid
            || _position.collateral[collateralIndex].mulDivDown(liquidatedCollatPrice, ORACLE_PRICE_SCALE)
                .mulDivDown(WAD, lif).zeroFloorSub(maxRepaid) < market.rcfThreshold,
        RecoveryCloseFactorConditionsViolated()
    );
}
```

完成上述计算后，我们接下来需要将计算结果回填到合约状态中，这段代码较为简单，读者可以自行阅读。

```solidity
uint128 newCollateral = _position.collateral[collateralIndex] - UtilsLib.toUint128(seizedAssets);
_position.collateral[collateralIndex] = newCollateral;
if (newCollateral == 0 && seizedAssets > 0) {
    _position.collateralBitmap = _position.collateralBitmap.clearBit(collateralIndex);
}
_marketState.withdrawable += UtilsLib.toUint128(repaidUnits);
_position.debt -= UtilsLib.toUint128(repaidUnits);
```

最后，我们需要清算者支付资金，此处假如清算者给定了 `callback` ，那么我们就会使用 `callback` 作为 `payer` 否则使用 `msg.sender` 作为 payer。考虑到清算者可以会卖出担保品来偿还贷款，所以此处我们可以看到 Midnight 先将担保品发送给了清算者，然后在存在 `callback` 的情况下，调用了 `onLiquidate` callback，最后在要求清算者支付债务。

```solidity
address payer = callback != address(0) ? callback : msg.sender;

SafeTransferLib.safeTransfer(market.collateralParams[collateralIndex].token, receiver, seizedAssets);

if (callback != address(0)) {
    require(
        ILiquidateCallback(callback)
            .onLiquidate(
                msg.sender,
                id,
                market,
                collateralIndex,
                seizedAssets,
                repaidUnits,
                borrower,
                receiver,
                data,
                badDebt
            ) == CALLBACK_SUCCESS,
        WrongLiquidateCallbackReturnValue()
    );
}

SafeTransferLib.safeTransferFrom(market.loanToken, payer, address(this), repaidUnits);
```

## 总结

至此，我们就完成了 midnight 内所有的代码阅读，midnight 没有传统意义上的存款和贷款操作，这些操作都被统一到了 `take` 函数内部，而且 midnight 也是链上订单簿设计的典范，我们可以看到 midnight 团队设计链下撮合、链上清算订单簿的权衡和取舍，以及设计 tick 方面的思考。
