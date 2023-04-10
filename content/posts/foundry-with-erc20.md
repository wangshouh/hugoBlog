---
title: Foundry教程：编写测试部署ERC-20代币智能合约
date: 2022-07-14T21:37:27Z
tags: [foundry,solidity,ERC-20]
aliases: ["/2022/07/14/foundry-with-erc20"]
---

## 概述

本博客的内容主要分为以下四部分：

一是Foundry的介绍与安装，主要介绍为什么选择Foundry进行智能合约开发和安装过程中的各种官方文档中未提及的问题；

二是智能合约的编写，主要介绍如何使用Foundry初始化开发环境，导入其他Solidity模块；

三是智能合约的测试，介绍Foundry中测试工具，以及如何使用Solidity编写测试脚本，以及输出Gas报告等内容；

四是智能合约的部署，介绍如何使用`Anvil`构建本地测试环境并进行合约测试，并介绍如何将合约部署至测试网络。

本文介绍的内容都会较为初级，如果您是高级开发人员，建议您直接阅读[文档](https://book.getfoundry.sh/) 。本文所有代码均位于 [github 仓库](https://github.com/wangshouh/LearnFoundry) 中，读者可自行参考。

## Foundry的介绍与安装

### 介绍

在智能合约编写领域，较为著名的智能合约编译和测试工作流为[hardhat](https://hardhat.org/)，hardhat使用npm包进行管理，使用`JavaScript`作为测试工作流。但其速度受限于JavaScript的性能，总体而言较为缓慢，且需要在开发流程中变换使用JavaScript与Solidity两种编程语言。[Foundry](https://book.getfoundry.sh/)改变了这一工作流。

首先，Foundry使用Rust编写，其编译Solidity智能合约的速度更快，同时如果您使用Linux系统，foundry的安装也会非常简单。

![性能对比](https://raw.githubusercontent.com/foundry-rs/foundry/master/.github/compilation-benchmark.png)

其次，在开发流程中，Foundry仅使用`solidity`一种编程语言，智能合约工程师可以仅使用`solidity`完成智能合约编写、测试和部署。而且，Foundry提供了一套完整的开发工具箱，主要包括以下三部分:

- forge，用于合约代码的编译和测试；
- cast，用于与智能合约进行交互，包括处在各类网络中的智能合约
- anvil，用于在本地构建区块链网络

最后，Foundry在对导入包的管理时通过`git submodule`进行管理，可以随时同步更新。个人认为较npm的管理方式更加的优雅和可控。

如果您想更加全面的了解Foundry，我个人推荐您去阅读一下Foundry仓库的[README.md](https://github.com/foundry-rs/foundry)以及它的[文档](https://book.getfoundry.sh/)

### Foundry的安装

Foundry开发流程中需要`Git`工具，由于此内容较为简单，读者可自行查阅安装方法。

对于Windows用户而言，我个人不建议直接使用官方文档给出的从头编译Foundry的方法，该方法需要你安装一套完整的Rust开发环境，而且编译过程中会出现大量的无关的编译产物(大概有600MB)，且编译时间并不短，在10代酷睿i7的CPU和16G内存下，编译时间长达数分钟。

为了降低安装难度，我更加推荐直接使用官方文档提供的一键脚本，直接安装官方提供的编译产物。但此方法只适用于Linux系统。

如果你使用的Mac系统，可以跳过下面对Linux系统的讨论，直接运行后文给出的终端命令。

如果你使用的系统就是Linux，请注意是否使用了最新的版本，如`Debian11`或`Ubuntu 22.04`等版本。如果您使用`Debian10`此类低版本系统，会出现因缺少关键运行库而报错以致程序无法运行。经过询问开发人员，得知官方是在`Ubuntu 20.04`系统下进行的项目编译，依赖部分较高版本运行库，如果使用`Debian10`等低版本系统会出现错误。

> 开发者认为兼容低版本Liunx系统会显著提高编译环境的复杂性，在短期内，官方不会兼容较低版本的Liunx。

如果你使用Windows，我个人推荐安装较高版本Linux的虚拟环境后再安装Foundry。如果您使用Windows10及以上的版本，您可以使用WSL虚拟环境。经过测试，`Ubuntu 22.04 LTS`的WSL版本是符合Foundry一键脚本运行条件的，你可以在[这里](https://apps.microsoft.com/store/detail/ubuntu-2204-lts/9PN20MSR04DW?hl=zh-cn&gl=CN)找到它的安装包并安装在你的Windows中。

如果你配置好了符合条件的Linux系统，可以直接使用下面给出的命令一键安装Foundry:

```bash
curl -L https://foundry.paradigm.xyz | bash
```

运行完上述命令后，在运行下列命令:

```bash
foundryup
```

最后可以通过一下命令检验是否安装成功:

```bash
forge -h
```

![forgehelp](https://img.wang.232232.xyz/img/2022/07/15/forgehelp.png)

如果想了解更多关于安装的信息，可以自行阅读官方给出的[文档](https://book.getfoundry.sh/getting-started/installation)

> 如果后续需要更新`foundry`，可以在此运行`foundryup`命令，运行后会自动更新当前`foundry`

## 智能合约的编写

### 初始化开发环境

在初始化开发环境前，请确认你有以太坊钱包。由于下文存在导出以太坊账户私钥的敏感操作，所以这里建议你重新创建一个用于代码开发的以太坊账户。

以太坊提供了[测试网络](https://ethereum.org/en/developers/docs/networks/#ethereum-testnets)供开发者使用。

在下文中，我主要使用MetaMask作为钱包，同时主要使用`Goerli TestNet`。当然，你的账户中需要一些测试用ETH，可以前往这个[水龙头](https://goerlifaucet.com/)获取。注意，此水龙头要求您注册`Alchemy`账号。

> 由于此文编写时`Ropsten TestNet`仍未废弃，所以后文采用了此测试网络。

本节内容主要参考了官方教程的[First Steps with Foundry](https://book.getfoundry.sh/getting-started/first-steps)

简单的来说就是使用以下命令初始化开发环境:

```bash
forge init ERC20Test
```

其中，`ERC20Test`可以更改为你想命名的项目名字。

接下来，我们需要安装一些开发库以更加方便地编写代码逻辑，此处我们将引入`solmate`和`Openzeppelin`两个开发库。前者是经过优化的且简单易读的智能合约开发库，但就仅实现了部分ERC功能；后者未经过优化，但包含的内容较多。在此次开发过程中，我们主要使用`solmate`。

> `solmate`的项目地址在[这里](https://github.com/Rari-Capital/solmate); `Openzeppelin`项目地址在[这里](https://github.com/OpenZeppelin/openzeppelin-contracts)

我们可以使用`forge`工具非常简单的导入这两个库，使用的命令如下:

```bash
forge install Rari-Capital/solmate Openzeppelin/openzeppelin-contracts 
```

安装完成后的目录如下:

![InstallTree](https://book.getfoundry.sh/images/nft-tutorial/nft-tutorial-project-structure.png)

我个人推荐使用VSCode作为Solidity的编辑器，一般来说，只需要进行下述两步操作:

1. 安装Solidity扩展[插件](https://marketplace.visualstudio.com/items?itemName=JuanBlanco.solidity)
2. 在项目目录中输入以下命令`forge remappings > remappings.txt`，该命令将生成映射文件避免报错

如果你想获得更多信息，可以参考官方文档给出的[Integrating with VSCode](https://book.getfoundry.sh/config/vscode)中的内容

### 智能合约的编写

本节主要介绍Solidity智能合约的编写，本节内容面向具有一定编程经验的开发者。如果你读者对本节的内容仍无法理解，可以先行阅读以下材料：

- solidity的[官方文档](https://docs.soliditylang.org/en/v0.8.15/)
- [Solidity by Example](https://solidity-by-example.org/)
- Solidity极简入门，或称[WTF Solidity](https://github.com/AmazingAng/WTFSolidity)

我们首先进行下述重命名:

- `src/Counter.sol` => `src/token.sol`
- `script/Counter.s.sol` => `script/token.s.sol`
- `test/Counter.t.sol` => `test/token.t.sol`

> 对于Foundry来说，`.s.sol`和`.t.sol`均为功能性代码的后缀，这两个后缀名虽然使用Solidity作为开发语言但作用不同于智能合约，主要起辅助作用

本次编写的智能合约与[ERC20](https://eips.ethereum.org/EIPS/eip-20)有关。简单来说ERC20允许我们在以太坊中进行发币。本文介绍的智能合约将不仅仅涉及简单的发币功能，还将增加代币与以太坊ETH互换的功能，开发者提取互换费用的功能，可以实现机制简单的ICO。

更加详细的来说，本智能合约主要功能是用户需要向智能合约中转入ETH后获得代币，开发者可以提取用户为获得代币而转移到智能合约中的ETH。

打开`src/token.sol`，写入以下内容，或者前往[此处](https://gist.github.com/wangshouh/dc3a0f9b4a1bdd522aa6d38f133a3417)直接下载代码。

```solidity 
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import "solmate/tokens/ERC20.sol";
import "openzeppelin-contracts/contracts/access/Ownable.sol";

error NoPayMintPrice();
error WithdrawTransfer();
error MaxSupply();

contract SDUFECoin is ERC20, Ownable {
    
    uint256 public constant MINT_PRICE = 0.00000001 ether;
    uint256 public constant MAX_SUPPLY  = 1_000_000;

    constructor (
        string memory _name,
        string memory _symbol,
        uint8 _decimals
    ) ERC20 (_name, _symbol, _decimals) {}

    function mintTo(address recipient) public payable {
        if (msg.value < MINT_PRICE) {
            revert NoPayMintPrice();
        } else {
            uint256 amount = msg.value / MINT_PRICE;
            uint256 nowAmount = totalSupply + amount;
            if (nowAmount <= MAX_SUPPLY) {
                _mint(recipient, amount);
            } else {
                revert MaxSupply();
            }
        }
    }

    function withdrawPayments(address payable payee) external onlyOwner {
        uint256 balance = address(this).balance;
        (bool transferTx, ) = payee.call{value: balance}("");
        if (!transferTx) {
            revert WithdrawTransfer();
        }
    }
}
```

第1行代码表示该智能合约属于MIT开源协议;

第2行代码表示该智能合约要求Solidity版本大于0.8.13;

第4、5行代码表示该智能合约需要导入的库;

第7、8、9行代码声明了接下来需要使用的两个错误，`NoPayMintPrice`错误将出现在合约执行人未转入ETH而直接交换代币的情况下; `WithdrawTransfer`发生在非智能合约创造者提取合约资金的情况下;`MaxSupply`错误发生在代币发行量超过阈值。

第11行代码声明智能合约主体，使用`is`标识符表示该智能合约是对`ERC20`和`Ownable`继承，前者主要包括ERC20中的各种核心实现(引用自`solmate`)，后者主要实现了权限控制(引用自`Openzeppelin`)，避免非合约创造者在合约内提取ETH。

第13行代码规定了交换价格为`0.00000001 ether`，该常量确定了1ETH兑换1个代币的量价关系。我们将在后文解释为什么使用`0.00000001`作为比值;

第14行代码规定了代币总发行量为`1_000_000_000`，其为常量; 

第14-20行为对代币基本属性的构造器，其中`_name`规定了代币名称; `_symbol`规定了代币的缩写; `_decimals`规定了代币的基数，类似于ETH中的`ether`单位。我们上文所定义的代币发行总量代表这发现10^10个单位代币，如果你将所有代币铸造出来放在在MetaMask钱包中，显示的数量为100个。换而言之，MetaMask等钱包显示数量总是由铸造出的代币个数除以其基数。这也说明了为什么使用`0.00000001 ether`作为最小铸造价格，该价格可以保证你转入1eth将获得在钱包中显示为1的代币。此处与一般的智能合约不同，我们并没有直接给出常量的内容，在后文部署智能合约的时候，我们会通过注入的方式规定变量名，这极大方便了合约复用。

> 对于ETH的单位的详细说明可以参考这个[网页](https://gwei.io/cn/)

第22-34行规定了最为重要的铸造函数，该函数接受一个变量`recipient`即代币接受者，同时通过`payable`关键词也可接受转入的ETH。如果想获得更多关于`public`和`payable`的信息可以参考这篇[中文教程](https://github.com/AmazingAng/WTFSolidity/tree/main/03_Function#solidity%E6%9E%81%E7%AE%80%E5%85%A5%E9%97%A83-%E5%87%BD%E6%95%B0%E7%B1%BB%E5%9E%8B)。总体而言，该函数可以接受一个规定的变量`recipient`和一个隐含的变量，即转入的ETH数量(通过`msg.value`获得数值)。如果你想更加直观的理解该函数所接受的两个参数，可以前往[这里](https://ropsten.etherscan.io/address/0x6f719490dec688b8c7c394f5259ae5aa788c3a5d#writeContract)查看，或参见下图:

![ethscan.png](https://img.wang.232232.xyz/img/2022/07/16/ethscan.png)

对于铸造函数内的逻辑较为简单，只需要注意`revert`用于报错。而`_mint`函数和`totalSupply`变量实际来自`solmate`，读者可自行查询函数定义。总而言之，`_mint`函数是核心方法，`totalSupply`变量存储有当前的代币总发行量。该变量也可以直接在[etherscan](https://ropsten.etherscan.io/address/0x6f719490dec688b8c7c394f5259ae5aa788c3a5d#readContract)中查阅，或参见下图:

![totalSupplyScan.png](https://img.wang.232232.xyz/img/2022/07/16/totalSupplyScan.png)

该函数的在第一个if判断中实现了规避交换价格低于最低价格的交易；第二个if实现了判断当前总发行量是否超标的逻辑。

> 本智能合约为实现`_burn`函数，该函数的主要作用是燃烧代币，即减少代币数量，可用于通货紧缩的经济模型。

第36-42行实现了提取合约内ETH的功能，参数`payee`是提取地址，该合约通过`external`关键词对`onlyOwner`进行了扩展，而`onlyOwner`的主要作用是检查调用者是否为合约指定的`Owner`，默认为合约创建者，当然也可以通过`Ownable.sol`中实现的`transferOwnership`函数更改合约的`Owner`。第37行代码可以获得该合约内ETH的总量，第38行通过一个底层函数`call`实现资金转移，并将转移的结果赋值给`transferTx`。如果该值为`false`，则证明调用失败。

针对于`call`的具体信息可以参考[中文文档](https://learnblockchain.cn/docs/solidity/units-and-global-variables.html#address-related)

> `call`是一个底层函数，存在一定的安全问题，但可以减少转移时的gas耗费。该函数在未来可能会被弃用。如果想知道比较安全的提取资金的方式可以参考中文Solidity文档中的[讨论](https://solidity-cn.readthedocs.io/zh/develop/security-considerations.html#id4)或者参考著名NFT交易所`Opensea`给出的[示例](https://docs.opensea.io/docs/4-setting-a-price-and-supply-limit-for-your-contract#withdrawing-funds)

在此处的回调中，由于我们严格指定了合约调用人，而且不存在复杂的回调问题，所以使用`call`函数是合理的。而且通过`onlyOwner`实现了所谓[检查-生效-交互](https://solidity-cn.readthedocs.io/zh/develop/security-considerations.html#checks-effects-interactions)模式。

## 智能合约的测试

### 编写测试脚本

为了确保智能合约的安全和有效，我们会对智能合约进行全面的测试。在下文中我们将使用`solidity`编写测试文件全面覆盖每一个函数。为方便读者阅读，首先给出[全文代码](https://gist.github.com/wangshouh/477f142ce69db25f4537ca46e96b9010)。

为方便讨论，我们先给出测试文件的框架:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../src/token.sol";

contract tokenTest is Test {
    SDUFECoin private token;
    using stdStorage for StdStorage;
    address internal constant receiver = address(1);

    function setUp() public {
        token = new SDUFECoin("SDUFECoinTest", "SDCT", 8);
    }

    //后文给出的函数全部位于此处
    
}
```

上述代码给出了各种初始化设置，首先导入了需要测试的`token.sol`和forge库中的`Test.sol`。后者提供了一系列的测试用函数和`cheatcode`。所谓`cheatcode`直译为作弊码，由于测试需要覆盖一些不易发生的特殊情况。我们需要提供作弊的方式直接生成这类特殊情况。在后文中大家会看到一个示例。

第8行声明`token`属于`SDUFECoin`(SDUFECoin由`src/token.sol`定义); 
第9行使用了`Using For`语法，具体可以参考[这篇文章](https://docs.soliditylang.org/en/v0.8.14/contracts.html#using-for);
第10行声明`receiver`变量，该变量为地址类型(address)、常量(constant)、尽可在合约内及继承合约调用(internal)。`address(1)`指地址`0x0000000000000000000000000000000000000001`

> 注意在以太坊中，`address(0)`即0地址具有特殊含义，具体可参考[这篇文章](https://zhuanlan.zhihu.com/p/34363341)。简单来说，该地址是黑洞地址，代币一旦转入无法转出，且该地址可以通过合约调用给出ERC20代币。

第12-14行使用了`setUp()`函数对`SDUFECoin`进行初始化，在此给出在`src/token.sol`中的代码:

```solidity
constructor (
    string memory _name,
    string memory _symbol,
    uint8 _decimals
) ERC20 (_name, _symbol, _decimals) {}
```

通过对比，不难看出我们依次对代币的名称(设置为SDUFECoinTest)、缩写(设置为SDCT)和基数(设置为8)进行了设置。

在下文中，我们会编写一系列的测试用函数，注意测试函数应与`setUp()`函数处以同一层级，保持相同缩进。如果您看不懂这句话，可以参考上文给出的[完整代码](https://gist.github.com/wangshouh/477f142ce69db25f4537ca46e96b9010)

首先编写一个简单的测试函数，目的是测试不给交易费用则交易失败的逻辑。代码如下:

```solidity
function testFailNoMintPricePaid() public {
    token.mintTo(address(1));
}
```
注意测试函数名都应以`test`开头，如果判断失败逻辑应包含`Fail`字段。如上文所述，本次测试判断不输入ETH则失败的逻辑，故而函数名为`testFailNoMintPricePaid`，测试函数都应使用`public`字段声明。

编写测试输入大于最小交易金额E则交易成功的测试函数:
```solidity
function testSwapPaid() public {
    token.mintTo{value: 0.01 ether}(address(1));
}
```

此处在`mintTo`函数后使用`{value: 0.01 ether}`，说明输入0.01 ether，显然该函数应测试的是成功逻辑，所以此处没有`Fail`字段。


测试输入小于最小交易金额E则交易失败的测试函数:
```solidity
function testFailMinPrice() public {
    token.mintTo{value: 0.000000001 ether}(address(1));
}
```

此处转入的金额小于我们所设定的`MINT_PRICE`，应该触发`NoPayMintPrice`错误。

通过输入高额交易量测试超过最大供应量失败的测试函数:
```solidity
function testFailMaxsupply() public {
    token.mintTo{value: 0.015 ether}(address(1));
}
```
此处向`mintTo`函数输入了`0.015 ether`，经过简单计算尽可以得出，输入的金额可以铸造`1_500_000`单位代币，显然大于最大供应量，故而此处应抛出异常，使用`Fail`测试。

与上文内容相同，但此处我们使用`cheatcode`进行测试即通过直接篡改运行时数据测试:
```solidity
function testFailMaxsupplyUseCheat() public {
    uint256 slot = stdstore
        .target(address(token))
        .sig("totalSupply()")
        .find();
    bytes32 loc = bytes32(slot);
    bytes32 mockedTotalSupply = bytes32(abi.encode(1_000_000));
    vm.store(address(token), loc, mockedTotalSupply);
    token.mintTo{value: 0.00000001 ether}(address(1));
}
```

在`slot`变量中使用`target`获得合约地址，使用`sig`对所需获得`totalSupply`变量进行编码，使用`find`取出`totalSupply`当前数值。`loc`变量将`slot`进行`bytes32`编码，`mockedTotalSupply`先对`1_000_000`(该值为最大代币供应量)进行`abi`进行编码，而后转译为`bytes32`。定义完成以上内容后使用`vm.store`将数据直接写入合约代码、对于该函数具体的内容可以参考[文档](https://book.getfoundry.sh/cheatcodes/)。上述内容写入后`totalSupply`值就会变成`1_000_000`，即最大发行量。后文在调用`mintTo`函数铸造代币，如果符合我们的代码逻辑则此处应该抛出异常。出现错误则代表`testFailMaxsupplyUseCheat`通过检查。

> 上述内容涉及到了智能合约底层，简单来说，智能合约需要编译成字节码(bytes32)格式存储在区块链上，EVM虚拟机获取链上编译后的合约然后运行。上文中我们所操作的就是编译后的智能合约，所以大量使用了`bytes32`和`abi`。更多关于`abi`的信息可以参考[这里](https://www.quicknode.com/guides/solidity/what-is-an-abi)

在后文给出的两个函数都较为复杂，主要作用判断`withdrawPayments`函数的作用。

首先我们检测在符合代码逻辑的前提下，合约是否能正确运行。代码如下:

```solidity
function testWithdrawalWorksAsOwner() public {
    address payable payee = payable(address(0x1337));
    uint256 priorPayeeBalance = payee.balance;
    token.mintTo{value: 0.0001 ether}(address(receiver));
    assertEq(address(token).balance, 0.0001 ether);
    uint256 tokenBalance = address(token).balance;
    token.withdrawPayments(payee);
    assertEq(payee.balance, priorPayeeBalance + tokenBalance);
}
```

首先我们声明一个提取合约ETH的接收者，即`payee`;
然后定义存储当前接受者在接受ETH前的账户余额，方便后文进行判断;
然后调用`mintTo`函数进行铸造，保证合约内存储一定量的ETH，方便后文进行提取操作;

`assertEq(address(token).balance, 0.0001 ether);`此行代码判断合约内的ETH余额是否等于`0.0001 ether`，即上文的铸造费用。在测试环境内没有gas费用所以一定相等。更多关于`assertEq`的描述可以参考[文档](https://book.getfoundry.sh/reference/ds-test#asserteq)

`uint256 tokenBalance = address(token).balance;`将合约内的ETH余额数量赋值给`tokenBalance`

`token.withdrawPayments(payee);`调用`withdrawPayments`函数将ETH提取给`payee`

> 此处其实是由合约创立者调用的函数，如果不使用`startPrank`等函数则`msg.sender`属性不会改变，即所有函数都隐含有合约创立者调用。

`assertEq(payee.balance, priorPayeeBalance + tokenBalance);`此行代码较为简单，即判断完成ETH提取后账户余额是否等于提取ETH前的余额和合约内的余额。如果两者相等，则证明函数逻辑没有问题。

最后测试在不是合约创立者的情况下提取合约内ETH失败的逻辑，由于此处为测试失败，所以测试函数名内含有`Fail`字段。

```solidity
function testWithdrawalFailsAsNotOwner() public {
    token.mintTo{value: token.MINT_PRICE()}(address(receiver));
    assertEq(address(token).balance, token.MINT_PRICE());
    vm.expectRevert("Ownable: caller is not the owner");
    vm.startPrank(address(0xd3ad));
    token.withdrawPayments(payable(address(0xd3ad)));
    vm.stopPrank();
}
```

此处省略前两行代码的具体含义。直接讨论较为重要且前文未提及的代码。

`vm.expectRevert("Ownable: caller is not the owner");`该行代码的作用为清空前文所有`cheatcode`的状态。一般来说调用此函数后一般立即`Prank`。如果想知道更多关于`expectRevert`的信息可以参考[文档](https://book.getfoundry.sh/cheatcodes/expect-revert#expectrevert)

`vm.startPrank(address(0xd3ad));`该行代码改变了默认的调用者，此处将调用者改为`0xd3ad`，通过此行代码的调用，意味着下文的代码调用者不再是合约创建者，理论上合约应该会调用失败。在下文，我们直接调用了`token.withdrawPayments(payable(address(0xd3ad)));`，尝试将合约内的资金提取给非合约创建者，此行代码应抛出异常。

`vm.stopPrank();`停止调用者切换。

### 运行测试脚本

对于测试脚本的运行较为简单，直接在终端内输入下述命令即可:

```bash
forge test
```

如果一切顺利，读者将会看到类似下图的输出:

![erc20TestOutput.png](https://img.wang.232232.xyz/img/2022/07/16/erc20TestOutput.png)

此图证明所有的测试都已经通过，我们接下来可以进行一系列其他操作。

下文我们将给出最为简单但极其有用的gas报告。

在终端输入下述命令:
```bash
forge test --gas-report
```

读者可获得类似下图的输出:

![erc20gasReport.png](https://img.wang.232232.xyz/img/2022/07/16/erc20gasReport.png)

在此处，gas费的单位应该是`gwei`，其值相当于`0.000000001Ether`

## 智能合约的部署

### 智能合约部署脚本

`foege`提供了`forge create`命令用于合约部署，该命令具有较多参数，对于复杂的合约部署需要书写较为复杂的命令。在此处，我们使用另一种更加方便且易读的`solidity script`的方式部署合约。如果读者对`forge create`感兴趣，可以自行参考[文档](https://book.getfoundry.sh/forge/deploying)

`solidity script`方式部署合约，正如其名，我们需要使用`solidity`编写部署脚本，一般来说都较为简单，我们需要在`script/token.s.sol`中写入以下内容:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Script.sol";
import "../src/token.sol";

contract TokenScript is Script {
    function run() external {
        vm.startBroadcast();

        SDUFECoin token = new SDUFECoin("SDUFECoinTest", "SDCT", 8);

        vm.stopBroadcast();
    }
}
```

如果读者已经自行阅读过上文给出的代码说明，此脚本内容应该可以都读懂，此处将主要解释`vm.startBroadcast()`和`vm.stopBroadcast()`

`vm.startBroadcast()`代表广播开始，读者如果了解区块链技术底层，应知道每一笔交易都需要进行全网广播以保证每一个节点获得交易信息并进行存储，在一定时间内将其打包进入区块。智能合约的部署与之类似，也需要广播合约的字节码，保证节点将其打包进入以太坊区块。`startBroadcast`的功能正是开启广播，以为这在此后的代码将被广播进入区块。

`SDUFECoin token = new SDUFECoin("SDUFECoinTest", "SDCT", 8);`此行代码说明了构造了一个完整的代币，此行代码所对照的字节码将被广播，这一意味着合约上链。

`vm.stopBroadcast`代币广播关闭，我们已将所需要的代码进行了广播，所以此处关闭了广播功能。

此部分内容也可以参考官方文档[Solidity Scripting](https://book.getfoundry.sh/tutorials/solidity-scripting#solidity-scripting)，内容基本与上述内容一致。

### 智能合约的本地部署

合约的本地部署可以认为是系统性测试的一种。把智能合约部署到本地网络中，我们可以通过`cast`等工具调用合约函数，得到在区块链真实环境的测试结果，与上文提出的测试函数相比更加直观。

在本节内容中，我们主要使用以下两种工具:

1. `anvil`，该工具主要用于搭建一个完全本地化的以太坊环境，并提供10个含有1000 ETH的账户。在此节内容中，我们主要使用一些较为简单的功能，`anvil`有`fork`特定区块等高级功能，但在此处我们并不会使用，若想了解具体内容，可以参考[Overview of Anvil](https://book.getfoundry.sh/anvil/)和[anvil reference](https://book.getfoundry.sh/reference/anvil/)，此节内容主要使用前者所介绍的功能，如果向更加全面的了解此模块，建议阅读后者。

2. `cast`，工具主要用于与区块链RPC进行交互，比如进行合约内函数调用、发起交易、查询链上数据等，可以认为是一个命令行式的`etherscan`。可以参考[Overview of Cast](https://book.getfoundry.sh/cast/)和[cast reference](https://book.getfoundry.sh/reference/cast/)

在终端运行`anvil`命令，你将看到如下输出:

![anvailOutput](https://img.wang.232232.xyz/img/2022/07/17/anvailOutputc024161a0b159123.png)

**注意该终端窗口不可关闭。**

在项目根目录下创建`.env`文件，输入以下内容:
```
LOCAL_ACCOUNT=ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
```

注意`ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80`应自行替换成你输出的结果中的任一一个`Private Keys`值。

运行以下命令将 `.env` 中的数据读取为环境变量:

```bash
source .env
```

在终端内输入以下命令进行合约部署:
```bash
forge script script/token.s.sol:TokenScript --fork-url http://localhost:8545  --private-key $LOCAL_ACCOUNT --broadcast
```

如果正确部署则应得到如下结果:
![localBlockChainDeploy](https://img.wang.232232.xyz/img/2022/07/17/localBlockChainDeploye4286cb9f805c704.png)

在运行`anvil`的终端窗口内应看到如下输出:

![anvailDeploy158e6bd2d0baccba.png](https://img.wang.232232.xyz/img/2022/07/17/anvailDeployd39edcfee3d516c4.png)

为方便终端命令编写，使用下述命令将合约地址保存为系统变量:
```bash
export TOKEN_ADDRESS="0xe7f1725e7734ce288f8367e1bb143e90bb3f0512"
```

首先我们查询以下部署的智能合约的各项属性是否正确:

![contructorTest.png](https://img.wang.232232.xyz/img/2022/07/17/contructorTest41da6de23c567c52.png)

此处使用的命令为`cast call $TOKEN_ADDRESS "name()(string)`类型，`(string)`说明对以太坊测试网络返回的结果应该如何解析成何种数据类型，默认返回16进制数字，在不指定数据类型的情况下，几乎无法阅读。

`cast call`的作用是在不发起交易的情况下获得智能合约的属性，对应`etherscan`中的`readContract`界面，前往[此网页](https://ropsten.etherscan.io/address/0x6f719490dec688b8c7c394f5259ae5aa788c3a5d#readContract)可以查看。

经过简单的核对，我们发现这与我们部署时的属性完全相同。

> 由于`foundry`仍在更新，如果上述命令无法正常运行，请自行参考[文档](https://book.getfoundry.sh/reference/cast/cast-call)修改

我们主要验证一个核心函数，即`mintTo`函数，在终端内输入以下命令:
```bash
cast send --value 0.0001ether --private-key $LOCAL_ACCOUNT $TOKEN_ADDRESS "mintTo(address)" 0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266 -j
```

`cast send`主要用于签名发布交易，此处中的`0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266`可以替换成任一一个公钥地址。`--value`代表发送的代币数量，`--private-key`代表私钥，`-j`代表以`json`的形式进行输出。

此处最难理解的应该是`"mintTo(address)"`该行代表调用`mintTo`函数，括号内表明编码的数据类型，即将`0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266`编码为`address`类型。如果函数需要多个参数，在函数名后面可以逐个列出并使用空格分割，比如`cast send --ledger 0x... "deposit(address,uint256)" 0x... 1`

对于`cast send`的命令，更多信息请参考[文档](https://book.getfoundry.sh/reference/cast/cast-send)

完成上述操作后，我们使用`cast call`命令获取以下合约内的发行量(totalSupply)属性，命令如下:
```bash
cast call $TOKEN_ADDRESS "totalSupply()(uint256)"
```

完整输出结果截图如下:
![castSendMintTo](https://img.wang.232232.xyz/img/2022/07/17/castSendMintTo4fd00c9cc7ef40a5.png)

由于`withdrawPayments`具有较高风险，所以在此处我们也对其进行测试，使用以下命令:
```bash
cast send --private-key 59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d $TOKEN_ADDRESS "withdrawPayments(address)" 0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266 -j
```

此处，我们将`private-key`的值进行了替换，即合约调用者已不再是合约的拥有者，该命令运行后会得到报错。如下:
```
ProviderError(JsonRpcError(JsonRpcError { code: 3, message: "execution reverted: Ownable: caller is not the owner", data: Some(String("0x08c379a0000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000204f776e61626c653a2063616c6c6572206973206e6f7420746865206f776e6572")) }))
```

然后我们使用正确的合约拥有者调用此函数，命令如下:
```bash
cast send --private-key $LOCAL_ACCOUNT $TOKEN_ADDRESS "withdrawPayments(address)" 0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266 -j
```

输出结果如下:
```json
{
   "blockHash":"0xad1145a52589d4305b7504b18054b8457645be28cb7cc6e88fe7e13c808862e0",
   "blockNumber":"0x5",
   "contractAddress":null,
   "cumulativeGasUsed":"0x78a5",
   "effectiveGasPrice":"0xd20b3bd6",
   "from":"0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266",
   "gasUsed":"0x78a5",
   "logs":[
      
   ],
   "logsBloom":"0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
   "status":"0x1",
   "to":"0xe7f1725e7734ce288f8367e1bb143e90bb3f0512",
   "transactionHash":"0x9ed612dba2182a734121ad150220864e051ab92e66f82de1ca4354d3401975c7",
   "transactionIndex":"0x0"
}
```

经过上述测试，说明该函数的运行是正常的。

如果读者想继续进行测试，可以自行测试`transferOwnership`等函数，这些函数由`openzeppelin/Ownable.sol`定义。

当然，读者也可自行尝试其他`cast`命令，如`cast balance`(此命令用于查询指定地址的ETH余额，参见[cast-balance](https://book.getfoundry.sh/reference/cast/cast-balance)文档)

如果进行完成所有测试，可以前往运行`anvil`的终端，使用`Crtl + C`快捷键结束运行。

将合约部署到本地网络其实也是一种测试手段，在下文我们将真正的将合约部署到以太坊上。

### 智能合约的网络部署

通过上述大量的测试，我们可以确定合约可以部署到以太坊环境中，故而在本篇博客的最后，我们将介绍如何将合约上链。

合约上链需要以下准备:

1. 以太坊钱包;

2. 以太坊RPC服务商;

3. etherscan API密钥

考虑到许多初级开发者可能没有准备足够的ETH，所以我们选择将合约部署到测试网络。在前文(初始化开发环境)中，读者应该已经准备好了钱包和测试用ETH，此处不再赘述如何获得。

我们首先介绍如何获得Ethersan的API密钥。该过程较为简单，步骤如下:

1. 前往Etherscan的官网[注册页面](https://etherscan.io/register)

2. 依次填入各类信息

3. 等待Etherscan的激活邮件，点击邮件中的激活链接认证账户

4. 前往[MyAPIkey页面](https://etherscan.io/myapikey)，点击`Add`，输入API名称即可。

最终结果如下图:

![etherscanAPI](https://img.wang.232232.xyz/img/2022/07/17/etherscanAPI2e1f373733e81b35.png)

打开`.env`文件输入以下内容:
```
ETHERSCAN_KEY=你的API密钥
```

其次，我们需要一个RPC接口与以太坊网络交互。由于我们没有在本地运行完整的以太坊本地节点，所以我们没有办法直接与以太坊网络通信。一种比较简单的与以太坊网络通信的方法就是借助`Relay Network`，或者简单的认为是一个API接口。市面上有非常多的服务商提供此类服务，较为著名的有`infura`和`alchemy`。前者是全球最大的`Relay Network`服务商，也是我所使用的服务商。该服务商提供了每日10万次的免费调用额度，而且仅使用以太坊网络不需要绑定VISA等信用卡。

![infuraPrice.png](https://img.wang.232232.xyz/img/2022/07/17/infuraPrice91fd149912c1379c.png)

获得RPC URL的步骤如下:

1. 前往[注册页面](https://infura.io/register)注册账户并通过邮件激活账户

2. 在Welcome页面随便选择，点击`sumbit`提交

3. 在`Create your first project`界面选择`Ethereum`，项目名字随便取一个

4. 在`KEYS`选项卡内更改`ENDPOINTS`至你想要的测试网络，在此处我选择了`Ropsten`，读者可根据手中持有的测试ETH选择对应的网络

5. 复制下图红框内的链接

![infuraAPISet](https://img.wang.232232.xyz/img/2022/07/17/infuraAPISetf36450a33c0c10dc.png)

打开`.env`文件，输入以下内容:
```
ROPSTEN_RPC_URL=替换为自己的网址
```

最后，我们需要导出一个极其敏感的数据，即以太坊账户私钥，这里非常建议您使用新建的账户。此处以MetaMask浏览器扩展为例，如果你选择了其他钱包请自行查找有关教程。

首先按照下图操作:

![metamaskFirstStep.png](https://img.wang.232232.xyz/img/2022/07/17/metamaskFirstStep4fae46d6481d14bc.png)


操作结束后应获得下图结果:

![metamaskStep2.png](https://img.wang.232232.xyz/img/2022/07/17/metamaskStep2b99f205b61f42763.png)

点击`导出私钥`的按钮，将显示的私钥复制下来，写入`.env`
```
PRIVATE_KEY=自行替换
```

经过上述操作我们已基本完成了部署到以太坊网络中的大部分操作。

在终端内依次输入下述命令:
```bash
#读取.env文件内的内容并将其保存为环境变量
source .env
#与部署到本地网络类似，使用命令部署到以太坊中
forge script script/token.s.sol:TokenScript --rpc-url $ROPSTEN_RPC_URL  --private-key $PRIVATE_KEY --broadcast --verify --etherscan-api-key $ETHERSCAN_KEY -vvvv
```

> 此命令中的`-vvvv`的含义是显示4级测试输出，具体可参考[文档](https://book.getfoundry.sh/forge/tests?highlight=vvvv#logs-and-traces)

等待以太坊网络确认并认证，最终输出如下图:

![TestOutput](https://book.getfoundry.sh/images/solidity-scripting/contract-verified.png)

> 此图仅为示例，具体内容由于合约地址会出现不同

点击输出结果中的`URL`，即可再EtherScan中访问智能合约，此处给出本次教程部署的[智能合约网址](https://ropsten.etherscan.io/address/0x6f719490dec688b8c7c394f5259ae5aa788c3a5d)。正如上文所述，通过Etherscan提供的`readContract`和`writeContract`也可以可视化的进行一系列测试。

## 总结

本文完整介绍了智能合约的开发工作流，不同于很多其他语言，智能合约由于其新生性，很多文档并不全面，尤其设`solmate`等库，完全没有文档。这要求读者需要自行阅读合约源码，或者选择`openzeppelin`这种[文档](https://docs.openzeppelin.com/contracts/4.x/)较为完整的库。读者读完此文章后如感觉意犹未尽，可以参考Foundry给出的`NFT tutorial`部署一个NFT智能合约，官网中解释并不十分详细，但也给出了完整的代码与开发流程，可以前往[此网页](https://book.getfoundry.sh/tutorials/solmate-nft)查看。本教程未来一定会更新，你可以订阅本博客的[RSS](https://blog.wongssh.cf/atom.xml)获取最新的文章。
