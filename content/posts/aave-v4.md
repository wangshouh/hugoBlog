---
title: "现代 DeFi: AAVE V4"
date: 2025-12-11T23:47:33Z
tags: [aave,defi]
math: true
---

## 概述

本文核心内容其实是对 AAVE v4 代码仓库的 [Overview 文档](https://github.com/aave/aave-v4/blob/main/docs/overview.md) 的翻译，但是相比于文档，本文补充了与该部分对应的代码，所以本文可以视为以文档作为纲领对 AAVE v4 代码库的阅读。 AAVE v4 继承了 AAVE v3 内的一些概念，对于这些概念，建议读者阅读笔者之前编写的 [AAVE 交互指南](https://blog.wssh.dev/posts/aave-interactive/) 一文，该文内给出了概念的功能和数学表达。

## 代码概述

我们可以使用 `find . -path "./src/*.sol"  -not -path "./src/dependencies/*" | xargs wc -l | sort -nr` 命令获得 AAVE v4 内的所有智能合约的行数:

```
		8263 total
    1049 ./src/spoke/Spoke.sol
     989 ./src/hub/Hub.sol
     557 ./src/spoke/libraries/LiquidationLogic.sol
     520 ./src/spoke/interfaces/ISpoke.sol
     429 ./src/hub/interfaces/IHub.sol
     330 ./src/hub/interfaces/IHubBase.sol
     321 ./src/hub/HubConfigurator.sol
     286 ./src/spoke/SpokeConfigurator.sol
     256 ./src/position-manager/SignatureGateway.sol
     236 ./src/spoke/libraries/PositionStatusMap.sol
     235 ./src/spoke/interfaces/ISpokeBase.sol
     226 ./src/hub/interfaces/IHubConfigurator.sol
     210 ./src/spoke/interfaces/ISpokeConfigurator.sol
     208 ./src/hub/libraries/AssetLogic.sol
     205 ./src/libraries/math/WadRayMath.sol
     164 ./src/position-manager/NativeTokenGateway.sol
     149 ./src/position-manager/interfaces/ISignatureGateway.sol
     141 ./src/position-manager/libraries/EIP712Hash.sol
     137 ./src/hub/AssetInterestRateStrategy.sol
     121 ./src/spoke/TreasurySpoke.sol
     114 ./src/access/AccessManagerEnumerable.sol
     107 ./src/libraries/math/MathUtils.sol
     101 ./src/hub/interfaces/IAssetInterestRateStrategy.sol
      89 ./src/spoke/libraries/KeyValueList.sol
      89 ./src/position-manager/interfaces/INativeTokenGateway.sol
      83 ./src/libraries/types/EIP712Types.sol
      83 ./src/libraries/math/PercentageMath.sol
      77 ./src/spoke/AaveOracle.sol
      75 ./src/misc/UnitPriceFeed.sol
      68 ./src/spoke/interfaces/ITreasurySpoke.sol
      65 ./src/access/interfaces/IAccessManagerEnumerable.sol
      58 ./src/position-manager/GatewayBase.sol
      53 ./src/spoke/interfaces/IAaveOracle.sol
      52 ./src/hub/libraries/SharesMath.sol
      51 ./src/utils/NoncesKeyed.sol
      42 ./src/hub/libraries/Premium.sol
      41 ./src/utils/Rescuable.sol
      39 ./src/position-manager/interfaces/IGatewayBase.sol
      33 ./src/spoke/instances/SpokeInstance.sol
      32 ./src/hub/interfaces/IBasicInterestRateStrategy.sol
      28 ./src/utils/Multicall.sol
      26 ./src/interfaces/IRescuable.sol
      25 ./src/spoke/interfaces/IPriceOracle.sol
      20 ./src/interfaces/INoncesKeyed.sol
      17 ./src/position-manager/interfaces/INativeWrapper.sol
      13 ./src/libraries/types/Roles.sol
      13 ./src/interfaces/IMulticall.sol
```

相比于 AAVE v3 代码体量和复杂度而言，AAVE v4 显然大幅度降低了代码复杂度，并且舍弃了过去复杂的面向对象的工程框架，也与时俱进的使用了“组合大于继承”的规范。

## Hub and Spokes

在流动性管理中使用 `hub-and-spoke model` 架构，其中 hub 处理流动性，而 spoke 处理特定资产的 borrow 和 lend。

![AAVE v4 architecture](https://img.gopic.xyz/aave-v4-arch.png)

Spokes 是一个独立模块，可以连接一个或者多个 hub。User actions (主要包括 supply / withdraw 和 borrow / repay) 会通过 Spoke 进行路由。Spoke 会基于 reserve configuration 和可用容量与合适的 Hub 交互。当流动性返回 Hub 时，Spokes 需要支付基础利率(取决于 Hub 的利率策略)和风险溢价(取决于用户的担保品组合)。

Hub 可以连接数量不定的 Spokes，每一个 Spokes 都可以为 Hub 提供债务与之对应的利息。Hub 内部可以管理流动性、利率、可用容量和其它参数。

### Hub

Hub 是不可变的，并且作为 AAVE v4 流动性管理的核心。AAVE v4 系统内可用存在多个 Hub，并且每一个 Hub 都维持自己的 Spokes 集合并进行管理。Hub 内部的的状态变量如下:

```solidity
/// @dev Number of assets listed in the Hub.
uint256 internal _assetCount;

/// @dev Map of asset identifiers to Asset data.
mapping(uint256 assetId => Asset) internal _assets;

/// @dev Map of asset identifiers and spoke addresses to Spoke data.
mapping(uint256 assetId => mapping(address spoke => SpokeData)) internal _spokes;

/// @dev Map of asset identifiers to set of spoke addresses.
mapping(uint256 assetId => EnumerableSet.AddressSet) internal _assetToSpokes;

/// @dev Set of underlying addresses listed as assets in the Hub.
EnumerableSet.AddressSet internal _underlyingAssets;
```

其中 `_assets` 内部存储有所有资产需要的数据，而 `_spokes` 内部包含了 Hub 对 Spokes 的配置，每一个 Hub 都可以为它的 Spokes 设置 supply/borrow 的容量以及其他参数，我们会在后文介绍 Spoke 时的介绍内部的参数。`_assetToSpokes` 用于存储 `assetId` 对应的 Spokes 列表，而 `_underlyingAssets` 代表 Hub 内部包含所有的 assets 列表。

Hub 的设计目标是尽可能的简单，核心是进行实施不变量检查。Hub 的核心特性是:

- 维护授权 Spokes 支持资产的注册情况
- 设置 Spokes 可以添加(add) 或者抽取(draw) 的流动性限额
- 强制检查不变量，我们会在后文介绍一些不变量的检查

我们首先介绍 Hub 维护资产注册情况的特性，核心函数是 `addAsset` 和 `updateAssetConfig`。`addAsset` 的具体实现如下:

```solidity
/// @inheritdoc IHub
function addAsset(
  address underlying,
  uint8 decimals,
  address feeReceiver,
  address irStrategy,
  bytes calldata irData
) external restricted returns (uint256) {
  require(
    underlying != address(0) && feeReceiver != address(0) && irStrategy != address(0),
    InvalidAddress()
  );
  require(
    MIN_ALLOWED_UNDERLYING_DECIMALS <= decimals && decimals <= MAX_ALLOWED_UNDERLYING_DECIMALS,
    InvalidAssetDecimals()
  );
  require(!_underlyingAssets.contains(underlying), UnderlyingAlreadyListed());

  uint256 assetId = _assetCount++;
  IBasicInterestRateStrategy(irStrategy).setInterestRateData(assetId, irData);
  uint256 drawnRate = IBasicInterestRateStrategy(irStrategy).calculateInterestRate({
    assetId: assetId,
    liquidity: 0,
    drawn: 0,
    deficit: 0,
    swept: 0
  });

  uint256 drawnIndex = WadRayMath.RAY;
  uint256 lastUpdateTimestamp = block.timestamp;
  _assets[assetId] = Asset({
    ...
  });
  _underlyingAssets.add(underlying);
  _addFeeReceiver(assetId, feeReceiver);

  emit AddAsset(assetId, underlying, decimals);
  emit UpdateAssetConfig(
    assetId,
    AssetConfig({
      feeReceiver: feeReceiver,
      liquidityFee: 0,
      irStrategy: irStrategy,
      reinvestmentController: address(0)
    })
  );
  emit UpdateAsset(assetId, drawnIndex, drawnRate, 0);

  return assetId;
}
```

我们忽略过一些语句，只关注一些核心内容。`assetId` 是 `_assetCount` 决定的，每次加入新的资产都会自增 `_assetCount`。在利率策略中，我们使用 `IBasicInterestRateStrategy(irStrategy).setInterestRateData(assetId, irData);` 初始化利率策略合约。在 AAVE v4 内部，利率是比较有趣的，简单来说，AAVE v4 内的利率被分为基础利率和 User Risk Premium。基础利率就是上述代码内的 `drawnRate`，该利率代表 Hub 内流动性的成本。`drawnIndex` 是一个用于累积利率的变量，类似 AAVE v3 内部的 `liquidityIndex` 变量。简单来说，我们将 `drawnIndex` 视为一个自资产创建以来一直被累积的利率，当用户提供资产时，我们会使用 `drawnIndex` 进行贴现。

在上述代码内的 `reinvestmentController` 是一个特殊的并没有在上文内提及的特性。该特性允许 Hub 内的空闲流动性拿出去进行再投资。此处会涉及到 `sweep` 和 `reclaim` 函数，其中 `sweep` 函数是用于在 Hub 内提取流动性，而 `reclaim` 用于 `reinvestmentController` 放回流动性。需要注意的，`reinvestmentController` 在 Hub 内提取资金不需要支付利息。

另一个用于资产管理的函数是 `updateAssetConfig`，该函数的实现如下:

```solidity
/// @inheritdoc IHub
function updateAssetConfig(
  uint256 assetId,
  AssetConfig calldata config,
  bytes calldata irData
) external restricted {
  require(assetId < _assetCount, AssetNotListed());
  Asset storage asset = _assets[assetId];
  asset.accrue();

  require(config.liquidityFee <= PercentageMath.PERCENTAGE_FACTOR, InvalidLiquidityFee());
  require(config.feeReceiver != address(0) && config.irStrategy != address(0), InvalidAddress());
  require(
    config.reinvestmentController != address(0) || asset.swept == 0,
    InvalidReinvestmentController()
  );

  if (config.irStrategy != asset.irStrategy) {
    asset.irStrategy = config.irStrategy;
    IBasicInterestRateStrategy(config.irStrategy).setInterestRateData(assetId, irData);
  } else {
    require(irData.length == 0, InvalidInterestRateStrategy());
  }

  address oldFeeReceiver = asset.feeReceiver;
  if (oldFeeReceiver != config.feeReceiver) {
    _mintFeeShares(asset, assetId);
    IHub.SpokeConfig memory spokeConfig;
    spokeConfig.active = _spokes[assetId][oldFeeReceiver].active;
    spokeConfig.paused = _spokes[assetId][oldFeeReceiver].paused;
    _updateSpokeConfig(assetId, oldFeeReceiver, spokeConfig);
    asset.feeReceiver = config.feeReceiver;
    _addFeeReceiver(assetId, config.feeReceiver);
  }

  asset.liquidityFee = config.liquidityFee;
  asset.reinvestmentController = config.reinvestmentController;

  asset.updateDrawnRate(assetId);

  emit UpdateAssetConfig(assetId, config);
}
```

在更新配置时，我们第一步是调用 `accrue` 进行利率累积计算，我们在后续介绍利率时，可以介绍 `accure` 内部对利率的计算，但此处我们可以给出 `drawnIndex` 的计算，我们可以明显看到 `drawnIndex` 本质上是单位资产自协议创建以来到现在累积的利息。

```solidity
previousIndex.rayMulUp(
  MathUtils.calculateLinearInterest(asset.drawnRate, lastUpdateTimestamp)
);
```

假如新的 `feeReceiver` 与 `oldFeeReceiver` 不同时，我们需要调用 `_mintFeeShares(asset, assetId);` 为旧的手续费接受者发送手续费。

在介绍 Hub 内部的流动性限额问题之前，在 Hub 合约内存在四个函数，注意这四个函数只能由 Spoke 进行调用。

1. `add` 用于存款人(lender) 添加流动性
2. `remove` 用于存款人移除流动性
3. `draw` 用于借款人借出资产
4. `restore` 用于借款人偿还资产

除了上述 4 个基础功能的函数，Hub 也提供了两个函数用于坏账处理:

1. `reportDeficit` 用于 Spoke 通知 Hub 坏账情况
2. `eliminateDeficit` 用于 Spoke 通知 Hub 坏账清除

上述 6 个函数都存在对应的验证函数，分别是 `_validateAdd` / `_validateRemove` / `_validateDraw` / `_validateRestore` / `_validateReportDeficit` / `_validateEliminateDeficit` 。这些函数只是初步检查用户输入的参数是否合理，主要是检查是否超过了 Hub 为 Spoke 设置的限额。在调用完成这些 `_validate` 函数后，代码内还会进行一些更加细致的检查，

其中 `_validateAdd` 函数检查内容较少，我们在该函数内部检查增加的流动性是否超过 `spoke.addCap`。

```solidity
function _validateAdd(
  Asset storage asset,
  SpokeData storage spoke,
  uint256 amount
) internal view {
  require(amount > 0, InvalidAmount());
  require(spoke.active, SpokeNotActive());
  require(!spoke.paused, SpokePaused());
  uint256 addCap = spoke.addCap;
  require(
    addCap == MAX_ALLOWED_SPOKE_CAP ||
      addCap * MathUtils.uncheckedExp(10, asset.decimals) >=
      asset.toAddedAssetsUp(spoke.addedShares) + amount,
    AddCapExceeded(addCap)
  );
}
```

在此处，我们也可以简单看一下 `add` 函数的实现，与大部分较为现代的智能合约类似，AAVE v4 没有使用 callback 通知调用者发送资产的方式或者使用 `transferFrom` 直接从 Spoke 处转移资产，而是使用了事后检查机制，我们可以看到如下代码:

```solidity
  uint256 liquidity = asset.liquidity + amount;
  uint256 balance = asset.underlying.balanceOf(address(this));
  require(balance >= liquidity, InsufficientTransferred(liquidity.uncheckedSub(balance)));
  uint120 shares = asset.toAddedSharesDown(amount).toUint120();
  require(shares > 0, InvalidShares());
  asset.addedShares += shares;
  spoke.addedShares += shares;
  asset.liquidity = liquidity.toUint120();

  asset.updateDrawnRate(assetId);

  emit Add(assetId, msg.sender, shares, amount);
```

Spoke 在添加资产时应该首先向 Hub 内转入代币，然后调用 `add` 函数通知 Hub 添加资产的数量，最后 Hub 会在合约内检查 Spoke 发送的资产数量。

`_validateRemove` 代码也十分简单，只是检查用户输入的 `amount > 0`。但是我们需要注意在 `remove` 函数内，我们进行了额外的检查:

```solidity
uint256 liquidity = asset.liquidity;
require(amount <= liquidity, InsufficientLiquidity(liquidity));
```

该检查避免了 Spoke 借出大于 Hub 持有的资产数量，对应于上文内介绍的 Hub 需要保持 Total borrowed assets <= total supplied assets 的不变量。上述 `InsufficientLiquidity` 在 `draw` 函数内也有出现，都是为了保证借出资产 <= 供应资产不变量的成立。

`_validateDraw` 需要使用 `_getSpokeDrawn` 计算基础的负债情况和 `_getSpokePremium` 计算额外的负债情况，该参数与 Premium Debt 有关，这部分内容会在后文进行介绍。此处主要检查 Spoke 对 Hub 的总负债是否大于限额，另一个有趣的是此处的出现了 `spoke.deficitRay` 变量，该变量指当前 Spoke 的坏账情况

```solidity
/// @dev Spoke with maximum cap have unlimited draw capacity.
function _validateDraw(
  Asset storage asset,
  SpokeData storage spoke,
  uint256 amount,
  address to
) internal view {
  require(to != address(this), InvalidAddress());
  require(amount > 0, InvalidAmount());
  require(spoke.active, SpokeNotActive());
  require(!spoke.paused, SpokePaused());
  uint256 drawCap = spoke.drawCap;
  uint256 owed = _getSpokeDrawn(asset, spoke) + _getSpokePremium(asset, spoke);
  require(
    drawCap == MAX_ALLOWED_SPOKE_CAP ||
      drawCap * MathUtils.uncheckedExp(10, asset.decimals) >=
      owed + amount + uint256(spoke.deficitRay).fromRayUp(),
    DrawCapExceeded(drawCap)
  );
}
```

而 `_validateRestore` 主要用于对 Spoke 向 Hub 清偿债务进行检查。在进行债务清偿时，Spoke 会向 Hub 传入 `drawnAmount` 和 `premiumAmountRay` 等参数，其中 `drawnAmount` 指清偿的正常债务，而 `premiumAmountRay` 代表清偿的风险溢价。`_validateRestore` 的实现也很简单，只是检查 Spoke 清偿的债务是否大于已有的全部债务:

```solidity
function _validateRestore(
  Asset storage asset,
  SpokeData storage spoke,
  uint256 drawnAmount,
  uint256 premiumAmountRay
) internal view {
  require(drawnAmount > 0 || premiumAmountRay > 0, InvalidAmount());
  require(spoke.active, SpokeNotActive());
  require(!spoke.paused, SpokePaused());
  uint256 drawn = _getSpokeDrawn(asset, spoke);
  uint256 premiumRay = _getSpokePremiumRay(asset, spoke);
  require(drawnAmount <= drawn, SurplusDrawnRestored(drawn));
  require(premiumAmountRay <= premiumRay, SurplusPremiumRayRestored(premiumRay));
}

```

而用于检查坏账的 `_validateReportDeficit` 和 `_validateRestore` 代码逻辑是相同的，因为报告坏账本质上也是一种特殊的清偿行为，而 `_validateEliminateDeficit` 则非常简单，只会检查消除坏账的 `amount > 0`。

### Spokes

Spokes 是可升级的，并且是 AAVE v4 内对某种资产进行 lending 和 borrowing 协调的核心组件。Spokes 可以注册到 Hubs 中，并且被允许从 Hubs 内借出流动性。用户与 Spokes 的交互会随后被转化为 Hubs 的交互。Spokes 主要管理以下方面:

- 处理 supply, withdraw, borrow 和 repay 功能
- 管理 reserve 配置，每一个 Spoke 内部都包含不同的 `reserveId`，这些 Id 与 Hub 内部的 `assetId` 不同
- 管理用户的数据和配置
- 管理 Oracle 交互
- 如果需要，提供紧急暂停功能以中止用户操作
- 内部使用 share-based accounting 来简化记账并且确保资产在利息累计时仍处在 Hub 的容量限制内

我们首先介绍 `supply` 函数，该函数是将用户的资产发送给 Hub，该函数的运作原理较为简单，

```solidity
/// @inheritdoc ISpokeBase
function supply(
  uint256 reserveId,
  uint256 amount,
  address onBehalfOf
) external onlyPositionManager(onBehalfOf) returns (uint256, uint256) {
  Reserve storage reserve = _getReserve(reserveId);
  UserPosition storage userPosition = _userPositions[onBehalfOf][reserveId];
  _validateSupply(reserve);

  reserve.underlying.safeTransferFrom(msg.sender, address(reserve.hub), amount);
  uint256 suppliedShares = reserve.hub.add(reserve.assetId, amount);
  userPosition.suppliedShares += suppliedShares.toUint120();

  emit Supply(reserveId, msg.sender, onBehalfOf, suppliedShares, amount);

  return (suppliedShares, amount);
}
```

此处我们需要额外注意 `onBehalfOf` 参数，该参数实际上是 AAVE v4 内 Position manager 使用的。Position manager 是一个代替用户与 Spoke 合约交互的合约，比如 `SignatureGateway` 作为 Position Manager 时允许用户提交 EIP-712 交易实现与 Spoke 的交互。用户可以直接调用 Spoke 内的  `setUserPositionManager` 函数更新 Position Manager 或者使用 `setUserPositionManagerWithSig` 利用签名实现更新 Position Manager 的操作。

与 `supply` 函数相反，Spoke 内还提供 `withdraw` 函数用于用户提取资产，对于大部分只是存入资产的用户，`supply` 和 `withdraw` 就是最常被使用的函数。`withdraw` 函数的核心代码也很简单，如下:

```solidity
uint256 withdrawnAmount = MathUtils.min(
  amount,
  hub.previewRemoveByShares(assetId, userPosition.suppliedShares)
);
uint256 withdrawnShares = hub.remove(assetId, withdrawnAmount, msg.sender);

userPosition.suppliedShares -= withdrawnShares.toUint120();

if (_positionStatus[onBehalfOf].isUsingAsCollateral(reserveId)) {
  uint256 newRiskPremium = _refreshAndValidateUserAccountData(onBehalfOf).riskPremium;
  _notifyRiskPremiumUpdate(onBehalfOf, newRiskPremium);
}
```

由于用户的存款是可以获得利息的，所以此处使用 `MathUtils.min` 函数查找用户可以 withdraw 的最大金额。与 Morpho 等协议一致，AAVE v4 也使用了 share 机制。另外，我们可以看到此处调用了 `_refreshAndValidateUserAccountData` 函数计算新的 riskPremium，我们会在下一节内介绍该机制。简单来说，riskPremium 与用户的担保品构成有关，担保品等级越高那么支付的 Premium 越低，而 `_notifyRiskPremiumUpdate` 会将 `newRiskPremium` 发送给 Hub。`_refreshAndValidateUserAccountData` 内另一个重要的条件判断是:

```solidity
require(
  accountData.healthFactor >= HEALTH_FACTOR_LIQUIDATION_THRESHOLD,
  HealthFactorBelowThreshold()
);
```

该条件判断避免了用户 `withdraw` 后的健康因子问题。可能部分读者并不清楚健康因子的含义，读者可以在 [AAVE 交互指南](https://blog.wssh.dev/posts/aave-interactive/) 一文内了解该概念。

此处可以展开介绍一下 AAVE v4 的 share 机制，Hub 内部的 `Asset` 结构体内包含 `addedShares` 内容，但是我们注意到 `Asset` 结构体内并不包含 `totalAmount` 参数，这是因为 `totalAmount` 需要进行计算，在 Hub 的 `AssetLogic` 库内部，我们可以找到如下代码:

```solidity
function drawn(IHub.Asset storage asset, uint256 drawnIndex) internal view returns (uint256) {
  return asset.drawnShares.rayMulUp(drawnIndex);
}

function totalOwed(IHub.Asset storage asset, uint256 drawnIndex) internal view returns (uint256) {
  return asset.drawn(drawnIndex) + asset.premium(drawnIndex);
}

function totalAddedAssets(IHub.Asset storage asset) internal view returns (uint256) {
  uint256 drawnIndex = asset.getDrawnIndex();
  return
    asset.liquidity +
    asset.swept +
    asset.deficitRay.fromRayUp() +
    asset.totalOwed(drawnIndex) -
    asset.realizedFees -
    asset.getUnrealizedFees(drawnIndex);
}
```

上述代码中 `totalAddedAssets` 是计算最核心的函数，我们可以看到计算过程中，我们计算了 Hub 的总资产与待支付手续费之差，其中 Hub 的总资产包含以下部分:

- 当前系统流动性(`asset.liquidity`)
- 被用于再投资的资产(`asset.swept`)
- 系统坏账(`deficitRay`)，在上文我们已经提到坏账本质上会被视为已经被偿还的债务
- Hub 对外的总欠款(`asset.totalOwed(drawnIndex)`)，为了计入利息，我们需要传入 `drawnIndex` 参数

而 Hub 待支付的资产主要包括 `asset.realizedFees` 已实现手续费，该变量会在 `accrue` 累积利息时被更新，代码如下:

```solidity
asset.realizedFees += asset.getUnrealizedFees(drawnIndex).toUint120();
```

我们可以看到 `asset.realizedFees` 的更新直接调用了 `getUnrealizedFees`。由于这部分利息累计和手续费计算会涉及到 Premium Debt 概念，我们会在后文介绍 Risk Premium 一并介绍。

接下来，我们介绍借贷协议中的另一个核心功能 `borrow` 函数。该函数的核心如下:

```solidity
uint256 drawnShares = hub.draw(reserve.assetId, amount, msg.sender);
userPosition.drawnShares += drawnShares.toUint120();
if (!positionStatus.isBorrowing(reserveId)) {
  positionStatus.setBorrowing(reserveId, true);
}

uint256 newRiskPremium = _refreshAndValidateUserAccountData(onBehalfOf).riskPremium;
_notifyRiskPremiumUpdate(onBehalfOf, newRiskPremium);

emit Borrow(reserveId, msg.sender, onBehalfOf, drawnShares, amount);
```

我们可以看到本质上该函数调用了 hub 的 `draw` 函数直接将 Hub 内的资产发送给用户，然后更新了用户在 Spoke 内的状态，并调用 `_refreshAndValidateUserAccountData` 函数检查 HF 并通知 Hub 更新 Risk Premium。

`repay` 用于偿还债务，但该函数相比于上文介绍的 `borrow` 等函数更加复杂。复杂度主要来自用户的债务分为两部分，一部分是 `drawnDebt` 而另一部分是 Premium Debt，我们需要在合约内计算这部分的债务数量，并且通知 Hub 在用户偿还债务后的 Premium 的情况。

```solidity
IHubBase.PremiumDelta memory premiumDelta = IHubBase.PremiumDelta({
  sharesDelta: -userPosition.premiumShares.toInt256(),
  offsetDeltaRay: -userPosition.premiumOffsetRay.toInt256(),
  accruedPremiumRay: 0, // populated below
  restoredPremiumRay: 0 // populated below
});

uint256 drawnDebtRestored;
uint256 realizedPremiumRay;
(drawnDebtRestored, , realizedPremiumRay, premiumDelta.accruedPremiumRay) = _getUserDebt(
  reserve.hub,
  reserve.assetId,
  userPosition
);

(drawnDebtRestored, premiumDelta.restoredPremiumRay) = _calculateRestoreAmount(
  drawnDebtRestored,
  realizedPremiumRay + premiumDelta.accruedPremiumRay,
  amount
);

uint256 premiumDebtRestored = premiumDelta.restoredPremiumRay.fromRayUp();
reserve.underlying.safeTransferFrom(
  msg.sender,
  address(reserve.hub),
  drawnDebtRestored + premiumDebtRestored
);
uint256 restoredShares = reserve.hub.restore(reserve.assetId, drawnDebtRestored, premiumDelta);

userPosition.applyPremiumDelta(premiumDelta);
userPosition.drawnShares -= restoredShares.toUint120();
if (userPosition.drawnShares == 0) {
  _positionStatus[onBehalfOf].setBorrowing(reserveId, false);
}

uint256 newRiskPremium = _calculateUserAccountData(onBehalfOf).riskPremium;
_notifyRiskPremiumUpdate(onBehalfOf, newRiskPremium);
```

以上代码内出现了大量与 Premium Debt 有关的参数，我们会在后文介绍。此处我们关注一下 `_calculateRestoreAmount` 函数，该函数用于在给定 `drawnDebt` 和 `premiumDebtRay` 债务情况以及用户偿还的 `amount` 基础上计算实际的偿还金额。我们可以代码内看到，`premiumDebt` 的还款等级高于 `drawnDebt`。

```solidity
function _calculateRestoreAmount(
  uint256 drawnDebt,
  uint256 premiumDebtRay,
  uint256 amount
) internal pure returns (uint256, uint256) {
  uint256 premiumDebt = premiumDebtRay.fromRayUp();
  if (amount >= drawnDebt + premiumDebt) {
    return (drawnDebt, premiumDebtRay);
  }

  if (amount < premiumDebt) {
    // amount.toRay() cannot overflow here
    return (0, amount.toRay());
  }

  return (amount - premiumDebt, premiumDebtRay);
}
```

除了上年介绍的 `supply` / `withdraw` / `borrow` / `repay` 外，在 Spoke 内，用户还可以调用 `setUsingAsCollateral` 函数，默认情况下，用户 supply 的资产并不被视为借贷中的担保品，用户需要手动调用 `setUsingAsCollateral` 函数设置，该函数也非常简单:

```solidity
function setUsingAsCollateral(
  uint256 reserveId,
  bool usingAsCollateral,
  address onBehalfOf
) external onlyPositionManager(onBehalfOf) {
  _validateSetUsingAsCollateral(_getReserve(reserveId), usingAsCollateral);
  PositionStatus storage positionStatus = _positionStatus[onBehalfOf];

  if (positionStatus.isUsingAsCollateral(reserveId) == usingAsCollateral) {
    return;
  }
  positionStatus.setUsingAsCollateral(reserveId, usingAsCollateral);

  if (usingAsCollateral) {
    _refreshDynamicConfig(onBehalfOf, reserveId);
  } else {
    uint256 newRiskPremium = _refreshAndValidateUserAccountData(onBehalfOf).riskPremium;
    _notifyRiskPremiumUpdate(onBehalfOf, newRiskPremium);
  }

  emit SetUsingAsCollateral(reserveId, msg.sender, onBehalfOf, usingAsCollateral);
}
```

核心工作就是更新 `positionStatus` 内的 `usingAsCollateral` 字段，以及刷新与担保品有关的动态配置或者 RiskPremium 数据。此处出现了 `DynamicConfig`，这是 AAVE v4 引入的新功能，允许用户使用过去的担保品配置，但是一旦执行可能存在危险的操作时，比如使用刚刚介绍的 `setUsingAsCollateral` 将某一个资产的担保品属性删除，那么就会触发刷新操作，将用户的配置刷新为最新配置。我们会在后文详细介绍该机制。

上文中，我们介绍了 Spoke 与 Hub 的交互，接下来，我们介绍 Spoke 内的数据结构，这些数据结构分为两种:

1. 与 Hub 有关的数据结构
2. 与 User 有关的数据结构

与 Hub 有关的数据结构主要位于 `Reserve` 内部，在 Spoke 的状态变量定义中，我们可以找到如下代码:

```solidity
/// @dev Map of reserve identifiers to their Reserve data.
mapping(uint256 reserveId => Reserve) internal _reserves;
```

其中 `Reserve` 结构体定义如下:

```solidity
struct Reserve {
  address underlying;
  //
  IHubBase hub;
  uint16 assetId;
  uint8 decimals;
  uint24 dynamicConfigKey;
  bool paused;
  bool frozen;
  bool borrowable;
  uint24 collateralRisk;
}
```

此处定义了 `Reserve` 与 Hub 之间的映射关系，以及 `Reserve` 在 Hub 内的 `assetId` 数值。我们需要注意，每一个 Spoke 都在内部维护有自己的 `reserveId`，数值与 Hub 内的 `assetId` 并不同。在 `addReserve` 内部，我们可以看到 `reserveId` 的计算代码 `uint256 reserveId = _reserveCount++;`。

而 `dynamicConfigKey` 用于 AAVE v4 引入的新特性，我们会在后文介绍动态配置时详细分析该字段的作用。`paused` 和 `frozen` 代表资产被暂停，以及被冻结。而 `collateralRisk` 是一个用于 Premium Debt 的参数，我们即将在下一节介绍。

对于 `paused` 和 `frozen` 的不同，我们可以以下代码观察到。简单来说，一旦 reserve 处于 `paused` 状态，那么该资产无法倍执行任何操作，但如果只是处于 `frozen` 状态，那么该资产仍可以被 withdraw，但不能被设置为担保品。

```solidity
function _validateSupply(Reserve storage reserve) internal view {
  require(!reserve.paused, ReservePaused());
  require(!reserve.frozen, ReserveFrozen());
}

function _validateWithdraw(Reserve storage reserve) internal view {
  require(!reserve.paused, ReservePaused());
}

function _validateBorrow(Reserve storage reserve) internal view {
  require(!reserve.paused, ReservePaused());
  require(!reserve.frozen, ReserveFrozen());
  require(reserve.borrowable, ReserveNotBorrowable());
  // health factor is checked at the end of borrow action
}

function _validateRepay(Reserve storage reserve) internal view {
  require(!reserve.paused, ReservePaused());
}

function _validateSetUsingAsCollateral(
  Reserve storage reserve,
  bool usingAsCollateral
) internal view {
  require(!reserve.paused, ReservePaused());
  // can disable as collateral if the reserve is frozen
  require(!usingAsCollateral || !reserve.frozen, ReserveFrozen());
}
```

接下来，我们介绍用户的头寸 `_userPositions` 状态变量，该状态变量维护用户存入的 reserve 数据:

```solidity
/// @dev Map of user addresses and reserve identifiers to user positions.
mapping(address user => mapping(uint256 reserveId => UserPosition)) internal _userPositions;
```

此处的核心是 `UserPosition` 结构体，该结构体定义如下:

```solidity
struct UserPosition {
  uint120 drawnShares;
  uint120 premiumShares;
  //
  uint200 realizedPremiumRay;
  //
  uint200 premiumOffsetRay;
  //
  uint120 suppliedShares;
  uint24 dynamicConfigKey;
}
```

上述结构体中的 `drawnShares` 记录用户借款的 Base Debt 数据，而 `premiumShares` / `realizedPremiumRay` / `premiumOffsetRay` 都是用来记录 Premium Debt 数据。`suppliedShares` 主要记录存款数据，而 `dynamicConfigKey` 也适用于 Dynamic Config。所谓 Dynamic Config 与之最相关的是 Spoke 内的  `_dynamicConfig` 状态变量，该状态变量的定义如下:

```solidity
/// @dev Map of reserve identifiers and dynamic configuration keys to the dynamic configuration data.
mapping(uint256 reserveId => mapping(uint24 dynamicConfigKey => DynamicReserveConfig))
  internal _dynamicConfig;
```

在此处，我们介绍一下 AAVE v4 引入的动态配置的功能。简单来说，目前 AAVE v4 内部创建一个新的配置都会存在 `dynamicConfigKey` 编号。我们可以通过 `_dynamicConfig` 和 `dynamicConfigKey` 找到某一个资产历史上所有的配置。在 上文介绍的 `withdraw` / `borrow` 函数内，我们可以看到 `_refreshAndValidateUserAccountData` 函数，该函数的底层实现最核心代码是:

```solidity
UserAccountData memory accountData = _processUserAccountData(user, true);
```

此处使用了 `_processUserAccountData` 函数，`_processUserAccountData` 的定义如下:

```solidity
function _processUserAccountData(
  address user,
  bool refreshConfig
) internal returns (UserAccountData memory accountData) {
```

此处的 `refreshConfig` 代表是否需要刷新刚刚介绍的 `UserPosition` 内部的 `dynamicConfigKey` 字段，该函数存在很多计算，这些计算其实与 Premium Debt 有关，我们会在后文介绍。此处我们主要关注 `_processUserAccountData` 内的如下代码:

```solidity
uint256 collateralFactor = _dynamicConfig[reserveId][
  refreshConfig
    ? (userPosition.dynamicConfigKey = reserve.dynamicConfigKey)
    : userPosition.dynamicConfigKey
].collateralFactor;
```

我们可以看到假如 `refreshConfig = true`，那么我们会将 `userPosition.dynamicConfigKey` 设置为目前最新的 `reserve.dynamicConfigKey`，反之，我们则使用用户原有的 `userPosition.dynamicConfigKey`。我们可以视为用户在自己头寸内存入了配置的快照，如无必要，我们会直接使用快照的配置。那么到底是什么时候会刷新配置？广义上所有可能导致协议穿仓的操作都需要刷新配置，此处我们列出所有会导致配置刷新的操作:

1. `withdraw` 提取资产，显然使用旧的配置进行资产提取可能导致新配置下协议出现坏账损失
2. `borrow` 借出资产
3. 使用 `setUsingAsCollateral` 撤销某一个 reserve 的担保品地位

除了上述三种方式外，还有两种特殊情况:

1. 用户自己调用 `updateUserDynamicConfig` 函数更新配置，假如新的配置对用户更友好，用户可能调用该函数刷新配置
2. Spoke 管理者调用 `updateDynamicReserveConfig` 函数直接修改 `dynamicConfigKey` 对应的参数，这是一种特殊的操作，正常来说，管理者都应该使用 `addDynamicReserveConfig` 增加配置而不是更新配置，但假如原配置对协议可能存在巨大不利时，管理者还是有权直接修改旧配置，这等同于直接刷新用户的配置

最后，我们介绍一下 `_positionStatus` 状态变量，该状态变量其实存储着用户的资产属性，比如某一个资产是否作为担保品或者某一个资产是否被借出。核心是 `PositionStatus` 结构体与配套的 `src/spoke/libraries/PositionStatusMap.sol` 库，`PositionStatusMap` 提供了一系列核心函数。

```solidity
/// @dev Map of user addresses to their position status.
mapping(address user => PositionStatus) internal _positionStatus;
```

我们首先看一下 `PositionStatus` 的定义:

```solidity
struct PositionStatus {
  mapping(uint256 bucket => uint256) map;
  bool hasPositiveRiskPremium;
}
```

与 AAVE v3 一致，AAVE v4 内资产依旧使用了如下 Bitmap 来标记:

![Asset Bitmap](https://img.gopic.xyz/49cbebfeffedb13722f3fa5049e7c269.png)

这就是 `PositionStatus` 内 `map` 的 `uint256` value 部分。但与 AAVE v3 不同，AAVE v4 支持超过 128 种资产，原理是通过分桶，即 `map` 内的 `bucket` 属性，我们可以通过 `PositionStatusMap.sol` 内的如下函数看到分桶的算法:

```solidity
/// @notice Converts a reserveId to its corresponding bucketId.
function bucketId(uint256 reserveId) internal pure returns (uint256 wordId) {
  assembly ('memory-safe') {
    wordId := shr(7, reserveId)
  }
}
```

上述算法的本质是 `reserveId` 与 128 进行除法，然后去结果即可。`PositionStatusMap` 内配置了很多操作 bitmap 的函数，为了节约篇幅，我们就不在此处进行介绍。而 `PositionStatus` 内的 `hasPositiveRiskPremium` 主要用于 `_notifyRiskPremiumUpdate` 判断是否需要执行算法重新计算 Premiun Debt。Premium Debt 的计算是较为复杂的。

最后，我们介绍 Oracle 功能，在目前的 Spoke 实现中，Oralce 通过 `address public immutable ORACLE;` 定义。这并不是一个可更新的地址变量，Oracle 的实现可以参考 `AaveOracle` 合约，AAVE Oracle 是每一个 Spoke 管理者自己部署的，我们可以通过如下函数看到:

```solidity
/// @inheritdoc IPriceOracle
function getReservePrice(uint256 reserveId) external view returns (uint256) {
  return _getSourcePrice(reserveId);
}
```

限于篇幅，我们就不再介绍 `AaveOracle` 内的其他代码。至此，我们就基本介绍完成了 Spoke 内的所有功能，但在上文中，我们省略了一些机制的详细介绍，比如一直提到的 Premium Debt、利息机制和清算机制，这些都是我们要在后文详细分析的内容。

## Risk Premium

在 AAVE v4 内，我们将债务分为两类:

1. drawn debt 代表基础债务，Spoke 需要支付给 Hub 的借用流动性的基础费用，我们称该利率为 drawn rate
2. Premium Debt 代币风险溢价债务，这部分受用户提供的担保品影响，担保品资产的风险等级决定了用户需要支付多少借款利率溢价。

在本节中，我们将首先介绍 Spoke 内部的 Premium 机制，然后介绍 Hub 内部的处理。这是因为 Premium Debt 是发生在用户侧的参数，从 Spoke 开始介绍较为方便。

### Premium

#### Collateral Risk

Collateral Risk $CR_i$ 取决于资产 $i$ 的质量，精度为 BPS，数据范围为 0 到 1000_00。其中 0 代表最高水平的资产，这种资产可以视为无风险。反之 1000_00 代表最低风险水平的资产，即担保品内风险最大的资产。该参数是 Spoke 的风险参数(risk parameters)，这意味着相同的资产在不同的 spokes 内部会存在不同的 Collateral risk 参数。 在上文中，我们介绍过 `Reserve` 结构体内的 `collateralRisk` 就是用来存储 Collateral risk 参数的。

Spoke 管理员可以使用 `addReserve` 和 `updateReserveConfig` 来创建或者更新 Reserve 对应的 collateral risk 参数，在这些函数调用中，我们会使用 `_validateReserveConfig` 函数检查输入的 collateral risk 参数:

```solidity
/// @dev The maximum allowed collateral risk value for a reserve, expressed in BPS (e.g. 100_00 is 100.00%).
uint24 internal constant MAX_ALLOWED_COLLATERAL_RISK = 1000_00;

function _validateReserveConfig(ReserveConfig calldata config) internal pure {
  require(config.collateralRisk <= MAX_ALLOWED_COLLATERAL_RISK, InvalidCollateralRisk());
}
```

#### User Risk Premium

User Risk Premium $RP_u$ 代表用户 $u$ 借款时提供的担保品集合的质量。该参数取决于多个动态输入:

- 用户 $u$ 在担保品 $i$ 上提供的数量($C_{u,i}$ )，由于 AAVE 会将用户的担保品作为可借出的资产，所以该参数会随着利息的累积而单调递增
- 资产价格($P_i$): 资产价格在连续波动中，某一些资产具有更低的波动率
- Collateral Risk ($CR_i$): Governor 在 Spoke 内配置的风险参数

在 `_processUserAccountData` 函数内，我们会读取这些参数并使用后文介绍的算法计算用户的 User Risk Premium，此处我们主要给出在计算 User Risk Premium 之前的步骤。当然，这些步骤不只是为 User Risk Premium 准备数据，也是为其他环节准备数据。这里的核心是 `UserAccountData` 结构体:

```solidity
struct UserAccountData {
  uint256 riskPremium;
  uint256 avgCollateralFactor;
  uint256 healthFactor;
  uint256 totalCollateralValue;
  uint256 totalDebtValue;
  uint256 activeCollateralCount;
  uint256 borrowedCount;
}
```

在此处，我们会介绍 `_processUserAccountData` 内对 `UserAccountData` 内除 `riskPremium` 字段的计算，这也是 Spoke 的最核心函数。我先分析 `_processUserAccountData` 内部计算前的准备代码:

```solidity
PositionStatus storage positionStatus = _positionStatus[user];

uint256 reserveId = _reserveCount;
KeyValueList.List memory collateralInfo = KeyValueList.init(
  positionStatus.collateralCount(reserveId)
);
bool borrowing;
bool collateral;
while (true) {
  (reserveId, borrowing, collateral) = positionStatus.next(reserveId);
  if (reserveId == PositionStatusMap.NOT_FOUND) break;

  UserPosition storage userPosition = _userPositions[user][reserveId];
  Reserve storage reserve = _reserves[reserveId];

  uint256 assetPrice = IAaveOracle(ORACLE).getReservePrice(reserveId);
  uint256 assetUnit = MathUtils.uncheckedExp(10, reserve.decimals);
```

此处我们注意到 `KeyValueList.List` 类型，这是 AAVE v4 开发团队封装的一个特殊的数组，该数组内的元素是 `key` 和 `value` 的打包版本，我们可以通过如下函数简单了解 `KeyValueList.List` 的工作原理:

```solidity
function add(List memory self, uint256 idx, uint256 key, uint256 value) internal pure {
  require(key < _MAX_KEY && value < _MAX_VALUE, MaxDataSizeExceeded());
  self._inner[idx] = pack(key, value);
}
```

`KeyValueList.List` 类型最大的特点是可以进行 `sortByKey` 排序，我们在后文会介绍计算 Risk Premium 算法时读者就会理解排序的作用。

在完成基本的初始化后，我们进入了一个 `while` 玄幻，该循环的跳出条件时已经遍历了 `positionStatus` 内的所有 bit。简单来说，我们在 `while` 循环内会遍历头寸内所有的担保品和借出资产。我们使用 `positionStatus.next` 在 bitmap 内不断搜索用户设置的担保资产和借出资产。然后，我们会使用 `reserveId` 提取用户的头寸信息 `userPosition` 和担保品的配置信息 `reserve`。最后，我们调用 `ORACLE` 获得资产报价并处理资产精度问题。

完成上述流程后，我们获得了 `reserveId` / `assetPrice` / `assetUnit` / `userPosition` 等信息，接下来，我们首先处理了担保品的信息计算，主要计算 `UserAccountData` 结构体内的 `totalCollateralValue` / `avgCollateralFactor` 和 `activeCollateralCount` 字段。

```solidity
if (collateral) {
  uint256 collateralFactor = _dynamicConfig[reserveId][
    refreshConfig
      ? (userPosition.dynamicConfigKey = reserve.dynamicConfigKey)
      : userPosition.dynamicConfigKey
  ].collateralFactor;
  if (collateralFactor > 0) {
    uint256 suppliedShares = userPosition.suppliedShares;
    if (suppliedShares > 0) {
      // cannot round down to zero
      uint256 userCollateralValue = (reserve.hub.previewRemoveByShares(
        reserve.assetId,
        suppliedShares
      ) * assetPrice).wadDivDown(assetUnit);
      accountData.totalCollateralValue += userCollateralValue;
      collateralInfo.add(
        accountData.activeCollateralCount,
        reserve.collateralRisk,
        userCollateralValue
      );
      accountData.avgCollateralFactor += collateralFactor * userCollateralValue;
      accountData.activeCollateralCount = accountData.activeCollateralCount.uncheckedAdd(1);
    }
  }
}
```

上述代码的第一步时读取 `collateralFactor`，此处涉及到上文介绍的 Dynamic Config，此处就不再详细介绍。然后，会使用读取的 `suppliedShares` 换算为资产数量，在 AAVE v4 内，也使用了借贷协议最常使用的 share 机制，我们在后文介绍利率时会分析 Hub 内的 `previewRemoveByShares` 函数。有趣的是此处的 `avgCollateralFactor` 并不是真实的 factor 数值，此处计算结果是 `collateralFactor * userCollateralValue`。

此处的大部分计算都很简单，我们只需要注意到 `collateralInfo.add` 函数。在上文，我们已经给出了 `add(List memory self, uint256 idx, uint256 key, uint256 value)` 函数定义，此处的 `key` 是 `reserve.collateralRisk` 而 `value` 值是 `userCollateralValue`。在之前的介绍中，我们已经给出了 `add` 函数的实现，此处我们使用 `reserve.collateralRisk` 作为 key。这意味着我们在后期可以调用 `sortByKey` 方法按照 `reserve.collateralRisk` 的大小进行排序。

对于用户已经借出的资产，我们会使用如下代码计算 `accountData.totalDebtValue` 和 `accountData.borrowedCount`，这些代码更加简单。

```solidity
if (borrowing) {
  (uint256 drawnDebt, uint256 premiumDebt, , ) = _getUserDebt(
    reserve.hub,
    reserve.assetId,
    userPosition
  );
  // we can simplify since there is no precision loss due to the division here
  accountData.totalDebtValue += ((drawnDebt + premiumDebt) * assetPrice).wadDivUp(assetUnit);
  accountData.borrowedCount = accountData.borrowedCount.uncheckedAdd(1);
}
```

此处的 `_getUserDebt` 函数内部会涉及到一些关于 Premium Debt 计算的机制，我们会在后文介绍 Premium Offset 时介绍该函数的具体实现，此处我们暂时跳过该函数的解析。在上文，我们已经在 `accountData` 内累计了担保资产和可借资产的数据，我们接下来可以计算 position 的健康因子。此处代码的 `accountData.totalCollateralValue > 0` 分支用来计算真实的 factor。

```solidity
if (accountData.totalDebtValue > 0) {
  // at this point, `avgCollateralFactor` is the collateral-weighted sum (scaled by `collateralFactor` in BPS)
  // health factor uses this directly for simplicity
  // the division by `totalCollateralValue` to compute the weighted average is done later
  accountData.healthFactor = accountData
    .avgCollateralFactor
    .wadDivDown(accountData.totalDebtValue)
    .fromBpsDown();
} else {
  accountData.healthFactor = type(uint256).max;
}

if (accountData.totalCollateralValue > 0) {
  accountData.avgCollateralFactor = accountData
    .avgCollateralFactor
    .wadDivDown(accountData.totalCollateralValue)
    .fromBpsDown();
}
```

理想情况下，User Risk Premium  应该随着市场的波动而被实时修改，但是限于 EVM 区块链特性，我们无法实时更新参数。所以在 AAVE v4 内部，User Risk Premium 只有在用户操作影响到担保品时会更新。另外，用户可以无许可的在自己的头寸上触发更新逻辑(`updateUserRiskPremium`)。

#### Risk Premium Algorithm

Risk Premium 算法会按照以下步骤计算获得用户的 risk premium，同时在此过程中，算法也会输出在当前的头寸债务情况下，用户需要多少保证金。而保证金集合中的 Collateral Risk 的加权平均数就是 User Risk Premium:

1. 按照保证金资产的风险进行升序排序: 按照 Collateral Risk 计算担保品的 risk value，并且按照最低风险到最高风险进行排序，在代码内我们使用 `collateralInfo.sortByKey();` 方法排序
2. 计算当前头寸的 total debt: 计算用户的包含利息的总债务情况，可以直接使用 `accountData.totalDebtValue`
3. 迭代担保品资产来计算可以覆盖用户头寸总债务的担保品资产的数量，在此迭代过程中，我们会使用辅助变量 `debtValueLeftToCover`，该变量会被初始化为 `totalDebtValue`:
   1. 计算 `debtValueLeftToCover = debtValueLeftToCover - userCollateralValue`。此处的 `userCollateralValue` 实际上是上文的代码中计算的
   2. 如果 `debtValueLeftToCover = 0` 意味着所有的债务都被覆盖，所以我们可以直接跳出循环
   3. 如果 `debtValueLeftToCover > 0` > 意味着还有债务没有覆盖，继续循环
4. 基于迭代过程中担保品种类和数量计算担保品的 Collateral Risk 的加权平均数

计算 User Risk Premium 会使用如下公式:
$$
RP_u = f(CR_i, C_{u, i}, P_i) = \frac{\sum_{i=1}^n CR_iC_{u, i}P_i}{\sum_{i=1}^nC_{u, i}P_i}
$$
上述公式内的符号含义如下:

- $CR_i$ 是资产 $i$ 的 Collateral Risk 
- $C_{u, i}$ 是用户 $u$ 提供的资产 $i$ 的数量，但是只有经过以上迭代算法确定用于覆盖用户债务的担保品的数量才可以被用于上述公式计算
- $P_i$ 是资产 $i$ 的价格

最简单的情况是用户的债务只使用一种担保品就可以全部覆盖，那么 $RP_u = f(CR_0, C_{u,0}, P_0) = CR_0$。但在代码中，我们并不是直接使用上述公式，对于分子中的 $CR_iC_{u, i}$ ，可以通过计算 `userCollateralValue * collateralRisk`，而对于分母，其数值等效于用户使用担保品可以覆盖的债务，一般直接等于 `accountData.totalDebtValue`，在用户资不抵债的情况下，我们可以使用 `accountData.totalDebtValue - debtValueLeftToCover` 获得，所以实际上，我们使用如下代码计算 Risk Premium:

```solidity
// sort by collateral risk in ASC, collateral value in DESC
collateralInfo.sortByKey();

// runs until either the collateral or debt is exhausted
uint256 debtValueLeftToCover = accountData.totalDebtValue;

for (uint256 index = 0; index < collateralInfo.length(); ++index) {
  if (debtValueLeftToCover == 0) {
    break;
  }

  (uint256 collateralRisk, uint256 userCollateralValue) = collateralInfo.get(index);
  userCollateralValue = userCollateralValue.min(debtValueLeftToCover);
  accountData.riskPremium += userCollateralValue * collateralRisk;
  debtValueLeftToCover = debtValueLeftToCover.uncheckedSub(userCollateralValue);
}

if (debtValueLeftToCover < accountData.totalDebtValue) {
  accountData.riskPremium /= accountData.totalDebtValue.uncheckedSub(debtValueLeftToCover);
}
```

`accountData.riskPremium` 在 `for` 循环内一直累加 `userCollateralValue * collateralRisk`，最终在离开循环时的数值等同于 $\sum_{i=1}^n CR_iC_{u, i}P_i$。此处可能有读者好奇 $P_i$ 部分什么时候出现的，读者可以自行阅读前文中计算 `userCollateralValue` 的代码。

而分母部分可以简单视为用户的担保品的总价值，一般来说，用户的担保品价值大于债务，即 `debtValueLeftToCover = 0`，此时直接使用 `accountData.totalDebtValue` 即可。但是假如用户目前的 `debtValueLeftToCover > 0`，其实就等同于用户头寸处于资不抵债的情况，此时用户担保品的总价值只等于 `accountData.totalDebtValue - debtValueLeftToCover`。

至此，我们就完成了对 Premium Debt 的计算介绍，此处我们还没有涉及到任何和 Premium 债务利率累计的分析，我们会在下一节内介绍。

#### Premium Offset

在本节中，我们主要介绍 Premium Debt 的利息计算问题。需要注意的是，Premium Debt 的计算与 Drawn rate 息息相关，但本节中，我们将不会介绍 Drawn rate 的利息累计的原理。这部分内容我们会在后文介绍，本文主要介绍 Premium Debt 的利息累计问题。

`RiskPremium` 主要在 `_notifyRiskPremiumUpdate` 发挥核心作用，该函数一方面会更新 `UserPosition` 内的数据。另一方面，也会调用 Hub 的 `refreshPremium` 函数在 Hub 内部进行 risk premium 的调整。在 `_notifyRiskPremiumUpdate` 内部，这两行代码给出了 risk premium 产生利息的方法。

```solidity
uint256 newPremiumShares = userPosition.drawnShares.percentMulUp(newRiskPremium);
uint256 newPremiumOffsetRay = newPremiumShares * drawnIndex;
```

简单来说，User risk premium 只是一个附加利息，我们假设计算出 risk premium = 5%，那么就会在原有的 drawn rate 产生的利息基础上额外增加 5% 的利息，所以本质上利息依旧是在 `drawnShares` 内累计。为了理解 risk premium 的利息累计，我们就需要对 draw share 的工作原理进行介绍，此处我们只介绍 draw debt 利息的数学原理。

当用户第一次 supply 或者 borrow 资产时，我们会使用 `amount / drawIndex` 的方法计算出 `drawShares`。当用户最终 withdraw 或者 repay 时，我们会计算 `drawShares * drawIndex` 获得提取资产或者还款的金额。而 `drawIndex` 是一个自协议产生以来一直累计利息的变量，我们可以认为 `drawIndex` 代表一单位资产从协议部署就存入到现在的价值。对于金融背景的朋友，可以将其理解为一个贴现因子，本质上是用户 supply 或 borrow 时，资产会被贴现到协议部署时，而用户 withdraw 或者 repay 时，我们会按照最新的贴现因子计算出这部分资产的价值。

基于上述知识，我们会发现 `newPremiumShares` 作为 `drawShares` 的部分，假如只使用该数值，意味着 risk premium 是对协议部署以来整个历史时期征收手续费，这显然不合理，因为用户大概率不是在协议部署时就存入或借出的资产。此时，我们需要一个修正因子，该因子可以去掉协议部署到 risk premium 计算之间的历史时期的利息，其实就是以上代码中的 `newPremiumOffsetRay`。我们只需要使用 `newPremiumShares` 与最新的 `drawIndex` 相乘，然后减去 `premiumOffsetRay` 就可以计算出正确的利息。在 `src/hub/libraries/Premium.sol` 文件内，我们可以看到如下计算 Premium debt 的函数:

```solidity
function calculateAccruedPremiumRay(
  uint256 premiumShares,
  uint256 drawnIndex,
  uint256 premiumOffsetRay
) internal pure returns (uint256) {
  return premiumShares * drawnIndex - premiumOffsetRay;
}
```

但是在 `UserPosition` 结构体内部，我们可以看到除了 `premiumShares` 和 `premiumOffsetRay` 外的 `realizedPremiumRay` 参数。我们可以在 `_notifyRiskPremiumUpdate` 看到对这些变量的修改或者初始化:

```solidity
uint256 oldPremiumShares = userPosition.premiumShares;
uint256 oldPremiumOffsetRay = userPosition.premiumOffsetRay;
uint256 accruedPremiumRay = Premium.calculateAccruedPremiumRay({
  premiumShares: oldPremiumShares,
  drawnIndex: drawnIndex,
  premiumOffsetRay: oldPremiumOffsetRay
});

uint256 newPremiumShares = userPosition.drawnShares.percentMulUp(newRiskPremium);
uint256 newPremiumOffsetRay = newPremiumShares * drawnIndex;

userPosition.premiumShares = newPremiumShares.toUint120();
userPosition.premiumOffsetRay = newPremiumOffsetRay.toUint200();
userPosition.realizedPremiumRay = (userPosition.realizedPremiumRay + accruedPremiumRay)
  .toUint200();
```

简单来说，我们每次调用 `_notifyRiskPremiumUpdate` 都会使用 `calculateAccruedPremiumRay` 函数计算出最新的 `accruedPremiumRay` 然后将其累积到 `realizedPremiumRay` 参数的。`calculateAccruedPremiumRay` 的实现也非常简单:

```solidity
function calculateAccruedPremiumRay(
  uint256 premiumShares,
  uint256 drawnIndex,
  uint256 premiumOffsetRay
) internal pure returns (uint256) {
  return premiumShares * drawnIndex - premiumOffsetRay;
}
```

可能有读者好奇为什么每次更新都需要计算利息并累计到 `realizedPremiumRay` 内部? 这是因为 `_notifyRiskPremiumUpdate` 会在 `withdraw` / `borrow` /  `repay` /  `liquidationCall` 四个函数被使用到，这些函数会影响到 debt shares 或者 RiskPremium 变量，每次影响到这些变量时，我们需要将已经产生的 Premium debt 债务累积到 `realizedPremiumRay`，然后更新 `premiumShares` 和 `premiumOffsetRay` 准备累计未来的利息。

最后，我们分析 `_notifyRiskPremiumUpdate` 内的最后一段代码，这部分代码会调用 Hub 的 `refreshPremium` 函数更新 Hub 内的 Premium 数据。此处我们看到使用了 `signedSub`，这是因为 PremiumDelta 允许 Spoke 传入正数或者负数进行调整。

```solidity
IHubBase.PremiumDelta memory premiumDelta = IHubBase.PremiumDelta({
  sharesDelta: newPremiumShares.signedSub(oldPremiumShares),
  offsetDeltaRay: newPremiumOffsetRay.signedSub(oldPremiumOffsetRay),
  accruedPremiumRay: accruedPremiumRay,
  restoredPremiumRay: 0
});

hub.refreshPremium(assetId, premiumDelta);
emit RefreshPremiumDebt(reserveId, user, premiumDelta);
```

在 Spoke 内部，大部分函数都是直接调用 `_notifyRiskPremiumUpdate`，但是特例是 `repay` 函数，该函数会调用 Hub 函数的 `restore` 函数，该函数在 Hub 内部定义如下:

```solidity
function restore(
  uint256 assetId,
  uint256 drawnAmount,
  PremiumDelta calldata premiumDelta
) external returns (uint256) {
```

所以此处 Spoke 内的 `repay` 函数在调用 `restore` 时需要带有 `premiumDelta` 参数，所以 `repay` 函数会计算 premium 情况。以下代码显示了部分的 `premiumDelta` 的计算过程:

```solidity
IHubBase.PremiumDelta memory premiumDelta = IHubBase.PremiumDelta({
  sharesDelta: -userPosition.premiumShares.toInt256(),
  offsetDeltaRay: -userPosition.premiumOffsetRay.toInt256(),
  accruedPremiumRay: 0, // populated below
  restoredPremiumRay: 0 // populated below
});

uint256 drawnDebtRestored;
uint256 realizedPremiumRay;
(drawnDebtRestored, , realizedPremiumRay, premiumDelta.accruedPremiumRay) = _getUserDebt(
  reserve.hub,
  reserve.assetId,
  userPosition
);

(drawnDebtRestored, premiumDelta.restoredPremiumRay) = _calculateRestoreAmount(
  drawnDebtRestored,
  realizedPremiumRay + premiumDelta.accruedPremiumRay,
  amount
);

uint256 premiumDebtRestored = premiumDelta.restoredPremiumRay.fromRayUp();
```

`PremiumDelta` 结构体内的字段基本都在上文出现过，所以此处我们就不再介绍每一个字段的含义。而 `_getUserDebt` 和 `_calculateRestoreAmount` 都应已经在上文进行了介绍。此处在设置 `premiumDelta` 时，我们将 `sharesDelta` 和 `offsetDeltaRay` 都设置为当前存储变量的相反数，这会清空 Hub 和 Spoke 内对当前用户头寸的 `premiumShares`  和 `premiumOffsetRay` 数值，这是因为偿还债务时，由于影响了债务数量，我们本身就需要重新计算这两个数值。在 `repay` 函数的后期，我们会使用 `_notifyRiskPremiumUpdate` 再次将用户头寸内的 Premium 数据更新。

上述代码的核心作用是:

1.  读取内部数据补充 `premiumDelta` 内的缺失的 `accruedPremiumRay` 和 `restoredPremiumRay` 数据
2. 计算出用户偿还的 `drawnDebtRestored` 和 `premiumDebtRestored` 的资金数量

上述代码很难阅读的原因是为了避免变量数量超过限制，所以 AAVE v4 在很多函数上都使用了单个变量重复使用的情况，换言之， `drawnDebtRestored` 在代码内的不同位置具有不同的含义，这其实并不是一种很好的行为，但是迫于 solidity 限制，开发者不得不进行这种反模式操作。

在后续代码内，我们会进行了代币转移，然后调用了 `hub` 的 `restore` 函数处理 `drawnDebtRestored` 和 `premiumDelta`，即通知 Hub 以方便 Hub 在其合约内部处理 drawn debt 或者 premium debt。然后，Hub 会返回用户到底偿还的 debt share 数值。最后，Spoke 调用 `applyPremiumDelta` 处理 Premium 的修改和 `drawnShares` 的修改。

```solidity
reserve.underlying.safeTransferFrom(
  msg.sender,
  address(reserve.hub),
  drawnDebtRestored + premiumDebtRestored
);
uint256 restoredShares = reserve.hub.restore(reserve.assetId, drawnDebtRestored, premiumDelta);

userPosition.applyPremiumDelta(premiumDelta);
userPosition.drawnShares -= restoredShares.toUint120();
if (userPosition.drawnShares == 0) {
  _positionStatus[onBehalfOf].setBorrowing(reserveId, false);
}
```

在 `repay` 函数的最后，我们会重新计算 risk premium 后再次调用 `_notifyRiskPremiumUpdate` 更新。

```solidity
uint256 newRiskPremium = _calculateUserAccountData(onBehalfOf).riskPremium;
_notifyRiskPremiumUpdate(onBehalfOf, newRiskPremium);
```

在最后，我们介绍 Spoke 内最特殊的会发送给 Hub 进行 preimun shares 更新的函数。我们在上文一直提到当我们出现了穿仓情况，穿仓的清理等同于还款。所以 Hub 内的 `reportDeficit` 内也允许 Spoke 传入 `premiumDelta` 参数。我们可以在 Spoke 内的 `_reportDeficit` 函数内看到具体计算逻辑:

```solidity
UserPosition storage userPosition = _userPositions[user][reserveId];
Reserve storage reserve = _reserves[reserveId];
IHubBase hub = reserve.hub;
uint256 assetId = reserve.assetId;
(
  uint256 drawnDebtReported,
  ,
  uint256 realizedPremiumRay,
  uint256 accruedPremiumRay
) = _getUserDebt(hub, assetId, userPosition);

IHubBase.PremiumDelta memory premiumDelta = IHubBase.PremiumDelta({
  sharesDelta: -userPosition.premiumShares.toInt256(),
  offsetDeltaRay: -userPosition.premiumOffsetRay.toInt256(),
  accruedPremiumRay: accruedPremiumRay,
  restoredPremiumRay: realizedPremiumRay + accruedPremiumRay
});
uint256 deficitShares = hub.reportDeficit(assetId, drawnDebtReported, premiumDelta);
userPosition.applyPremiumDelta(premiumDelta);
userPosition.drawnShares -= deficitShares.toUint120();
positionStatus.setBorrowing(reserveId, false);

emit ReportDeficit(reserveId, user, deficitShares, premiumDelta);
```

由于报告坏账等同于直接清空所有债务，所以此处我们直接将 `sharesDelta` 和 `restoredPremiumRay` 全部设置为当前的所有 draw debt 和 premium debt。另外，我们可以发现由于所有的债务都已经被清理，所以也不需要像 `repay` 函数调用 `_notifyRiskPremiumUpdate` 函数再次更新。

#### Hub Premium

在 AAVE v4 内部，Hub 会在自己的状态内存储一套完全类似 Spoke 的 Premium 的存储，我们在上文已经展示过 `SpokeData` 内的字段，此处我们再次展示 `SpokeData` 结构体的构成:

```solidity
struct SpokeData {
  uint120 drawnShares;
  uint120 premiumShares;
  //
  uint200 premiumOffsetRay;
  //
  uint200 realizedPremiumRay;
  //
  uint120 addedShares;
  uint40 addCap;
  uint40 drawCap;
  uint24 riskPremiumThreshold;
  bool active;
  bool paused;
  //
  uint200 deficitRay;
}
```

我们可以看到 SpokeData 记录了 `premiumShares` / `premiumOffsetRay` 和 `realizedPremiumRay`，这本质上是对 Spoke 报送数据的汇总。而 Hub 不仅会在 Spoke 内部进行数据汇总，也会在 Asset 内部进行数据汇总，比如 `Asset` 的结构体，比如以下字段:

```solidity
uint200 realizedPremiumRay;
//
uint200 premiumOffsetRay;
//
uint120 drawnShares;
uint120 premiumShares;
```

接下来，我们就会介绍 Hub 内如何利用 Spoke 内通过 `restore` / `reportDeficit`  / `refreshPremium` 三个函数通过 `PremiumDelta` 结构体的内容更新内部的数据。我们首先分析最常被使用 `refreshPremium` 函数，该函数的实现如下:

```solidity
/// @inheritdoc IHubBase
function refreshPremium(uint256 assetId, PremiumDelta calldata premiumDelta) external {
  Asset storage asset = _assets[assetId];
  SpokeData storage spoke = _spokes[assetId][msg.sender];

  asset.accrue();
  require(spoke.active, SpokeNotActive());
  // no premium change allowed
  require(premiumDelta.restoredPremiumRay == 0, InvalidPremiumChange());
  _applyPremiumDelta(asset, spoke, premiumDelta);
  asset.updateDrawnRate(assetId);

  emit RefreshPremium(assetId, msg.sender, premiumDelta);
}
```

此处的 `asset.accrue();` 是用来累计利息的，我们会在后文介绍该部分内容。此处我们可以看到 `premiumDelta.restoredPremiumRay == 0` 避免 Spoke 在 `refreshPremium` 过程内减少用户的 Premium debt 数量。接下来的 `_applyPremiumDelta` 函数是一个重点函数，该函数负责将 `premiumDelta` 内的数据写入到状态内。

`_applyPremiumDelta` 函数核心主要被分为三个部分，分别用来检查并写入 `asset` 级别的 Premiun、检查并写入 `spoke` 级别的 Premium 以及进行不变量检查。对于 `aseet` 级别的数据写入，我们可以在 `_applyPremiumDelta` 内看到如下代码:

```solidity
// asset premium change
(
  asset.premiumShares,
  asset.premiumOffsetRay,
  asset.realizedPremiumRay
) = _validateApplyPremiumDelta(
  drawnIndex,
  asset.premiumShares,
  asset.premiumOffsetRay,
  asset.realizedPremiumRay,
  premiumDelta
);
```

该函数的核心是 `_validateApplyPremiumDelta` 函数，有关代码如下:

```solidity
uint256 premiumRayBefore = Premium.calculatePremiumRay({
  premiumShares: premiumShares,
  drawnIndex: drawnIndex,
  premiumOffsetRay: premiumOffsetRay,
  realizedPremiumRay: realizedPremiumRay
});

uint256 newPremiumShares = premiumShares.add(premiumDelta.sharesDelta);
uint256 newPremiumOffsetRay = premiumOffsetRay.add(premiumDelta.offsetDeltaRay);
uint256 newRealizedPremiumRay = realizedPremiumRay +
  premiumDelta.accruedPremiumRay -
  premiumDelta.restoredPremiumRay;

uint256 premiumRayAfter = Premium.calculatePremiumRay({
  premiumShares: newPremiumShares,
  drawnIndex: drawnIndex,
  premiumOffsetRay: newPremiumOffsetRay,
  realizedPremiumRay: newRealizedPremiumRay
});

require(
  premiumRayAfter + premiumDelta.restoredPremiumRay == premiumRayBefore,
  InvalidPremiumChange()
);
return (
  newPremiumShares.toUint120(),
  newPremiumOffsetRay.toUint200(),
  newRealizedPremiumRay.toUint200()
);
```

上述代码是 Hub 使用 `premiumDelta` 传入的数据重新计算了 `premiumRay`。上述代码内最难理解是是最后的不变量检查，我们需要在 Hub 内检查 Spoke 内的计算是否正确。我们在上文介绍过 Premium 本质上是在 drawn debt 基础上征收的额外利息，所以此处存在一个不变量，即 draw index 不变的情况下，利用当前参数计算出的 Premiun 总量是一致的。当然，额外的情况是假如存在还款行为(restore) 的情况下，premiun 会减少。

在以上代码内，我们先使用当前 Spoke 内存储的数据计算 Premiun，然后使用 Spoke 发送的 `premiumDelta` 再次计算。最后我们检查 `premiumRayAfter + premiumDelta.restoredPremiumRay == premiumRayBefore`。

继续回到 `_applyPremiumDelta` 函数，在完成 asset 级别的计算后，我们会对 spoke 级别调用 `_validateApplyPremiumDelta` 进行计算，限于篇幅，此处不再介绍。在 `_applyPremiumDelta` 的最后，我们会判断当前的 Premiun 风险是否过高:

```solidity
uint24 riskPremiumThreshold = spoke.riskPremiumThreshold;
require(
  riskPremiumThreshold == MAX_RISK_PREMIUM_THRESHOLD ||
    spoke.premiumShares <= spoke.drawnShares.percentMulUp(riskPremiumThreshold),
  InvalidPremiumChange()
);
```

`refreshPremium` 函数的最核心部分的 `_applyPremiumDelta`已经介绍完成了，剩下的代码部分包含 `asset.updateDrawnRate(assetId);` 代码，该代码用于更新资产的利率。当然，更新方法也非常简单，只是将系统内的数据发送给 `asset.irStrategy` 合约请求最新的 drawn rate:

```solidity
function updateDrawnRate(IHub.Asset storage asset, uint256 assetId) internal {
  uint256 drawnIndex = asset.drawnIndex;
  uint256 newDrawnRate = IBasicInterestRateStrategy(asset.irStrategy).calculateInterestRate({
    assetId: assetId,
    liquidity: asset.liquidity,
    drawn: asset.drawn(drawnIndex),
    deficit: asset.deficitRay.fromRayUp(),
    swept: asset.swept
  });
  asset.drawnRate = newDrawnRate.toUint96();

  emit IHub.UpdateAsset(assetId, drawnIndex, newDrawnRate, asset.realizedFees);
}

```

在 `restore` 函数内部，对于 Premium 的更新也是核心调用了 `_applyPremiumDelta(asset, spoke, premiumDelta);` 函数，并没有特殊的部分。在处理完成 Premium 数据更新后，`restore` 之后的操作非常简单，主要是检查资金转移情况并更新流动性:

```solidity
uint256 premiumAmount = premiumDelta.restoredPremiumRay.fromRayUp();
uint256 liquidity = asset.liquidity + drawnAmount + premiumAmount;
uint256 balance = asset.underlying.balanceOf(address(this));
require(balance >= liquidity, InsufficientTransferred(liquidity.uncheckedSub(balance)));
asset.liquidity = liquidity.toUint120();
```

`reportDeficit` 的最核心代码依旧是 `_applyPremiumDelta` 函数。除此之外，`reportDeficit` 函数的核心作用是更新内部的 `deficitRay` 变量，记录坏账情况:

```solidity
uint256 deficitAmountRay = uint256(drawnShares) *
  asset.drawnIndex +
  premiumDelta.restoredPremiumRay;
asset.deficitRay += deficitAmountRay.toUint200();
spoke.deficitRay += deficitAmountRay.toUint200();
```

## Interest Accrual

每一个借贷仓位的利息可以被分为同时发生的两部分：

1. *drawn debt* 用户借出资产的正常利息累计，利率取决于用户借出资产在 Hub 内的利用率，我们使用 $R_{sbase,i}$ 表示资产 $i$ 在基础利率(base reate)
2. *premium debt* 是用户为担保品集合额外支付的利息，具体利率取决于用户的担保品情况，可以使用 $R_{sbase,i}RP_u$ 计算，其中 $RP_u$ 是用户 $u$ 的 Risk Premium ，我们已经在上文介绍过相关算法和代码

用户 $u$ 在借出资产 $i$ 债务 $D_{u, i}$ 的预期总利息的增加等同于 base debt 和 premium debt 之和:
$$
D_{u,i} = D_{u,ibase} + D_{u,ipremium}
$$
其中， $D_{u,ibase}$ 是用户 $u$ 在资产 $i$ 的 base debt 数量，而 $D_{u,ipremium}$ 是用户 $u$ 在资产 $i$ 的 premium debt 数量。上述所有描述都是内部的，与用户无关。用户只会简单的看到债务以更高的利率  $R_{u,i}$ 增加。

在代码实现中，我们会发现 Spoke 不处理利息问题，比如在 Spoke 的 `withdraw` 函数内，我们可以看到如下代码:

```solidity
uint256 withdrawnAmount = MathUtils.min(
  amount,
  hub.previewRemoveByShares(assetId, userPosition.suppliedShares)
);
uint256 withdrawnShares = hub.remove(assetId, withdrawnAmount, msg.sender);
```

本质上，`withdrawnAmount` 数量是由 Hub 决定的，所有的利息计算都会在 Hub 内进行。最核心的部分代码位于 `AssetLogic` 库内部，我们本节中介绍的代码基本都来自该库。

但除了利息外，本节将介绍的另一个内容是手续费问题，在 AAVE v4 内利息部分会被扣除手续费，这部分手续费可以使用 `_mintFeeShares` 分配给 Hub 内的 `asset.feeReceiver`。

### Base Debt

Base Debt 是用户未偿还的债务头寸的核心部分，这部分与 Hub 内提取的流动性有关。当用户在 Spoke $s$ 借出资产 $i$，系统会记录这笔借出的数量作为用户的 base debt。我们使用 $D_{u,ibase}$ 记录用户 $u$ 借出的资产 $i$ 的数量。这代表 Spoke 代表用户在 Hub 内借出的流动性。

在用户借款的时刻，用户的 base debt 就等于借出资产的数量。在 Spoke 内部，我们会直接使用 Hub 的 `draw` 函数返回的 share 值作为用户的债务情况，代码如下:

```solidity
uint256 drawnShares = hub.draw(reserve.assetId, amount, msg.sender);
userPosition.drawnShares += drawnShares.toUint120();
```

那么，我们可以分析 Hub 内的 `draw` 函数的实现:

```solidity
asset.accrue();
_validateDraw(asset, spoke, amount, to);

uint256 liquidity = asset.liquidity;
require(amount <= liquidity, InsufficientLiquidity(liquidity));

uint120 drawnShares = asset.toDrawnSharesUp(amount).toUint120();
asset.drawnShares += drawnShares;
spoke.drawnShares += drawnShares;
```

此处，我们可以看到 `asset` 的两个操作，分别是 `accrue` 和 `toDrawnSharesUp`。我们首先介绍 `toDrawnSharesUp` 的实现。`toDrawnSharesUp` 函数的核心内容就是利用 drawn index 对用户的 `amount` 进行折现处理，所以我们可以看到下文给出的代码中，我们利用 `rayDivUp` 计算了资产数量对应的 shares 数值。

```solidity
/// @notice Calculates the drawn index of a specified asset based on the existing drawn rate and index.
function getDrawnIndex(IHub.Asset storage asset) internal view returns (uint256) {
  uint256 previousIndex = asset.drawnIndex;
  uint40 lastUpdateTimestamp = asset.lastUpdateTimestamp;
  if (
    lastUpdateTimestamp == block.timestamp || (asset.drawnShares == 0 && asset.premiumShares == 0)
  ) {
    return previousIndex;
  }
  return
    previousIndex.rayMulUp(
      MathUtils.calculateLinearInterest(asset.drawnRate, lastUpdateTimestamp)
    );
}

/// @notice Converts an amount of drawn assets to the equivalent amount of shares, rounding up.
function toDrawnSharesUp(
  IHub.Asset storage asset,
  uint256 assets
) internal view returns (uint256) {
  return assets.rayDivUp(asset.getDrawnIndex());
}
```

在上文中，我们可以看到 `getDrawnIndex` 在函数内部进行了 drawn index 的更新计算，我们会使用 `drawnRate` 和上次更新时间使用线性方法计算利息。此处的 `calculateLinearInterest` 实现如下:

```solidity
function calculateLinearInterest(
  uint96 rate,
  uint40 lastUpdateTimestamp
) internal view returns (uint256 result) {
  assembly ('memory-safe') {
    if gt(lastUpdateTimestamp, timestamp()) {
      revert(0, 0)
    }
    result := sub(timestamp(), lastUpdateTimestamp)
    result := add(div(mul(rate, result), SECONDS_PER_YEAR), RAY)
  }
}
```

相比于 AAVE v3 而言，AAVE v4 并没有使用之前的连续复利计算方法，因为连续复利计算需要进行复杂的指数计算，为了简化利率计算，AAVE v4 直接使用了单利计算。从上文的代码中，我们可以看到 `rate` 本身属于年利率。

在有了上述基础后，我们可以介绍 `accrue` 函数，该函数的实现其实很简单:

```solidity
/// @notice Accrues interest and fees for the specified asset.
function accrue(IHub.Asset storage asset) internal {
  if (asset.lastUpdateTimestamp == block.timestamp) {
    return;
  }

  uint256 drawnIndex = asset.getDrawnIndex();
  asset.realizedFees += asset.getUnrealizedFees(drawnIndex).toUint120();
  asset.drawnIndex = drawnIndex.toUint120();
  asset.lastUpdateTimestamp = block.timestamp.toUint40();
}
```

相比于上文直接调用 `getDrawnIndex` 但并不将其写入状态，此处的 `accrue` 函数会将最新计算获得的 `drawnIndex` 写入状态变量。此处使用的 `getUnrealizedFees` 用于计算手续费，这些手续费与债务的利息累计有关，在计算过程中，我们会计算 draw debt 与 premium debt 的利息增加情况，然后使用 `asset.liquidityFee` 计算出利息应该支付手续费的部分。

```solidity
function getUnrealizedFees(
  IHub.Asset storage asset,
  uint256 drawnIndex
) internal view returns (uint256) {
  uint256 previousIndex = asset.drawnIndex;
  if (previousIndex == drawnIndex) {
    return 0;
  }

  uint256 liquidityFee = asset.liquidityFee;
  if (liquidityFee == 0) {
    return 0;
  }

  uint120 drawnShares = asset.drawnShares;
  uint256 liquidityGrowthDrawn = drawnShares.rayMulUp(drawnIndex) -
    drawnShares.rayMulUp(previousIndex);

  uint256 realizedPremiumRay = asset.realizedPremiumRay;
  uint120 premiumShares = asset.premiumShares;
  uint256 premiumOffsetRay = asset.premiumOffsetRay;
  uint256 premiumRayAfter = Premium.calculatePremiumRay({
    premiumShares: premiumShares,
    drawnIndex: drawnIndex,
    premiumOffsetRay: premiumOffsetRay,
    realizedPremiumRay: realizedPremiumRay
  });
  uint256 premiumRayBefore = Premium.calculatePremiumRay({
    premiumShares: premiumShares,
    drawnIndex: previousIndex,
    premiumOffsetRay: premiumOffsetRay,
    realizedPremiumRay: realizedPremiumRay
  });
  uint256 liquidityGrowthPremium = premiumRayAfter.fromRayUp() - premiumRayBefore.fromRayUp();

  return (liquidityGrowthDrawn + liquidityGrowthPremium).percentMulDown(liquidityFee);
}
```

上述代码看似复杂，但逻辑其实并不复杂，首先计算出 draw debt 的利息情况 `liquidityGrowthDrawn`，然后使用上文内已经介绍过的 `calculatePremiumRay` 方法计算 Premium debt 随着 draw index 增加的 `liquidityGrowthPremium`。在最后，我们使用 `percentMulDown` 计算出在当前 `liquidityFee` 数值下，利息增加应该支付的手续费。

那么一定有读者好奇，这部分手续费到底是谁会支付？在上文中，我们介绍了 `totalAddedAssets` 函数，该函数的实现如下:

```solidity
function totalAddedAssets(IHub.Asset storage asset) internal view returns (uint256) {
  uint256 drawnIndex = asset.getDrawnIndex();
  return
    asset.liquidity +
    asset.swept +
    asset.deficitRay.fromRayUp() +
    asset.totalOwed(drawnIndex) -
    asset.realizedFees -
    asset.getUnrealizedFees(drawnIndex);
}
```

而 `toAddedAssetsUp` 或 `toAddedSharesUp` 等函数都依赖 `totalAddedAssets` 报告当前的流动性情况，在 `add` 函数以及 `remove` 函数内，我们会使用 `toAddedSharesDown` 计算用户存款或提款时获得的 shares 数量，所以手续费本质上是由存款人承担的。

特殊的，在 `eliminateDeficit` 函数内，我们也会使用 `asset.toAddedSharesUp(deficitAmountRay.fromRayUp())` 计算偿还坏账金额对应的 shares 数量。在后文介绍 AAVE v4 内的清算机制时，我们会再次阅读该函数。

对于这些累积的手续费，我们可以使用 `_mintFeeShares` 为 `feeReceiver` 进行手续费对应的 shares 的铸造，相关代码如下:

```solidity
uint256 fees = asset.realizedFees;
uint120 shares = asset.toAddedSharesDown(fees).toUint120();
if (shares == 0) {
  return 0;
}

address feeReceiver = asset.feeReceiver;
SpokeData storage feeReceiverSpoke = _spokes[assetId][feeReceiver];
require(feeReceiverSpoke.active, SpokeNotActive());

asset.addedShares += shares;
feeReceiverSpoke.addedShares += shares;
asset.realizedFees = 0;
emit MintFeeShares(assetId, feeReceiver, shares, fees);
```

这是一个相对简单的函数，本质上就是将 `realizedFees` 转移到 `feeReceiverSpoke` 内部以及转移到 `asset.addedShares` 内部。此处需要额外注意的是 `feeReceiver` 的数据被保存在 `assets` 配置内部。

其实在 Hub 内，还存在另一个有趣的函数 `payFeeShares`，该函数也是一个用于在 Hub 之间进行 fee 划转的系统，但该函数与上文介绍的 `_mintFeeShares` 函数不同，该函数是直接将 `sender` 的 shares 直接转移给 `feeReceiver`。该函数被用于清算过程，在清算时，清算过程会产生清算费用(可以在 Spoke 使用 `liquidationFee` 进行定义)，这部分清算费用会直接调用 `payFeeShares` 将清算费用进行划转。关于该函数的具体使用，我们会在下文进行详细介绍。

```solidity
/// @inheritdoc IHubBase
function payFeeShares(uint256 assetId, uint256 shares) external {
  Asset storage asset = _assets[assetId];
  address feeReceiver = _assets[assetId].feeReceiver;
  SpokeData storage receiver = _spokes[assetId][feeReceiver];
  SpokeData storage sender = _spokes[assetId][msg.sender];

  asset.accrue();
  _validatePayFeeShares(sender, shares);
  _transferShares(sender, receiver, shares);
  asset.updateDrawnRate(assetId);

  emit TransferShares(assetId, msg.sender, feeReceiver, shares);
}
```

在 Hub 内，还存在 `transferShares` 函数，该函数可以简单粗暴的将作为 `msg.sender` 的 Spoke 的 shares 直接转移 `toSpoke` 接收方。该函数实现很简单，并且目前 Spoke 内并不存在对该函数的调用。这个函数目前在 AAVE v4 系统内的作用暂不明确。

### Premium Debt

Premium Debt  代表用户债务的一部分，这部分债务代表由于用户的担保品的质量而额外累积的利息。$D_{u,premium}$ 是额外的用户 $u$ 的额外的利息累计。与 base debt 不同，premium debt 不源自 Hub 实际借出的资产，只是一个记账条目记录由于 User Risk Premium 的额外债务。Premium debt 是一个完全依靠 drawn debt 的债务，我们在上文介绍的 `_applyPremiumDelta` 函数完成了 Premium debt 的累计。

## Liquidation Engine

AAVE v4 修改了 AAVE v3 内部的基于固定 close-factor 的清算机制。在 AAVE v3 内部，清算者可清算的金额取决于:

1. 假如健康因子大于 0.95 且担保品和债务价值至少为 2000 美元(对应代码内的 `MIN_BASE_MAX_CLOSE_FACTOR_THRESHOLD` 参数)时，清算者最多清算 50% (我们称该比例为 *close factor*)总债务
2. 假如健康因子在 0.95 以下 或 担保品或债务低于 2000 美元时，最多可清算 100% 债务

但是上述清算机制一直苦于粉尘头寸问题，所谓的粉尘头寸是指清算后由于可能存在一些零星的未被清算的债务，这些粉尘债务随着时间推移会逐渐累积，而清算这些粉尘债务的成本大于收益，AAVE v3 在 [BGD. Aave v3.3 (feat Umbrella)](https://governance.aave.com/t/bgd-aave-v3-3-feat-umbrella/20129) 内就引入了一些机制解决该问题，我们可以在 [aave-v3-origin](https://github.com/aave-dao/aave-v3-origin) 的 [LiquidationLogic.sol](https://github.com/aave-dao/aave-v3-origin/blob/main/src/contracts/protocol/libraries/logic/LiquidationLogic.sol) 内看到如下代码:

```solidity
// to prevent accumulation of dust on the protocol, it is enforced that you either
// 1. liquidate all debt
// 2. liquidate all collateral
// 3. leave more than MIN_LEFTOVER_BASE of collateral & debt
if (
  vars.actualDebtToLiquidate < vars.borrowerReserveDebt &&
  vars.actualCollateralToLiquidate + vars.liquidationProtocolFeeAmount <
  vars.borrowerCollateralBalance
) {
  bool isDebtMoreThanLeftoverThreshold = MathUtils.mulDivCeil(
    vars.borrowerReserveDebt - vars.actualDebtToLiquidate,
    vars.debtAssetPrice,
    vars.debtAssetUnit
  ) >= MIN_LEFTOVER_BASE;

  // @note floor rounding
  bool isCollateralMoreThanLeftoverThreshold = ((vars.borrowerCollateralBalance -
    vars.actualCollateralToLiquidate -
    vars.liquidationProtocolFeeAmount) * vars.collateralAssetPrice) /
    vars.collateralAssetUnit >=
    MIN_LEFTOVER_BASE;

  require(
    isDebtMoreThanLeftoverThreshold && isCollateralMoreThanLeftoverThreshold,
    Errors.MustNotLeaveDust()
  );
}
```

上述使用的参数 `MIN_LEFTOVER_BASE` 数值是 `MIN_BASE_MAX_CLOSE_FACTOR_THRESHOLD / 2`，清算者在清算后必须要留下 1000 美金以上资产或者债务，假如留下的金额小于此数值，那么清算者不能部分清算，但仍可以选择完全清算该借贷头寸。

在 AAVE v4 种，我们对上述清算规则进行了修改，其中最大的修改是删除了 close factor 机制。AAVE v4 内的清算者可以直接清算充足的债务使得被清算头寸的健康度回到协议配置的目标健康度(`targetHealthFactor`)。当然，目标健康度 `TargetHealthFactor` 一定大于最低可被清算健康度 `HEALTH_FACTOR_LIQUIDATION_THRESHOLD`。

另外，AAVE v4 引入了荷兰式拍卖机制来确定清算激励，并且设置了粉尘头寸的 safeguards，以此确保协议内不存在粉尘头寸，降低协议可能的风险。与上文介绍的利息累计相反，清算几乎完全发生在 Spoke 内部，对于 Hub 而言，清算只是一次 restore 的还款过程。但是假如出现了坏账情况，Spoke 需要使用 `reportDeficit` 函数向 Hub 进行坏账汇报。

本节介绍的代码基本都位于 `LiquidationLogic` 库内部。在此处我们也简单介绍一下 AAVE v4 清算部分与 AAVE v3 的不同:

- 引入目标健康度(*Target Health Factor*) 机制替代过去的 close factor 机制，清算者只需保证清算后头寸可以达到目标健康度即可
- 动态粉尘处理机制，V3 使用了 `MIN_LEFTOVER_BASE` 参数硬编码了头寸最低持有资产的数量，假如被清算后的头寸内资产数量小于该数值则会该笔清算交易不成功。AAVE v4 通过去掉  close-factor 优化了粉尘处理。但如果担保品或者债务被全部清算，粉尘有可能仍然存在
- 荷兰式拍卖式清算奖励，V3 使用静态的清算激励，并不依赖于借款人的健康因子。AAVE v4 引入了可变的清算激励，该激励会随着健康因子的下降而线性上涨。Governance 可以配置两个 spoke 范围内的参数设置清算奖励曲线: `healthFactorForMaxBonus` 和 `liquidationBonusFactor`

### 参数与配置

AAVE V4 可以设置以下参数配置清算机制:

| 参数                         | 描述                                                         | 限制                                                         |
| ---------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `TargetHealthFactor`         | Governor 可以配置的一个 spoke 范围的参数，该参数表示借款头寸被清算后需要达到的健康度 | 必须要≥ `HEALTH_FACTOR_LIQUIDATION_THRESHOLD` .              |
| `DUST_LIQUIDATION_THRESHOLD` | 硬编码阈值来避免极端小的剩余债务                             | 硬编码参数以 1_000 USD 为基础单位                            |
| `maxLiquidationBonus`        | 每一个 resver 都会存在最大的清算奖励，使用 bps 表示。数值 105_00 表示在清算时会存在以偿还债务的 5_00 bps 额外支付的担保品 | 必须 ≥ 100_00                                                |
| `healthFactorForMaxBonus`    | Spoke 范围参数。该参数使用 WAD 作为单位，定义了 HF 可以决定的最大清算奖励 | `healthFactorForMaxBonus` < `HEALTH_FACTOR_LIQUIDATION_THRESHOLD`. |
| `liquidationBonusFactor`     | Spoke 范围百分比，使用 bps 作为单位，定义了最低奖金；例如，当 HF 等于 `HEALTH_FACTOR_LIQUIDATION_THRESHOLD` 时，80_00 产生的清算奖励相当于最高奖金的80%。 | liquidationBonusFactor 必须 ≤ 100_00                         |

在 Spoke 内部，我们可以看到 `updateLiquidationConfig` 函数，该函数会对上述的参数进行检查:

```solidity
function updateLiquidationConfig(LiquidationConfig calldata config) external restricted {
  require(
    config.targetHealthFactor >= HEALTH_FACTOR_LIQUIDATION_THRESHOLD &&
      config.liquidationBonusFactor <= PercentageMath.PERCENTAGE_FACTOR &&
      config.healthFactorForMaxBonus < HEALTH_FACTOR_LIQUIDATION_THRESHOLD,
    InvalidLiquidationConfig()
  );
  _liquidationConfig = config;
  emit UpdateLiquidationConfig(config);
}
```

我们可以解析一下此处的 `DUST_LIQUIDATION_THRESHOLD` 等常数的默认数值。`DUST_LIQUIDATION_THRESHOLD` 被定义在 `LiquidationLogic` 库合约内部:

```solidity
uint256 public constant DUST_LIQUIDATION_THRESHOLD = 1000e26;
```

该参数的精度与预言机的精度有关，此处的 `1e26` 精度是因为假设了 oralce 使用了 8 位精度。在 `DUST_LIQUIDATION_THRESHOLD` 被使用的地方，我们可以看到如下精度计算。此处的 `toWad` 将 oracle 返回值与 1e18 相乘，而 collateral 本身的精度因为除 `collateralAssetUnit` 而被抵消。

```solidity
(params.collateralReserveBalance - collateralToLiquidate).mulDivDown(
  params.collateralAssetPrice.toWad(),
  collateralAssetUnit
) <
DUST_LIQUIDATION_THRESHOLD;
```

而 `HEALTH_FACTOR_LIQUIDATION_THRESHOLD` 函数的精度是 1e18，我们可以在 Spoke 内部看到 *Expressed in WAD (18 decimals) (e.g. 1e18 is 1.00).* 的注释。

```solidity
uint64 public constant HEALTH_FACTOR_LIQUIDATION_THRESHOLD = 1e18;
```

### Liquidation 流程

在介绍清算流程前，我们首先介绍 `liquidationCall` 所需要的参数。在 Spoke 内部，我们可以看到如下定义:

```solidity
function liquidationCall(
  uint256 collateralReserveId,
  uint256 debtReserveId,
  address user,
  uint256 debtToCover,
  bool receiveShares
) external {
```

其中 `collateralReserveId` 用于指定需要清算的担保品，而 `debtReserveId` 代表需要清算的债务，而 `user` 代表需要被清算的用户地址。上述参数用来定义需要被清算的头寸，但这些参数并没有包含清算资产的数量。清算数量是由 `debtToCover` 确定的，该参数代表清算者清算的债务数量。最后的 `receiveShares` 参数代表清算者希望直接在 Spoke 内部获得一定数量的存款，还是直接获得底层资产。前者只是进行一次内部的 shares 划转，而后者会进行一次真正的资转移。

以下简述了 AAVE v4 的清算流程:

1. 检查清算资格: 当头寸的健康因子低于 `HEALTH_FACTOR_LIQUIDATION_THRESHOLD`，任何人都可以发起清算；但是头寸持有者不被允许清算自己的仓位。协议会检索当前头寸的债务价值、健康因子和总担保品的价值，此处计算健康因子主要使用上文内已经介绍的 `_processUserAccountData` 函数
2. 决定偿还债务: 取决于设置的 `TargetHealthFactor`，协议计算将头寸恢复为 `TargetHealthFactor` 所需要偿还的债务价值。计算偿还数量依赖于借款人的当前债务和担保品
3. 处理粉尘债务: 如果借款人在标准清算后的剩余债务小于 `DUST_LIQUIDATION_THRESHOLD`，并且清算人也同意可以全额偿还债务，那么 aave v4 会上调可清算余额，以方便全部债务可以被覆盖。但是，如果清算人清算的目标债务等于担保品 $C_i$ 的全部数量，在多担保品情况下，粉尘 $D_{dust}$ 仍可能存在。但假如只有一种担保品，那么粉尘债务会被视为协议的坏账
4. 计算清算人获得的担保品数量并处理担保品粉尘: 将偿还的债务转化为担保品的价值并且根据清算情况计算清算奖励。我们可以使用清算前头寸的健康因子和担保品的 `maxLiquidationBonus` 计算出清算激励。我们就在这一步内完成激励的计算并且将运算结果数量的担保品转移给清算者。如果清算者选择的担保品无法满足，其他担保品也会被纳入清算。最后，我们会考虑担保品粉尘。
5. 执行担保品转移和债务偿还: 根据清算人的偿还金额减少借款人的债务。将包含清算激励的担保品转移给清算者，但此过程中要减去协议费用。这部分协议费用将被发送给 Hub 并以 shares 形式表示
6. 释放 event 和更新状态: `LiquidationCall` 事件会被释放，事件内部包含清算的详细情况。借款人和 reserves 的利息会被更新。如果头寸仍存在债务但不包含担保品，此时系统会将这部分债务记为协议坏账

通过上述描述，我们可以总体上看到 AAVE v4 的清算流程，接下来，我们会对其中的每一个环节进行包含代码的分析。我们首先分析检查清算资格的流程。在介绍核心逻辑前，我们首先要关注 Spoke 内对数据的准备流程。该流程会将 Spoke 内的大量状态变量抽取到内存内的结构体内，然后使用该结构体调用 `LiquidationLogic` 内的 `liquidateUser` 函数。而清算过程中，最核心的逻辑都发生在 `LiquidationLogic` 内部。

以下代码展示了 Spoke 如何将状态变量内的数据抽取出来将其转化为 `LiquidationLogic.LiquidateUserParams` 参数，这些代码都较为简单，我们可以注意到 `collateralDynConfig` 没有被更新，这说明清算过程中，我们会强制使用仓位中快照的参数。此处调用的 `_calculateUserAccountData` 和 `_getUserDebt` 都在上文有所介绍，此处不再赘述。

```solidity
Reserve storage collateralReserve = _getReserve(collateralReserveId);
Reserve storage debtReserve = _getReserve(debtReserveId);
DynamicReserveConfig storage collateralDynConfig = _dynamicConfig[collateralReserveId][
  _userPositions[user][collateralReserveId].dynamicConfigKey
];
UserAccountData memory userAccountData = _calculateUserAccountData(user);

LiquidationLogic.LiquidateUserParams memory params = LiquidationLogic.LiquidateUserParams({
  collateralReserveId: collateralReserveId,
  debtReserveId: debtReserveId,
  oracle: ORACLE,
  user: user,
  debtToCover: debtToCover,
  healthFactor: userAccountData.healthFactor,
  drawnDebt: 0, // populated below
  premiumDebt: 0, // populated below
  accruedPremiumRay: 0, // populated below
  totalDebtValue: userAccountData.totalDebtValue,
  activeCollateralCount: userAccountData.activeCollateralCount,
  borrowedCount: userAccountData.borrowedCount,
  liquidator: msg.sender,
  receiveShares: receiveShares
});
(params.drawnDebt, params.premiumDebt, , params.accruedPremiumRay) = _getUserDebt(
  debtReserve.hub,
  debtReserve.assetId,
  _userPositions[user][debtReserveId]
);
```

当我们将上述内容写入到 `LiquidateUserParams` 结构体后，我们可以使用如下代码调用 `liquidateUser` 函数:

```solidity
bool isUserInDeficit = LiquidationLogic.liquidateUser(
  collateralReserve,
  debtReserve,
  _userPositions,
  _positionStatus,
  _liquidationConfig,
  collateralDynConfig,
  params
);
```

其中 `_userPositions` / `_positionStatus` / `_liquidationConfig` 都是 Spoke 内的状态变量，方便 `LiquidationLogic` 内的函数检索 Spoke 内的数据。接下来，我们将介绍上文简述清算流程中的第一个环节，即**检查清算资格**。在 `liquidateUser` 函数内，我们第一步时调用 `hub.previewRemoveByShares` 函数计算当前用户的 `collateralReserve` 对应的资产数量。

```solidity
uint256 collateralReserveBalance = collateralReserve.hub.previewRemoveByShares(
  collateralReserve.assetId,
  positions[params.user][params.collateralReserveId].suppliedShares
);
_validateLiquidationCall(
  positionStatus[params.user].isUsingAsCollateral(params.collateralReserveId),
  ValidateLiquidationCallParams({
    user: params.user,
    liquidator: params.liquidator,
    debtToCover: params.debtToCover,
    collateralReserveHub: address(collateralReserve.hub),
    debtReserveHub: address(debtReserve.hub),
    collateralReservePaused: collateralReserve.paused,
    collateralReserveFrozen: collateralReserve.frozen,
    debtReservePaused: debtReserve.paused,
    healthFactor: params.healthFactor,
    collateralReserveId: params.collateralReserveId,
    collateralFactor: collateralDynConfig.collateralFactor,
    collateralReserveBalance: collateralReserveBalance,
    debtReserveBalance: params.drawnDebt + params.premiumDebt,
    receiveShares: params.receiveShares
  })
);
```

此处我们需要分析上述代码内的 `_validateLiquidationCall` 函数，该函数的实现如下所述，具体检查的内容包括:

1. 避免自清算行为，由于 AAVE v4 坏账是全局承担的，所以理论上仓位持有人有动机自清算为 Hub 带来额外的坏账
2. 保证 `params.debtToCover > 0`，这是一个简单的参数检查
3. 保证 `collateralReservePaused` 和 `debtReservePaused` 都处于非暂停状态
4. `collateralReserveBalance > 0` 和 `debtReserveBalance > 0` 保证清算的担保品和债务是存在的
5. 保证头寸处于可清算状态(即 `healthFactor < HEALTH_FACTOR_LIQUIDATION_THRESHOLD`)
6. 避免清算者清算不作为担保品的资产(`collateralFactor > 0 && isBorrowerUsingAsCollateral`)
7. 假如清算者选择了 `receiveShares` 选项，那么此处的要求待清算的担保品不处于冻结状态

> 在上文内，我们已经介绍过 Spoke 内的 reserve 存在几个不同的状态，此处再次引用上文内的内容: 一旦 reserve 处于 `paused` 状态，那么该资产无法倍执行任何操作，但如果只是处于 `frozen` 状态，那么该资产仍可以被 withdraw，但不能被设置为担保品。此处假如清算者选择 `receiveShares`，这本质上是将资产划转给清算者，当资产处于 `frozen` 状态时，我们不应该允许清算者获得 shares

```solidity
function _validateLiquidationCall(
  bool isBorrowerUsingAsCollateral,
  ValidateLiquidationCallParams memory params
) internal pure {
  require(params.user != params.liquidator, ISpoke.SelfLiquidation());
  require(params.debtToCover > 0, ISpoke.InvalidDebtToCover());
  require(!params.collateralReservePaused && !params.debtReservePaused, ISpoke.ReservePaused());
  require(params.collateralReserveBalance > 0, ISpoke.ReserveNotSupplied());
  require(params.debtReserveBalance > 0, ISpoke.ReserveNotBorrowed());
  require(
    params.healthFactor < HEALTH_FACTOR_LIQUIDATION_THRESHOLD,
    ISpoke.HealthFactorNotBelowThreshold()
  );
  require(
    params.collateralFactor > 0 && isBorrowerUsingAsCollateral,
    ISpoke.CollateralCannotBeLiquidated()
  );
  if (params.receiveShares) {
    require(!params.collateralReserveFrozen, ISpoke.CannotReceiveShares());
  }
}
```

至此，我们就介绍完成了清算流程的第一步，接下来，我们需要执行 **决定偿还债务** 和 **处理粉尘债务** 的环节。这部分主要涉及 `_calculateLiquidationAmounts` 函数。我们首先阅读 `liquidateUser` 内对 `_calculateLiquidationAmounts` 的参数调用:

```solidity
LiquidationAmounts memory liquidationAmounts = _calculateLiquidationAmounts(
  CalculateLiquidationAmountsParams({
    healthFactorForMaxBonus: liquidationConfig.healthFactorForMaxBonus,
    liquidationBonusFactor: liquidationConfig.liquidationBonusFactor,
    targetHealthFactor: liquidationConfig.targetHealthFactor,
    debtReserveBalance: params.drawnDebt + params.premiumDebt,
    collateralReserveBalance: collateralReserveBalance,
    debtToCover: params.debtToCover,
    totalDebtValue: params.totalDebtValue,
    healthFactor: params.healthFactor,
    maxLiquidationBonus: collateralDynConfig.maxLiquidationBonus,
    collateralFactor: collateralDynConfig.collateralFactor,
    liquidationFee: collateralDynConfig.liquidationFee,
    debtAssetPrice: IAaveOracle(params.oracle).getReservePrice(params.debtReserveId),
    debtAssetDecimals: debtReserve.decimals,
    collateralAssetPrice: IAaveOracle(params.oracle).getReservePrice(
      params.collateralReserveId
    ),
    collateralAssetDecimals: collateralReserve.decimals
  })
);
```

上述代码中大部分参数都是直接从状态变量检索获得的，只有 `debtAssetPrice` 和 `collateralAssetPrice` 变量是从 AAVE Oracle 内读取的。对于债务偿还金额的过程，我们可以细分为以下几个步骤:

1. 计算清算奖励，该部分会调用 `calculateLiquidationBonus` 函数。在 AAVE v4 内，我们使用了类荷兰式拍卖的清算激励机制
2. 计算待清算的债务数量
3. 根据待清算债务被清算后剩余的债务情况，判断是否需要粉尘债务处理，在上文中，我们已经介绍了粉尘债务的相关概念
4. 根据债务情况计算出需要向清算人支付的担保品数量

我们首先研究 AAVE v4 内的清算奖励的机制。AAVE v4 的清算激励是基于借款人的健康度，并且在最小值和最大值之间线性变化:

- **最大清算激励区域**: 当 HF ≤ `healthFactorForMaxBonus` 时，清算者可以获得最大的清算激励(`maxLiquidationBonus` ) 减去一部分清算手续费(`liquidationFee`)。比如: 如果 `maxLiquidationBonus = 105_00` 且 `liquidationFee = 10_00`，那么清算者可以获得清算债务的 5% 作为激励，当由于存在 10% 的手续费，所以清算者最终可以获得 4.5% 的清算激励
- **阈值区域**: 当 HF = `HEALTH_FACTOR_LIQUIDATION_THRESHOLD` 时，清算激励等于 `liquidationBonusFactor × maxLiquidationBonus`，以此确保在该情况下，清算者的激励不为零
- **线性区域**: 当 HF 介于 `healthFactorForMaxBonus` 和清算阈值之间时，清算激励会从 `liquidationBonusFactor × maxLiquidationBonus` 到 `maxLiquidationBonus` 之间线性增加

我们可以使用如下公式计算出清算者可以获得最大清算奖励:
$$
\text{maxLB} = (\text{maxLB} - 100\\%) \times \text{lbFactor} + 100\\%
$$
上述公式内的：

- $\text{maxLB}$: 代表某种担保品可以获得最大清算激励，该数值大于 100%，比如 103% 代表最大可以获得 3% 的最大清算激励
- $\text{lbFactor}$ 代表在最小情况下，清算者可以获得最大清算激励的比例，该数值一定小于 100%，比如在 $\text{maxLB} = 103 \%$ 的情况下，$\text{lbFactor} = 50\%$ 意味着清算者最小获得 $101.5\%$ 的担保品

有了上述参数后，我们可以使用如下公式计算清算者激励:
$$
\text{lb} = \begin{cases}
\text{maxLB} & \text{if } hf\_{\text{beforeLiq}} \le \text{hfForMaxBonus} \\\\
\text{minLB} + (maxLB - minLB) \times \frac{\text{HF\\_LIQ\\_THRESHOLD} - hf\_{\text{beforeLiq}}}{\text{HF\\_LIQ\\_THRESHOLD} - \text{hfForMaxBonus}} & \text{if } hf\_{\text{beforeLiq}} > \text{hfForMaxBonus}
\end{cases}
$$
上述公式中:

- $\text{HF\\_LIQ\\_THRESHOLD}$ 一旦用户头寸健康因子低于该参数，那么该头寸就是可被清算的
- $hf_{\text{beforeLiq}}$ 代表用户清算前的健康因子
- $\text{hfForMaxBonus}$ 代表当前配置下协议可以给予的最大的清算激励奖励

在代码中，我们会使用如下函数计算用户的清算奖励。以下代码中的 `healthFactorForMaxBonus` 就是上文中的 maxLB，而 `liquidationBonusFactor` 就是上文中的 lbFactor。

```solidity
uint256 liquidationBonus = calculateLiquidationBonus({
  healthFactorForMaxBonus: params.healthFactorForMaxBonus,
  liquidationBonusFactor: params.liquidationBonusFactor,
  healthFactor: params.healthFactor,
  maxLiquidationBonus: params.maxLiquidationBonus
});
```

在具体的代码实现中，我们使用如下代码进行实现:

```solidity
function calculateLiquidationBonus(
  uint256 healthFactorForMaxBonus,
  uint256 liquidationBonusFactor,
  uint256 healthFactor,
  uint256 maxLiquidationBonus
) internal pure returns (uint256) {
  if (healthFactor <= healthFactorForMaxBonus) {
    return maxLiquidationBonus;
  }

  uint256 minLiquidationBonus = (maxLiquidationBonus - PercentageMath.PERCENTAGE_FACTOR)
    .percentMulDown(liquidationBonusFactor) + PercentageMath.PERCENTAGE_FACTOR;

  // linear interpolation between min and max
  // denominator cannot be zero as healthFactorForMaxBonus is always < HEALTH_FACTOR_LIQUIDATION_THRESHOLD
  return
    minLiquidationBonus +
    (maxLiquidationBonus - minLiquidationBonus).mulDivDown(
      HEALTH_FACTOR_LIQUIDATION_THRESHOLD - healthFactor,
      HEALTH_FACTOR_LIQUIDATION_THRESHOLD - healthFactorForMaxBonus
    );
}
```

上述代码首先利用 $\text{minLB} = (\text{maxLB} - 100\%) \times \text{lbFactor} + 100\%$ 进行计算，然后使用上文给出的计算方法对清算奖励进行计算。上述代码计算出结果是一个百分比，我们会在后文将该数值与可清算担保品相乘计算出清算者可以获得的担保品数量。

按照最初的步骤，我们已经完成了最初的激励因子计算部分，接下来，我们需要完成可清算债务的计算。该过程主要使用 `_calculateDebtToLiquidate` 函数完成。在上文，我们提到偿还债务数量取决于设置的 `TargetHealthFactor`，即我们需要清算一定的债务使得头寸恢复为 `TargetHealthFactor` 

```solidity
uint256 debtToLiquidate = _calculateDebtToLiquidate(
  CalculateDebtToLiquidateParams({
    debtReserveBalance: params.debtReserveBalance,
    debtToCover: params.debtToCover,
    totalDebtValue: params.totalDebtValue,
    healthFactor: params.healthFactor,
    targetHealthFactor: params.targetHealthFactor,
    liquidationBonus: liquidationBonus,
    collateralFactor: params.collateralFactor,
    debtAssetPrice: params.debtAssetPrice,
    debtAssetUnit: debtAssetUnit
  })
);
```

在 `_calculateDebtToLiquidate` 函数内部，我们第一步就是根据 `targetHealthFactor` 计算出清算到目标健康度所需要的清算的债务数量。在以下代码中，我们使用 `_calculateDebtToTargetHealthFactor` 完成该计算。我们可以看到假如清算者指定的清算数量 `params.debtToCover < debtReserveBalance`，那么我们会暂时将 `debtToLiquidate` 设置为清算者指定的数量。在完成了 `_calculateDebtToTargetHealthFactor` 计算后，假如发现 `debtToTarget < debtToLiquidate`，即清算至目标健康度的债务小于用户指定的清算数量，那么我们就会将 `debtToLiquidate` 设置为 `debtToTarget` 数量。

```solidity
uint256 debtToLiquidate = params.debtReserveBalance;
if (params.debtToCover < debtToLiquidate) {
  debtToLiquidate = params.debtToCover;
}

uint256 debtToTarget = _calculateDebtToTargetHealthFactor(
  CalculateDebtToTargetHealthFactorParams({
    totalDebtValue: params.totalDebtValue,
    healthFactor: params.healthFactor,
    targetHealthFactor: params.targetHealthFactor,
    liquidationBonus: params.liquidationBonus,
    collateralFactor: params.collateralFactor,
    debtAssetPrice: params.debtAssetPrice,
    debtAssetUnit: params.debtAssetUnit
  })
);
if (debtToTarget < debtToLiquidate) {
  debtToLiquidate = debtToTarget;
}
```

接下来，在 `_calculateDebtToLiquidate` 函数内(注意，我们暂时忽略了 `_calculateDebtToTargetHealthFactor` 的实现)，我们会执行粉尘债务的处理工作。假如我们发现按照 `debtToLiquidate` 进行清算，清算后的债务价值低于 `DUST_LIQUIDATION_THRESHOLD`，协议后增加可清算债务的数量，提供给清算者直接清算全部债务的能力。

```solidity
bool leavesDebtDust = debtToLiquidate < params.debtReserveBalance &&
  (params.debtReserveBalance - debtToLiquidate).mulDivDown(
    params.debtAssetPrice.toWad(),
    params.debtAssetUnit
  ) <
  DUST_LIQUIDATION_THRESHOLD;

if (leavesDebtDust) {
  // target health factor is bypassed to prevent leaving dust
  debtToLiquidate = params.debtReserveBalance;
}

return debtToLiquidate;
```

至此，我们就完成了可清算债务数量的计算。然后，我们回到 `_calculateDebtToTargetHealthFactor` 的实现，这其实也不是一个复杂函数。我们第一步使用了 `liquidationPenalty` 参数，这种参数

```solidity
/// @notice Calculates the amount of debt needed to be liquidated to restore a position to the target health factor.
function _calculateDebtToTargetHealthFactor(
  CalculateDebtToTargetHealthFactorParams memory params
) internal pure returns (uint256) {
  uint256 liquidationPenalty = params.liquidationBonus.bpsToWad().percentMulUp(
    params.collateralFactor
  );

  // denominator cannot be zero as `liquidationPenalty` is always < PercentageMath.PERCENTAGE_FACTOR
  // `liquidationBonus.percentMulUp(collateralFactor) < PercentageMath.PERCENTAGE_FACTOR` is enforced in `_validateDynamicReserveConfig`
  // and targetHealthFactor is always >= HEALTH_FACTOR_LIQUIDATION_THRESHOLD
  return
    params.totalDebtValue.mulDivUp(
      params.debtAssetUnit * (params.targetHealthFactor - params.healthFactor),
      (params.targetHealthFactor - liquidationPenalty) * params.debtAssetPrice.toWad()
    );
}
```

我们首先给出在进行 $\Delta D$ 的债务偿还后，计算出的 $\text{HF}\_{\text{target}}$ 的值:
$$
\text{HF}\_{\text{target}} = \frac{(\text{Collateral Value} - \Delta D \times LB) \times CF}{D - \Delta D}
$$
上述公式内的 $LB$ 指的是 Liquidation Bonus，而  CF 指的是 Collateral Factor。我们的目标是求解出上文中 $\Delta D$ 的数学表达式:
$$
\begin{align*}
\text{HF}\_{\text{target}}({D - \Delta D}) &= \text{Collateral Value} \times CF - \Delta D \times LB \times CF\\\\
\text{HF}\_{\text{target}}D - \text{HF}\_{\text{target}}\Delta D &= \text{HF}\_{\text{current}}D - \Delta D \times LB \times CF\\\\
\Delta D \times LB \times CF - \text{HF}\_{\text{target}}\Delta D &= (\text{HF}\_{\text{current}} - \text{HF}\_{\text{target}})D\\\\
\Delta D &= \frac{(\text{HF}\_{\text{target}} - \text{HF}\_{\text{current}}) D}{\text{HF}\_{\text{target}}  -  LB \times CF}
\end{align*}
$$
在上述代码内，我们将 $LB \times CF$ 的值使用 `liquidationPenalty` 进行了计算。额外需要注意的，上述计算过程中，并没有考虑债务的价格，所以在最后编写代码时，我们协议将 `debtAssetPrice` 也列到分母中除掉。

我们使用 `_calculateDebtToLiquidate` 函数完成了待清算债务的计算，进一步我们协议考虑粉尘债务的影响。我们在此处再次介绍所谓粉尘债务的概念，所谓粉尘债务就是指完成清算后，用户的债务低于 `DUST_LIQUIDATION_THRESHOLD`，目前设置为 1000 美金。在这种情况下，AAVE v4 会允许清算者将被清算的头寸完全清算。

在具体实现中，我们首先使用刚刚计算出的 `debtToLiquidate` 进一步计算获得待清算的担保品的数量，计算方法是 `(debtToLiquidate * debtAssetPrice * liquidationBonus) / collateralAssetPrice`。我们可以直接将其翻译为以下代码。以下代码内的 `collateralAssetUnit` 和 `debtAssetUnit` 都是用于进行代币精度调整的额外因子。而 `PercentageMath.PERCENTAGE_FACTOR` 则用于抵消 `liquidationBonus` 的精度影响。

```solidity
uint256 collateralToLiquidate = debtToLiquidate.mulDivDown(
  params.debtAssetPrice * collateralAssetUnit * liquidationBonus,
  debtAssetUnit * params.collateralAssetPrice * PercentageMath.PERCENTAGE_FACTOR
);
```

接下来，我们判断是否存在粉尘头寸。首先，只有在待清算担保品数量小于担保品余额时，我们才会进行判断，因为假如需要清算的担保品数量大于用户的余额，此时不需要额外的判断。假如待清算的担保品数量小于用户余额，我们会计算剩余的担保品价值 `(collateralReserveBalance - collateralReserveBalance) * collateralAssetPrice`与 `DUST_LIQUIDATION_THRESHOLD` 进行比较。

```
bool leavesCollateralDust = collateralToLiquidate < params.collateralReserveBalance &&
  (params.collateralReserveBalance - collateralToLiquidate).mulDivDown(
    params.collateralAssetPrice.toWad(),
    collateralAssetUnit
  ) <
  DUST_LIQUIDATION_THRESHOLD;
```

最后，我们会对 `debtToLiquidate` 进行调整。此处的调整会发生在以下两种情况下:

1. `collateralToLiquidate > params.collateralReserveBalance` 需要清算的担保品数量大于用户的担保品余额
2. 在 `leavesCollateralDust` 情况下，我们需要清算用户的全部债务头寸，但是假如 `debtToLiquidate >= params.debtReserveBalance`，那么我们就需要进行调整，所以此处调整的发生条件是 `leavesCollateralDust && debtToLiquidate < params.debtReserveBalance`

对于上述两种情况，我们都需要将担保品全部清算(即 `collateralToLiquidate = params.collateralReserveBalance`)，也需要以此修改 `debtToLiquidate` 的数值，我们会使用 `collateralToLiquidate` 重新计算 `debtToLiquidate` 的数值。完成上述计算后，我们需要确定调整后的 `debtToLiquidate` 与清算者指定的 `params.debtToCover` 进行比较。

```
if (
  collateralToLiquidate > params.collateralReserveBalance ||
  (leavesCollateralDust && debtToLiquidate < params.debtReserveBalance)
) {
  collateralToLiquidate = params.collateralReserveBalance;

  debtToLiquidate = collateralToLiquidate.mulDivUp(
    params.collateralAssetPrice * debtAssetUnit * PercentageMath.PERCENTAGE_FACTOR,
    params.debtAssetPrice * collateralAssetUnit * liquidationBonus
  );
}

// revert if the liquidator does not cover the necessary debt to prevent dust from remaining
require(params.debtToCover >= debtToLiquidate, ISpoke.MustNotLeaveDust());
```

最后，我们进入了 `_calculateLiquidationAmounts` 最后环节，在该环节中我们需要处理手续费问题。此处我们需要注意，我们的手续费主要针对清算奖励(`liquidationBonus` 部分)进行征收。我们使用 `collateralToLiquidate` 代表清算的总担保品数量，而 `collateralToLiquidator` 代表真正支付给清算者的资金，注意支付给清算者的资金需要扣除手续费(`liquidationFee`)。我们可以获得如下计算方法，其中等式两侧计算的都是手续费的数值:
$$
\text{collateralToLiquidate} - \text{collateralToLiquidator} = \text{collateralToLiquidate}\times\frac{(\text{LB} - 1)\times\text{liquidationFee}}{\text{LB}}
$$
所以，我们可以获得如下代码:

```solidity
uint256 collateralToLiquidator = collateralToLiquidate -
  collateralToLiquidate.mulDivDown(
    params.liquidationFee * (liquidationBonus - PercentageMath.PERCENTAGE_FACTOR),
    liquidationBonus * PercentageMath.PERCENTAGE_FACTOR
  );

return
  LiquidationAmounts({
    collateralToLiquidate: collateralToLiquidate,
    collateralToLiquidator: collateralToLiquidator,
    debtToLiquidate: debtToLiquidate
  });
```

至此，我们就完成了清算环节中的 **计算清算人获得的担保品数量并处理担保品粉尘** 环节，接下来，我们需要完成了 **执行债务偿还和担保品转移** 环节。该环节主要依赖于 `_liquidateCollateral` 和 `_liquidateDebt` 函数，这两个函数本质上都依赖于 `hub.remove` 和 `hub.restore` 函数。

我们首先阅读 `_liquidateCollateral` 函数，该函数的实现十分简单，代码的核心是计算 `sharesToLiquidate` 和 `sharesToLiquidator` 数值。我们首先使用 `previewRemoveByAssets` 计算出 `sharesToLiquidate`，即清算担保品的总价值对应的 shares 数量，然后我们会计算更新后的 `userSuppliedShares` 数值。然后，我们会进行真正的担保品转账工作，假如清算者选择了 `receiveShares` 选项，那么我们会在 `positions` 为其增加 `sharesToLiquidator`。但如果用户选择了直接提取资产，我们会调用 `hub.remove` 为用户直接发送资产。最后，我们会更新被清算者的 `suppliedShares`，然后视情况调用 `payFeeShares` 支付清算费用。

```solidity
ISpoke.UserPosition storage collateralPosition = positions[params.user][
  params.collateralReserveId
];
IHubBase hub = collateralReserve.hub;
uint256 assetId = collateralReserve.assetId;

uint256 sharesToLiquidate = hub.previewRemoveByAssets(assetId, params.collateralToLiquidate);
uint120 userSuppliedShares = collateralPosition.suppliedShares - sharesToLiquidate.toUint120();

uint256 sharesToLiquidator;
if (params.collateralToLiquidator > 0) {
  if (params.receiveShares) {
    sharesToLiquidator = hub.previewAddByAssets(assetId, params.collateralToLiquidator);
    if (sharesToLiquidator > 0) {
      positions[params.liquidator][params.collateralReserveId]
        .suppliedShares += sharesToLiquidator.toUint120();
    }
  } else {
    sharesToLiquidator = hub.remove(assetId, params.collateralToLiquidator, params.liquidator);
  }
}

collateralPosition.suppliedShares = userSuppliedShares;

if (sharesToLiquidate > sharesToLiquidator) {
  hub.payFeeShares(assetId, sharesToLiquidate.uncheckedSub(sharesToLiquidator));
}

return (sharesToLiquidate, sharesToLiquidator, userSuppliedShares == 0);
```

对于 `_liquidateDebt` 函数，此处代码的核心是计算 `premiumDebtToLiquidate` 和 `drawnDebtToLiquidate`。我们在上文提到过用户偿还债务时会优先偿还 premium debt，清算本质上是一种还款行为，所以此处我们第一步就是判断偿还债务的数量是否可以覆盖 premium debt，假如不可以覆盖，那么清算者清偿的债务只包含 premium debt。此处的 `premiumDelta` 的构建和使用与 Spoke 内的 `repay` 函数完全一致。此处需要注意，我们需要将清算者的代币转移给 Hub 来保证后续的 `restore` 的正常执行。

```solidity
uint256 premiumDebtToLiquidateRay = params.debtToLiquidate.toRay().min(
  debtPosition.realizedPremiumRay + params.accruedPremiumRay
);
uint256 premiumDebtToLiquidate = premiumDebtToLiquidateRay.fromRayUp();
uint256 drawnDebtToLiquidate = params.debtToLiquidate - premiumDebtToLiquidate;

IHubBase.PremiumDelta memory premiumDelta = IHubBase.PremiumDelta({
  sharesDelta: -debtPosition.premiumShares.toInt256(),
  offsetDeltaRay: -debtPosition.premiumOffsetRay.toInt256(),
  accruedPremiumRay: params.accruedPremiumRay,
  restoredPremiumRay: premiumDebtToLiquidateRay
});

debtReserve.underlying.safeTransferFrom(
  params.liquidator,
  address(debtReserve.hub),
  drawnDebtToLiquidate + premiumDebtToLiquidate
);
uint256 drawnSharesLiquidated = debtReserve.hub.restore(
  debtReserve.assetId,
  drawnDebtToLiquidate,
  premiumDelta
);
debtPosition.applyPremiumDelta(premiumDelta);
debtPosition.drawnShares -= drawnSharesLiquidated.toUint120();

bool isDebtPositionEmpty = false;
if (debtPosition.drawnShares == 0) {
  positionStatus.setBorrowing(params.debtReserveId, false);
  isDebtPositionEmpty = true;
}

return (drawnSharesLiquidated, premiumDelta, isDebtPositionEmpty);
```

至此，我们基本完成了清算的所有流程，但此时还差最后一步，即坏账处理。假如当前头寸的所有担保品都已经被清算，但是仍存在债务，我们需要将这部分债务记为坏账。我们可以通过以下任意条件知道当前头寸仍存在担保品

1. 被清算的担保品仍存在一定数量
2. 可用的担保品的数量大于 1

假如用户的担保品被耗尽已经确定，那么接下来我们需要确定债务仍存在，包含两种情况:

1. 当前清算者清算的债务资产仍存在
2. 头寸借出的债务资产种类大于 1

上述条件综合可以获得如下函数:

```solidity
function _evaluateDeficit(
  bool isCollateralPositionEmpty,
  bool isDebtPositionEmpty,
  uint256 activeCollateralCount,
  uint256 borrowedCount
) internal pure returns (bool) {
  if (!isCollateralPositionEmpty || activeCollateralCount > 1) {
    return false;
  }
  return !isDebtPositionEmpty || borrowedCount > 1;
}
```

> 实际上，`borrowedCount` 会在清算函数中被调整，在 `_liquidateDebt` 内部，我们可以看到 `positionStatus.setBorrowing(params.debtReserveId, false);` 调整。而 `activeCollateralCount` 会在 Spoke 内的 `_processUserAccountData` 遍历担保品时计算

至此，我们就完成了清算过程中最核心的 `liquidateUser` 函数的构建，接下来，我们回到 Spoke 内的 `liquidationCall` 函数继续进行分析，剩余的代码如下:

```solidity
bool isUserInDeficit = LiquidationLogic.liquidateUser(
  collateralReserve,
  debtReserve,
  _userPositions,
  _positionStatus,
  _liquidationConfig,
  collateralDynConfig,
  params
);

uint256 newRiskPremium = 0;
if (isUserInDeficit) {
  _reportDeficit(user);
} else {
  newRiskPremium = _calculateUserAccountData(user).riskPremium;
}
_notifyRiskPremiumUpdate(user, newRiskPremium);
```

假如用户存在坏账，Spoke 会调用 `_reportDeficit` 向 Hub 汇报用户的坏账情况，我们在上文已经介绍过该函数。简单来说，该函数会计算当前用户头寸剩余的 Premium Debt 和 Drawn debt 的情况，然后直接汇报给 Hub。

否则，我们就会计算当前用户头寸的 `riskPremium`，然后使用 `_notifyRiskPremiumUpdate` 触发 Hub 更新，这套逻辑与 `repay` 的实现几乎完全一致。

## 总结

至此，我们完成了 AAVE v4 核心部分的代码分析，分别包括:

1. Hub 和 Spoke 的分离架构
2. Risk Premium 的执行机制
3. Premium Rate 和 Debt Rate 的利息累计作用
4. 类荷兰式拍卖的清算奖励和抗粉尘的清算机制

但本文并没有介绍一些额外的可以被视为外围模块的 AAVE v4 合约，比如各种 Position Manager 的合约。笔者在编写此文时发现 AAVE v4 在 `tests/misc` 对部分属性使用 z3 进行了形式化证明，在下一篇文章内，我们将以 AAVE v4 为例介绍如何使用 z3 对某些属性进行形式化证明。

