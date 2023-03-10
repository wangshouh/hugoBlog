---
title: "EVM底层探索:字节码级分析最小化代理标准EIP1167"
date: 2022-08-22T11:47:33Z
tags: [EIP-1167,solidity,EVM]
aliases: ["/2022/08/22/deep-in-eip1167"]
---
## 概述

本文主要介绍最小化代理合约`EIP1167`的相关内容。为了实现最小化，`EIP1167`使用了`bytecode`(字节码)作为主要编码方式，即直接使`EVM`汇编指令进行编写。本文将在`openzeppelin`提供的[合约](https://docs.openzeppelin.com/contracts/4.x/api/proxy#Clones)基础上，为读者逐个字节码的解析`EIP1167`，帮助读者理解`EVM`底层汇编和`EIP1167`的实现原理。

注意虽然`EIP1167`也实现了代理合约，但其不具有合约升级能力，如果你希望构造可升级合约，请阅读以下文章:

- [Foundry教程：使用多种方式编写可升级的智能合约(上)]({{<ref "foundry-contract-upgrade-part1" >}})
- [Foundry教程：使用多种方式编写可升级的智能合约(下)]({{<ref "foundry-contract-upgrade-part2" >}})

> 如果读者没有代理合约开发经验，也建议阅读上文获得一些关于代理合约的基本知识。

建议读者在阅读后文之前可以简单读一下[EIP1167标准](https://eips.ethereum.org/EIPS/eip-1167)。

## openzeppelin实现

我们在此处首先给出`openzeppelin`的合约[实现](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.7.3/contracts/proxy/Clones.sol#L25)，代码如下:
```solidity
function clone(address implementation) internal returns (address instance) {
    /// @solidity memory-safe-assembly
    assembly {
        let ptr := mload(0x40)
        mstore(ptr, 0x3d602d80600a3d3981f3363d3d373d3d3d363d73000000000000000000000000)
        mstore(add(ptr, 0x14), shl(0x60, implementation))
        mstore(add(ptr, 0x28), 0x5af43d82803e903d91602b57fd5bf30000000000000000000000000000000000)
        instance := create(0, ptr, 0x37)
    }
    require(instance != address(0), "ERC1167: create failed");
}
```

上述代码描述了代理合约生成的基本结构。我们采用从顶向下分析的方法，首先关注合约生成的核心代码`instance := create(0, ptr, 0x37)`。查阅`EVM`汇编[表格](https://www.evm.codes/#f0)，我们可以知道此函数接受三个变量，分别是:

- value, 传递给新合约的ETH(以`wei`计费)
- offset, 新合约代码在内存中的起始位置
- size, 新合约的代码长度

本质上来说，此函数实现获取内存中的合约代码并将其进行部署的功能。在此处，我们没有向新合约传递ETH，规定了新合约的代码在内存中的起始位置为`ptr`，长度为`0x37 byte`，即55 byte 或 110 个16进制数字。


我们可以断定以下代码的功能是构造新合约的字节码:
```
let ptr := mload(0x40)
mstore(ptr, 0x3d602d80600a3d3981f3363d3d373d3d3d363d73000000000000000000000000)
mstore(add(ptr, 0x14), shl(0x60, implementation))
mstore(add(ptr, 0x28), 0x5af43d82803e903d91602b57fd5bf30000000000000000000000000000000000)
```

正如前文所述，由于`EIP1167`完全使用字节码编程，而`solidity`对内存级控制并不擅长，所以我们在此处只能提供内联汇编实现代码构建。当然，对于一般的`solidity`合约，你可以参考[此文](https://github.com/AmazingAng/WTFSolidity/tree/main/24_Create)。

接下来，我们对字节码构造部分进行解释，注意在此节中，我们不会指明生成的字节码的作用，此部分内容位于下一节。

此处用到了`implementation`变量，即需要被代理合约的地址，我们在此处假设其值为`0xbebebebebebebebebebebebebebebebebebebebe`。

`let ptr := mload(0x40)`。此处对`ptr`的值进行初始化。初始化的方法是读取(使用[mload](https://www.evm.codes/#51)函数读取指定地址的值) 0x40 地址的值。此处使用`0x40`地址进行读取的原因是此地址内存储着空闲内存的起始位置。在此处举一个例子，如果你的合约已经把`0x60`前的内存都填满了，读取`0x40`位置时，会获得`0x61`这个值。使用`0x40`中存储的地址可以有效避免内存覆写冲突问题的出现。

> 实际上`mload(0x40)`返回的是内存目前的占用量，其等同于空闲内存的起始位置，具体可以参考[Layout in Memory](https://docs.soliditylang.org/en/latest/internals/layout_in_memory.html#layout-in-memory)

当我们获取到空闲内存的起始位置后，我们接下来就可以构造`EIP1167`合约的字节码。

代码中各个汇编函数的作用如下:

- `mstore(offset, value)`的作用为向指定内存地址内写入`value`数据。注意`offset`的单位为`byte`且`value`的长度必须为`32 byte`
- `add(a, b)`的作用为`a + b`
- `shl(shift, value)`的作用为将`value`左移`shift`个`bit`。注意单位为`bit`


综合以上内容，我们可以得到每行代码的具体作用。

`mstore(ptr, 0x3d602d80600a3d3981f3363d3d373d3d3d363d73000000000000000000000000)`，我们首先在`ptr`后插入了`0x3d602d80600a3d3981f3363d3d373d3d3d363d73000000000000000000000000`(32 byte)。

`mstore(add(ptr, 0x14), shl(0x60, implementation))`，我们首先将`implementation`的地址(20 byte)通过`shl`左移`0x60 bit`，即`12 byte`，形成`32 byte`的标准数据。得到标准数据后，我们将数据写入`ptr + 0x14`处，即`ptr`后`20 byte`(40个16进制数)，最终形成以下数据:
```
0x3d602d80600a3d3981f3363d3d373d3d3d363d73bebebebebebebebebebebebebebebebebebebebe000000000000000000000000
```
最后，我们通过`mstore(add(ptr, 0x28), 0x5af43d82803e903d91602b57fd5bf30000000000000000000000000000000000)`写入数据，形成以下数据:
```
0x3d602d80600a3d3981f3363d3d373d3d3d363d73bebebebebebebebebebebebebebebebebebebebe5af43d82803e903d91602b57fd5bf30000000000000000000000000000000000
```
根据上文给出的`create`的参数，我们发现部署合约时仅读取此字节码的前`0x37 byte`的数据，即使用以下数据构造合约:
```
0x3d602d80600a3d3981f3363d3d373d3d3d363d73bebebebebebebebebebebebebebebebebebebebe5af43d82803e903d91602b57fd5bf3
```
关于此此字节码的作用，我们会在下文进行解释。

上述流程可以用下图进行概括:
![EIP1167 Memory](https://img.gejiba.com/images/d2242462d9a2b0834ee49137ee92e4fa.png)

此图展示了上述汇编代码对内存的修改情况。其中最上方的`ptr`、`ptr + 0x14`、`ptr + 0x28`等值表示当前的内存地址，`0x14`等值的单位均为`byte`。

## 字节码解析

### 运行流程

在进行字节码分析前，我们需要在顶层理解`EIP1167`是如何运行的，其核心在于`delegatecall`的使用。

合约运行分为以下几个步骤:

1. 获得`calldata`，用户发送的`calldata`中包含需要调用的函数和对应的参数，我们需要获得`calldata`以便于后期进行转发。
1. 使用`delegatecall`发送`calldata`。合约在获得`calldata`后可以通过`delegatecall`进行委托调用，代理合约会把被代理合约内的代码拉取到本地输入`calldata`进行运行，并将结果保存到代理合约内。
1. 获得`delegatecall`返回的结果并并储存到内存中
1. 向用户返回结果或错误

以上就是`EIP1167`的运行流程，接下来我们会解释如何通过`0x3d602d80600a3d3981f3363d3d373d3d3d363d73bebebebebebebebebebebebebebebebebebebebe5af43d82803e903d91602b57fd5bf3`字节码实现这一功能。

### 初始化

我们将智能合约分为两部分，一部分是在创建合约时运行的代码，我们称为创建代码(creation code 或 Deploy code)，另一部分则是逻辑代码(runtime code)。

前者主要实现以下功能:

- 运行`constructor`构造器函数
- 进行合约变量初始化
- 将`runtime code`复制到内存中

一个比较好的类比是创建代码类似软件的安装包，它会根据用户的输入选择安装文件夹释放文件并进行软件的初始化。类比无法使我们接近本质，所以我们在此处给出`go-ethereum`的合约创建[源代码](https://sourcegraph.com/github.com/ethereum/go-ethereum@master/-/blob/core/vm/evm.go?L406):
```go
func (evm *EVM) create(caller ContractRef, codeAndHash *codeAndHash, gas uint64, value *big.Int, address common.Address, typ OpCode) ([]byte, common.Address, uint64, error) {
	// Depth check execution. Fail if we're trying to execute above the
	// limit.
	if evm.depth > int(params.CallCreateDepth) {
		return nil, common.Address{}, gas, ErrDepth
	}
	if !evm.Context.CanTransfer(evm.StateDB, caller.Address(), value) {
		return nil, common.Address{}, gas, ErrInsufficientBalance
	}
	nonce := evm.StateDB.GetNonce(caller.Address())
	if nonce+1 < nonce {
		return nil, common.Address{}, gas, ErrNonceUintOverflow
	}
	evm.StateDB.SetNonce(caller.Address(), nonce+1)
	// We add this to the access list _before_ taking a snapshot. Even if the creation fails,
	// the access-list change should not be rolled back
	if evm.chainRules.IsBerlin {
		evm.StateDB.AddAddressToAccessList(address)
	}
	// Ensure there's no existing contract already at the designated address
	contractHash := evm.StateDB.GetCodeHash(address)
	if evm.StateDB.GetNonce(address) != 0 || (contractHash != (common.Hash{}) && contractHash != emptyCodeHash) {
		return nil, common.Address{}, 0, ErrContractAddressCollision
	}
	// Create a new account on the state
	snapshot := evm.StateDB.Snapshot()
	evm.StateDB.CreateAccount(address)
	if evm.chainRules.IsEIP158 {
		evm.StateDB.SetNonce(address, 1)
	}
	evm.Context.Transfer(evm.StateDB, caller.Address(), address, value)

	// Initialise a new contract and set the code that is to be used by the EVM.
	// The contract is a scoped environment for this execution context only.
	contract := NewContract(caller, AccountRef(address), value, gas)
	contract.SetCodeOptionalHash(&address, codeAndHash)

	if evm.Config.Debug {
		if evm.depth == 0 {
			evm.Config.Tracer.CaptureStart(evm, caller.Address(), address, true, codeAndHash.code, gas, value)
		} else {
			evm.Config.Tracer.CaptureEnter(typ, caller.Address(), address, codeAndHash.code, gas, value)
		}
	}

	start := time.Now()

	ret, err := evm.interpreter.Run(contract, nil, false)

	// Check whether the max code size has been exceeded, assign err if the case.
	if err == nil && evm.chainRules.IsEIP158 && len(ret) > params.MaxCodeSize {
		err = ErrMaxCodeSizeExceeded
	}

	// Reject code starting with 0xEF if EIP-3541 is enabled.
	if err == nil && len(ret) >= 1 && ret[0] == 0xEF && evm.chainRules.IsLondon {
		err = ErrInvalidCode
	}

	// if the contract creation ran successfully and no errors were returned
	// calculate the gas required to store the code. If the code could not
	// be stored due to not enough gas set an error and let it be handled
	// by the error checking condition below.
	if err == nil {
		createDataGas := uint64(len(ret)) * params.CreateDataGas
		if contract.UseGas(createDataGas) {
			evm.StateDB.SetCode(address, ret)
		} else {
			err = ErrCodeStoreOutOfGas
		}
	}

	// When an error was returned by the EVM or when setting the creation code
	// above we revert to the snapshot and consume any gas remaining. Additionally
	// when we're in homestead this also counts for code storage gas errors.
	if err != nil && (evm.chainRules.IsHomestead || err != ErrCodeStoreOutOfGas) {
		evm.StateDB.RevertToSnapshot(snapshot)
		if err != ErrExecutionReverted {
			contract.UseGas(contract.Gas)
		}
	}

	if evm.Config.Debug {
		if evm.depth == 0 {
			evm.Config.Tracer.CaptureEnd(ret, gas-contract.Gas, time.Since(start), err)
		} else {
			evm.Config.Tracer.CaptureExit(ret, gas-contract.Gas, err)
		}
	}
	return ret, address, contract.Gas, err
}
```

上述代码的核心为:
```go
ret, err := evm.interpreter.Run(contract, nil, false)
```
此行代码将合约字节码进行了运行，我们再次给出合约字节码:
```
0x3d602d80600a3d3981f3363d3d373d3d3d363d73bebebebebebebebebebebebebebebebebebebebe5af43d82803e903d91602b57fd5bf3
```
当我们进行合约运行时，`EVM`解释器会从头进行运行字节码，在此处给出字节码代表的操作码:
```
[00]	RETURNDATASIZE	
[01]	PUSH1	2d
[03]	DUP1	
[04]	PUSH1	0a
[06]	RETURNDATASIZE	
[07]	CODECOPY	
[08]	DUP2	
[09]	RETURN
```

> 完整的字节码结果可以参考[这里](https://www.evm.codes/playground?unit=Wei&codeType=Bytecode&code='z602d80600ax981f336zx7zzx6z73~~~~~5af4z82803e90z91602b57fd5bf3'~yyyyxdybexz3%01xyz~_)

我们仅给出了`3d602d80600a3d3981f3`代表的代码，因为`RETURN`操作码会中止合约运行。在创建过程中，我们仅会运行上述9个操作码。

| 操作码编号 | 操作码名称 | 操作码作用 | 运行后的堆栈情况 | 运行后的内存 |
| --------- | --------- | ------- | ----- | ----- |
| 3d | RETURNDATASIZE | 向堆栈中写入`CALL`、`DELEGATECALL`的返回值的长度 | 0 | - |
| 602d | PUSH1 2d | 向堆栈中的第一个位置推入`2d` | 2d 0 | - |
| 80 | DUP1 | 复制堆栈中的第一个值 | 2d 2d 0 | - |
| 600a | PUSH1 0a | 向堆栈中的第一个位置推入`0a` | 0a 2d 2d 0 | - |
| 3d | RETURNDATASIZE | 同上 | 0 0a 2d 2d 0 | - |
| 39 | CODECOPY | 读取堆栈中的数据依次作为代码写入内存的起始位置、代码起始的读取位置和读取的代码长度 | 2d 0 | [0-2d]:runtime code
| 81 | DUP2 | 复制堆栈中第2个元素推入堆栈 | 0 2d 0 | [0-2d]: runtime code |
| f3 | RETURN | 在堆栈中依顺序读出元素作为返回值的内存起始位置和长度 | 0 | [0-2d]: runtime code

如果读者不太理解上述的操作码的含义，可以参考[EVM Codes](https://www.evm.codes/)查询操作码的参数和功能。

读者可能发现我们使用`RETURNDATASIZE`进行了堆栈操作，而在此处写显然没有任何`CALL`之类的操作。我们在此处使用`RETURNDATASIZE`的作用只是向堆栈中写入`0`。使用`PUSH1 0`写入`0`会消耗`3 gas`，而使用`RETURNDATASIZE`写入则仅消耗`2 gas`。后者更节省`gas`费用。

当`EVM`运行完上述代码后，`go-ethereum`客户端会把返回的字节码存储到`ret`变量中，通过`evm.StateDB.SetCode(address, ret)`将合约地址和合约代码保存到`StateDB`数据库中，在我们后期进行调用时，仅运行`EVM`返回的以下字节码:
```
363d3d373d3d3d363d73bebebebebebebebebebebebebebebebebebebebe5af43d82803e903d91602b57fd5bf3
```

接下来我们会介绍上述`runtime`代码的具体功能。

### 获取Calldata

正如本节概述，我们进行合约代理的第一步是获得用户向合约发送的`calldata`，使用的字节码为`363d3d37`。

获得`calldata`的操作码为[CALLDATACOPY](https://www.evm.codes/#37)，要求以下参数:

- destOffset, 将calldata复制到内存中的起始位置
- offset, 需要复制的calldata的起始位置
- size, 需要复制的calldata的长度

在此处，我们将`calldata`整体进行复制到内存中，具体操作如下表:

| 操作码编号 | 操作码名称 | 操作码作用 | 运行后的堆栈情况 | 运行后的内存 |
| --------- | --------- | ------- | ----- | ----- |
| 36 | CALLDATASIZE | 获得calldata的长度 | cds | - |
| 3d | RETURNDATASIZE | 如前文所述，一种向堆栈中推入0的廉价方式 | 0 cds | - |
| 3d | RETURNDATASIZE | 同上 | 0 0 cds | - |
| 37 | CALLDATACOPY | 如前所述复制calldata到内存 | - | [0-cds]Calldata |

在上文给出的流程中，我们仍使用了`RETURNDATASIZE`向堆栈中填入0 。

另一点需要注意的是栈属于后进先出(`LIFO`)的数据类型，所以我们需要先推入`size`参数再推入`offset`最后推入`destOffset`参数。

![Stack Picture](https://img.gejiba.com/images/1fdb103df145333ed968714379188f9b.png)

经过以上流程，我们成功把`calldata`复制到内存中，下一步则需要使用`calldata`进行`delegatecall`

### Delegatecall

此过程对应`3d3d3d363d73bebebebebebebebebebebebebebebebebebebebe5af4`字节码。

此流程最核心的操作码为[DELEGATECALL](https://www.evm.codes/#f4)，所需参数如下:

- gas, 委托调用所需要的`gas`费用
- address, 委托调用目标合约地址
- argsOffset, 委托调用所需要的`calldata`在内存中的起始位置
- argsSize, `calldata`的长度(以byte计)
- retOffset, 返回值在内存中存储的起始位置
- retSize, 返回值长度(以byte计)

在运行完成后，此操作码会向堆栈推入`1`(运行成功)或`0`(运行失败)。

此处我们介绍用于辅助的操作码[GAS](https://www.evm.codes/#5a)，该操作码不需要读取堆栈而直接向堆栈内推入gas数据。

根据我们目前的内存情况和目标，我们应该构建以下堆栈以供`DELEGATECALL`调用:
```
[ GAS | address | 0 | cds | 0 | 0 ]
```
此处把`retOffset`和`retSize`设置为`0`的原因是我们在此处无法知道返回值的长度，所以将其设置为`0`。但设置为0并不意味着我们读返回值，返回值仍会被保存在`return data`的特殊区域，我们会在下一环节读取返回值。

> 在`EVM`中，存储区域有代码存储区域、堆栈、内存、存储、`CallData`存储和`Return data`存储区域。我们在上文中，由于不清楚返回值的情况，所以通过设置`retOffset`和`retSize`为0，阻止了返回值直接写入内存，在后文我们会在`Return data`中提取返回值。

> 更多关于EVM存储可以参考[此网站](https://www.evm.codes/about#executionenv)

我们依旧采用表格的方式逐步分析字节码:
| 操作码编号 | 操作码名称 | 运行后的堆栈情况 |
| --------- | --------- | ----- | 
| 3d | RETURNDATASIZE | 0 |
| 3d | RETURNDATASIZE | 0 0 |
| 3d | RETURNDATASIZE | 0 0 0 |
| 36 | CALLDATASIZE | cds 0 0 0 |
| 3d | RETURNDATASIZE | 0 cds 0 0 0 |
| 73bebebebebebebebebebebebebebebebebebebebe | PUSH20 bebe... | addr 0 cds 0 0 0 |
| 5a | GAS | gas addr 0 cds 0 0 0 |
| f4 | DELEGATECALL | success 0 |

*由于此处不涉及内存读写，所以我们删除了此列。同时考虑到读者可以理解此列表中大大部分操作码，所以也删掉了"操作码作用"一列

读者可能发现了此处我们在堆栈中多写入了一个`0`，这是因为`delegatecall`后，我们无法再通过`RETURNDATASIZE`操作码以廉价的方式写入`0`，我们在此处多填入一个0以方便后期使用。

### 获取Returndata

此流程对应的字节码为`3d82803e`

核心操作码为[RETURNDATACOPY](https://www.evm.codes/#3e)，所需参数为:

- destOffset, `Returndata`复制到内存中的起始位置
- offset, 需要复制的`Returndata`的起始位置
- size, 需要复制的`Returndata`的长度

当然，此处主要使用的辅助操作码为`DUPn`(其中`n∈[1, 16]`)，其主要为将堆栈中的第`n`个元素复制并推入堆栈。如目前堆栈中存在`0 1`两个元素，使用`DUP2`后运行完后堆栈为`1 0 1`，即将第二个元素`1`复制并推入堆栈。

在此处我们给出分析表格:
| 操作码编号 | 操作码名称 | 运行后的堆栈情况 | 内存 |
| --------- | --------- | ----- | ---- |
| 3d | RETURNDATASIZE | rds success 0 | [0-cds]Calldata |
| 82 | DUP3 | 0 rds success 0 | [0-cds]Calldata |
| 80 | DUP1 | 0 0 rds success 0 | [0-cds]Calldata |
| 3e | RETURNDATACOPY | success 0 | [0-rds]Returndata |

在此过程中，我们完成将返回值复制进入内存，在下一步中，我们会真正把返回值或错误返回给用户。

### 返回

此流程对应操作码为`903d91602b57fd5bf3`

在返回值之前，我们需要判断`success`的值，如果此值为`1`，说明`delegatecall`成功，我们以正常形式返回内存中的结果; 如果此值为`0`，说明`delegatecall`失败，我们则使用`REVERT`，以错误信息的形式返回内存中的值。

上述过程依赖于[JUMPI](https://www.evm.codes/#57)操作码，此操作码接受以下参数:

- counter, 需要跳转的代码位置
- b, 若`b`不为`0`则进行跳转，否则则不跳转继续运行。

与`JUMPI`对应的跳转位置需要存在`JUMPDEST`操作码标识跳转位置。

此处所使用的另两个重要操作码`RETURN`和`REVERT`所需参数相同，均为:

- offset, 返回值在内存中存储的起始位置
- size, 返回值的长度

> 注意这两个操作码一旦运行则标志合约运行的结束。EVM读取到这两个操作码后，不会再继续运行。

为了方便读者理解跳转关系，我们使用一下图像:
```
|           0x00000024      90             swap1                 0 suc
|           0x00000025      3d             returndatasize        rds 0 suc
|           0x00000026      91             swap2                 success 0 rds
|           0x00000027      602b           push1 0x2b            0x2b success 0 rds
|       ,=< 0x00000029      57             jumpi                 0 rds
|       |   0x0000002a      fd             revert
|       `-> 0x0000002b      5b             jumpdest              0 rds
\           0x0000002c      f3             return
```
上图来自[EIP1167文档](https://eips.ethereum.org/EIPS/eip-1167)

第一列为操作码的位置，第二列为操作码的编号，第三列为操作码名称，最后一列为堆栈情况。

为了最后进行值的返回，我们首先构造通过一系列操作将堆栈中最后两个元素设置为`0 rds`，方便`RETURN`或`REVERT`操作码使用。同时也构造了`0x2b success`作为`JUMPI`所需要的参数。图中也表示了不同的跳转方式，当`success`不为0时，即`delegatecall`运行成功，代码跳转到`0x2b`运行执行`RETURN`，返回正常值。如果`success`为0，则执行`revert`，将返回值作为异常抛出。

## 总结

本文主要介绍了以下内容:

- `openzeppelin`的`clone`函数生成字节码的过程
- `go-ethereum`创建智能合约的源代码
- `EIP1167`字节码具体运作流程

除此之外，我们还介绍EVM运行环境的基本情况和常见字节码的含义。

读者可以阅读[EIP1167文档](https://eips.ethereum.org/EIPS/eip-1167)中给出的流程图进一步理解字节码运行流程。
