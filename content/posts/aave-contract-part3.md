---
title: "深入解析AAVE智能合约:取款"
date: 2022-12-28T23:47:33Z
tags: [aave,defi]
math: true
---

## 概述

本文主要介绍`AAVE V3`合约中的取款`withdraw`函数。在阅读本文前，请读者确保已经阅读过以下文章:

1. [AAVE交互指南](https://hugo.wongssh.cf/posts/aave-interactive/)，本文将大量使用此文中给出的各种数学计算公式
1. [深入解析AAVE智能合约:存款](https://hugo.wongssh.cf/posts/aave-contract-part1/)，此篇文章内给出的部分函数和大部分数据结构在本文内页有所使用，重复部分在本文内不再解释

读者也可选读[深入解析AAVE智能合约:计算和利率](https://hugo.wongssh.cf/posts/aave-contract-part2/)，此文介绍了数学计算底层实现逻辑，与代码逻辑关系不大，读者可选读此文。

本文可认为是对[深入解析AAVE智能合约:存款](https://hugo.wongssh.cf/posts/aave-contract-part1/)的进一步补充，由于取款逻辑较为简单，所以此文的关键在于进一步深挖某些常用函数。这些函数在《存款》一文中虽有提及但未深入探讨的函数，如`updateInterestRates`等。

## 代码分析

在`src/protocol/pool/Pool.sol`合约内，我们可以找到如下函数:
```solidity
function withdraw(
    address asset,
    uint256 amount,
    address to
) public virtual override returns (uint256) {
    return
        SupplyLogic.executeWithdraw(
            _reserves,
            _reservesList,
            _eModeCategories,
            _usersConfig[msg.sender],
            DataTypes.ExecuteWithdrawParams({
                asset: asset,
                amount: amount,
                to: to,
                reservesCount: _reservesCount,
                oracle: ADDRESSES_PROVIDER.getPriceOracle(),
                userEModeCategory: _usersEModeCategory[msg.sender]
            })
        );
}
```
此处各变量的具体含义如下:

1. `_reserves` 资产地址和该资产的存款数据`ReserveData`的对应关系
1. `_reservesList` 资产`id`及其地址之间的对应关系，设置此映射目的是节省`gas`
1. `_eModeCategories` E-Mode资产id`eModeCategoryId`与EMode资产信息`eModeCategory`的映射
1. `_usersConfig` 用户地址与其设置之间的对应关系
1. `_reservesCount` 当前质押品的种类数量
1. `oracle` 预言机地址
1. `userEModeCategory` 用户启用`E-Mode`的种类

> 此处使用的变量已经在[深入解析AAVE智能合约:存款](https://hugo.wongssh.cf/posts/aave-contract-part1#基础数据结构)进行了相关讨论。

使用`Solidity Visual Developer`插件对其进行调用分析，结果如下:
```
└─ Pool::withdraw
   ├─ SupplyLogic::executeWithdraw | [Ext] ❗️  🛑 
   │  ├─ DataTypes.ReserveData::cache | [Int] 🔒   
   │  ├─ DataTypes.ReserveData::updateState | [Int] 🔒  🛑 
   │  ├─ SupplyLogic::type
   │  ├─ ValidationLogic::validateWithdraw | [Int] 🔒   
   │  ├─ DataTypes.ReserveData::updateInterestRates | [Int] 🔒  🛑 
   │  ├─ DataTypes.UserConfigurationMap::isUsingAsCollateral | [Int] 🔒   
   │  ├─ DataTypes.UserConfigurationMap::isBorrowingAny | [Int] 🔒   
   │  ├─ ValidationLogic::validateHFAndLtv | [Int] 🔒   
   │  │  └─ ValidationLogic::validateHealthFactor | [Int] 🔒   
   │  │     ├─ GenericLogic::calculateUserAccountData | [Int] 🔒   
   │  │     │  ├─ GenericLogic::type
   │  │     │  ├─ EModeLogic::getEModeConfiguration | [Int] 🔒   
   │  │     │  │  └─ IPriceOracleGetter::getAssetPrice | [Ext] ❗️   
   │  │     │  ├─ GenericLogic::_getUserBalanceInBaseCurrency | [Priv] 🔐   
   │  │     │  │  └─ DataTypes.ReserveData::getNormalizedIncome | [Int] 🔒   
   │  │     │  ├─ EModeLogic::isInEModeCategory | [Int] 🔒   
   │  │     │  └─ GenericLogic::_getUserDebtInBaseCurrency | [Priv] 🔐   
   │  │     │     ├─ userTotalDebt::rayMul | [Int] 🔒   
   │  │     │     └─ DataTypes.ReserveData::getNormalizedDebt | [Int] 🔒   
   │  │     └─ DataTypes::CalculateUserAccountDataParams
   │  └─ DataTypes.UserConfigurationMap::setUsingAsCollateral | [Int] 🔒  🛑 
   ├─ DataTypes::ExecuteWithdrawParams
   └─ IPoolAddressesProvider::getPriceOracle | [Ext] ❗️   
```

通过此调用流程，我们可以了解到`withdraw`函数基本情况。

## executeWithdraw

与`supply`函数的逻辑基本一致，`withdraw`函数仅通过入口作用，真正的逻辑代码位于`executeWithdraw`函数内，在本节内，我们将详细分析此函数的实现。

### 缓存与更新

与`executeSupply`函数一致，`executeWithdraw`在函数最开始进行了缓存及更新状态的相关操作，代码如下:
```solidity
DataTypes.ReserveData storage reserve = reservesData[params.asset];
DataTypes.ReserveCache memory reserveCache = reserve.cache();

reserve.updateState(reserveCache);
```
此处使用的基本都在[深入解析AAVE智能合约:存款](https://hugo.wongssh.cf/posts/aave-contract-part1#数据缓存)内进行了相关介绍和分析。总结来说，`updateState`完成了以下功能:

1. 更新`Index`系列变量
1. 更新风险准备金

### 获取存款数额

使用如下代码获得用户的存款余额情况:
```solidity
uint256 userBalance = IAToken(reserveCache.aTokenAddress)
    .scaledBalanceOf(msg.sender)
    .rayMul(reserveCache.nextLiquidityIndex);
```
其中，`scaledBalanceOf`函数定义如下:
```solidity
function scaledBalanceOf(address user)
    external
    view
    override
    returns (uint256)
{
    return super.balanceOf(user);
}
```
显然，通过此函数，我们可以获得经过**折现后的用户存款数额**。然后，经过`.rayMul(reserveCache.nextLiquidityIndex);`则可以得到当前用户存款的本息和。

> 关于此处为什么获得的是**折现后的用户存款数额**? 请阅读[上一篇文章](https://hugo.wongssh.cf/posts/aave-contract-part1#存款代币铸造)内讨论**存款代币的铸造**这一节。简单来说，我们记录用户存款数额时就使用了折现后的数额

### 确定取款数额

使用以下代码确定用户的取款数额:
```solidity
uint256 amountToWithdraw = params.amount;

if (params.amount == type(uint256).max) {
    amountToWithdraw = userBalance;
}
```
此处唯一需要注意的是，当用户取款数量为`type(uint256).max`时，我们将取款数额设置为用户的账户余额。

### 验证取款

验证取款主要通过以下代码实现:
```solidity
ValidationLogic.validateWithdraw(
    reserveCache,
    amountToWithdraw,
    userBalance
);
```

其中`validateWithdraw`函数实现代码如下:
```solidity
function validateWithdraw(
    DataTypes.ReserveCache memory reserveCache,
    uint256 amount,
    uint256 userBalance
) internal pure {
    require(amount != 0, Errors.INVALID_AMOUNT);
    require(
        amount <= userBalance,
        Errors.NOT_ENOUGH_AVAILABLE_USER_BALANCE
    );

    (bool isActive, , , , bool isPaused) = reserveCache
        .reserveConfiguration
        .getFlags();
    require(isActive, Errors.RESERVE_INACTIVE);
    require(!isPaused, Errors.RESERVE_PAUSED);
}
```
此函数依次校验了以下内容:

1. 用户取款数额是否为`0`，如果为 0 ，则报错
1. 取款数额是否大于用户账户余额，如果大于则报错
1. 存款池是否属于启用状态`isActive`，如果不属于则报错
1. 存款池是否被暂停，如果被暂停则报错

> 关于`isActive`、`isPaused`等内容，请阅读上一篇文章中的[特殊数据结构](https://hugo.wongssh.cf/posts/aave-contract-part1#特殊数据结构)一节。

### 利率更新

使用以下函数实现利率更新:
```solidity
reserve.updateInterestRates(
    reserveCache,
    params.asset,
    0,
    amountToWithdraw
);
```

在上一篇文章中的[更新利率](https://hugo.wongssh.cf/posts/aave-contract-part1#更新利率)一节内，我们讨论了`updateInterestRates`中的参数问题而没有讨论具体的`calculateInterestRates`函数的具体实现，我们在本文中将分析此函数。

在具体分析函数的逻辑代码之前，我们首先给出函数的调用代码:
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
此函数使用的参数含义请参考[上一篇文章](https://hugo.wongssh.cf/posts/aave-contract-part1#更新利率)。

在了解具体的输入参数后，我们着手分析函数的逻辑部分。在阅读以下内容前，请读者复习[AAVE交互指南](https://hugo.wongssh.cf/posts/aave-interactive/)中的[贷款利率计算](https://hugo.wongssh.cf/posts/aave-interactive#贷款利率计算)一节。

在`calculateInterestRates`函数内，我们首先定义了一系列初始变量，定义代码如下:

```solidity
CalcInterestRatesLocalVars memory vars;

vars.totalDebt = params.totalStableDebt + params.totalVariableDebt;

vars.currentLiquidityRate = 0;
vars.currentVariableBorrowRate = _baseVariableBorrowRate;
vars.currentStableBorrowRate = getBaseStableBorrowRate();
```

其中，各变量含义如下:

1. `totalDebt` 当前资产的总贷出数量
1. `currentLiquidityRate` 存款利率
1. `currentVariableBorrowRate` 浮动利率
1. `currentStableBorrowRate` 固定利率

> 此处`_baseVariableBorrowRate`即初始化浮动利率，而`getBaseStableBorrowRate()`为初始化固定利率。在之前文章内，我们使用 $R_0$ 表示此数值。

接下来，我们需要获得 利用率 $U$ 和 固定利率负债与浮动利率负债之比 $ratio$ 这两个参数。其计算公式分别为:

$$ U = \frac{Total\ borrowed}{Total\ supplied} $$
$$ ratio = \frac{Total\ Stable\ Debt}{Total\ Debt}$$

具体实现代码如下:
```solidity
if (vars.totalDebt != 0) {
    // 计算 ratio 变量值
    vars.stableToTotalDebtRatio = params.totalStableDebt.rayDiv(
        vars.totalDebt
    );
    // 计算当前流动性池的资产
    vars.availableLiquidity =
        IERC20(params.reserve).balanceOf(params.aToken) +
        params.liquidityAdded -
        params.liquidityTaken;
    // 计算当前总供给 Total supplied
    // 使用 资产 + 负债
    vars.availableLiquidityPlusDebt =
        vars.availableLiquidity +
        vars.totalDebt;
    // 计算 U (未考虑跨链数据)
    vars.borrowUsageRatio = vars.totalDebt.rayDiv(
        vars.availableLiquidityPlusDebt
    );
    // 计算 考虑跨链数据后修正的 U
    vars.supplyUsageRatio = vars.totalDebt.rayDiv(
        vars.availableLiquidityPlusDebt + params.unbacked
    );
}
```
我们首先处理二阶段利率问题，公式如下:

$$
R_{base} = \begin{cases}
   R_0 + \frac{U_t}{U_{optimal}} \times R_{slope1} &\text{if } U < U_{optimal} \\\
   R_0 + R_{slope1} + \frac{U_t-U_{optimal}}{1-U_{optimal}} \times R_{slope2} &\text{if } U \geq U_{optimal}
\end{cases}
$$

无论是计算固定利率还是计算浮动利率，上述公式都是有效的。

代码如下:
```solidity
if (vars.borrowUsageRatio > OPTIMAL_USAGE_RATIO) {
    uint256 excessBorrowUsageRatio = (vars.borrowUsageRatio -
        OPTIMAL_USAGE_RATIO).rayDiv(MAX_EXCESS_USAGE_RATIO);

    vars.currentStableBorrowRate +=
        _stableRateSlope1 +
        _stableRateSlope2.rayMul(excessBorrowUsageRatio);

    vars.currentVariableBorrowRate +=
        _variableRateSlope1 +
        _variableRateSlope2.rayMul(excessBorrowUsageRatio);
} else {
    vars.currentStableBorrowRate += _stableRateSlope1
        .rayMul(vars.borrowUsageRatio)
        .rayDiv(OPTIMAL_USAGE_RATIO);

    vars.currentVariableBorrowRate += _variableRateSlope1
        .rayMul(vars.borrowUsageRatio)
        .rayDiv(OPTIMAL_USAGE_RATIO);
}
```

此处的`MAX_EXCESS_USAGE_RATIO`的定义为`WadRayMath.RAY - optimalUsageRatio`，也就是 $1-U_{optimal}$ 。

然后，我们处理固定利率的特定问题，即最佳固定利率负债总负债比率常数 $O_{ratio}$ 的影响。公式如下:

$$
R_t = \begin{cases}
   R_{base} &\text{if } ratio < O_{ratio} \\\
   R_{base} + \frac{ratio - O_{ratio}}{1 - O_{ratio}} \times offset &\text{if } ratio \geq O_{ratio}
\end{cases}
$$

> $offset$ 是提前给定的参数。

上述公式仅对于固定利率有效，使用代码表示如下:
```solidity
if (vars.stableToTotalDebtRatio > OPTIMAL_STABLE_TO_TOTAL_DEBT_RATIO) {
    uint256 excessStableDebtRatio = (vars.stableToTotalDebtRatio -
        OPTIMAL_STABLE_TO_TOTAL_DEBT_RATIO).rayDiv(
            MAX_EXCESS_STABLE_TO_TOTAL_DEBT_RATIO
        );
    vars.currentStableBorrowRate += _stableRateExcessOffset.rayMul(
        excessStableDebtRatio
    );
}
```

正如上文所述，此处使用的`_stableRateExcessOffset`是合约创建时给定的。

最后，我们分析如何计算存款利率，公式如下:

$$ {LR}_t = \bar{R_t}{U_t} \times (1 - {Reserve\ Factor})$$

$$ 
\bar{R_t} = \frac{variable\\ debt}{total\\ debt} * {R_{variable}} + \frac{stable\\ debt}{total\\ debt} * {R_{stable}}
$$

简单来说，存款利率就是浮动利率和固定利率的加权平均数与利用率的乘积。

> 注意此处使用的利用率为考虑跨链因素后修正的利用率，即`supplyUsageRatio`。上述公式中的 ${Reserve\ Factor}$ 为风险准备金率。

使用代码表示为:

```solidity
vars.currentLiquidityRate = _getOverallBorrowRate(
    params.totalStableDebt,
    params.totalVariableDebt,
    vars.currentVariableBorrowRate,
    params.averageStableBorrowRate
).rayMul(vars.supplyUsageRatio).percentMul(
        PercentageMath.PERCENTAGE_FACTOR - params.reserveFactor
    );
```

其中使用的`_getOverallBorrowRate`函数用来计算固定利率和浮动利率的加权平均数，代码如下:

```solidity
function _getOverallBorrowRate(
    uint256 totalStableDebt,
    uint256 totalVariableDebt,
    uint256 currentVariableBorrowRate,
    uint256 currentAverageStableBorrowRate
) internal pure returns (uint256) {
    uint256 totalDebt = totalStableDebt + totalVariableDebt;

    if (totalDebt == 0) return 0;

    uint256 weightedVariableRate = totalVariableDebt.wadToRay().rayMul(
        currentVariableBorrowRate
    );

    uint256 weightedStableRate = totalStableDebt.wadToRay().rayMul(
        currentAverageStableBorrowRate
    );

    uint256 overallBorrowRate = (weightedVariableRate + weightedStableRate)
        .rayDiv(totalDebt.wadToRay());

    return overallBorrowRate;
}
```

到此，我们就计算完了所有参数。

### 代币燃烧

取款即意味着燃烧掉手中的`AToken`以换回对应的资产。所以燃烧代币是其中重要的一步，此处需要调用`burn`方法，代码如下:

```solidity
IAToken(reserveCache.aTokenAddress).burn(
    msg.sender,
    params.to,
    amountToWithdraw,
    reserveCache.nextLiquidityIndex
);
```

此处调用的`burn`实现代码如下:

```solidity
function burn(
    address from,
    address receiverOfUnderlying,
    uint256 amount,
    uint256 index
) external virtual override onlyPool {
    _burnScaled(from, receiverOfUnderlying, amount, index);
    if (receiverOfUnderlying != address(this)) {
        IERC20(_underlyingAsset).safeTransfer(receiverOfUnderlying, amount);
    }
}
```

可以注意到此处在代币燃烧`burn`后就进行了对应资产的转移。

对于具体的`_burnScaled`，代码实现如下:
 
```solidity
function _burnScaled(
    address user,
    address target,
    uint256 amount,
    uint256 index
) internal {
    uint256 amountScaled = amount.rayDiv(index);
    require(amountScaled != 0, Errors.INVALID_BURN_AMOUNT);

    uint256 scaledBalance = super.balanceOf(user);
    uint256 balanceIncrease = scaledBalance.rayMul(index) -
        scaledBalance.rayMul(_userState[user].additionalData);

    _userState[user].additionalData = index.toUint128();

    _burn(user, amountScaled.toUint128());

    if (balanceIncrease > amount) {
        uint256 amountToMint = balanceIncrease - amount;
        emit Transfer(address(0), user, amountToMint);
        emit Mint(user, user, amountToMint, balanceIncrease, index);
    } else {
        uint256 amountToBurn = amount - balanceIncrease;
        emit Transfer(user, address(0), amountToBurn);
        emit Burn(user, target, amountToBurn, balanceIncrease, index);
    }
}
```

上述代码逻辑流程如下:

1. 计算折现后的取款数额(`amountScaled`)
1. 获取用户当前折现后的存款数额(`scaledBalance`)
1. 计算用户当前代币利息(`balanceIncrease`)，此处使用的`additionalData`内包含上一次的贴现因子
1. 存储本次贴现因子
1. 以折现后的数额为基准燃烧代币
1. 判断用户取款数额与利息的大小
    1. 如果取款数额小于利息，则释放铸造(`Mint`)事件
    1. 如果取款数额大于利息，则释放燃烧(`Burn`)事件

### 质押属性处理

由于存在资产作为质押品以贷出其他资产的情况存在，所以需要对质押情况进行相关处理。代码如下:

```solidity
if (userConfig.isUsingAsCollateral(reserve.id)) {
    if (userConfig.isBorrowingAny()) {
        ValidationLogic.validateHFAndLtv(
            reservesData,
            reservesList,
            eModeCategories,
            userConfig,
            params.asset,
            msg.sender,
            params.reservesCount,
            params.oracle,
            params.userEModeCategory
        );
    }

    if (amountToWithdraw == userBalance) {
        userConfig.setUsingAsCollateral(reserve.id, false);
        emit ReserveUsedAsCollateralDisabled(params.asset, msg.sender);
    }
}
```

如果取款资产具有质押品属性且用户存在借款行为，则进行`HF`(Health Factor 健康因子)的校验。

> 关于`HF`参数，读者可参考 AAVE 交互指南 中的[贷款参数](https://hugo.wongssh.cf/posts/aave-interactive#贷款参数)一节。简单来说，如果`HF < 1`，则用户存在违约风险，需要对用户进行清算

此处也进行了质押属性的处理，如果用户取出所有存款，则直接关闭此资产的质押选型。

核心函数`validateHFAndLtv`的代码如下:
```solidity
function validateHFAndLtv(
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(uint256 => address) storage reservesList,
    mapping(uint8 => DataTypes.EModeCategory) storage eModeCategories,
    DataTypes.UserConfigurationMap memory userConfig,
    address asset,
    address from,
    uint256 reservesCount,
    address oracle,
    uint8 userEModeCategory
) internal view {
    DataTypes.ReserveData memory reserve = reservesData[asset];

    (, bool hasZeroLtvCollateral) = validateHealthFactor(
        reservesData,
        reservesList,
        eModeCategories,
        userConfig,
        from,
        userEModeCategory,
        reservesCount,
        oracle
    );

    require(
        !hasZeroLtvCollateral || reserve.configuration.getLtv() == 0,
        Errors.LTV_VALIDATION_FAILED
    );
}
```

上述代码要求满足以下两个条件之一:

1. 当前资产的 LTV 不为 0
1. 当前资产设置中的 LTV 为 0

此处使用的`validateHealthFactor`的具体实现如下:
```solidity
function validateHealthFactor(
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(uint256 => address) storage reservesList,
    mapping(uint8 => DataTypes.EModeCategory) storage eModeCategories,
    DataTypes.UserConfigurationMap memory userConfig,
    address user,
    uint8 userEModeCategory,
    uint256 reservesCount,
    address oracle
) internal view returns (uint256, bool) {
    (, , , , uint256 healthFactor, bool hasZeroLtvCollateral) = GenericLogic
        .calculateUserAccountData(
            reservesData,
            reservesList,
            eModeCategories,
            DataTypes.CalculateUserAccountDataParams({
                userConfig: userConfig,
                reservesCount: reservesCount,
                user: user,
                oracle: oracle,
                userEModeCategory: userEModeCategory
            })
        );

    require(
        healthFactor >= HEALTH_FACTOR_LIQUIDATION_THRESHOLD,
        Errors.HEALTH_FACTOR_LOWER_THAN_LIQUIDATION_THRESHOLD
    );

    return (healthFactor, hasZeroLtvCollateral);
}
```

显然，此代码验证`healthFactor`是否大于`1`，若小于`1`则报错。

### 释放事件

到此，我们介绍完了基本所有的流程，在所有逻辑代码运行结束后，释放`Withdraw`事件，代码如下:

```solidity
emit Withdraw(params.asset, msg.sender, params.to, amountToWithdraw);
```

完成提款。

## 验证分析

在[质押属性处理](#质押属性处理)一节中，我们省略了`calculateUserAccountData`函数的具体实现，在本节中，我们将分析此函数的具体实现。

此函数的定义如下:
```solidity
function calculateUserAccountData(
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(uint256 => address) storage reservesList,
    mapping(uint8 => DataTypes.EModeCategory) storage eModeCategories,
    DataTypes.CalculateUserAccountDataParams memory params
)
    internal
    view
    returns (
        uint256,
        uint256,
        uint256,
        uint256,
        uint256,
        bool
    )
```

该函数的返回值依次为:

1. 用户的总质押品价值
1. 用户的总债务
1. 用户的平均LTV
1. 用户的平均清算门槛
1. 用户的健康系数
1. 如果 LTV 为 0 ，则返回 True

此处也需要考虑用户的输入，输入依次为:

1. `reservesData` 质押品相关信息，详情参考[基础数据结构](https://hugo.wongssh.cf/posts/aave-contract-part1#基础数据结构)一节
1. `reservesList` 资产`id`及其地址之间的对应关系
1. `eModeCategories` 当前所属的`EMode`类型
1. `params` 复合参数数据结构，详细分析见下文

`params`定义如下:
```solidity
struct CalculateUserAccountDataParams {
    UserConfigurationMap userConfig;
    uint256 reservesCount;
    address user;
    address oracle;
    uint8 userEModeCategory;
}
```

该数据结构中，各参数含义如下:

1. `userConfig` 用户持有及贷出资产的压缩列表，详情参考[基础数据结构](https://hugo.wongssh.cf/posts/aave-contract-part1#基础数据结构)一节
1. `reservesCount` 流动性池内资产的数量
1. `user` 用户地址
1. `oracle` 资产价格预言机地址
1. `userEModeCategory` 用户使用的`EMode`类型

接下来，我们分析此函数的逻辑部分。

```solidity
if (params.userConfig.isEmpty()) {
    return (0, 0, 0, 0, type(uint256).max, false);
}
```

当用户未存入或借出资产时，其健康系数为最大且 `LTV` 不为 0 

接下来，代码会查看用户当前是否启用`EMode`并查询`EMode`资产的相关信息，代码如下:
```solidity
if (params.userEModeCategory != 0) {
    (
        vars.eModeLtv,
        vars.eModeLiqThreshold,
        vars.eModeAssetPrice
    ) = EModeLogic.getEModeConfiguration(
        eModeCategories[params.userEModeCategory],
        IPriceOracleGetter(params.oracle)
    );
}
```

`getEModeConfiguration`函数的具体定义如下:
```solidity
function getEModeConfiguration(
    DataTypes.EModeCategory storage category,
    IPriceOracleGetter oracle
)
    internal
    view
    returns (
        uint256,
        uint256,
        uint256
    )
{
    uint256 eModeAssetPrice = 0;
    address eModePriceSource = category.priceSource;

    if (eModePriceSource != address(0)) {
        eModeAssetPrice = oracle.getAssetPrice(eModePriceSource);
    }

    return (category.ltv, category.liquidationThreshold, eModeAssetPrice);
}
```

该代码较为简单，其主体逻辑是访问预言机获得价格信息并返回相关信息。  

接下来，我们进入`while (vars.i < params.reservesCount)`循环，也是此代码的核心部分。

进入循环后，我们会首先查询用户是否存在此资产的质押或贷出，代码如下:
```solidity
if (!params.userConfig.isUsingAsCollateralOrBorrowing(vars.i)) {
    unchecked {
        ++vars.i;
    }
    continue;
}
```

此处使用的`isUsingAsCollateralOrBorrowing`函数如下:
```solidity
function isUsingAsCollateralOrBorrowing(
    DataTypes.UserConfigurationMap memory self,
    uint256 reserveIndex
) internal pure returns (bool) {
    unchecked {
        require(
            reserveIndex < ReserveConfiguration.MAX_RESERVES_COUNT,
            Errors.INVALID_RESERVE_INDEX
        );
        return (self.data >> (reserveIndex << 1)) & 3 != 0;
    }
}
```

要想了解此代码，读者应了解`UserConfigurationMap`的基本结构，我们在[深入解析AAVE智能合约:存款](https://hugo.wongssh.cf/posts/aave-contract-part1#基础数据结构)中进行过讨论，此处我们再次给出相关数据结构，如下图:

![UserConfigurationMap](https://img.gejiba.com/images/44730e1a6fb788ad2d9adf3678e89a39.png)

为判断用户是否存在此资产的质押或贷出，我们只需要判断对应两位上的数字是否存在`1`。我们首先使用`self.data >> (reserveIndex << 1)`将待判断位置右移至第 0 位和第 1 位。然后将其与`3`(二进制表示为`11`)进行`&`(`and`)操作，判断结果是否为`0`，如果为`0`则证明该资产被标识为`00`，即该资产既不作为用户的质押资产也不作为用户的贷出资产。

> 注意此处`&`的`3`的类型为`uint256`，其`11`前存在 254 位的`0`，这也保证了在`&`操作时其他资产并不会对结果产生影响。

在确定用户存在对应资产的质押或贷出，我们将查询资产对应的资产地址，代码如下:
```solidity
vars.currentReserveAddress = reservesList[vars.i];

if (vars.currentReserveAddress == address(0)) {
    unchecked {
        ++vars.i;
    }
    continue;
}

DataTypes.ReserveData storage currentReserve = reservesData[
    vars.currentReserveAddress
];

(
    vars.ltv,
    vars.liquidationThreshold,
    ,
    vars.decimals,
    ,
    vars.eModeAssetCategory
) = currentReserve.configuration.getParams();

unchecked {
    vars.assetUnit = 10**vars.decimals;
}
```

关于`reservesList`、`reservesData`的数据结构问题，请参考本系列第一篇文章中的[基础数据结构](https://hugo.wongssh.cf/posts/aave-contract-part1#基础数据结构)。

上述代码中使用的核心函数`getParams`定义如下:
```solidity
function getParams(DataTypes.ReserveConfigurationMap memory self)
    internal
    pure
    returns (
        uint256,
        ...
    )
{
    uint256 dataLocal = self.data;

    return (
        dataLocal & ~LTV_MASK,
        (dataLocal & ~LIQUIDATION_THRESHOLD_MASK) >>
            LIQUIDATION_THRESHOLD_START_BIT_POSITION,
        (dataLocal & ~LIQUIDATION_BONUS_MASK) >>
            LIQUIDATION_BONUS_START_BIT_POSITION,
        (dataLocal & ~DECIMALS_MASK) >> RESERVE_DECIMALS_START_BIT_POSITION,
        (dataLocal & ~RESERVE_FACTOR_MASK) >>
            RESERVE_FACTOR_START_BIT_POSITION,
        (dataLocal & ~EMODE_CATEGORY_MASK) >>
            EMODE_CATEGORY_START_BIT_POSITION
    );
}
```
我们在[特殊数据结构](https://hugo.wongssh.cf/posts/aave-contract-part1#特殊数据结构)一节讨论过此函数的基本原理。此函数的返回值依次为`ltv`、`Liquidation threshold`、`Liquidation bonus`、`Decimals`、`reserve factor`、`eMode category`，这些名词的具体含义请自行查询[特殊数据结构](https://hugo.wongssh.cf/posts/aave-contract-part1#特殊数据结构)一节。

在查询完资产的基本情况后，我们继续查询资产的当前价格，代码如下:
```solidity
vars.assetPrice = vars.eModeAssetPrice != 0 &&
    params.userEModeCategory == vars.eModeAssetCategory
    ? vars.eModeAssetPrice
    : IPriceOracleGetter(params.oracle).getAssetPrice(
        vars.currentReserveAddress
    );
```
总结来说，上述代码实现了以下功能:

1. 如果已查询到了资产的价格且用户处于对应的`eMode`则直接使用`eModeAssetPrice`
1. 如果不满足上述条件，则直接调用预言机查询资产价格

如果资产满足存在清算阈值且被用户作为担保品，则进入以下处理:

```solidity
if (
    vars.liquidationThreshold != 0 &&
    params.userConfig.isUsingAsCollateral(vars.i)
) {
    // 计算当前资产的价值
    vars.userBalanceInBaseCurrency = _getUserBalanceInBaseCurrency(
        params.user,
        currentReserve,
        vars.assetPrice,
        vars.assetUnit
    );
    // 累加计入总抵押品价值
    vars.totalCollateralInBaseCurrency += vars
        .userBalanceInBaseCurrency;
    // 判断当前资产是否与用户 EMode 相匹配
    vars.isInEModeCategory = EModeLogic.isInEModeCategory(
        params.userEModeCategory,
        vars.eModeAssetCategory
    );
    // 计算 平均 ltv
    if (vars.ltv != 0) {
        vars.avgLtv +=
            vars.userBalanceInBaseCurrency *
            (vars.isInEModeCategory ? vars.eModeLtv : vars.ltv);
    } else {
        vars.hasZeroLtvCollateral = true;
    }
    // 计算平均清算阈值
    vars.avgLiquidationThreshold +=
        vars.userBalanceInBaseCurrency *
        (
            vars.isInEModeCategory
                ? vars.eModeLiqThreshold
                : vars.liquidationThreshold
        );
}
```

此处，我们使用`_getUserBalanceInBaseCurrency`获取循环中当前资产价值，该函数的定义为:

```solidity
function _getUserBalanceInBaseCurrency(
    address user,
    DataTypes.ReserveData storage reserve,
    uint256 assetPrice,
    uint256 assetUnit
) private view returns (uint256) {
    uint256 normalizedIncome = reserve.getNormalizedIncome();
    uint256 balance = (
        IScaledBalanceToken(reserve.aTokenAddress)
            .scaledBalanceOf(user)
            .rayMul(normalizedIncome)
    ) * assetPrice;

    unchecked {
        return balance / assetUnit;
    }
}
```

上述代码较为简单，其中`normalizedIncome`是当前的贴现因子，`scaledBalanceOf`可以获得用户经过贴先后的资产余额。

我们也使用了`isInEModeCategory`判断当前资产是否处于与用户匹配的`EMode`状态，该函数定义如下:

```solidity
function isInEModeCategory(
    uint256 eModeUserCategory,
    uint256 eModeAssetCategory
) internal pure returns (bool) {
    return (eModeUserCategory != 0 &&
        eModeAssetCategory == eModeUserCategory);
}
```

该函数仅通过判断用户当前启用的`EMode`资产类型与当前资产的`EMode`类型是否相同实现判断匹配的目的。

> 读者如果无法理解为什么需要匹配，请参考 AAVE交互指南 中 [EMode](https://hugo.wongssh.cf/posts/aave-interactive#e-mode) 一节。

通过上述代码，我们可以获得用户质押品的相关情况。同时，我们也需要获得用户借款情况，获取方法如下:

```solidity
if (params.userConfig.isBorrowing(vars.i)) {
    vars.totalDebtInBaseCurrency += _getUserDebtInBaseCurrency(
        params.user,
        currentReserve,
        vars.assetPrice,
        vars.assetUnit
    );
}
```

上述代码中使用的`_getUserDebtInBaseCurrency`与`_getUserBalanceInBaseCurrency`实现方面基本一致，此处不再分析相关代码。

最后，我们进行循环变量增加以实现进一步循环:

```solidity
unchecked {
    ++vars.i;
}
```

经过对用户资产的遍历，我们已经获得了用户质押品和贷款的综合情况，接下来我们可以计算一些综合性变量，如 平均LTV(`avgLtv`) 和 平均清算阈值(`avgLiquidationThreshold`)，计算方法如下:

```solidity
unchecked {
    vars.avgLtv = vars.totalCollateralInBaseCurrency != 0
        ? vars.avgLtv / vars.totalCollateralInBaseCurrency
        : 0;
    vars.avgLiquidationThreshold = vars.totalCollateralInBaseCurrency !=
        0
        ? vars.avgLiquidationThreshold /
            vars.totalCollateralInBaseCurrency
        : 0;
}
```

最后，我们计算用户当前的健康因子(`healthFactor`)，计算公式如下:

$$H_f = \frac{\sum({Collateral_i\ in\ ETH} \times {Liquidation\ Threshold}_i)}{Total\ Borrows\ in\ ETH}$$

实现如下:

```solidity
vars.healthFactor = (vars.totalDebtInBaseCurrency == 0)
    ? type(uint256).max
    : (
        vars.totalCollateralInBaseCurrency.percentMul(
            vars.avgLiquidationThreshold
        )
    ).wadDiv(vars.totalDebtInBaseCurrency);
```

经过上述计算，我们可以获得一系列数据，最后将其返回:

```solidity
return (
    vars.totalCollateralInBaseCurrency,
    vars.totalDebtInBaseCurrency,
    vars.avgLtv,
    vars.avgLiquidationThreshold,
    vars.healthFactor,
    vars.hasZeroLtvCollateral
);
```

> 对于这些返回值的含义，我们在本节开始就已经提到，此处不再赘述

## 总结

本文主要讨论`withdraw`函数的相关设计，其中大部分核心函数已经在上文提及，但本文也在上文的基础上进行了部分函数实现进行了深挖以帮助读者进一步了解`AAVE`。

本文也进一步对`AAVE`中各主要参数的计算提供了`solidity`实现，希望对读者有所帮助。
