---
title: "Foundry教程：使用多种方式编写可升级的智能合约(上)"
date: 2022-07-26T21:07:17Z
tags: [EternalStorage, EIP-897, EIP-1822, solidity]
aliases: ["2022/07/18/foundry-contract-upgrade-part1"]
---

## 概述

在以太坊智能合约中，很长时间都保持着“一次部署，永不修改”的情况。但随着智能合约的逐渐发展，出现了诸如修复BUG、增加特性、修复漏洞等需要修改智能合约的需求，我们非常希望可以编写可升级的智能合约。

经过智能合约开发的不断努力和`solidity`语言的创新，编写可升级的智能合约成为显示本文主要介绍在智能合约部署过程中，我们可以通过多种方式编写可升级的智能合约。由于该方面的标准较多，我们无法在一篇内解释所有的方法，所以此篇仅解释了在以太坊历史上属于较为早期的方案，名单如下:

- Eternal Storage
- EIP-897 Proxy
- EIP-1822 UUPS

本教程所使用的代码均可在[github仓库](https://github.com/wangshouh/upgradeContractLearn)。

## Eternal Storage

该方法由[openzeppelin](https://github.com/OpenZeppelin/openzeppelin-labs/tree/master/upgradeability_using_eternal_storage)提出，方法最早的思想来源于[此博客](https://blog.colony.io/writing-upgradeable-contracts-in-solidity-6743f0eecc88/)。

**注意该方法不能实现真正的合约升级，且适用范围有限**

该方法是为了解决合约重新部署后，原始合约内的数据消失的问题，基本思路是将合约的逻辑部分与数据存储分离，当我们需要重新部署合约时，将新合约的数据来源指向数据存储合约。从该方法的思路看出，这种方法的适用范围非常小，故而本节内容主要是为了让读者熟悉以太坊智能合约互相调用和多合约部署等基础知识。

### 存储智能合约

由于在[上一篇文章](https://blog.wongssh.cf/2022/07/14/foundry-with-erc20/)或者[CSDN](https://blog.csdn.net/WongSSH/article/details/125837346)内，我们已经完整介绍了`foundry`环境的搭建，所以此文中我们将省略一部分对foundry框架的具体解释。

首先创建一个完整的项目:
```bash
forge init upgradeContract
```

在`src\EternalStorage`文件夹下创建专用于数据存储的合约`EternalStorage.sol`，该合约主要用于数据存储。在合约内写入以下内容:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

contract EternalStorage {
    mapping(bytes32 => uint256) public UIntStorage;

    function getUIntValue(bytes32 record) public view returns (uint256) {
        return UIntStorage[record];
    }

    function setUIntValue(bytes32 record, uint256 value) public {
        UIntStorage[record] = value;
    }
}
```
上述代码逻辑较为简单，最关键的是该合约维护了两个用于数据存储的映射，并提供了对外接口获取或设置映射值，对于使用Java等面向对象程序员来说这是一种常见的操作。

该代码内是在本系列教程内第一次出现`view`，所以在此处，我们对其进行解释。`view`关键词的作用是使函数可以读取但不能改变链上变量。

虽然合约内容较为简单，但我们仍提倡进行合约测试。由于我们在此项目内没有使用传统的项目框架，所以我们不能使用上一个教程内提到的`forge remappings > remappings.txt`生成映射文件避免报错。我们需要在项目根目录下创建`.vscode\settings.json`文件，写入以下内容:
```json
{
    "solidity.packageDefaultDependenciesContractsDirectory": "src",
    "solidity.packageDefaultDependenciesDirectory": "lib"
}
```

在`test/EternalStorage/EternalStorage.t.sol`文件内写入测试代码:
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../../src/EternalStorage/EternalStorage.sol";

contract ContractTest is Test {

    EternalStorage private storageEth;

    function setUp() public {
        storageEth = new EternalStorage();
    }

    function testGetIntValue() public {
        storageEth.setUIntValue(keccak256('votes'), 1);
        uint256 intValue = storageEth.getUIntValue(keccak256('votes'));
        assertEq(intValue, 1);
    }
}
```

该合约内出现了`keccak256`函数，该函数的作用是获得信息的摘要值，使用的算法名称为`keccak256`(SHA3算法)，具体可参见[维基百科](https://zh.wikipedia.org/zh-cn/SHA-3)。如果想进一步了解该函数，建议阅读《图解密码学技术》第7章。

运行`forge test`命令，发现所有测试均可通过，我们将进入下一阶段，编写调用数据存储存储智能合约的功能性合约。

### 功能智能合约

该合约的主要作用使进行投票并获得投票后的结果，为了尽可能减少文章长度，此处给出的代码仅用了表达思想，而没有考虑太多的实用性。在`src/EternalStorage/voteFirst.sol`写入下述代码:
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

error GetValueFail();

contract VoteFirst {
    address eternalStorage;

    constructor(address _eternalStorage) {
        eternalStorage = _eternalStorage;
    }

    function getNumberOfVotes() public returns (uint256) {
        (bool success, bytes memory data) = eternalStorage.call(
            abi.encodeWithSignature("getUIntValue(bytes32)", keccak256("votes"))
        );
        if (!success) {
            revert GetValueFail();
        }

        return abi.decode(data, (uint256));
    }

    function vote() public {
        uint256 voteNum = getNumberOfVotes() + 1;
        (bool success, ) = eternalStorage.call(
            abi.encodeWithSignature(
                "setUIntValue(bytes32,uint256)",
                keccak256("votes"),
                voteNum
            )
        );
        if (!success) {
            revert("Call Error");
        }
    }
}

```

该合约使用`call`函数，该方法存在风险，但仅需要知道对方的地址和函数名即可运行，更加方便。`call`通过二进制编码调用目标地址内的函数。为了获取指定函数的二进制编码，此处使用了`abi.encodeWithSignature`函数，正如代码所示，此函数第一个参数是函数的名称(包含数据类型)，其他参数为函数所需的具体参数。读者可以在可以阅读[WTF Solidity 第22讲 Call](https://github.com/AmazingAng/WTFSolidity/tree/main/22_Call)获得更多信息。

> 由于此合约需要在`EternalStorage`环境下运行所以使用了`call`而非`delegatecall`。两者更加具体的区别可以参考[WTFSolidity 第23讲 delegatecall](https://github.com/AmazingAng/WTFSolidity/tree/main/23_Delegatecall#delegatecall)

值得注意的是，由于在此合约内使用了`call`函数，导致存储合约接收到的`msg.value`不再是用户的地址而是`VoteFirst`合约的地址，如果希望在`VoteFirst`合约内加入有关用户地址的代码，请在`VoteFirst`内将`msg.value`直接作为参数传递给`VoteFirst`而不是在`EternalStorage`合约内使用`msg.value`属性。下图很好的说明了这一点:

![Call](https://img.wang.232232.xyz/img/2022/07/19/callAddress34aa16020bf82c2a.png)

此图来自[WTFSolidity 第23讲 delegatecall](https://github.com/AmazingAng/WTFSolidity/tree/main/23_Delegatecall#delegatecall)

在此处，我们编写一个简单的测试函数(`test/EternalStorage/StoragteUpdate.t.sol`):
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../../src/EternalStorage/EternalStorage.sol";
import "../../src/EternalStorage/voteFirst.sol";

contract ContractTest is Test {
    EternalStorage private storageEth;
    VoteFirst private voteFirst;

    function setUp() public {
        storageEth = new EternalStorage();
        address storageAddress = address(storageEth);
        voteFirst = new VoteFirst(storageAddress);
    }

    function testGetValueFromFirst() public {
        voteFirst.vote();
        uint256 voteNum = voteFirst.getNumberOfVotes();
        assertEq(voteNum, 1);
    }
}
```

该测试函数较为简单，核心在于使用`address()`函数获得`EternalStorage`合约部署地址，再使用此地址作为构造参数构造`voteFirst`合约。使用`forge test`进行测试。

在此处，我们也将代码部署到`anvil`中进行测试。具体的环境搭建请参考[上一篇教程](https://blog.wongssh.cf/2022/07/14/foundry-with-erc20/)。在此处，我们默认你已经完成了`anvil`的启动，并且设置了`LOCAL_ACCOUNT`系统变量为你的私钥，`LOCAL_RPC_URL`为`anvil`的API地址(默认为`http://127.0.0.1:8545`)。由于此处部署的合约较多，所以我们将使用`forge create`方法，具体可参考[文档](https://book.getfoundry.sh/forge/deploying)

首先部署`src/EternalStorage/EternalStorage.sol`合约
```bash
forge create --rpc-url $LOCAL_RPC_URL --private-key $LOCAL_ACCOUNT src/EternalStorage/EternalStorage.sol:EternalStorage
```

将`anvail`中输出的区块地址使用`export`命令保存为`STORAGE_ADDRESS`，在此处我使用的命令如下：
```bash
export STORAGE_ADDRESS=0x5fbdb2315678afecb367f032d93f642f64180aa3(该地址请自行更改)
```

接下来我们使用此地址作为构造器参数部署`src/EternalStorage/voteFirst.sol`代码中的`VoteContract`合约
```bash
forge create --rpc-url $LOCAL_RPC_URL --constructor-args $STORAGE_ADDRESS --private-key $LOCAL_ACCOUNT src/EternalStorage/voteFirst.sol:VoteFirst
```
将此合约地址保存为`V1`系统变量。
```bash
export V1=0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512
```

使用`cast send`调用`vote`函数进行测试:
```bash
cast send --private-key $LOCAL_ACCOUNT $V1 "vote()"
```

为了确定调用结果已经写入，我们使用`cast call`调用`getNumberOfVotes`函数:
```bash
cast call $V1 "getNumberOfVotes()(uint256)"
```

如果一切正常，此处应返回`1`

### 升级部署

由于在`src/EternalStorage/voteFirst.sol`中使用了`call`函数，在后续开发过程中，你发现该函数在[官方文档](https://docs.soliditylang.org/en/latest/units-and-global-variables.html#members-of-address-types)中被警告`You should avoid using .call() whenever possible when executing another contract function as it bypasses type checking, function existence check, and argument packing.`

基于此警告，你希望改用其他方法编写智能合约，即升级当前的智能合约。我们在`src/EternalStorage`文件夹下新建了`voteSecond.sol`文件，并写如了以下代码:
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

contract EternalStorage {
    mapping(bytes32 => uint256) public UIntStorage;

    function getUIntValue(bytes32 record) public view returns (uint256) {
        return UIntStorage[record];
    }

    function setUIntValue(bytes32 record, uint256 value) public {
        UIntStorage[record] = value;
    }
}

contract VoteSecond {
    address eternalStorage;

    constructor(address _eternalStorage) {
        eternalStorage = _eternalStorage;
    }

    function getNumberOfVotes() public view returns(uint) {
        return EternalStorage(eternalStorage).getUIntValue(keccak256('votes'));
    }

    function vote() public {
        uint256 orgianVote = EternalStorage(eternalStorage).getUIntValue(keccak256('votes'));
        EternalStorage(eternalStorage).setUIntValue(keccak256('votes'), orgianVote+1);
    }
}
```

上述代码展示了在已知需调用合约代码的情况下的一种跨合约调用方式，核心为`_Name(_Address)`。`_Name`为需要调用合约的名称，在此处为`EternalStorage`，也正是我们存储数据合约的名称。`_Address`为合约地址。我们可以非常方便的在`_Name(_Address)`后调用`_Name`中的函数。这种方法更加倍推荐，也更加易于理解。

我们需要编写测试函数对合约是否升级生效进行测试，以下代码依旧写在`test/EternalStorage/StoragteUpdate.t.sol`文件内:
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
// import "../../src/EternalStorage/EternalStorage.sol";
import "../../src/EternalStorage/voteFirst.sol";
import "../../src/EternalStorage/voteSecond.sol";

contract ContractTest is Test {
    EternalStorage private storageEth;
    VoteFirst private voteFirst;
    VoteSecond private voteSecond;

    function setUp() public {
        storageEth = new EternalStorage();
        address storageAddress = address(storageEth);
        voteFirst = new VoteFirst(storageAddress);
        voteSecond = new VoteSecond(storageAddress);
    }

    function testGetValueFromFirst() public {
        voteFirst.vote();
        uint256 voteNum = voteFirst.getNumberOfVotes();
        assertEq(voteNum, 1);
    }

    function testGetValueFromSecond() public {
        voteSecond.vote();
        uint256 voteNum = voteSecond.getNumberOfVotes();
        assertEq(voteNum, 1);
    }

    function testCrossGetValue() public {
        voteFirst.vote();
        uint256 voteNum = voteSecond.getNumberOfVotes();
        assertEq(voteNum, 1);
    }
}

```
注意此处删去了对`EternalStorage.sol`合约的导入，原因在于`voteSecond.sol`中已经含有此合约。

对于此合约的部署方式如下:
```bash
forge create --rpc-url $LOCAL_RPC_URL --constructor-args $STORAGE_ADDRESS --private-key $LOCAL_ACCOUNT src/EternalStorage/voteSecond.sol:VoteSecond
```
当然也与上次相同。我们将合约地址保存为系统变量:
```bash
export V2=0xCf7Ed3AccA5a467e9e704C703E8D87F634fB0Fc9
```

首先，我们测试一下该合约是否仍可访问上个合约内的数据:
```
cast call $V2 "getNumberOfVotes()(uint256)"
```

根据返回的结果，我们发现数据之前的投票数据仍可访问。读者可自行进行其他测试，会发现`voteFirst`与`voteSecond`由于共用`EternalStorage`合约存储数据，双方的数据可以互相修改、互相访问。

当然，如果没有其他操作，我们第一次部署的`voteFirst`将会一直运行(当然存在合约销毁的方式，在此处我们不再赘述)

### 优缺点

优点:
- 简单易懂。如果读者有OO的开发经验，对此方式并不陌生。
- 没有使用任何附加库，代码总体而言较为简洁
- 部署方便，只需要多运行一个用于数据存储的智能合约

缺点:
- 会出现多合约并行的情况，无法阻止旧合约访问和修改数据
- 合约地址改变，需要在接入端更改合约地址
- 由于无法增加存储变量，所以对旧合约进行完全的升级

在现实世界中，[morpher](https://www.morpher.com/)使用了这种方法，他们的数据存储合约在[这里](https://github.com/Morpher-io/MorpherProtocol/blob/master/contracts/MorpherState.sol)，该合约对应的功能性合约是一个ERC20代币合约，代码在[这里](https://github.com/Morpher-io/MorpherProtocol/blob/master/contracts/MorpherToken.sol)

## EIP-897 Proxy

EIP-897是由`openzeppelin-labs`提出一种可更新合约的编写方式。它使用`delegatecall`函数通过代理运行的方式实现可升级合约。具体思路是首先编写代理合约，代理合约中含有`fallback()`函数，将所有请求转发至此函数内运行。`fallback`函数内通过`solidity assembly`提供的底层方法将`calldata`(即用户请求数据)转发到逻辑函数内运行，运行完毕后再将数据返回给用户。示意图如下:

![proxyContract](https://s-bj-3358-blog.oss.dogecdn.com/svg/proxyContract.svg)

此示意图没有考虑存储模型，如果读者的项目不涉及存储则可以使用。

### 原理解析

此节内容大量涉及`solidity assembly`的内容，建议读者阅读以下文章:

- [Understanding Ethereum Smart Contract Storage](https://programtheblockchain.com/posts/2018/03/09/understanding-ethereum-smart-contract-storage/)。此文介绍在`solidity`底层数据的存储方式

- [A Practical Introduction To Solidity Assembly: Part 0](https://mirror.xyz/0xB38709B8198d147cc9Ff9C133838a044d78B064B/nk40v2MJKSHXXNSlbqqhpwJf4MtZ9V2Vp8P_bSNwjYc)。该文也介绍了很多智能合约底层内容，仅阅读`Part 0` 即可理解此节中的大部分内容。

- [Solidity汇编 中文文档](https://solidity-cn.readthedocs.io/zh/develop/assembly.html#id4)，可以以此文档作为参考，随时查阅。


通过上文给出的思路，我们发现对于代理合约最重要的应该是`delegatecall`函数，我们上文中给出的`delegatecall`或者`call`函数都使用了`abi.encodeWithSignature()`函数对函数名进行编码然后调用。显然在无法知道具体函数名的代理合约中，我们无法使用此方法。我们只能考虑使用最底层的`delegatecall`的汇编形式，具体如下`delegatecall(g, a, in, insize, out, outsize)`，其中`g`为`gas`费用、`a`为调用函数的地址、`in`表示输入内存的开始地址、`insize`表示输入内存的长度。`out`与`outsize`表示输出的地址和长度。我们自然可以想到可以通过直接将用户发送`calldata`转移给`delegatecall`函数实现对合约代理。

> `calldata`中含有函数名和函数参数信息。一个具体的例子是`0x6057361d000000000000000000000000000000000000000000000000000000000000000a`，对于此`calldata`，它的前4字节，即`6057361d`是一个函数选择器(Selector)，后面字节的内容是调用函数所需的数据。这个数据是有限的，所以在`solidity`编程中我们需要注意函数的入参数量不能大于6个。跟过关于函数选择器的内容可以参考[教程](https://github.com/AmazingAng/WTFSolidity/tree/main/29_Selector)

代理合约较为简单，在`src/EasyProxy/ProxyEasy.sol`内写入以下内容:
```solidity
contract ProxyEasy {
    public address LogicAddress

    function setLogicAddress(address _logicContract) public {
        LogicAddress = _logicContract
    }

    fallback() external {
        address _impl = LogicAddress;

        assembly {
            let ptr := mload(0x40)
            calldatacopy(ptr, 0, calldatasize())
            let result := delegatecall(gas(), _impl, ptr, calldatasize(), 0, 0)
            let size := returndatasize()
            returndatacopy(ptr, 0, size)

            switch result
            case 0 {
                revert(ptr, size)
            }
            default {
                return(ptr, size)
            }
        }
    }
}
```

这是一个简单的代理合约。在此我们只讨论`fallback`函数中的`solidity assembly`部分。该部分参考了[openzeppelin blog](https://blog.openzeppelin.com/proxy-patterns/)。

`let ptr := mload(0x40)`该指令从`0x40`位置获得一个空闲内存块的指针。该内容可以参见[英文文档](https://docs.soliditylang.org/en/v0.8.15/internals/layout_in_memory.html#layout-in-memory)。

`calldatacopy(ptr, 0, calldatasize())`指令主要使用了`calldatacopy`操作码，该操作码的形式为`calldatacopy(t, f, s)`，功能是从调用数据的位置 f 的拷贝 s 个字节到内存的位置 t。在此处的功能是将所有的`calldata`复制到上文所述的`ptr`指针中。

`let result := delegatecall(gas(), _impl, ptr, calldatasize(), 0, 0)`指令主要使用了`delegatecall`，该操作码的形式和功能已在上文给出。此处将`out`与`outsize`设置为0的原因是我们暂时不知道返回值大小，所以将其统一放在暂存区，后续过程中我们可以通过`returndata`和`returndatasize`访问这两个数据。`let size := returndatasize()`获得`returndata`的字节长度，`returndatacopy(ptr, 0, size)`此指令将`returndata`从暂存区复制出来。`switch`代码块实现错误处理。

通过上述底层操作，我们成功实现了函数的委托代理运行。成功解决了代理合约中的一大问题。我们可能会写出下列错误代码(`src/EasyProxy/ProxyError.sol`):
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

contract NumberStorage {
    uint256 public number;

    function setNumber(uint256 _uint) public {
        number = _uint;
    }

    function getNumber() public view returns (uint256) {
        return number;
    }
}

contract ProxyEasy {
    address public otherContractAddress;

    constructor(address _otherContract) {
        otherContractAddress = _otherContract;
    }

    function setOtherAddress(address _otherContract) public {
        otherContractAddress = _otherContract;
    }

    fallback() external {
        address _impl = otherContractAddress;

        assembly {
            let ptr := mload(0x40)
            calldatacopy(ptr, 0, calldatasize())
            let result := delegatecall(gas(), _impl, ptr, calldatasize(), 0, 0)
            let size := returndatasize()
            returndatacopy(ptr, 0, size)

            switch result
            case 0 {
                revert(ptr, size)
            }
            default {
                return(ptr, size)
            }
        }
    }
}

```

你可以将此合约部署到`anvil`中，部署方法为:

1. 部署`NumberStorage`，并保存其部署地址至系统变量`ERR_NUM`:`forge create --private-key $LOCAL_ACCOUNT src/EasyProxy/ProxyError.sol:NumberStorage`
1. 使用第一步中的部署地址作为构造参数部署`ProxyEasy`:`forge create --rpc-url $LOCAL_RPC_URL --constructor-args $ERR_NUM --private-key $LOCAL_ACCOUNT src/EasyProxy/ProxyError.sol:ProxyEasy`
1. 将上述部署地址保存到`ERR_PROXY`

使用`cast`对其进行测试，使用下述命令调用`setNumber()`函数:
```bash
cast send --private-key $LOCAL_ACCOUNT $ERR_PROXY "setNumber(uint256)" 100
```

按我们的预期使用命令`cast call $ERR_PROXY "number()(uint256)"`应得到的输出为`100`，但如果运行后会发现此处出现报错，原因为`number`的值为空。这十分令人疑惑，我们使用命令`cast call $ERR_PROXY "otherContractAddress()(address)"`获取代理合约所代理的合约地址时，输出为`0x0000000000000000000000000000000000000000000000000000000000000064`。我们发现此处的输出正是`100`(16进制转10进制后的结果)。

对于下述原因的讨论，我主要参考了[Openzeppelin Blog](https://blog.openzeppelin.com/proxy-patterns/)，如想更加详细知道此处的原理，请读者自行阅读[Understanding Ethereum Smart Contract Storage](https://programtheblockchain.com/posts/2018/03/09/understanding-ethereum-smart-contract-storage/)。

首先，我们给出代理合约与逻辑合约在运行前的存储分布表:
| 存储槽 | 代理合约| 逻辑合约 |
|---------|---------------|-----------|
| 0       | 逻辑合约地址`otherContractAddress` | 变量`number` |

当我们调用代理合约时，代理合约使用`delegatecall`调用逻辑合约。由于`delegatecall`调用的运行环境是发起合约，所以逻辑合约运行后直接进行了就地修改，将运行环境(代理合约)中的第一个状态变量存储槽(slot 0)修改为100，但此存储槽实际上是状态变量`otherContractAddress`值，即逻辑合约地址存储的地方，导致了存储冲突，逻辑合约直接覆写了代理合约的中最重要的变量——逻辑合约地址变量。简单而言，运行时，逻辑合约会直接按照自己运行前的存储表向代理合约内存储变量而不考虑代理合约自身的存储情况。

> 由于下文会出现很多类似的存储分布表，在此简述一种简单的判断是否存在存储冲突的方法: 如果两表在同一行内都存在值，则会冲突。

显然这种内存冲库最为简单的解决方案是修改逻辑合约使其不再`slot 0`中填入数据。如下表:
| 存储槽 | 代理合约 | 逻辑合约 |
|--------|---------|----------|
| 0       | 逻辑合约地址`otherContractAddress` |                 |
| 1       |                        | 变量`number` |

最简单的解决方案是设计一个`ProxyStorage`合约，存储逻辑合约地址等信息，逻辑合约和代理合约均继承`ProxyStorage`合约。由于逻辑合约继承了`ProxyStorage`合约，`ProxyStorage`定义的各个变量不会被逻辑合约覆写，解决了存储冲突问题。此处，我们没有讨论代理合约，这是因为代理合约不是真正导致存储冲突的原因，在具体实现过程中，我们不会在继承`ProxyStorage`后的代理合约内增加变量(即使布置了也会因为存储冲突被覆写)。

**在代理合约中最重要的变量就是`逻辑合约地址`**，这个事实非常重要，在后文中，我们讨论的代理合约的出发点就是如何保存代理合约内的逻辑合约地址。

> 在上文中，逻辑合约为什么会在代理合约内存储修改数据？
>
> 这一问题其实很好解决，读者可自行查阅`delegatecall`函数的功能。该函数使用的运行环境正是发起`delegatecall`方法的合约。可以参考下图:

![Delegatecall Picture](https://img.wang.232232.xyz/img/2022/07/20/delegatecallc19b967e1be09876.png)

*此图来自[Solidity极简入门: 23. Delegatecall](https://github.com/AmazingAng/WTFSolidity/tree/main/23_Delegatecall)

### 合约编写测试

经过上文的讨论，我们知道编写一个能发挥正常作业的智能合约需要继承关系，在`src/EasyProxy/ProxyEasy.sol`中写入以下代码:
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

contract ProxyStorage {
    address public otherContractAddress;

    function setOtherAddressStorage(address _otherContract) internal {
        otherContractAddress = _otherContract;
    }
}

contract DataLayout is ProxyStorage {
    uint256 public number;
}

contract NumberStorage is DataLayout { 
    function setNumber(uint256 _uint) public {
        number = _uint;
    }

    function getNumber() public view returns (uint256) {
        return number;
    }
}

contract NumberStorageUp is DataLayout {
    function setNumber(uint256 _uint) public {
        number = _uint;
    }

    function getNumber() public view returns (uint256) {
        return number;
    }

    function addNumber() public {
        number += 1;
    }
}

contract ProxyEasy is ProxyStorage {
    constructor(address _otherContract) {
        setOtherAddress(_otherContract);
    }

    function setOtherAddress(address _otherContract) public {
        super.setOtherAddressStorage(_otherContract);
    }

    fallback() external {
        address _impl = otherContractAddress;

        assembly {
            let ptr := mload(0x40)
            calldatacopy(ptr, 0, calldatasize())
            let result := delegatecall(gas(), _impl, ptr, calldatasize(), 0, 0)
            let size := returndatasize()
            returndatacopy(ptr, 0, size)

            switch result
            case 0 {
                revert(ptr, size)
            }
            default {
                return(ptr, size)
            }
        }
    }
}

```

注意，在编写代码中，我们需要先思考在代理合约内存储那些信息，为了减少代码长度，我只想在代理合约内存储逻辑合约地址这一个变量，当然你可以思考加入版本号等变量。然后我们在智能合约中首先声明`ProxyStorage`合约，并在合约内声明需要代理的逻辑合约的地址。编写完此合约后，我们编写逻数据存储合约(`DataLayout`)，这是为了保证我们在后续合约升级时不发生变量冲突问题。编写完此合约后我们可以编写与`DataLayout`对应的逻辑合约。最后声明代理合约，该合约也需要注意继承问题，以及需要对`ProxyStorage`中的变量全部进行初始化。总体而言，该合约并不复杂。

如果你需要编写更加复杂的合约，你必须保持合约中的变量排列顺序。根据上文给出的原理，一旦合约内的变量顺序改变，则会导致存储槽之间的冲突，进一步导致变量覆写的出现。可以使用继承现有逻辑数据存储合约的方式保证合约变量实例化属性不会改变。加入你需要加入更多的变量，你需要将变量声明全部放在一个新的合约内，在此处我们称为`NewDataLayout`，最终形成的继承关系如下图:

![Inherit DataLayout](https://s-bj-3358-blog.oss.dogecdn.com/svg/mermaid-diagram-2022-07-23-172842.svg)

这是一种较为优雅的解决方案，可以有效避免读者在编写升级合约时发生变量存储冲突的情况。

> 上述内容，简而言之就是严格实现逻辑与数据存储相分离的原则。数据存储层之间层层继承避免数据存储冲突。逻辑合约由于其多变性则不需要遵循层层继承的原则。

当然，在此处我们没有使用新的变量，我们只编写了不增加变量的升级合约`NumberStorageUp`，相较于原合约仅增加了`addNumber()`函数。

接下来，我们需要编写测试函数用于测试代理合约(test/EasyProxy/ProxyEasy.t.sol)是否成功:
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../../src/EasyProxy/ProxyEasy.sol";

contract ContractTest is Test {
    NumberStorage private numberStorage;
    NumberStorageUp private numberStorageUp;
    ProxyEasy private proxy;

    function setUp() public {
        numberStorage = new NumberStorage();
        numberStorageUp = new NumberStorageUp();
        address numberAddress = address(numberStorage);
        proxy = new ProxyEasy(numberAddress);
    }

    function testGetNumber() public {
        (bool ok,) = address(proxy).call(
            abi.encodeWithSignature("setNumber(uint256)", 100)
        );
        if (!ok) {
            revert("DelegateCall Error");
        }

        (bool success, bytes memory data) = address(proxy).call(
            abi.encodeWithSignature("getNumber()")
        );

        if (!success) {
            revert("DelegateCall Error");
        }
        uint256 returnNumber = abi.decode(data, (uint256));
        assertEq(returnNumber, 100);
    }

    function testNumberUpgrade() public {
        (bool ok,) = address(proxy).call(
            abi.encodeWithSignature("setNumber(uint256)", 100)
        );
        if (!ok) {
            revert("DelegateCall Error");
        }

        proxy.setOtherAddress(address(numberStorageUp));

        (bool addSuccess, ) = address(proxy).call(
            abi.encodeWithSignature("addNumber()")
        );

        if (!addSuccess) {
            revert("Add Fail");
        }

        (bool success, bytes memory data) = address(proxy).call(
            abi.encodeWithSignature("getNumber()")
        );

        if (!success) {
            revert("DelegateCall Error");
        }
        uint256 returnNumber = abi.decode(data, (uint256));
        assertEq(returnNumber, 101);
    }
}
```

此处为了避免编译无法通过的问题(如果直接调用使用Proxy.getNumber函数，会出现没有定义的错误而导致编译失败)，此处大量使用了`call`等底层函数。使用`forge test`启动测试，发现测试通过，即该合约可以起到作用。

### 合约部署

由于部署涉及到三个不同的合约及地址问题，如果使用常规的`cast create`命令则过于冗杂。在此处，我们考虑使用更加人性化的`script`的部署方法。在`script/EasyProxy/ProxyEasy.sol`写入下述内容:

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Script.sol";
import "../../src/EasyProxy/ProxyEasy.sol";

contract MyScript is Script {
    function run() external {
        vm.startBroadcast();

        NumberStorage numberStorage = new NumberStorage();
        NumberStorageUp numberStorageUp = new NumberStorageUp();
        address numberAddress = address(numberStorage);
        ProxyEasy proxy = new ProxyEasy(numberAddress);

        vm.stopBroadcast();
    }
}
```

该内容基本与测试脚本中的`setUp`一致，对于部署脚本而言，每进行一次`new`操作，则进行一次部署。该脚本的很多内容都已在上文和[上一篇博客](https://blog.wongssh.cf/2022/07/14/foundry-with-erc20/)中进行了详细叙述，在此不再解释具体原理。

比较有意思的是，在部署过程中，出现了两个合约同时部署在同一个区块的情况，与我们之前的测试场景有所不同请注意。合约地址出现的顺序与代码顺序一致。

![MoreContractScriptDeploy.png](https://img.wang.232232.xyz/img/2022/07/21/MoreContractScriptDeploy23da0bb826de77e7.png)

从上至下，依次为`NumberStorage`、`NumberStorageUp`、`ProxyEasy`合约。

### 补充

上述方法仅是一种简化方式，没有考虑很多问题，如果你想获得生产环境级的智能合约代码，可以参考一下这个[仓库](https://github.com/OpenZeppelin/openzeppelin-labs/tree/master/upgradeability_using_inherited_storage)

上述方法一般被称为`Inherited Storage`，除此方法外还存在`Eternal Storage`和`Unstructured Storage`两种方法。前者的思路为使用在上文提到的`EternalStorage`存储代理合约地址，而后者使用了`solidity assembly`将代理合约地址存放到距离`slot 0`较远的存储槽。如果想深入了解这些方法，可以参考[Openzeppelin Proxy Patterns Blog](https://blog.openzeppelin.com/proxy-patterns/)

随着以太坊的进一步发展，`Unstructured Storage`标准化为了下文中所分析的`EIP-1822`标准。

该代理合约已被`OpenZeppelin`提供了相关的包，可以前往[此处](https://docs.openzeppelin.com/contracts/4.x/api/proxy#Proxy)查看文档

## EIP-1822 UUPS

此标准其实是对上一节最后所提到的`Unstructured Storage`方法的标准化。该方法的核心仍是使用汇编`delegatecall`实现委托代理，但是在存储变量方面，此方法没有使用代理合约与逻辑合约继承同一合约以避免逻辑合约地址存储冲突的解决方案，而是对数据冲突的关键——代理合约中的地址存储问题进行了处理，将逻辑合约的地址存储到了一个距0地址非常远的随机地址中，避免了逻辑合约写入变量覆盖地址的问题。如下述内存表:

| 存储槽 | 代理合约 | 逻辑合约 |
|--------|---------|----------|
| 0 | | 代理合约的第1个变量 |
| 1 | | 代理合约的第2个变量 |
| ... | | 代理合约的变量或空 |
| 0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7 | 逻辑合约地址 | 几乎不可能存储值 |

通过上述存储表，我们可以明显看出逻辑合约在运行时可以向小于`0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7`的任一存储槽写入数据而不导致存储冲突。

### 核心实现

在此处，我们给出该思路的核心部分:
```solidity
contract Proxy {
    constructor(bytes memory constructData, address contractLogic) public {
        assembly {
            sstore(0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7, contractLogic)
        }
        (bool success, bytes memory _ ) = contractLogic.delegatecall(constructData);
        require(success, "Construction failed");
    }

    fallback() external payable {
        assembly {
            let contractLogic := sload(0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7)
            calldatacopy(0x0, 0x0, calldatasize)
            let success := delegatecall(sub(gas, 10000), contractLogic, 0x0, calldatasize, 0, 0)
            let retSz := returndatasize
            returndatacopy(0, 0, retSz)
            switch success
            case 0 {
                revert(0, retSz)
            }
            default {
                return(0, retSz)
            }
        }
    }
}
```

构造函数接受两个参数，`constructData`和`contractLogic`。前者是逻辑合约的构造器函数的选择器，后者是逻辑合约的地址。`assembly`部分使用了`sstore`实现了将地址存储到`0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7`地址的功能。然后使用`delegatecall`函数调用`constructData`，即逻辑合约的构造函数实现对逻辑合约构造。

> 为什么此处不直接在逻辑合约内使用`constructor`直接初始化？
> 在逻辑合约会与代理合约组合后，逻辑合约在进行初始化才是有意义的。因为`delegatecall`函数的特点是逻辑合约在代理合约内环境中存储变量，这意味部署代理合约前的在逻辑合约内的操作都是无效的。我们在部署时进行的初始化无法影响到组合后的代理合约，即意味着初始化的无效。

对于另一核心部分`fallback()`函数，我们在上文已经进行了大篇幅的阐述，此处不再给出具体分析。

除了上述代理合约，在`EIP-1822`中还有另一个标准合约`Proxiable Contract`，代码如下:
```solidity
contract Proxiable {
    // Code position in storage is keccak256("PROXIABLE") = "0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7"

    function updateCodeAddress(address newAddress) internal {
        require(
            bytes32(
                0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7
            ) == Proxiable(newAddress).proxiableUUID(),
            "Not compatible"
        );
        assembly {
            sstore(
                0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7,
                newAddress
            )
        }
    }

    function proxiableUUID() public pure returns (bytes32) {
        return
            0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7;
    }
}
```

根据标准，所有基于`EIP-1822`的合约都应该实习此合约，此合约的目的是保存向后兼容。合约内容非常简单，在`updateCodeAddress(address)`升级函数中要求新合约的`proxiableUUID`为`0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7`。部分读者可能对`Proxiable(newAddress)`该语法表示疑惑，编译器认为此语法是一种显性类型转换，即将地址转换为合约。我查阅了很多文档，并未提及存在此方法，如果读者对此有更好地解释，请向我发送[邮件](mailto:wongshouhao@outlook.com)。

该合约的作用体现在长时间升级更新项目过程中，可能出现了`EIP-1822`升级的情况，由于升级可能导致`0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7`该存储槽地址改变，通过此合约内的逻辑可以拒绝升级避免错误的发生。

> 为什么升级会改变存储槽的地址?
>
> 从`Proxiable`合约的注释中，我们就可以知道`0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7`其实是`keccak256("PROXIABLE")`的值。在`EIP-1822`的[说明](https://eips.ethereum.org/EIPS/eip-1822)中，编写者给出了一个例子，存储槽的地址可能会改变为类似`keccak256("PROXIABLE-ERC1822-v1")`的值。

### 工具合约

由于在`EIP-1822`给出了一些安全提示，在此处我们也实现一部分安全提示中的合约。这一部分给出的合约和安全提示也适用于上一节`EIP-897`。

首先是最重要的`owner`合约，任何在逻辑合约中可能导致严重破环的函数均应该使用`owner`合约中的`onlyOwner`修饰器进行修饰。下文给出的代码仅是最简实现，如果要在正式环境下使用请引用`openzeppelin-contracts/contracts/access/Ownable.sol`。我们的最简`owner`合约如下:
```solidity
contract Owned {
    address owner;

    function setOwner(address _owner) internal {
        owner = _owner;
    }

    modifier onlyOwner() {
        require(
            msg.sender == owner,
            "Only owner is allowed to perform this action"
        );
        _;
    }
}
```

合约结构仍是一贯的简单，其中`modifier`是在本系列教程中第一次出现，使用`modifier`定义的`onlyOwner()`函数可以用来修饰正常函数。一般来说，其中`require`的用法为`require(condition, message);`，第一个参数为应符合的条件，第二个参数为如不符合第一个条件则弹出的错误信息。此处要求`msg.sender`必须为`owner`，如不符则给出`Only owner is allowed to perform this action`的错误信息。该`require`也可以更改为`require(msg.sender == owner)`语句，但此时不会给出错误信息。`_`代表被修饰函数的逻辑，即在运行被修饰函数的逻辑之前，必须通过`require`的检查。

其次，我们实现一个设计极其巧妙的`LibraryLock`合约，该合约的作用是在逻辑合约代理成功后锁定合约避免被用户使用`call`调用。合约攻击者可能使用`selfdestruct`进行销毁逻辑合约，这会导致代理合约无法正常的工作。将`call`锁定则不会受其影响。在`EIP-1822`的标准文档中认为使用`LibraryLock`合约可以避免一些问题，如`SELFDESTRUCT`、`CALLCODE`等，我没有找到一些更加详细的文档解释这些问题。由于此合约设计比较巧妙，所以在此处我们仍进行实现，代码如下:
```
contract LibraryLockDataLayout {
    bool public initialized = false;
}

contract LibraryLock is LibraryLockDataLayout {
    modifier delegatedOnly() {
        require(
            initialized == true,
            "The library is locked. No direct 'call' is allowed"
        );
        _;
    }

    function initialize() internal {
        initialized = true;
    }
}
```

我们首先声明了`LibraryLockDataLayout`作为下述`LibraryLock`的数据存储层。先创建数据存储合约，再编写数据处理合约继承数据存储合约是一个非常好的编码习惯，这样可以保证在未来升级合约时不会出现变量覆盖等问题。在此处，`LibraryLockDataLayout`仅声明了`initialized`为初始化变量。后文用于处理该变量的合约为`LibraryLock`，该合约的首先声明了`delegatedOnly()`修饰器，主要用来修饰避免被直接`call`的函数，后由声明了`initialized`函数，该函数将`initialized`修改为`true`。当然，该函数为`internal`属性避免被外界调用而导致函数锁定。

此合约的功能的实现完全依赖于`delegatecall`在代理合约环境下运行的特性。在代理合约部署中，代理合约调用逻辑合约中的构造器函数，构造器函数中存在的`initialize()`被调用，合约在代理合约内将`initialized`修改为`true`，使所有被锁定的函数正常使用。但如果你直接`call`逻辑合约，`call`合约会在逻辑合约内进行运行，而逻辑合约内的`initialized`仍为 `false`导致函数依旧被锁定。上述代码生动展示了`call`与`delegatecall`的运行环境的不同。

我们要实现的最后一个工具合约是需要视场景而定的，即逻辑合约数据存储合约。我们在此节将假设一种场景，我们设计了一个ERC20代币合约，由于考虑到合约升级使用了`EIP-1822`，合约发行后过于受欢迎而导致初始发币量不足，所以使用代理合约的升级功能提高合约发现量。在此处，我们简化ERC20合约，仅考虑数量关系。显然，在设计合约时，我们需要使用至少两个变量，我们使用`DataLayout`合约存储，正如上文所述，数据管理层合约需要层层代理避免出现变量重排冲突的现象。所以合约代码如下:
```solidity
contract DataLayout is LibraryLockDataLayout {
    uint256 public totalSupply;
    uint256 public supplyAmount;
}
```

此合约较为简单，不再详细叙述。

### 总体实现

由于总体合约的代码较长，请前往本教程的[github仓库](https://github.com/wangshouh/upgradeContractLearn/blob/master/src/EIP-1822/EIP1822.sol)获取代码。

在此我们给出合约的继承关系图:

![contract-inherit](https://s-bj-3358-blog.oss.dogecdn.com/svg/mermaid-diagram-2022-07-22-204139.svg)

上图中，给出`LibraryLockDataLayout`与`LibraryLock`都是用于数据存储的合约，`LibraryLock`和`NumberStorage`都是依赖于数据存储合约的数据处理合约。而`Owned`和`Proxiable`则是纯粹的工具合约，在合约升级时不需要考虑。

在我们代码中，`NumberStorageUp`合约是对`NumberStorage`合约最简单的升级，因为升级过程不涉及变量变化，所以我们只需要更改`NumberStorage`的逻辑即可。继承关系图如下:

![NumberStorageUp](https://s-bj-3358-blog.oss.dogecdn.com/svg/mermaid-diagram-2022-07-22-205153.svg)

假设你的`NumberStorage`升级过程中需要增加新的变量，你需要将新增加的变量定义在一个新的合约中，在此处我们将这个存储新变量的合约称为`NewDataLayout`，我们需要该新合约继承过去的存储合约。最终形成的继承图如下:

![Add Variable](https://s-bj-3358-blog.oss.dogecdn.com/svg/mermaid-diagram-2022-07-22-205730.svg)

在此处给出一个`NewDataLayout`的示例:
```solidity
contract NewDataLayout is DataLayout {
    uint256 public x;
}
```

使用这种继承关系是为了保证变量的排序在升级过程中不会发生改变。

### 合约测试

对于此合约进行的测试较为冗长，所以在此处我们不再展示不重要的测试代码，你可以在[Github仓库](https://github.com/wangshouh/upgradeContractLearn/blob/master/test/EIP1822/EIP1822.t.sol)里找到完整的测试代码。

测试代码的中的`setUp()`函数较为新颖，在此处给出代码:
```solidity
function setUp() public {
    number = new NumberStorage();
    numberUp = new NumberStorageUp();
    address numberAddress = address(number);
    proxy = new Proxy(
        abi.encodeWithSignature("constructor1()"),
        numberAddress
    );
}
```

相比于`EIP-897`的`setUp()`函数，此处增加了一个`abi.encodeWithSignature("constructor1()")`，如果你阅读过仓库中完整的`NumberStorage`的代码，你就会知道`constructor1()`是`NumberStorage`的构造函数。正如上文所述，我们在进行代理时将逻辑合约的构造函数传入完成逻辑合约初始化和逻辑合约代理的双重任务。

测试合约中的`testInit()`优化了上文`EIP-897`的部分代码，在此给出代码:
```solidity
function testInit() public {
    (bool initCall, bytes memory initSupply) = address(proxy).call(
        abi.encodeWithSignature("totalSupply()")
    );

    require(initCall, "Init test call Error");

    uint256 returnNumber = abi.decode(initSupply, (uint256));
    assertEq(returnNumber, 1000);
}
```
在此处我们获取`totalSupply`的值时直接使用了`totalSupply()`的函数，这是`public`变量的隐含方法，即可以通过`变量名()`的函数访问变量的值，所以在合约内编写`get`函数对`public`变量而言没有意义。同时，此测试代码还使用了上文刚刚给出的`require`方法优化了错误抛出的代码量，不再需要编写`if`判断。

测试合约中的`testAddNumber()`较为常规，用来测试`addNumber(uint256))`函数是否正常运行。在使用`call`调用合约时需要将合约所需变量类型填入括号我们已经阐述过多遍。

测试合约中的`testFailOverMax()`用来测试`totalSupply`变量是否发挥作用。函数解析请参见[上一篇博客](https://blog.wongssh.cf/2022/07/14/foundry-with-erc20/)中的“编写测试脚本”一节中的`testFailMaxsupplyUseCheat`函数的解析。

测试合约中的`testFailDirectCall()`用来逻辑合约部署后是否可以直接调用。

测试合约中的`testContractUpgradeGet()`用来测试升级后的合约中的`supplyAmount`的值是否丢失。

测试合约中的`testFailUpgradeByOwner()`用来测试如果不是合约创建者是否可以升级合约。具体解析请参考[上一篇博客](https://blog.wongssh.cf/2022/07/14/foundry-with-erc20/)中的“编写测试脚本”一节中的`testWithdrawalFailsAsNotOwner()`函数的解析。

### Debug

在上文中，我们一直没有给出如果出现测试错误如何进行调试，在此处我们给出一个极其简单的`debug`方法。在编写完智能合约后，我们应该首先保证合约没有语法错误，`vscode`的`solidity`插件可以找到大部分的语法错误和编译错误。在编写合约过程中，我们在易出现错误的地方插入`require`等语句抛出异常。正如我们在`EIP-1822`实现代码中展示的那样。

在保证合约没有错误后，我们应编写一系列的测试代码，调用函数并检测函数的输出。此过程主要检查逻辑错误，也是错误最易发生的阶段，我建议使用`forge test -vvv`及以上的测试输出。`-vvv`会在测试出现错误后给出测试错误函数的栈调用情况。如果使用`-vvvv`则会展示所有测试函数的栈调用情况，无论对错。我们在下文中展示了在`EIP-1822`中最为复杂的栈调用情况:

![ForgeTest](https://pic.rmb.bdstatic.com/bjh/5d743d3ed6aec9aa7725362088717822.png)

该栈调用对应的代码如下:
```solidity
function testFailOverMax() public {
    uint256 slot = stdstore
        .target(address(proxy))
        .sig("supplyAmount()")
        .find();
    bytes32 loc = bytes32(slot);
    bytes32 mockedsupplyAmount = bytes32(abi.encode(10_000));
    vm.store(address(proxy), loc, mockedsupplyAmount);
    (bool addCall, ) = address(proxy).call(
        abi.encodeWithSignature("addNumber(uint256)", 100)
    );

    require(addCall, "Add Error");
}
```

首先我们就看到了大量的以`VM:`前缀的调用，这些调用是我们使用`forge-std`标准库中的测试函数的结果。即下述代码:
```solidity
uint256 slot = stdstore
    .target(address(proxy))
    .sig("supplyAmount()")
    .find();
bytes32 loc = bytes32(slot);
bytes32 mockedsupplyAmount = bytes32(abi.encode(10_000));
vm.store(address(proxy), loc, mockedsupplyAmount);
```

鉴于篇幅有限，我们在此处不对此栈调用进行完整讨论，只介绍与我们内容较为相关的部分。

```bash
    ├─ emit SlotFound(who: Proxy: [0xefc56627233b02ea95bae7e19f648d7dcd5bb132], fsig: 0x6f3d8fe8, keysHash: 0x290decd9548b62a8d60345a988386fc84ba6bc95484008f6362f93160ef3e563, slot: 2)
```

上述调用显示了我们在上文给出的存储表的正确性。`SlotFound`输出中明确给出了`supplyAmount`变量所在的`slot`为2而且该变量其实在`proxy`的空间内。在此也给出正确的`Proxy`合约内的存储表:
| 存储槽 | 变量名 |
| ------ | ----- |
| 0 | initialized |
| 1 | totalSupply |
| 2 | supplyAmount |
| ... | ... |
| 0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7 | contractLogic(逻辑合约地址) |

下述输出也比较重要:
```bash
    ├─ [0] VM::store(Proxy: [0xefc56627233b02ea95bae7e19f648d7dcd5bb132], 0x0000000000000000000000000000000000000000000000000000000000000002, 0x0000000000000000000000000000000000000000000000000000000000002710) 
    │   └─ ← ()
```

`VM:store()`中的第一个参数为合约地址，第二个参数为存储槽地址(16进制表示的`2`)，第三个参数为在存储槽内存储的变量值(16进制表示的`10000`)。

下述输出为真正的合约调用过程:
```bash
    ├─ [5160] Proxy::addNumber(100) 
    │   ├─ [4825] NumberStorage::addNumber(100) [delegatecall]
    │   │   └─ ← "Greater than the maximum supply"
    │   └─ ← "Greater than the maximum supply"
    └─ ← "Add Error"
```

输出显示了`Proxy`中的`addNumber`函数通过`delegatecall`调用`NumberStorage`中的`addNumber`的过程。`←`显示了调用输出的结果，在此处为抛出的异常。

如果你是想理解函数调用逻辑和代码底层的开发者，我们建议使用`forge test -vvvv`命令，对所有的测试函数给出栈调用结果并对其进行分析。这样可以更加直观的理解EVM底层和函数调用的逻辑。

## 总结

通过这一篇博客相信读者已经对合约代理的基本形式和本质有了了解。合约代理的核心方法就是`fallback`函数和`delegatecall`方法，而在合约代理过程中最大的问题是存储冲突。我们在本文中分别介绍了使用继承和使用存储槽避免存储冲突的方法。这就是代理的核心所在，无论在后文提出怎样的花哨方法，其本质就是解决存储冲突问题。

当然在读完本文后，希望读者在合约测试和调试方面有所收获，本文的核心并不在合约的编写，我准备在未来的写作计划中加入智能合约实战的内容，在哪里读者可以获得一些复杂合约的编写经验。在本文的下篇中，我依旧使用最为简单的`NumberStorage`和`NumberStorageUp`合约作为基础合约。

本来想在一篇文章内完成对所有可升级合约标准的解释。但写到这里已经到了1万多字，所以在此处我决定将本教程分为上下两篇。下篇文章将介绍以下标准:

- EIP-1967
- EIP-1538
- EIP-2535

下篇将采用与上篇不太相同的写作方式。原因是上篇所介绍的代理方法都是较为早期的以太坊标准而且考虑到读者对于智能合约代理可能并不熟悉，所以上篇尽可能使用了从头编写的方法，而在下篇中的标准都比较新而且有非常完善的`openzeppelin`库的支持，下篇主要采用解释`openzeppelin`库的方法进行讲解，尽可能使代码符合生产标准。
