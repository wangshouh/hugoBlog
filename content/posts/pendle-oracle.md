---
title: "深入浅出借贷市场内的 Pendle 预言机"
date: 2025-09-17T08:00:00Z
tags: [defi]
math: true
---

## 概述

在构建基于 Uniswap V4 Hook 的借贷协议 [Licredity](https://github.com/Licredity) 时，我们希望引入 PT 作为担保品，但此前我并没有详细了解过 PT 预言机的开发生态，所以我阅读了目前 Morpho 内几个较大使用 PT 的市场，并阅读了这些市场内的预言机实现。

本文默认读者熟悉 Morpho 的标准预言机实现，假如读者对此不熟悉，可以阅读 [现代 DeFi: 最小化借贷协议 Morpho](https://blog.wssh.trade/posts/morpho-bule/) 一文。

## [PT-USDe-25SEP2025 / DAI](https://app.morpho.org/ethereum/market/0x45d97c66db5e803b9446802702f087d4293a2f74b370105dc3a88a278bf6bb21/pt-usde-25sep2025-dai?subTab=advanced)

该市场本质上使用了 [0x59CadA9800cc83E68Ba795c8A5982F4e6dB037eC](https://etherscan.io/address/0x59CadA9800cc83E68Ba795c8A5982F4e6dB037eC#code) 作为预言机，源代码如下:

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.19;

contract PendleSparkLinearDiscountOracle {
    uint256 private constant SECONDS_PER_YEAR = 365 days;
    uint256 private constant ONE = 1e18;

    address public immutable PT;
    uint256 public immutable maturity;
    uint256 public immutable baseDiscountPerYear; // 100% = 1e18

    constructor(address _pt, uint256 _baseDiscountPerYear) {
        require(_baseDiscountPerYear <= 1e18, "invalid discount");
        require(_pt != address(0), "zero address");

        PT = _pt;
        maturity = PTExpiry(PT).expiry();
        baseDiscountPerYear = _baseDiscountPerYear;
    }

    function latestRoundData()
        external
        view
        returns (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound)
    {
        uint256 timeLeft = (maturity > block.timestamp) ? maturity - block.timestamp : 0;
        uint256 discount = getDiscount(timeLeft);

        require(discount <= ONE, "discount overflow");

        return (0, int256(ONE - discount), 0, 0, 0);
    }

    function decimals() external pure returns (uint8) {
        return 18;
    }

    function getDiscount(uint256 timeLeft) public view returns (uint256) {
        return (timeLeft * baseDiscountPerYear) / SECONDS_PER_YEAR;
    }
}

interface PTExpiry {
    function expiry() external view returns (uint256);
}
```

直接将 PT-USDe-25SEP2025 视为折价的 USDe，并且假设 `1 USDe = 1 DAI`，并没有处理 USDe 可能的脱锚问题。在 Pendle PT 内部，PT 必须实现以下接口：

```solidity
interface IPPrincipalToken is IERC20Metadata {
    function burnByYT(address user, uint256 amount) external;

    function mintByYT(address user, uint256 amount) external;

    function initialize(address _YT) external;

    function SY() external view returns (address);

    function YT() external view returns (address);

    function factory() external view returns (address);

    function expiry() external view returns (uint256);

    function isExpired() external view returns (bool);
}
```

但需要注意的，由于在 Licredity Oracle 内部进行 `maxStaleness` 的预言机返回数据过期检验，所以此处我们不能直接使用 `PendleSparkLinearDiscountOracle` 的源代码，此处的 `return (0, int256(ONE - discount), 0, 0, 0);` 无法通过过期检查。

## [PT-USDe-25SEP2025 / USDC](https://app.morpho.org/ethereum/market/0x7a5d67805cb78fad2596899e0c83719ba89df353b931582eb7d3041fd5a06dc8/pt-usde-25sep2025-usdc?subTab=advanced)

### MetaOracleDeviationTimelock

该预言机并没有使用 Morpho 自己的预言机实现，而是单独实现了一个符合 Morpho 接口的预言机，该预言机地址是 `0xe6aBD3B78Abbb1cc1Ee76c5c3689Aa9646481Fbb`。

Morpho 要求的预言机接口为:

```solidity
interface IOracle {
    /// @notice Returns the price of 1 asset of collateral token quoted in 1 asset of loan token, scaled by 1e36.
    /// @dev It corresponds to the price of 10**(collateral token decimals) assets of collateral token quoted in
    /// 10**(loan token decimals) assets of loan token with `36 + loan token decimals - collateral token decimals`
    /// decimals of precision.
    function price() external view returns (uint256);
}
```

简单来说，Morpho 要求返回 1 单位担保品以债务资产计价的情况。在 [Morpho](https://github.com/morpho-org/morpho-blue/blob/main/src/Morpho.sol#L511) 内部，我们在 `_isHealthy` 内部使用该价格:

```solidity
/// @dev Returns whether the position of `borrower` in the given market `marketParams` is healthy.
/// @dev Assumes that the inputs `marketParams` and `id` match.
function _isHealthy(MarketParams memory marketParams, Id id, address borrower) internal view returns (bool) {
    if (position[id][borrower].borrowShares == 0) return true;

    uint256 collateralPrice = IOracle(marketParams.oracle).price();

    return _isHealthy(marketParams, id, borrower, collateralPrice);
}

/// @dev Returns whether the position of `borrower` in the given market `marketParams` with the given
/// `collateralPrice` is healthy.
/// @dev Assumes that the inputs `marketParams` and `id` match.
/// @dev Rounds in favor of the protocol, so one might not be able to borrow exactly `maxBorrow` but one unit less.
function _isHealthy(MarketParams memory marketParams, Id id, address borrower, uint256 collateralPrice)
    internal
    view
    returns (bool)
{
    uint256 borrowed = uint256(position[id][borrower].borrowShares).toAssetsUp(
        market[id].totalBorrowAssets, market[id].totalBorrowShares
    );
    uint256 maxBorrow = uint256(position[id][borrower].collateral).mulDivDown(collateralPrice, ORACLE_PRICE_SCALE)
        .wMulDown(marketParams.lltv);

    return maxBorrow >= borrowed;
}

```

继续阅读 `0xe6aBD3B78Abbb1cc1Ee76c5c3689Aa9646481Fbb` 源代码，该预言机实现了

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.20;

import {Initializable} from "@openzeppelin/contracts/proxy/utils/Initializable.sol";
import {IMetaOracleDeviationTimelock} from "./interfaces/IMetaOracleDeviationTimelock.sol";
import {IOracle} from "./interfaces/IOracle.sol";

/// @title MetaOracleDeviationTimelock
/// @author Steakhouse Financial
/// @notice A meta-oracle that selects between a primary and backup oracle based on price deviation and timelocks.
/// @dev Switches to backup if primary deviates significantly, switches back when prices reconverge.
/// MUST be initialized by calling the `initialize` function.
contract MetaOracleDeviationTimelock is IMetaOracleDeviationTimelock, Initializable {
    // --- Configuration (set during initialization) ---
    IOracle public primaryOracle;
    IOracle public backupOracle;
    uint256 public deviationThreshold; // Scaled by 1e18 (e.g., 0.01e18 for 1%)
    uint256 public challengeTimelockDuration; // Duration in seconds
    uint256 public healingTimelockDuration; // Duration in seconds

    // --- State ---
    IOracle public currentOracle; // Currently selected oracle
    uint256 public challengeExpiresAt; // Timestamp when challenge period ends (0 if not challenged)
    uint256 public healingExpiresAt; // Timestamp when healing period ends (0 if not healing)

    /// @param _primaryOracle The primary price feed.
    /// @param _backupOracle The backup price feed.
    /// @param _deviationThreshold The maximum allowed relative deviation (scaled by 1e18) before a challenge can be initiated.
    /// @param _challengeTimelockDuration The duration (seconds) a challenge must persist before switching to backup.
    /// @param _healingTimelockDuration The duration (seconds) prices must remain converged before switching back to primary.
    function initialize(
        IOracle _primaryOracle,
        IOracle _backupOracle,
        uint256 _deviationThreshold,
        uint256 _challengeTimelockDuration,
        uint256 _healingTimelockDuration
    ) external initializer {
        require(address(_primaryOracle) != address(0), "Invalid primary oracle");
        require(address(_backupOracle) != address(0), "Invalid backup oracle");
        require(address(_primaryOracle) != address(_backupOracle), "Oracles must be different");
        require(_deviationThreshold > 0, "Deviation threshold must be positive");

        primaryOracle = _primaryOracle;
        backupOracle = _backupOracle;
        deviationThreshold = _deviationThreshold;
        challengeTimelockDuration = _challengeTimelockDuration;
        healingTimelockDuration = _healingTimelockDuration;

        // Check initial deviation
        uint256 initialPrimaryPrice = _primaryOracle.price();
        uint256 initialBackupPrice = _backupOracle.price();
        uint256 initialDeviation;
        if (initialBackupPrice == 0) {
            initialDeviation = initialPrimaryPrice == 0 ? 0 : type(uint256).max;
        } else {
            uint256 diff;
            if (initialPrimaryPrice >= initialBackupPrice) {
                diff = initialPrimaryPrice - initialBackupPrice;
            } else {
                diff = initialBackupPrice - initialPrimaryPrice;
            }
            initialDeviation = (diff * 10**18) / initialBackupPrice;
        }
        require(initialDeviation <= _deviationThreshold, "MODT: Initial deviation too high");

        currentOracle = _primaryOracle; // Start with the primary oracle
    }

    /// @inheritdoc IOracle
    function price() public view returns (uint256) {
        try currentOracle.price() returns (uint256 currentPrice) {
            return currentPrice;
        } catch {
            if (isPrimary()) {
                return backupOracle.price();
            } else {
                return primaryOracle.price();
            }
        }
    }

    /// @inheritdoc IMetaOracleDeviationTimelock
    function primaryPrice() public view returns (uint256) {
        return primaryOracle.price();
    }

    /// @inheritdoc IMetaOracleDeviationTimelock
    function backupPrice() public view returns (uint256) {
        return backupOracle.price();
    }

    /// @notice Checks if the primary oracle is currently selected.
    function isPrimary() public view returns (bool) {
        return currentOracle == primaryOracle;
    }

    /// @notice Checks if the backup oracle is currently selected.
    function isBackup() public view returns (bool) {
        return currentOracle == backupOracle;
    }

    /// @notice Checks if a challenge is currently active.
    function isChallenged() public view returns (bool) {
        return challengeExpiresAt > 0;
    }

    /// @notice Checks if a healing period is currently active.
    function isHealing() public view returns (bool) {
        return healingExpiresAt > 0;
    }

    /// @notice Calculates the absolute relative deviation between primary and backup oracles.
    /// @dev Deviation is calculated as `abs(primaryPrice - backupPrice) * 1e18 / backupPrice`.
    /// Returns 0 if backupPrice is 0 and primaryPrice is 0.
    /// Returns type(uint256).max if backupPrice is 0 and primaryPrice is non-zero.
    function getDeviation() public view returns (uint256) {
        uint256 currentPrimaryPrice = primaryOracle.price();
        uint256 currentBackupPrice = backupOracle.price();

        if (currentBackupPrice == 0) {
            return currentPrimaryPrice == 0 ? 0 : type(uint256).max;
        }

        uint256 diff;
        if (currentPrimaryPrice >= currentBackupPrice) {
            diff = currentPrimaryPrice - currentBackupPrice;
        } else {
            diff = currentBackupPrice - currentPrimaryPrice;
        }

        // Use uint256 for intermediate multiplication to avoid overflow before division
        return (diff * 10**18) / currentBackupPrice;
    }

    /// @notice Checks if the deviation exceeds the configured threshold.
    function isDeviant() public view returns (bool) {
        return getDeviation() > deviationThreshold;
    }

    /// @notice Initiates a challenge if the primary oracle is active and deviation threshold is exceeded.
    /// @dev Starts a timelock period (`challengeTimelockDuration`).
    function challenge() external {
        require(isPrimary(), "MODT: Must be primary oracle");
        require(!isChallenged(), "MODT: Already challenged");
        require(isDeviant(), "MODT: Deviation threshold not met");

        challengeExpiresAt = block.timestamp + challengeTimelockDuration;
        emit ChallengeStarted(challengeExpiresAt);
    }

    /// @notice Revokes an active challenge if the deviation is no longer present.
    function revokeChallenge() external {
        require(isPrimary(), "MODT: Must be primary oracle"); // Should still be primary
        require(isChallenged(), "MODT: Not challenged");
        require(!isDeviant(), "MODT: Deviation threshold still met");

        challengeExpiresAt = 0;
        emit ChallengeRevoked();
    }

    /// @notice Checks if the challenge has expired.
    function hasChallengeExpired() public view returns (bool) {
        return isChallenged() && block.timestamp >= challengeExpiresAt;
    }

    /// @notice Checks if the challenge can be accepted.
    function canAcceptChallenge() public view returns (bool) {
        return isPrimary() && isChallenged() && block.timestamp >= challengeExpiresAt && isDeviant();
    }

    /// @notice Accepts the challenge after the timelock expires, switching to the backup oracle.
    /// @dev Requires the deviation to still be present.
    function acceptChallenge() external {
        require(isPrimary(), "MODT: Must be primary oracle");
        require(isChallenged(), "MODT: Not challenged");
        require(block.timestamp >= challengeExpiresAt, "MODT: Challenge timelock not passed");
        require(isDeviant(), "MODT: Deviation resolved"); // Deviation must persist

        currentOracle = backupOracle;
        challengeExpiresAt = 0;
        emit ChallengeAccepted(address(currentOracle));
    }

    /// @notice Initiates the healing process if the backup oracle is active and prices have reconverged.
    /// @dev Starts a timelock period (`healingTimelockDuration`).
    function heal() external {
        require(isBackup(), "MODT: Must be backup oracle");
        require(!isHealing(), "MODT: Already healing");
        require(!isDeviant(), "MODT: Deviation threshold still met");

        healingExpiresAt = block.timestamp + healingTimelockDuration;
        emit HealingStarted(healingExpiresAt);
    }

    /// @notice Revokes an active healing process if the deviation threshold is exceeded again.
    function revokeHealing() external {
        require(isBackup(), "MODT: Must be backup oracle"); // Should still be backup
        require(isHealing(), "MODT: Not healing");
        require(isDeviant(), "MODT: Deviation threshold not met");

        healingExpiresAt = 0;
        emit HealingRevoked();
    }

    /// @notice Checks if the healing has expired.
    function hasHealingExpired() public view returns (bool) {
        return isHealing() && block.timestamp >= healingExpiresAt;
    }

    /// @notice Checks if the healing can be accepted.
    function canAcceptHealing() public view returns (bool) {
        return isBackup() && isHealing() && block.timestamp >= healingExpiresAt && !isDeviant();
    }

    /// @notice Accepts the healing after the timelock expires, switching back to the primary oracle.
    /// @dev Requires the prices to still be converged (not deviant).
    function acceptHealing() external {
        require(isBackup(), "MODT: Must be backup oracle");
        require(isHealing(), "MODT: Not healing");
        require(block.timestamp >= healingExpiresAt, "MODT: Healing timelock not passed");
        require(!isDeviant(), "MODT: Deviation occurred"); // Prices must remain converged

        currentOracle = primaryOracle;
        healingExpiresAt = 0;
        emit HealingAccepted(address(currentOracle));
    }
}
```

上述预言机的代码逻辑并不复杂。预言机内包含两个预言机，分别被称为:

1. `primaryOracle` 核心预言机，主要负责报价工作
2. `backupOracle` 备用预言机，当主预言机与备用预言机的价差过大时，可以通过挑战将预言机切换为备用预言机

> 代码内的 `currentOracle` 的数值可能是 `primaryOracle` 或是 `backupOracle`。另外需要注意的是，假如 `currentOracle` 的 `price` 函数出现问题，那么会自动切换为另一个预言机请求报价

我们可以将上述预言机视为如下状态机:

![MetaOracle Deviation Timelock](https://img.gopic.xyz/metdaoracletimelock.svg)

其中，`challenge` 开启的条件如下:

1. `currentOracle == primaryOracle`
2. 当前并不属于挑战时间锁(`ChallengeTimeLock`) 内部
3. `primaryOracle` 和 `backupOracle` 价格偏差大于最初设置

当 `challenge` 挑战时间锁内出现`primaryOracle` 和 `backupOracle` 价格偏差恢复的情况，可以调用 `revokeChallenge` 撤销挑战。当挑战时间锁完成并且此时价格偏差仍未恢复，可以调用 `acceptChallenge` 将 `currentOracle` 切换为 `backupOracle`。

当然，我们也可以在 `primaryOracle` 与 `backupOracle` 报价偏差恢复后，将预言机切换为 `primaryOracle`，该流程被称为 `heal` 过程，其过程与 `challenge` 基本类似，此处不再赘述。

简单来说，`MetaOracleDeviationTimelock` 合约允许用户设置 `primaryOracle` 和 `backupOracle` 进行报价。当 `MetaOracleDeviationTimelock` 的 `price` 函数被调用时，`currentOracle` 会返回报价，但假如 `currentOracle` 报价失败，那么会立即使用另一个预言机的报价返回。

同时，该预言机也允许随着 `primaryOracle` 和 `backupOracle` 报价价差的情况，进行预言机切换，当报价价差过大且时间锁完成时，`currentOracle` 会被切换为 `primaryOracle`;反之，当预言机之间的价差恢复时，`currentOracle` 会被切换为 `backupOracle`。


开发者为 `PT-USDe-25SEP2025` 设置了以下预言机：

1. `primaryOracle` 对应的地址是 `0xA5d21a73312221c1C7D94702316c02B2485036B9`
2. `backupOracle` 对应的地址是 `0x0DE0eCde53eA9d38351b2CE0635430506c0AA330`

这两个地址均部署了标准的 `MorphoChainlinkOracleV2` 预言机，我们会在后文分析这两个预言机的配置情况。

### Primary Oracle

在 Primary Oracle 内部，开发者使用了 Morpho 的标准预言机，并只设置了 `BASE_FEED_1`，设置的地址是 `0x93F4E501ddcB7e8FD18f5839204AeC0234B2B2E0`。该地址内部署了 `PendleChainlinkOracle` 合约，该合约的源代码可以在 [pendle-core-v2-public](https://github.com/pendle-finance/pendle-core-v2-public/blob/main/contracts/oracles/PtYtLpOracle/chainlink/PendleChainlinkOracle.sol) 内找到，该合约的核心部分代码如下:

```solidity
function(IPMarket, uint32) internal view returns (uint256) private immutable _getRawPendlePrice;

constructor(address _market, uint32 _twapDuration, PendleOracleType _baseOracleType) {
    factory = msg.sender;
    market = _market;
    twapDuration = _twapDuration;
    baseOracleType = _baseOracleType;
    (uint256 fromTokenDecimals, uint256 toTokenDecimals) = _readDecimals(_market, _baseOracleType);
    (fromTokenScale, toTokenScale) = (10 ** fromTokenDecimals, 10 ** toTokenDecimals);
    _getRawPendlePrice = _getRawPendlePriceFunc();
}

// =================================================================
//                          PRICING FUNCTIONS
// =================================================================

function _getPendleTokenPrice() internal view returns (int256) {
    return _descalePrice(_getRawPendlePrice(IPMarket(market), twapDuration));
}

function _descalePrice(uint256 price) private view returns (int256 unwrappedPrice) {
    return PMath.Int((price * fromTokenScale) / toTokenScale);
}

// =================================================================
//                          USE ONLY AT INITIALIZATION
// =================================================================

function _getRawPendlePriceFunc()
    internal
    view
    returns (function(IPMarket, uint32) internal view returns (uint256))
{
    if (baseOracleType == PendleOracleType.PT_TO_SY) {
        return PendlePYOracleLib.getPtToSyRate;
    } else if (baseOracleType == PendleOracleType.PT_TO_ASSET) {
        return PendlePYOracleLib.getPtToAssetRate;
    } else {
        revert("not supported");
    }
}

function _readDecimals(
    address _market,
    PendleOracleType _oracleType
) internal view returns (uint8 _fromDecimals, uint8 _toDecimals) {
    (IStandardizedYield SY, , ) = IPMarket(_market).readTokens();

    uint8 syDecimals = SY.decimals();
    (, , uint8 assetDecimals) = SY.assetInfo();

    if (_oracleType == PendleOracleType.PT_TO_ASSET) {
        return (assetDecimals, assetDecimals);
    } else if (_oracleType == PendleOracleType.PT_TO_SY) {
        return (assetDecimals, syDecimals);
    }
}
```

在以上代码内，我们可以看到一种并不常见的智能合约编程技术，即 solidity 内的函数式编程方法。在 solidity 内，实际上允许函数返回值是另一个函数。此处在构造过程中调用的 `_getRawPendlePriceFunc` 就是一个这种特殊函数，其返回值仍是一个函数，并且该函数被初始化到了 `_getRawPendlePrice` 内部。此处 `_getRawPendlePrice` 在 `PendleOracleType.PT_TO_ASSET` 情况下的实现如下:

```solidity
/**
 * This function returns the twap rate PT/Asset on market, but take into account the current rate of SY
 This is to account for special cases where underlying asset becomes insolvent and has decreasing exchangeRate
 * @param market market to get rate from
 * @param duration twap duration
 */
function getPtToAssetRate(IPMarket market, uint32 duration) internal view returns (uint256) {
    (uint256 syIndex, uint256 pyIndex) = getSYandPYIndexCurrent(market);
    if (syIndex >= pyIndex) {
        return getPtToAssetRateRaw(market, duration);
    } else {
        return (getPtToAssetRateRaw(market, duration) * syIndex) / pyIndex;
    }
}

/// @notice returns the raw rate without taking into account whether SY is solvent
function getPtToAssetRateRaw(IPMarket market, uint32 duration) internal view returns (uint256) {
    uint256 expiry = market.expiry();

    if (expiry <= block.timestamp) {
        return PMath.ONE;
    } else {
        uint256 lnImpliedRate = getMarketLnImpliedRate(market, duration);
        uint256 timeToExpiry = expiry - block.timestamp;
        uint256 assetToPtRate = MarketMathCore._getExchangeRateFromImpliedRate(lnImpliedRate, timeToExpiry).Uint();
        return PMath.ONE.divDown(assetToPtRate);
    }
}

function getMarketLnImpliedRate(IPMarket market, uint32 duration) internal view returns (uint256) {
    uint32[] memory durations = new uint32[](2);
    durations[0] = duration;

    uint216[] memory lnImpliedRateCumulative = market.observe(durations);
    return (lnImpliedRateCumulative[1] - lnImpliedRateCumulative[0]) / duration;
}
```

上述代码使用 `getMarketLnImpliedRate` 函数从 `market` 内读取 TWAP 数据，然后使用 `(lnImpliedRateCumulative[1] - lnImpliedRateCumulative[0]) / duration` 将其转化为该段时间内平均数。

Primary Oracle 存在的问题是 Pendle 预言机返回的实际上是 `PT-USDe-25SEP2025 / USDe` 的报价，所以 Primary Oracle 假设了 `USDe` 和 `USDC` 之间的价格不会脱锚。我们会在 Backup Oracle 内解决此问题

### Backup Oracle

Backup Oracle 也使用了标准的 Morpho 预言机实现，并设置了以下参数:

1. `BASE_FEED_1` 也使用了 Pendle 的 `PT-USDe-25SEP2025 / USDe` 报价预言机，实际上就是上文介绍的 `0x93F4E501ddcB7e8FD18f5839204AeC0234B2B2E0`
2. `BASE_FEED_2` 使用了 Chainlink 的 `USDe / USD ` 预言机，用来处理可能的 USDe 脱锚问题，地址是 `0xa569d910839Ae8865Da8F8e70FfFb0cBA869F961`，返回的报价精度为 8
3. `QUOTE_FEED_1` 使用了 `DummyFeed`，该预言机只用于修正数值精度

为什么需要 `DummyFeed` 修正精度? 在 MorphoChainlinkOracleV2 内存在以下代码:

```solidity
SCALE_FACTOR = 10
    ** (
        36 + quoteTokenDecimals + quoteFeed1.getDecimals() + quoteFeed2.getDecimals() - baseTokenDecimals
            - baseFeed1.getDecimals() - baseFeed2.getDecimals()
    ) * quoteVaultConversionSample / baseVaultConversionSample;
    }
```

在此案例中，`quoteTokenDecimals` 是 USDC 的精度，即 `quoteTokenDecimals = 6`，而 `baseTokenDecimals` 是 USDe 的精度，即 `baseTokenDecimals = 18`，而 `baseFeed1.getDecimals` 的返回值是 18，且 `baseFeed2.getDecimals()` 的返回值是 8。

如果不考虑 `QUOTE_FEED_1` 的精度，那么 `SCALE_FACTOR` 的计算结果是 $10^{36 + 6 - 18 - 18 - 8} = 10^{-2}$。显然，该计算由于出现负数会导致计算溢出。解决方案是设置 `quoteFeed1` 的 `decimals`。在此案例中，我们将 `quoteFeed1` 的 `decimals` 设置为 12，由此避免计算 `SCALE_FACTOR` 过程中的下溢。但为了避免 `quoteFeed1` 影响最后的结果，我们将 `quoteFeed1` 的 `price` 设置为 `1e12`。

实际上，我们可以将 `quoteFeed1` 设置的精度设置为大于 2 的某一个数字，该预言机存在只是为了避免计算 `SCALE_FACTOR` 过程中的下溢。

## [PT-cUSDO-20NOV2025 / USDC](https://app.morpho.org/ethereum/market/0x8a71a66ac828c2b6d4f8accce5859aba0822b502f3833bec4aff09479affffdb/pt-cusdo-20nov2025-usdc?subTab=advanced)

对于 `PT-cUSDO-20NOV2025 / USDC` 市场，开发者使用了 `0xCE84fD399620B7D20e18AB6009Fc8A0cdaad880f` 作为该市场的预言机，该合约是一个可升级合约，其实现代码位于 `0x04236557c2dac11175638a598b28a9b22fdd8d06` 内部，合约被命名为 `EOPendlePTFeedHybrid`。该合约的计算价格使用了 `MAX(1, MIN(LinearDiscount, TWAP PTtoAsset))` 公式，混合了 Pendle 的 TWAP 报价和线性折扣计算。

其中 Pendle TWAP 报价使用了如下函数获取数据:

```solidity
function _getPtToAssetTWAPRate() internal view returns (uint256) {
    return IPTOracle(ptOracle).getPtToAssetRate(ptMarket, twapDuration) * twapNumeratorMultiplier
        / twapDenominatorMultiplier;
}
```

而线性折扣计算使用了如下代码:

```solidity
function _getLinearDiscountRate() internal view returns (uint256) {
    uint256 timeLeft = (maturity > block.timestamp) ? maturity - block.timestamp : 0;
    uint256 discount = (timeLeft * baseDiscountPerYear) / SECONDS_PER_YEAR;

    if (discount > ONE) revert DiscountOverflow();

    return ONE - discount;
}
```

## [AAVE PT USDe September 2025](https://app.aave.com/reserve-overview/?underlyingAsset=0xbc6736d346a5ebc0debc997397912cd9b8fae10a&marketName=proto_mainnet_v3)

最近 AAVE 也增加了 PT 作为担保品的选项，此处我们对 `AAVE PT USDe September 2025` 市场对预言机进行简单分析，该预言机的地址是 `0x8B17C02d22EE7D6B8D6829ceB710A458de41E84a`。其核心代码如下:

```solidity
  /// @inheritdoc ICLSynchronicityPriceAdapter
  function latestAnswer() external view returns (int256) {
    int256 currentAssetPrice = ASSET_TO_USD_AGGREGATOR.latestAnswer();
    if (currentAssetPrice <= 0) {
      return 0;
    }

    uint256 price = (uint256(currentAssetPrice) * (PERCENTAGE_FACTOR - getCurrentDiscount())) /
      PERCENTAGE_FACTOR;

    return int256(price);
  }
  /// @inheritdoc IPendlePriceCapAdapter
  function getCurrentDiscount() public view returns (uint256) {
    uint256 timeToMaturity = (MATURITY > block.timestamp) ? MATURITY - block.timestamp : 0;

    return (timeToMaturity * discountRatePerYear) / SECONDS_PER_YEAR;
  }
```

与上文出现的预言机类似，该预言机也使用了简单的线性计算当前 PT 的价格，但有趣的是，AAVE 并不假设 PT 的底层资产 USDe 与 USD 不脱锚，所以引入了 `ASSET_TO_USD_AGGREGATOR` 的选项来处理可能的脱锚。但 AAVE 在此处的 `ASSET_TO_USD_AGGREGATOR` 实际上使用是 `0x3E7d1eAB13ad0104d2750B8863b489D65364e32D` 预言机，该预言机是 `USDT / USD` 预言机，也就是说 USDe 脱锚并不会导致 PT 预言机价格下降，而是 USDT 脱锚会导致 PT 预言机价格下降。

> `ASSET_TO_USD_AGGREGATOR` 另一个有趣的特性是该预言机存在报价上限，即 USDT / USD 价格不会大于 1.04

## 总结

在 Morpho 内，我们可以观察到以下三种不同的预言机配置:

1. 根据利率线性计算 PT 价格，SparkDAO 管理的市场基本都使用了这种类似的预言机
2. 使用 Pendle 官方 TWAP 预言机作为主预言机，同时配置 TWAP 预言机和底层资产价格预言机作为备用预言机，这种方案有效避免了底层资产脱锚的风险，同时没有显著提高 gas 成本，Steakhouse 设计了这套方案
3. 使用 Pendle 官方 TWAP 预言机和利率线性计算，使用较小的数值

在 AAVE 内，我们主要观察到使用了 利率线性计算 PT 价格 的预言机。

一个有趣的观察是很多预言机都没有考虑底层资产脱锚的情况，而 AAVE 奇怪的使用了 USDT 脱锚衡量 USDe 脱锚。