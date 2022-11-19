---
title: 深入解析Safe多签钱包智能合约:模块
date: 2022-09-10T16:47:33Z
tags: [solidity,Safe]
aliases: ["/2022/08/27/deep-in-safe-part-2"]
---

## 概述

在上一篇[博客](https://hugo.wongssh.cf/posts/deep-in-safe-part-1/)中，我们已经讨论了`safe`合约的代理部署和核心的`GnosisSafe`合约。在此博客内，我们主要讨论在上一篇文章内没有介绍的`safe`合约内各个模块的概念和代码。我们会按照各模块在`GnosisSafe`合约内出现的顺序进行解释。

## OwnerManager

在`GnosisSafe.sol`的`setUp`函数中，我们使用了此模块中的`setupOwners`函数。

此模块主要涉及签名者的管理等功能。

### setupOwners

此函数的功能为初始化签名者(`owner`)和需要签名的数量(`threshold`)变量。

在函数体的开始，我们看到一系列使用`require`的条件检查代码。代码中的注释已经较为详细的介绍了每个条件限制检查的目的，我们在此不再赘述。

为初始化`owners`映射，合约使用了`for`循环。在研究`for`循环前，我们首先给出`owners`的定义:
```solidity
mapping(address => address) internal owners;
```
与我们直观上认为`owners`应该为一个列表不同，`Gnosis`使用了映射来管理`owners`，实现了近似于单向列表的功能。使用映射的一大好处是映射底层的存储使用了哈希进行存储可以实现快速的寻址查找，大幅降低了时间复杂度。

> `mapping`底层的存储逻辑可以参考[此文](https://programtheblockchain.com/posts/2018/03/09/understanding-ethereum-smart-contract-storage/)

由于使用了链表作为数据结构，这也导致初始化等步骤较为繁琐。我们首先以一种较为直观的方式介绍如何进行初始化。假设`address[]`由`A`、`B`、`C`三部分构成。当我们进行初始化时，映射的变化如下:
```
第一次循环:
0x1 => A

第二次循环
0x1 => A
A => B

第三次循环
0x1 => A
A => B
B => C

跳出循环后
0x1 => A
A => B
B => C
C => 0x1
```

在代码中，我们使用`require(owner != address(0) && owner != SENTINEL_OWNERS && owner != address(this) && currentOwner != owner, "GS203");`要求`owner`符合以下条件:

1. 不为地址`0`和`0x1`。前者属于特殊地址，后者属于构建`owners`结构的核心地址
1. `owner`不能是合约自身
1. `owner`不能与`address[]`中的上一个元素相同，此处的`currentOwner`即上一个循环中的`owner`。

除此之外，我们又使用了`require(owners[owner] == address(0), "GS204");`确保此地址没有在`owners`中出现，避免重复。

在循环的最后使用`owners[currentOwner] = owner;`完成与上一节点的链接工作，也使用了`currentOwner = owner;`重置`currentOwner`为下一循环进行准备。

在循环结束后，我们使用`owners[currentOwner] = SENTINEL_OWNERS;`完成最后地址与`0x1`的链接。也初始化了一些其他变量。

### addOwnerWithThreshold

此函数用于增加在`owners`中增加`owner`，同时也修改`threshold`。

我们假设在上述链表后增加`D`地址。结果如下:
```
0x1 => D
D => A
A => B
B => C
C => 0x1
```
其中，`owners[owner] = owners[SENTINEL_OWNERS];`完成`D => A`的链接，而`owners[SENTINEL_OWNERS] = owner;`完成`0X1 => D`的链接。

剩余代码完成了对`ownerCount`和使用`changeThreshold`完成对`threshold`的修正。

此处使用的`changeThreshold`函数较为简单，代码如下:
```solidity
function changeThreshold(uint256 _threshold) public authorized {
    // Validate that threshold is smaller than number of owners.
    require(_threshold <= ownerCount, "GS201");
    // There has to be at least one Safe owner.
    require(_threshold >= 1, "GS202");
    threshold = _threshold;
    emit ChangedThreshold(threshold);
}
```
相信读者可以直接读懂其含义，我们不再进行解释。

我们可以看到在函数定义中加入了`authorized`修饰符，此修饰符定义位于`src/common/SelfAuthorized.sol`中，代码如下:
```solidity
contract SelfAuthorized {
    function requireSelfCall() private view {
        require(msg.sender == address(this), "GS031");
    }

    modifier authorized() {
        // This is a function call as it minimized the bytecode size
        requireSelfCall();
        _;
    }
}
```
显然，此修饰符的作用是保证此函数只能被自己调用。这是为了保证安全，因为增删修改`owners`也应该通过多签名实现，即我们应该通过`execTransaction`间接调用这些函数。所以函数的`msg.sender`应该为`address(this)`。

### removeOwner

顾名思义，用于删除`owner`，也可以使用`changeThreshold`修改`threshold`变量。

与增加成员不同，删除成员需要三个参数:

- prevOwner 链表中指向需删除元素的地址
- owner 需要删除的元素
- _threshold 新的`threshold`变量

假设我们要删除以下链表中的`D`，则需要将`prevOwner`设置为`0x1`，将`owner`设置为`D`。
```
0x1 => D
D => A
A => B
B => C
C => 0x1
```
上述链表修改后的结果:
```
0x1 => A
A => B
B => C
C => 0x1

D => 0
```
在具体代码实现中，我们首先使用`require(ownerCount - 1 >= _threshold, "GS201");`检查删除一位`owner`后数量是否能满足`_threshold`的要求。然后使用`require(owner != address(0) && owner != SENTINEL_OWNERS, "GS203");`进行常规的地址校验。最后使用`require(owners[prevOwner] == owner, "GS205");`确保`prevOwner`位于链表内。

在进行具体的链表修改时，我们使用`owners[prevOwner] = owners[owner];`修改`prevOwner`的指向使链表跳过`owner`，然后我们使用`owners[owner] = address(0);`将删除元素指向`address(0)`。

剩余的代码主要用于`threshold`变量，较为简单，我们不再赘述。

### swapOwner

此函数的功能是用于替换`owners`中的成员，相当于将`addOwnerWithThreshold`和`removeOwner`函数结合起来。与`removeOwner`函数所需要参数相同，此函数也需要3个参数，我们在此不再赘述参数功能。

我们依旧使用之前的链表:
```
0x1 => D
D => A
A => B
B => C
C => 0x1
```
我们将`D`替换为`E`，为了达成目的，我们需要将`prevOwner`设置为`0x1`。

替换结果如下:
```
0x1 => E
E => A
A => B
B => C
C => 0x1

D => 0
```
在具体代码实现上，我们首先使用`require(newOwner != address(0) && newOwner != SENTINEL_OWNERS && newOwner != address(this), "GS203");`判断`newOwner`是否符合以下条件:

1. 不等于`address(this)`，即合约自身地址
1. 不等于`SENTINEL_OWNERS`，即`0x1`

我们使用`require(oldOwner != address(0) && oldOwner != SENTINEL_OWNERS, "GS203");`，判断`oldOwner`也符合上述条件。

最后，我们通过`require(owners[prevOwner] == oldOwner, "GS205");`保证输入的`prevOwner`指向`oldOwner`。

修改链表的具体方法如下:
```solidity
owners[newOwner] = owners[oldOwner];
owners[prevOwner] = newOwner;
owners[oldOwner] = address(0);
```
首先我们使用`owners[newOwner] = owners[oldOwner];`将新节点的指向原节点的后节点，然后我们使用`owners[prevOwner] = newOwner;`将之前指向旧节点的节点指向新节点。最后我们使用`owners[oldOwner] = address(0);`将需要替换的节点指向0地址，相当于丢弃原节点。

### getOwners

此函数要与返回`owners`的列表，主要实现了链表到数组的改变。

我们首先声明输出`array`，代码如下:
```solidity
address[] memory array = new address[](ownerCount);
```
此处我们使用了定长的`address[]`，我们可以在`ownerCount`变量中提取长度。

接下来我们使用链表的前后映射的属性实现了链表向数组的转换，代码如下:
```solidity
address currentOwner = owners[SENTINEL_OWNERS];
while (currentOwner != SENTINEL_OWNERS) {
    array[index] = currentOwner;
    currentOwner = owners[currentOwner];
    index++;
}
```
仍以我们之前的链表为例:
```
0x1 => D
D => A
A => B
B => C
C => 0x1
```
在循环前，我们将`currentOwner`设置为`0x1`，循环跳出条件为`currentOwner != SENTINEL_OWNERS`，即循环到链表结束。在循环中，我们每次都会先将`currentOwner`放入`array`，然后使用映射取出当前元素的下一个元素，并重新设置`currentOwner`变量，在循环的最后，我们累加`index`。由于链表前后都使用映射链接，我们逐次取出下一个元素。

## FallbackManager

此模块主要用于管理`fallback`函数，使用了`EIP-1822 UUPS`可升级合约架构，如果读者对此标准不熟悉，可阅读[Foundry教程：使用多种方式编写可升级的智能合约(上)](https://hugo.wongssh.cf/posts/foundry-contract-upgrade-part1/#eip-1822-uups)。

本模块使用了代理合约架构，其实际逻辑合约位于[此地址](https://etherscan.io/address/0xf48f2b2d2a534e402487b3ee7c18c33aec0fe5e4)。如果读者观察此逻辑合约的代码会发现其与创建使用的`Singleton`有所不同，后者位于[此地址](https://etherscan.io/address/0xd9db270c1b5e3bd161e8c8503c55ceabee709552)。关于双方的差别，我们会在以后介绍。直观的代码区别可通过[此网页](https://etherscan.io/contractdiffchecker?a2=0xd9Db270c1B5E3Bd161E8c8503c55cEABeE709552&a1=0xf48f2b2d2a534e402487b3ee7c18c33aec0fe5e4)查看。

### internalSetFallbackHandler

我们可以看到此处使用了一个近似`UUPS`的架构，在定义变量中，我们可以看到使用了`FALLBACK_HANDLER_STORAGE_SLOT`作为特定存储槽。而`internalSetFallbackHandler`的功能正是修改此存储槽内的数据。函数主体使用了以太坊汇编中的`sstore`指令。

`sstore(key, value)`函数接受参数为存储数据的地址槽和数据。

### setFallbackHandler

用于设定合约的`fallback`变量，较为简单。但此处需要注意的是此函数使用了`authorized`修饰符，这意味着此函数不能直接调用，只能通过`execTransaction`进行调用。

### fallback

此函数是这一模块中最重要的函数，为主合约通过`fallback`的功能。此函数的主体使用了一个较为简单的代理合约。

我们首先`iszero`判断`handler`是否为全0地址，确定`handler`存在后，我们使用`calldatacopy(0, 0, calldatasize())`将`calldata`从EVM的`call data`区域读出。

为了方便调用的`fallback`合约识别请求者的地址，我们使用了类似`Meta-transactions`的处理方式，即在原来的`calldata`后拼接`caller`的地址。

> `Meta-transactions`的详细实现可以参考[此文](https://hugo.wongssh.cf/posts/eip712-extend/#meta-transactions)

在此处我们使用了`mstore(calldatasize(), shl(96, caller()))`增加`caller`地址。将`caller()`左移`96 bit`的原因是删去原变量中填充的`0`，或称`padding`。

> 此处涉及到在EVM中任何变量都占有32 bytes，而地址类型实际上只有 20 bytes 。为了保证地址可以填充满一个地址槽，使用了在地址前增加 96 bit `0` 的方法。

我们使用`call`进行合约调用，`call`所需要的参数为:

- gas Gas值
- address 需要调用的合约地址
- value 调用时转移的 ETH 数量
- argsOffset 调用所需要的参数在内存中的开始地址
- argsSize 调用所需要的参数的长度
- retOffset 返回值写入内存的地址
- retSize 返回值写入内存的长度

> 将`retOffset`和`retSize`均设为`0`代表不将返回值写入内存而保存在EVM的`return data`存储区域。

基于以上参数，我们使用了以下代码进行调用:
```solidity
let success := call(gas(), handler, 0, 0, add(calldatasize(), 20), 0, 0)
```
其中，较为复杂的是`retSize`参数，我们使用了`add(calldatasize(), 20)`代码，即原`calldata`长度与地址长度( 20 bytes )的和。

在最后，我们使用`returndatacopy(0, 0, returndatasize())`将`return data`从EVM的`return data`区域写回内存。

在最后，我们通过`iszero(success)`判断了调用是否成功，并进行了数据返回。

在此处，我们没有需要被调用的合约的具体内容，我们会在后文介绍这一部分。

## ModuleManager

此模块主要用于模块合约的管理，模块合约主要为主合约增加各种功能。读者可以在[Github](https://github.com/safe-global/safe-modules)上找到一些示例模块。但此仓库已经长时间没有进行过更新。

当我们引入`safe-modules`后，如果我们需要使用`modules`提供的功能，我们需要直接与`modules`合约进行交互。`modules`合约则调用`GnosisSafe`合约中的`execTransactionFromModule`等函数实现功能。我们会在后几篇博客内介绍`Gnosis`提供的一部分模块。

### setupModules

此函数主要用于初始化`modules`映射，与上文出现的`OwnerManager`中的`owners`相同，`modules`映射也是一个链表，所以在后文中的增删改查操作，我会减少介绍的篇幅。

我们首先使用`require(modules[SENTINEL_MODULES] == address(0), "GS100");`确保`modules`此前未被初始化。

然后，我们使用`modules[SENTINEL_MODULES] = SENTINEL_MODULES;`初始化链表为`0x1 => 0x1`形式，方便后文对链表进行增删改查。

最后，我们使用以下代码:
```solidity
if (to != address(0))
    // Setup has to complete successfully or transaction fails.
    require(execute(to, 0, data, Enum.Operation.DelegateCall, gasleft()), "GS000");
```
此函数通过`delegatecall`操作调用`to`合约内的代码，读者可以在`to`地址合约上编写一系列更加复杂的初始化函数，以实现更加复杂的初始化。当然，直接将`to`设置为`0`对于一般用户而言就足够了。

### enableModule

此函数的主要功能是在`modules`中增加相关模块。

如果读者没有在`modules`中增加对应的模块，那么模块就无法通过调用`execTransactionFromModule`等函数实现功能。

此函数的核心部分与`OwnerManager`中的`addOwnerWithThreshold`类似，都是在链表中增加地址。

### disableModule

此函数主要用于在`modules`中删除模块。此函数核心部分也与`OwnerManager`中的`removeOwner`类似类似。但此函数不需要`_threshold`参数。

### execTransactionFromModule

此函数主要由编写的模块合约进行调用以实现模块的核心功能。比如读者编写了一个可以解析其他签名格式的模块，在验证完多个用户的签名后，我们需要使用`execTransactionFromModule`将具体的交易委托`GnosisSafe`核心合约运行。

此函数的具体代码如下:
```solidity
function execTransactionFromModule(
    address to,
    uint256 value,
    bytes memory data,
    Enum.Operation operation
) public virtual returns (bool success) {
    // Only whitelisted modules are allowed.
    require(msg.sender != SENTINEL_MODULES && modules[msg.sender] != address(0), "GS104");
    // Execute transaction without further confirmations.
    success = execute(to, value, data, operation, gasleft());
    if (success) emit ExecutionFromModuleSuccess(msg.sender);
    else emit ExecutionFromModuleFailure(msg.sender);
}
```
此函数的参数如下:

- to 目标地址
- value 转移ETH的数量
- data 交易中包含的 `calldata`
- operation 交易类型，`DelegateCall`或`call`

在此函数的最开始，我们使用`require(msg.sender != SENTINEL_MODULES && modules[msg.sender] != address(0), "GS104");`校验合约是否已被授权。然后，我们使用`execute(to, value, data, operation, gasleft())`直接运行交易。

值得注意的是，此过程中没有校验签名的步骤，读者需要在模块中自行实现。这也说明模块的编写一定需要注意安全性，否则会严重影响`safe`合约资产安全。我不建议读者使用任何未经审计的模块。

> 我们会在后文为大家介绍一些常见模块的实现

### execTransactionFromModuleReturnData

此函数的主体部分建立在`execTransactionFromModule`基础上，但增加了返回值的功能，这一部分是使用汇编解决的，代码如下:
```solidity
assembly {
    // Load free memory location
    let ptr := mload(0x40)
    // We allocate memory for the return data by setting the free memory location to
    // current free memory location + data size + 32 bytes for data size value
    mstore(0x40, add(ptr, add(returndatasize(), 0x20)))
    // Store the size
    mstore(ptr, returndatasize())
    // Store the data
    returndatacopy(add(ptr, 0x20), 0, returndatasize())
    // Point the return data to the correct memory location
    returnData := ptr
}
```
此函数核心是从EVM的`return data`区域获取`returnData`，为实现这一目的，我们使用了[returndatacopy](https://www.evm.codes/#3e)函数，此函数需要以下参数:

- destOffset 复制后的`returnData`在内存中的位置
- offset `returnData`在内存中的起始位置，如果将此值设置为`0`即意味着在`EVM`的`returnData`中提取数据
- size `Returndata`的长度，可以通过[returnDataSize](https://www.evm.codes/#3d)获得

综合以上，我们首先使用`let ptr := mload(0x40)`获取到空闲内存的开始地址。

为了保证后续运行的代码不会占用这一区域，我们对`0x40`中对空闲内存开始地址进行更新，代码如下:
```solidity
mstore(0x40, add(ptr, add(returndatasize(), 0x20)))
```
此代码将`0x40`中代表的空闲内存地址指针更新为`prt + returnDataSize() + 0x20`，为我们后文进行的操作空出内存。

> 有读者可能会疑惑为什么我们之前使用汇编读取`0x40`进行内存操作时不更新地址，因为我们之前进行的各种汇编代码后没有其他代码，所以更新`0x40`是无意义的。但此处的代码运行后，还有其他依赖于`0x40`地址的代码运行，如果我们不更新`0x40`的指向地址会造成内存覆写的冲突。

我们接下来构造返回值变量，由于我们没有使用`solidity`所以我们需要手动操作内存构建。首先为了保持兼容性，我们假设`returnData`都为动态数据，所以我们首先构建头部长度字段(即前 32 bytes)，使用`mstore(ptr, returndatasize())` 代码。然后我们构建除头部字段以外的具体数据字段，代码为`returndatacopy(add(ptr, 0x20), 0, returndatasize())`。这些汇编代码都较为简单，读者可自行理解一下。

最后我们将`returnData`进行赋值`returnData := ptr`，实现全部过程。

### isModuleEnabled

此函数的功能是给定模块地址判断此模块在合约内是否启用。代码逻辑简单，不再介绍。

### getModulesPaginated

此函数的作用是返回当前合约内的模块地址，但增加了`pageSize`参数限定返回的模块数量。

该函数所需要的参数如下:

- start 在链表中进行检索时开始的地址，如果不确定可设置为`address(0x1)`
- pageSize 返回的模块地址数组的长度

在代码中，首先声明了`array = new address[](pageSize);`为定长的数组。然后代码通过一个与`OwnerManager`中的`getOwners()`类似的循环获得各模块地址。

但在代码的最后，使用了`mstore(array, moduleCount)`进行设置正确长度。正如前文一直提起的`array`等动态对象的前 32 bytes 存储着变量长度。我们使用`mstore`正好可以对其前 32 bytes 进行修改，修改为正确的`moduleCount`。这一步比较神奇，我不知道为什么需要重置长度。

> 这一步也生动体现了在`solidity`中所有变量都是指向内存的指针。

## GuardManager

此模块较为简单，在`execTransaction`中，我们使用了此模块用于审查交易是否符合规范。此模块包含一个接口和部分管理函数。基本原理是将交易数据通过`call`调用我们自行编写的符合接口的`guard`合约中去审查交易，具体代码如下:
```solidity
Guard(guard).checkTransaction(
    // Transaction info
    to,
    value,
    data,
    operation,
    safeTxGas,
    // Payment info
    baseGas,
    gasPrice,
    gasToken,
    refundReceiver,
    // Signature info
    signatures,
    msg.sender
);
```

一些具体的`guard`合约实例在`src/examples/guards`文件夹内，读者可以自行参考。

`setGuard(address guard)`函数用于设置`guard`合约地址。我们可以看到`guard`合约地址被存储在`0x4a204f620c8c5ccdca3fd54d003badd85ba500436a431f0cbda4f558c93c34c8`地址槽中，我们可以直接使用`sstore`向此地址槽内覆写`guard`合约地址。

注意此函数被`authorized`修饰，这意味着此函数只能通过`GnosisSafe`合约自调用的方式修改。

`getGuard()`函数用于返回`guard`合约地址，核心代码较为简单，不进行介绍。

## Executor

此合约只提供一个函数`execute`，此函数在`GnosisSafe`主合约内也比较重要。由于`Call`和`DelegateCall`的流程大致相似，我金辉介绍其中一个`DelegateCall`流程，代码如下:
```solidity
assembly {
    success := delegatecall(txGas, to, add(data, 0x20), mload(data), 0, 0)
}
```
此函数较为简单，`delegatecall`的汇编所需要参数在之前的文章内已经多次提过。如果读者不熟悉可以前往[此网站](https://www.evm.codes/#f4)查询。此处由于`data`属于`bytes`动态类型，所以我们需要将其前 32 bytes 的头部(头部内包含长度信息)进行截取，仅保留头部以外的数据部分。

## 其他模块

此处主要介绍属于`common`的模块，这些模块具有通用性，大量在各个模块内使用。各个模块的主要功能为:

- `Enum.sol` 包含一个简单的`Operation`变量，我们不会在后文介绍此合约
- `EtherPaymentFallback.sol` 包含一个`receive()`函数，主要用于接受ETH资产，我们也不会进行介绍
- `SecuredTokenTransfer.sol` 包含一个`transferToken`函数，用于转移ERC20代币，我们会详细介绍此函数
- `SelfAuthorized.sol` 包含一个`authorized`装饰器，我们在前文已多次用到，此处会简单介绍一下实现过程
- `SignatureDecoder.sol` 包含`signatureSplit`函数，用于分割多签名，下文会详细介绍
- `Singleton.sol` 只包含`singleton`变量，我们会在后文介绍它的神奇功能
- `StorageAccessible.sol` 此模块给了调用者获得合约内部`internal`变量的功能，下文会详细介绍

### SecuredTokenTransfer

此处没有使用`IERC20`此类的接口，而是徒手实现了相关逻辑。我们首先构建调用代币合约`transfer`函数所需要的`calldata`，使用了`bytes memory data = abi.encodeWithSelector(0xa9059cbb, receiver, amount);`。

> 当然，读者可以直接使用`abi.encodeWithSignature`函数，具体参考[文档](https://docs.soliditylang.org/en/latest/units-and-global-variables.html#abi-encoding-and-decoding-functions)

在完成`calldata`的构建后，我们使用了`call`进行调用，此处的`gas`参数选择了`sub(gas(), 10000)`，这是为了保证`call`结束后也可以为剩余代码的运行预留下`10000`单位`gas`。在`argsOffset`和`argsSize`等参数设置上，由于`data`属于动态类型，所以使用`add(data, 0x20)`跳过头部，也使用了`mload(data)`进行长度读取。

此处没有选择直接将数据推入`EVM`的`return data`暂存区，而是把数据推送到了内存中的`scratch space`区域，详情参考[文档](https://docs.soliditylang.org/en/latest/internals/layout_in_memory.html)。这是为了下一步进行判断调用是否成功准备的。在判断调用是否成功方面使用了以下代码:
```solidity
switch returndatasize()
    case 0 {
        transferred := success
    }
    case 0x20 {
        transferred := iszero(or(iszero(success), iszero(mload(0))))
    }
    default {
        transferred := 0
    }
```
此处没有直接使用`success`作为判断`call`是否成功的标志而是使用了结合了多种条件进行判断。

当`returnDataSize()`为`0`时，直接使用`success`作为判断交易是否成功，此种情况应对没有返回值的`transfer`函数; 

当`returnDataSize()`为`0x20`时，我们通过判断`suucess`或`returnDataSize`的长度判断交易是否成功，这种情况应对有返回值的`transfer`函数。

当以上两种情况都不满足时，则证明交易失败。

> 目前大部分合约都会实现ERC20接口，读者可以使用简单的接口进行实现。

### SelfAuthorized

我们已经多次使用过此合约内的`SelfAuthorized`修饰器，此修饰器的核心函数如下:
```solidity
function requireSelfCall() private view {
    require(msg.sender == address(this), "GS031");
}
```
这个函数的作用时检查`msg.sender`是否与当前合约地址相同。如果函数使用了此修饰器，只能通过设置`execTransaction`中的`Enum.Operation`为`Call`，使用`call`可以保证`msg.sender`为合约本地地址。

### Singleton

此合约仅包含一个变量声明`address private singleton;`，为什么我们需要这一个变量声明？

如果读者记得我们在[这一篇](https://hugo.wongssh.cf/posts/foundry-contract-upgrade-part1#%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90)中介绍的通过继承解决存储冲突问题的方案，此合约的作用与之类似，功能是为了保证`singleton`不会被其他变量覆写。

在`GnosisSafe`合约中，我们可以看到:
```solidity
contract GnosisSafe is
    EtherPaymentFallback,
    Singleton,
    ModuleManager,
    OwnerManager,
    SignatureDecoder,
    SecuredTokenTransfer,
    ISignatureValidatorConstants,
    FallbackManager,
    StorageAccessible,
    GuardManager
```
主合约在几乎最前面就继承了`Singleton`，保证`singleton`声明在存储区域中不会被覆写。

### StorageAccessible

在此模块中，我首先看到了`getStorageAt`函数，此函数的功能是获取指定存储槽内的数据。此函数中各参数的含义为:

- `offset` 需要读取的存储槽位置
- `length` 读取的长度

> 关于存储槽位置和布局问题，读者可以参考[Understanding Ethereum Smart Contract Storage](https://programtheblockchain.com/posts/2018/03/09/understanding-ethereum-smart-contract-storage/)

我们向读者演示此函数的用法，以上文提到的`Singleton`变量为例，此变量由于继承顺序问题被最先定义在`0`存储槽，使用以下命令就可以获得`Singleton`的值:
```bash
cast call 0xd1d1D4e36117aB794ec5d4c78cBD3a8904E691D0 "getStorageAt(uint256,uint256)(bytes)" 0 1 --rpc-url https://rpc.ankr.com/eth
```
我们可以看到返回值如下图:
![Call getStorageAt](https://img-blog.csdnimg.cn/img_convert/96d9e7e56b53bf0f549c4b385ec571fa.png)

接下来，我们介绍函数的具体代码如下:
```solidity
function getStorageAt(uint256 offset, uint256 length) public view returns (bytes memory) {
    bytes memory result = new bytes(length * 32);
    for (uint256 index = 0; index < length; index++) {
        // solhint-disable-next-line no-inline-assembly
        assembly {
            let word := sload(add(offset, index))
            mstore(add(add(result, 0x20), mul(index, 0x20)), word)
        }
    }
    return result;
}
```
我们首先使用`bytes memory result = new bytes(length * 32);`为返回值声明相关变量。此处直接声明了`bytes`的长度。

由于[sload](https://www.evm.codes/#54)每次仅能读取特定位置上的`32 bytes`，所以我们使用了循环进行多次读取以获得满足长度要求的返回结果。`let word := sload(add(offset, index))`，我们读取指定位置的存储内容。

读取结束后，我们需要将读取结果写入`result`变量，在写入过程中，我们首先应该保证数据写入到动态数据的非头部区域，所以我们使用了`add(result, 0x20)`进行计算。然后我们需要计算读取数据在非头部区域的具体位置，此处使用`mul(index, 0x20)`，即当前索引与`0x20`的乘积。最后综合上述两式，我们就可以得到具体的写入位置，最后通过`mstore(add(add(result, 0x20), mul(index, 0x20)), word)`完成写入。

`simulateAndRevert(address targetContract, bytes memory calldataPayload)`函数的功能是允许我们向`targetContract`目标合约发送`calldataPayload`的`delegatecall`请求进行相关测试，但没有任何副作用(即交易不会改变`targetContract`的合约状态)。为了提供尽可能多的信息，返回值被编码为`success:bool || response.length:uint256 || response:byte`，依次为调用是否成功、调用产生的返回值长度、真正的返回值。

具体实现类似我们在`src/proxies/GnosisSafeProxyFactory.sol`中介绍的`calculateCreateProxyWithNonceAddress`函数，即在代码运行完成后使用`revert`强行中断代码抛出错误。根据EVM的特性一旦发生错误，则不会对当前合约状态进行改变。

在代码实现上，我们首先使用`let success := delegatecall(gas(), targetContract, add(calldataPayload, 0x20), mload(calldataPayload), 0, 0)`进行`delegatecall`操作，此类型代码我们已经多次见过，所以此处不再详细介绍。

在获得返回值后，我们提提供以下代码实现返回值格式拼接:
```solidity
mstore(0x00, success)
mstore(0x20, returndatasize())
returndatacopy(0x40, 0, returndatasize())
```
向内存中依次写入相关数据。

最后我们使用`revert`抛出错误。`revert(offset,size)`函数会将以`offset`开始的长度为`size`内存作为错误信息抛出。

## 总结

到目前为止，我们已经分析了`Gnosis Safe`合约的所有部分。

我们会在下一篇文章内分析`GnosisSafe`合约中的`handler`部分，此部分用于处理`fallback`和`examples`中的`guards`。当然，围绕着`GnosisSafe`还有许多其他的由不同组织开发的模块，我们会选择其中一些酌情进行分析。

相信读者在读完这两篇文章后已经可以在源代码的角度使用或修改`GnosisSafe`合约。如果读者有任何问题，欢迎使用[我的博客](https://hugo.wongssh.cf/)公布的邮箱联系我。
