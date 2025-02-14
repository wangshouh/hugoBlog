---
title: "现代 DeFi: Uniswap V4"
date: 2025-01-01T01:00:00Z
tags: [defi]
math: true
---

## 概述

在上一篇博客内，我们详细介绍了 Uniswap V3 的基础原理。在本篇博客内，我们将继续介绍与 Uniswap V3 差别不大的 Uniswap v4 的原理及代码。需要注意的，Uniswap v4 的 AMM 曲线部分与 Uniswap v3 是一致的，但增加了一些优化的业务逻辑。本文将跳过所有与 Uniswap v3 类似的部分，只介绍 Uniswap v4 的特性。

本文将主要介绍 Uniswap V4 的以下特性：

1. Singleton Pool。所有的代币的交换池都位于单一合约内部，降低了在链式兑换时的 gas 消耗
2. Flash Accounting。一个用于跟踪交易过程中净流入和净流出代币的系统，只需要用户保证一笔交易中最后的代币平衡即可，并不需要向 Uniswap v3 一样保持交易中每一次 swap 调用的平衡
3. Hooks。最为知名的特性，为 Uniswap v4 增加了巨大的可拓展性
4. ERC 6909。一个类似 ERC1155 的协议，使用该 ERC 标准实现了用户可以将部分资产存入 Uniswap V4 内部已实现更加高效的兑换

## ERC 6909

与其他文章不同，本文首先介绍 Uniswap V4 目前最不为大众所熟知的 ERC6909 协议。ERC 6909 也被 Uniswap V4 的核心合约 `PoolManager` 所继承。

ERC6909 在接口上与 ERC1155 是类似的。该合约可以实现多种 ERC20 代币存放在同一个合约内，并在同一个合约内管理。比如用户可以使用 `function balanceOf(address owner, uint256 id) external view returns (uint256 amount);` 函数查询某一个地址名下的某一个 ERC20 代币在 ERC6909 内的持有数量。而 `function allowance(address owner, address spender, uint256 id) external view returns (uint256 amount);` 则可以查询地址之间的对某一个资产的授权数量。

ERC6909 类似包装代币，一种最简单且精准的类比是认为 ERC6909 是一种将 ERC20 代币包装为 ERC1155 代币的方案。当然，我们也可以看到 ERC6909 依靠单个合约管理多种资产的思想也类似 Uniswap V4 的 Singleton Pool 的设计思路。

Uniswap V4 在标准的 ERC6909 接口外，还提供了 `mint` 函数:

```solidity
function mint(address to, uint256 id, uint256 amount) external onlyWhenUnlocked {
    unchecked {
        Currency currency = CurrencyLibrary.fromId(id);
        // negation must be safe as amount is not negative
        _accountDelta(currency, -(amount.toInt128()), msg.sender);
        _mint(to, currency.toId(), amount);
    }
}
```

我们先不关心函数内的其他部分，可以看到该函数实现了 `_mint` 方法，可以为用户铸造某种代币的 ERC6909 的包装版本。

> 上述函数并没有具体的代币转入流程，这是因为 Uniswap V4 引入了 **Flash Accounting** 机制，我们会在合约的闪存(`transient storage`) 内存入代币的盈亏数值，并在交易完成时做最终结算。我们可以看到此处使用 `_accountDelta(currency, -(amount.toInt128()), msg.sender);` 语句在为用户减少了一部分资产。

当然，Uniswap V4 也提供了解除包装的函数，该函数被称为 `burn` 函数:

```solidity
function burn(address from, uint256 id, uint256 amount) external onlyWhenUnlocked {
    Currency currency = CurrencyLibrary.fromId(id);
    _accountDelta(currency, amount.toInt128(), msg.sender);
    _burnFrom(from, currency.toId(), amount);
}
```

与上述 `mint` 函数相反，此处使用 `_accountDelta(currency, amount.toInt128(), msg.sender);` 增加了用户的可用资产。

此处我们可以讨论一下上述函数中的 `CurrencyLibrary` 中的 `toId` 和 `fromId` 函数，前者用于代币地址向 ERC6909 的 ID 转化，而后者则用于 ID 向地址的转换。上述代码都定义在 `src/types/Currency.sol` 内部，这部分代码是相当有意思的，因为 Uniswap 使用了大量的 solidity 新语法。

在 `Currency.sol` 的开始部分，我们就可以看到如下代码:

```solidity
type Currency is address;

using {greaterThan as >, lessThan as <, greaterThanOrEqualTo as >=, equals as ==} for Currency global;
using CurrencyLibrary for Currency global;

function equals(Currency currency, Currency other) pure returns (bool) {
    return Currency.unwrap(currency) == Currency.unwrap(other);
}

function greaterThan(Currency currency, Currency other) pure returns (bool) {
    return Currency.unwrap(currency) > Currency.unwrap(other);
}

function lessThan(Currency currency, Currency other) pure returns (bool) {
    return Currency.unwrap(currency) < Currency.unwrap(other);
}

function greaterThanOrEqualTo(Currency currency, Currency other) pure returns (bool) {
    return Currency.unwrap(currency) >= Currency.unwrap(other);
}
```

此处我们首先使用了 `type Currency is address;` 为 `address` 类型创建别名。然后使用 `using A for B` 的语法为 `Currency` 类型增加了一些特性。需要注意的，此处使用了操作运算符的定义，所以在最后增加了 `global` 关键词。

之后的代码中出现了 `Currency.unwrap` 函数，该函数也是新版本 solidity 为用户自定义类型增加的新特性。在较新版本的 solidity 中，假如用户定义了 `type C is V` ，其中 `C` 是用户自定义类型，而 `V` 是 solidity 提供的原始类型，那么用户可以使用 `C.wrap` 将 `V` 类型转化为 `C` 类型，也可以使用 `C.unwrap` 将 `V` 类型转化为 `C` 类型。使用这种方法的好处是，solidity 会提供默认的类型检查，避免错误的类型转化发生。

