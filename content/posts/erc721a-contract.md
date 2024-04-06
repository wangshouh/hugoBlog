---
title: "NFT合约分析:ERC721A"
date: 2023-02-06T10:27:17Z
tags: [nft]
---

## 概述

本文主要介绍标准NFT实现的一个变体，即`ERC721A`合约实现的相关细节。`ERC721A`是由著名NFT系列[Azuki](https://www.azuki.com/)提出，该系列NFT是著名的蓝筹NFT。本文主要聚焦于`Azuki`提出的`ERC721A`合约的代码细节分析。

与传统的`ERC721`实现相比，`ERC721A`在批量铸造(batch mint)方面具有显著的`gas`优势，这得益于`ERC721A`的惰性初始化方面的设计。关于`ERC721A`与普通`ERC721`实现的对比，我们将会在下文展开说明。

本文要求读者具有基础的`solidity`知识，希望读者对标准`ERC721`有所了解。

读者可在阅读本文前，酌情阅读以下参考材料:

- [ERC721A 官网](https://www.erc721a.org/)
- [ERC721A 官方仓库](https://github.com/chiru-labs/ERC721A)
- [Azuki ERC721A 介绍](https://www.azuki.com/erc721a)

本文基于目前的最新版本(`4.2.3`)合约代码进行分析。

## ERC721实现

由于下文涉及到`ERC721A`与`ERC721`的技术对比，考虑到部分读者可以对`ERC721`合约实现并不清楚，本节简要的介绍`ERC721`正常实现的铸造功能，本节主要基于[solmate](https://github.com/transmissions11/solmate/blob/main/src/tokens/ERC721.sol)的实现版本。

> `solmate`实现都较为短小精悍且经过`gas`优化，我个人较为推崇。`solmate`的`ERC721`实现仅有 231 行，读者可自行阅读。

在`solmate`合约中，我们可以看到核心数据结构为:

```solidity
mapping(uint256 => address) internal _ownerOf;
mapping(address => uint256) internal _balanceOf;
```

其中，各映射功能如下:

- `_ownerOf` 记录 tokenId 与持有者的关系
- `_balanceOf` 记录持有人所持有的 NFT 数量

其铸造方法定义如下:

```solidity
function _mint(address to, uint256 id) internal virtual {
    require(to != address(0), "INVALID_RECIPIENT");

    require(_ownerOf[id] == address(0), "ALREADY_MINTED");

    // Counter overflow is incredibly unrealistic.
    unchecked {
        _balanceOf[to]++;
    }

    _ownerOf[id] = to;

    emit Transfer(address(0), to, id);
}
```

通过此函数，我们更新了`_ownerOf`和`_balanceOf`实现用户铸造 NFT 的功能。我们可以发现用户每次铸造NFT都需要更新`_ownerOf`和`_balanceOf`映射。众所周知，在操作码`gas`消耗中，更新存储需要消耗大量`gas`。如果用户批量铸造，会在此过程中消耗大量`gas`。

> 根据[数据](https://crocswap-assets-public.s3.us-east-2.amazonaws.com/EVMGasOptim.pdf)(PDF警告)，在ETH价格为 1500 美元时，更新存储的价格为 7.5 美元，而写入存储的价格为 30 美元。这意味着仅在`mint`过程中，更新映射会浪费大量资产。

转账函数定义如下:

```solidity
function transferFrom(
    address from,
    address to,
    uint256 id
) public virtual {
    require(from == _ownerOf[id], "WRONG_FROM");

    require(to != address(0), "INVALID_RECIPIENT");

    require(
        msg.sender == from || isApprovedForAll[from][msg.sender] || msg.sender == getApproved[id],
        "NOT_AUTHORIZED"
    );

    // Underflow of the sender's balance is impossible because we check for
    // ownership above and the recipient's balance can't realistically overflow.
    unchecked {
        _balanceOf[from]--;

        _balanceOf[to]++;
    }

    _ownerOf[id] = to;

    delete getApproved[id];

    emit Transfer(from, to, id);
}
```

由于对于每个`tokenId`都维护有一个`mapping`映射，所以转账逻辑实现也较为简单。

总体来看，对于每一个NFT，在`solmate`实现的智能合约中，都维持有以下两个映射:

- `mapping(uint256 => address) internal _ownerOf;` 标识NFT的拥有者
- `mapping(uint256 => address) public getApproved;` 记录NFT的授权情况

## 优势

在上一节中，我们介绍了常规NFT实现的基本情况，正如上文所述，常规实现在批量`mint`铸造阶段会消耗大量`gas`。为了解决这一问题，`ERC721A`引入惰性初始化机制。简单来说，在批量铸造时，不再记录`tokenId`与用户地址的映射关系，而是记录起始`tokenId`和数量与用户的映射关系。在本节中，我们不对此实现的技术细节进行分析，我们会在本文稍后部分对此进行讨论。

在批量铸造阶段，`ERC721A`与`OpenZeppelin`实现的对比如下:

|                            | ERC721       | ERC721A        |
| -------------------------- | ------------ | -------------- |
| 批量铸造 5 个 NFT       | 155949 gas   | 63748 gas      |
| 转移 5 个 NFT         | 226655 gas   | 334450 gas     |
| 铸造的 Base Fee              | 200 gwei     | 200 gwei       |
| 转移的 Base Fee          | 40 gwei      | 40 gwei        |
| 总花费    | 0.0403 ether | 0.0261 ether   |

> 如果读者对于此处的`gas`计算的细节感兴趣，可以阅读[以太坊机制详解:Gas Price计算](https://hugo.wongssh.cf/posts/ethereum-gas/)。我们在此处不详细讨论计算方式。我们可以注意到铸造阶段的`Base fee`较高，这考虑到了NFT铸造导致的网络拥堵情况。

显然，惰性初始化机制对于批量铸造阶段的`gas`节省是具有明显优势的，但惰性加载将初始化的成本转移到了转账部分，我们可以看到在转移NFT时的成本有所上升。但需要注意，第一次转账后由于彻底完成了初始化，所有后续转账的成本会降低，如下:

|                            | ERC721       | ERC721A        |
| -------------------------- | ------------ | -------------- |
| First transfer             | 45331 gas    | 92822 gas      |
| Subsequent transfers       | 45331 gas    | 44499 gas      |

通过表格可以看出，除第一次转账消耗的`gas`明显增多，但随后转账的价格与常规的NFT转账并无区别。

总结来说，`ERC721A`实现了低成本的批量铸造，但将部分成本转移到了第一次转账中。这种设计充分考虑到了铸造阶段可能出现的以太坊网络拥堵而造成`gas`价格飙升的情况，而用户后期转账是偶发的且不会导致网络拥堵的。通过这种特殊的成本转嫁机制，`ERC721A`降低用户的总成本。

> 换言之，如果您认为您的NFT项目不存在批量铸造的情况或不会导致以太坊网络拥堵，可以选择常规NFT实现。

## 具体实现

在讨论了`ERC721A`的基本内容后，为进一步增加我们对`ERC721A`的理解，我们将对其合约进行阅读分析。`ERC721A`的开源仓库位于[github](https://github.com/chiru-labs/ERC721A/blob/main/contracts/ERC721A.sol)。此处，我们仅讨论`ERC721A`的主合约，而暂不讨论`extensions`部分。

对于NFT合约的分析，存储数据结构和`_mint`函数是一个很好的入手点。我们首先关注存储数据结构。

在NFT数据存储中，我们可以看到`solmate`等常规实现都使用了`mapping(uint256 => address) internal _ownerOf`将单个`tokenId`与持有者对应。但`ERC721A`是对批量铸造进行特殊优化的，开发者认为在批量铸造过程中，用户持有的NFT的`tokenId`往往是连续的，如下图:

![ERC721A TokenId](https://img.gejiba.com/images/0fef83638b0ce7f4838d4a02283c82d8.png)

### 基本数据结构

在批量铸造过程中，用户铸造连续的NFT是极其常见的。为了实现连续分配`tokenID`以降低`gas`消耗的目的，我们需要一些更加复杂的数据结构设计，具体代码设计如下:

```solidity
// The next token ID to be minted.
uint256 private _currentIndex;

// The number of tokens burned.
uint256 private _burnCounter;

// Token name
string private _name;

// Token symbol
string private _symbol;

// Mapping from token ID to ownership details
// An empty struct value does not necessarily mean the token is unowned.
// See {_packedOwnershipOf} implementation for details.
//
// Bits Layout:
// - [0..159]   `addr`
// - [160..223] `startTimestamp`
// - [224]      `burned`
// - [225]      `nextInitialized`
// - [232..255] `extraData`
mapping(uint256 => uint256) private _packedOwnerships;

// Mapping owner address to address data.
//
// Bits Layout:
// - [0..63]    `balance`
// - [64..127]  `numberMinted`
// - [128..191] `numberBurned`
// - [192..255] `aux`
mapping(address => uint256) private _packedAddressData;

// Mapping from token ID to approved address.
mapping(uint256 => TokenApprovalRef) private _tokenApprovals;

// Mapping from owner to operator approvals
mapping(address => mapping(address => bool)) private _operatorApprovals;
```

与其他简单参数相比，我们主要关注复杂的参数:

1. `_packedOwnerships` 类似常规NFT实现中的`_ownerOf`，我们通过此映射查询某 `tokenID` 的拥有者，但此结构是打包方式的，即我们并不指定每一个 tokenID 对应的拥有者而是仅记录开头
1. `_packedAddressData` 类似常规NFT实现中的 `_balanceOf` ，用于查询某一用户所拥有的NFT的相关数据。此处的`aux`是指附加信息，比如用户当前使用的NFT铸造白名单数量，请根据自身项目酌情修改

此处，我们简单介绍数据读取的部分函数，关于在`uint256`压缩数据结构内进行数据读取的具体方法，我们已在 [深入解析AAVE智能合约:存款](https://blog.wssh.trade/posts/aave-contract-part1#特殊数据结构) 介绍过类似的`uint256`压缩数据提取方法。简单来说，就是使用`&`操作的特性实现数据提取。我们给出`balanceOf`的代码实现:

```solidity
function balanceOf(address owner) public view virtual override returns (uint256) {
    if (owner == address(0)) _revert(BalanceQueryForZeroAddress.selector);
    return _packedAddressData[owner] & _BITMASK_ADDRESS_DATA_ENTRY;
}
```

基于 `1 & 1 = 1` 、 `0 & 1 = 0`和`0 & 0 = 0`，我们可以通过将待提取位数(此处为0至63位置为 1 即可)。此处的`_BITMASK_ADDRESS_DATA_ENTRY`与我们设想的类似:

```solidity
uint256 private constant _BITMASK_ADDRESS_DATA_ENTRY = (1 << 64) - 1;
```

> 根据我们的设想，此处应填写 `0xffffffffffffffff`(总计 16 个 `f`)，正好为 0-63 位均为 1 。但 `ERC721A`开发者团队使用了位移方法表示，事实上是一致的

对于其他并不是从 0 开始的元素提取，我们需要使用位移以移除不必要数据，此处以提取 `numberMinted` 为例进行分析:

```solidity
function _numberMinted(address owner) internal view returns (uint256) {
    return (_packedAddressData[owner] >> _BITPOS_NUMBER_MINTED) & _BITMASK_ADDRESS_DATA_ENTRY;
}
```

首先将数据右移 64 位(即`_BITPOS_NUMBER_MINTED`)使`balance`占用的数据因溢出而移除，而后使用 `&` 操作符提取对应的数据，此处也需要提取 64 位数据，所以仍使用了`_BITMASK_ADDRESS_DATA_ENTRY`

对于其他数据的提取，我们不再赘述。

在数据写入函数方面，ERC721A 仅提供`_setAux`函数，该函数的实现代码如下:

```solidity
function _setAux(address owner, uint64 aux) internal virtual {
    uint256 packed = _packedAddressData[owner];
    uint256 auxCasted;
    // Cast `aux` with assembly to avoid redundant masking.
    assembly {
        auxCasted := aux
    }
    packed = (packed & _BITMASK_AUX_COMPLEMENT) | (auxCasted << _BITPOS_AUX);
    _packedAddressData[owner] = packed;
}
```

首先我们将输入的`aux`变量转化为`uint256`类型，以方便后期处理。此后，我们将`packed`与`(1 << 192) - 1`进行 `&` 操作，此步骤可以将 `aux` 占用 `[192..255]` 重置为 0 ，然后使用 `|` 操作符向该区域内填入最新的`aux`。

> 总结来说，我们可以通过与指定区域置为 1 的 `mask` 进行 `&` 操作提取指定区域内的数据。另一方面，我们可以通过 `|` 操作向置为 0 的区域写入数据。

### 铸造

#### 基本函数

铸造使用了`_mint`函数，其函数定义是:

```solidity
function _mint(address to, uint256 quantity) internal virtual
```

该函数规定了以下参数:

- `to` 铸造NFT接受地址
- `quantity` 铸造的NFT数量

> 由于`ERC721A`只能铸造固定数量的 NFT，所以无法指定铸造NFT的`tokenID`

其函数的运行逻辑简单如下:

1. 运行 `_beforeTokenTransfers`，此函数应根据具体目的编写
1. 设置 `_packedOwnerships`,以方便查询NFT的拥有者
1. 设置`_packedAddressData`,方便查询某一用户的所有NFT
1. 释放`Transfer`事件
1. 运行 `_afterTokenTransfers`，此函数应根据具体目的编写

接下来，我们将结合代码进行分析。

最先运行的 `_beforeTokenTransfers` 和最后运行的 
`_afterTokenTransfers` 都是由用户自定义的函数，用于实现白名单等功能。函数具体定义如下:

```solidity
function _beforeTokenTransfers(
    address from,
    address to,
    uint256 startTokenId,
    uint256 quantity
) internal virtual {}

function _afterTokenTransfers(
    address from,
    address to,
    uint256 startTokenId,
    uint256 quantity
) internal virtual {}
```

读者可根据自身需求，通过继承覆盖的方式定义这两个函数。

接下来，我们设置一些核心数据，这些数据的设置是 `_mint` 函数的核心。值得注意的是，这些函数都定义在 `unchecked` 代码块中，因为 NFT 的各个参数设置不会产生溢出情况，通过 `unchecked` 可以避免编译过程中插入溢出检查代码以减少 gas 消耗。

> 简而言之，在某些已经确定不会出现数据溢出的场景中使用 `unchecked` 包裹代码可以减少 gas 消耗

最开始，我们设置表示 NFT 所有者的 `_packOwnershipData` 数据结构，具体设置方法如下:

```solidity
_packedOwnerships[startTokenId] = _packOwnershipData(
    to,
    _nextInitializedFlag(quantity) | _nextExtraData(address(0), to, 0)
);
```
为方便读者理解代码，在此处，我们给出 `_packedOwnerships` 的定义:

```solidity
// Bits Layout:
// - [0..159]   `addr`
// - [160..223] `startTimestamp`
// - [224]      `burned`
// - [225]      `nextInitialized`
// - [232..255] `extraData`
mapping(uint256 => uint256) private _packedOwnerships;
```

我们先对 `_packOwnershipData` 函数的输入参数进行分析，需要解决 `_nextInitializedFlag` 和 `_nextExtraData` 的定义问题，

前者定义如下:

```solidity
function _nextInitializedFlag(uint256 quantity) private pure returns (uint256 result) {
    // For branchless setting of the `nextInitialized` flag.
    assembly {
        // `(quantity == 1) << _BITPOS_NEXT_INITIALIZED`.
        result := shl(_BITPOS_NEXT_INITIALIZED, eq(quantity, 1))
    }
}
```

显然，此函数用于设置 `nextInitialized` 标识，如果铸造的数量为 1 ，我们将此标识置为 1 (即 True )。当然，我们也使用了位移操作使其处于合适的位置。


> `nextInitialized` 是初始化的标识，如果此标识为 `True` 则说明此 NFT 对应的地址已被初始化。如果此标识为 `False` (正如上文所见，单次铸造多于 1 个 NFT 就会使标识为 `False` )，则意味着这段连续的 NFT 中除第一个外其他 NFT 均为初始化。如下图:
> ![ERC721A Init](https://files.catbox.moe/20sjdu.svg)

后者定义如下:

```solidity
function _nextExtraData(
    address from,
    address to,
    uint256 prevOwnershipPacked
) private view returns (uint256) {
    uint24 extraData = uint24(prevOwnershipPacked >> _BITPOS_EXTRA_DATA);
    return uint256(_extraData(from, to, extraData)) << _BITPOS_EXTRA_DATA;
}
```

此函数用于写入额外的信息，开发者需要自行定义 `_extraData` 函数以实现相关数据的写入。

此过程的核心函数为 `_packOwnershipData` ，其定义如下:

```solidity
function _packOwnershipData(address owner, uint256 flags) private view returns (uint256 result) {
    assembly {
        // Mask `owner` to the lower 160 bits, in case the upper bits somehow aren't clean.
        owner := and(owner, _BITMASK_ADDRESS)
        // `owner | (block.timestamp << _BITPOS_START_TIMESTAMP) | flags`.
        result := or(owner, or(shl(_BITPOS_START_TIMESTAMP, timestamp()), flags))
    }
}
```

有了上述 `_nextInitializedFlag` 和 `_nextExtraData` 的补充和注释，相信读者可以理解 `_packOwnershipData` 的实现原理，简单来说，该函数使用 `or` 操作符拼接 `owner` 、 `timestamp` 和 `flags` 以实现最终的数据结构。显然，我们只需要构造以下部分作为`flags`输入，即可完成 `_packOwnershipData` 的构造:

```
// - [224]      `burned`
// - [225]      `nextInitialized`
// - [232..255] `extraData`
```

> 读者可以注意到 `owner` 、 `timestamp` 和 `flags` 均为 `uint256` 数据类型，所以直接使用 `or` 进行拼接是合适的

接下来设置 `_packedAddressData` 数据结构。此数据结构定义如下:

```solidity
// Bits Layout:
// - [0..63]    `balance`
// - [64..127]  `numberMinted`
// - [128..191] `numberBurned`
// - [192..255] `aux`
mapping(address => uint256) private _packedAddressData;
```

`mint` 过程仅涉及 `balance` 和 `numberMinted` 两部分数据。所以设置较为简单，代码如下:

```solidity
_packedAddressData[to] += quantity * ((1 << _BITPOS_NUMBER_MINTED) | 1);
```

我们使用 `((1 << _BITPOS_NUMBER_MINTED) | 1)` 构造(此处 `_BITPOS_NUMBER_MINTED = 64` )出如下二进制数字 (以 16 进制表示):

```
0b10000001
```

> 使用 Python 运行 `bin((64 << 1) | 1)` 可以获得此结果

所以我们可以直接将数字与 `balance` 和 `numberMinted` 对齐相加。

在释放 `Transfer` 事件前，我们需要对 NFT 接受方的地址进行简单校验，即保证 NFT 接受方的地址不为 0 地址，校验代码如下:

```solidity
uint256 toMasked = uint256(uint160(to)) & _BITMASK_ADDRESS;

if (toMasked == 0) _revert(MintToZeroAddress.selector);
```

此处进行了一个有趣的操作，将地址转化为 `uint256` 后与 `0` 进行比较。此处涉及 `address` 与 `uint256` 类型的转化。众所周知， `address` 类型事实上就是 `uint160` ，两者可以直接转化。

> 如果读者对 `address` 类型不熟悉，可参考 [文档](https://docs.soliditylang.org/en/latest/types.html#address)

在直接转化后，为了避免直接转化导致的高位不为 0 的特殊情况出现，我们使用 `_BITMASK_ADDRESS` 进行清理。此常量定义如下:

```solidity
uint256 private constant _BITMASK_ADDRESS = (1 << 160) - 1;
```

通过使用此常量进行 `&` ，我们可以保证 `address` 与 `uint256` 的安全转换。

> **特殊情况** 指用户在调用函数时使用 `uint256` 类型的 `to` 进行调用。在 `abi` 打包时，`address` 和 `uint256` 打包结果一致，都可以进行函数调用，可能存在用户使用 `uint256` 类型的 `to` 进行函数调用。

> 事实上，上述 `& _BITMASK_ADDRESS` 可能是一个多余操作，具体讨论请参考评论区。

释放 `Transfer` 事件，此处我们可以一窥 `emit` 背后的原理:

```solidity
uint256 end = startTokenId + quantity;
uint256 tokenId = startTokenId;

do {
    assembly {
        // Emit the `Transfer` event.
        log4(
            0, // Start of data (0, since no data).
            0, // End of data (0, since no data).
            _TRANSFER_EVENT_SIGNATURE, // Signature.
            0, // `address(0)`.
            toMasked, // `to`.
            tokenId // `tokenId`.
        )
    }
    // The `!=` check ensures that large values of `quantity`
    // that overflows uint256 will make the loop run out of gas.
} while (++tokenId != end);
```

常规实现中， `Transfer` 定义如下:

```solidity
event Transfer(address indexed _from, address indexed _to, uint256 indexed _tokenId);
```

> 来自 [EIP-721 标准](https://eips.ethereum.org/EIPS/eip-721#specification) 原文

我们可以看到此事件抛出了 3 个 `topic`，但事实上 `Transfer` 作为事件名称也需要占用一个 `topic` ，所以此处使用了 `log4` 操作码。

此操作码需要的变量如下:

1. `offset` 抛出内容位于内存的起始位置
1. `size` 抛出内容的长度(与 `offset` 参数共同使用)
1. `topic1` 抛出的的变量
1. `topic2`
1. `topic3`
1. `topic4`

> 有读者好奇为什么存在 `offset` 和 `size` 参数？ 如果读者仔细阅读过 Events 部分的 [Solidity 文档](https://docs.soliditylang.org/en/latest/abi-spec.html#events) 就会理解这一问题。 文档中明确指出 `events` 可以提供合约地址、 最多 4 个 `topic` 和一些任意长度二进制数据。此处的 `offset` 和 `size` 参数就是指明任意长度的二进制数据的

> 在编写 `solidity` 代码时，假设存在 `event foo(uint256 _a, uint256 indexed _b)` 定义，其中 `_a` 会以二进制数据的形式抛出(即通过 `offset` 和 `size` 定义抛出)，而 `_b` 则以 `topic` 的形式抛出。

至此，读者应该可以很好的理解 `log4` 在代码中的具体功能。此处也使用了 `do while` 循环以逐一抛出每个 `tokenId` 的 `Transfer` 事件。

#### 补充函数

在 `ERC721A` 的官方实现中，开发者提供了一些其他的 `mint` 函数实现，这些实现的主体逻辑与 `_mint` 类似，但提供了一些特别的功能或者符合一些特定的 ERC 标准。

我们首先分析 `_mintERC2309` 函数，此函数根据 [ERC 2309](https://eips.ethereum.org/EIPS/eip-2309) 标准编写。在介绍函数具体实现前，我们简单介绍一下 `ERC 2309` 的具体内容。

`ERC 2309` 主要解决在大规模铸造和代币转账过程中释放过多 `event` 的问题。如在标准 `_mint` 函数实现中，我们在最后使用了 `while` 循环以逐一释放事件。这显然是低效的，且无法用于大规模代币铸造。

为解决这一问题， `ERC 2309` 的开发者设计了一个新的事件:

```solidity
event ConsecutiveTransfer(uint256 indexed fromTokenId, uint256 toTokenId, address indexed fromAddress, address indexed toAddress);
```

基于此事件，我们可以一次性释放所有代币转移的事件，大大降低了 gas 消耗。

对于 `_mintERC2309` 具体实现，与 `_mint` 基本一致，除了增加了以下代码:

1. ERC2309 最大转移量检查
    `if (quantity > _MAX_MINT_ERC2309_QUANTITY_LIMIT) _revert(MintERC2309QuantityExceedsLimit.selector);`
    用于判断单次转移量是否超过 `5000`
1. `ConsecutiveTransfer` 事件抛出
    `emit ConsecutiveTransfer(startTokenId, startTokenId + quantity - 1, address(0), to);`
    由于使用了 `solidity` 语法编写，所以此处也减少了大量安全性代码编写(如上文的 `address` 到 `uint256` 转化等)。

另一个实现 `mint` 功能的函数是 `_safeMint` 函数，此函数会判断 NFT 接收地址 `to` 的属性，以避免 NFT 接受方不具有接受 NFT 的能力。

此部分逻辑代码如下:

```solidity
unchecked {
    if (to.code.length != 0) {
        uint256 end = _currentIndex;
        uint256 index = end - quantity;
        do {
            if (!_checkContractOnERC721Received(address(0), to, index++, _data)) {
                _revert(TransferToNonERC721ReceiverImplementer.selector);
            }
        } while (index < end);
        // Reentrancy protection.
        if (_currentIndex != end) _revert(bytes4(0));
    }
}
```

当接受方为一合约地址时，我们需要使用 `_checkContractOnERC721Received` 函数判断接受方是否可以接受 NFT，此函数定义如下:

```solidity
function _checkContractOnERC721Received(
    address from,
    address to,
    uint256 tokenId,
    bytes memory _data
) private returns (bool) {
    try ERC721A__IERC721Receiver(to).onERC721Received(_msgSenderERC721A(), from, tokenId, _data) returns (
        bytes4 retval
    ) {
        return retval == ERC721A__IERC721Receiver(to).onERC721Received.selector;
    } catch (bytes memory reason) {
        if (reason.length == 0) {
            _revert(TransferToNonERC721ReceiverImplementer.selector);
        }
        assembly {
            revert(add(32, reason), mload(reason))
        }
    }
}
```

我们在 [深入解析Safe多签钱包智能合约:Fallback合约](https://blog.wssh.trade/posts/deep-in-safe-part-3/#receiver) 内已经对 `onERC721Received` 的相关内容进行了分析，读者可自行阅读理解。此处，我们主要对 `try/catch` 这一少见的 `solidity` 关键词进行分析。

`try` 关键词后必须为一个外部函数调用，在此处为 
`ERC721A__IERC721Receiver(to).onERC721Received(_msgSenderERC721A(), from, tokenId, _data)`，即调用了外部 `ERC721A__IERC721Receiver` 的 `onERC721Received` 函数。 `return` 会将外部调用的返回值封装为特定的函数名，此处为 `retval` 。

如果外部调用和返回值封装没有出现错误，就会运行第一个语句块的语句，此处为 
`return retval == ERC721A__IERC721Receiver(to).onERC721Received.selector;`

该语句块较为简单，不再具体分析。

`catch` 用来捕获错误， `solidity` 提供了以下 `catch` 语句:

- `catch Error(string memory reason) { ... }` 用于捕获 `revert("reasonString")` 或 `require(false, "reasonString")` 等语句造成的错误
- `catch Panic(uint errorCode) { ... }` 用于捕获 `panic` 类型错误，如 `assert` 、除以 0 等错误
- `catch (bytes memory lowLevelData) { ... }` 用于直接捕获底层错误信息，涵盖所有类型错误

> 在真实场景下，显然我们无法保证调用的合约使用 `solidity` 编写，所以使用最后一张 `catch` 方法是有必要的。

显然，此处使用的是最后一种 `catch` 语句。在捕获到底层错误后，我们首先使用 `if` 语句判断此错误信息是否长度为 `0` ，如果长度为 0 ，则意味着我们没有具体的错误信息，采取直接抛出 `TransferToNonERC721ReceiverImplementer.selector` 的策略。

此处使用了 `_revert` 函数，此函数是对 `revert` 包装，定义如下:

```solidity
function _revert(bytes4 errorSelector) internal pure {
    assembly {
        mstore(0x00, errorSelector)
        revert(0x00, 0x04)
    }
}
```

此函数是对抛出 `errorSelector` 错误信息的 `revert` 的包装。读者应该可以理解此函数内部的 `yul` 代码，较为简单。

如果错误信息 `reason` 长度不为 0 ，我们则考虑抛出此信息。使用 `revert` 抛出错误信息是一个好的选择。

> 可能有读者对 `revert` 操作码不熟悉，此操作码会抛出指定的错误信息、回滚当前状态并返还未使用的 `gas` 费用。使用 `revert` 操作码可以构建出稳赚不陪的偷跑(`front-running`)机器人，可参考 [Setting Bear Traps in the Dark Forest](https://paulbrower.codes/posts/bear-traps-in-the-dark-forest/) 。

`revert(offset, size)` 需要以下参数以抛出错误信息:

1. `offset` 错误信息在内存中的起始位置
1. `size` 错误信息的长度

由于 `reason` 属于 `bytes` 类型，此类型属于 `array` ，其在内存中的存在方式如下图:

```
+--------+--------+
| length |  ....  |
+--------+--------+
|         \_______/
|             length     
reason
```

`reason` 在内存中大致如上图。其在内存中的起始位置保存在 `reason` 代表的数字中，然后 `32 bytes` 是变量占据的内存长度，而后 `length` 长度的内容为其真正存储的内容。

> 如果读者阅读过我之前的一系列关于智能合约的文章，相信可以理解这一内容。简单来说，在 `solidity` 内所有变量都是指向内存特定位置的指针。但由于数据类型的不同，其在内存中的结构也不相同，可以参考 [solidity 文档](https://docs.soliditylang.org/en/latest/internals/layout_in_memory.html) 

有了上述内容，我们可以理解 `revert(add(32, reason), mload(reason))` 的具体含义。

我们使用 `add(32, reason)` 跳过 `reason` 的长度部分以其内容的起始部分作为 `offset` ，使用 `mload(reason)` 读取 `reason` 的前 `32 bytes` ，这正是 `reason` 的长度信息。使用上述操作，可以保证 `revert` 抛出的错误信息不包含长度内容。

至此，我们完成了 `_safeMint` 的核心代码分析。

### 授权

授权，或称 `approve` 是 NFT 的核心逻辑之一，也是 NFT 可组合性的基础之一。

#### `_approve`

实现 `approve` 的核心函数为 `_approve` 函数，其代码如下:

```solidity
function _approve(
    address to,
    uint256 tokenId,
    bool approvalCheck
) internal virtual {
    address owner = ownerOf(tokenId);

    if (approvalCheck && _msgSenderERC721A() != owner)
        if (!isApprovedForAll(owner, _msgSenderERC721A())) {
            _revert(ApprovalCallerNotOwnerNorApproved.selector);
        }

    _tokenApprovals[tokenId].value = to;
    emit Approval(owner, to, tokenId);
}
```

其逻辑大致如下:

1. 查询待授权 NFT 的所有者
1. 进行资格审查，判断函数调用者是否有权进行授权
1. 设置 `_tokenApprovals` 映射，确定授权

在资格审查方面，要求函数调用者满足以下条件:

1. `approvalCheck` 为`false` 且函数调用者是 NFT 拥有者
1. `approvalCheck` 为 `true` 且函数调用者被授权控制 NFT 拥有者的 **所有** NFT

首先分析 `ownerOf` 函数，其定义如下:

```solidity
function ownerOf(uint256 tokenId) public view virtual override returns (address) {
    return address(uint160(_packedOwnershipOf(tokenId)));
}
```

显然，我们需要分析 `_packedOwnershipOf` 的实现:

```solidity
function _packedOwnershipOf(uint256 tokenId) private view returns (uint256 packed) {
    if (_startTokenId() <= tokenId) {
        packed = _packedOwnerships[tokenId];
        if (packed & _BITMASK_BURNED == 0) {
            if (packed == 0) {
                if (tokenId >= _currentIndex) _revert(OwnerQueryForNonexistentToken.selector);
                for (;;) {
                    unchecked {
                        packed = _packedOwnerships[--tokenId];
                    }
                    if (packed == 0) continue;
                    return packed;
                }
            }
            return packed;
        }
    }
    _revert(OwnerQueryForNonexistentToken.selector);
}
```

该函数的基本逻辑如下:

![packedOwnershipOf](https://blogimage.4everland.store/packedOwnershipOf.drawio.svg)

通过上述流程图，读者应该可以理解查询 `packed` 的流程，其中的核心步骤是 `for` 循环代码块内的回溯。正如在 [mint](#铸造) 所说明的，`_packedOwnerships` 内仅存储 `startTokenId` ，所以此处使用 `--tokenId` 进行回溯查询。

> 此处使用了映射的性质，如果映射中的键不存在，那么返回的值为空。此处使用的 `_packedOwnerships[tokenId]` 会在 `tokenId` 不存在时返回空值。

在理解 `_packedOwnershipOf` 和 `ownerOf` 的基础上，理解 `_approve` 实现是容易的。

#### 其他函数

本部分主要介绍关于 `approval` 授权相关的其他函数，这些函数在是实现上都较为简单。

```solidity
function setApprovalForAll(address operator, bool approved) public virtual override {
    _operatorApprovals[_msgSenderERC721A()][operator] = approved;
    emit ApprovalForAll(_msgSenderERC721A(), operator, approved);
}
```

此处使用了 `_operatorApprovals` 映射以实现将拥有者所有 NFT 同一授权为其他地址，映射定义如下:

```solidity
mapping(address => mapping(address => bool)) private _operatorApprovals;
```

`getApproved` 函数用于确定某个 NFT 被授权地址，实现如下:

```solidity
function getApproved(uint256 tokenId) public view virtual override returns (address) {
    if (!_exists(tokenId)) _revert(ApprovalQueryForNonexistentToken.selector);

    return _tokenApprovals[tokenId].value;
}
```

在返回被授权者前，该函数使用了 `_exists` 确定对应的 NFT 存在，`_exists` 实现如下:

```solidity
function _exists(uint256 tokenId) internal view virtual returns (bool) {
    return
        _startTokenId() <= tokenId &&
        tokenId < _currentIndex && // If within bounds,
        _packedOwnerships[tokenId] & _BITMASK_BURNED == 0; // and not burned.
}
```

配合注释，读者应该可以理解此函数的具体逻辑

### 转账


转账方面的基础函数为 `transferFrom` 函数，其他所有转账函数都建立在此函数的基础上，该函数的逻辑设计如下:

1. 使用 `_packedOwnershipOf` 函数获得 NFT 持有者地址
1. 校验函数请求者是否是 NFT 拥有者或具有授权
1. 删除待转移 NFT 的授权
1. 修改 `_packedAddressData` 映射增减 `balance`
1. 修改 `_packedOwnerships` 映射
1. 释放转移事件

函数定义如下:

```solidity
function transferFrom(
    address from,
    address to,
    uint256 tokenId
) public payable virtual override
```

该函数的参数为:

1. `from` 待转移 NFT 的拥有者地址
1. `to` 待转移 NFT 的接收者地址
1. `tokenId` 待转移 NFT 的 `tokenId`

根据上述流程，我们将逐个解析其中使用的函数。

```solidity
uint256 prevOwnershipPacked = _packedOwnershipOf(tokenId);

from = address(uint160(uint256(uint160(from)) & _BITMASK_ADDRESS));

if (address(uint160(prevOwnershipPacked)) != from) _revert(TransferFromIncorrectOwner.selector);
```

通过 `_packedOwnershipOf` 函数获得 NFT 拥有者地址，使用 `address(uint160(uint256(uint160(from)) & _BITMASK_ADDRESS))` 进行数据类型转化。如果我们发现调用参数中的 `from` 与 NFT 拥有者不同，则直接抛出错误。

接下来，我们使用以下代码校验 NFT 转移的相关权限问题:

```solidity
(uint256 approvedAddressSlot, address approvedAddress) = _getApprovedSlotAndAddress(tokenId);

if (!_isSenderApprovedOrOwner(approvedAddress, from, _msgSenderERC721A()))
    if (!isApprovedForAll(from, _msgSenderERC721A())) _revert(TransferCallerNotOwnerNorApproved.selector);
```

满足以下条件则继续运行:

函数调用者为 NFT 拥有者或被授权者 或 函数调用者存在 `isApprovedForAll` 权限。

如果上述条件全不满足，则抛出异常。

该部分中最复杂的函数为`_getApprovedSlotAndAddress`:

```solidity
function _getApprovedSlotAndAddress(uint256 tokenId)
    private
    view
    returns (uint256 approvedAddressSlot, address approvedAddress)
{
    TokenApprovalRef storage tokenApproval = _tokenApprovals[tokenId];
    assembly {
        approvedAddressSlot := tokenApproval.slot
        approvedAddress := sload(approvedAddressSlot)
    }
}
```

该函数会返回两个底层数据，即授权地址在 `storage` 中的位置`approvedAddressSlot`和授权地址的值 `approvedAddress`。

> 理解此代码需要对 EVM 的存储结构有一定了解，推荐阅读 [ Understanding Ethereum Smart Contract Storage ](https://programtheblockchain.com/posts/2018/03/09/understanding-ethereum-smart-contract-storage)

当函数调用者满足条件后，我们进入真正的 NFT 转移程序。首先清除待转移 NFT 的原有授权，代码如下:

```solidity
assembly {
    if approvedAddress {
        sstore(approvedAddressSlot, 0)
    }
}
```

直接将 `_tokenApprovals` 中 NFT 对应的值清空。

接下来，我们进入了最复杂的 NFT 转移阶段，该阶段的逻辑大致如下:

1. 修正转移双方的 `balance` 参数
    ```solidity
    --_packedAddressData[from];
    ++_packedAddressData[to]; 
    ```
1. 更新 `tokenId` 对应的 `_packedOwnerships` 数据:
    ```solidity
    _packedOwnerships[tokenId] = _packOwnershipData(
        to,
        _BITMASK_NEXT_INITIALIZED | _nextExtraData(from, to, prevOwnershipPacked)
    );
    ```
    由于转移过程必须进行初始化，所以此处将转移的 NFT 的 `nextInitialized` 设置为 `True`
1. 考虑下一个 NFT 是否被初始化，
    如转移下图中 `tokenId = 3` 的 NFT:
    ![NFT List Example](https://files.catbox.moe/20sjdu.svg)
    该 NFT 转移后，由于破坏了拥有者 `0x2` 的连续性，所以我们需要重写 `tokenId = 4` 的对应数据，代码如下:
    
    ```solidity
    if (prevOwnershipPacked & _BITMASK_NEXT_INITIALIZED == 0) {
    uint256 nextTokenId = tokenId + 1;
    if (_packedOwnerships[nextTokenId] == 0) {
        if (nextTokenId != _currentIndex) {
            _packedOwnerships[nextTokenId] = prevOwnershipPacked;
        }
    }
    ```

    此处使用了 `_packedOwnerships[nextTokenId] == 0` 排除了 `tokenId = 4` 转移的特殊情况。该 NFT 位于连续 NFT 的末尾，转移此 NFT 不会破环连续性

至此，我们完成了 NFT 的转移的核心流程。接下来就是已经介绍过的 `Transfer` 释放流程:

```solidity
uint256 toMasked = uint256(uint160(to)) & _BITMASK_ADDRESS;
assembly {
    // Emit the `Transfer` event.
    log4(
        0, // Start of data (0, since no data).
        0, // End of data (0, since no data).
        _TRANSFER_EVENT_SIGNATURE, // Signature.
        from, // `from`.
        toMasked, // `to`.
        tokenId // `tokenId`.
    )
}
if (toMasked == 0) _revert(TransferToZeroAddress.selector);
```

`safeTransferFrom` 作为 `transferFrom` 的安全版本，该函数只是增加了 `_checkContractOnERC721Received` 检测，此检测函数已在上文进行了介绍，此处不再赘述。

### 销毁

`burn` 销毁的核心函数为 `_burn` 函数，由于销毁事实上相当于将 NFT 转移给 0 地址，所以其大量逻辑与 `transfer` 类似。

`_burn` 函数定义如下:

```solidity
function _burn(uint256 tokenId, bool approvalCheck) internal virtual
```

参数含义如下:

1. `tokenId` 待销毁 NFT 的 `tokenId`
1. `approvalCheck` 是否检测函数调用者的权限

大致流程如下:

1. 获取待销毁 NFT 拥有者的信息
1. 如果设置 `approvalCheck` 为 `true` 则检测函数调用者的相关权限
1. 清空待销毁 NFT 的授权 `approve` 数据
1. 减少拥有者的 `balance`
1. 在 `_packedOwnerships` 中写入销毁信息
1. 恢复代币连续性
1. 释放事件

接下来，我们详细分析具体的代码实现:

```solidity
uint256 prevOwnershipPacked = _packedOwnershipOf(tokenId);

address from = address(uint160(prevOwnershipPacked));

(uint256 approvedAddressSlot, address approvedAddress) = _getApprovedSlotAndAddress(tokenId);
```

此处代码与 `transferFrom` 函数的开始部分基本一致，但在 `from` 处理方面进行了简化。

接下来，我们检查调用者的相关权限并清空授权，代码如下:

```solidity
if (approvalCheck) {
    if (!_isSenderApprovedOrOwner(approvedAddress, from, _msgSenderERC721A()))
        if (!isApprovedForAll(from, _msgSenderERC721A())) _revert(TransferCallerNotOwnerNorApproved.selector);
}

assembly {
    if approvedAddress {
        // This is equivalent to `delete _tokenApprovals[tokenId]`.
        sstore(approvedAddressSlot, 0)
    }
}
```

此部分代码与 `transferFrom` 函数完全一致，不再详细介绍。

```solidity
_packedAddressData[from] += (1 << _BITPOS_NUMBER_BURNED) - 1;
_packedOwnerships[tokenId] = _packOwnershipData(
    from,
    (_BITMASK_BURNED | _BITMASK_NEXT_INITIALIZED) | _nextExtraData(from, address(0), prevOwnershipPacked)
);
```

此处使用 `_packedAddressData[from] += (1 << _BITPOS_NUMBER_BURNED) - 1;` 代码将 `balance -= 1` 和 `numberBurned += 1` 合并一起执行。

其中 `_BITPOS_NUMBER_BURNED` 的值为 128，为方便读者理解，我们再次给出 `_packedAddressData` 的格式:

```
// Bits Layout:
// - [0..63]    `balance`
// - [64..127]  `numberMinted`
// - [128..191] `numberBurned`
// - [192..255] `aux`
mapping(address => uint256) private _packedAddressData;
```

为方便理解，我们将原有代码进行重写:

```solidity
_packedAddressData[from] = _packedAddressData[from] + (1 << 128) - 1
```

如此来看，我们首先使用加法完成了 `numberBurned` 的更新，然后使用减法完成了 `balance` 的更新。

对于 `_packOwnershipData` 函数，最重要的是分析以下部分:

```solidity
(_BITMASK_BURNED | _BITMASK_NEXT_INITIALIZED) | _nextExtraData(from, address(0), prevOwnershipPacked)
```
我们将 `burned` 和 `_BITMASK_NEXT_INITIALIZED` 置为 `True` 并写入 `extraData` 部分。

最后我们还是讨论 **连续性** 问题，假如当前的代币拥有如下图:

![ERC721A Owner](https://files.catbox.moe/20sjdu.svg)

我们将 `tokenId = 3` 的代币销毁，那么我们需要修正 `tokenId = 4` 的 NFT 以避免 NFT 丢失。这部分代码与 `transferFrom` 是一致的，实现如下:

```solidity
if (prevOwnershipPacked & _BITMASK_NEXT_INITIALIZED == 0) {
    uint256 nextTokenId = tokenId + 1;
    if (_packedOwnerships[nextTokenId] == 0) {
        if (nextTokenId != _currentIndex) {
            _packedOwnerships[nextTokenId] = prevOwnershipPacked;
        }
    }
}
```

简单来说，我们只需要将 `tokenId = 2` 的数据放入 `tokenId = 4` 的 NFT 中即可。

对于释放事件，使用了 `emit Transfer(from, address(0), tokenId);` 语句，较为简单。

> 有读者可能发现为什么在 `ERC721A` 内的编码风格并不统一，有使用底层 `log4` 释放事件的，有使用 `emit` 释放事件的。这可能是我没有使用 `Realse` 版本的代码而是直接 `clone` 了开发中的代码。

## 总结

在本文中，我们分析了 `ERC721A` 合约的主体逻辑，但仍存在部分代码没有分析。这些代码实现都较为简单，故不再本文继续介绍。

总体而言，`ERC721A`通过对连续 NFT 的合并处理大幅度降低了 NFT 批量铸造的 gas 消耗。