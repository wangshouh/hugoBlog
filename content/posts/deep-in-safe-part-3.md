---
title: 深入解析Safe多签钱包智能合约:Fallback合约
date: 2022-09-22T13:47:33Z
tags: [solidity,Safe,EIP-165]
aliases: ["/2022/09/20/deep-in-safe-part-3/"]
---

## 概述

在[深入解析Safe多签钱包智能合约:模块]({{<ref "deep-in-safe-part-2" >}})中分析`FallbackManager`模块时，限于篇幅限制且`fallback`合约自成一体，所以我们没有介绍具体的`fallback`模块。此篇文章的主要目的是完成这一缺陷，全面介绍`fallback`合约。

本文涉及的代码主要位于`src/handler`内，读者可自行查阅[此仓库](https://github.com/wangshouh/foundry-safe)。

## 合理性分析

此节主要关注于我们为什么需要`Fallback`合约这一主题，希望可以为读者在后文阅读源代码时起到提纲挈领的作用。

### Fallback

在上文中，我们可以知道`fallback`函数的主体逻辑是进行了代理合约式的处理将逻辑代码交给此处的`fallback`合约执行。我们首先应当知道`fallback`函数的作用，此函数用于接受一切无法与其他函数名匹配到的调用都会被发送到`fallback`函数中，进一步这些调用被转发到`fallback`合约内。

我们可以认为`fallback`合约提供了对于`GnosisSafe`主合约的强大补充，避免了`GnosisSafe`因不具有某些函数而导致无法进行关键功能。在目前，`fallback`合约提供了以下功能:

1. 向前兼容`1.3.0`之前的`safe`合约的功能
1. 接受 `ERC1155` 代币的功能
1. 接受 `ERC721` NFT 的功能
1. 基于`ERC165`实现的向外界暴露接口实现的功能

这些功能体现了`Safe`合约开发团队的一个基本目标，即尽可能保持`GnosisSafe`主合约的稳定性，将部分新的必要的功能放在可以通过简单的代理方式升级的`fallback`合约中。当然，也将部分兼容性功能放在了`fallback`合约内。

> 当然，这不意味着我们可以大肆修改`fallback`合约以增加功能。在代码设计上，复杂而非必要的功能应该以`modules`的形式开发。

在最后，我们希望可以获得任一`Safe`合约的`fallback`合约地址。为达成这一目的，我们需要查询`fallback`合约地址存储的变量。在`src/base/FallbackManager.sol`中，我们可以看到如下定义:
```solidity
bytes32 internal constant FALLBACK_HANDLER_STORAGE_SLOT = 0x6c9a6c4a39284e37ed1cf53d337577d14212a4870fb976a4366c693b939918d5;
```
此变量属于`internal`，这意味着似乎无法从外界进行读取。实际上，所有的`solidity`变量均可以被读取，但前提是需要知道变量的具体位置。而上述`FALLBACK_HANDLER_STORAGE_SLOT`正好告诉了我们变量的存储位置，我们可以通过以下命令读取:
```bash
cast storage 0xDE06d17Db9295Fa8c4082D4f73Ff81592A3aC437 0x6c9a6c4a39284e37
ed1cf53d337577d14212a4870fb976a4366c693b939918d5 --rpc-url https://rpc.ankr.com/eth
```
其中，`0xDE06d17Db9295Fa8c4082D4f73Ff81592A3aC437`为我选择的`Lido`名下的一个`safe`多签钱包地址，读者可以自行替换为其他多签钱包地址。

代码运行结果如下:
```
0x000000000000000000000000f48f2b2d2a534e402487b3ee7c18c33aec0fe5e4
```
这正是在存储中的“裸”地址(没有被编码)，其代表的真实地址为`f48f2b2d2a534e402487b3ee7c18c33aec0fe5e4`，其`etherscan`地址在[这](https://etherscan.io/address/0xf48f2b2d2a534e402487b3ee7c18c33aec0fe5e4)。

### Receiver

我们在上文提到`fallback`合约提供了这两个功能:

1. 接受 `ERC1155` 代币的功能
1. 接受 `ERC721` NFT 的功能

可能有读者比较困惑，我们直接进行转账等行为时似乎没有这些要求。这是因为我们一直使用的时非安全转账函数，如`transferFrom`。而`ERC721`和`ERC1155`都提供了安全转账函数`safeTransferFrom`。此函数提供一个对代币接受者的校验，避免代币转移到无法接受代币的地址内。

为了方便读者理解，我们给出一个`solmate`中的`NFT`合约中的安全转账函数实现，代码如下:
```solidity
function safeTransferFrom(
    address from,
    address to,
    uint256 id
) public virtual {
    transferFrom(from, to, id);

    require(
        to.code.length == 0 ||
            ERC721TokenReceiver(to).onERC721Received(msg.sender, from, id, "") ==
            ERC721TokenReceiver.onERC721Received.selector,
        "UNSAFE_RECIPIENT"
    );
}
```
我们可以看到此`safe`函数在`transferFrom`增加了`to.code.length == 0 || ERC721TokenReceiver(to).onERC721Received(msg.sender, from, id, "") == ERC721TokenReceiver.onERC721Received.selector`条件来校验接收地址`to`是否具有接受代币的资格。当地址满足以下两个条件之一即可接受代币:

1. 接受地址为EOA(即使用私钥控制的地址)，此条件提供`to.code.length == 0`判断地址是否存在代码来实现。当地址内没有代码时，我们可以认为此地址由用户私钥控制。

1. 接受地址实现了`onERC721Received`接口并返回`0x150b7a02`值

读者可能已经发现了此处要求接受合约满足实现接口的条件。而`GnosisSafe`开发者为了避免用户在使用`safe`系列转账函数时，合约无法接受代币，所以在`Fallback`合约内实现了这些要求的`Receiver`接口。

> 关于这些接口实现的细节，读者可以自行参考[ERC721](https://eips.ethereum.org/EIPS/eip-721)和[ERC1155](https://eips.ethereum.org/EIPS/eip-1155#safe-transfer-rules)中的详细规定。

除此之外，我们还在`fallback`合约中实现了另一个较为常用的`EIP-165`标准用来辅助实现`Receiver`接口的实现。`EIP165`的功能是方便其他合约可以通过`supportsInterface()`函数判断合约是否实现了某一个接口，代码如下:
```solidity
function supportsInterface(bytes4 interfaceId) external view virtual override returns (bool) {
    return
        interfaceId == type(ERC1155TokenReceiver).interfaceId ||
        interfaceId == type(ERC721TokenReceiver).interfaceId ||
        interfaceId == type(IERC165).interfaceId;
}
```
代码较为简单，其中`interfaceId`对于每一个接口都有唯一的值与之对应。其他合约可以提供一个特定接口的`interfaceId`调用`supportsInterface`函数，如果对方合约实现了此接口就会返回`True`，否则就返回`Fasle`。

## 代码分析

相信读者通过上一节已经对`fallback`合约基本功能有了一定的认识，此节我们会按照上文给出的`fallback`合约的功能逐一介绍代码实现。

### 兼容代码分析

在此部分，我们会看到一些在 1.2.0 版本`GnosisSafe`主合约中实现的函数，此部分函数很多都已被废弃，我们不建议使用此部分函数用于实际操作。
#### isValidSignature 1

此函数我们在`GnosisSafe`中介绍[合约签名]({{<ref "deep-in-safe-part-1#%E5%90%88%E7%BA%A6%E7%AD%BE%E5%90%8Dcontract-signature" >}})曾使用此函数。当合约实现此函数时，就意味着合约可以实现合约签名的特性。该特性由`EIP1271`规定，具体可以参考[基于链下链上双视角深入解析以太坊签名与验证]({{<ref "ecsda-sign-chain#eip-1271" >}})文章。

此函数需要以下参数:

- _data 需要验证签名的原数据
- _signature 签名数据，用户可以通过`getMessageHash`获得代签数据的哈希值后使用自己的私钥进行签名

具体代码如下:
```solidity
function isValidSignature(bytes calldata _data, bytes calldata _signature) public view override returns (bytes4) {
    // Caller should be a Safe
    GnosisSafe safe = GnosisSafe(payable(msg.sender));
    bytes32 messageHash = getMessageHashForSafe(safe, _data);
    if (_signature.length == 0) {
        require(safe.signedMessages(messageHash) != 0, "Hash not approved");
    } else {
        safe.checkSignatures(messageHash, _data, _signature);
    }
    return EIP1271_MAGIC_VALUE;
}
```
我们首先将`msg.sender`初始化为`safe`合约变量，这使我们可以使用`safe`合约内的变量和函数。

> 我们在`FallbackManager`模块中使用了`call`进行调用代理合约，所以此处`msg.sender`正是发起请求的`safe`合约地址。

然后，我们通过`getMessageHashForSafe`函数获得`_data`对于的签名字段。对于`getMessageHashForSafe`函数，我们会在下文进行详细介绍。

获得`_data`的签名字段后，我们进入分支判断，当满足以下任一条件后我们会返回`EIP1271_MAGIC_VALUE`，即`0x20c13b0b`:

1. `_signature`签名为空，但我们可以在`GnosisSafe`主合约内查询到该用户在之前对于此`messageHash`进行过授权
1. `_signature`不为空，且经过调用`GnosisSafe`主合约的`checkSignatures`函数发现签名正确

#### getMessageHash

此函数需要以下参数:

- message 需要验证签名的数据

此函数会返回用于签名的代签数据。由于此函数过于简单，我们在此处也给出其核心逻辑的实现函数`getMessageHashForSafe`，具体代码如下:
```solidity
function getMessageHashForSafe(GnosisSafe safe, bytes memory message) public view returns (bytes32) {
    bytes32 safeMessageHash = keccak256(abi.encode(SAFE_MSG_TYPEHASH, keccak256(message)));
    return keccak256(abi.encodePacked(bytes1(0x19), bytes1(0x01), safe.domainSeparator(), safeMessageHash));
}
```
在此处，我们首先将待签数据与`SAFE_MSG_TYPEHASH`拼接在一起，然后将以下数据进行拼接:
```
0x19 || 0x01 || domainSeparator || safeMessageHash
```
此种`hash`值获得的方式类似`EIP712`结构化哈希，具体可以参考[这篇博客]({{<ref "ecsda-sign-chain#%E7%AD%BE%E5%90%8D-1" >}})。

#### isValidSignature 2

此函数用于协调`GnosisSafe`合约对`EIP1271`的特殊改造版本与标准版本。简单来说，`GnosisSafe`规定`EIP1271_MAGIC_VALUE`，即验证签名成功后返回的签名为`0x20c13b0b`，这与`EIP1271`标准规定的`0x1626ba7e`是不符的。为了协调两者，使`GnosisSafe`也具有通用的合约签名能力，开发者设计了此函数。

其代码实现如下:
```solidity
function isValidSignature(bytes32 _dataHash, bytes calldata _signature) external view returns (bytes4) {
    ISignatureValidator validator = ISignatureValidator(msg.sender);
    bytes4 value = validator.isValidSignature(abi.encode(_dataHash), _signature);
    return (value == EIP1271_MAGIC_VALUE) ? UPDATED_MAGIC_VALUE : bytes4(0);
}
```
较为简单，核心在于使用接口实现将`msg.sender`(即`GnosisSafe`主合约)初始化为`ISignatureValidator`对象，使用`isValidSignature`函数判断去签名是否正确，并在最后使用三目表达式返回标准版本的`EIP1271_MAGIC_VALUE`。

> 此处的函数名`isValidSignature`与我们在此节介绍的第一个函数名是相同的，但两者所需要参数类型不同，所以`abi`编码后的结果也不同，读者不必担心混淆问题。

#### getModules

为兼容老版本函数名所设计的函数，在当前版本种可使用`getModulesPaginated`函数代替。其实现就是对`getModulesPaginated`的简单封装，不进行解释。

#### simulate

用于模拟代码在目标合约内运行，此函数需要以下参数:

- targetContract 需要运行代码的目标合约
- calldataPayload 需要目标合约运行的字节码`Bytecode`

此函数一个较为现代的实现为`src/common/StorageAccessible.sol`合约中的`simulateAndRevert`函数。此函数本质上就是对`simulateAndRevert`的包装。

> 在目前以太坊中，如果读者想测试合约运行字节码的情况，建议使用`eth_call`接口，或者使用`foundry`封装好的`cast call`命令。`eth_call`会使代码在合约内运行但不会消耗`gas`和改变区块链状态。

此函数的代码较为复杂，但所幸开发者为我们留下了大量注释。在函数体的开始，开发者使用了以下代码:
```solidity
targetContract;
calldataPayload;
```
根据注释，我们可以知道把两个变量直接放在函数体开始仅是为了避免编译器报错。核心代码位于`assembly`内。

在汇编代码块内，如往常一样，通过`mload`操作码在`0x40`地址内读取指向空闲内存的指针。接下来使用`mstore(internalCalldata, "\xb4\xfa\xba\x09")`向指针对于的内存中写入`0x64faba09`，即`simulateAndRevert(address,bytes)`(我们在上文提及的`StorageAccessible.sol`中的`simulateAndRevert`)函数的签名。

> 使用`\xb4\xfa\xba\x09`不足够直观，读者可使用`hex"64faba09"`代替，后者也可以直接将 16 进制字符转为`bytes`。

接下来我们构建一个用于请求`simulateAndRevert`完整的`calldata`。这意味着我们需要把以下`calldata`:
```
sig(simulate) || args
```
转换为:
```
sig(simulateAndRevert) || args
```
> 上述`calldata`中的`sig()`指获得函数签名的方法，即对函数名及参数进行`keccak256`哈希计算，可以使用`cast sig`进行直接调用，详情可参考[此篇博客]({{<ref "foundry-contract-upgrade-part2#%E5%90%88%E7%BA%A6%E6%B5%8B%E8%AF%95-1" >}})

> 上述`args`指用户输入的参数，由于`simulate` 和`simulateAndRevert`使用参数相同，所以不必继续修改。`||`表示拼接。

简单来说，我们只需要将`calldata`中的前 4 字节进行替换。我们使用`calldatacopy`对`calldata`进行复制。`calldatacopy`需要以下参数:

- destOffset 复制到内存中的起始位置
- offset 需要复制的数据在`calldata`中的起始位置
- size 复制的`calldata`长度

根据上文结论，我们只需要将`calldata`中的自第 4 bytes 开始的数据复制到内存中的`internalCalldata`的第 4 bytes 后，翻译成代码为:
```yul
calldatacopy(add(internalCalldata, 0x04), 0x04, sub(calldatasize(), 0x04))
```
此代码正好可以完成对`internalCalldata`的构建。

接下来，我们进行`call`操作，但在进行`call`操作前，我们使用了`pop`在内联汇编内返回数据。

> 我们在之前解释的含有内联汇编的函数均没有返回值，此函数作为含有返回值的函数较为特殊

在`pop`内部，使用了一个较为传统的`call`函数，因为我们在此前已多次解释过此函数，所以在此处不进行详细解释，具体可参考[文档](https://www.evm.codes/#f1)。一个很特殊的对方是此处直接选择将`call`的`success`返回值(即`call`返回值的前 32 bytes ，用来标志`call`请求是否成功)写入内存，而没有选择传统的先写入`return data`区域再复制的方法。原因在于`call`返回的`success`变量长度固定为 32 bytes ，直接写入内存是可行的。

> 不建议将`call`返回的其他数据写入内存，原因在于长度未知可能导致内存覆写

此处选择将`success`写入内存的`0x00`区域，此处属于`scratch space for hashing methods`，写入此区域具有安全性，且不用考虑`0x40`指针移动问题。

在处理完`success`部分后，我们需要考虑`call`返回值中的实际数据部分。在此处，我们先计算返回值长度`let responseSize := sub(returndatasize(), 0x20)`，只是在`returndatasize`基础上减去`success`的长度`0x20`。

接下来，我们避免内存覆写问题，我们需要手动调整`0x40`指向的区域，即原有指针增加`responseSize`。具体使用的代码如下:
```yul
response := mload(0x40)
mstore(0x40, add(response, responseSize))
```
调整完指针后，我们在将`returndata`中的非`success`部分进行复制，使用`returndatacopy(response, 0x20, responseSize)`。最终，我们判断`success`是否为`0`，如果为`0`，则证明调用失败，应中止调用。在上文中，我们把`success`变量存储到了`0x00`位置，此处只需要使用`mload`提取即可，代码如下:
```yul
if iszero(mload(0x00)) {
    revert(add(response, 0x20), mload(response))
}
```
当然，在`revert`时我们也返回一段内存信息。

### 接受代码分析

此部分主要介绍用于适配`safe`系列转账函数的代码，较为简单，大多只需要返回一个`Magic value`即可。此部分的代码位于`src/handler/DefaultCallbackHandler.sol`内、。

较为简单，不再进行解析。

## 总结

本文介绍了`GnosisSafe`模块中较为简单的最后一部分`fallback`。涉及以下内容:

1. EIP-165返回支持的接口ID
1. EIP-721、EIP-1155 的`safe`系列交易