Uniswap v4 此处的代码就显示了在新版本 solidity 下，开发者该如何定义自己的类型以及为类型重载运算符。如果读者希望进一步了解相关内容，可以参考 [User-defined Value Types](https://docs.soliditylang.org/en/v0.8.28/types.html#user-defined-value-types) 部分的文档。

在 `CurrencyLibrary` 内部，Uniswap V4 定义了以下函数：

```solidity
function toId(Currency currency) internal pure returns (uint256) {
    return uint160(Currency.unwrap(currency));
}

// If the upper 12 bytes are non-zero, they will be zero-ed out
// Therefore, fromId() and toId() are not inverses of each other
function fromId(uint256 id) internal pure returns (Currency) {
    return Currency.wrap(address(uint160(id)));
}
```

这两段函数较为简单，但注释指出 `fromId()` 和 `toId()` 并不是完全互逆的。这是因为假如用户给出了一个特殊的 `id`，比如给定 `0x1000000000000000000000001f9840a85d5af5bf1d1762f925bdaddc4201f984` 的 ID，那么由于 `uint160(id)` 会截断大于 160 bit 的部分，所以最终返回的结果会是 `0x1f9840a85d5af5bf1d1762f925bdaddc4201f984` 代币地址。所以存在多个 ID 对应同一种代币的情况。故此，Uniswap v4 注释内指出 `fromId` 和 `toId` 并不是完全互逆关系。

## ERC 7751 和 Custom Revert

为了优化异常的抛出，Uniswap v4 引入了 ERC 7751 协议，并编写了 `CustomRevert` 库来处理 Revert。所谓的 ERC 7751 是一种带有上下文的错误抛出方法。一个简单的应用场景是当 Uniswap V4 出现报错后，在之前的情况下，我们不知道是 Uniswap v4 的主合约给出的报错还是因为 ERC20 代币实现给出的报错，我们只能通过 trace 的方式发现调用出错的位置。但 ERC 7751 允许我们给出报错的合约地址以及相关的上下文，这可以使得开发者不在 trace 的情况下就可以快速判断问题。

上述功能在 `src/libraries/CustomRevert.sol` 内的 `bubbleUpAndRevertWith` 函数内进行了实现:

```solidity
/// @notice bubble up the revert message returned by a call and revert with a wrapped ERC-7751 error
/// @dev this method can be vulnerable to revert data bombs
function bubbleUpAndRevertWith(
    address revertingContract,
    bytes4 revertingFunctionSelector,
    bytes4 additionalContext
) internal pure {
    bytes4 wrappedErrorSelector = WrappedError.selector;
    assembly ("memory-safe") {
        // Ensure the size of the revert data is a multiple of 32 bytes
        let encodedDataSize := mul(div(add(returndatasize(), 31), 32), 32)

        let fmp := mload(0x40)

        // Encode wrapped error selector, address, function selector, offset, additional context, size, revert reason
        mstore(fmp, wrappedErrorSelector)
        mstore(add(fmp, 0x04), and(revertingContract, 0xffffffffffffffffffffffffffffffffffffffff))
        mstore(
            add(fmp, 0x24),
            and(revertingFunctionSelector, 0xffffffff00000000000000000000000000000000000000000000000000000000)
        )
        // offset revert reason
        mstore(add(fmp, 0x44), 0x80)
        // offset additional context
        mstore(add(fmp, 0x64), add(0xa0, encodedDataSize))
        // size revert reason
        mstore(add(fmp, 0x84), returndatasize())
        // revert reason
        returndatacopy(add(fmp, 0xa4), 0, returndatasize())
        // size additional context
        mstore(add(fmp, add(0xa4, encodedDataSize)), 0x04)
        // additional context
        mstore(
            add(fmp, add(0xc4, encodedDataSize)),
            and(additionalContext, 0xffffffff00000000000000000000000000000000000000000000000000000000)
        )
        revert(fmp, add(0xe4, encodedDataSize))
    }
}
```

上述代码使用了 yul 汇编完成，虽然看上去很复杂，但实际上很简单。上述代码本质上是抛出了以下错误的 ABI 编码版本:

```solidity
error WrappedError(address target, bytes4 selector, bytes reason, bytes details);
```

在 `bubbleUpAndRevertWith` 函数内，`details` 只是一个 `bytes4 additionalContext` 类型。上述代码本质上是在完成了 `WrappedError` 的 ABI 编码工作。我们可以分部分来研究上述代码的功能:

第一部分是用来计算调用合约的返回值占据的字节数:

```
let encodedDataSize := mul(div(add(returndatasize(), 31), 32), 32)
```

在 `WrappedError` 内，我们会给出调用其他合约返回的报错内容，这部分报错内容都存储在 call 之后的返回值内，我们可以通过 `returndatasize` 获得这部分错误返回值的大小。由于 solidity 内都以 32 bytes 作为基础单位，所以此处我们需要将 `returndatasize` 计算为 32 的倍数。具体逻辑实现上，我们首先增加将 `returndatasize` 增加 `31` ，这是为了处理 `returndatasize = 1` 这种情况。即使 `returndatasize = 1`，使用上述代码仍可以计算出 `32` 的大小。其他的先除(`div` )后乘(`mul`) 是一种典型的将每一个数字重整为另一个数字倍数的方法。我们可以在 Uniswap v3 内的 `tickSpacingToMaxLiquidityPerTick` 函数内看到类似的操作。

第二部分使用 `let fmp := mload(0x40)` 语句获取当前 solidity 的空闲内存地址的起点。solidity 编译器使用了动态的内存布局方案，每次使用或者释放内存后，都会修改 `0x40` 内的数据，将其指向当前未使用的空闲内存的起点。在大部分内存安全的代码方案内，我们都会 `mload(0x40)` 读取空闲内存起点，避免后续占用到已被其他函数使用到的内存造成安全问题。

第三部分用来写入 `WrappedError` 错误选择器以及报错的合约地址。

```
mstore(fmp, wrappedErrorSelector)
mstore(add(fmp, 0x04), and(revertingContract, 0xffffffffffffffffffffffffffffffffffffffff))
```

与 solidity 定义的 `log` 方法一致，solidity 内抛出错误实际上也会计算错误的选择器，然后将其作为最初的 4 bytes。此处的 `wrappedErrorSelector` 就首先被写入了内存，然后跳过 `wrappedErrorSelector` 占据的最初 4 bytes(`add(fmp, 0x04)` 就是跳过 `wrappedErrorSelector` 占据的字节)，我们接下来需要写入给出报错的地址。根据 solidity 的彬吗规范，给出报错的地址应该占据 256 bit。此处使用 `and(revertingContract, 0xffffffffffffffffffffffffffffffffffffffff))` 清理了 `revertingContract` 变量可能存在的高位垃圾，然后将其写入内存。

此时的内存结构如下:

```
wrappedErrorSelector (4 bytes) + revertingContract(32 bytes)
```

第四部分写入出现报错的 `selector` 和错误函数的返回值。比如 Uniswap V4 调用 ERC20 代币的 `transfer` 失败，那么此处就写入 `transfer` 的选择器和返回的错误信息:

```
mstore(
    add(fmp, 0x24),
    and(revertingFunctionSelector, 0xffffffff00000000000000000000000000000000000000000000000000000000)
)
```

上述代码完成了 `selector` 的写入，因为 `selector` 的类型是 `bytes4`，所以此处直接写入内存即可，但注意 `bytes4` 在 ABI 编码后会转化为 `uint256` 。我们需要计算写入时需要跳过的内存长度。我们需要跳过 `wrappedErrorSelector (4 bytes) + revertingContract(32 bytes)` 的内存结构，该结构的长度为 36(`0x24`)，所以我们此处使用 `add(fmp, 0x24)` 确定了起始位置，然后写入了 `revertingFunctionSelector`，此处也需要注意使用与`0xffffffff00000000000000000000000000000000000000000000000000000000` 进行 `and` 操作去掉其他垃圾位。

完成上述操作后，内存结构如下:

```
wrappedErrorSelector (4 bytes) 
+ revertingContract(32 bytes) 
+ revertingFunctionSelector(32 bytes)
```

第五部分内，我们写入动态类型的 offset。因为我们最后写入的 `reason` 和 `detail` 都是 `bytes` 类型，该类型要求写入动态类型的起始位置。在后文内，我们称之为 `reason offset` 和 `detail offset` 。代码如下：

```
// offset revert reason
mstore(add(fmp, 0x44), 0x80)
// offset additional context
mstore(add(fmp, 0x64), add(0xa0, encodedDataSize))
```

此处，我们首先确定 `reason offset` 的写入位置 ，我们目前的内存已占用了 `4 + 32 + 32 = 68`，所以使用 `add(fmp, 0x44)` 跳过之前已写入的数据。而关于 `bytes` 动态类型的起始位置计算则需要排除 `wrappedErrorSelector (4 bytes)`，这一点额外重要，即计算动态类型的 offset 不需要包括选择器的占用部分。所以计算获得的 `reason offset` 应该为 `revertingContract(32 bytes) + revertingFunctionSelector(32 bytes) + reason offset(32 bytes) + details offset(32 bytes)`。我们使用 `mstore(add(fmp, 0x44), 0x80)` 写入 `reason offset`。

接下来，我们需要 `detail offset`，该变量的写入位置为 `wrappedErrorSelector (4 bytes) + revertingContract(32 bytes) + revertingFunctionSelector(32 bytes) + reason offset(32 bytes) = 0x64`，而写入的数值是 `add(0xa0, encodedDataSize)`。该数值中的 `0xa0` 包含以下部分 `revertingContract(32 bytes) + revertingFunctionSelector(32 bytes) + reason offset(32 bytes) + reason length(32 bytes) + detail offset(32 bytes) = 160`，除此之外还包含 `encodedDataSize` 部分。

> 在具体的编码过程中，我们往往最后确认 offset 的数值，所以动态类型的 offset 并没有非常难以确认

上述代码编写完成后，内存布局如下:

```
wrappedErrorSelector (4 bytes) 
+ revertingContract(32 bytes) 
+ revertingFunctionSelector(32 bytes)
+ reason offset(32 bytes)
+ detail offset(32 bytes)
```

接下来，我们可以写入 `reason` 的真实内容。此处需要注意写入 bytes 类型的内容，需要在原有内容前增加长度信息。我们先完成 `reason` 的长度写入然后完成具体的内容写入:

```
// size revert reason
mstore(add(fmp, 0x84), returndatasize())
// revert reason
returndatacopy(add(fmp, 0xa4), 0, returndatasize())
```

此处使用 `returndatacopy` 将失败的调用的返回结果写入内存。读者可以自行阅读 [文档](https://www.evm.codes/?fork=cancun#3e) 获得 `returndatacopy` 的参数。

完成上述操作后内存布局为:

```
wrappedErrorSelector (4 bytes) 
+ revertingContract(32 bytes) 
+ revertingFunctionSelector(32 bytes)
+ reason offset(32 bytes)
+ detail offset(32 bytes)
+ reason length(32 bytes)
+ return data(encodedDataSize)
```

最后，我们写入 `detail` 的内容。Uniswap v4 约定 `detail` 是一个 `bytes4 additionalContext` 参数，所以长度确定为 4。相关代码如下:

```
// size additional context
mstore(add(fmp, add(0xa4, encodedDataSize)), 0x04)
// additional context
mstore(
    add(fmp, add(0xc4, encodedDataSize)),
    and(additionalContext, 0xffffffff00000000000000000000000000000000000000000000000000000000)
)
```

此处也使用了 `and` 去掉垃圾位。完成上述所有步骤后，我们的内存布局如下:

```
wrappedErrorSelector (4 bytes) 
+ revertingContract(32 bytes) 
+ revertingFunctionSelector(32 bytes)
+ reason offset(32 bytes)
+ detail offset(32 bytes)
+ reason length(32 bytes)
+ return data(encodedDataSize)
+ detail length(32 bytes)
+ detail data(32 bytes)
```

完成上述所有工作后，我们直接使用 `revert(fmp, add(0xe4, encodedDataSize))` 抛出报错。代码内的 `0xe4` 实际上就是除了 `return data` 外，其他固定长度部分之和。

由于上述代码最后使用 `revert` 结束，所以此处我们没有进行内存的清理工作，但其他函数内，开发者如果使用内联汇编操作内存，应当考虑内存的清理工作。

在 `CurrencyLibrary` 库内，我们使用了上述函数报错 `transfer` 的错误：

```solidity
if (!success) {
    CustomRevert.bubbleUpAndRevertWith(
        Currency.unwrap(currency), IERC20Minimal.transfer.selector, ERC20TransferFailed.selector
    );
}
```

考虑到文章的长度，此处不会详细介绍 `CurrencyLibrary` 内给出的 `transfer` 函数的构造方法，读者可以自行阅读 yul 源代码。该函数最后就使用了以下代码清理内存:

```
// Now clean the memory we used
mstore(fmp, 0) // 4 byte `selector` and 28 bytes of `to` were stored here
mstore(add(fmp, 0x20), 0) // 4 bytes of `to` and 28 bytes of `amount` were stored here
mstore(add(fmp, 0x40), 0) // 4 bytes of `amount` were stored here
```

关于 `CustomRevert` 内的其他函数，都是一些简单的内联汇编代码，读者可以自行阅读。

## Unlock 与 Delta

在上文内，我们指出 Uniswap V4 使用了 Flash Accounting 机制，在一笔复杂的 swap 交易内，部分交易可以产生“亏空”，但只要最终可以实现平衡即可。比如在 USDT -> DAI -> USDC 的兑换路径内，我们可以先完成 DAI -> USDC 的兑换，虽然此时我们没有 DAI，但是 Uniswap V4 不会直接 revert 交易。但我们完成整条兑换路径后，Uniswap V4 会检查用户资产的平衡，此时如果没有平衡则会直接报错。

在这里存在一个核心问题，即 Uniswap V4 如何知道一笔交易是否结束。在此处，Uniswap V4 使用了一个特殊的 `unlock` 函数，该函数代码如下:

```solidity
function unlock(bytes calldata data) external override returns (bytes memory result) {
    if (Lock.isUnlocked()) AlreadyUnlocked.selector.revertWith();

    Lock.unlock();

    // the caller does everything in this callback, including paying what they owe via calls to settle
    result = IUnlockCallback(msg.sender).unlockCallback(data);

    if (NonzeroDeltaCount.read() != 0) CurrencyNotSettled.selector.revertWith();
    Lock.lock();
}
```

上述函数很简单，当用户调用 `unlock` 函数时，`unlock` 函数会解锁合约，然后使用 `callback` 回调请求合约，当 `unlockCallback` 结束后，`unlock` 函数会检查 `NonzeroDeltaCount` 的内容，如果用户在该笔交易的最后没有实现平衡，就会直接 revert 交易。在 `unlock` 函数的最后，我们会继续锁定合约。

在 Uniswap V4 合约内，`modifyLiquidity` 和 `swap` 等涉及到资金函数都会存在 `onlyWhenUnlocked` 的修饰函数，所以如果请求合约没有调用 `unlock` 函数，那么该合约就无法调用其他函数。所以任何与 Uniswap V4 交互的交易都需要首先调用 `unlock` 函数解锁合约，否则其他的调用都会无效。一旦调用 `unlock` 函数后，请求合约必须使用 `unlockCallback` 函数接受回调，请求合约需要在回调内处理 `swap` 等函数。`unlock` 函数最终使用 `NonzeroDeltaCount.read()` 保证交易过程中的平衡。

简单来说，`unlock` 函数使用 `unlock` 与 `callback` 逻辑的组合最终实现了在用户交易的结束检查账户的平衡。事实上，上述逻辑也可以实现防止重入的功能。

接下来，我们具体分析 `unlock` 内部的具体内容。`Lock` 是一个使用了 `transient storage` 进行存储的对象。该部分的代码如下(注意，以下代码位于 BUSL 授权下，不建议读者在自己的商业项目内直接摘抄，后文内出现的 BUSL 代码都会有此警告):

```solidity
library Lock {
    // The slot holding the unlocked state, transiently. bytes32(uint256(keccak256("Unlocked")) - 1)
    bytes32 internal constant IS_UNLOCKED_SLOT = 0xc090fc4683624cfc3884e9d8de5eca132f2d0ec062aff75d43c0465d5ceeab23;

    function unlock() internal {
        assembly ("memory-safe") {
            // unlock
            tstore(IS_UNLOCKED_SLOT, true)
        }
    }

    function lock() internal {
        assembly ("memory-safe") {
            tstore(IS_UNLOCKED_SLOT, false)
        }
    }

    function isUnlocked() internal view returns (bool unlocked) {
        assembly ("memory-safe") {
            unlocked := tload(IS_UNLOCKED_SLOT)
        }
    }
}
```

由于 Solidity 目前还没有高级语言对闪存( `transient storage` )的支持，所以此处只能使用 `yul` 汇编进行管理。此处的 `tstore` 就是向指定闪存存储槽写入数据，而 `tload` 则是在指定闪存存储槽内读取数据。此处选择的存储槽是 `bytes32(uint256(keccak256("Unlocked")) - 1)` ，此处使用 `keccak256("Unlocked")) - 1` 的原因是为了避免可能的存储槽碰撞。假如读者直接使用 ``bytes32(uint256(keccak256("Unlocked")))` 作为重要变量的存储位置，可能存在攻击者利用 `mapping` 类型通过哈希碰撞攻击的可能性。事实上，大部分自定义存储槽都会使用 `bytes32(uint256(keccak256(VAR NAME)) - 1)` 的形式，此处的 `VAR NAME` 就是该存储槽存储的变量名。

然后，我们可以分析 `NonzeroDeltaCount` 内的内容。`NonzeroDeltaCount` 内部记录了当前系统内没有被平衡的代币数量。`_accountDelta` 内部具有 `NonzeroDeltaCount` 的操作代码:

```solidity
function _accountDelta(Currency currency, int128 delta, address target) internal {
    if (delta == 0) return;

    (int256 previous, int256 next) = currency.applyDelta(target, delta);

    if (next == 0) {
        NonzeroDeltaCount.decrement();
    } else if (previous == 0) {
        NonzeroDeltaCount.increment();
    }
}
```

当某一个代币账户从零值转化为其他数值时，我们会增加 `NonzeroDeltaCount` 的数值，反之则减少 `NonzeroDeltaCount` 的数值。当 `NonzeroDeltaCount == 0` 时，则说明当前所有的代币都被平衡了。

此处的 `applyDelta` 函数是比较有趣的，该部分代码位于 `src/libraries/CurrencyDelta.sol` 内部，注意该部分代码也位于 BUSL 授权下，读者应该谨慎使用。代码如下:

```solidity
library CurrencyDelta {
    /// @notice calculates which storage slot a delta should be stored in for a given account and currency
    function _computeSlot(address target, Currency currency) internal pure returns (bytes32 hashSlot) {
        assembly ("memory-safe") {
            mstore(0, and(target, 0xffffffffffffffffffffffffffffffffffffffff))
            mstore(32, and(currency, 0xffffffffffffffffffffffffffffffffffffffff))
            hashSlot := keccak256(0, 64)
        }
    }

    function getDelta(Currency currency, address target) internal view returns (int256 delta) {
        bytes32 hashSlot = _computeSlot(target, currency);
        assembly ("memory-safe") {
            delta := tload(hashSlot)
        }
    }

    /// @notice applies a new currency delta for a given account and currency
    /// @return previous The prior value
    /// @return next The modified result
    function applyDelta(Currency currency, address target, int128 delta)
        internal
        returns (int256 previous, int256 next)
    {
        bytes32 hashSlot = _computeSlot(target, currency);

        assembly ("memory-safe") {
            previous := tload(hashSlot)
        }
        next = previous + delta;
        assembly ("memory-safe") {
            tstore(hashSlot, next)
        }
    }
}
```

此处的 `applyDelta` 内部首先使用 `_computeSlot` 计算了用户持有的某一个代币的所在闪存地址。此处使用的 `_computeSlot` 函数实现很有意思。简单来说，我们可以将上述代码转化为 `keccak256(target || currency)`，其中 `||` 是字节拼接函数(`concat`)。此处有读者好奇，此处标记为 `memory-safe`，但最终没有清理内存。这是因为 solidity 文档内约定 `0x00 - 0x3f (64 bytes): scratch space for hashing methods.`。简单来说 `0x00 - 0x3f` 这部分内存只是用来进行哈希计算的，所以我们可以不清理内存或者修改 `0x40` 空闲内存指针。

最后，我们可以简单看一下 `NonzeroDeltaCount` 内部给出的代码，这部分代码也位于 BUSL 授权下，读者应该谨慎使用:


```solidity
library NonzeroDeltaCount {
    // The slot holding the number of nonzero deltas. bytes32(uint256(keccak256("NonzeroDeltaCount")) - 1)
    bytes32 internal constant NONZERO_DELTA_COUNT_SLOT =
        0x7d4b3164c6e45b97e7d87b7125a44c5828d005af88f9d751cfd78729c5d99a0b;

    function read() internal view returns (uint256 count) {
        assembly ("memory-safe") {
            count := tload(NONZERO_DELTA_COUNT_SLOT)
        }
    }

    function increment() internal {
        assembly ("memory-safe") {
            let count := tload(NONZERO_DELTA_COUNT_SLOT)
            count := add(count, 1)
            tstore(NONZERO_DELTA_COUNT_SLOT, count)
        }
    }

    /// @notice Potential to underflow. Ensure checks are performed by integrating contracts to ensure this does not happen.
    /// Current usage ensures this will not happen because we call decrement with known boundaries (only up to the number of times we call increment).
    function decrement() internal {
        assembly ("memory-safe") {
            let count := tload(NONZERO_DELTA_COUNT_SLOT)
            count := sub(count, 1)
            tstore(NONZERO_DELTA_COUNT_SLOT, count)
        }
    }
}
```

以上代码也较为简单，不再赘述。

在 Uniswap v4 内部，还存在一些与 Delta 可以配合使用的函数。第一个是 `mint` 函数，与 Uniswap v3 不同，Uniswap v4 中的 `mint` 函数用于 ERC6909 的代币铸造:

```solidity
function mint(address to, uint256 id, uint256 amount) external onlyWhenUnlocked {
    unchecked {
        Currency currency = CurrencyLibrary.fromId(id);
        // negation must be safe as amount is not negative
        _accountDelta(currency, -(amount.toInt128()), msg.sender);
        _mint(to, currency.toId(), amount);
    }
}
```

实际上，用户铸造代币时必不需要立即将代币注入到 Uniswap v4 的池子内部，只需要最终平衡账户即可。

与 `mint` 函数相反，Uniswap v4 主合约内部还包含 `burn` 函数，该函数主要用于销毁 ERC6909 代币，并且增加用户的可用余额。该函数实现如下：

```solidity
function burn(address from, uint256 id, uint256 amount) external onlyWhenUnlocked {
    Currency currency = CurrencyLibrary.fromId(id);
    _accountDelta(currency, amount.toInt128(), msg.sender);
    _burnFrom(from, currency.toId(), amount);
}
```

简单来说，`mint` 和 `burn` 函数的功能都是将用户在 Flash Accounting 中的余额转化为 ERC6909 代币。但假如用户在 `Flash Accounting` 内存在余额，而且不希望使用 `mint` 函数将余额铸造为 ERC6909 代币，那么可以直接使用 `take` 函数将余额对应的底层资产直接提取:

```solidity
function take(Currency currency, address to, uint256 amount) external onlyWhenUnlocked {
    unchecked {
        // negation must be safe as amount is not negative
        _accountDelta(currency, -(amount.toInt128()), msg.sender);
        currency.transfer(to, amount);
    }
}
```

相反，假如用户持有某种 ERC20 代币，希望将此代币转化为交易过程中的 Flash Accounting 中的可用余额，那么用户可以调用 `settle()` 或者 `settleFor(address recipient)` 函数，两者的区别在于 `settle()` 函数默认使用 `msg.sender` 作为 `recipient`。该函数定义如下:

```solidity
/// @inheritdoc IPoolManager
function settleFor(address recipient) external payable onlyWhenUnlocked returns (uint256) {
    return _settle(recipient);
}
```

此处使用了 `_settle` 内部函数，该函数的定义如下:

```solidity
function _settle(address recipient) internal returns (uint256 paid) {
    Currency currency = CurrencyReserves.getSyncedCurrency();

    // if not previously synced, or the syncedCurrency slot has been reset, expects native currency to be settled
    if (currency.isAddressZero()) {
        paid = msg.value;
    } else {
        if (msg.value > 0) NonzeroNativeValue.selector.revertWith();
        // Reserves are guaranteed to be set because currency and reserves are always set together
        uint256 reservesBefore = CurrencyReserves.getSyncedReserves();
        uint256 reservesNow = currency.balanceOfSelf();
        paid = reservesNow - reservesBefore;
        CurrencyReserves.resetCurrency();
    }

    _accountDelta(currency, paid.toInt128(), recipient);
}
```

简单来说，该函数根据当前余额与之前代币余额的差值为用户增加账户余额。但可能有读者好奇，Uniswap v4 主合约如何知道为用户的哪种代币增加余额以及如何定义之前的代币余额。这实际上涉及另一个函数 `sync`，该函数定义如下:

```solidity
function sync(Currency currency) external {
    // address(0) is used for the native currency
    if (currency.isAddressZero()) {
        // The reserves balance is not used for native settling, so we only need to reset the currency.
        CurrencyReserves.resetCurrency();
    } else {
        uint256 balance = currency.balanceOfSelf();
        CurrencyReserves.syncCurrencyAndReserves(currency, balance);
    }
}
```

实际上当用户调用 `settle` 之前，用户需要首先调用 `sync` 函数锁定一些变量，然后向 Uniswap v4 主合约注入资产，最后调用 `settle` 函数将资产增加到 Flash Accounting 可用余额内部。此处所有的记录都位于闪存内部，如果读者对 `resetCurrency` 等函数感兴趣，可以自行阅读 `src/libraries/CurrencyReserves.sol` 内部的代码，注意该部分代码依旧位于 BUSL 授权下。

## Hooks

由于 Uniswap v4 的 hooks 具有复杂的权限系统以及相关的校验代码，所以在介绍其他业务逻辑代码前，我们首先介绍一下 Uniswap V4 的 hooks 的权限问题。这部分代码都位于 `src/libraries/Hooks.sol` 内部。Uniswap V4 的 hooks 的权限可以使用以下结构体汇总:

```solidity
struct Permissions {
    bool beforeInitialize;
    bool afterInitialize;
    bool beforeAddLiquidity;
    bool afterAddLiquidity;
    bool beforeRemoveLiquidity;
    bool afterRemoveLiquidity;
    bool beforeSwap;
    bool afterSwap;
    bool beforeDonate;
    bool afterDonate;
    bool beforeSwapReturnDelta;
    bool afterSwapReturnDelta;
    bool afterAddLiquidityReturnDelta;
    bool afterRemoveLiquidityReturnDelta;
}
```

当上述权限被设置后，当 Uniswap V4 执行到相关部分后就会调用指定的 hooks。如果我们将 `beforeInitialize` 和 `beforeAddLiquidity` 设置权限后，那么当资金池初始化前和流动性添加前都会调用我们指定的 hooks。

上述权限分别代表:

1. `beforeInitialize` 和 `afterInitialize` 资金池初始化前和初始化后调用
2. `beforeAddLiquidity` 和 `afterAddLiquidity` 流动性添加前和添加后调用
3. `beforeRemoveLiquidity` 和 `afterRemoveLiquidity` 流动性移除前和移除后调用
4. `beforeSwap` 和 `afterSwap` 代币兑换前和兑换后调用
5. `beforeDonate` 和 `afterDonate` 代币捐赠前和捐赠后调用
6. `beforeSwapReturnDelta` 和 `afterSwapReturnDelta` 这两个是一个特殊的权限，代表 hook 可以在兑换过程中修改 hook 地址对应的代币 `Delta`。或者说如果 hook 被赋予这个权限，那么 hook 可以在兑换过程中掌握一部分代币。
7. `afterAddLiquidityReturnDelta` 和 `afterRemoveLiquidityReturnDelta` 也是特殊权限，代表 hook 可以在流动性添加前和流动性移除后修改 hook 合约的代币 `Delta`

上述 hook 权限内最难以理解的可能就是 `beforeSwapReturnDelta` 等带有 `ReturnDelta` 标识的权限。这些权限都是用于修改 hook 在用户调用过程中的 Flash Accounting 系统内 hook 地址所对应的代币 Delta 数量。以下代码给出了在 Uniswap V4 内的 `swap` 函数内的 hook 调用:

```solidity
(swapDelta, hookDelta) = key.hooks.afterSwap(key, params, swapDelta, hookData, beforeSwapDelta);

// if the hook doesn't have the flag to be able to return deltas, hookDelta will always be 0
if (hookDelta != BalanceDeltaLibrary.ZERO_DELTA) _accountPoolBalanceDelta(key, hookDelta, address(key.hooks));
```

那么 Uniswap V4 系统如何识别 hook 是否启用了某一个权限？简单来说，Uniswap V4 使用 hook 地址的最后 14 位进行识别。假如我们的 hook 地址的最后 14 bit 的数值为 `10 0100 0000 0000`。那么就意味着该 hook 启用了 `beforeInitialize` 和 `afterAddLiquidity` 权限。

我们使用 `hasPermission` 函数检查 hook 是否具有某一个权限。

```solidity
function hasPermission(IHooks self, uint160 flag) internal pure returns (bool) {
    return uint160(address(self)) & flag != 0;
}
```

此处提供 `flag` 与地址取并计算地址是否启用了某一个权限。此处的 `flag` 是一个常量，定义如下:

```solidity
uint160 internal constant ALL_HOOK_MASK = uint160((1 << 14) - 1);

uint160 internal constant BEFORE_INITIALIZE_FLAG = 1 << 13;
uint160 internal constant AFTER_INITIALIZE_FLAG = 1 << 12;

uint160 internal constant BEFORE_ADD_LIQUIDITY_FLAG = 1 << 11;
uint160 internal constant AFTER_ADD_LIQUIDITY_FLAG = 1 << 10;

uint160 internal constant BEFORE_REMOVE_LIQUIDITY_FLAG = 1 << 9;
uint160 internal constant AFTER_REMOVE_LIQUIDITY_FLAG = 1 << 8;

uint160 internal constant BEFORE_SWAP_FLAG = 1 << 7;
uint160 internal constant AFTER_SWAP_FLAG = 1 << 6;

uint160 internal constant BEFORE_DONATE_FLAG = 1 << 5;
uint160 internal constant AFTER_DONATE_FLAG = 1 << 4;

uint160 internal constant BEFORE_SWAP_RETURNS_DELTA_FLAG = 1 << 3;
uint160 internal constant AFTER_SWAP_RETURNS_DELTA_FLAG = 1 << 2;
uint160 internal constant AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG = 1 << 1;
uint160 internal constant AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG = 1 << 0;
```

通过以上常量，我们就可以根据 hook 地址推导出该 hook 地址所支持的权限类型。由于目前使用了 14 bit 作为权限位，所以目前想获得一个合适的 hook 地址也需要一定的算力。假如读者真的需要部署 hook 地址，那么可以尝试使用 [createXcrunch](https://github.com/HrikB/createXcrunch) 工具挖掘地址。该工具可以使用 GPU 算力加速计算。

当用户使用 hook 创建池子时，初始化函数会调用以下函数来校验用户给定的 hook 地址是否正确:

```solidity
function isValidHookAddress(IHooks self, uint24 fee) internal pure returns (bool) {
    // The hook can only have a flag to return a hook delta on an action if it also has the corresponding action flag
    if (!self.hasPermission(BEFORE_SWAP_FLAG) && self.hasPermission(BEFORE_SWAP_RETURNS_DELTA_FLAG)) return false;
    if (!self.hasPermission(AFTER_SWAP_FLAG) && self.hasPermission(AFTER_SWAP_RETURNS_DELTA_FLAG)) return false;
    if (!self.hasPermission(AFTER_ADD_LIQUIDITY_FLAG) && self.hasPermission(AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG))
    {
        return false;
    }
    if (
        !self.hasPermission(AFTER_REMOVE_LIQUIDITY_FLAG)
            && self.hasPermission(AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG)
    ) return false;

    // If there is no hook contract set, then fee cannot be dynamic
    // If a hook contract is set, it must have at least 1 flag set, or have a dynamic fee
    return address(self) == address(0)
        ? !fee.isDynamicFee()
        : (uint160(address(self)) & ALL_HOOK_MASK > 0 || fee.isDynamicFee());
}
```

我们可以通过上述代码看到部分权限间存在依赖关系。依赖关系如下:

1. 只有设置 `beforeSwap` 权限后，才可以设置 `beforeSwapReturnDelta` 权限
2. 只有设置 `afterSwap` 权限后，才可以设置 `afterSwapReturnDelta` 权限
3. 只有设置 `afterAddLiquidity` 权限后，才可以设置 `afterAddLiquidityReturnDelta` 权限
4. 只有设置 `afterRemoveLiquidity` 权限后，才可以设置 `afterRemoveLiquidityReturnDelta` 权限

除此外，假如用户没有设置 hook 地址，那么池子就不可以选择动态费率。或者用户可以选择设置 hook 地址，但设置 hook 地址必须满足以下两个条件之一：

1. hook 地址具有至少一个权限
2. 启用动态费率

> 此处可以简单介绍一下动态费率的工作原理：在 Uniswap V4 的核心合约内存在 `updateDynamicLPFee` 函数，而池子对应的 hook 合约可以调用此函数来更新池子的手续费费率。所以只要池子启用动态费率，那么 hook 地址就被赋予了调整费率的功能

hook 合约需要在指定位置被调用。调用 hook 合约的函数底层使用了 `callHook` 和 `callHookWithReturnDelta` 实现。这两个底层函数都使用了 yul 汇编实现。

我们首先介绍 `callHook` 函数，该函数用于返回一般的 hook 调用的结果:

```solidity
function callHook(IHooks self, bytes memory data) internal returns (bytes memory result) {
    bool success;
    assembly ("memory-safe") {
        success := call(gas(), self, 0, add(data, 0x20), mload(data), 0, 0)
    }
    // Revert with FailedHookCall, containing any error message to bubble up
    if (!success) CustomRevert.bubbleUpAndRevertWith(address(self), bytes4(data), HookCallFailed.selector);

    // The call was successful, fetch the returned data
    assembly ("memory-safe") {
        // allocate result byte array from the free memory pointer
        result := mload(0x40)
        // store new free memory pointer at the end of the array padded to 32 bytes
        mstore(0x40, add(result, and(add(returndatasize(), 0x3f), not(0x1f))))
        // store length in memory
        mstore(result, returndatasize())
        // copy return data to result
        returndatacopy(add(result, 0x20), 0, returndatasize())
    }

    // Length must be at least 32 to contain the selector. Check expected selector and returned selector match.
    if (result.length < 32 || result.parseSelector() != data.parseSelector()) {
        InvalidHookResponse.selector.revertWith();
    }
}
```

上述代码使用 `call` 调用了 hook 合约。这个 `call` 语句是十分经典的，如果读者阅读过一些其他项目的源代码，就会发现这种 call 语句是十分常见的。`call` 函数需要以下参数 `call(gas, address, value, argsOffset, argsSize, retOffset, retSize`。一般来说，读者可以直接使用 `gas()` 函数作为 `gas` 参数传入。`gas()` 函数会返回当前交易所剩余的 gas 总量。需要注意的，由于 64 规则的存在，EVM 限制了在使用 `call` 调用外部合约时，最多只会传入 `63 / 64` 的 gas 数量，所以即使我们手动指定所有 gas 都拿来调用外部合约，实际上还会剩余 `1 / 64` 的 gas 用以处理报错。

而 `argsOffset` 和 `argsSize` 参数用来传入 calldata，前者指定 `calldata` 的在内存中的开始位置，而后者用于指定 calldata 的长度。此处我们希望使用 `bytes memory data` 作为 calldata。那么作为 `bytes memory` 类型，`data` 在 yul 汇编内会被作为指针处理，指向 `data` 在内存中的起始位置。另外，作为 `bytes memory` 类型，`data` 的最初 32 bytes 的内容实际上是 `data` 的长度。所以，实际上 `data` 的起始位置是 `add(data, 0x20)` ，该位置跳过了最初 32 bytes 的长度信息，而 `argSize` 则可以直接使用 `mload(data)`。`mload(data)` 会读取 `data` 的最初 32 bytes 的内容，而此内容刚好对应 `data` 的长度信息。

在 `callHook` 函数内，我们将 `retOffset` 和 `retSize` 都设置为 `0` ，这是因为我们不需要 `call` 帮我们直接将返回值额写入内存，我们会在后文自行处理。

假如 `callHook` 内部的 `call` 失败，那么我们会使用 `bubbleUpAndRevertWith` 函数抛出带有上下文信息的错误。

`callHook` 内部的后续工作就是将 `call` 的返回值从 `returndata` 区域写回内存区域。在这部分中，我们需要注意 uniswap 使用了 `memory-safe` 编程，所以第一步就是获取空闲的内存指针并且修复该内存指针。

```solidity
// allocate result byte array from the free memory pointer
result := mload(0x40)
// store new free memory pointer at the end of the array padded to 32 bytes
mstore(0x40, add(result, and(add(returndatasize(), 0x3f), not(0x1f))))
```

我们首先使用 `mload(0x40)` 获取当前的空闲内存指针，然后使用 `mstore(0x40, add(result, and(add(returndatasize(), 0x3f), not(0x1f))))` 指令更新 `0x40` 部分的空闲内存指针。此处需要特别注意，我们将返回的数据修正为 `bytes memory` 类型，所以我们需要 `returndatasize + 32`，因为我们需要额外的空间存储返回值的长度。同时，我们需要将返回值重整为 32 bytes 倍数的长度。此处使用了 `and(x, not(0x1f))` 的技巧，使用上述指令可以将 `x` 重整为 32 的倍数，但需要注意 `and(x, not(0x1f))` 是一个向下重整的操作，比如 `x = 65`，那么 `and(x, not(0x1f))` 的结果会是 `64`，所以为了确保最后获得充分的空间，我们一般使用 `and(x + 31, not(0x1f))` 来获得最接近 `x` 的并且比 `x` 大的数值。需要注意，刚刚我们需要 `returndatasize + 32` 保证长度信息的存储，此处我们还需要 `x + 31` 保证重整为 64 的倍数过程中不会出现问题，所以此时就使用了 `add(returndatasize(), 0x3f)`。

完成最为复杂的 `0x40` 修复后，我们使用以下代码将 `returndata` 写入内存:

```solidity
// store length in memory
mstore(result, returndatasize())
// copy return data to result
returndatacopy(add(result, 0x20), 0, returndatasize())
```

最后，我们还要求 hook 的返回值必须存在内容并且必须返回和 `data` 内一致的选择器。

```solidity
if (result.length < 32 || result.parseSelector() != data.parseSelector()) {
    InvalidHookResponse.selector.revertWith();
}
```

> 因为 `result` 本身就包含  32 bytes 的长度信息，所以此处 `result.length < 32` 成立，即意味着 hook 被调用后没有返回任何数据

此处使用了 `parseSelector` 函数，该函数较为简单：

```solidity
function parseSelector(bytes memory result) internal pure returns (bytes4 selector) {
    // equivalent: (selector,) = abi.decode(result, (bytes4, int256));
    assembly ("memory-safe") {
        selector := mload(add(result, 0x20))
    }
}
```

而 `callHookWithReturnDelta` 函数实际上就是在 `callHook` 基础上增加了使用 `parseReturnDelta` 进行返回值的处理的过程。

```solidity
function callHookWithReturnDelta(IHooks self, bytes memory data, bool parseReturn) internal returns (int256) {
    bytes memory result = callHook(self, data);

    // If this hook wasn't meant to return something, default to 0 delta
    if (!parseReturn) return 0;

    // A length of 64 bytes is required to return a bytes4, and a 32 byte delta
    if (result.length != 64) InvalidHookResponse.selector.revertWith();
    return result.parseReturnDelta();
}
```

此处使用的 `parseReturnDelta` 函数定义如下:

```solidity
function parseReturnDelta(bytes memory result) internal pure returns (int256 hookReturn) {
    // equivalent: (, hookReturnDelta) = abi.decode(result, (bytes4, int256));
    assembly ("memory-safe") {
        hookReturn := mload(add(result, 0x40))
    }
}
```

读者可能还记得 `result` 在内存内前 32 bytes 代表 result 的长度，接下来 32 bytes 代表 `bytes4 selector` ，而剩下的内容就是 `hookReturn`，所以我们直接 `mload(add(result, 0x40))` 跳过前 64 bytes 的返回值即可。

最后，我们可以简单看一个高级函数 `beforeInitialize`:

```solidity
function beforeInitialize(IHooks self, PoolKey memory key, uint160 sqrtPriceX96) internal noSelfCall(self) {
    if (self.hasPermission(BEFORE_INITIALIZE_FLAG)) {
        self.callHook(abi.encodeCall(IHooks.beforeInitialize, (msg.sender, key, sqrtPriceX96)));
    }
}
```

我们可以看到此函数只是对 `callHook` 的简单封装。此处比较有意思的是 `noSelfCall(self)` 修饰器，该修饰器的定义如下:

```solidity
modifier noSelfCall(IHooks self) {
    if (msg.sender != address(self)) {
        _;
    }
}
```

在该修饰器的作用下，我们会发现 hook 合约是无法再次调用 Uniswap V4 内部可能造成 hook 活动的函数的，以此避免无限的循环调用的出现。

如果读者需要编写 hook 合约，遵循 `src/interfaces/IHooks.sol` 内的接口编写合约即可，比如 `IHook` 内部对于 `beforeInitialize` 的定义如下:

```solidity
function beforeInitialize(address sender, PoolKey calldata key, uint160 sqrtPriceX96) external returns (bytes4);
```

注意，此处的 `bytes4` 应该返回 `beforeInitialize` 的选择器。Uniswap V4 内的 hook 有一些实现较为简单，而另一些实现由于涉及到变量更新会较为复杂。在后文内，我们将逐一分析每一个 hook 的实现。

`afterInitialize` 是一个简单的函数，该函数也只是简单的检查权限并使用 `callHook` 调用 hook 函数。

```solidity
function afterInitialize(IHooks self, PoolKey memory key, uint160 sqrtPriceX96, int24 tick)
    internal
    noSelfCall(self)
{
    if (self.hasPermission(AFTER_INITIALIZE_FLAG)) {
        self.callHook(abi.encodeCall(IHooks.afterInitialize, (msg.sender, key, sqrtPriceX96, tick)));
    }
}
```

`beforeModifyLiquidity` 是一个较为复杂的函数，但复杂性也较低，该函数主要是根据 `params.liquidityDelta` 的正负判断当前调用 `beforeAddLiquidity` 还是 `beforeRemoveLiquidity` 接口。

```solidity
function beforeModifyLiquidity(
    IHooks self,
    PoolKey memory key,
    IPoolManager.ModifyLiquidityParams memory params,
    bytes calldata hookData
) internal noSelfCall(self) {
    if (params.liquidityDelta > 0 && self.hasPermission(BEFORE_ADD_LIQUIDITY_FLAG)) {
        self.callHook(abi.encodeCall(IHooks.beforeAddLiquidity, (msg.sender, key, params, hookData)));
    } else if (params.liquidityDelta <= 0 && self.hasPermission(BEFORE_REMOVE_LIQUIDITY_FLAG)) {
        self.callHook(abi.encodeCall(IHooks.beforeRemoveLiquidity, (msg.sender, key, params, hookData)));
    }
}
```

`afterModifyLiquidity` 函数是一个较为复杂的函数，注意该函数没有 `noSelfCall(self)` 的修饰符。但是在函数头部使用 `if (msg.sender == address(self)) return (delta, BalanceDeltaLibrary.ZERO_DELTA);` 规范了当 hook 递归调用 Uniswap V4 时的操作规范。该函数会返回 `callerDelta` 和 `hookDelta`。`afterModifyLiquidity` 函数也会根据 `params.liquidityDelta` 调用不同的 hook。同时会根据 `afterAddLiquidityReturnDelta` 或者 `afterRemoveLiquidityReturnDelta` 来决定是否将返回 `hookDelta` 数值。当 `hookDelta` 数值返回时，Uniswap V4 主合约就会将部分用户的 Delta 转移给 Hook。

```solidity
function afterModifyLiquidity(
    IHooks self,
    PoolKey memory key,
    IPoolManager.ModifyLiquidityParams memory params,
    BalanceDelta delta,
    BalanceDelta feesAccrued,
    bytes calldata hookData
) internal returns (BalanceDelta callerDelta, BalanceDelta hookDelta) {
    if (msg.sender == address(self)) return (delta, BalanceDeltaLibrary.ZERO_DELTA);

    callerDelta = delta;
    if (params.liquidityDelta > 0) {
        if (self.hasPermission(AFTER_ADD_LIQUIDITY_FLAG)) {
            hookDelta = BalanceDelta.wrap(
                self.callHookWithReturnDelta(
                    abi.encodeCall(
                        IHooks.afterAddLiquidity, (msg.sender, key, params, delta, feesAccrued, hookData)
                    ),
                    self.hasPermission(AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG)
                )
            );
            callerDelta = callerDelta - hookDelta;
        }
    } else {
        if (self.hasPermission(AFTER_REMOVE_LIQUIDITY_FLAG)) {
            hookDelta = BalanceDelta.wrap(
                self.callHookWithReturnDelta(
                    abi.encodeCall(
                        IHooks.afterRemoveLiquidity, (msg.sender, key, params, delta, feesAccrued, hookData)
                    ),
                    self.hasPermission(AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG)
                )
            );
            callerDelta = callerDelta - hookDelta;
        }
    }
}
```

`beforeSwap` 也是一个较为复杂的函数，因为在该函数内部设计到了手续费费率更新和用户 swap 数量的修改逻辑。此处需要注意的是，`beforeSwap` 启用 `beforeSwapReturnDelta` 权限后可以在用户兑换前修改用户的兑换数量。但兑换数量的正负数值实际上与 `exactIn` 或 `exactOut` 模式有关，我们需要保证叠加 hook 的返回结果后，用户的兑换模式不会发生改变。另一个值得注意的地方在于 `beforeSwap` 会返回 `lpFeeOverride` 来重载当前的 LP 手续费费率，该参数在用户设置静态费率下会生效。

```solidity
function beforeSwap(IHooks self, PoolKey memory key, IPoolManager.SwapParams memory params, bytes calldata hookData)
    internal
    returns (int256 amountToSwap, BeforeSwapDelta hookReturn, uint24 lpFeeOverride)
{
    amountToSwap = params.amountSpecified;
    if (msg.sender == address(self)) return (amountToSwap, BeforeSwapDeltaLibrary.ZERO_DELTA, lpFeeOverride);

    if (self.hasPermission(BEFORE_SWAP_FLAG)) {
        bytes memory result = callHook(self, abi.encodeCall(IHooks.beforeSwap, (msg.sender, key, params, hookData)));

        // A length of 96 bytes is required to return a bytes4, a 32 byte delta, and an LP fee
        if (result.length != 96) InvalidHookResponse.selector.revertWith();

        // dynamic fee pools that want to override the cache fee, return a valid fee with the override flag. If override flag
        // is set but an invalid fee is returned, the transaction will revert. Otherwise the current LP fee will be used
        if (key.fee.isDynamicFee()) lpFeeOverride = result.parseFee();

        // skip this logic for the case where the hook return is 0
        if (self.hasPermission(BEFORE_SWAP_RETURNS_DELTA_FLAG)) {
            hookReturn = BeforeSwapDelta.wrap(result.parseReturnDelta());

            // any return in unspecified is passed to the afterSwap hook for handling
            int128 hookDeltaSpecified = hookReturn.getSpecifiedDelta();

            // Update the swap amount according to the hook's return, and check that the swap type doesn't change (exact input/output)
            if (hookDeltaSpecified != 0) {
                bool exactInput = amountToSwap < 0;
                amountToSwap += hookDeltaSpecified;
                if (exactInput ? amountToSwap > 0 : amountToSwap < 0) {
                    HookDeltaExceedsSwapAmount.selector.revertWith();
                }
            }
        }
    }
}
```

上述函数中有一个较为奇特的函数 `getSpecifiedDelta` ，该函数的作用是将 `int256` 转化为 `int128`，该函数对应的源代码如下:

```solidity
function getSpecifiedDelta(BeforeSwapDelta delta) internal pure returns (int128 deltaSpecified) {
    assembly ("memory-safe") {
        deltaSpecified := sar(128, delta)
    }
}
```

上述函数使用了 `sar` 操作码，该操作码的作用是将 `delta` 右移 128 位。注意，`getSpecifiedDelta` 的目标是在 `int256` 的高 128 位内获取一个 `int128` 类型。此处需要补充在 EVM 内部，`int` 系列类型实际上都使用了补码的形式进行表示。

上述 `getSpecifiedDelta` 函数被定义在 `src/types/BeforeSwapDelta.sol` 内部，如果读者查看此文件，会发现与 `getSpecifiedDelta` 对应另一个 `getUnspecifiedDelta` 函数，该函数的作用是在一个 `int256` 的低 128 位内获取一个 `int128`。该函数的定义如下：

```solidity
function getUnspecifiedDelta(BeforeSwapDelta delta) internal pure returns (int128 deltaUnspecified) {
    assembly ("memory-safe") {
        deltaUnspecified := signextend(15, delta)
    }
}
```

此处使用 `signextend(b, x)` 方法，这可能是 EVM 内最复杂的操作码。该操作码的执行原理是：`b` 会指定当前 `x` 的在哪一个范围内构成反码。比如 `0b1111 1110`，该数值可以在认为是一个负数 `-2`，但也可以认为是一个正数 `254`。这主要取决于我们认为多少位表示一个数字。选择不同的位元，我们可能得到对某一个带符号整数不同的正负结论。在 `signextend(b, x)` 内，`b` 就用来确定某一个数字正负情况。假如 `b = 0`，那么 `x` 会被视为 `int8` 处理；如果 `b = 1`，那么 `x` 被视为 `int16` 处理。我们认为当我们调用 `signextend(0b11111110, 0)` 时，此时 `signextend` 会将 `0b11111110` 视为负数 `-2` ，将其扩展为 256 位。而调用 `signextend(0b11111110, 1)` 时，此时 `signextend` 会将 `0b11111110` 视为正数 254。

此处存在一个有趣的情况，即假如 `b = 15`，我们认为 `x` 的类型是一个 `int128`，但如果此时 `x` 为一个大于 `int128` 的类型，如 `int256` 的话，此时 `signextend` 不会处理大于 `int128` 的部分位元，而是直接将 `x` 截断为 `int128` 然后扩展为 `int256`。此时我们可以认为 `signextend` 就是将 `int256` 的低 128 位在保留原有正负符号的情况下提取为 `int128` 类型。本质上 ``signextend(b, x)`` 就是根据 `b` 判断 `x` 所需要扩展的范围，比如 `b = 15`，那么 `x` 的前 128 位需要符号扩展，然后直接将 `x` 的前 $256 − 8(a + 1)$ 位提取并扩展为带符号的 `int256` 类型。

我们可以看到 `getSpecifiedDelta` 在 `int256` 的高 128 位提取 `int128` ，而 `getUnspecifiedDelta` 在 `int256` 的低 128 位提取 `int128`。出现这两个函数的原因在于 Uniswap V4 定义了 `BalanceDelta` 类型，该类型属于 `int256`，但其高 128 位代表 `amount0`，低 128 位代表 `amount1`。我们在 `src/types/BalanceDelta.sol` 内部定义了 `BalanceDelta` 的四则运算和类型转化。

```solidity
function toBalanceDelta(int128 _amount0, int128 _amount1) pure returns (BalanceDelta balanceDelta) {
    assembly ("memory-safe") {
        balanceDelta := or(shl(128, _amount0), and(sub(shl(128, 1), 1), _amount1))
    }
}
```

我们使用 `toBalanceDelta` 将 `_amount0` 和 `_amount1` 拼接为 `balanceDelta` 类型。上述代码内较为复杂的 `and(sub(shl(128, 1), 1), _amount1)` 是为了清除 `_amount1` 可能存在的高位垃圾值。对于 `BalanceDelta` 的加法和减法都较为简单，读者可以自行阅读相关代码。

在上文内，我们可以看到 `amountToSwap += hookDeltaSpecified;` 的代码，此处的 `hookDeltaSpecified` 是使用 `int128 hookDeltaSpecified = hookReturn.getSpecifiedDelta();` 获得的。所以，`beforeSwap` 返回值的高 128 位用于修改用户 swap 的输入值，但是此处我们并不能确定修改的是哪一个代币的数值，此处可能存在以下四种情况:

```
// Set amount0 and amount1
// zero for one | exact input |
//    true      |    true     | amount 0 = amountToSwap + hookDeltaSpecified
//              |             | amount 1 = hookDeltaUnspecified
//    false     |    false    | amount 0 = amountToSwap + hookDeltaSpecified
//              |             | amount 1 = hookDeltaUnspecified
//    false     |    true     | amount 0 = hookDeltaUnspecified
//              |             | amount 1 = amountToSwap + hookDeltaSpecified
//    true      |    false    | amount 0 = hookDeltaUnspecified
//              |             | amount 1 = amountToSwap + hookDeltaSpecified
```

在 `afterSwap` 内部，我们使用了 `BalanceDelta` 类型。`afterSwap` 的源代码如下:

```solidity
function afterSwap(
    IHooks self,
    PoolKey memory key,
    IPoolManager.SwapParams memory params,
    BalanceDelta swapDelta,
    bytes calldata hookData,
    BeforeSwapDelta beforeSwapHookReturn
) internal returns (BalanceDelta, BalanceDelta) {
    if (msg.sender == address(self)) return (swapDelta, BalanceDeltaLibrary.ZERO_DELTA);

    int128 hookDeltaSpecified = beforeSwapHookReturn.getSpecifiedDelta();
    int128 hookDeltaUnspecified = beforeSwapHookReturn.getUnspecifiedDelta();

    if (self.hasPermission(AFTER_SWAP_FLAG)) {
        hookDeltaUnspecified += self.callHookWithReturnDelta(
            abi.encodeCall(IHooks.afterSwap, (msg.sender, key, params, swapDelta, hookData)),
            self.hasPermission(AFTER_SWAP_RETURNS_DELTA_FLAG)
        ).toInt128();
    }

    BalanceDelta hookDelta;
    if (hookDeltaUnspecified != 0 || hookDeltaSpecified != 0) {
        hookDelta = (params.amountSpecified < 0 == params.zeroForOne)
            ? toBalanceDelta(hookDeltaSpecified, hookDeltaUnspecified)
            : toBalanceDelta(hookDeltaUnspecified, hookDeltaSpecified);

        // the caller has to pay for (or receive) the hook's delta
        swapDelta = swapDelta - hookDelta;
    }
    return (swapDelta, hookDelta);
}
```

此处 `afterSwap` 要求传入 `beforeSwap` 的返回值 `beforeSwapHookReturn`。`afterSwap` 会在 hook 返回结果后直接修改 `beforeSwapHookReturn` 内的数值，最后返回 `hookDelta` 的数值。由于此处需要返回最终的 `hookDelta` 和 `swapDelta` 以供 Uniswap v4 合约修改 Flash Accounting 内部记录的数据，所以此处一个较为复杂的逻辑在于构建 `hookDelta`。在上文，我们提及 `BalanceDelta` 是一个特殊的 `int256`，其前 128 位代表 `amount0` 的数值而后 128 位代表 `amount1` 的数值。所以此处我们需要根据 `beforeSwapHookReturn` 和 `afterSwap` 的返回值构造 `hookDelta`。根据我们上文给出的表格:

```
// Set amount0 and amount1
// zero for one | exact input |
//    true      |    true     | amount 0 = hookDeltaSpecified
//              |             | amount 1 = hookDeltaUnspecified
//    false     |    false    | amount 0 = hookDeltaSpecified
//              |             | amount 1 = hookDeltaUnspecified
//    false     |    true     | amount 0 = hookDeltaUnspecified
//              |             | amount 1 = hookDeltaSpecified
//    true      |    false    | amount 0 = hookDeltaUnspecified
//              |             | amount 1 = hookDeltaSpecified
```

我们可以看到 `params.amountSpecified < 0(exactIn) == params.zeroForOne` 成立时，我们可以使用 `toBalanceDelta(hookDeltaSpecified, hookDeltaUnspecified)` 构造 `hookDelta`，否则可以使用 `toBalanceDelta(hookDeltaUnspecified, hookDeltaSpecified)` 构建 `hookDelta`。此处的 `swapDelta` 是 Uniswap V4 主合约构建好的，此处我们不需要关心。

最后，我们总结一下 `swap` 相关两个函数的作用:

1. `beforeSwap` 更新当前的 LP 费率、修改用户的 `amountToSwap` 、修改 `hookDelta`
2. `afterSwap` 修改 `hookDelta`

其他的 hook 函数调用在 Uniswap V4 内的代码都较为简单。如下:

```solidity
function beforeDonate(IHooks self, PoolKey memory key, uint256 amount0, uint256 amount1, bytes calldata hookData)
    internal
    noSelfCall(self)
{
    if (self.hasPermission(BEFORE_DONATE_FLAG)) {
        self.callHook(abi.encodeCall(IHooks.beforeDonate, (msg.sender, key, amount0, amount1, hookData)));
    }
}

/// @notice calls afterDonate hook if permissioned and validates return value
function afterDonate(IHooks self, PoolKey memory key, uint256 amount0, uint256 amount1, bytes calldata hookData)
    internal
    noSelfCall(self)
{
    if (self.hasPermission(AFTER_DONATE_FLAG)) {
        self.callHook(abi.encodeCall(IHooks.afterDonate, (msg.sender, key, amount0, amount1, hookData)));
    }
}
```

总结来看，Uniswap V4 内部的 `beforeSwap` 和 `afterSwap` 是最为复杂的 hook 调用，主要原因在于 `beforeSwap` 和 `afterSwap` 都是可以直接修改部分内部变量的。对于更加详细的关于 hook 的介绍，我们可能会在后期介绍 hook 合约编程时具体介绍。

## 初始化

初始化函数是我们阅读 Uniswap v4 的业务逻辑过程中所接触的第一个函数，该函数用于流动性池的初始化。该函数定义如下:

```solidity
function initialize(PoolKey memory key, uint160 sqrtPriceX96) external noDelegateCall returns (int24 tick) {}
```

初始化函数需要用户输入 `PoolKey` 类型的 `key` 变量，此处的 `PoolKey` 类型定义如下：

```solidity
struct PoolKey {
    /// @notice The lower currency of the pool, sorted numerically
    Currency currency0;
    /// @notice The higher currency of the pool, sorted numerically
    Currency currency1;
    /// @notice The pool LP fee, capped at 1_000_000. If the highest bit is 1, the pool has a dynamic fee and must be exactly equal to 0x800000
    uint24 fee;
    /// @notice Ticks that involve positions must be a multiple of tick spacing
    int24 tickSpacing;
    /// @notice The hooks of the pool
    IHooks hooks;
}
```

用户需要输入 `currency0` 和 `currency1`。但是不同于之前的 Uniswap V3 版本，目前允许用户使用动态费率，只需要将 `fee` 置为 `0x800000` 即可，其他情况下仍使用 `1000000 = 100%` 的精度进行计算。在动态费率下，我们有两种方案可以更新当前的费率。第一种方案是使用 hook 合约，正如上文所述，当 Uniswap V4 主合约调用 `beforeSwap` 的函数时，会根据 `hook` 的返回值更新当前的费率。还存在一种情况就是 hook 合约调用 Uniswap V4 主合约中的 `updateDynamicLPFee` 函数实现费率更新。

与 Uniswap v2 不同，Uniswap V4 内的 `tickSpacing` 目前则完全放开，用户可以任意选择 1 - 32767 的数值，数值越小，那么价格跳跃越小，但进行 swap 时可能需要更多 gas。而 `hooks` 参数是指当前流动性池所对应的 hook 合约地址。注意，一个池子只可以指向一个 hook 合约地址。

> 值得注意的，我们使用上述 `PoolKey` 可以计算获得一个 `PoolId`，所以当池子使用不同的 hook 时就会产生不同的 `PoolKey`

在 `initialize` 函数的入口部分，我们会进行参数的校验:

```solidity
if (key.tickSpacing > MAX_TICK_SPACING) TickSpacingTooLarge.selector.revertWith(key.tickSpacing);
if (key.tickSpacing < MIN_TICK_SPACING) TickSpacingTooSmall.selector.revertWith(key.tickSpacing);
if (key.currency0 >= key.currency1) {
    CurrenciesOutOfOrderOrEqual.selector.revertWith(
        Currency.unwrap(key.currency0), Currency.unwrap(key.currency1)
    );
}
```

首先检查 `tickSpacing` 参数是否位于 `MAX_TICK_SPACING` 与 `MIN_TICK_SPACING` 之间。然后校验 `currency0` 和 `currency1` 是否满足排序条件。地址较小的代币作为 `currency0` ，反之作为 `currency1`。我们可以看到 Uniswap v4 的特点：Uniswap V4 不在合约内进行各类计算，而是要求用户在本地完成计算，然后在链上只进行计算结果的校验。

我们进行使用以下代码进行 `hooks` 的校验:

```solidity
if (!key.hooks.isValidHookAddress(key.fee)) Hooks.HookAddressNotValid.selector.revertWith(address(key.hooks));
```

最后，我们需要校验 传入参数 `key` 内的 `fee` 参数是否正确。我们需要调用如下函数:

```solidity
uint24 lpFee = key.fee.getInitialLPFee();
```

此处调用了位于 `src/libraries/LPFeeLibrary.sol` 内部的 `getInitialLPFee` 函数，该函数定义如下:

```solidity
function isValid(uint24 self) internal pure returns (bool) {
    return self <= MAX_LP_FEE;
}

function validate(uint24 self) internal pure {
    if (!self.isValid()) LPFeeTooLarge.selector.revertWith(self);
}

function getInitialLPFee(uint24 self) internal pure returns (uint24) {
    // the initial fee for a dynamic fee pool is 0
    if (self.isDynamicFee()) return 0;
    self.validate();
    return self;
}
```

上述代码的核心目标是为了避免用户使用大于 100% 的费率，此处的存在常量 `uint24 public constant MAX_LP_FEE = 1000000;`

完成上述最初始的校验工作后，我们首先调用 `key.hooks.beforeInitialize(key, sqrtPriceX96);` 代码通知 hook 合约即将初始化流动性池，hook 可以根据 Uni swap v4 传入的参数进行相关响应。接下来，Uniswap v4 主合约会进行一系列初始化操作。第一步就是根据用户输入的 `key` 计算流动性池的唯一标识 `PoolId`。此处会使用 `PoolId id = key.toId();` 代码，该代码的核心部分就是 `toId` 函数，该函数定义如下:

```solidity
library PoolIdLibrary {
    /// @notice Returns value equal to keccak256(abi.encode(poolKey))
    function toId(PoolKey memory poolKey) internal pure returns (PoolId poolId) {
        assembly ("memory-safe") {
            // 0xa0 represents the total size of the poolKey struct (5 slots of 32 bytes)
            poolId := keccak256(poolKey, 0xa0)
        }
    }
}
```

因为 `PoolKey` 在内存内的布局占用 5 个 `uint256` ，所以此处进行 `keccak256` 计算时的长度为 `0xa0`。此处可以再次重申，在 yul 汇编内，`memory` 类型都被视为该数据结构在内存内的起始位置。

最后，我们会进行真正的流动性池的初始化，调用以下代码:

```solidity
tick = _pools[id].initialize(sqrtPriceX96, lpFee);
```

`_pools` 实际上是 `mapping(PoolId id => Pool.State) internal _pools;` 中定义的变量。我们可以看到在此变量内，我们将流动性池的 `PoolId` 与流动性池的状态进行了映射。而 `Pool.State` 的具体定义在 `src/libraries/Pool.sol` 内部，我们在此处摘录一下 `State` 的定义代码:

```solidity
struct State {
    Slot0 slot0;
    uint256 feeGrowthGlobal0X128;
    uint256 feeGrowthGlobal1X128;
    uint128 liquidity;
    mapping(int24 tick => TickInfo) ticks;
    mapping(int16 wordPos => uint256) tickBitmap;
    mapping(bytes32 positionKey => Position.State) positions;
}
```

此处的大部分变量都在 Uniswap v3 的主合约内出现过，我们不再详细介绍其功能。值得一提的是，此处的 `Slot0` 使用了手动打包的方法是因为使用这种方案可以降低 `Slot0` 在内存中占用的体积，降低内存占用成本。`Slot0` 的提取和写入的代码被定义在 `src/types/Slot0.sol` 内，我们可以简单分析几段代码:

```solidity
function tick(Slot0 _packed) internal pure returns (int24 _tick) {
    assembly ("memory-safe") {
        _tick := signextend(2, shr(TICK_OFFSET, _packed))
    }
}

function protocolFee(Slot0 _packed) internal pure returns (uint24 _protocolFee) {
    assembly ("memory-safe") {
        _protocolFee := and(MASK_24_BITS, shr(PROTOCOL_FEE_OFFSET, _packed))
    }
}
```

`tick` 函数内部的 `shr` 是为了向右移动 `_packed` 使得 ` tick` 变量位于最右侧，然后直接使用 `signextend` 将右移之后的结果转化为 `int24`，需要注意这种转化清除其他参数的数值，所以 `signextend` 的结果就是我们需要的最终结果。

`protocolFee` 函数则展示了一般的 `uint` 类型的数值提取，也是先使用 `shr` 方法将需要的参数移动到最右侧，然后直接使用 `MASK_24_BITS` 将其他数值使用 `and` 方法清空。

对于 `Slot0` 的数值写入，我们使用如下代码：

```solidity
function setProtocolFee(Slot0 _packed, uint24 _protocolFee) internal pure returns (Slot0 _result) {
    assembly ("memory-safe") {
        _result :=
            or(
                and(not(shl(PROTOCOL_FEE_OFFSET, MASK_24_BITS)), _packed),
                shl(PROTOCOL_FEE_OFFSET, and(MASK_24_BITS, _protocolFee))
            )
    }
}
```

简单来说，我们首先将 MASK 掩码移动到需要修改的参数位置，然后通过取 `not` 操作将掩码从原有的全 `1` 构成修改为全 `0` 构成，然后与 `_packed` 取 `and` 操作是的 `protocolFee` 所在的位置被置为 `0`。最后使用 `or` 将修正好的 `_protocolFee` 写入。

有了上述知识后，我们就可以完成 `initialize` 函数，该函数的定义如下:

```solidity
function initialize(State storage self, uint160 sqrtPriceX96, uint24 lpFee) internal returns (int24 tick) {
    if (self.slot0.sqrtPriceX96() != 0) PoolAlreadyInitialized.selector.revertWith();

    tick = TickMath.getTickAtSqrtPrice(sqrtPriceX96);

    // the initial protocolFee is 0 so doesn't need to be set
    self.slot0 = Slot0.wrap(bytes32(0)).setSqrtPriceX96(sqrtPriceX96).setTick(tick).setLpFee(lpFee);
}
```

此处的 `getTickAtSqrtPrice` 与 Uniswap v3 内是一致的，读者可以自行阅读之前的博客。

完成上述所有工作后，Uniswap v4 的主合约会使用 `key.hooks.afterInitialize(key, sqrtPriceX96, tick);` 代码通知 hook 合约已完成初始化，hook 合约可以进行其他操作。

## 流动性修改

流动性的修改需要调用 `modifyLiquidity` 函数，该函数的定义如下：

```solidity
function modifyLiquidity(
    PoolKey memory key,
    IPoolManager.ModifyLiquidityParams memory params,
    bytes calldata hookData
) external onlyWhenUnlocked noDelegateCall returns (BalanceDelta callerDelta, BalanceDelta feesAccrued) {}
```

此处的 `PoolKey` 就是需要操作的流动性池的 ID，而 `ModifyLiquidityParams` 结构体定义如下：

```solidity
struct ModifyLiquidityParams {
    // the lower and upper tick of the position
    int24 tickLower;
    int24 tickUpper;
    // how to modify the liquidity
    int256 liquidityDelta;
    // a value to set if you want unique liquidity positions at the same range
    bytes32 salt;
}
```

相比于 Uniswap v3，Uniswap v4 使用了 `salt` 替代了原有的 `owner` 参数。但是需要注意 `owner` 参数实际上还存在。我们先分析 `modifyLiquidity` 的第一部分代码:

```solidity
PoolId id = key.toId();
{
    Pool.State storage pool = _getPool(id);
    pool.checkPoolInitialized();

    key.hooks.beforeModifyLiquidity(key, params, hookData);

    BalanceDelta principalDelta;
    (principalDelta, feesAccrued) = pool.modifyLiquidity(
        Pool.ModifyLiquidityParams({
            owner: msg.sender,
            tickLower: params.tickLower,
            tickUpper: params.tickUpper,
            liquidityDelta: params.liquidityDelta.toInt128(),
            tickSpacing: key.tickSpacing,
            salt: params.salt
        })
    );

    // fee delta and principal delta are both accrued to the caller
    callerDelta = principalDelta + feesAccrued;
}
```

此处首先将用户输入 `key` 转化为 `PoolId`。然后使用 `_getPool` 函数获得 `Pool.State`，此处使用的 `_getPool` 函数定义如下:

```solidity
/// @notice Implementation of the _getPool function defined in ProtocolFees
function _getPool(PoolId id) internal view override returns (Pool.State storage) {
    return _pools[id];
}
```

然后使用 `checkPoolInitialized` 函数检查 Pool 是否初始化，该函数对应的代码如下:

```solidity
function checkPoolInitialized(State storage self) internal view {
    if (self.slot0.sqrtPriceX96() == 0) PoolNotInitialized.selector.revertWith();
}
```

然后，我们调用 hook 合约的 `beforeModifyLiquidity` 函数来通知 hook 合约。

后续的核心步骤是 `pool.modifyLiquidity` 函数，我们先跳过对该函数内部的具体分析。简单来说，`pool.modifyLiquidity` 函数会返回当前用户添加流动性所需要的两种代币数量 `principalDelta` 和可能存在的手续费收入 `feesAccrued`。最后统一计算 `callerDelta = principalDelta + feesAccrued;`。相比于 Uniswap v3 中的 `_modifyPosition` 函数，此处将用户修改的流动性和手续费结合实际上方便于用户进行 LP 的复利操作。在 Unsiwap v3 内，用户进行复利操作需要首先调用 `burn` 函数触发手续费更新，然后调用 `collect` 函数提取手续费，最后调用 `mint` 函数添加流动性。但在 Uniswap v4 内，只需要一步调用 `modifyLiquidity` 函数，只需要保证铸造的流动性数量与手续费数量相等，我们就可以使得 `callerDelta = 0` ，此时我们不需要想 Uniswap v4 的池子注入代币。

最后，我们将 `modifyLiquidity` 的最后部分代码展示一下:

```solidity
// event is emitted before the afterModifyLiquidity call to ensure events are always emitted in order
emit ModifyLiquidity(id, msg.sender, params.tickLower, params.tickUpper, params.liquidityDelta, params.salt);

BalanceDelta hookDelta;
(callerDelta, hookDelta) = key.hooks.afterModifyLiquidity(key, params, callerDelta, feesAccrued, hookData);

// if the hook doesn't have the flag to be able to return deltas, hookDelta will always be 0
if (hookDelta != BalanceDeltaLibrary.ZERO_DELTA) _accountPoolBalanceDelta(key, hookDelta, address(key.hooks));

_accountPoolBalanceDelta(key, callerDelta, msg.sender);
```

这些代码都是处理流动性修改后的收尾工作。首先抛出事件以方便 indexer 检索，然后调用 hook 合约中的 `afterModifyLiquidity` 函数，并处理可能的 `hookDelta` 返回值。此处使用了 `_accountPoolBalanceDelta` 函数进行了 `BalanceDelta` 的写入，该函数的定义如下:

```solidity
function _accountPoolBalanceDelta(PoolKey memory key, BalanceDelta delta, address target) internal {
    _accountDelta(key.currency0, delta.amount0(), target);
    _accountDelta(key.currency1, delta.amount1(), target);
}
```

完成了上述介绍后，我们具体介绍一下 `pool.modifyLiquidity` 函数，该函数具有以下几个功能：

1. 使用 `updateTick` 更新 Tick 内部的数据，具体可以参考 Uniswap v3 中的部分
2. 使用 `position.update` 函数更新计算 `Position` 内部的手续费收入
3. 计算指定流动性添加所吸引的代币数量以及修改全局流动性

在后文内部，我们不会介绍一些实现与 Uniswap v3 一致的函数，读者可以自行阅读笔者之前编写的 [现代 DeFi: Uniswap V3](https://blog.wssh.trade/posts/uniswap-v3/)。我们首先分析第一部分代码，该部分代码用于校验输入参数的正确性，代码如下:

```solidity
int128 liquidityDelta = params.liquidityDelta;
int24 tickLower = params.tickLower;
int24 tickUpper = params.tickUpper;
checkTicks(tickLower, tickUpper);
```

接下来，`modifyLiquidity` 完成了 Tick 相关数据的更新。由于 Uniswap V4 内部去掉了主合约内部的预言机逻辑，所以目前 Tick 内只记录了 `liquidityGrossAfter` / `liquidityNet` /  `feeGrowthOutside0X128` 和 `feeGrowthOutside1X128`。其中前者 `liquidityGrossAfter` 用于记录所有引用该 tick 的 Position 的流动性数量，主要功能是为了判断 tick 是否会被 flipped。而 `liquidityNet` 则记录该 tick 被跨越时流动性的改变数量。剩下的 `feeGrowthOutside0X128` 和 `feeGrowthOutside1X128` 用于手续费记录。该部分被定义在 `TickInfo` 结构体内部:

```solidity
struct TickInfo {
    // the total position liquidity that references this tick
    uint128 liquidityGross;
    // amount of net liquidity added (subtracted) when tick is crossed from left to right (right to left),
    int128 liquidityNet;
    // fee growth per unit of liquidity on the _other_ side of this tick (relative to the current tick)
    // only has relative meaning, not absolute — the value depends on when the tick is initialized
    uint256 feeGrowthOutside0X128;
    uint256 feeGrowthOutside1X128;
}
```

比较有趣的是在 `updateTick` 内部使用了如下内联汇编实现数据更新:

```solidity
TickInfo storage info = self.ticks[tick];

assembly ("memory-safe") {
    // liquidityGrossAfter and liquidityNet are packed in the first slot of `info`
    // So we can store them with a single sstore by packing them ourselves first
    sstore(
        info.slot,
        // bitwise OR to pack liquidityGrossAfter and liquidityNet
        or(
            // Put liquidityGrossAfter in the lower bits, clearing out the upper bits
            and(liquidityGrossAfter, 0xffffffffffffffffffffffffffffffff),
            // Shift liquidityNet to put it in the upper bits (no need for signextend since we're shifting left)
            shl(128, liquidityNet)
        )
    )
}
```

这是因为 `TickInfo` 的体积为 64 bytes，在 EVM 的存储布局内占用 2 个存储槽。在过去的方案内部，Uniswap v3 选择的方案是将 `TickInfo` 完全缓存到内存中，在修改完成后，再将内存中的结构体完全写入存储槽，这实际上并不有利于 Gas 优化，一种更好的选择就是上文给出的内联汇编代码。我们由于 `TickInfo` 占用了两个存储槽，所以更新 `liquidityGross` 和 `liquidityNet` 需要确定更新存储槽的位置。基于 solidity 存储打包的规则，结构体内的数据按顺序写入存储槽，所以我们可以直接使用 `info.slot` 获得 `liquidityGross` 和 `liquidityNet` 的位置，然后直接使用 `sstore` 进行修改。

> 但是对于 `feeGrowthOutside0X128` 和 `feeGrowthOutside1X128` 的修改，Uniswap v4 就使用了传统的方案，即直接使用 `info.feeGrowthOutside0X128 = self.feeGrowthGlobal0X128;` 和 `info.feeGrowthOutside1X128 = self.feeGrowthGlobal1X128;`。这种选择性的优化方案是值得思考的。

`updateTick` 函数的其他部分与 Uniswap v3 是几乎一致的，读者可以自行阅读一下源代码。

我们展示一下 `modifyLiquidity` 函数内部对于 Tick 的更新逻辑:

```solidity
if (liquidityDelta != 0) {
    (state.flippedLower, state.liquidityGrossAfterLower) =
        updateTick(self, tickLower, liquidityDelta, false);
    (state.flippedUpper, state.liquidityGrossAfterUpper) = updateTick(self, tickUpper, liquidityDelta, true);

    // `>` and `>=` are logically equivalent here but `>=` is cheaper
    if (liquidityDelta >= 0) {
        uint128 maxLiquidityPerTick = tickSpacingToMaxLiquidityPerTick(params.tickSpacing);
        if (state.liquidityGrossAfterLower > maxLiquidityPerTick) {
            TickLiquidityOverflow.selector.revertWith(tickLower);
        }
        if (state.liquidityGrossAfterUpper > maxLiquidityPerTick) {
            TickLiquidityOverflow.selector.revertWith(tickUpper);
        }
    }

    if (state.flippedLower) {
        self.tickBitmap.flipTick(tickLower, params.tickSpacing);
    }
    if (state.flippedUpper) {
        self.tickBitmap.flipTick(tickUpper, params.tickSpacing);
    }
}
```

此处使用的 `flipTick` 相比于 Uniswap v3 而言，已经变成了完全使用内联汇编构建的，但是该部分内联汇编较为简单，读者可以自行阅读。还有一个特殊部分在于 Uniswap v3 在初始化过程中就计算了 `maxLiquidityPerTick` 变量，在 Uniswap v4 中，我们改成了每次添加流动性时再计算。这可能是因为 Uniswap 开发团队评估了读取 `maxLiquidityPerTick` 存储槽的成本和每次添加流动性时计算 `maxLiquidityPerTick` 的成本，发现每次计算可能会更加便宜就选择了每次添加流动性时进行计算。

此处为了避免 solidity 编译器给出堆栈过深的警告。这是因为比如 `uint128 maxLiquidityPerTick` 这种变量都存储在栈内部，而 EVM 最多只允许 `SWAP16` 操作码，所以当在单一作用域时给出声明太多变量就会出现堆栈过深的警告。 Uniswap v4 使用了两种方案，第一种就是常规的规划作用域，当 solidity 离开作用域时，作用域内的所有变量都是丢弃。第二种方案就是声明 `ModifyLiquidityState memory state;` 变量，利用结构体内部的变量位于内存中的特性避免过多的临时变量占用栈。

在 `modifyLiquidity` 函数的第二部分，我们需要进行手续费计算，这部分代码基本完全使用了 Uniswap v3 中的代码:

```solidity
(uint256 feeGrowthInside0X128, uint256 feeGrowthInside1X128) =
    getFeeGrowthInside(self, tickLower, tickUpper);

Position.State storage position = self.positions.get(params.owner, tickLower, tickUpper, params.salt);
(uint256 feesOwed0, uint256 feesOwed1) =
    position.update(liquidityDelta, feeGrowthInside0X128, feeGrowthInside1X128);

// Fees earned from LPing are calculated, and returned
feeDelta = toBalanceDelta(feesOwed0.toInt128(), feesOwed1.toInt128());
```

在 `modifyLiquidity` 的第三部分，我们需要计算指定流动性添加所吸引的代币数量以及修改全局流动性。这部分代码也与 Uniswap v3 保持了一致性:

```solidity
if (liquidityDelta != 0) {
    Slot0 _slot0 = self.slot0;
    (int24 tick, uint160 sqrtPriceX96) = (_slot0.tick(), _slot0.sqrtPriceX96());
    if (tick < tickLower) {
        // current tick is below the passed range; liquidity can only become in range by crossing from left to
        // right, when we'll need _more_ currency0 (it's becoming more valuable) so user must provide it
        delta = toBalanceDelta(
            SqrtPriceMath.getAmount0Delta(
                TickMath.getSqrtPriceAtTick(tickLower), TickMath.getSqrtPriceAtTick(tickUpper), liquidityDelta
            ).toInt128(),
            0
        );
    } else if (tick < tickUpper) {
        delta = toBalanceDelta(
            SqrtPriceMath.getAmount0Delta(sqrtPriceX96, TickMath.getSqrtPriceAtTick(tickUpper), liquidityDelta)
                .toInt128(),
            SqrtPriceMath.getAmount1Delta(TickMath.getSqrtPriceAtTick(tickLower), sqrtPriceX96, liquidityDelta)
                .toInt128()
        );

        self.liquidity = LiquidityMath.addDelta(self.liquidity, liquidityDelta);
    } else {
        // current tick is above the passed range; liquidity can only become in range by crossing from right to
        // left, when we'll need _more_ currency1 (it's becoming more valuable) so user must provide it
        delta = toBalanceDelta(
            0,
            SqrtPriceMath.getAmount1Delta(
                TickMath.getSqrtPriceAtTick(tickLower), TickMath.getSqrtPriceAtTick(tickUpper), liquidityDelta
            ).toInt128()
        );
    }
}
```

至此，我们就完成了流动性添加部分的所有代码。

## Swap

Uniswap v4 的 `swap` 函数的相比于 Uniswap v3 在核心的 AMM 数学逻辑上并没有变化，但由于引入了 Hooks 和 Flash Accounting 机制，所以在代码上还有存在一定变化的。我们首先简单看一下 `swap` 函数的定义:

```solidity
function swap(PoolKey memory key, IPoolManager.SwapParams memory params, bytes calldata hookData)
    external
    onlyWhenUnlocked
    noDelegateCall
    returns (BalanceDelta swapDelta)
{}
```

此处出现了 `IPoolManager.SwapParams` 参数，该参数结构体定义如下:

```solidity
struct SwapParams {
    /// Whether to swap token0 for token1 or vice versa
    bool zeroForOne;
    /// The desired input amount if negative (exactIn), or the desired output amount if positive (exactOut)
    int256 amountSpecified;
    /// The sqrt price at which, if reached, the swap will stop executing
    uint160 sqrtPriceLimitX96;
}
```

这里需要注意一个有趣的部分，在 Uniswap v3 中，`amountSpecified > 0` 代表 `exactIn` 模式，但在 Uniswap v4 内 `amountSpecified > 0` 则代表 `exactOut` 模式。这是因为在 Uniswap v4 中，对于某些存在正负数值的变量统一使用用户视角，即资产从用户转移给流动性池，那么代表转移数量的数值应该为负数，反之，如果资产从 Pool 流向用户，那么代表转移数量的数值为正值。比如 `exactOut` 模式下 `amountSpecified` 代表用户从 Pool 内获得的代币数量，所以 `amountSpecified > 0`。一个更加明显的例子 `_accountDelta` 内部的一些参数正负问题。比如在 `mint` 函数，因为 `mint` 是用户减少个人资产增加 Pool 资产，所以 `mint` 函数内部使用了负数，即 `_accountDelta(currency, -(amount.toInt128()), msg.sender);` 

在 `swap` 函数内部，我们第一步仍旧是进行 `PoolKey` 参数的校验，校验相关代码如下:

```solidity
if (params.amountSpecified == 0) SwapAmountCannotBeZero.selector.revertWith();
PoolId id = key.toId();
Pool.State storage pool = _getPool(id);
pool.checkPoolInitialized();
```

之后，我们会调用 `beforeSwap` 函数通知 hook 合约，然后调用 `_swap` 的内部函数，该内部函数会返回 `swapDelta`，该参数代表当前用户在此次 swap 中造成代币余额变化。然后调用 `afterSwap` Hook，最终将 `swapDelta` 累加到用户的账户内部。

```solidity
BeforeSwapDelta beforeSwapDelta;
{
    int256 amountToSwap;
    uint24 lpFeeOverride;
    (amountToSwap, beforeSwapDelta, lpFeeOverride) = key.hooks.beforeSwap(key, params, hookData);

    // execute swap, account protocol fees, and emit swap event
    // _swap is needed to avoid stack too deep error
    swapDelta = _swap(
        pool,
        id,
        Pool.SwapParams({
            tickSpacing: key.tickSpacing,
            zeroForOne: params.zeroForOne,
            amountSpecified: amountToSwap,
            sqrtPriceLimitX96: params.sqrtPriceLimitX96,
            lpFeeOverride: lpFeeOverride
        }),
        params.zeroForOne ? key.currency0 : key.currency1 // input token
    );
}

BalanceDelta hookDelta;
(swapDelta, hookDelta) = key.hooks.afterSwap(key, params, swapDelta, hookData, beforeSwapDelta);

// if the hook doesn't have the flag to be able to return deltas, hookDelta will always be 0
if (hookDelta != BalanceDeltaLibrary.ZERO_DELTA) _accountPoolBalanceDelta(key, hookDelta, address(key.hooks));

_accountPoolBalanceDelta(key, swapDelta, msg.sender);
```

此处比较复杂的是 `SwapParams` 参数的构造，我们可以看到由于动态费率的引入，构建 `SwapParams` 时需要手动填写 `lpFeeOverride` 参数来重载原有的 LP 费率。

上述函数中最重要的就是 `_swap` 函数，该函数定义如下:

```solidity
function _swap(Pool.State storage pool, PoolId id, Pool.SwapParams memory params, Currency inputCurrency)
    internal
    returns (BalanceDelta)
{
    (BalanceDelta delta, uint256 amountToProtocol, uint24 swapFee, Pool.SwapResult memory result) =
        pool.swap(params);

    // the fee is on the input currency
    if (amountToProtocol > 0) _updateProtocolFees(inputCurrency, amountToProtocol);

    // event is emitted before the afterSwap call to ensure events are always emitted in order
    emit Swap(
        id,
        msg.sender,
        delta.amount0(),
        delta.amount1(),
        result.sqrtPriceX96,
        result.liquidity,
        result.tick,
        swapFee
    );

    return delta;
}
```

上述函数的核心部分是 `pool.swap` 函数。该函数是相当复杂的，但是实际上该函数与 Uniswap v3 中的 swap 逻辑是完全一致的。在上一篇介绍 Uniswap v3 的博客内，我们没有介绍 Protocol Fee 的部分，在本文内，我们会介绍这一部分。注意，在后文内部，我们会跳过与 Uniswap V3 完全相同的代码介绍，只介绍部分 Uniswap v4 中出现的代码。

在 Uniswap V4 的 `swap` 函数的最初阶段，除了常规的缓存结构体数值填充外，我们可以看到以下两个有趣的手续费逻辑：

```solidity
uint256 protocolFee =
    zeroForOne ? slot0Start.protocolFee().getZeroForOneFee() : slot0Start.protocolFee().getOneForZeroFee();

// if the beforeSwap hook returned a valid fee override, use that as the LP fee, otherwise load from storage
// lpFee, swapFee, and protocolFee are all in pips
{
    uint24 lpFee = params.lpFeeOverride.isOverride()
        ? params.lpFeeOverride.removeOverrideFlagAndValidate()
        : slot0Start.lpFee();

    swapFee = protocolFee == 0 ? lpFee : uint16(protocolFee).calculateSwapFee(lpFee);
}
```

其中前者用于获得 `protocolFee` 的数值，此处使用的 `getZeroForOneFee` 和 `getOneForZeroFee` 的定义如下:

```solidity
function getZeroForOneFee(uint24 self) internal pure returns (uint16) {
    return uint16(self & 0xfff);
}

function getOneForZeroFee(uint24 self) internal pure returns (uint16) {
    return uint16(self >> 12);
}
```

该部分代码是技术就是在 `uint24` 的后 12 位和前 12 位获取手续费数值。

而 `lpFee` 段则展示假如 hook 在 `beforeSwap` 内部返回 `lpFeeOverride` 字段后的处理方案。可以看到实际上假如 hook 返回的 `lpFeeOverride` 不包含 `isOverride` 字段，该返回值并不会覆盖当前的 LP 费率。此处使用的 `isOverride` 的函数定义如下:

```solidity
uint24 public constant OVERRIDE_FEE_FLAG = 0x400000;

function isOverride(uint24 self) internal pure returns (bool) {
    return self & OVERRIDE_FEE_FLAG != 0;
}
```

假如 hook 返回的手续费费率声明了重载，那么我们就会使用 `removeOverrideFlagAndValidate` 处理，该函数定义如下：

```solidity
function removeOverrideFlag(uint24 self) internal pure returns (uint24) {
    return self & REMOVE_OVERRIDE_MASK;
}

function removeOverrideFlagAndValidate(uint24 self) internal pure returns (uint24 fee) {
    fee = self.removeOverrideFlag();
    fee.validate();
}
```

最后我们会计算整体的手续费情况。假如存在 `protocolFee`，那么我们会调用 `uint16(protocolFee).calculateSwapFee(lpFee)` 进行计算。此处使用的 `calculateSwapFee` 的定义如下:

```solidity
// The protocol fee is taken from the input amount first and then the LP fee is taken from the remaining
// The swap fee is capped at 100%
// Equivalent to protocolFee + lpFee(1_000_000 - protocolFee) / 1_000_000 (rounded up)
/// @dev here `self` is just a single direction's protocol fee, not a packed type of 2 protocol fees
function calculateSwapFee(uint16 self, uint24 lpFee) internal pure returns (uint24 swapFee) {
    // protocolFee + lpFee - (protocolFee * lpFee / 1_000_000)
    assembly ("memory-safe") {
        self := and(self, 0xfff)
        lpFee := and(lpFee, 0xffffff)
        let numerator := mul(self, lpFee)
        swapFee := sub(add(self, lpFee), div(numerator, PIPS_DENOMINATOR))
    }
}
```

上述计算出的费率是以 Protocol Fee 收取第一轮，然后由 LP Fee 对剩余部分进行收取费率，所以最终的费率应该为

$$
\begin{align}
fee &= protocolFee + lpFee * (100\\% - protocolFee) / 100\\% \\\\
&= protocolFee + lpFee - (lpFee * protocolFee) / 100\\%
\end{align}
$$

此处的 100% 就是上文内出现的 `uint256 internal constant PIPS_DENOMINATOR = 1_000_000;`。

在计算完成最终的手续费后，我们需要判断该手续费是否合理，所以在 `swap` 函数内出现了以下代码:

```solidity
if (swapFee >= SwapMath.MAX_SWAP_FEE) {
    // if exactOutput
    if (params.amountSpecified > 0) {
        InvalidFeeForExactOut.selector.revertWith();
    }
}
```

此处的 `MAX_SWAP_FEE = 100%`。当 `swapFee` 大于 100% 后，我们只能使用 `exactOut` 模式。因为假如在 `swapFee` 大于 100% 的情况下选择 `exactIn` 模式就会出现所有的输入资产都作为手续费的情况。

完成手续费校验后，`swap` 函数处理了一种特殊情况:

```solidity
// swapFee is the pool's fee in pips (LP fee + protocol fee)
// when the amount swapped is 0, there is no protocolFee applied and the fee amount paid to the protocol is set to 0
if (params.amountSpecified == 0) return (BalanceDeltaLibrary.ZERO_DELTA, 0, swapFee, result);
```

即输入的 `params.amountSpecified == 0` 的情况，此时说明 `swap` 已经完全完成，用户无需进一步进行 `swap` 的其他逻辑。由于 Uniswap v4 引入了 `beforeSwap` hook 可以修改用户 `amountToSwap` 变量，所以这种情况是有可能出现的。

接下来，`swap` 函数检查了用户输入的 `sqrtPriceLimitX96` 是否正确，此处相比于 Uniswap v3 使用的三目表达式，Uniswap V4 修改为了正常的 If-Else 判断增加了代码可读性。而相比于 Uniswap V3，V4 版本的代码增加了更加有效的错误输出。

```solidity
if (zeroForOne) {
    if (params.sqrtPriceLimitX96 >= slot0Start.sqrtPriceX96()) {
        PriceLimitAlreadyExceeded.selector.revertWith(slot0Start.sqrtPriceX96(), params.sqrtPriceLimitX96);
    }
    // Swaps can never occur at MIN_TICK, only at MIN_TICK + 1, except at initialization of a pool
    // Under certain circumstances outlined below, the tick will preemptively reach MIN_TICK without swapping there
    if (params.sqrtPriceLimitX96 <= TickMath.MIN_SQRT_PRICE) {
        PriceLimitOutOfBounds.selector.revertWith(params.sqrtPriceLimitX96);
    }
} else {
    if (params.sqrtPriceLimitX96 <= slot0Start.sqrtPriceX96()) {
        PriceLimitAlreadyExceeded.selector.revertWith(slot0Start.sqrtPriceX96(), params.sqrtPriceLimitX96);
    }
    if (params.sqrtPriceLimitX96 >= TickMath.MAX_SQRT_PRICE) {
        PriceLimitOutOfBounds.selector.revertWith(params.sqrtPriceLimitX96);
    }
}
```

上述代码在 Uniswap V3 内表述为:

```solidity
require(
    zeroForOne
        ? sqrtPriceLimitX96 < slot0Start.sqrtPriceX96 && sqrtPriceLimitX96 > TickMath.MIN_SQRT_RATIO
        : sqrtPriceLimitX96 > slot0Start.sqrtPriceX96 && sqrtPriceLimitX96 < TickMath.MAX_SQRT_RATIO,
    'SPL'
);
```

在 `swap` 的核心计算过程中，大部分代码都与 Uniswap v3 相等，但部分代码进行了优化，比如在 `computeSwapStep` 函数传参过程中，我们使用了 `SwapMath.getSqrtPriceTarget` 函数获得 `PriceTarget`。`SwapMath.getSqrtPriceTarget` 是一个无分枝的函数，优化了 gas。

在计算 `protocolFee` 时，也使用了更加优化的方案，代码如下：

```solidity
if (protocolFee > 0) {
    unchecked {
        // step.amountIn does not include the swap fee, as it's already been taken from it,
        // so add it back to get the total amountIn and use that to calculate the amount of fees owed to the protocol
        // cannot overflow due to limits on the size of protocolFee and params.amountSpecified
        // this rounds down to favor LPs over the protocol
        uint256 delta = (swapFee == protocolFee)
            ? step.feeAmount // lp fee is 0, so the entire fee is owed to the protocol instead
            : (step.amountIn + step.feeAmount) * protocolFee / ProtocolFeeLibrary.PIPS_DENOMINATOR;
        // subtract it from the total fee and add it to the protocol fee
        step.feeAmount -= delta;
        amountToProtocol += delta;
    }
}
```

Uniswap V4 中的 `swap` 函数在最后增加了结果向 `BalanceDelta` 类型的转化，代码如下：

```solidity
unchecked {
    // "if currency1 is specified"
    if (zeroForOne != (params.amountSpecified < 0)) {
        swapDelta = toBalanceDelta(
            amountCalculated.toInt128(), (params.amountSpecified - amountSpecifiedRemaining).toInt128()
        );
    } else {
        swapDelta = toBalanceDelta(
            (params.amountSpecified - amountSpecifiedRemaining).toInt128(), amountCalculated.toInt128()
        );
    }
}
```

这部分本质上就是对 Uniswap V3 中计算 `amount0` 和 `amount1` 的另一个版本:

```solidity
(amount0, amount1) = zeroForOne == exactInput
    ? (amountSpecified - state.amountSpecifiedRemaining, state.amountCalculated)
    : (state.amountCalculated, amountSpecified - state.amountSpecifiedRemaining);
```

关于 swap 的其他部分基本与 Uniswap v3 是一致的，读者可以自行阅读相关代码。

## 总结

相比于 Uniswap V3 而言，Uniswap V4 并没有修改 AMM 的逻辑，`swap`  的逻辑几乎与 Uniswap V3 完全一致，但是相比于 Uniswap V3，Uniswap V4 增加了大量细微的优化调整。这些优化调整的核心部分包括:

1. ERC6909 机制使得 Uniswap V4 内部可以直接存储代币降低 transfer 成本
2. ERC7751 带来的新的错误抛出，使得开发者和用户可以在不使用额外工具下定位发生问题的合约
3. Flash Accounting 带来的更加方便的最终结算，用户可以一次性完成多笔 swap 并在最后完成统一结算
4. Hooks 带来的无限的开发想象，Hooks 内的 `beforeSwap` 和 `afterSwap` 可以直接修改部分内部变量，赋予了应用更大的空间
5. Singleton Pool 降低了链式 swap 时的 gas 消耗
6. 大量使用内联汇编进行 gas 优化，可以看到很多操作相比于 Uniswap v3 有了更加优化的汇编方案
