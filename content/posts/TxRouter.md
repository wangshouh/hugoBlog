---
title: "TxRouter使用指南"
date: 2023-04-10T23:47:33Z
tags: [huff]
---

## 概述

在 ETHBeijing 黑客松活动中，我花费三天时间构造了多方资产发送和聚合工具——[TxRouter](https://github.com/wangshouh/TxRouter) 工具。本文主要介绍该工具的具体功能、使用方法以及构造思路。

## 项目优势

TxRouter 是资产多方发送和聚合工具。更加详细的说，该工具解决了以下问题:

1. 单次将同一代币转移给多方的 ERC20 代币转账 gas 消耗问题
2. 从多个控制账户向同一账户进行资产聚合的问题

简单来说，该工具解决了一对多和多对一的资产 ERC20 代币资产转移问题。

我们最大的优势在于项目所有代码均使用了 Huff 语言实现，这使合约运行的 gas 效率极高。

目前与我们属于同一赛道的是商业化运营的闭源资产多方发送工具 [multisender](https://multisender.app/) 。我将其 `fork` 到本地测试网中进行了转账测试，如下:

![Multisender Gas](https://acjgpfqbqr.cloudimg.io/_img1_/a6b2b2135cc4ef5f61b7b6616fca683d.png)

而我们的项目测试如下:

![TxRouter Gas](https://acjgpfqbqr.cloudimg.io/_img1_/c26ed4bdc8e1828eefcbed5c72b72d6e.png)

与 `multisender` 相比，我们所需要的 gas 更低，且无需转入 ETH 作为手续费。而且，我们也提供了 Multisender 未提供的多对一资产聚合功能。

为了尽可能提高项目安全性，我们设计了 EIP1147 最小化代理的变体,该变体在保存最小化代理的基础上增加了 Owner 限制，即如果代理合约调用者不是部署时规定的 Owner 那么合约将直接 `revert` 拒绝服务。该功能减少了合约受攻击面，避免了代理合约内的资产被转移。

## 警告

为了尽可能节省 gas ，我没有进行防御性编程，这意味着用户需要自己审查 ERC20 代币合约的安全性。但一般来说，ERC20代币的代码不会有很大问题。

## 使用流程

由于目前 `TxRouter` 项目暂未进行主网部署，但读者可以在 `sepolia` 测试网中使用此工具。由于我前端能力较差，所以本项目暂时没有前端，在本文中，我将仅使用 `cast` 工具进行合约交互。

您可以在 `sepolia` 找到以下合约:

1. TxRouter 合约 `0x8fb08504bcec5fcdca87379dd568eb978a054c03`
2. TxRouterFactory 代理合约工厂 `0xd203d8e61d52c1723b4e95c4ed0b6645be94be0b`

> 此处的 `TxRouterFactory` 具有最小化代理合约的所有功能，仅增加了调用者限制功能，读者可以使用其 clone 其他合约

上述合约中，`TxRouter` 合约作为逻辑合约用于 `TxRouterFactory` 的代理操作，读者应先使用 `call` 命令获得生成的代理合约地址:

```bash
export TXROUTER=0x8fb08504bcec5fcdca87379dd568eb978a054c03
export FACTORY=0xd203d8e61d52c1723b4e95c4ed0b6645be94be0b
cast call $FACTORY "clone(address,address)(address)" $OWNER_ADDRESS $TXROUTER --rpc-url $RPC_URL
```

用户应自行替换 `OWNER_ADDRESS` 环境变量。上述命令将返回一个地址，这就是我们代理合约生成的地址。

使用以下命令真正执行交易:

```bash
cast send $FACTORY "clone(address,address)(address)" $OWNER_ADDRESS $TXROUTER --private-key $PRIVATE_KEY --rpc-url $RPC_URL
```

读者应执行设置 `$PRIVATE_KEY` 环境变量。一笔示例交易可以点此 [查询](https://sepolia.etherscan.io/tx/0xd52dac02d03b986d500003f3c5b6393ba15b61746f70c0817a9c6a41698ffc55)，我们也可以在 `Internal Txs` 中找到部署合约地址:

![Owner Address](https://acjgpfqbqr.cloudimg.io/_img1_/8907c11e0660d82e4a56bca756ade0b6.png)

将此代理合约地址存储至环境变量 `PROXY` 中，如下:

```bash
export PROXY=0x38a52ffe1a8140db4602b7d258ccaf684902e308
```

> 请用户自行替换 `0x38a52ffe1a8140db4602b7d258ccaf684902e308` 地址字段

在接下来的使用流程中，我们主要使用此代理合约。为了方便后期测试，用户可前往 [chaindrop](https://chaindrop.org/?chainid=11155111&token=0x6f14c02fc1f78322cfd7d707ab90f18bad3b54f5) 领取 USDC 测试代币。

> 注意，请勿将资产直接转移进入我部署的逻辑合约。逻辑合约对资产的转移没有检查，一旦转入极有可能造成资产损失。**用户仅应该与代理合约进行交互**。

TxRouter 合约具有以下函数及功能:

1. `multiTransfer(address, uint256[])`
	该函数用于一对多的资产转移，其中 `address` 表示需要转移的代币合约地址，`uint256[]` 是转移变量，每一个 uint256 元素的前 160 bit 为接受代币的地址而后 96 bit 为转移代币的数量。值得注意的是，此方法转移的是 TxRouter 合约内的资产
1. `multiApproveTransfer(address, address, uint256[])`
	该函数也用于一对多的资产转移，其中第一个 `address` 为授权人，第二个 `address` 为转移的代币合约地址，`uint256[]` 的构造方法与 `multiTransfer` 函数一致。
1. `multiAggregate` 
	该函数用于多对一的资产聚合，较为复杂，我们会在后文详细介绍

我们首先介绍最容易使用且 gas 消耗最少的 `multiTransfer` 方法，我们首先需要获得待转移 ERC20 代币地址。我们很容易在 etherscan 等网站中获得代币地址，如此处示例中使用的 USDC 代币，可以在 etherscan 中获得:

![USDC Address Etherscan](https://acjgpfqbqr.cloudimg.io/_img1_/bf97806e0c6ee4f4bac5effcbb5c766b.png)

将其地址保存进入环境变量:

```bash
export USDC=0x6f14C02Fc1F78322cFd7d707aB90f18baD3B54f5
```

我们需要将一部分资产转移进入代理合约 `PROXY` 的地址中，用户调用 transfer 等方法实现此功能，此处不再进行详细介绍。

接下来，我们需要进行代币转移配置 `uint256[]` 的构造，最简单的方法就是手动构造。用户可考虑使用 [简单脚本](https://github.com/wangshouh/TxRouter/blob/master/testDataGenerate/multi_transfer.py) 。如下:

![Multi transfer python](https://acjgpfqbqr.cloudimg.io/_img1_/9fc48731058b7c70de0c7ed61a22eec8.png)

获得 `uint256[]` 后，我们可以进行函数调用:

```bash
cast send $PROXY "multiTransfer(address,uint256[])" $USDC "[AFD48f565e1aC63f3e547227c9AD5243990f3D4000000002b5e3af16b1880000,Ff58d746A67C2E42bCC07d6B3F58406E8837E883000000015af1d78b58c40000]" --private-key $PRIVATE_KEY --rpc-url $RPC_URL
```

结果如下:

![MultiTransfer Result](https://acjgpfqbqr.cloudimg.io/_img1_/61f46f5d082c200cdb3ad5898360d507.png)

用户可以在 [此页面](https://sepolia.etherscan.io/tx/0x630d964234a2c2d936d7f65813dcad21c19b877e85459d8207fea54c2909335b) 找到该笔交易。即使我们仅进行了一对二的转账交易，消耗的 gas 仍比调用两次 `transfer` 更低。值得注意的是，随着资产转移方的数量增加，调用 `multiTransfer` 消耗的 gas 比直接调用 `transfer` 所节省的 gas 将线性增加，最终为用户实现极大的 gas 节省。

查询交易日志，我们发现所有的代币转移事件的 `from` 字段均为我们的代理合约，如下: 

![MultiTransfer](https://acjgpfqbqr.cloudimg.io/_img1_/a23e28a4c3c7eb24bcce8399f56fe69c.png)

有读者希望 `from` 为自己的地址而非代理合约地址，要想实现此目的，我们需要使用另一个函数 `multiApproveTransfer` 。值得注意的是，此函数消耗的 gas 较 `multiTransfer` 高，如果没有特殊需求，可以忽略此函数。使用此函数前需要使用自己的地址对代理合约进行授权，命令如下:

```bash
cast call $USDC "approve(address,uint256)(bool)" $PROXY 100ether --private-key $PRIVATE_KEY --rpc-url $RPC_URL
cast send $USDC "approve(address,uint256)(bool)" $PROXY 100ether --private-key $PRIVATE_KEY --rpc-url $RPC_URL
```

> 此处我们使用了先 `call` 后 `send` 的习惯，此习惯可以避免直接发送交易导致调用失败而浪费 gas

我们仍使用以上生成的代币转移配置，命令如下:

```bash
cast send $PROXY "multiApproveTransfer(address, address, uint256[])" 0xafd48f565e1ac63f3e547227c9ad5243990f3d40 $USDC "[AFD48f565e1aC63f3e547227c9AD5243990f3D4000000002b5e3af16b1880000,Ff58d746A67C2E42bCC07d6B3F58406E8837E883000000015af1d78b58c40000]" --private-key $PRIVATE_KEY --rpc-url $RPC_URL
```

其中 `0xafd48f565e1ac63f3e547227c9ad5243990f3d40` 需要替换为代币授权者。读者可以点 [此](https://sepolia.etherscan.io/tx/0x0da9a7e49ec4195bb01d4a374827fc2dfd0bd132f449b2932a2d50eea4ae6768) 查看示例交易。很明显，此处代币转移的 `from` 为代笔授权者，如下:

![Approve Multisend](https://acjgpfqbqr.cloudimg.io/_img1_/1b01c3bea527440635d17a755bfd8168.png)

最后，我们讨论最为复杂的多方资产聚合函数 `multiAggregate` ，该函数实现了多对一的资产聚合功能，其基础为 ERC20-Permit 机制，如果用户对此机制并不了解，请自行阅读 [ERC20-Permit](https://blog.wssh.trade/posts/eip712-extend/#erc20-permit) 一文。为尽可能节省 gas ，在设计中，我大量使用了 calldata 压缩技术且 ERC20-Permit 函数本身所需要的参数较多。这导致 `multiAggregate` 的调用是较为复杂的。

为方便读者阅读，我们首先给出 `Permit` 函数的定义:

```solidity
function permit(
    address owner,
    address spender,
    uint256 value,
    uint256 deadline,
    uint8 v,
    bytes32 r,
    bytes32 s
) public virtual {}
```

函数定义如下:

```solidity
    struct PermitCall {
        uint256 ownerValueV;
        bytes32 r;
        bytes32 s; 
    }

    function multiAggregate(address, address, uint256, PermitCall[] memory) external;
```

各参数定义如下:

1. 第一个 `address` 为代币合约地址
2. 第二个 `address` 为代币接收者
3. `uint256` 为 permit 函数中的 `address spender` 和 `uint256 deadline` 的组合，当然，为了实现拼接，我们将 `uint256 deadline` 改写为 `uint96` 类型，即最终实现 `uint160 spender || uint96 deadline` 的效果
4. `PermitCall[]` 是一系列可变参数的集合，核心为 `ownerValueV` ，其实质为 `uint160 owner || uint88 value || uint8 v`

正如读者所见，调用该函数是复杂的，我不建议新手使用此函数。在函数调用时，我们需要确保所有的 Permit 签名的 `deadline` 是一致的。了解该函数使用方法最简单的途径是直接阅读此函数的测试，请读者自行参考 [test_multiAggregate](https://github.com/wangshouh/TxRouter/blob/master/test/txRouter.t.sol#L132) 测试函数。

该函数可用于大规模资产归集，这在抢夺空投等场景下是极为有效的。

## 设计思路

在节省 gas 方面，我主要使用了以下途径:

第一， `calldata` 的压缩与聚合。

`calldata` 需要消耗不少 gas ，具体消耗如下:

> Calldata size: Each calldata byte costs gas, the larger the size of the transaction data, the higher the gas fees. Calldata costs 4 gas per byte equal to 0, and 16 gas for the others (64 before the hardfork Istanbul).
> 
> Calldata大小：每个calldata字节都要消耗gas，交易数据的大小越大，gas费用就越高。Calldata每字节花费4 gas，等于0时为零费用，而其他情况下为16 gas（在Istanbul硬分叉之前为64 gas）。

与 `transfer(address to, uint256 amount)` 的 `calldata` 参数相比，我们使用了 `uint160 to || uint96 amount` 的格式压缩参数，这有效减少了 `calldata` 的 gas 消耗。当然，也增加了解压缩的 gas 消耗，但解压缩消耗的 gas 是极低的，远远小于 calldata 的 gas 消耗。

除了 `calldata` 的压缩，另一个有效降低 gas 消耗的行为是聚合。众所周知，在合约内进行 `call` 操作是廉价的，相较于每一次发起交易的固定 21000 gas 成本，在合约内发起 `call` 调用的花费不包含此固定成本。

由于合约内 call 调用的 gas 计算是复杂的，本文不准备进行在此处展开，用户可自行参考 [EVM Codes](https://www.evm.codes/#f1?fork=merge) 给出的计算方法。

第二，`call` 调用过程中的参数复用。

为了进行可能实现 gas 的高效率，在设计过程中，我确定了静态内存分配的原则。以 `multiApproveTransfer` 为例，我提前规定了调用 `transferFrom` 所需要的参数在内存中的占用情况，复用了函数选择器`selector`  和 `from` 参数。这避免了多次内存写入的 gas 消耗。

第三，对内存使用的谨慎

在编写合约过程中，我尽可能避免对内存的使用而将所有参数保存在栈内，相较于栈操作，内存操作消耗的 gas 是较高的。在 TxRouter 的核心合约内，除了 `call` 调用参数由于必须在内存中保存而进行了 `mstore` 操作，其他所有操作都是在栈内完成的。

上述方法是在此项目中使用的，优点在于兼容 solidity 的各项规范，具有较好的互操作性，但事实上，solidity 的 abi 规范没有考虑到过多的 gas 节省问题，尤其是数组结构的编码。理论上，我们可以重新设计数组编码以实现更高的 gas 节省效率，但这意味着与当前以太坊智能合约开发体系完全不兼容。如果读者正在开发一些 MEV 项目等，可以考虑使用此方法。