---
title: "深入解析AAVE智能合约:存款"
date: 2022-12-10T23:47:33Z
tags: [aave,defi]
math: true
---

## 概述

我们在上一篇文章[AAVE交互指南](./aave-interactive)中主要介绍了`aave`前端、利率计算等内容，本篇文章
将在交互指南基础上介绍`aave-v3`的合约源代码的相关情况。

与之前所写的[深入解析Safe多签钱包智能合约](./deep-in-safe-part-1)系列文章不同，本文主要以我们在[AAVE交互指南](./aave-interactive)中进行的合约操作为主线进行分析介绍，较为实战化。

相比于其他项目，`AAVE`提供了一个较为完整的[文档](https://docs.aave.com/developers/getting-started/readme)。在文档内基本涵盖了所有函数的签名及其作用，读者也可作为阅读源代码的重要参考。

`AAVE`的总体架构如下:

![AAVE Frame](https://img.gejiba.com/images/fea838adc878cd119c68d4030311dfa6.png)

> 本文使用存款描述用户向流动性池内注入资产的行为，或称`supply`或`deposit`，当然在 V3 版本中，`deposit`已被遗弃。当然，有很多人认为此名词应翻译为**质押**，由于作者的写作习惯，后文统称为**存款**

## 代码准备

我们在此处仍使用`Foundry`作为开发和测试框架，使用以下命令初始化仓库:
```bash
forge init aave-v3 
```

前往[AAVE Releases](https://github.com/aave/aave-v3-core/releases)页面下载最新的源代码，并解压。将解压后的`contracts`中的文件转移到上文初始化的`aave-v3`仓库中的`src`文件夹下，最终形成如下目录结构:
```
.
├── foundry.toml
├── lib
│   └── forge-std
├── script
│   └── Counter.s.sol
├── src
│   ├── Counter.sol
│   ├── dependencies
│   ├── deployments
│   ├── flashloan
│   ├── interfaces
│   ├── misc
│   ├── mocks
│   └── protocol
└── test
    └── Counter.t.sol
```


## 整体逻辑

在介绍具体的合约代码前，我们首先应当明确存款行为的具体逻辑。作为金融系统，其逻辑具有相当的数学性，我们会结合具体的数学公式介绍存款的具体逻辑。与[上一篇文章](./aave-interactive)相比，本节给出的逻辑会更加详细且主要服务于后文代码解释，建议以本节为纲要以避免迷失在具体实现中。

本文主要参考了[AAVE V2 Whitepaper](https://github.com/aave/protocol-v2/blob/master/aave-v2-whitepaper.pdf)，此文档给出了具体的逻辑阐述。

> AAVE V3 的白皮书是建立在 V2 白皮书基础上的，所以 V3 白皮书仅介绍了与 V2 不同的部分，不足够详细。

我们引入以下参数:

- ${LR}_t$ 当前的存款利率(`currentLiquidityRate`)，计算方法为 
    ${LR}_t=\bar{R_t}{U_t}$(此公式在[上一篇文章](./aave-interactive#%E8%B4%A8%E6%8A%BC%E5%88%A9%E7%8E%87%E8%AE%A1%E7%AE%97)内有详细解释，读者可作为参考)

    参数含义如下:
    - $\bar{R_t}$ 为浮动存款和固定存款利率的加权平均数
    - $U_t$ 为利用率

- ${LI}_t$ 贴现因子(liquidityIndex)，计算方法为
    ${LI}\_t=({LR}\_t{\Delta}\_{year} + 1){LI}\_{t-1}$
    
    如果有读者阅读过原文，可能发现此遍历英文名为`cumulated liquidity index`，但本质是贴现因子。我们会在后文讨论此参数，当然读者也可以通过各种方式了解此概念。

> 有读者可能发现 ${LR}\_t{\Delta}\_{year} + 1$ 是以线性利率的形式进行的计算，与我们上一篇文章所说明的存款利率复利计算是不符的，但为什么上一篇文章内使用复利计算的结果与和约相同？ 原因在于此处的单利计算会在用户每一次进行操作时更新，高频率的单利计算与复利计算会渐趋一致

> 在AAVE的设计中，贴现因子的使用具有普遍性，如存款、贷款等情况下均使用了**贴现因子**概念，由于此文主要分析存款，所以若无特殊说明，后文的贴现因子均指存款的贴现因子

假设用户在 $t_0$ 时刻存入资产`Token`的数量为 $q$ ，我们在智能合约中记录的用户存入数值为 

$$\frac{q}{LI_{t0}}$$

在 $t1$ 时刻，用户取出资产，获得的资产数量为 

$$\frac{q}{LI_{t0}} \times LI_{t1}$$

此部分使用了金融学内简单的贴现概念，我们将所有存入资产均贴现到 $t_0$ 时期，在用户提款时将其折算回 $t_1$ 。此处使用的 $liquidityIndex$ 事实上就是 `贴现因子`。如果读者无法理解此部分，可简单选择任一金融学课本阅读此部分内容。

假设用户的存款数量用 ${ScB}_t(x)$ 表示，则用户存入 $m$ 单位质押品后，存款数量为:

$${ScB}\_t(x) = {ScB}\_{t-1}(x) + \frac{m}{{LI}\_t}$$

取出 $m$ 单位质押品后，存款数量为:

$${ScB}\_t(x) = {ScB}\_{t-1}(x) - \frac{m}{{LI}\_t}$$

总结来说，存款的核心步骤如下:

1. 计算当前的 ${LR}\_t{\Delta}\_{year} + 1$
1. 更新 ${LI}_t$
1. 计算 ${ScB}_t(x)$

当然，上述核心步骤会在合约编程中被高度复杂化。总体来说，复杂性主要来源于以下两点:

1. 提高用户体验
1. 更新数据

## 入口函数

在此处，我们在上一篇文章内进行存款交易的[EthTx 地址](https://ethtx.info/goerli/0xf7bc3325b84af8b169167e9c26ab262ddfa34a15c2155f524a155c8ec3dffacc/)。如下图:

![Supplt Eth Tx](https://img.gejiba.com/images/62feca221a43aede018acde6ae38217b.png)

非常明显，在存款交易时，我们使用了`supply`函数，并将`USDC`存入获得`aEthUSDC`。

查阅源代码，我们可以在`src/protocol/pool/Pool.sol`找到此函数，代码如下:
```solidity
function supply(
    address asset,
    uint256 amount,
    address onBehalfOf,
    uint16 referralCode
) public virtual override {
    SupplyLogic.executeSupply(
        _reserves,
        _reservesList,
        _usersConfig[onBehalfOf],
        DataTypes.ExecuteSupplyParams({
            asset: asset,
            amount: amount,
            onBehalfOf: onBehalfOf,
            referralCode: referralCode
        })
    );
}
```

此函数的各参数含义如下:

1. `asset` 存入资产的合约地址
1. `amount` 存入资产的数量
1. `onBehalfOf` 接受`aToken`代币的地址
1. `referralCode` 第三方集成商的标识。此参数主要用于第三方供应商检索集成交易

> 只有持有`aToken`的用户可以获得存款回报，所以一般情况下`onBehalfOf`仅为用户自己的地址，但用户也可以设置为其他人的地址以方便进行利益转移。

虽然此函数看似简单，但其内部调用了一些复杂函数。使用[Solidity Visual Developer](./smart-contract-tool#编辑器配置)中的`ftrace`工具获得如下调用栈:
```
└─ Pool::supply
   ├─ SupplyLogic::executeSupply | [Ext] ❗️  🛑 
   │  ├─ DataTypes.ReserveData::cache | [Int] 🔒   
   │  ├─ DataTypes.ReserveData::updateState | [Int] 🔒  🛑 
   │  ├─ ValidationLogic::validateSupply | [Int] 🔒   
   │  ├─ DataTypes.ReserveData::updateInterestRates | [Int] 🔒  🛑 
   │  ├─ ValidationLogic::validateUseAsCollateral | [Int] 🔒   
   │  │  ├─ DataTypes.UserConfigurationMap::isUsingAsCollateralAny | [Int] 🔒   
   │  │  ├─ DataTypes.UserConfigurationMap::getIsolationModeState | [Int] 🔒   
   │  │  └─ DataTypes.ReserveConfigurationMap::getDebtCeiling | [Int] 🔒   
   │  └─ DataTypes.UserConfigurationMap::setUsingAsCollateral | [Int] 🔒  🛑 
   └─ DataTypes::ExecuteSupplyParams
```

在此处，我们简单给出每个函数的作用:

1. `cache` 将`ReserveData`内部的数据进行缓存以降低 gas 消耗
1. `updateState` 更新贴现因子(即`Index`系列变量)等变量和准备金余额
1. `validateSupply` 校验存款限制条件
1. `updateInterestRates` 更新利率
1. `validateUseAsCollateral` 验证和设置抵押品

> 我们在后文中使用了**抵押品**，此名词指可以用于作为贷款担保的存款。在AAVE内，资产是否用作贷款担保由用户自己决定。

## 数据结构

### 基础数据结构

此节内容内有大量的变量被定义，读者可能无法完全理解，建议读者简单读一遍。在后文，我们会更加详细的叙述每一个变量的作用。

在此处，我们发现了一些未被定义的变量`_reserves`、`_reservesList`和`_usersConfig`，这些变量来自`src/protocol/pool/PoolStorage.sol`合约定义，代码如下:
```solidity
// Map of reserves and their data (underlyingAssetOfReserve => reserveData)
mapping(address => DataTypes.ReserveData) internal _reserves;

// Map of users address and their configuration data (userAddress => userConfiguration)
mapping(address => DataTypes.UserConfigurationMap) internal _usersConfig;

// List of reserves as a map (reserveId => reserve).
// It is structured as a mapping for gas savings reasons, using the reserve id as index
mapping(uint256 => address) internal _reservesList;
```

通过注释，我们可以获得各参数的定义:

1. `_reserves` 对于资产地址和该资产的存款数据`ReserveData`的对应关系
1. `_usersConfig` 用户地址与其设置之间的对应关系
1. `_reservesList` 资产`id`及其地址之间的对应关系，设置此映射目的是节省`gas`，其具体节省原理会在后文介绍

上述特殊的`DataType`均定义在`src/protocol/libraries/types/DataTypes.sol`中，为方便读者后文阅读，我们也将给出此处使用的各个特殊的数据结构的代码定义。

首先，我们给出`ReserveData`的代码定义:
```solidity
struct ReserveData {
    //stores the reserve configuration
    ReserveConfigurationMap configuration;
    //the liquidity index. Expressed in ray
    uint128 liquidityIndex;
    //the current supply rate. Expressed in ray
    uint128 currentLiquidityRate;
    //variable borrow index. Expressed in ray
    uint128 variableBorrowIndex;
    //the current variable borrow rate. Expressed in ray
    uint128 currentVariableBorrowRate;
    //the current stable borrow rate. Expressed in ray
    uint128 currentStableBorrowRate;
    //timestamp of last update
    uint40 lastUpdateTimestamp;
    //the id of the reserve. Represents the position in the list of the active reserves
    uint16 id;
    //aToken address
    address aTokenAddress;
    //stableDebtToken address
    address stableDebtTokenAddress;
    //variableDebtToken address
    address variableDebtTokenAddress;
    //address of the interest rate strategy
    address interestRateStrategyAddress;
    //the current treasury balance, scaled
    uint128 accruedToTreasury;
    //the outstanding unbacked aTokens minted through the bridging feature
    uint128 unbacked;
    //the outstanding debt borrowed against this asset in isolation mode
    uint128 isolationModeTotalDebt;
}
```

我们以表格的形式依次给出各参数的含义:

| 变量名 | 具体含义及用途 |
| ------ | ---------- |
| configuration | 包含有大量数据的配置项，会在后文介绍 |
| liquidityIndex | 流动性池自创立到更新时间戳之间的累计利率(贴现因子) |
| currentLiquidityRate | 当前的存款利率 |
| variableBorrowIndex | 浮动借款利率自流动性池建立以来的累计利率(贴现因子)  |
| currentVariableBorrowRate | 当前的浮动利率 |
| currentStableBorrowRate | 当前固定利率 |
| lastUpdateTimestamp | 上次数据更新时间戳 |
| id | 存储资产的`id` |
| aTokenAddress | `aToken`代币地址 |
| stableDebtTokenAddress | 固定利率借款代币地址 |
| variableDebtTokenAddress | 浮动利率借款代币地址 |
| interestRateStrategyAddress | 利率策略合约地址 |
| accruedToTreasury | 当前准备金余额 |
| unbacked | 通过桥接功能铸造的未偿还的无担保代币 |
| isolationModeTotalDebt | 以该资产借入的未偿债务的单独模式 |
 
此处存在一系列后缀为`Index`的变量，这一系列变量都应用于状态更新。当用户与合约进行交互时，每次交互都是促使存款和贷款数据更新，而上述`Index`变量的功能就是用于存款和贷款数据更新。

> 有读者好奇为什么此处使用 `uint128` 而不是 `uint256` 作为数字的基本类型呢? 原因在于 `AAVE` 在表示浮点数时使用一种较为简单的定点浮点数的表示方法。此处的各种利率均使用了`RAY`表示，其具有固定的 27 位小数，使用 `uint128` 足够进行表示且更节省存储空间。
>
> 关于此处数学运算的相关内容，读者可阅读[深入解析AAVE智能合约:计算和利率](./aave-contract-part2)。

`Index`系列变量实现了一个极其特殊的功能，即使用统一参数计算所有用户的质押收益或者贷款利息，此变量系列均属于贴现因子。正如上文所述，在本节内，我们所提及的贴现因子一般指存款的贴现因子。

`UserConfigurationMap`的定义如下:
```solidity
struct UserConfigurationMap {
    uint256 data;
}
```

具体来看，其结构如下图:

![UserConfig](https://img.gejiba.com/images/44730e1a6fb788ad2d9adf3678e89a39.png)

此数据结构定义了用户所存入及贷出的资产。此 256 bit 数据可用于表示 128 种不同资产的存款和贷款情况，其中低位代表是否存在贷款，而高位代表是否存在抵押物。上图展示了用户持有`Asset 0`和`Asset 2`的抵押物并贷出了`Asset 2`资产。

> 这是一种在`AAVE`中常用的数据表示方法，将数据编码为纯粹2进制后通过`uint256`保存。

### 特殊数据结构

在上文介绍`ReserveData`时，我们跳过其定义在最开始的`configuration`变量，其属于`ReserveConfigurationMap`数据类型

此数据类型是由多个参数压缩产生的。事实上，此数据类型底层为`uint256`，但为了减少数据存储的`gas`费用，程序员选择通过规定`uint256`中 256 位中不同位置的含义实现了在`uint256`内保存 18 个参数设置的目的，其对应表如下:

| 位置 | 参数 | 含义 |
| ---- | ---- | ---- |
| 0-15 | LTV | |
| 16-31 | Liquidation threshold |  |
| 32-47 | Liquidation bonus | |
| 48-55 | Decimals | 质押代币(ERC20)精度 |
| 56 | reserve is active | 质押品可以使用 |
| 57 | reserve is frozen | 质押品冻结，不可使用 |
| 58 | borrowing is enabled | 是否可贷出 |
| 59 | stable rate borrowing enabled | 是否可以以固定利率贷出 |
| 60 | asset is paused | 资产是否被暂停 |
| 61 | borrowing in isolation mode is enabled | 资产是否可以在`isolation mode`内使用 |
| 62-63 | 保留 | 保留位以待后期扩展 |
| 64-79 | reserve factor | 储备系数，即借款利息中上缴`AAVE`风险准备金的比例 |
| 80-115 | borrow cap in whole tokens | 代币贷出上限 |
| 116-151 | supply cap in whole tokens | 代币存款上限 |
| 152-167 | liquidation protocol fee | 在清算过程中，`AAVE`收取的费用 |
| 168-175 | eMode category | E-Mode 类别 |
| 176-211 | unbacked mint cap in whole tokens | 无存入直接铸造的代币数量上限(此变量用于跨链) |
| 212-251 | debt ceiling for isolation mode with decimals | 隔离模式中此抵押品的贷出资产上限 |
| 252-255 | unused | 未使用 |

此表格内的部分变量的作用留空是因为我们已经在[上一篇文章](./aave-interactive)内对这些变量的使用进行了讨论。也可以使用下图更加清晰的展示`ReserveConfigurationMap`的基础数据结构:

![ReserveConfigurationMap](https://img.gejiba.com/images/ab995b3accb9ee51453885c9c210e83a.png)

此处以`Liquidation threshold`为大家介绍如何进行数据写入和读取:

为了在`uint256`中指定的位置读取和写入，我们首先需要一个`Mask`，此`Mask`应在 16 - 31 位处置 0 。稍加思考，我们可以得到如下`Mask`：
```solidity
uint256 internal constant LIQUIDATION_THRESHOLD_MASK = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF0000FFFF;
uint256 internal constant LIQUIDATION_THRESHOLD_START_BIT_POSITION = 16;
```
此处我们也定义了`Liquidation threshold`在`uint256`中的起始位置。

> 注意 16 进制中每一个位相当于 2 进制的 4 位

写入函数如下:
```solidity
function setLiquidationThreshold(
    DataTypes.ReserveConfigurationMap memory self,
    uint256 threshold
) internal pure {
    require(
        threshold <= MAX_VALID_LIQUIDATION_THRESHOLD,
        Errors.INVALID_LIQ_THRESHOLD
    );

    self.data =
        (self.data & LIQUIDATION_THRESHOLD_MASK) |
        (threshold << LIQUIDATION_THRESHOLD_START_BIT_POSITION);
}
```
在写入前，我首先确认待写入的`threshold`少于 16 位，即`65535`，避免写入时发生越界写入。

在具体的写入过程中，我们遵循以下流程图:

![Write Bitmap](https://img.wongssh.cf/file/wongsshblog/svg/writebitmap.svg)

相信读者在读完上图后尽可以很好的理解合约源代码。

读取函数如下:
```solidity
function getLiquidationThreshold(
    DataTypes.ReserveConfigurationMap memory self
) internal pure returns (uint256) {
    return
        (self.data & ~LIQUIDATION_THRESHOLD_MASK) >>
        LIQUIDATION_THRESHOLD_START_BIT_POSITION;
}
```
此处我们不再进行绘图而采用推导的方式，仍取上一个例子，我们在`0x5233`(`0b 0101 0010 0011 0011`)中读出`0x23`部分。步骤如下:
1. 对 `Mask` 取反(~)，结果为`0b 0000 1111 1111 0000`
1. 对取反后的`Mask`与原数据取并`&`，结果为`0b 0000 0010 0011 0000`
1. 对取并后的结果进行向右移位，得到结果`0b 0010 0011` (`0x23`)

上述过程即读取流程。

## executeSupply 函数

本节主要介绍具体的存款逻辑，继续查看上文给出的`supply`函数，发现此函数核心调用了`SupplyLogic.executeSupply`函数，此函数功能较为复杂，我们会分成多个部分逐个介绍。

我们再次给出函数调用栈:
```
└─ Pool::supply
   ├─ SupplyLogic::executeSupply | [Ext] ❗️  🛑 
   │  ├─ DataTypes.ReserveData::cache | [Int] 🔒   
   │  ├─ DataTypes.ReserveData::updateState | [Int] 🔒  🛑 
   │  ├─ ValidationLogic::validateSupply | [Int] 🔒   
   │  ├─ DataTypes.ReserveData::updateInterestRates | [Int] 🔒  🛑 
   │  ├─ ValidationLogic::validateUseAsCollateral | [Int] 🔒   
   │  │  ├─ DataTypes.UserConfigurationMap::isUsingAsCollateralAny | [Int] 🔒   
   │  │  ├─ DataTypes.UserConfigurationMap::getIsolationModeState | [Int] 🔒   
   │  │  └─ DataTypes.ReserveConfigurationMap::getDebtCeiling | [Int] 🔒   
   │  └─ DataTypes.UserConfigurationMap::setUsingAsCollateral | [Int] 🔒  🛑 
   └─ DataTypes::ExecuteSupplyParams
```

此入口函数的定义如下:
```solidity
function executeSupply(
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(uint256 => address) storage reservesList,
    DataTypes.UserConfigurationMap storage userConfig,
    DataTypes.ExecuteSupplyParams memory params
) external 
```
需要以下参数输入:

1. `reservesData` 质押品相关数据 `ReserveData`，详细定义参考上文
1. `reservesList` 质押品`ID`与地址的对应关系
1. `DataTypes.UserConfigurationMap` 用户的借贷存储数据
1. `DataTypes.ExecuteSupplyParams` 用户在`supply`函数内输入的参数的打包

> 可能有读者好奇`reservesData`等数据保存在 PoolStorage 合约内，为什么需要在函数调用时传入这些存储内容? 原因在于 `executeSupply` 位于`library`库合约中，其不能访问原合约内的内容，所以此处将其传入。

### 数据缓存
首先，`executeSupply`进行数据缓存操作，此步骤主要是为了将`reservesData`中的部分数据提取出来，主要是`ReserveConfigurationMap`部分。

此部分代码如下:
```solidity
DataTypes.ReserveData storage reserve = reservesData[params.asset];
DataTypes.ReserveCache memory reserveCache = reserve.cache();
```

在`reservesData`映射中获得指定质押物的质押信息，然后我们将此资产的质押信息进行缓存。

缓存部分使用了`ReserveCache`数据结构，此数据结构定义如下:
```solidity
struct ReserveCache {
    uint256 currScaledVariableDebt;
    uint256 nextScaledVariableDebt;
    uint256 currPrincipalStableDebt;
    uint256 currAvgStableBorrowRate;
    uint256 currTotalStableDebt;
    uint256 nextAvgStableBorrowRate;
    uint256 nextTotalStableDebt;
    uint256 currLiquidityIndex;
    uint256 nextLiquidityIndex;
    uint256 currVariableBorrowIndex;
    uint256 nextVariableBorrowIndex;
    uint256 currLiquidityRate;
    uint256 currVariableBorrowRate;
    uint256 reserveFactor;
    ReserveConfigurationMap reserveConfiguration;
    address aTokenAddress;
    address stableDebtTokenAddress;
    address variableDebtTokenAddress;
    uint40 reserveLastUpdateTimestamp;
    uint40 stableDebtLastUpdateTimestamp;
}
```

此数据结构与 `ReserveData` 中的部分数据有直接对应关系，总结如下:

![Cache with ReserveData](https://img.gejiba.com/images/8ec11d055d00539559508dfa0d24d8a9.png)

在后文介绍具体代码时，我们会跳过直接相等的这九种数据字段，接下来我们逐一分析每个字段的来源。

```solidity
reserveCache.reserveFactor = reserveCache
    .reserveConfiguration
    .getReserveFactor();
```
此处的`reserveFactor`被称为 `储备系数` 。此系数规定将协议中的一部分收益分配给`AAVE`违约准备金，用于支持安全模块，所以波动性越低的资产，储备系数越小。

> 类似传统金融中的投资者保障基金提取交易手续费的操作

从代码中，我们可以看出此参数是在`reserveConfiguration`通过`getReserveFactor`提取出来的，对于此函数，我们已经在[上文](#特殊数据结构)内讨论过类似的函数。

```solidity
reserveCache.currScaledVariableDebt = reserveCache
    .nextScaledVariableDebt = IVariableDebtToken(
    reserveCache.variableDebtTokenAddress
).scaledTotalSupply();
```
此处定义的参数含义为:

1. `currScaledVariableDebt` 当前经过贴现的可变利率贷款总额
1. `nextScaledVariableDebt` 含义与上文相同，用于更新状态相关逻辑

`currScaledVariableDebt`和`nextScaledVariableDebt`均被暂时定义为通过`IVariableDebtToken`接口调取`scaledTotalSupply`获得的值。我们有必要指定此值的具体来历，使用`Solidity Visual Developer`通过的`inherbitance`功能查询`IVariableDebtToken`的继承关系，生成下图:

![Inherbit svg](https://img.wongssh.cf/file/wongsshblog/svg/inherbit.svg)

> 注意，此图为倒置图，被继承合约在下，继承合约在上，或称子合约在上

对`VariableDebtToken`的函数绘制相关图像(`graph this`)获得如下图像:

![VariableDebitToken func](https://img.wongssh.cf/file/wongsshblog/svg/variabledebtToken.svg)

通过回溯代码，我们发现`scaledTotalSupply`被定义在`src/protocol/tokenization/base/ScaledBalanceTokenBase.sol`中，具体代码如下:
```solidity
function scaledTotalSupply()
    public
    view
    virtual
    override
    returns (uint256)
{
    return super.totalSupply();
}
```
根据合约继承关系，此处的`super`指`MintableIncentivizedERC20`合约。简单阅读源代码，我们发现`MintableIncentivizedERC20`中不包含此定义，继续回溯`MintableIncentivizedERC20`的父合约`IncentivizedERC20`，发行定义如下:
```solidity
function totalSupply() public view virtual override returns (uint256) {
    return _totalSupply;
}
```
所以此函数仅是返回当前`VariableDebt`代币的总数量。

> 可能读者会问`_totalSupply`没有与贴现利率相除，似乎与我们上文给出的变量含义不符，事实上，贴现放缩步骤是在代币`mint`时实现的，具体请参考`src/protocol/tokenization/VariableDebtToken.sol`合约中的`mint`函数

下述代码给出了固定利率贷款的相关参数定义:
```solidity
(
    reserveCache.currPrincipalStableDebt,
    reserveCache.currTotalStableDebt,
    reserveCache.currAvgStableBorrowRate,
    reserveCache.stableDebtLastUpdateTimestamp
) = IStableDebtToken(reserveCache.stableDebtTokenAddress)
    .getSupplyData();
```
其中各参数含义如下:

1. `currPrincipalStableDebt` 当前已固定利率借入的本金
1. `currTotalStableDebt` 当前以固定利率借出的总资产(即本金与利息之和)
1. `currAvgStableBorrowRate` 平均固定利率
1. `stableDebtLastUpdateTimestamp` 固定利率更新时间

在具体数据获取方面，我们使用了`IStableDebtToken`接口中的`getSupplyData`函数，通过搜索我们可以得到此函数定义在`StableDebtToken.sol`合约内，具体代码如下:
```solidity
function getSupplyData()
    external
    view
    override
    returns (
        uint256,
        uint256,
        uint256,
        uint40
    )
{
    uint256 avgRate = _avgStableRate;
    return (
        super.totalSupply(),
        _calcTotalSupply(avgRate),
        avgRate,
        _totalSupplyTimestamp
    );
}
```

生成相关调用图，如下图:

![getSupplyData func](https://img.wongssh.cf/file/wongsshblog/contract/getSupplyData.svg)

通过调用图，我们发现`getSupplyData`的`super.totalSupply()`来自`IStableDebtToken`，显然这是一个接口其不存在具体实现，我个人认为此处是此插件的绘图错误。自己查找相关继承关系，我们发现实际上`totalSupply`被定义在`IncentivizedERC20`中，其功能与常规`ERC-20`合约对`totalSupply`的定义一致。

> 不同于上文给出的`scaledTotalSupply`变量，此处的`totalSupply`只是代币发行量而没有放缩

`currTotalStableDebt`的计算需要`_calcTotalSupply`函数，定义如下:

```solidity
function _calcTotalSupply(uint256 avgRate) internal view returns (uint256) {
    uint256 principalSupply = super.totalSupply();

    if (principalSupply == 0) {
        return 0;
    }

    uint256 cumulatedInterest = MathUtils.calculateCompoundedInterest(
        avgRate,
        _totalSupplyTimestamp
    );

    return principalSupply.rayMul(cumulatedInterest);
}
```

此处将`totalSupply`的结果与累计利率相乘获得当前以固定利率借出的总资产的价值。

其他变量来与都较为简单，在此处不再赘述。

> 限于篇幅，我们在此处不讨论具体的数学计算方法的合约实现，我们会在未来讨论这一话题。

最终的缓存的数据较为简单，此种类型缓存仅为了方便后期进行数据更新，不再赘述。
```solidity
reserveCache.nextTotalStableDebt = reserveCache.currTotalStableDebt;
reserveCache.nextAvgStableBorrowRate = reserveCache
    .currAvgStableBorrowRate;
```

### 数据更新

再介绍完数据缓存后，我们接下来讨论本处最为重要且核心的函数`updateState`，此函数的逻辑较为复杂，代码如下:
```solidity
function updateState(
    DataTypes.ReserveData storage reserve,
    DataTypes.ReserveCache memory reserveCache
) internal {
    _updateIndexes(reserve, reserveCache);
    _accrueToTreasury(reserve, reserveCache);
}
```
具体调用栈如下:
```solidity
└─ ReserveLogic::updateState
   ├─ ReserveLogic::_updateIndexes | [Int] 🔒  🛑 
   │  ├─ MathUtils::calculateLinearInterest | [Int] 🔒   
   │  ├─ cumulatedLiquidityInterest::rayMul | [Int] 🔒   
   │  └─ MathUtils::calculateCompoundedInterest | [Int] 🔒   
   │     └─ MathUtils::calculateCompoundedInterest | [Int] 🔒   : ..[Repeated Ref]..
   └─ ReserveLogic::_accrueToTreasury | [Int] 🔒  🛑 
      └─ MathUtils::calculateCompoundedInterest | [Int] 🔒   : ..[Repeated Ref]..
```

此函数中调用的两个其他函数的作用是:

- `_updateIndexes` 更新`Index`系列变量
- `_accrueToTreasury` 更新风险准备金

#### _updateIndexes

首先介绍用于更新`Index`(即贴现变量)的`_updateIndexes`函数，此函数代码如下:
```solidity
reserveCache.nextLiquidityIndex = reserveCache.currLiquidityIndex;
reserveCache.nextVariableBorrowIndex = reserveCache
    .currVariableBorrowIndex;
```
首先初始化 `nextLiquidityIndex` 和 `nextVariableBorrowIndex` 变量，这些变量用于计算新的 存款贴现因子 和 浮动借款贴现因子。

> 在上文进行`cache`缓存操作等行为中，我们没有对这两个变量进行初始化

复习一下 贴现因子 的计算公式，如下:

$${LI}\_t=({LR}\_t{\Delta}_{year} + 1){LI}\_{t-1}$$

我们首先使用以下代码:
```solidity
uint256 cumulatedLiquidityInterest = MathUtils
    .calculateLinearInterest(
        reserveCache.currLiquidityRate,
        reserveCache.reserveLastUpdateTimestamp
    );
```
计算 ${LR}\_t{\Delta}\_{year} + 1$ 的数值，具体的实现读者可以自行阅读`calculateLinearInterest`的代码实现。 

通过以下代码实现完整计算:
```solidity
reserveCache.nextLiquidityIndex = cumulatedLiquidityInterest.rayMul(
    reserveCache.currLiquidityIndex
);
```
其中，`rayMul`代表乘法，而`currLiquidityIndex`即 ${LI}_{t-1}$

当我们获得`nextLiquidityIndex`，需要将其从`reserveCache` 缓存内更新到`reserve`中，代码如下:
```solidity
reserve.liquidityIndex = reserveCache
    .nextLiquidityIndex
    .toUint128();
```
其中的`toUint128()`定义在`src/dependencies/openzeppelin/contracts/SafeCast.sol`中，用于保证`uint256`向`uint128`转换的安全无溢出。

更新`nextVariableBorrowIndex`的过程与上述过程基本类似，但有一点不同，浮动利率贴现因子严格按照复利进行计算，调用`calculateCompoundedInterest`函数计算复利。

> 由于此节专注于介绍存款流程，关于此处的贷出资产的利率和贴现因子等计算，我们不进行详细叙述，待后文介绍

最后，我们使用`reserve.lastUpdateTimestamp = uint40(block.timestamp);`更新了`lastUpdateTimestamp`时间戳。

#### _accrueToTreasury

首先，我们可以知道质押品的准备金率可以通过`reserveFactor`获得，此变量我们已经提前缓存到了`reserveCache`中，提取的准备金数量为当前借贷总量与风险准备金率的乘积。所以我们只需要解决计算借贷**增量**的问题。

> 此处我们只需要计算由于借贷利率导致的借贷的增量数据

此问题可以分解为以下两部分:

1. 计算浮动利率贷款增量
    
    已知目前贴现后的贷款总量和前贴现因子(`currVariableBorrowIndex`)和现贴现因子(`nextVariableBorrowIndex`)。我们可以通过以下公式进行计算:

    $${VD}\_{accrued} = {ScVB}\_{t} \times {VI}\_{t} - {ScVB}\_{t-1} \times {VI}\_{t-1} $$
    
    此公式中，${VD}_{accrued}$ 即表示浮动利率贷款增量，而 $ScVB$ 则表示浮动利率贴现因子

    具体代码如下:
    ```solidity
    vars.prevTotalVariableDebt = reserveCache.currScaledVariableDebt.rayMul(
        reserveCache.currVariableBorrowIndex
    );

    vars.currTotalVariableDebt = reserveCache.currScaledVariableDebt.rayMul(
        reserveCache.nextVariableBorrowIndex
    );
    ```
1. 计算固定利率贷款增量

    已知当前固定贷款总额(`reserveCache.currTotalStableDebt`)、 固定平均利率(`currAvgStableBorrowRate`)和以固定利率贷出的本金(`currPrincipalStableDebt`)。

    基于以上参数，我们首先计算出上一阶段的固定利率(基于`reserveLastUpdateTimestamp`参数和 `currAvgStableBorrowRate` 平均固定利率)，然后使用此参数计算出上一阶段的固定利率贷款总额`prevTotalStableDebt`。


有了上述参数计算风险准备金提取时极其简单的，代码如下:
```solidity
// 债务增量计算
vars.totalDebtAccrued =
    vars.currTotalVariableDebt +
    reserveCache.currTotalStableDebt -
    vars.prevTotalVariableDebt -
    vars.prevTotalStableDebt;
// 待铸造风险准备金计算
vars.amountToMint = vars.totalDebtAccrued.percentMul(
    reserveCache.reserveFactor
);
// 更新相关数据
if (vars.amountToMint != 0) {
    reserve.accruedToTreasury += vars
        .amountToMint
        .rayDiv(reserveCache.nextLiquidityIndex)
        .toUint128();
}
```

### 校验设置

此节主要讨论`validateSupply`函数，此函数主要用于校验存款是否符合一系列限制参数。

首先校验存款数量是否为 `0`，代码如下:
```solidity
require(amount != 0, Errors.INVALID_AMOUNT);
```

然后校验存款池是否被启用、是否处于暂停或冻结状态，代码如下:
```solidity
(bool isActive, bool isFrozen, , , bool isPaused) = reserveCache
    .reserveConfiguration
    .getFlags();
require(isActive, Errors.RESERVE_INACTIVE);
require(!isPaused, Errors.RESERVE_PAUSED);
require(!isFrozen, Errors.RESERVE_FROZEN);
```

> 此处使用了`reserveConfiguration`中的`getFlags()`函数，此函数被定义在`src/protocol/libraries/configuration/ReserveConfiguration.sol`中，主要基于位操作提取相应的参数

最后判断用户存款后市是否超过存款限额，代码如下:
```solidity
uint256 supplyCap = reserveCache.reserveConfiguration.getSupplyCap();
require(
    supplyCap == 0 ||
        (IAToken(reserveCache.aTokenAddress).scaledTotalSupply().rayMul(
            reserveCache.nextLiquidityIndex
        ) + amount) <=
        supplyCap *
            (10**reserveCache.reserveConfiguration.getDecimals()),
    Errors.SUPPLY_CAP_EXCEEDED
);
```

> 此处涉及到 ERC20 代币的精度问题，这也是一种顶点浮点数。如 USDT 的精度为 6， 则表示 1 USDT 在合约内使用 1e6 此数值表示，也意味着 USDT 精度最多为 6 位小数

### 更新利率

对于利率的更新主要集中在`updateInterestRates`函数内，但此函数将核心的利率计算分配给了`interestRateStrategyAddress`合约。此合约内包含一系列利率计算参数和具体的逻辑函数。

在具体计算利率时，我们需要以下参数:
```solidity
struct CalculateInterestRatesParams {
    uint256 unbacked;
    uint256 liquidityAdded;
    uint256 liquidityTaken;
    uint256 totalStableDebt;
    uint256 totalVariableDebt;
    uint256 averageStableBorrowRate;
    uint256 reserveFactor;
    address reserve;
    address aToken;
}
```

这些参数的含义为:

- `unbacked` 无存入直接铸造的代币数量上限(此变量用于跨链)
- `liquidityAdded` 流动性增加量，在此处为存入资产的数量
- `liquidityTaken` 流动性移除量，此变量用于贷出资产的情况，故而此值在存入流程内置为 0
- `totalStableDebt`和`totalVariableDebt` 固定利率总贷出量和浮动利率总贷出量
- `averageStableBorrowRate` 平均贷款固定利率
- `reserveFactor` 准备金比率
- `aToken` `aToken`地址

> 可能有读者好奇为什么会出现无存入入直接铸造代币的情况？ 此情况发生在跨链时，用户可能在另一区块链内存入了资产而在当前区块链铸造代币的情况

我们可以在`reserveCache`和`reserve`中找到此处使用的大部分变量，具体的构造代码如下:
```solidity
(
    vars.nextLiquidityRate,
    vars.nextStableRate,
    vars.nextVariableRate
) = IReserveInterestRateStrategy(reserve.interestRateStrategyAddress)
    .calculateInterestRates(
        DataTypes.CalculateInterestRatesParams({
            unbacked: reserveCache
                .reserveConfiguration
                .getUnbackedMintCap() != 0
                ? reserve.unbacked
                : 0,
            liquidityAdded: liquidityAdded,
            liquidityTaken: liquidityTaken,
            totalStableDebt: reserveCache.nextTotalStableDebt,
            totalVariableDebt: vars.totalVariableDebt,
            averageStableBorrowRate: reserveCache
                .nextAvgStableBorrowRate,
            reserveFactor: reserveCache.reserveFactor,
            reserve: reserveAddress,
            aToken: reserveCache.aTokenAddress
        })
    );
```

为减少本文篇幅，我们将具体的利率计算放在未来介绍。

在完成具体的利率计算后，我们将这些数据写入`reserve`，代码如下:
```solidity
reserve.currentLiquidityRate = vars.nextLiquidityRate.toUint128();
reserve.currentStableBorrowRate = vars.nextStableRate.toUint128();
reserve.currentVariableBorrowRate = vars.nextVariableRate.toUint128();
```

### 转移资产

当用户存入资产时，用户资产会被转移到流动性池内，此步骤通过一些函数完成:
```solidity
IERC20(params.asset).safeTransferFrom(
    msg.sender,
    reserveCache.aTokenAddress,
    params.amount
);
```
直接调用`ERC20`代币都实现的`safeTransferFrom`功能，将代币转移。

> 注意用户在存款前已经进行了`approve`授权操作，所以此处可以进行直接划转资产。

此次的`safeTransferFrom`被定义在`src/dependencies/gnosis/contracts/GPv2SafeERC20.sol`中，主要是用于兼容不同`ERC20`实现，一般来说，ERC20在转账失败后，有以下两者操作:

1. 使用`revert`回退交易和抛出异常
1. 返回`False`代表交易错误

具体代码如下:
```solidity
function safeTransferFrom(
    IERC20 token,
    address from,
    address to,
    uint256 value
) internal {
    bytes4 selector_ = token.transferFrom.selector;

    // solhint-disable-next-line no-inline-assembly
    assembly {
        let freeMemoryPointer := mload(0x40)
        mstore(freeMemoryPointer, selector_)
        mstore(
            add(freeMemoryPointer, 4),
            and(from, 0xffffffffffffffffffffffffffffffffffffffff)
        )
        mstore(
            add(freeMemoryPointer, 36),
            and(to, 0xffffffffffffffffffffffffffffffffffffffff)
        )
        mstore(add(freeMemoryPointer, 68), value)

        if iszero(call(gas(), token, 0, freeMemoryPointer, 100, 0, 0)) {
            returndatacopy(0, 0, returndatasize())
            revert(0, returndatasize())
        }
    }

    require(getLastTransferResult(token), "GPv2: failed transferFrom");
}
```
总体流程大致为:

1. 获取空闲内存地址
1. 构造请求`calldata`
    1. 写入`transferFrom`选择器(4 bytes)
    1. 写入`from`变量，此变量属于`address`类型，应占用 20 bytes(使用`and`运算保证长度)，但注意`mstore`一次写入 32 bytes，所以此处写入了32 bytes 的数据
    1. 写入`to`变量，首先计算起始内存位置，4 bytes 选择器加上上步写入的 32 bytes 的地址数据，所以此处起始位置应为 36 bytes
    1. 写入`value`变量
1. 使用`call`操作符发送请求
1. 判断`call`操作是否为`0`，如果为`0`，则意味着调用失败，进行`returndata`拷贝和`revert`回退操作

> `revert`含义为回退当前的`call`调用，恢复状态变量，并返回堆栈信息。值得注意的是，`returndata`区域内的数据并不会被回退。

通过以上操作，我们只能保证调用的准确性，而没有对某些通过返回值表示错误的合约的进行兼容，所以此处又使用了`getLastTransferResult`进行判断。此函数较为简单，功能为审核以下两种返回情况:

1. 返回值为空时，并进一步判断合约地址确实是合约地址时，我们认为调用正确
1. 返沪值包含数据时，我们需要判断数据是否为`True`，如果为`True`，则认为调用正确

如果不属于以上情况，我们通过以下函数:
```solidity
function revertWithMessage(length, message) {
    mstore(0x00, "\x08\xc3\x79\xa0")
    mstore(0x04, 0x20)
    mstore(0x24, length)
    mstore(0x44, message)
    revert(0x00, 0x64)
}
```
返回报错

> 由以上代码，我们可以充分认识到ERC20合约的多样性。建议读者使用一些较为通用的实现方案而不是自己造轮子

> 如果您无法理解上述内容，建议读者阅读[EVM底层探索:字节码级分析最小化代理标准EIP1167](./deep-in-eip1167)系列文章。在这些文章内，我较为详细的讨论了字节码问题。当然，我之前的文章基本均涉及到`yul`底层编程，读者可以按照我的写作的时间顺序逐个阅读

### 存款代币铸造

在完成存款资产转移后，合约需要铸造对应的`ATokens`，代码如下:
```solidity
bool isFirstSupply = IAToken(reserveCache.aTokenAddress).mint(
    msg.sender,
    params.onBehalfOf,
    params.amount,
    reserveCache.nextLiquidityIndex
);
```
此处调用了`AToken`的`mint`方法，此方法定义如下:
```solidity
function mint(
    address caller,
    address onBehalfOf,
    uint256 amount,
    uint256 index
) external virtual override onlyPool returns (bool) {
    return _mintScaled(caller, onBehalfOf, amount, index);
}
```
进一步查找`_mintScaled`方法定义如下:
```solidity
function _mintScaled(
    address caller,
    address onBehalfOf,
    uint256 amount,
    uint256 index
) internal returns (bool) {
    uint256 amountScaled = amount.rayDiv(index);
    require(amountScaled != 0, Errors.INVALID_MINT_AMOUNT);

    uint256 scaledBalance = super.balanceOf(onBehalfOf);
    uint256 balanceIncrease = scaledBalance.rayMul(index) -
        scaledBalance.rayMul(_userState[onBehalfOf].additionalData);

    _userState[onBehalfOf].additionalData = index.toUint128();

    _mint(onBehalfOf, amountScaled.toUint128());

    uint256 amountToMint = amount + balanceIncrease;
    emit Transfer(address(0), onBehalfOf, amountToMint);
    emit Mint(caller, onBehalfOf, amountToMint, balanceIncrease, index);

    return (scaledBalance == 0);
}
```
上述代码的运行流程如下:

1. 使用`amount.rayDiv(index)`计算应铸造的`AToken`数量并判断铸造数量是否为`0`
1. 使用`super.balanceOf(onBehalfOf)`获得用户当前持有的代币数量
1. 计算用户代币利息(`_userState[onBehalfOf].additionalData`为上一次的贴现因子)
1. 更新`_userState[onBehalfOf].additionalData`为当前贴现因子
1. 铸造代币`_mint(onBehalfOf, amountScaled.toUint128());`
1. 计算代币铸造量`uint256 amountToMint = amount + balanceIncrease;`
1. 释放事件
1. 返回账户是否为从零开始铸造代币

> 上述流程在真正的计算流程中，我们使用以折现后的代币数量进行计算，但为了抛出相关事件，我们有将折现后的代币数量与贴现因子相乘。这可能是为了一般用户方便查询自己当前的代币数量。但如此操作也增加了合约的复杂性，显然，在合约复杂性和用户体验方面，AAVE 开发者选择了用户体验

### 更新抵押品

此部分主要处理用户质押品情况，正如上一篇文章所述，在AAVE内的存款可以作为贷款抵押品存在也可以单纯作为存款操作，本部分主要处理此情况。

大部分情况下，用户资产是否作为贷款质押品都由用户自己决定，本部分代码仅针对用一些特殊情况。在这些特殊情况下，用户第一笔存入的资产会被默认为贷款抵押资产，启动资产抵押选项。

上述功能可通过运行以下代码实现:
```solidity
if (isFirstSupply) {
    if (
        ValidationLogic.validateUseAsCollateral(
            reservesData,
            reservesList,
            userConfig,
            reserveCache.reserveConfiguration
        )
    ) {
        userConfig.setUsingAsCollateral(reserve.id, true);
        emit ReserveUsedAsCollateralEnabled(
            params.asset,
            params.onBehalfOf
        );
    }
}
```
其中`userConfig.setUsingAsCollateral(reserve.id, true);`启动用户当前资产的抵押选项。

至于包含那些特殊情况，我们需要研究`validateUseAsCollateral`函数，此函数定义如下:
```solidity
function validateUseAsCollateral(
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(uint256 => address) storage reservesList,
    DataTypes.UserConfigurationMap storage userConfig,
    DataTypes.ReserveConfigurationMap memory reserveConfig
) internal view returns (bool) {
    if (!userConfig.isUsingAsCollateralAny()) {
        return true;
    }
    (bool isolationModeActive, , ) = userConfig.getIsolationModeState(
        reservesData,
        reservesList
    );

    return (!isolationModeActive && reserveConfig.getDebtCeiling() == 0);
}
```
根据代码，我们可以得出特殊情况包含的具体范围:

1. 用户当前状态下不存在任何一种抵押品资产，此判断通过`isUsingAsCollateralAny`实现。
1. 用户未启动了`isolation mode`且当前资产未被纳入`isolation mode`

出现上述情况之一，用户的存款会自动启用质押选项。下图展示了在无DAI存款情况下，进行存款自动启用质押选项的AAVE对话框:

![auto AAVE](https://img.gejiba.com/images/79bd0aad69d00e2d621e2e2f4aada1b8.png)

> 显然这也是为了增加用户体验增加代码复杂度的又一案例

## supplyWithPermit

在上文中，我们深挖了在`Pool`合约内的`supply`函数，这也是大部分用户存款时使用的函数，但事实上，`AAVE`也提供了另一个使用体验更好的函数`supplyWithPermit`。此函数的核心在于`Permit`，笔者在之前的文章内讨论过此概念，如果读者不了解此概念，请阅读[EIP712的扩展使用](./eip712-extend)，我们在本文的最后讨论了`Permit`这一函数。

在此处，我们给出`supplyWithPermit`的代码:
```solidity
function supplyWithPermit(
    address asset,
    uint256 amount,
    address onBehalfOf,
    uint16 referralCode,
    uint256 deadline,
    uint8 permitV,
    bytes32 permitR,
    bytes32 permitS
) public virtual override {
    IERC20WithPermit(asset).permit(
        msg.sender,
        address(this),
        amount,
        deadline,
        permitV,
        permitR,
        permitS
    );
    SupplyLogic.executeSupply(
        _reserves,
        _reservesList,
        _usersConfig[onBehalfOf],
        DataTypes.ExecuteSupplyParams({
            asset: asset,
            amount: amount,
            onBehalfOf: onBehalfOf,
            referralCode: referralCode
        })
    );
}
```

与正常的`supply`函数相比，此函数增加了`IERC20WithPermit(asset).permit`部分。此部分为`supply`增加了特殊的功能，即调用者不需要在使用此函数前进行`approve`授权操作，该授权操作隐含在`EIP712`签名中。如果读者无法理解此内容，请阅读[基于链下链上双视角深入解析以太坊签名与验证](./ecsda-sign-chain)和[EIP712的扩展使用](./eip712-extend)。

## 总结

终于我们完成了对于AAVE存款部分的描述，可见相对于[Safe](./deep-in-safe-part-1)等功能性合约，AAVE作为DeFi合约充分体现了其复杂性。本文是对AAVE V3版本存款的简单描述，由于篇幅和主题限制，本文对于部分函数的深挖不足，读者可根据自身需求在本文基础上继续深挖部分函数。
