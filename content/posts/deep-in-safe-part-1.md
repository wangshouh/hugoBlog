---
title: "深入解析Safe多签钱包智能合约:代理部署与核心合约"
date: 2022-08-27T16:47:33Z
tags: [Safe,solidity,EIP-712,EIP-1271]
aliases: ["/2022/08/27/deep-in-safe-part-1"]
---
## 概述

`Safe`(或称`Gnosis Safe`)是目前在以太坊中使用最为广泛的多签钱包。本文主要解析此钱包的逻辑设计和代码编写。

读者可以前往[Safe Contracts](https://github.com/safe-global/safe-contracts)获得源代码。

## 预备知识

### Safe优势

作为智能合约钱包，`Safe`支持多签名批准交易。这带来了以下优势:

1. 更高的安全性。将资产放置在多签钱包内可以有效避免因为个人单一私钥的泄露而导致的资产丢失。用户可以将多签设置为`2-of-3`形式，个人保存两个私钥并将第三个私钥作为备份。当遭受黑客攻击时，泄露1个私钥对资产安全性没有影响。

1. 更加高级的交易设置。相对于以太坊用户，智能合约具有可编程性，这意味着用户可以自行编辑一些交易逻辑，比如将多个交易聚合起来一起执行(`batched transactions`)。此部分由`Safe Library contracts`提供。

1. 更加灵活的访问管理。用户可以在钱包内加入具有特定功能的模块，比如限制单一用户每日最大可批准金额。这对于DAO是十分有用的。此部分由`Safe Modules` 提供。

上述仅仅是对`Safe`优势的简单介绍。如果读者想了解更多关于此方面的介绍，请参考[Gnosis Safe 官网](https://docs.safe.global)

### 以太坊账户

在以太坊网络中，具有地址的账户被分为以下两类:

- EOA(externally owned accounts) 我们平常使用的使用账户均属于这一类型。这一类型的账户具有公钥和私钥。

- Contract accounts 合约账户。我们创建的合约也均有对应的区块地址，但没有私钥可以用于签名等操作，这一类型的账户被称为合约账户。与EOA相比，合约账户内存在代码逻辑，可以进行编写一些复杂操作。

值得注意的是，在以太坊中，EOA与合约账户是被同等对待的。合约账户可以发送交易，也可以接受ETH。

### 多签钱包

多签钱包是指需要使用多个私钥进行签名完成交易的钱包。它们的形式一般被标记为`m-of-n`，即需要`n`个签名人中的`m`个签名人进行签名确认。在实际形式上，存在一些加密算法可以实现签名聚合等操作，比如`schnorr`、`BLS`等算法都可以实现原生上的多签。

但上述方法一般依赖于一些特定的密码学算法，构建基于这些算法的钱包具有一定的复杂性而且要求设计者具有较高的密码学造诣。而使用智能合约实现多签钱包较为简单，因为智能合约具有数据存储和处理功能，这大大降低了多签钱包智能合约的设计难度。

我们会在后文向读者介绍`Gnosis Safe`的多签钱包的构造逻辑和代码。

![Multisig Wallet](https://acjgpfqbqr.cloudimg.io/_img1_/e581928ae548dc61b05db4e3eb36ce3a.png)

### 中继商

在以太坊生态内，用户只能使用ETH作为Gas支付的货币。随着ERC20代币的日益繁荣，很多用户有了使用ERC20代币支付Gas的需求，在此需求刺激下，以太坊生态环境内出现了一种特殊的实体——中继商。它们运行用户向其支付ERC20代币，然后由中继商代替用户进行交互。

值得注意的是中继商进行上述操作需要合约支持，比较著名的有`EIP2771 MetaTranscation`标准，具体可以参考[EIP712的扩展使用]({{<ref "eip712-extend.md#meta-transactions">}})。当然，`Gnosis Safe`合约对于中继商进行交易进行了很好的支持，我们会在下文逐渐介绍。

## 代码准备

由于`Github`仓库也用于`Gnosis Safe`团队日常开发，在完成阶段性开发后进行审计，所以直接`clone`仓库会获得未经审计的代码。一种更好的方法是前往[Github Release](https://github.com/safe-global/safe-contracts/releases)下载源代码。

当我们下载并解压代码后，我们在项目目录中输入`forge init foundry-safe`，然后我们将下载的代码中的`contracts`文件夹中的合约文件转移到`foundry-safe`项目中的`src`目录中。

在后文中，我们将按照合约的生命周期逐渐分析源代码。

在此处，我们给出在`Etherscan`网站中的各个合约地址:

1. [Proxy Factory](https://etherscan.io/address/0xa6b71e26c5e0845f74c812102ca7114b6a896ab2) 
1. [GnosisSafeProxy](https://etherscan.io/address/0xd9Db270c1B5E3Bd161E8c8503c55cEABeE709552)
1. [Gnosis Safe: Relay service - Transactions](https://etherscan.io/address/0x4d953115678b15ce0b0396bcf95db68003f86fb5)

## 代理工厂合约

当我们获取代码后，我们先研究合约的部署过程。参见下图:

![Proxy Deploy](https://acjgpfqbqr.cloudimg.io/_img1_/4ab785c630b3af1c6909a13305d0ce91.png)

这一部分的代码主要参考`src/proxies/GnosisSafeProxyFactory.sol`合约。为了方便研究合约，我们也给出此合约在以太坊主网中的[地址](https://etherscan.io/address/0xa6b71e26c5e0845f74c812102ca7114b6a896ab2)。

此流程的主要目的是使用工厂函数`createProxy`创建逻辑合约的代理合约。使用代理合约的模式的目的是为了节省`gas fee`。

### 最简核心实现

我们首先分析最简单的`createProxy`函数。读者可以前往[此网页](https://etherscan.io/tx/0x33de87c67f48620c08f1c1e98e4dd7a87baf74d481bd806160a4cd164ef76d81)查看一个真实的`createProxy`交易。

`createProxy`函数代码如下:
```solidity
function createProxy(address singleton, bytes memory data) public returns (GnosisSafeProxy proxy) {
    proxy = new GnosisSafeProxy(singleton);
    if (data.length > 0)
        // solhint-disable-next-line no-inline-assembly
        assembly {
            if eq(call(gas(), proxy, 0, add(data, 0x20), mload(data), 0, 0), 0) {
                revert(0, 0)
            }
        }
    emit ProxyCreation(proxy, singleton);
}
```

通过[natspec](https://docs.soliditylang.org/en/latest/natspec-format.html)注释，我们可以得到各个参数的含义:

- singleton 为逻辑合约的地址，在以太坊主网上地址为`0xd9Db270c1B5E3Bd161E8c8503c55cEABeE709552`
- data 为调用逻辑合约(`GnosisSafe.sol`)的初始化calldata，我们会在后文介绍。

我们首先使用`proxy = new GnosisSafeProxy(singleton);`创造了代理合约。此流程背后其实调用了[create](https://www.evm.codes/#f0)函数。

此处较难理解的是以下代码:
```solidity
call(gas(), proxy, 0, add(data, 0x20), mload(data), 0, 0)
```
关于`call`的参数可以参考[此网页](https://www.evm.codes/#f1)。此函数的形式为`call(gas,addr,value,argsOffset,argsLength,retOffset,retLength)`，各参数含义如下:

- gas 进行`call`所需要的gas
- addr 目标合约地址
- value 进行`call`操作转移的ETH
- argsOffset 进行`call`操作发送的`calldata`在内存中的开始位置
- argsLength 进行`call`操作发送的`calldata`的长度
- retOffset 返回值写入内存的开始位置
- retLength 返回值的长度

在此处，我们使用`add(data, 0x20)`获得`calldata`在内存中的起始位置。其原理为在内存中存储的`data`属于`array`类型，此数据类型在第一个内存槽内存储有长度，其余地址槽内存储有真实的数据。我们通过`add(data, 0x20)`获得真实数据的起始位置，然后通过`mload(data)`获得`data`的前 32 byte 中存储的长度。

> 上述内容可以参考[Memory Management](https://docs.soliditylang.org/en/v0.8.15/assembly.html#memory-management)文档

完成上述操作，我们使用了`if`判断`call`的是否正确执行，`call`正确执行会返回`True`，在数值上等同于`1`。

有了以上知识，我们可以分析[此交易](https://etherscan.io/tx/0x33de87c67f48620c08f1c1e98e4dd7a87baf74d481bd806160a4cd164ef76d81)的`Input Data`，我们点击`Decode Input Data`以更加友好的方式分析变量，我们可以看到`singleton`变量为`0xd9Db270c1B5E3Bd161E8c8503c55cEABeE709552`，`data`为一个复杂用于合约初始化的`bytes`，由于此初始化涉及到`GnosisSafe`的核心实现，我们会在后文进行分析。

最后此代码释放`ProxyCreation`事件，此事件的第一个参数为代理合约地址，第二个参数为复制的逻辑合约地址

读者可能会感觉上述流程极其奇怪，一是没有使用`require`进行错误断言，二是在`call`流程中没有使用`solidity`抽象出的`call`函数。出现上述的原因在于此部分代码是5年前写的，使用了`solidity`的远古版本，因为一直可以正常运行，所以没有更新。

### 复杂核心实现

本小节介绍其他的合约部署实现。

我们首先研究`deployProxyWithNonce`函数，此函数的作用是使用`create2`部署合约，但不会调用代理合约初始化初始化函数(即没有进行上文给出的`call`流程)。

此函数的核心使用了[create2](https://www.evm.codes/#f5)函数，该函数所需要的参数如下:

- value 转移给代理合约的ETH
- offset 合约初始化代码在内存中的偏移量
- size 初始化代码的长度
- salt 用于计算部署合约地址的参数

结合以上参数，我们可以获得确定的合约地址，计算方法如下:
```
keccak256(
    0xff + sender_address + salt + keccak256(initialisation_code)
)[12:]
```

此函数源代码如下:
```solidity
function deployProxyWithNonce(
    address _singleton,
    bytes memory initializer,
    uint256 saltNonce
) internal returns (GnosisSafeProxy proxy) {
    // If the initializer changes the proxy address should change too. Hashing the initializer data is cheaper than just concatinating it
    bytes32 salt = keccak256(abi.encodePacked(keccak256(initializer), saltNonce));
    bytes memory deploymentData = abi.encodePacked(type(GnosisSafeProxy).creationCode, uint256(uint160(_singleton)));
    // solhint-disable-next-line no-inline-assembly
    assembly {
        proxy := create2(0x0, add(0x20, deploymentData), mload(deploymentData), salt)
    }
    require(address(proxy) != address(0), "Create2 call failed");
}
```

首先，我们应该构建出可以部署的字节码。我们可以通过`type(GnosisSafeProxy).creationCode`获得需要部署合约的创建字节码。

> 注意上述表述为创建代码而不是运行代码，具体请参考[此文章]({{<ref "deep-in-eip1167#%E5%88%9D%E5%A7%8B%E5%8C%96" >}}) 和 [Deconstructing a Solidity Part II: Creation vs. Runtime](https://blog.openzeppelin.com/deconstructing-a-solidity-contract-part-ii-creation-vs-runtime-6b9d60ecb44c/)

但我们发现一个问题，我们部署的合约包含一个构造器(`src/proxies/GnosisSafeProxy.sol`)，代码如下:
```solidity
constructor(address _singleton) {
    require(_singleton != address(0), "Invalid singleton address provided");
    singleton = _singleton;
}
```

上述代码说明我们在构造对应字节码的过程中需要填入对应的参数。深入研究创建代码(可以通过与`proxyCreationCode()`交互获得)，我们发现此代码总是在字节码的最后按照内存长度逐一读取参数。也就是说，我们需要在`creationCode`后增加EVM标准内存长度( 32 byte )的代理合约的地址。在此处，我们使用了`uint256(uint160(_singleton))`进行了转换，将合约地址转换为`uint256`数据类型，此数据类型恰好占用 32 byte 。

在获得创建代码和构造器参数后，我们使用了`abi.encodePacked`对参数进行合并，此过程的目的是生成符合EVM标准的字节码。此函数的作用是将各参数进行编码并非标准的合并，详细可以参考[文档](https://docs.soliditylang.org/en/latest/abi-spec.html#non-standard-packed-mode)。

此处我们使用 `0xd9Db270c1B5E3Bd161E8c8503c55cEABeE709552` 作为 `_singleton` 构造器参数，为读者展示了最后拼接获得的 `deploymentData` 的具体字节码，读者可前往[此网站](https://www.evm.codes/playground?unit=Wei&codeType=Bytecode&code='R34wns_ZprtmTe63w3qe68339j8Tt2xojsn33_Zp8sUww5Uxo0U929Urrruvmjvm14nca_t17f08c379ayyyzjh0401wwxo01828s382h22jho01qc4x229139x400l1rrt1w9s390pwuqzaj54jv02lm9083v1x2179055rrxabqlu39XfeRvu54m7fa6l486eyyy00u35Vxr_wuhoX5b36Z37Z36u845af43dZ3eujVxi5kdup3dXfea2g69i66k582212od142929k49653a491wWd6r332de1as68c5f3e07c5c823xc2777ib9552gk6f6c6343zix033496eW6mc69gok696e6Wc6_46f6eo6m4g7265kkoi726fW69g65gySd9db2ic1b5e3bdm1e8c8r3c55ceabeei9552'~YYYYYz000ySSSx60w80vk~~~~u6ztx405T0r50qw6Tpfd5bo20n156s0m16l19k73j81i70h52xg64_57ZuwYffXuf3W76V1415Ul0Ts1SzzRxwt2%01RSTUVWXYZ_ghijklmnopqrstuvwxyz~_)观察和运行代码。

获得关键的`deploymentData`参数后，我们可以非常简单的实现`create2`。此处基本与上一节给出的`call`类似，我们在此处不再赘述。对于最后结果使用了`require`进行断言。

在目前，我们不建议仍使用此方法进行`create2`。我个人更建议大家使用由`solidity`抽象的`create`语法。即下述语法:
```solidity
proxy = new GnosisSafeProxy{salt: salt}(_singleton)
```
显然，只是用`solidity`抽象语法更加简洁易懂。

上述函数仅仅作为合约构建过程中的中间函数，此函数是为了`createProxyWithNonce`使用的。此函数较为简单，相当于在`deployProxyWithNonce`基础上，增加了`call`流程实现初始化。具体的`call`流程与`createProxy`类似，我们在此处不再赘述。

`createProxyWithNonce`是目前使用最为广泛的创建代理合约的函数。读者可前往[此网页](https://etherscan.io/address/0xa6b71e26c5e0845f74c812102ca7114b6a896ab2#events)查看。

![More createProxyWithNonce](https://acjgpfqbqr.cloudimg.io/_img1_/ad20d43e5be9a1adf7abc42b2183f429.png)

`createProxyWithCallback`是在`createProxyWithNonce`基础上实现的一个极其不常见的函数。简单来说，此函数的作用是在创建完成代理合约后会向指定的合约地址进行`proxyCreated`请求。

其核心代码如下:
```solidity
if (address(callback) != address(0)) callback.proxyCreated(proxy, _singleton, initializer, saltNonce);
```

此处要求`callback`实现以下接口:
```solidity
interface IProxyCreationCallback {
    function proxyCreated(
        GnosisSafeProxy proxy,
        address _singleton,
        bytes calldata initializer,
        uint256 saltNonce
    ) external;
}
```

使用此函数创建代理合约的交易极为罕见。

### 辅助函数

本节主要介绍`GnosisSafeProxyFactory`中的三个辅助函数。这些函数也使用的比较少，不属于核心实现。

最为简单是以下两个辅助函数:

- proxyRuntimeCode() 获得`Runtime`代码
- proxyCreationCode() 获得`create`代码

如果读者无法理解两者的区别，请参考[此文章]({{<ref "deep-in-eip1167#%E5%88%9D%E5%A7%8B%E5%8C%96" >}})

还有一个极其鸡肋的函数:

- calculateCreateProxyWithNonceAddress 此函数用于计算待部署的代理合约的地址

此合约通过`revert`中断合约创建流程，并返回代理合约地址等信息。但使用此函数，需要提交一个`from`为`GnosisSafeProxyFactory`地址的交易，对于一般用户而言不是很好构建，而且上述计算过程也会消耗较多`gas`，我建议读者通过相关公式在链下进行计算。

## 代理合约

此部分是工厂合约部署出的合约，与工厂合约相比，代理合约较为简单。此节介绍的代码位于`src/proxies/GnosisSafeProxy.sol`。

此部分可以参考我之前写的[使用多种方式编写可升级的智能合约(上)]({{<ref "foundry-contract-upgrade-part1#%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90" >}})。

此处使用`let _singleton := and(sload(0), 0xffffffffffffffffffffffffffffffffffffffff)`获取逻辑合约地址。此流程使用`and`操作使地址满足EVM要求。

另一段复杂的代码如下:
```solidity
// 0xa619486e == keccak("masterCopy()"). The value is right padded to 32-bytes with 0s
if eq(calldataload(0), 0xa619486e00000000000000000000000000000000000000000000000000000000) {
    mstore(0, _singleton)
    return(0, 0x20)
}
```
此处代码其实没有非常大的用途，相当于对`IProxy`的实现。`IProxy`定义如下:
```solidity
interface IProxy {
    function masterCopy() external view returns (address);
}
```

此处实现了对代理合约调用`masterCopy()`指令会返回逻辑合约的地址。当判断出来调用了函数`masterCopy()`函数选择器时，我们使用`mstore`将地址释放到内存中的第一个内存槽，然后使用`return`返回前 32 byte ，即返回逻辑合约地址。

其余的代码主要实现了代理相关的逻辑，读者可自行参考我之前写的文章。

## 核心代码

我们在此节会进入核心代码`GnosisSafe.sol`。此代码串联了各个模块，结构具有一定的复杂性。

![Gnosis Safe Inherit](https://acjgpfqbqr.cloudimg.io/_img1_/3097ec492fc1eb1ef12d12f7d8fda2e8.png)

由于此处涉及到大量外部模块，我们在此处不会详细介绍模块的实现仅会提及模块的功能，具体实现会在后文提及。
### Setup

我们跳过了没有任何参数的构造器函数，直接讨论`setUp`函数。在此处设计`setUp`函数的原因在于此合约通过代理合约的形式部署，而代理合约部署时不会也无法调用构造器函数。此处给出的构造器函数几乎没有作用。

此处，我们也就使用之前的[交易](https://etherscan.io/tx/0x33de87c67f48620c08f1c1e98e4dd7a87baf74d481bd806160a4cd164ef76d81)作为示例。我们可以在`Input Data`获得输入的`data`。如果读者还记得我们上文的讨论，就会知道此`data`的作用正是初始化代理合约。

我们对此`data`使用`cast --calldata-decode`进行解析：
```bash
cast --calldata-decode "setup(address[],uint256,address,bytes,address,address,uint256,address)" 0xb63e800d000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000016000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000002000000000000000000000000d85bf7de2a15fb2cf44f5beec271f804a0e6c881000000000000000000000000ab6647ad2a897d814d4c111a36d9fba6ed8ec28a00000000000000000000000000000000000000000000000000000000000000140000000000000000000000000000000000000000000000000000000000000000
```

运行截图如下:
![Cast Calldata decode](https://acjgpfqbqr.cloudimg.io/_img1_/c6fd9f764ac6bd98160a90ae06682ade.png)

此交易进行了最简单的初始化，仅初始化了`_owners`和`_threshold`。这两个参数的含义如下:

- _owners 规定多签钱包的拥有者列表
- _threshold 规定单一交易需要签名的数量

如上述交易规定`[0xd85bf7de2a15fb2cf44f5beec271f804a0e6c881, 0xab6647ad2a897d814d4c111a36d9fba6ed8ec28a]`为多签钱包拥有者，且单一交易需要两人均签名同意。

如果读者设计的多签钱包仅用于安全的存储资金，那么仅需要初始化这两个参数。

接下来我们介绍其他参数的作用:

- `to` 用于初始化模块的地址
- `data` 用于初始化模块的`calldata`
- `fallbackHandler` 应对`fallback`情况的合约地址，可以设置为[此地址](https://etherscan.io/address/0xf48f2b2d2a534e402487b3ee7c18c33aec0fe5e4)
- `payment`和`paymentReceiver` 此参数是为中继器等机构设计的参数

我们接下来逐行介绍`setUp`函数代码及功能:

`setupOwners(_owners, _threshold);`设置钱包拥有者和单一交易所需要签名的数量。我们会在后文介绍`OwnerManager`时进行更加详细的介绍。

`if (fallbackHandler != address(0)) internalSetFallbackHandler(fallbackHandler);`设置`fallback`函数以处理特殊情况。我们会在介绍`FallbackManager`时进行介绍。

`setupModules(to, data);`进行模块初始化的操作，我们会在介绍`ModuleManager`时进行分析相关代码。

以下代码较难理解:
```solidity
if (payment > 0) {
    handlePayment(payment, 0, 1, paymentToken, paymentReceiver);
}
```

由于此处涉及到`handlePayment`函数，我们在此处一并给出代码:
```solidity
function handlePayment(
    uint256 gasUsed,
    uint256 baseGas,
    uint256 gasPrice,
    address gasToken,
    address payable refundReceiver
) private returns (uint256 payment) {
    // solhint-disable-next-line avoid-tx-origin
    address payable receiver = refundReceiver == address(0) ? payable(tx.origin) : refundReceiver;
    if (gasToken == address(0)) {
        // For ETH we will only adjust the gas price to not be higher than the actual used gas price
        payment = gasUsed.add(baseGas).mul(gasPrice < tx.gasprice ? gasPrice : tx.gasprice);
        require(receiver.send(payment), "GS011");
    } else {
        payment = gasUsed.add(baseGas).mul(gasPrice);
        require(transferToken(gasToken, receiver, payment), "GS012");
    }
}
```

简单阅读就可以发现此函数的作用为向`refundReceiver`返还`gas`费用。当然，我们可以选择使用任意的代币进行返还。

在了解`handlePayment`函数的作用后，我们就可以理解初始化过程的代码。此代码的作用是为中继商设置的，实现用户可以委托中继商进行合约初始化的功能。当用户使用`GnosisSafeProxyFactory`创建`GnosisSafe`合约后，用户可以首先向未初始化的合约内转入资产，然后由中继商代为初始化。中继商在初始化过程中，通过设置`paymentReceiver`等参数转移合约内的资产以覆盖自己的`gas`成本。为了方便各位理解，我们进行合约调用测试。

我们首先需要进行一些步骤以保证`foundry`可以成功编译和测试合约。首先删除`src/test`，此文件夹内包含`GnosisSafe`编写的辅助测试合约，这些合约对于我们进行测试是不需要的。然后修改`src/interfaces/ISignatureValidator.sol`中的`function isValidSignature(bytes memory _data, bytes memory _signature) public view virtual returns (bytes4);`修改为`function isValidSignature(bytes calldata _data, bytes calldata _signature) public view virtual returns (bytes4);`。如果使用原代码会出现接口与实现不对应的情况。

我们需要在`test/utils/MockERC20.sol`创建一个类似`ERC20`的合约。代码如下:
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;


contract MockERC20 {
    uint256 public transferAmount;
    address public receiver;
    
    function transfer(address refundReceiver, uint256 amount) public {
        transferAmount = amount;
        receiver = refundReceiver;
    }

    receive() external payable {}

}
```
此合约只需要实现`transfer`函数以保证`handlePayment`可以正常运行。接下来，我们需要编写测试合约，我们在此处仅给出`setUp`部分，完整的代码请参考[Github仓库](https://github.com/wangshouh/foundry-safe/blob/master/test/GnosisSafe.t.sol)

代码如下:
```solidity
function setUp() public {
    Token = new MockERC20();
    SingletonTest = new GnosisSafe();
    Safe = new GnosisSafeProxy(address(SingletonTest));
    
    address[] memory ownerAddress = new address[](2);
    uint256 threshold = 2;

    ownerAddress[0] = address(0xd85bF7de2a15FB2Cf44f5beEc271F804A0E6C881);
    ownerAddress[1] = address(0xaB6647aD2A897D814D4c111A36d9fba6ED8ec28A);

    IGnosis(address(Safe)).setup(            
        ownerAddress,
        threshold,
        address(0),
        "",
        address(0),
        address(Token),
        10000,
        payable(address(1))
    );
}
```
初始化过程中的参数来自[此交易](https://etherscan.io/tx/0x33de87c67f48620c08f1c1e98e4dd7a87baf74d481bd806160a4cd164ef76d81)，但在此处我们将`paymentToken`等参数进行了设置，最后我们编写以下函数测试`payment`是否成功:
```solidity
function testTransfer() public {
    assertEq(Token.receiver(), address(1));
    assertEq(Token.transferAmount(), 10000);
}
```

在测试中，我们发现`receiver`获得了转移的代币。

当然，对于一般的用户而言，此功能并不是非常重要，主要实现了使用ERC20代币支付初始化Gas功能，即用户使用`GnosisSafeProxyFactory`创建合约后向合约转移ERC20代币，与中继商协商价格后由中继商进行初始化，同时中继商使用`handlePayment`函数在合约内收回等价值的代币。

### execTransaction

根据`natspec`提供的信息，我们可以得到`execTransaction`所需要的参数:

- to 支付目标地址
- value 支付数量
- data 交易所携带的信息，即调用目标合约的`calldata`
- operation 交易的类型，包括`Call`和`DelegateCall`方式
- safeTxGas 设置交易的`gas`费用
- baseGas 与交易执行无关的`gas`费用，主要为中继商设置
- gasPrice 用于付款计算的gas价格，主要为中继商设置
- refundReceiver 提取资金的中继商地址
- signature 交易签名

在分析代码之前，我们首先给出一个在以太坊中的[真实交易](https://etherscan.io/tx/0x8c9dfce2b756a88203f63d6837a3ac98c559ea37ee006a1f66170ddb309df108)。此交易实现了提取多签钱包内的资金进行转账的功能，向目标用户转移了 8000 ETH。读者可以自行使用`cast --calldata-decode`自行解码`calldata`进行分析。

我们可以看到在函数体内大量使用了`{}`花括号进行分割代码，这是为了避免`Stack too deep`错误，具体可以参考[这篇文章](https://soliditydeveloper.com/stacktoodeep)。

我们首先分析第一代码块中的代码，如下:
```solidity
bytes32 txHash;
{
    bytes memory txHashData =
        encodeTransactionData(
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
            nonce
        );
    // Increase nonce and execute transaction.
    nonce++;
    txHash = keccak256(txHashData);
    checkSignatures(txHash, txHashData, signatures);
}
```
此代码的作用主要为验证`EIP712`签名。在此处使用了`encodeTransactionData`将交易数据编码为`EIP712`规定的结构化数据形式。我们会在后文对此函数进行介绍。然后，我们通过`keccak256`计算哈希，完成`EIP712`的结构化数据哈希流程。

读者可自行阅读我之前写的这两篇文章以理解上述流程:

- [基于链下链上双视角深入解析以太坊签名与验证]({{<ref "ecsda-sign-chain" >}})

- [EIP712的扩展使用]({{<ref "eip712-extend" >}})

在获得相关数据后，我们使用`checkSignatures(txHash, txHashData, signatures);`进行验证签名是否正确。此函数我们会在下文进行介绍。

接下来，我们分析第二代码块:
```solidity
address guard = getGuard();
{
    if (guard != address(0)) {
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
    }
}
```

此代码主要涉及`GuardManager`模块，在此处我们不详细分析其具体实现。它的功能是检测交易是否符合合约部署者所设置的其他条件.当然，这些条件需要用户自行编写合约并进行部署，然后调用`setGuard(address guard)`进行设置。

在后面介绍`GuardManager`模块时，我们会再次进行说明。

接下来介绍用于`gas`相关设置的代码块，代码如下:
```solidity
require(gasleft() >= ((safeTxGas * 64) / 63).max(safeTxGas + 2500) + 500, "GS010");
{
    uint256 gasUsed = gasleft();
    // If the gasPrice is 0 we assume that nearly all available gas can be used (it is always more than safeTxGas)
    // We only substract 2500 (compared to the 3000 before) to ensure that the amount passed is still higher than safeTxGas
    success = execute(to, value, data, operation, gasPrice == 0 ? (gasleft() - 2500) : safeTxGas);
    gasUsed = gasUsed.sub(gasleft());
    // If no safeTxGas and no gasPrice was set (e.g. both are 0), then the internal tx is required to be successful
    // This makes it possible to use `estimateGas` without issues, as it searches for the minimum gas where the tx doesn't revert
    require(success || safeTxGas != 0 || gasPrice != 0, "GS013");
    // We transfer the calculated tx costs to the tx.origin to avoid sending it to intermediate contracts that have made calls
    uint256 payment = 0;
    if (gasPrice > 0) {
        payment = handlePayment(gasUsed, baseGas, gasPrice, gasToken, refundReceiver);
    }
    if (success) emit ExecutionSuccess(txHash, payment);
    else emit ExecutionFailure(txHash, payment);
}
```

在进行具体交易代码前，我们可以看到合约使用`require(gasleft() >= ((safeTxGas * 64) / 63).max(safeTxGas + 2500) + 500, "GS010");`检查了`gasleft`的数值。

我们首先分析约束`gasleft`大于`(safeTxGas * 64) / 63`的原因，这一要求是基于[EIP150](https://eips.ethereum.org/EIPS/eip-150)。`EIP150`规定:
```
63 / 64 * gas available = Call/DelegateCall gas
```
即`call`操作传递的`gas`应为当前可用`gas`的`63 / 64`。在此处，`Call/DelegateCall gas`即我们为交易设置的`safeTxGas`，而`gas available`(可用gas)即合约的`gasleft`。我们可用简单的计算得到`gas left = (safeTxGas * 64) / 63`。当然，此时显示了`gasleft`的可在交易中传递`safeTxGas`的最小情况。如果`gasleft < (safeTxGas * 64) / 63`情况出现，我们就无法满足向交易传递`safeTxGas`的要求。

除了验证符合`EIP150`的条件，我们还需要保证`gas`费用足够合约抛出`events`。此部分的成本为`2500`。所以有了另一个限制条件`gasleft > safeTxGas + 2500`。

最后，我们还需要在满足上述两个限制的基础上保留另一部分`gas`保证代码运行，此部分数值为`500`。

对上述条件进行组合，我们可以得到最终条件。

在完成`gasleft`的校验后，我们进入了交易执行的核心模块。首先声明`gasUsed`变量。然后，我们进入了交易执行的核心代码，如下:
```solidity
success = execute(to, value, data, operation, gasPrice == 0 ? (gasleft() - 2500) : safeTxGas);
```
此处使用的`execute`位于`Executor`模块，主要用于进行交易，核心就是封装了`Call`和`DelegateCall`函数。我们会在后文为大家介绍此模块的实现。

在此处，我们给出`execute`的函数的定义，如下:
```solidity
function execute(
    address to,
    uint256 value,
    bytes memory data,
    Enum.Operation operation,
    uint256 txGas
) internal returns (bool success)
```
我们关注个参数的含义:

- to 目标地址
- value 交易转移的ETH数量
- data 交易包含的`calldata`
- operation 决定交易为`call`或`delegatecall`
- txGas 交易消耗的`gas`值

在此处，较为复杂的为`txGas`参数为`gasPrice == 0 ? (gasleft() - 2500) : safeTxGas)`。这显然是一个三目表达式，当设置`gasPrice`为0时，交易发送的`gas`为`gasleft() - 2500`。而如果设置了`gasPrice`，则交易发送`safeTxGas`数量的`gas`。进行如此设置的合理性在于，正如上文所述，`gasPrice`参数一般由交易中继商设置，中继商会设置`gasPrice`等变量，这些设置最终关乎中继商可在交易内获得的回报。所以当设置`gasPrice`时，严格限制发送的`gas`数量为`safeTxGas`是有必要的。当然，对于普通用户而言，我们只需要交易可以正常进行而不关注交易过程的具体`gas`消耗，所以如果用户没有设置`gasPrice`参数，则会对交易设置`gasleft() - 2500`的`gas`。此处预留`2500`是为了保证`events`可以顺利抛出。

> 值得注意的是，虽然我们对交易设置了较高的`gas`，但并不意味着相较于`safeTxGas`的消耗的`gas`多，原因在于，交易执行方会将多余的`gas`进行返还操作。

在进行交易执行后，我们通过`gasUsed = gasUsed.sub(gasleft());`计算上述交易步骤消耗的具体`gas`数量。着主要方便中继商在合约内提取手续费。

完成上述核心步骤后，我们接下来主要处理中继商提取手续费和抛出`events`的过程。

首先，我们可用看到在此处进行了一系列条件检测，如下:
```solidity
require(success || safeTxGas != 0 || gasPrice != 0, "GS013");
```
一旦不满足以下三个条件，合约会停止运行并抛出异常:

1. 交易未成功
1. 未设置`safeTxGas`
1. 未设置`gasPrice`

交易未成功抛出异常可能对于大家而言比较好理解，但为什么同时要求满足未设置`safeTxGas`和`gasPrice`的条件呢？因为此参数主要由中继商设置，我们知道即使交易失败也会消耗`gas`，所以我们需要在交易失败的条件下继续运行后面的中继商提取交易手续费的逻辑代码，避免中继商在失败交易中蒙受损失。

最后我们观察中继商提取手续费的代码块:
```solidity
uint256 payment = 0;
if (gasPrice > 0) {
    payment = handlePayment(gasUsed, baseGas, gasPrice, gasToken, refundReceiver);
}
```
当交易设置的`gasPrice`大于`0`时，我们通过`handlePayment`函数计算中继商手续费数量并将其返回给中继商。我们会在后文详细介绍`handlePayment`函数。

手续费提取的具体交易可以参考[这个交易](https://etherscan.io/tx/0x2cf86cdeb052d0accb71711edd0225c192765c9ac8b86e6ad1adc1d556b216ae)，如下图:

![Relay Tx](https://acjgpfqbqr.cloudimg.io/_img1_/ae44573b77a595cbad95fca045b3781d.png)

在交易的最后代码，我们进行了抛出事件和调用`Guard`合约中的`checkAfterTransaction`进行监控。

### handlePayment

在上文中，我们在`setup`和`execTransaction`中都使用了这一重要的参数。我们会在此处详细介绍此函数的参数和代码逻辑。由于此处使用到了`gas`相关的基础知识，特别是`EIP1559`相关的`gas`机制，建议读者先行阅读[此文]({{<ref "ethereum-gas" >}})

我们首先从参数分析，此函数需要以下参数:

- gasUsed 用于计算手续费的`gas`数值
- baseGas 类似`EIP1559`中的`Base Fee`，具体可以参考[此文]({{<ref "ethereum-gas#base-fee" >}})
- gasPrice `gas`的价格
- gasToken 用于支付`gas`的代币，即中继商提取手续费时提取的代币种类
- refundReceiver 提取手续费资金的获得者，一般为中继商自身钱包地址

在此函数体中的第一行代码通过三目表达式定义了`receiver`变量。当用户设置`refundReceiver`参数时，即采用用户的设定参数; 否则使用`tx.origin`作为提取手续费的接收者。

接下来我们进入一个分支结构，根据代币类型进行分支判断。我们首先分析`gasToken == address(0)`的情况，即选择`gasToken`为ETH的情况，在此处，我们使用以下公式计算中继商在合约内提取的手续费:
```
Gas Fee = (gasUsed + baseGas) * gasPrice
```
当然，此处的`gasPrice`也通过一个三目表达式进行选择，要求`gasPrice`为用户设置的`gasPrice`和交易内含的`tx.gasprice`中的较大值。当计算完成后，我们便将`Gas Fee`数量的ETH通过`send`发送给接收者。

> 用户可以在[solidity 官方文档](https://docs.soliditylang.org/en/latest/units-and-global-variables.html#block-and-transaction-properties)中查询到所有的`tx`结构体中的参数。

在其他代币分支，我们使用了类似公式进行计算。但由于以太坊交易`tx`的数据结构中不包含以其他代币计费的情况，所以在此处我们无法使用`tx.gasprice`。在此处，我们只能接受函数设置的`gasPrice`变量。然后，我们通过位于`src/common/SecuredTokenTransfer.sol`中`transferToken`进行代币转移。

通过此函数，用户可以通过中继商使用任何代币支付`gas`费用。

### checkSignatures

我们在`execTransaction`通过此函数检测交易多签是否符合签名者的数量限制。此函数需要以下参数:

- dataHash 交易参数的`EIP712`哈希值，具体可以参考[此文章]({{<ref "ecsda-sign-chain#%E7%AD%BE%E5%90%8D-1" >}})
- data 需要进行签名的数据
- signatures 需要进行检查的签名数据

此部分的核心代码为`checkNSignatures(dataHash, data, signatures, _threshold);`，对于此函数我们会在下文进行介绍。

### checkNSignatures

由于此函数所需要的参数与`checkSignatures`有大量重叠，所以在此处我们不再进行相关的参数介绍。

在函数的起始位置，合约首先检查了`signatures.length >= requiredSignatures.mul(65)`。这是为了保证`signatures`聚合签名的长度符合预期。

> 此处的常数为`65`的原因是在最小签名(即仅包含`v`、`r`、`s`)的长度为`65 bytes`。具体可以参考[此文章]({{<ref "ecsda-sign-chain#%E4%BB%A5%E5%A4%AA%E5%9D%8A%E4%BA%A4%E6%98%93%E7%AD%BE%E5%90%8D" >}})。

值得注意的是，`GnosisSafe`为了满足多种签名方式并存的情况，修改了部分签名的定义。读者可以阅读相关[文档](https://docs.safe.global/advanced/smart-account-signatures)进行学习。当然，我们也会在后文尽可能解释`Gnosis`的签名格式。

我们使用`for`循环和`signatureSplit`函数进行签名分割。`signatureSplit`被定义在`src/common/SignatureDecoder.sol`合约中，我们会在后文进行分析。

`GnosisSafe`支持多种签名方式，通过不同的`v`值进行判断，包括以下几种类型:

| v值 | 签名类型 |
| --- | ------- |
| 0 | 合约签名(EIP1271) |
| 1 | 预签名签名 |
| v > 30 | `eth_sign`签名 |
| 31 > v > 26 | ECSDA签名 |

在`Gnosis`的签名规定中，签名包含两部分，分别是静态部分和动态部分。所有的签名类型都具有静态部分，只有合约签名具有动态部分。顾名思义，静态部分的程度都是已知的`65 bytes`，且由`v r s`三部分构成; 而动态部分的长度不固定，作为合约签名的附属部分存在。在多个签名最后聚合时，我们必须保证静态部分在前而动态部分在后，同时保证静态部分按升序排列。

#### 合约签名(Contract Signature)

读者在阅读此部分时需要对`EIP1271`标准有一定理解，如果读者对此没有了解，请先阅读[此文章]({{<ref "ecsda-sign-chain#eip-1271" >}})。简单来说，合约将签名权授予某拥有私钥的用户，由此用户进行签名。接受合约签名的合约使用合约签名对`签名验证合约`调用`isValidSignature`函数，获得此签名是否是正确的合约签名。

我们首先给出合约签名的静态格式:
```
{ 32-bytes 签名验证合约 r }{ 32-bytes 签名数据位置 s }{ 0 v }
```

在这里比较神奇的一点是由于签名有`65 bytes`的长度限制，我们在此处无法完整将完整合约签名的编码，所以在此处我们设置了`签名数据位置`参数，即合约签名数据在组合后的多签名中的位置。

> 注意合约签名数据的位置必须位于常规签名(即包含`v r s`字段的签名)的后面，否则会影响函数读取签名。上述表达都较为抽象，我们十分建议读者阅读文档中的[示例](https://docs.safe.global/advanced/smart-account-signatures#examples)以更好地理解上述表述。

合约签名的动态部分，即签名数据部分格式如下:
```
{32-bytes signature length}{bytes signature data}
```

在此处我们注意到`signature data`没有具体长度，此签名的长度其实取决于`签名验证合约`中的`isValidSignature`的代码逻辑。

根据`EIP1271`的相关流程，我们需要首先获得`签名验证合约`的地址。根据上文给出的`合约签名的格式`，我们通过对`r`值的转换获得对应的地址，使用代码如下:
```solidity
currentOwner = address(uint160(uint256(r)));
```
由上文我们给出的“静态部分在前，动态部分在后”的规则，我们需要校验指向合约动态部分的`s`值是否在静态部分之外，使用`require(uint256(s) >= requiredSignatures.mul(65), "GS021");`代码进行判断。

> 此处我们使用了每个签名的静态部分长度固定为`65 bytes`进行判断

接下来，我们检查签名数据是否位于多签名内。通过动态部分的格式，我们知道`s + 32`即签名数据的起始位置，我们使用`require(uint256(s).add(32) <= signatures.length, "GS022");`检查签名数据的起始位置是否位于多签名内。

上文给出的条件检查并不能完全保证合约签名中的`s`指向的数据位于多签数据内，因为可能多签名由多于`requiredSignatures`的签名组成，这导致使用`require(uint256(s) >= requiredSignatures.mul(65), "GS021");`不能正确判断`s`指向的数据是否在多签数据内。

我们需要通过一些更加复杂的手段判断`s`指向的数据是否在多签数据内。如果需要更加精确的判断，我们需要获得多签数据的具体长度。具体来说，我们需要获取动态数据的长度。其核心函数为:
```solidity
contractSignatureLen := mload(add(add(signatures, s), 0x20))
```
要理解此代码，读者需要对于EVM底层数据存储有所了解。`signatures`属于`bytes`，本质上属于动态类型，变量相当于指向底层数据在内存的指针。而我们需要先通过`s`获得动态数据的起始位置。由于`signatures`只是指向内存的指针，我们在此指针后增加`s`就可以获得动态数据的起始内存地址。但需要注意`signatures`属于动态类型，根据`solidity`的规范，此数据的头部`32 bytes`为数据长度，所以我们需要在`signatures + s`的基础上增加`0x20`实现跳过`signatures`的长度数据的作用获取的真正的动态数据起始位置。

> 建议阅读[Layout of State Variables in Storage](https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html#mappings-and-dynamic-arrays)以进一步了解上述步骤。

当我们获得到动态数据在内存中的起始位置后，我们可以通过`mload(offset)`直接获取到动态数据的前 32 bytes ，即动态数据长度。

接下来我们需要具体计算判断`s`指向的数据是否在多签数据内，通过计算签名长度确定，代码如下:
```solidity
require(uint256(s).add(32).add(contractSignatureLen) <= signatures.length, "GS023");
```

在经过一系列检查后，我们终于进行提取签名数据的流程，代码如下:
```solidity
contractSignature := add(add(signatures, s), 0x20)
```
非常简单粗暴的将动态数据整体提取出来。为什么需要将长度和签名数据同时提取？正如上文所述，前32 bytes作为长度，后面数据作为数据是`solidity`中动态数据类型的基本形式。为了保证与`solidity`规定相符，我们在此处也使用了此种数据格式进行数据提取。

在获取到完整的签名数据后，我们只需要对签名验证合约(`r`)发送`isValidSignature`请求即可，代码如下:
```solidity
require(ISignatureValidator(currentOwner).isValidSignature(data, contractSignature) == EIP1271_MAGIC_VALUE, "GS024");
```
使用接口进行函数调用，较为简单。具体的接口实现由签名验证合约决定。如果验证正确，合约会返回`0x20c13b0b`已证明签名正确。

#### 预认证签名(Pre-Validated Signatures)

预认证签名的具体形式如下:
```
{32-bytes hash validator}{32-bytes ignored}{1}
```
从前之后依次由`r`、`s`、`v`变量表示。

此函数依赖于合约中的映射:
```solidity
mapping(address => mapping(bytes32 => uint256)) public approvedHashes;
```

此映射反映了某用户是否对特定的信息进行了预签名。如果进行了预签名，则`uint256`会被置为`1`。此过程通过`approveHash`实现，此函数代码如下:
```solidity
function approveHash(bytes32 hashToApprove) external {
    require(owners[msg.sender] != address(0), "GS030");
    approvedHashes[msg.sender][hashToApprove] = 1;
    emit ApproveHash(hashToApprove, msg.sender);
}
```
此函数较为简单，我们在后文不再进行介绍。

而对于此签名的检查是极其简单的，代码如下:
```solidity
currentOwner = address(uint160(uint256(r)));
require(msg.sender == currentOwner || approvedHashes[currentOwner][dataHash] != 0, "GS025");
```

符合条件的交易需要满足以下条件:

1. 发送者为签名中的`r`，此时发送的任何签名都被认可
1. 发送者对于`dataHash`已进行过授权

上述条件为`或`的关系，满足任一一点即证明签名有效。

#### Eth_sign签名

签名的具体形式如下:
```
{32-bytes r}{32-bytes s}{1-byte v}
```
这与传统的`ECSDA`签名基本一致，但此处为了使用`v`实现签名类型区分的作用，所以相比于正常的`v`(即27或28)，此处的`v`值进行了`+ 4`操作，即取值可以为`31`或`32`。

由于此处使用了传统签名方法，所以检验签名的方式极其简单，使用`ecrecover`预编译函数，代码如下:
```solidity
currentOwner = ecrecover(keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", dataHash)), v - 4, r, s);
```

> 此处需要补充的一点是`eth_sign`接口会在签名信息前增加`\x19Ethereum Signed Message:\n32`，这是为了防止签名被滥用，具体请参考[Github](https://github.com/ethereum/go-ethereum/pull/2940)

#### ECSDA签名

与其他签名验证相比，此签名验证较为简单，我们在此处不进行解释。

上述内容主要介绍了各种类的签名，我们处理验证签名外，我们还需要验证签名人是否符合要求。在介绍具体的判断代码前，我们需要简单了解一下用于存储签名人的映射，变量名为`owners`。在此处。我们不加解释给出`owners`中的映射情况(下图假设`a`、`b`、`c`均为设置的签名人):
```
0x1 => a
a => b
c => 0x1
```
关于此映射情况的来源，我们会在介绍`setupOwners`函数中进行解释。

所以在验证签名人身份时，我们可以使用`owners[currentOwner] != address(0) && currentOwner != SENTINEL_OWNERS`进行判断。

而`currentOwner > lastOwner`是为了保证签名人的地址按升序排列。

综合以上，我们使用下列代码进行判断:
```solidity
require(currentOwner > lastOwner && owners[currentOwner] != address(0) && currentOwner != SENTINEL_OWNERS, "GS026");
```


### requiredTxGas

此函数用于估计交易的`gas`耗费，此函数使用的参数较为简单且多次使用过，所以我们不再具体介绍参数含义。

函数代码如下:
```solidity
uint256 startGas = gasleft();
// We don't provide an error message here, as we use it to return the estimate
require(execute(to, value, data, operation, gasleft()));
uint256 requiredGas = startGas - gasleft();
// Convert response to string and return via error message
revert(string(abi.encodePacked(requiredGas)));
```
代码较为简单，值得注意的是为了避免我们在估计`gas`消耗时完成不必要的交易，我们在代码的最后使用`revert`函数直接抛出异常，达到交易中止的目的。当然，我们在`revert`返回的错误信息中编码了`requiredGas`，使用户可以通过错误信息获得交易的估计`gas`。

### encodeTransactionData

此函数用于编码交易数据，此函数使用的参数我们在前文都进行过相关介绍。对于此函数，我们不会进行详细介绍，读者可以参考我之前的博客[基于链下链上双视角深入解析以太坊签名与验证]({{<ref "ecsda-sign-chain#eip712" >}})

### getTransactionHash

此函数主要依赖于`encodeTransactionData`函数，仅进行了`keccak256`哈希操作。值得注意的是，此函数的返回结果就是签名者用于签名的信息。

## 总结

我们完成了`GnosisSafe`的代理相关合约和最为复杂的主合约的分析，相信读者也可以理解`GnosisSafe`的基本运作模式。

在`Proxy`相关合约中，`src/proxies/GnosisSafeProxy.sol`提供具体的代理合约代码，而`src/proxies/GnosisSafeProxy.sol`提供多种函数供用户进行代理合约部署。

在`src/GnosisSafe.sol`合约中，核心函数为`execTransaction`，其他函数基本都为此服务。读者可以以此函数为主线进行研究。当然，由于`GnosisSafe`的野望，合约内存在大量为中继商设计的函数，这一部分由于很难看到交易实例，所以我个人给出的内容可以存在于现实不符的情况。当然此部分对于核心实现没有很大影响。

如果读者谋求较为简单的实现，可以前往[此仓库](https://github.com/m1guelpf/lil-web3/blob/main/src/LilGnosis.sol)。
