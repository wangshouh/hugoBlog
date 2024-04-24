---
title: "智能合约开发效率工具"
date: 2022-11-12T10:47:33Z
tags: [solidity]
aliases: ["/2022/11/12/smart-contract-tool/"]
---
## 概述

随着区块链的发展，智能合约的开发逐渐成为一片新的蓝海。与智能合约发展一同进步的其实还有一系列的智能合约开发工具和安全审计工具，但由于此方面很少有人介绍，导致大量新型工具并不为人所熟知。本文主要介绍智能合约开发和安全审计工具，以智能合约开发和测试为主线，依次介绍涉及以下方面的工具:

1. 编辑器配置
1. 代码编写、测试与部署
1. 合约安全测试
1. 链上合约调用分析
1. 区块数据获取

本文章主要面向使用`solidity`语言进行太坊或以太坊兼容链智能合约开发的程序员。

目前，已经有人对以太坊常用工具进行了汇总，读者可以参考 [Swiss Knife](https://swiss-knife.xyz/) 网站，如下:

![Swiss Knife](https://blogimage.4everland.store/SwissKnife.png)

## 编辑器配置

作为智能合约开发者，拥有一个配置良好的编辑器可以大幅度提高合约开发效率。由于目前智能合约开发刚刚起步，并没有使用专业的IDE，但通用编辑器`VSCode`配合各种插件可以拥有非常好的开发体验。在此处，我建议按照以下插件:

1. [solidity](https://marketplace.visualstudio.com/items?itemName=JuanBlanco.solidity)
1. [Solidity Visual Developer](https://marketplace.visualstudio.com/items?itemName=tintinweb.solidity-visual-auditor)

前者可以提供一些基础的语言特性高亮和自动补齐功能，而后者专门为智能合约审计人员设计，提供了大量的附加功能，方便开发者理解合约。对于一般开发者而言，基础插件的功能较好理解，此处我们着重介绍`Solidity Visual Developer`，该插件提供了大量的合约可视化能力，如下图所示:

![Solidity Graph](https://img.gejiba.com/images/68867ae24e707c9093bb1c61f0ed92c8.gif)

除此之外，此插件还提供了以下功能:

一是更加多样化的颜色绚烂、参数显示和函数信息显示，如下图所示:

![highlighting](https://img.gejiba.com/images/69cb00566ab2693a464eb57b6c2a9660.png)

二是引入了更加丰富的关键词信息和安全建议展示，如下图：
![code augmentation](https://img.gejiba.com/images/f6c45fe94cffbf0b72a9a2163d6e32e9.png)

三是增加了显示所有函数选择器的展示，如下图:
![Function Signature Hashes](https://img.gejiba.com/images/63512ff0614211e9f82b0726ca77e0e7.png)

四是增加了合约概览展示，如下图:
![Outline View](https://img.gejiba.com/images/4c167e4a1c301e620fadc3b514982158.png)

总而言之，`Solidity Visual Developer`提供了大量对于智能合约开发者实用的功能，无论开发者从事合约开发抑或是合约审计，此插件都可以大幅度增加开发体验。

## 代码编写、测试与部署

在代码编写、测试与部署环节，我个人推荐使用`foundry`框架，此框架具有以下特点:

1. 易于安装(仅针对Linux和MacOS系统)
1. 可以快速编译智能合约
1. 使用 solidity 作为测试框架语言，而不需要使用 javascript或 typescript 作为测试辅助语言
1. 提供了本地测试网络`anvil`和以太坊交互工具`cast`

对于代码编写和单元测试方面是较为复杂的，由于本文主要推荐开发工具而不是针对代码编写和测试，我们在此处不进行完整介绍。

如果读者对于此方面想更加深入的了解，可以选择阅读我之前写的[Foundry教程：编写测试部署ERC-20代币智能合约]({{<ref "foundry-with-erc20" >}})。

除了阅读我个人编写的文章，一个更好的途径是阅读[官方文档](https://book.getfoundry.sh/)

就目前的合约测试而言，一个基本的共识是使用`foundry`作为框架进行单元测试，而后使用`hardhat`进行集成测试，读者可根据自己开发的实际情况选择。

## 链上合约阅读

对于很多合约开发者而言，阅读并理解其他人已部署的链上合约是一个非常重要的学习途径，正常的阅读链上合约的方法是首先查询合约部署方是否已经把合约开源在了`github`上，对于大部分合约而言，我们都可以非常简单的找到其开源在`gtihub`上的版本。

对于部分合约而言，其可能已经经过了`etherscan`的`Contract Source Code Verified`但没有`github`的开源版本，对于这些合约，我们可以简单地使用`cast`工具获得其合约代码，命令如下:
```bash
cast etherscan-source -d weth 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2 --etherscan-api-key $etherscan
```
其中`$etherscan`需要为读者自行设置的`etherscan`的API密钥。

> 上述命令中的`0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2`是`WETH`的合约地址

另一种更加简单的方法是使用`etherscan.deth.net`提供的服务，我们仍以获取`WETH`的合约代码为例，当我们已知其合约地址为`0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2`的情况下，我们可以访问`https://etherscan.deth.net/address/0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2`，读者可以看到如下页面:

![Deth](https://img.gejiba.com/images/2be2f013327c4fdec30a5e6abbe7af5c.png)

这样就可以在在线`VSCode`内浏览代码，但不能进行修改等操作。

对于部分没有经过验证的智能合约，我本人不太建议分析此类合约，但如果读者有特殊需求可以通过使用[emvdis](https://github.com/Arachnid/evmdis)进行反汇编，在此处我们给出一个示例，假如上文使用的`WETH`合约未经过验证，我们首先通过以下命令获得合约字节码:
```bash
cast code 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2 --rpc-url https://rpc.ankr.com/eth > test.evm
```
此命令会下载`0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2`地址内的字节码并将其存储在`test.evm`中，使用任一编辑工具删除`test.evm`中最开始的`0x`标识并保存。然后调用此命令:
```bash
cat test.evm | evmdis > test.output
```
此命令会将反汇编结果输送到`test.output`中，如果您的系统内安装了`VSCode`可以使用`code test.output`在`vscode`内打开此文件，最终输出如下:

![EthDis Output](https://blogimage.4everland.store/ethdisOutput.png)

读者可以通过分析合约最开始的跳转表配合`cast 4byte`命令获得合约包含的函数，比如使用`cast 4byte 0x18160DDD`可以分析获得此合约包含`totalSupply()`函数。

对于部分未开源合约，我们可能需要阅读其字节码版本或者字节码的反汇编版本。对于字节码的阅读，我推荐使用 [bytegraph](https://bytegraph.xyz/) 工具，该工具会展示字节码之间的调用关系，更加方便读者阅读字节码。

![ByteGraph](https://blogimage.4everland.store/bytegraph.png)

对于反汇编字节码，我个人比较推荐 [dedaub](https://app.dedaub.com) 的反汇编工具，该工具是我目前使用过的最好的反汇编工具。但是需要注意，目前使用此服务需要进行登录。

![Dedaub Decompiled](https://blogimage.4everland.store/DedaubDecompiled.png)

## 链上合约调用分析

链上交易的分析是了解合约结构的一个重要方法，此处我们介绍以下三种可用于链上交易分析的网站:

1. [Samczsun Tx](https://tx.eth.samczsun.com/)
1. [Ethtx Info](https://ethtx.info)
1. [tenderly](https://dashboard.tenderly.co/explorer)
1. [Blocksec Phalcon](https://phalcon.blocksec.com/)

我们可以通过下述表格体现网站的不同:

| 网站名称 | 加载速度 | 包含功能 | 支持链 |
| -------- | -------- | -------- | ----- |
| [Samczsun Tx](https://tx.eth.samczsun.com/) | 快 | 交易基本信息、各参与方价值变化、交易行为、调用过程 | 以太坊及其主流L2、BSC |
| [Ethtx Info](https://ethtx.info) | 快 | 交易释放事件、各参与方价值变化、代币转账、调用流程 | 以太坊及Goerli测试网 |
| [tenderly](https://dashboard.tenderly.co/explorer) | 慢 | 交易基本信息、代币转账、调用流程 | 包含几乎所有EVM兼容链 |
| [Blocksec Phalcon](https://phalcon.blocksec.com/) | 中 | 交易基本信息、代币转账、调用流程、 **交易模拟** | 以太坊、BSC、Ploygon |

其中，`Samczsun Tx`和`Ethtx Info`均采用了较为简单的界面设计，提供的功能基本一致，而`tenderly`采用了较为复杂的UI设计，且提供了调用debug功能，值得注意的是此功能需要注册账户才可以使用。

在此处，我们给出一笔位于以太坊主网上的复杂清算交易，其哈希为`0x11d0050b5040438b8f95b4a6d07b31656242f30405e5e931d75b2cca19dfc94e`，读者可以在不同网站内查询此交易以进一步判断哪个网站更适合自己。当然，读者也可以直接使用下述链接直接跳转:

1. [Samczsun Tx](https://tx.eth.samczsun.com/ethereum/0x11d0050b5040438b8f95b4a6d07b31656242f30405e5e931d75b2cca19dfc94e)
1. [Ethtx Info](https://ethtx.info/mainnet/0x11d0050b5040438b8f95b4a6d07b31656242f30405e5e931d75b2cca19dfc94e/)
1. [tenderly](https://dashboard.tenderly.co/tx/mainnet/0x11d0050b5040438b8f95b4a6d07b31656242f30405e5e931d75b2cca19dfc94e)
1. [Blocksec Phalcon](https://phalcon.blocksec.com/tx/eth/0x11d0050b5040438b8f95b4a6d07b31656242f30405e5e931d75b2cca19dfc94e)

总结来说，如果读者查询的是一笔比较简单的交易且不希望通过合约代码深入理解具体调用流程，使用`Samczsun Tx`和`Ethtx Info`是一个不错的选择。如果读者希望获得更加详细的包含合约代码的深度分析，使用`tenderly`是必要的，如下图:

![Tenderly Debug](https://img.gejiba.com/images/380990901719a981372a24ddcc42a050.png)

如果读者希望模拟一笔交易在任意区块任意位置的运行，请选择`Blocksec Phalcon`，其提供了一个较为好用的交易模拟系统，如下图:

![Tx Simulator](https://img.gejiba.com/images/6265b2cf13885ef52a9d4d230b375519.png)

如果读者更倾向于使用终端继续相关测试，那么`cast run`命令可以满足大部分读者的需求，在此处给出一个简单示例，在终端内输入以下命令:
```bash
cast run \
    0xd15e0237413d7b824b784e1bbc3926e52f4726e5e5af30418803b8b327b4f8ca \
    --rpc-url https://rpc.ankr.com/eth --quick
```

> 此处的`quick`标识此交易直接被默认为块内的第一个交易执行，如果不增加此参数，则会将与此交易位于同一区块内此交易之前的所有交易重放执行一遍，较为耗时。

即会在本地测试环境内重放交易`0xd15e0237413d7b824b784e1bbc3926e52f4726e5e5af30418803b8b327b4f8ca`并输出对应的堆栈，如下:
```
Traces:
  [29962] 0xc02a…6cc2::transfer(0x40950267d12e979ad42974be5ac9a7e452f9505e, 105667789681831058)
    ├─ emit Transfer(param0: 0xc564ee9f21ed8a2d8e7e76c085740d5e4c5fafbe, param1: 0x40950267d12e979ad42974be5ac9a7e452f9505e, param2: 105667789681831058)
    └─ ← 0x0000000000000000000000000000000000000000000000000000000000000001


Transaction successfully executed.
Gas used: 51618
```

如果读者想深入研究交易流程内发生详细流程，可增加`--debug`开启`debug`模式，使用后会展示如下TUI界面:

![Debug TUI](https://img.gejiba.com/images/bffd1d4e166a08f75c03840a20d9b10d.png)

但阅读此界面需要读者拥有相当高的合约编程经验和对EVM的底层了解，在此处我们不再详细介绍。

## 区块数据获取

对于部分开发者而言，可能具有获取大量区块数据的需求，使用目前已有的以太坊RPC URL可以获得，在此处我们给出通过`RPC`获取区块数据的方法，命令如下:
```bash
cast block latest --rpc-url https://rpc.ankr.com/eth
```
在此处，我们省略输出。此命令事实上使用了`eth_getBlockByNumber`接口，具体定义读者可自行参考[文档](https://ethereum.github.io/execution-apis/api-documentation/)。

目前 RPC 服务商支持直接使用加密货币支付且价格最低的是 [dRPC](https://drpc.org?ref=174727) 提供的服务，该服务开源直接使用以太坊地址登录，且允许用户直接使用 Arbitrum 内的 USDT 等稳定币进行预充值。

## 总结

本文介绍了大量有助于智能合约开发者的相关工具，由于本文仅作为工具推荐文章，所有没有深度探讨其在智能合约开发复杂场景中的使用，我会在未来介绍智能合约开发的相关文章内穿插使用这些工具，以进一步增加其实用性。
