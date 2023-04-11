---
title: "Foundry教程：使用多种方式编写可升级的智能合约(下)"
date: 2022-07-30T21:27:17Z
tags: [EIP-1967,EIP-2535,solidity]
aliases: ["/2022/07/24/foundry-contract-upgrade-part2"]
---
## 概述

正如我们在[上篇博客](https://blog.wongssh.cf/2022/07/18/foundry-contract-upgrade-part1/)结尾时所述，本文主要依靠`openzeppelin`库介绍代理合约的编写。

本文主要介绍的代理类型如下:

- EIP-1967
- EIP-2535

由于本文依赖于`Openzeppelin/openzeppelin-contracts`进行介绍EIP标准，所以请读者使用以下命令在项目内安装对应的库:

```bash
forge install Openzeppelin/openzeppelin-contracts
```

你可以在[github仓库](https://github.com/wangshouh/upgradeContractLearn)内找到本文所使用的全部代码。

因为此文主要使用`openzeppelin`编写智能合约，建议阅读以下文章:

- [Writing Upgradeable Contracts](https://docs.openzeppelin.com/upgrades-plugins/1.x/writing-upgradeable)
- [Proxy Upgrade Pattern](https://docs.openzeppelin.com/upgrades-plugins/1.x/proxies)

在`openzeppelin`的文档中，一般称逻辑合约的英文为`implementation contract`而不是`logic contract`

## EIP-1967

本合约是上篇介绍的`EIP-1822 UUPS`的进一步标准化版本，读者可以在这里找到[ERC文档](https://eips.ethereum.org/EIPS/eip-1967)。该标准被`etherscan`等区块链浏览器支持，可以提供完整的代理合约展示和交互功能。你可以前往[USDC合约](https://etherscan.io/address/0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48)查看情况。如下图:

![etherscanproxy.png](https://pic.rmb.bdstatic.com/bjh/e84cde8fe8d8d14dd1ea65a540cb07d7.png)

### 基本标准

此标准文档与`EIP-1822`文档大有不同。由于`EIP-1822`较为古老，在其文档中仍存在大量的解释性内容由于解释合约运行的原理。但在`EIP-1967`中，由于其创建时间较晚，合约代理的基本原理已被智能合约开发者所熟知，所以在`EIP-1967`的文档中没有介绍代理合约的基本原理，主要是对存储槽、事件进行了标准化和解释。在本节内容中，我们将介绍`EIP-1822`的基本标准和指定这些标准的原因。

首先被定义的就是逻辑合约(Logic Contract)的地址，在`EIP-1822`中我们一般采用`keccak256("PROXIABLE")`值，即`0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7`，该值其实可以有开发者自行决定。但在`EIP-1967`中，为了方便区块链浏览器的访问，该地址被标准化为`keccak256('eip1967.proxy.implementation') - 1`，即`0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc`。

你可以在`lib/openzeppelin-contracts/contracts/proxy/ERC1967/ERC1967Upgrade.sol`第21行中找到此地址槽的定义:
```
bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
```

在此处我们选择`keccak256('eip1967.proxy.implementation') - 1`的原因是为了避免潜在的攻击。如果想阅读以下的讨论，请先行阅读[Understanding Ethereum Smart Contract Storage](https://programtheblockchain.com/posts/2018/03/09/understanding-ethereum-smart-contract-storage/)中的`mapping`部分。

为了方便读者理解以下内容，我将进行一次对某智能合约的假想攻击。假设当前存在一份使用`keccak256("PROXIABLE")`的值作为逻辑合约存储槽的代理合约。且作为攻击者的我们通过阅读逻辑合约源代码发现在逻辑合约内存在`mapping`数据结构。通过阅读[Understanding Ethereum Smart Contract Storage](https://programtheblockchain.com/posts/2018/03/09/understanding-ethereum-smart-contract-storage/)，我们已知在合约内`mapping`数据结构存储在`keccak256(key, slot)`的地址内，且`key`和`slot`拼接方式已知。显然，我们可以阅读代理合约的代码或存储的状态变量得到`slot`的值，一般而言我们也可以通过交互合约操作`key`的值。如果满足上述条件，我们可以构造一种特殊的`key`和`slot`的拼接使其值等于`PROXIABLE`实现将逻辑合约存储槽写入特定的`value`。在代理合约内一般仅存在`fallback`等函数，一旦逻辑合约地址被改变，则资金无法转移。这是极其严重的事故。但在`EIP-1967`中。使用了`keccak256('eip1967.proxy.implementation') - 1`导致无法在简单地使用`mapping`的`keccak256(key, slot)`存储槽进行占用。除非你可以将`keccak256('eip1967.proxy.implementation') - 1`转换为`keccak256(x)`的形式。但基于哈希函数的不可逆性，我们无法计算出`x`的值，导致无法构造攻击用的`(slot, key)`。

当然，该标准与`EIP-1822`仍存在一点不同，就是逻辑合约(Logic Contract)的地址可以为空，但前提是存在信标代理(Beacon contract)的地址存储槽不为空。我们将在下一段介绍信标代理。

同时，标准也规定了每次升级合约应给出`Upgraded`的`event`。见`lib/openzeppelin-contracts/contracts/proxy/ERC1967/ERC1967Upgrade.sol`第33行:
```solidity
event Upgraded(address indexed implementation);
```
此处`indexed`的属性用于检索`event`日志，由于在此处我们仅涉及合约编写，所以我们不在此阐述其作用。

`EIP-1967`也加入了一个我们过去没有给出的合约类型——信标代理(Beacon Contract)。ERC标准规定信标代理的地址存储在`bytes32(uint256(keccak256('eip1967.proxy.beacon')) - 1)`中，其值为`0xa3f0ad74e5423aebfd80d3ef4346578335a9a72aeaee59ff6cb3582b35133d50`。

我们可以在`lib/openzeppelin-contracts/contracts/proxy/ERC1967/ERC1967Upgrade.sol`中的第142行查到以下代码:
```solidity
bytes32 internal constant _BEACON_SLOT = 0xa3f0ad74e5423aebfd80d3ef4346578335a9a72aeaee59ff6cb3582b35133d50;
```

信标代理的作用是同一逻辑合约可以实现多个代理合约共同代理。在这里给出一种情况，如果你开发了一个NFT发布合约希望为以尝鲜为目的的客户服务。但为了体现项目的区块链属性，你决定让用户可以获得智能合约地址等信息。如果使用一般的架构，你需要为每一个用户部署一份相同NFT合约。这将消耗大量的`gas`费，而且这些部署的NFT合约逻辑完全类似。作为开发者的我们可以考虑使用信标代理的架构，即开发一个通用逻辑的NFT合约，使用信标代理架构为其实现多个代理合约。这样部署NFT合约的费用降低为了部署一个逻辑简单的代理合约的`gas`费。当然如果你的项目中存在高净值用户需要复杂的NFT逻辑，你可以为其进行单独部署合约，然后改变代理合约内的地址存储槽内的信息实现合约升级。未来，此项目可能作为我们的实战内容出现在我的博客中。

上述方案可以称为`BeaconProxy`，此方法的基本原理是修改逻辑合约地址的获得。在以往的模式中，我们将逻辑合约地址存储在代理合约内部，但在`BeaconProxy`方案中，我们将逻辑合约地址单独放置在一个智能合约(下称此合约为存储合约)内，要求代理合约在每次调用逻辑合约时先去读取存储合约内的逻辑合约地址。下面给出`test/EIP1967/EIP1967.t.sol`中`testInit()`栈调用:
```bash
Traces:
  [20126] ContractTest::testInit() 
    ├─ [13293] BeaconProxy::name() 
    │   ├─ [2308] UpgradeableBeacon::implementation() [staticcall]
    │   │   └─ ← NFTImplementation: [0xce71065d4017f316ec606fe4422e11eb2c47c246]
    │   ├─ [3191] NFTImplementation::name() [delegatecall]
    │   │   └─ ← "TEST"
    │   └─ ← "TEST"
    └─ ← ()
```

为了方便读者与常规调用进行对比，我们在此给出`test/EIP1822/EIP1822.t.sol`中`testInit()`:
```bash
Traces:
  [12752] ContractTest::testInit() 
    ├─ [7131] Proxy::totalSupply() 
    │   ├─ [2318] NumberStorage::totalSupply() [delegatecall]
    │   │   └─ ← 1000
    │   └─ ← 1000
    └─ ← ()
```

由给出的栈调用，我们可以明显看到`BeaconProxy`在调用真正的合约`NFTImplementation`前调用了`UpgradeableBeacon`获取了合约地址，而在常规方法中，则是直接调用了指定的逻辑合约`NumberStorage`，而没有在调用逻辑合约前进行获取地址的操作。当我们需要升级智能合约时，我们首先升级逻辑合约，再升级存储合约，这样依赖于存储合约的所有代理合约都可以同步升级。

当然，正如升级合约会触发事件，升级信标代理也会触发以下事件(可见`lib/openzeppelin-contracts/contracts/proxy/ERC1967/ERC1967Upgrade.sol`第147行):
```solidity
event BeaconUpgraded(address indexed beacon);
```

同时，也规定了信标代理内必须存在以下函数(在`lib/openzeppelin-contracts/contracts/proxy/beacon/IBeacon.sol`实现了该接口):
```solidity
function implementation() returns (address)
```

> 接口中不对函数进行定义，仅会指明该函数的存在，而由继承该接口的合约实现。当我们需要调用其他合约时，可以选择仅导入对方合约的接口，避免合约体积增大而导致gas费上升，具体的实战案例可以参考[WTF solidity](https://github.com/AmazingAng/WTFSolidity/tree/main/14_Interface#%E6%8E%A5%E5%8F%A3)。在此合约中具体实现可以参考`lib/openzeppelin-contracts/contracts/proxy/beacon/UpgradeableBeacon.sol`第35行。

`EIP-1967`也规定了合约拥有者的地址存储位置(Admin address)，该存储操位于`bytes32(uint256(keccak256('eip1967.proxy.admin')) - 1)`，即`0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103`，可在`lib/openzeppelin-contracts/contracts/proxy/ERC1967/ERC1967Upgrade.sol`的第106行找到定义。同时也规定了改变此存储槽中的内容必须触发下述事件:
```solidity
event AdminChanged(address previousAdmin, address newAdmin);
```
在`lib/openzeppelin-contracts/contracts/proxy/ERC1967/ERC1967Upgrade.sol`第111行进行了定义。

最终我们对上文内容进行总结。下表为`EIP-1967`规定的存储槽列表:
| 存储槽位置 | 存储槽名称 | 作用 |
| --------- | ---------- | ---- |
| bytes32(uint256(keccak256('eip1967.proxy.implementation')) - 1) |  Logic contract address | 存储逻辑合约地址 |
| bytes32(uint256(keccak256('eip1967.proxy.beacon')) - 1) | Beacon contract address | 存储信标代理合约地址 |
| bytes32(uint256(keccak256('eip1967.proxy.admin')) - 1) |  Admin address | 存储代理合约拥有者的地址 |

下表为`EIP-1967`规定的事件列表:
| 事件名称 | 事件代码 | 触发条件 |
| ------- | ------- | --------- |
| Upgraded | `event Upgraded(address indexed implementation);` | 逻辑合约地址升级 |
| BeaconUpgraded | `event BeaconUpgraded(address indexed beacon);` | 信标代理合约升级 |
| AdminChanged | `event AdminChanged(address previousAdmin, address newAdmin);` | 合约拥有者改变 |

### Openzeppelin架构

代理合约的基础架构如下:

![ProxySystem](https://s-bj-3358-blog.oss.dogecdn.com/svg/mermaid-diagram-2022-07-25-195542.svg)

此图过于简单，我们在此列出UML图:

![Contract UML](https://s-bj-3358-blog.oss.dogecdn.com/svg/mermaid-diagram-2022-07-25-210049.svg)

在UML图中，以`#`开头的函数代表此函数仅能在合约内调用`internal`; `+`开头的函数或变量代表`public`; `-`开头的函数或变量代表`private`，即不能在合约外调用; 斜体函数名为抽象函数，即在当前合约内仅注明了函数名，我们需要在继承合约内实现。注意此图中省略了部分函数，如果想获得详细信息，请查阅[文档](https://docs.openzeppelin.com/contracts/4.x/api/proxy)。

我们首先介绍`Proxy`合约，该合约的主体部分是`fallback()`函数，使用了下述代码:
```solidity
function _fallback() internal virtual {
    _beforeFallback();
    _delegate(_implementation());
}
```
其中`beforeFallback()`代表在合约`delegatecall`进行前执行的函数，如果没有特殊需求，可以不使用此函数。而`delegatecall()`函数的代码如下:
```solidity
function _delegate(address implementation) internal virtual {
    assembly {
        calldatacopy(0, 0, calldatasize())
        let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)
        returndatacopy(0, 0, returndatasize())

        switch result
        case 0 {
            revert(0, returndatasize())
        }
        default {
            return(0, returndatasize())
        }
    }
}
```
上述代码我们已经多次使用过，但如果你仔细对比，你会发现部分地址从`0x40`被替换为了`0`，两者基本等效，我们可以从[Layout in Memory](https://docs.soliditylang.org/en/v0.8.15/internals/layout_in_memory.html)查阅到以下内容:
```
0x00 - 0x3f (64 bytes): scratch space for hashing methods
0x40 - 0x5f (32 bytes): currently allocated memory size (aka. free memory pointer)
```

`0`属于暂存空间的一部分，而我们直接常用的`0x40`则指向当前为空的内存地址。在一般情况下，`0x40`指向的空间就是以`0`开始的内存块，所以我们可以简单地认为`0`与`0x40`是等价的，但需要注意这种等价是不可靠的，只有在合约刚刚启动时才会存在。

同时我们发现了另一个与我们之前编写的`fallback`函数不同的是，在`openzeppelin`中使用了参数`implementation`，该参数来自`_implementation()`函数。而后者为抽象函数，我们需要进行在继承合约内进行实现。在UML图中，我们可以看出`ERC1967Proxy`和`BeaconProxy`都实现了此函数，但两者的实现方法不同，正是这种不同使`BeaconProxy`可以实现通过一次调用实现所有代理合约的升级。

在`ERC1967Proxy`中的实现`implementation`如下:
```solidity
function _implementation() internal view virtual override returns (address impl) {
    return ERC1967Upgrade._getImplementation();
}

//_getImplementation()来自`ERC1967Upgrade.sol`
function _getImplementation() internal view returns (address) {
    return StorageSlot.getAddressSlot(_IMPLEMENTATION_SLOT).value;
}
```

为了方便读者阅读，我们将位于`ERC1967Upgrade`中的`_implementation()`函数也一并列出。`StorageSlot`来自`utils/StorageSlot.sol`，该合约中的函数主要作用是用常规的函数调用代替了`assembly`，你可以查阅源代码。简单分析以下就可以得到以下结论: `ERC1967Proxy`采用了直接在地址槽内读取逻辑合约地址的方法获取逻辑合约地址，这与`ERC-1822`是一致的。

在`BeaconProxy`中的实现`implementation`如下:
```solidity
function _implementation() internal view virtual override returns (address) {
    return IBeacon(_getBeacon()).implementation();
}

//下列函数来自`ERC1967Upgrade.sol`
function _getBeacon() internal view returns (address) {
    return StorageSlot.getAddressSlot(_BEACON_SLOT).value;
}
```
这里出现了一个奇特合约名称`IBeacon`，这是一个接口合约，接口合约仅对函数名称进行定义而不对函数体进行定义，接口合约可以用于合约的远程调用，具体可参考[WTF solidity](https://github.com/AmazingAng/WTFSolidity/tree/main/14_Interface#%E6%8E%A5%E5%8F%A3)。该合约主体部分如下:
```solidity
interface IBeacon {
    function implementation() external view returns (address);
}
```
在`openzeppelin`下有对此接口的进行实现的合约`UpgradeableBeacon.sol`，其中`implementation()`函数如下:
```solidity
function implementation() public view virtual override returns (address) {
    return _implementation;
}
```
该函数过于简单，作用就是返回函数内存储的`_implementation`的值。`virtual`关键词声明此函数在可以在子合约可以被重写，`override`关键词说明此函数重写了其母合约的`virtual`函数。两者出现在一起没有问题，说明此合约重写了母合约中的函数，而且允许我们通过继承此合约重写此函数。

通过上述一系列操作，最终的效果就是当代理合约进行`delegatecall`操作时，合约会调用`_implementation()`函数获取逻辑合约地址。`_implementation()`函数在`ERC1967Upgrade`进行了实现，该实现要求逻辑合约。

![Beacon FlowChart](https://s-bj-3358-blog.oss.dogecdn.com/svg/mermaid-diagram-2022-07-26-103729.svg)

当然上述流程仅给出了获取逻辑合约地址的流程，其他流程与我们之前熟悉的`EIP-1822`相同。

接下来，我们分析`ERC1967Proxy`和`BeaconProxy`的构造器，了解构造器对于我们进行合约部署是必要的。

首先给出`ERC1967Proxy`的构造器，如下:
```solidity
constructor(address _logic, bytes memory _data) payable {
    _upgradeToAndCall(_logic, _data, false);
}

//下列函数来自`ERC1967Upgrade.sol`
function _upgradeToAndCall(
    address newImplementation,
    bytes memory data,
    bool forceCall
) internal {
    _upgradeTo(newImplementation);
    if (data.length > 0 || forceCall) {
        Address.functionDelegateCall(newImplementation, data);
    }
}
```

为方便分析，我们也将母合约中的函数一并给出。与我们之前熟悉的`ERC1822`的构造器基本类似，要求我们输入逻辑合约地址和使用abi编码调用的函数名。在部署时，基本与之前给出`ERC1822`类似，在下文介绍部署时，我们不会再讨论`ERC1967Proxy`的部署。

给出`BeaconProxy`的构造函数:
```solidity
constructor(address beacon, bytes memory data) payable {
    _upgradeBeaconToAndCall(beacon, data, false);
}

//下列函数来自`ERC1967Upgrade.sol`
function _upgradeBeaconToAndCall(
    address newBeacon,
    bytes memory data,
    bool forceCall
) internal {
    _setBeacon(newBeacon);
    emit BeaconUpgraded(newBeacon);
    if (data.length > 0 || forceCall) {
        Address.functionDelegateCall(IBeacon(newBeacon).implementation(), data);
    }
}
```

在此合约内，我们可以看到与`ERC1967Proxy`结构基本类似，但是要求`beacon`的地址。`beacon`合约必须是实现`IBeacon`接口的合约，在日常使用中，我们可以直接使用`UpgradeableBeacon`合约或者个人编写的继承合约。此类合约的部署比较复杂，我们将在下文详细叙述并给出实战说明如何进行`beacon`合约的部署。

我们也给出`ERC1967Upgrade`中的其他函数作用:

- `_get`类函数，注意这是一类函数，包括`_getImplementation()`、`_getAdmin()`、`_getBeacon()`函数，用于获取各个存储槽内的数据类型，建议继承后设置`public`函数;
- `_set`类函数，这也是一类函数，主要包括`_setImplementation`等，用于直接改变存储槽内的数据，不建议使用;
- `_upgradeToAndCallUUPS`函数，用于`EIP1822`函数的升级，要求原`EIP1822`合约内存在`proxiableUUID()`函数，我们之前编写的`EIP1822`合约是符合定义的

注意上述函数都为`internal`属性，只能在合约或子合约内调用，如果你想部署的代理合约可以调用这些函数，你需要通过继承`ERC1967Proxy`或`BeaconProxy`合约编写`public`函数实现在部署后的调用。

### 合约编写

经过上述讨论，我们已经大致知道了`openzeppelin`的整体架构，接下来我们进行智能合约的代码编写。由于`EIP-1967`的正常模式，即直接将逻辑合约地址存储到对应的存储槽中的模式与`EIP-1822`类似，我们在此不进行详细讨论。我们主要介绍`BeaconProxy`模式。

首先，因为`BeaconProxy`和`ERC1967Upgrade`合约内有大量的函数属于`internal`类型，我们没有办法进行外部调用，所以我们需要继承`BeaconProxy`合约编写函数。具体代码(`src/EIP-1967/proxy.sol`)如下:
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "openzeppelin-contracts/contracts/proxy/beacon/BeaconProxy.sol";

contract NFTProxy is BeaconProxy {
    constructor(address beacon, bytes memory data) BeaconProxy(beacon, data) {
        _changeAdmin(msg.sender);
    }

    modifier OnlyContractOwner() {
        require(msg.sender == _getAdmin(), "Not Contract Owner");
        _;
    }

    function getAdmin() public view {
        _getAdmin();
    }

    function changeAdmin(address _newAdmin) public OnlyContractOwner {
        _changeAdmin(_newAdmin);
    }

    function upgradeProxy(address newBeacon, bytes memory data)
        public
        OnlyContractOwner
    {
        _upgradeBeaconToAndCall(newBeacon, data, false);
    }
}
```

首先编写`constructor`，在原构造器的基础上增加了`_changeAdmin(msg.sender);`函数。该函数在构造器内运行后实现了初始化时的`owner`定义，在此处我们简单的将其定义为合约创建者。

`OnlyContractOwner()`修饰函数的作用是保证合约内部分函数仅能由合约拥有者调用。关于`modifier`函数的具体说明，我们在上篇已经进行了介绍。

`getAdmin()`和`changeAdmin()`函数都比较简单，在此不给出具体说明。

`upgradeProxy`该函数为此合约内的一个核心函数，用于逻辑合约升级，该函数的主要部分是从`ERC1967Upgrade`中继承的`_upgradeBeaconToAndCall`函数，此函数的说明已在上文给出。

对于用于存储逻辑合约地址的`UpgradeableBeacon`合约，审查源代码可以发现`openzeppelin`给出了我们所需要的所有功能，所以我们不需要编写新合约进行继承。

除了上述代理合约体系所需要的工具合约外，我们也需要自行编写逻辑合约，此次使用的逻辑合约较为简单，你可以在`src/EIP-1967/NFTImplementation.sol`。此合约的具体代码如下:
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "openzeppelin-contracts/contracts/proxy/utils/Initializable.sol";

contract NFTData {
    string public name;
    uint256 public currentTokenId;
    uint256 public totalSupply;
}

contract NFTImplementation is NFTData, Initializable {
    function initialize(
        string memory _name,
        uint256 _totalSupply
    ) public initializer {
        name = _name;
        totalSupply = _totalSupply;
    }

    function mint() public returns (uint256) {
        require(currentTokenId+1 < totalSupply, "Over Max");
        currentTokenId += 1;
        return currentTokenId;
    }
}

contract NFTImplementationUp is NFTImplementation {
    function burn() public returns (uint256) {
        currentTokenId -= 1;
        return currentTokenId;
    }
}
```

与我们之前编写的逻辑合约类似，此合约也选择了数据与逻辑相分离的设计思路，我们将合约所需要的数据存储在`NFTData`，具体操作数据的逻辑则放在`NFTImplementation`和`NFTImplementationUp`中，这种设计方便我们后期进行升级。与之前不同的是初始化函数`initialize`函数使用了`initializer`修饰符。该修饰符由`Initializable`合约提供，可以有效避免初始化中的一系列问题，如多次初始化等。在`BeaconProxy`体系中，我们不建议在后续升级中重新进行初始化。重新进行初始化会导致`BeaconProxy`升级出现问题，我们不能仅通过改变`UpgradeableBeacon`中存储的逻辑合约地址实现升级，而需要调用代理合约内的`upgradeProxy`函数，这显然与我们的需要不同。

以上就是代理合约、信标合约和逻辑合约的所有内容，我们接下来会介绍合约部署和测试的相关情况。

### 合约测试

由于`BeaconProxy`的复杂性，合约测试有近100行，由于篇幅限制，我们不会在此处给出所有的合约测试函数的分析，你可以在[这里](https://github.com/wangshouh/upgradeContractLearn/blob/master/test/EIP1967/EIP1967.t.sol)找到完整代码。

我们给出`setUp()`函数的分析，`setUp()`函数作为初始化函数，其中的内容主要涉及合约部署部分，具体代码如下:
```solidity
function setUp() public {
    NFT = new NFTImplementation();
    NFTUp = new NFTImplementationUp();
    upgrade = new UpgradeableBeacon(address(NFT));
    proxy = new NFTProxy(
        address(upgrade),
        abi.encodeWithSignature("initialize(string,uint256)", "TEST", 1000)
    );
}
```

大部份代码逻辑与我们熟悉的`ERC1822`的初始化相同，但此处值得注意的是在初始化`NFTProxy`，我们使用的不是逻辑合约的地址而是存储逻辑合约的`UpgradeableBeacon`合约的地址，这也是实现一次升级所有代理合约升级的基石。

其次，对于大家比较重要的测试是测试一次升级是否可以实现多个代理合约的直接升级，我们给出对此测试的代码:
```solidity
function testMultiProxy() public {
    NFTProxy proxyNext = new NFTProxy(
        address(upgrade),
        abi.encodeWithSignature("initialize(string,uint256)", "TEST2", 1000)
    );
    uint256 ProxyNextBeforeMint = ProxyMint(address(proxyNext));       
    assertEq(ProxyNextBeforeMint, 1); 
    uint256 ProxyBeforeMint = ProxyMint(address(proxy));
    assertEq(ProxyBeforeMint, 1);

    upgrade.upgradeTo(address(NFTUp));

    uint256 ProxyNextBurnId = ProxyBurn(address(proxyNext));
    assertEq(ProxyNextBurnId, 0);  
    uint256 ProxyBurnId = ProxyBurn(address(proxy));
    assertEq(ProxyBurnId, 0); 
}
```

由于逻辑合约升级后增加了`burn`函数，所以此处我们在合约升级前调用`mint`铸造而在合约升级后使用`burn`销毁。这样既测试了合约升级后数据是否会丢失，也测试了逻辑合约是否真正升级成功。注意此测试函数中出现的`ProxyMint`和`ProxyBurn`都是我自行编写的，具体实现可以参考源代码，这里个函数作用就是提供代理合约调用对应的逻辑合约内的函数。

在此测试函数内实现逻辑合约升级的为`upgrade.upgradeTo(address(NFTUp));`，通过此行代码，我们对`UpgradeableBeacon`中存储逻辑合约的状态变量进行了升级，进而升级了两份代理合约。

## EIP-2535💎

`EIP-2535`标准就是著名的钻石模型。钻石模型的基本功能如下:

- 在合约中原子性的增加、减少或替换函数
- 将合约内函数的增加、减少、替换通过约定的`event`给出
- 通过提供合约查询公开函数的信息
- 解决以太坊合约最大24KB的限制
- 允许可升级函数在未来更改为不可升级函数

此模型与我们之前介绍的`EIP-1967`等传统代理模型不同，此模型没有采用无序存储合约地址的方法，而是在通过映射约定不同的函数和对应的合约地址，此方法属于有序存储。有序存储的特点使代理合约可以实现一个合约对应多个逻辑合约。当然，此模型也是使用`delegatecall`完成函数调用，合约数据仅保存在代理合约内。

![facetFunction.jpeg](https://pic.rmb.bdstatic.com/bjh/dd906c6d1bc27c12ade78ce05e6fefaa.jpeg)

上图中给出了一个示意图，其中的切面(`Facet`)实际就是代理合约所调用的逻辑合约，此图中展示了三个逻辑合约。

由于此方法暂时没有`openzeppelin`的完整实现，我们需要自己编写合约的大部分内容，但我们仍会引入部分`openzeppelin`库以减少代码编写量。

### 标准内容

本节内容参考了[Introduction to EIP-2535 Diamonds](https://eip2535diamonds.substack.com/p/introduction-to-the-diamond-standard)和[EIP-2535标准](https://eips.ethereum.org/EIPS/eip-2535)

在编写真正的代码前，我们需要了解`EIP-2535`标准的内容。本文主要介绍核心部分，即数据存储和函数部分。

#### 数据存储

我们之前一直强调代理合约的关键在于逻辑合约地址存储和合约数据存储，在此之前我们介绍了通过继承解决合约存储问题和通过随机地址槽存储数据解决数据冲突问题。在`EIP2535`中，由于涉及多个逻辑合约及其对应的状态变量，我们需要一些更加复杂的机制解决此类数据冲突问题，但基本思路与我们之前介绍的方法相同。

由于本节内容仍涉及`solidity`底层数据的存储方式，请先阅读[Understanding Ethereum Smart Contract Storage](https://programtheblockchain.com/posts/2018/03/09/understanding-ethereum-smart-contract-storage/)

第一种方法是继承存储(`Inherited Storage`)，顾名思义就是通过继承解决数据存储冲突问题。这种方法我们在上文进行过实践，可以参考本文上篇的`EIP-897 Proxy`节中的`原理实现`。该方案最大的优点就是简单易用而且在业界也有广泛的使用。

但也具有以下缺点:

1. 严重影响合约复用，你无法将当前合约部署到任何一个其他项目中。

1. 使用钻石模型的合约一般都非常庞大，存储合约可能存在几百个状态变量，在编写合约时我们需要继承存储合约，这意味着我们必须小心翼翼地避免合约内的函数或本地变量与之前定义的状态函数名不同。我们使用的常规编辑器不会提示所有命名冲突问题，这意味我们需要自己查找项目中是否存在可能的命名冲突，这是麻烦的。

继承方案影响合约复用的原因在于继承子合约在编译时会加入继承母合约的代码。在继承存储方案中，逻辑合约在编译后会加入存储合约中的代码。这意味你将逻辑合约进行复用时就会在你原本的代理合约中加入一系列无用的状态变量，导致代理合约内数据混乱。下表给出了一种可能的情形:

| 地址槽 | 代理合约 | 继承存储合约的逻辑合约   |
|:---:|:-----------:|:------------:|
| 0   | Test 1      | 原存储合约内的第1个变量 |
| 1   | Test 2      | 原存储合约内的第2个变量 |
| 2   |             | 原存储合约内的第3个变量 |
| 3   |             | 原存储合约内的第4个变量 |

此情况假设原本代理合约内存在`Test 1`和`Test 2`两个变量; 继承存储合约的逻辑合约内有4个变量。一旦将继承存储合约的逻辑合约部署到代理合约中，就会发生严重的数据冲突问题。这导致了继承方案在合约复用性中存在严重缺陷。

第二种方法依旧是选择随机存储槽存储逻辑合约所需要的数据，此方案通常被称为`Diamond Storage`。与之前仅提供随机数据存储槽存储代理合约地址不同，在`EIP2535`中，我们需要为不同类型的逻辑合约设计存储地址以保证其数据存储不会与其他逻辑合约冲突。在具体实现上，通常采用`library`库合约实现存储，如下代码:

```solidity
library MyStructStorage {
  bytes32 constant MYSTRUCT_POSITION = 
    keccak256("com.mycompany.projectx.mystruct");

  struct MyStruct {
    uint var1;
    bytes var2;
    mapping (address => uint) var3;
  }

  function myStructStorage()
    internal 
    pure 
    returns (MyStruct storage mystruct) 
  {
    bytes32 position = MYSTRUCT_POSITION;
    assembly {
      mystruct.slot := position
    }
  }
}
```

与我们之前所常见的合约不同，上述代码描述了`solidity`中的库`library`。库是一种特殊的合约类型，库需要部署但只能通过`delegatecall`的方式调用且库合约不能存储状态变量。当然，与一般合约不同，库合约的调用是不需要使用`delegatecall`关键词的，如下调用库合约的代码:
```solidity
contract TestStruct {
    function myFunction(uint256 inputUint) external {
        MyStructStorage.MyStruct storage mystruct = MyStructStorage
            .myStructStorage();

        mystruct.var1 = inputUint;
    }
}
```
我们可以使用`library.function()`的形式直接调用库中的函数。上述调用库函数的代码含义为对`var3`进行赋值，你可以在`src/EIP-2535/storageLibraryTest.sol`找到完整代码。

我们给出的库合约首先声明了一个结构体`MyStruct`，在结构体内声明了我们需要到所有的变量。将所有变量声明在结构体内有利于我们在`myStructStorage()`直接操作结构体存储位置，也方便了后期在逻辑合约内直接一次性规定所有变量的存储位置。如果我们单独规定每一个变量，这意味着我们需要手动操作每一个变量的存储位置，这是很大的工作量。对于操作结构体的存储位置，在`solidity`中提供了`slot`属性进行直接调整，你可以在[官方文档](https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html#json-output)找到一些资料。在`myStructStorage()`函数中，我们使用`solidity inline assembly`的方法直接规定了结构体的开始存储槽位置。在此处我们规定合约结构体存储槽的开始位置为`keccak256("com.mycompany.projectx.mystruct")`，其中`com.mycompany.projectx.mystruct`可以自行规定。

在函数调用时，我们声明符合`MyStructStorage.MyStruct`形式的变量`mystruct`，并调用`MyStructStorage.myStructStorage()`对其进行赋值。`MyStructStorage.myStructStorage()`函数会返回一个存储在`keccak256("com.mycompany.projectx.mystruct")`位置的结构体。这与`EIP1967`对逻辑合约地址存储槽等进行定义类似，只不过此处定义了一个结构体。

上述描述可能过于抽象，我们可以使用`forge debug`进行具体的代码分析。`forge debug`会给出合约操作码、栈情况、内存情况和源代码四者之间的关系，在项目根目录终端中键入以下命令:
```bash
forge debug --debug src/EIP-2535/storageLibraryTest.sol:TestStruct --sig "myFunction(uint256)" 16
```
该命令会打开`debug`窗口，如下所示:

![debugInit.png](https://pic.rmb.bdstatic.com/bjh/c69d14f6ae70eae09824980b17c7bfdd.png)

自上而下依次为合约EVM操作码、EVM堆栈情况、EVM内存和合约源代码。我建议您读者进入此页面后单击`m`和`t`键会显示更多信息。使用鼠标滚动即可逐步运行EVM操作码，当然也可以`k`或`j`键。跟多关于键位操作可以参考[Foundry 官方文档](https://book.getfoundry.sh/forge/debugger#navigating)。如果你想查询所有的操作码含义可以参考[此网站](https://www.evm.codes/)。

> 如果你的终端中没有显示`contract call`，那说明你的代码可能被编译过，你需要在原代码中加入一些修改，如果增加空行，再次运行上述命令重新编译即可。

在`debug`中大量的操作码用于初始化合约，我们在此不对初始化操作码进行介绍。关键在于`039`和`05a`位置的操作码。

在`039`位置时的输出如下:

![039debug](https://pic.rmb.bdstatic.com/bjh/603d990a2b9b894a74b86643006504d4.png)

`039`位置的操作码将结构体存储的位置`75fd42e3768eaf1c351f0f1eee6ed52a2603059b48bfb1eee0baed20051c00ef`，即`keccak256("com.mycompany.projectx.mystruct")` 推入栈。

在`05a`位置时的输出如下:

![05adebug](https://pic.rmb.bdstatic.com/bjh/09a2c78d192e4f4bd6b9e86da7ce2349.png)

我们可以在栈中看到上一步压入的`75fd42e3768eaf1c351f0f1eee6ed52a2603059b48bfb1eee0baed20051c00ef`和`10`。后者就是我们指定的`inputUint`，我们在`debug`命令中将其设置为`16`(其16进制即为`10`)，可见我们的此处进行的数据存储是成功的。

上述给出的方法就是被称为`Diamond Storage`的存储方案，在下面常见的介绍钻石模型的图中，就使用了此方案。

![DiamondDiagram.png](https://pic.rmb.bdstatic.com/bjh/ec61bc64efc8d5b24ce86f3fbc459bfd.png)

该方案也是使用最广泛的方案。我们在后文进行代码实现时会采取这种存储方案。如果读者没有特殊需求，可以选择此方案。值得注意的是，选择此方案增加状态变量只能增加到结构体的最后，否则会因为数据排序错位而导致数据冲突。

此方案的问题在于我们每次进行数据调用都需要调用一次`MyStructStorage.MyStruct storage mystruct = MyStructStorage.myStructStorage();`代码获得`mystruct`对象再进行数据修改，这是麻烦和乏味的。

在阅读EVM操作码时，你可以需要以下工具:

- [keccak256 online](https://emn178.github.io/online-tools/keccak_256.html)
- [函数选择器生成](https://abi.hashex.org/)

第三种方案是对通过继承解决数据存储冲突问题方案的改进，我们通过合约的`internal`标识避免变量名冲突问题。

使用此方案有以下好处:

1. 合约可读性和组织性好
1. 获取数据更加方便，不需要每次书写`MyStructStorage.MyStruct storage mystruct = MyStructStorage.myStructStorage();`获取结构体
1. 可以与`Diamond Storage`方案一同使用
1. 在`gas`费用方面更有效率
1. 合约复用功能更好，不会出现数据冲突问题

使用此方案需要按照以下步骤:
第一步，在`AppStorage.sol`文件中把所有需要的存储变量到`AppStorage`结构体中，如下代码:
```solidity
struct AppStorage {
  uint256 secondVar;
  uint256 firstVar;
  uint256 lastVar;
}
```

第二步，在需要使用变量的切面函数使用`AppStorage internal s`声明结构体。如下代码:
```solidity
import "./AppStorage.sol"

contract StakingFacet {
  AppStorage internal s;

  function myFacetFunction() external {
    s.lastVar = s.firstVar + s.secondVar;
  }
}
```

经过以上两步就完成变量的存储和调用。如果读者后期需要升级合约，需要在`struct AppStorage`结构体后增加变量，不可以打乱变量排列顺序，这与继承存储方案是一致的。

我们可以混用`Diamond Storage`和`AppStorage`方案。但需要在`AppStorage.sol`加入以下代码:
```solidity
library LibAppStorage {
    function diamondStorage() internal pure returns (AppStorage storage ds) {
        assembly {
            ds.slot := 0
        }
    }
}
```
因为我们没有对`AppStorage`的地址进行设置，所以我们可以直接通过`ds.slot := 0`方法将其从`slot 0`处提取出来。

该方案在业界也有实际应用，你可以查看Aavegotchi Contracts的[LibAppStorage](https://github.com/aavegotchi/aavegotchi-contracts/blob/master/contracts/Aavegotchi/libraries/LibAppStorage.sol)

#### 函数部分

为了方便下文叙述，我们在此对所使用的名词进行解释:

1. 钻石合约(`Diamond`)，即直接与用户交互的代理合约，是`delegatecall`的发起者和状态变量的存储者
1. 切面合约(`Facet`)，即逻辑合约，用于编写操作状态变量的函数
1. 放大镜合约(`loupe`)。一个特殊的切面合约，用于返回钻石中各个切面合约的具体内容，包括切面合约地址、切面合约中的函数选择器等内容。这有一个具体的[实例](https://louper.dev/diamond/0x10e138877df69Ca44Fdc68655f86c88CDe142D7F)

在`EIP-2535`中，最重要的函数就是`diamondCut`，该函数的功能是增加、修改或替换钻石合约中函数和切面合约。如果钻石合约为不可变合约，则可以不实现此函数。该函数运行后会抛出`DiamondCut`事件。此函数的接口如下:
```solidity
interface IDiamondCut {
    enum FacetCutAction {Add, Replace, Remove}
    // Add=0, Replace=1, Remove=2

    struct FacetCut {
        address facetAddress;
        FacetCutAction action;
        bytes4[] functionSelectors;
    }

    /// @notice Add/replace/remove any number of functions and optionally execute
    ///         a function with delegatecall
    /// @param _diamondCut Contains the facet addresses and function selectors
    /// @param _init The address of the contract or facet to execute _calldata
    /// @param _calldata A function call, including function selector and arguments
    ///                  _calldata is executed with delegatecall on _init
    function diamondCut(
        FacetCut[] calldata _diamondCut,
        address _init,
        bytes calldata _calldata
    ) external;

    event DiamondCut(FacetCut[] _diamondCut, address _init, bytes _calldata);
}
```

### 标准实现

为了方便讨论，此处选择的代码来自由`EIP-2535`的提出者[Nick Mudge](https://github.com/mudgen)编写的`diamond-3`。下表为该作者编写过的三个不同版本的`diamond`合约的对比:

| Implementation | diamondCut<br>complexity | diamondCut<br>gas cost | loupe<br>complexity | loupe<br>gas cost |
| -------------- | ------------------------ | ---------------------- | ------------------- | ----------------- |
| diamond-1      | low                      | medium                 | medium              | high              |
| diamond-2      | high                     | low                    | high                | high              |
| diamond-3      | medium                   | high                   | low                 | low               |

*此图来自[这里](https://github.com/mudgen/diamond)

`diamond-3`的[原始仓库](https://github.com/mudgen/diamond-3)的实现使用了`pragma solidity ^0.7.6`，直接使用会出现版本报错。为了解决此问题和保持教程的一致性，我推荐继续使用[我的仓库](https://github.com/wangshouh/upgradeContractLearn)，下面的讨论也是基于我的仓库。

我们首先分析较为简单的钻石合约，该合约的实现位于`src/EIP-2535/Diamond.sol`。此合约完成了存储的初始化和委托转发功能。

由于委托转发或称代理功能是钻石合约最重要的功能，所以我们先解释回调函数。完整代码如下:
```solidity
fallback() external payable {
    LibDiamond.DiamondStorage storage ds;
    bytes32 position = LibDiamond.DIAMOND_STORAGE_POSITION;
    assembly {
        ds.slot := position
    }
    address facet = ds.selectorToFacetAndPosition[msg.sig].facetAddress;
    require(facet != address(0), "Diamond: Function does not exist");
    assembly {
        calldatacopy(0, 0, calldatasize())
        let result := delegatecall(gas(), facet, 0, calldatasize(), 0, 0)
        returndatacopy(0, 0, returndatasize())
        switch result
            case 0 {
                revert(0, returndatasize())
            }
            default {
                return(0, returndatasize())
            }
    }
}
```

与之前的代理合约相比，钻石合约首先通过下述代码获得钻石合约的设置数据:
```solidity
LibDiamond.DiamondStorage storage ds;
bytes32 position = LibDiamond.DIAMOND_STORAGE_POSITION;
assembly {
    ds.slot := position
}
```
钻石合约的设置数据包含函数与合约地址的对应关系，即映射`selectorToFacetAndPosition`。我们可以在`src/EIP-2535/libraries/LibDiamond.sol`中找到对此映射的定义。

数据读取过程为首先读取结构体的结构，通过此代码
`LibDiamond.DiamondStorage storage ds;`，
再读取结构体的位置，通过此代码`ds.slot := position`。数据类型与数据存储槽结合就可以完成数据的读取。

读取获得的数据包含函数与合约地址的对应关系，我们通过`address facet = ds.selectorToFacetAndPosition[msg.sig].facetAddress;`获得函数对应的切面合约地址。最后通过查找的切面合约地址进行`delegatecall`。

> `msg.sig`是用户与合约交互时发送的函数选择器。上篇已经介绍函数选择器方面的内容

除了此核心部分，我们可以看到钻石合约也通过`LibDiamond.setContractOwner(_args.owner);`规定了合约拥有者。 但仅实现了`ERC173`接口功能，完整的接口实现可参考`src/EIP-2535/facets/OwnershipFacet.sol`。

> `ERC173`功能可参考接口文件`src/EIP-2535/interfaces/IERC173.sol`

钻石合约也初始化了`ERC165`的内容。`ERC165`用于判断合约是否实现了某个接口，允许用户花费最多`3000 gas`调用`supportsInterface`函数获得合约是否支持某接口的信息。如果支持则返回`true`。对于`ERC165`的具体实现参考`src/EIP-2535/facets/DiamondLoupeFacet.sol`中的`supportsInterface`函数。钻石合约在构造器内已经声明了支持的部分接口。

> 对于`ERC165`和`ERC173`，我给出的信息不多，建议大家直接阅读文中给出的源代码和阅读ERC标准文档

由上文可知，在`EIP535`中最重要的函数就是`diamondCut`函数，我们在下文将着重介绍此函数。

在`diamond-3`的实现中，`diamondCut`等关键函数位于`src/EIP-2535/libraries/LibDiamond.sol`。此库中也包括上文提到的代理设置数据。其中较为重要的数据由以下两个:

一是`selectorToFacetAndPosition`，此映射的功能是在已知函数的前提下，寻找对应的合约地址，具体实现如下:
```solidity
mapping(bytes4 => FacetAddressAndPosition) selectorToFacetAndPosition
struct FacetAddressAndPosition {
    address facetAddress;
    uint16 functionSelectorPosition; // position in facetFunctionSelectors.functionSelectors array
}
```

> 此处给出的代码经过了调整，为了优化阅读体验，我们将`FacetAddressAndPosition`一并给出。

二是`facetFunctionSelectors`，此映射的功能是在已知地址的前提下，寻找地址内的对应函数，具体实现如下:
```solidity
mapping(address => FacetFunctionSelectors) facetFunctionSelectors;
struct FacetFunctionSelectors {
    bytes4[] functionSelectors;
    uint16 facetAddressPosition; // position of facetAddress in facetAddresses array
}
```

> 我们可以看到除了直接的对应关系，结构体内还加入了`functionSelectorPosition`和`facetAddressPosition`参数用于标识参数在集合中的位置。

三是`facetAddresses`，即切面合约地址集合。
```
address[] facetAddresses;
```

四是`FacetCutAction`，该参数是枚举类型，定义在`src/EIP-2535/interfaces/IDiamondCut.sol`中，具体的实现如下:
```solidity
enum FacetCutAction {Add, Replace, Remove}
```

有了以上参数，我们就可以分析具体的函数实现方式。在`diamondCut`中，根据`FacetCutAction`不同，分别使用了以下函数:

1. addFunctions
1. replaceFunctions
1. removeFunctions

`addFunctions`需要两个参数分别为:

1. _facetAddress 切面合约地址
1. _functionSelectors 需要增加的函数的集合

![Add function](https://s-bj-3358-blog.oss.dogecdn.com/svg/mermaid-diagram-2022-07-29-213039.svg)

上图给出了`addFunctions`的逻辑框架，但缺少了部分赋值和计算的细节。其中较难理解的是`functionSelectorPosition`和`facetAddressPosition`。前者是函数选择器在`cetFunctionSelectors.functionSelectors`中的位置; 后者是切面合约地址在`facetAddresses`中的地址。

> 这里的映射关系较为混乱，建议大家多读几遍源代码。对于函数存在两个存储变量，分别是`facetFunctionSelectors`中的`bytes4[] functionSelectors;`和`selectorToFacetAndPosition`，后者通过`FacetAddressAndPosition`中的`functionSelectorPosition`记录对应函数选择器在`functionSelectors`中的位置。

读者可能发现`enforceHasContractCode`函数中存在一个特殊的汇编命令`extcodesize`，该汇编命令由`eip-1052`规定，其功能为当地址为空或不存在时返回`0`值。在此处，我们使用此函数用于判断切面合约是否存在。

`removeFunctions`所需要的参数与`addFunctions`相同，该函数的核心是它迭代调用的另一个函数`removeFunction`。`removeFunction`的作用原理如下:

![removeFunction](https://s-bj-3358-blog.oss.dogecdn.com/svg/mermaid-diagram-2022-07-30-212724.svg)

总体而言，代码复杂的地方在于多映射关系之间的互相关系。我们首先通过`selectorToFacetAndPosition`获得需要删除的函数的位置，然后通过`facetFunctionSelectors`获得此地址下`functionSelectors`集合最后的索引位置。由于`solidity`没有提供按索引删除集合元素的功能，我们只能使用`pop`函数删除最后一个元素。如果需要删除的函数就在对应`functionSelectors`的最后，我们可以直接使用`pop`删除。如果不在最后，我们需要使用其他手段。在代码实现中，作者通过将原来的最后函数先提取出来，使用原最后一个函数覆盖需要替换的函数。这样的话需要删除的函数就被原最后一个函数覆盖了，就可以使用`pop`删除。

如果删除函数后切面合约内没有存在的函数时，我们就需要删除切面合约。删除过程与删除函数的过程基本类似，也是使用了覆盖的方法，我们在此不再赘述。

`removeFunctions`实际上可以认为是删除函数和增加函数的联合体。读者应该可以自行阅读并理解代码。

另一个比较重要的函数是`louper`函数，它的接口定义在`src/EIP-2535/interfaces/IDiamondLoupe.sol`，具体实现可以参考`src/EIP-2535/facets/DiamondLoupeFacet.sol`。`louper`定义了以下函数:

1. `facets()`，返回所有切面合约的地址和切面合约内存储的函数
1. `facetFunctionSelectors(address)`，返回特定切面合约中的函数
1. `facetAddresses()`，返回所有切面合约的地址
1. `facetAddress(bytes4)`，返回函数对应的切面
1. `supportsInterface(bytes4)`，返回合约是否支持某接口

### 合约编写

我们将在下文中，以`diamond-3`的代码为基础构建我们的钻石合约。首先，我们需要在合约中加入存储，在此处我们使用`AppStorage`方案进行存储。在`src/EIP-2535/libraries/LibAppStorage.sol`写入以下内容:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

struct AppStorage {
    string name;
    uint256 totalSupply;
    uint256 maxSupply;
}

library LibAppStorage {
    function diamondStorage() internal pure returns (AppStorage storage ds) {
        assembly {
            ds.slot := 0
        }
    }
}
```
其中`AppStorage`是核心组件，而使用`LibAppStorage`库是为了方便与使用`DiamondStorage`方案的合约一同使用，在此处，我们不会用到此函数。

接下来，我们需要修改钻石合约的部分内容，主要为数据初始化，此部分修改较少，不再解释，具体参考`src/EIP-2535/Diamond.sol`。

> 注意此处我把逻辑数据的初始化放在了钻石合约中，这种方式可能与完全解耦的思想有所违背，读者可以自行修改。在此处，这样进行初始化是没有问题的。此处将所有初始化数据放在一个结构体中是为了防止栈溢出报错。我们在之前已经提及`EVM`规定单一`solidity`函数中最多有6个变量，通过结构体的组合，我们可以绕过这一限制。

然后，我们编写切面合约`TestFacet`，具体代码参考`src/EIP-2535/facets/TestFacet.sol`，代码如下:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;
pragma experimental ABIEncoderV2;

import "../libraries/LibAppStorage.sol";

contract TestFacet {

    AppStorage internal s;
    
    function name() external view returns (string memory) {
        return s.name;
    }

    function totalSupply() external view returns (uint256) {
        return s.totalSupply;
    }

    function maxSupply() external view returns (uint256) {
        return s.maxSupply;
    }

    function setName(string memory _name) external {
        s.name = _name;
    } 
}
```
对于读者而言，此代码应该较好理解，我们在此处不再赘述。

综上所述，了解基本原理后，钻石模型的编写较为简单，我们可以在`diamond-3`的基础山快速构建我们需要的钻石合约。比较复杂的是钻石合约的测试和部署。

### 合约测试

此节代码可以参考`test/EIP2535/Diamond.t.sol`。

在代码开头声明协议和所需要的`solidity`的版本后，我们进行了一系列的导入:
```solidity
import "../../src/EIP-2535/interfaces/IDiamondCut.sol";
import "../../src/EIP-2535/interfaces/IDiamondLoupe.sol";
import "../../src/EIP-2535/interfaces/IERC173.sol";

import "forge-std/Test.sol";

import "../../src/EIP-2535/Diamond.sol";
import "../../src/EIP-2535/facets/TestFacet.sol";
import "../../src/EIP-2535/facets/DiamondCutFacet.sol";
import "../../src/EIP-2535/facets/DiamondLoupeFacet.sol";
import "../../src/EIP-2535/facets/OwnershipFacet.sol";
```

自上而下可以将导入的文件分为三部分:

1. 接口，因为此处的调用往往需要复杂的参数，我们选择使用接口而不是`call`的方式进行跨合约调用
1. `forge`标准库的测试合约
1. 切面合约

在`setUp()`函数中，我们需要将各个合约进行初始化，对于切面合约而言，初始化较为简单，只需要使用`new`关键词即可，如下:
```solidity
cutfacet = new DiamondCutFacet();
loupefacet = new DiamondLoupeFacet();
ownerfacet = new OwnershipFacet();
testfacet = new TestFacet();
```

而对于钻石合约而言，我们可以看到它的构造器如下:
```solidity
constructor(
    IDiamondCut.FacetCut[] memory _diamondCut,
    DiamondArgs memory _args
)
```
其中最为关键的参数为`IDiamondCut.FacetCut[]`，查看`IDiamondCut`接口获得`FacetCut`结构，如下:
```go
struct FacetCut {
    address facetAddress;
    FacetCutAction action;
    bytes4[] functionSelectors;
}
```
我们需要获得每个切面的地址和`functionSelectors`。前者可以通过`address()`函数获得; 后者是由切面内函数选择器组成的`array`，我们首先对器进行声明:
```solidity
bytes4[] memory cutFunctions = new bytes4[](1);
bytes4[] memory loupeFunctions = new bytes4[](4);
bytes4[] memory ownerFunctions = new bytes4[](2);
IDiamondCut.FacetCut[] memory _diamondCut = new IDiamondCut.FacetCut[](
    3
);
```
关于数组声明可以参考[官方文档](https://solidity-cn.readthedocs.io/zh/develop/types.html#index-16)或[WTFSolidity 第6讲](https://github.com/AmazingAng/WTFSolidity/tree/main/06_ArrayAndStruct)

接下来，我们需要进行赋值操作，一个完整的案例如下:

对于我们而言，关键在于获得函数选择器。一个完整的函数选择器是对函数名和参数类型组成的字符串进行`keccak256`哈希计算后取前八位获得，如我们在后文会用到的`transferOwnership`函数，此函数在`src/EIP-2535/interfaces/IERC173.sol`定义，定义如下:
```
function transferOwnership(address _newOwner) external;
```
其函数选择器字符串应为`transferOwnership(address)`，注意函数选择器字符串内不包括变量名`_newOwner`。获得函数选择器字符串后，我们可以在终端内允许以下命令:
```bash
cast sig "transferOwnership(address)"
```
输出结果为`0xf2fde38b`，这正是我们需要的。考虑到常用的函数是有限的，有以太坊开发者组建了一个可以根据函数选择器逆向选择器字符串的[网站](https://sig.eth.samczsun.com/)，我们可以在这个网站测试函数选择器输出是否正确:
![ethSigDb](https://img.gejiba.com/images/0491352301162d59c6f2908b4b6bd5e9.png)
显然，上述查询结果证明我们是正确的。当然，此功能也被集成到了`cast`命令中，读者可以运行以下命令:
```bash
cast 4byte 0xf2fde38b
```
输出为`transferOwnership(address)`。

我们在后文均采用此种手动获得函数选择器的方法。另一种方法是使用`Hardhat`编写脚本自动获得，由于我们此教程不涉及`Hardhat`，读者可以自行参考[Foundry-Hardhat-Diamonds](https://github.com/Timidan/Foundry-Hardhat-Diamonds)中的[genSelectors.js](https://github.com/Timidan/Foundry-Hardhat-Diamonds/blob/master/scripts/genSelectors.js)

对于读者而言，最难实现的函数应该是`diamondCut`的选择器，其定义如下:
```solidity
function diamondCut(
    FacetCut[] calldata _diamondCut,
    address _init,
    bytes calldata _calldata
) external;

struct FacetCut {
    address facetAddress;
    FacetCutAction action;
    bytes4[] functionSelectors;
}
```
其中包含结构体`FacetCut`。对于结构体在函数选择器字符串中，我们需要对其进行展开，根据`FacetCut`的定义，展开后的结构如下`(address,uint8,bytes4[])`。由于此处使用了`FacetCut[]`，故在`(address,uint8,bytes4[])`后需要增加`[]`，最终整体如下:
```bash
cast sig "diamondCut((address,uint8,bytes4[])[],address,bytes)"
```

> 此处将枚举类型`FacetCutAction`的类型定义为`uint8`，是因为两者可以隐式互相转换。

获得函数选择器后，我们可以轻松对`functionSelectors`进行赋值，一个完整的例子如下:
```solidity
cutFunctions[0] = bytes4(0x1f931c1c); //diamondCut((address,uint8,bytes4[])[],address,bytes)
_diamondCut[0] = (
    IDiamondCut.FacetCut({
        facetAddress: address(cutfacet),
        action: IDiamondCut.FacetCutAction.Add,
        functionSelectors: cutFunctions
    })
);
```

> 为了方便读者核对函数选择器，我将函数选择器的字符串以注释的形式进行了附注

在此处我们基本解决了`diamond`部署中最难的问题，所以我们不在给出`setUp()`函数中的其他部分，读者可自行查阅[仓库](https://github.com/wangshouh/upgradeContractLearn)

在合约测试中，为了优化合约测试代码数量，大量使用了接口进行函数调用，接口的使用方法如下:
```solidity
IERC173(address(diamond)).owner();
```
较为简单，不再详细说明。

### 部署与升级

由于此次合约部署较为复杂，我们使用`script`的方式进行合约部署，对于合约的部署而言，与测试中的`setUp`函数基本一致，读者可以自行查看`script/EIP-2535/Diamond.s.sol`中的`SetupScript`合约。编写完成后，使用以下命令进行部署:
```bash
source .env
forge script script/EIP-2535/Diamond.s.sol:SetupScript --private-key $LOCAL_ACCOUNT --broadcast --rpc-url http://127.0.0.1:8545
```
对于已部署的钻石合约进行`DiamondCut`操作，直接使用函数较难填写参数，理论上我们需要自行构造`calldata`,然后进行调用，较为复杂。如果选择部署合约后进行升级，这也不太合适。通过`script`操作，我们可以简化操作，具体代码如下:
```solidity
contract UpdateScript is Script {
    function run() external {
        vm.startBroadcast();
        TestFacet testfacet = new TestFacet();
        bytes4[] memory testFunctions = new bytes4[](3);
        IDiamondCut.FacetCut[] memory _testDiamondCut = new IDiamondCut.FacetCut[](1);

        testFunctions[0] = bytes4(0x06fdde03); //name
        testFunctions[1] = bytes4(0x18160ddd); //totalSupply
        testFunctions[2] = bytes4(0xd5abeb01); //maxSupply

        _testDiamondCut[0] = (
            IDiamondCut.FacetCut({
                facetAddress: address(testfacet),
                action: IDiamondCut.FacetCutAction.Add,
                functionSelectors: testFunctions
            })
        );
        
        IDiamondCut(address(0xCf7Ed3AccA5a467e9e704C703E8D87F634fB0Fc9)).diamondCut(
            _testDiamondCut,
            address(0x0),
            new bytes(0)
        );
        vm.stopBroadcast();
    }
}
```

上述代码可以实现对钻石合约进行`diamondCut`操作。使用以下命令启动:
```bash
forge script script/EIP-2535/Diamond.s.sol:UpdateScript --private-key $LOCAL_ACCOUNT --broadcast --rpc-url http://127.0.0.1:8545
```

> 此处`0xCf7Ed3AccA5a467e9e704C703E8D87F634fB0Fc9`为钻石合约地址，读者可以根据实际情况自行替换

## 总结

本篇文章以上下篇的形式介绍了以下标准:

- EIP-897 Proxy
- EIP-1822 UUPS
- EIP-1967
- EIP-2535

如果读者不需要部署非常大的合约，我建议使用`EIP-1967`作为首选。因为它的结构简单、易于部署、标准化程度高且有`etherscan`的支持。合约开发过程中，我们可以依靠`openzeppelin`的框架，对于开发者十分友好。而且还有`beacon`这种模型支持多代理开发，方便平台为客户提供标准化合约部署服务。

如果读者需要编写合约非常复杂，可以使用`EIP-2535`钻石模型，该模型显然非常适合大规模合约代理，但另一方面其部署难度非常高，代码编写具有一定的复杂性，存储模型不直观，较为考验开发者的开发能力。

本文没有涉及到全部的代理模型，且本文是以以太坊标准为主线展开。读者如果想进一步学习，`openzeppelin blog`中的[Contract Upgrade](https://blog.openzeppelin.com/the-state-of-smart-contract-upgrades/)是一份非常好的材料。本文可能会在后期增加补充文章，可以订阅本博客的[RSS](https://blog.wongssh.cf/atom.xml)。
