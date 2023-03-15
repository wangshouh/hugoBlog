---
title: "Foundry 高级测试: Fuzz、Invariant与形式化证明"
date: 2023-03-14T23:47:33Z
tags: [foundry,solidity]
---

## 概述

本文以较为简单的 `WETH` 合约为例，介绍在 `Foundry` 架构中常用的几种较为高级的测试方法，如下:

1. Fuzz Testing 基于属性的单元测试的升级版
1. Invariant Testing 基于随机数据整体调用的测试
1. Formal Verification 形式化证明

本文也会给出上述测试手段的 `github ci` 配置文件。

这些方法都是简单的单元测试的升级版，关于 `Foundry` 的使用，读者可以参考我之前编写的 [Foundry教程：编写测试部署ERC-20代币智能合约](https://blog.wssh.trade/posts/foundry-with-erc20/)。

## 环境准备

此文使用 `WETH9` 作为待测试合约，该合约有一个无伤大雅的特殊错误，本文期望可以使用 `Invariant Test` 将其测试出来。

为了防止读者的 `foundry` 版本过低而不支持本文中的部分操作，请读者运行以下命令进行升级:

```bash
foundryup
```

在读者准备进行合约测试的文件夹内运行以下命令:

```bash
forge init WETHTest
```

接下来，我们使用 `cast` 命令下载 WETH 合约文件，此方法已经在 [智能合约开发效率工具](https://blog.wssh.trade/posts/smart-contract-tool/) 内提及。但为保持文章完整性，我们在此处在介绍一次。

使用 `cd WETHTest` 切换进入文件夹内，创建 `.env` 文件，填入以下内容:

```env
etherscan=您的etherscan API密钥
```

运行 `source .env` 读取`.env`中的设定并置为环境变量，运行以下命令将合约读取到 `src` 目录中，

```bash
cast etherscan-source 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2 --etherscan-api-key $etherscan > src/WETH.sol
```

最后增加映射以避免报错，命令如下:

```bash
forge remappings > remappings.txt
```

至此，读者应该可以在 `src` 目录中看到这一合约，项目整体结构如下:

```
.
├── foundry.toml
├── lib
│   └── forge-std
├── script
│   └── Counter.s.sol
├── src
│   ├── Counter.sol
│   └── WETH.sol
└── test
    └── Counter.t.sol
```

我们在此处没有介绍如何进行 `foundry.toml` 的配置，这些内容将在后文介绍测试方法时在给出。

我们给出 WETH 合约中函数的作用:

1. `deposit` 存入，将用户的 ETH 转换为 WETH
1. `withdraw` 退回，将会用户的 WETH 转换为 ETH
1. `totalSupply` 展示当前合约所以 ETH 的数量
1. `approve` 授权，将 WETH 的使用权利让渡给其他用户
1. `transfer` 转移自己的 WETH 代币
1. `transferFrom` 转移他人授权的 WETH 代币

## 修正合约

由于 `WETH` 使用的合约较为古老，直接进行测试会出现以下报错:

```
Error: 
Discovered incompatible solidity versions in following
: test/WETH.t.sol (^0.8.13) imports:
    lib/forge-std/src/Test.sol (>=0.6.2 <0.9.0)
    src/WETH.sol (^0.4.18)
    ...
```

所以此处我们对 `WETH` 合约继续简单的版本升级，首先在其头部加入开源协议标识，如下:

```
// SPDX-License-Identifier: GPL-3.0
```

> 此处我们选择了与原合约相同的开源协议

在旧版本中，我们使用了以下回退函数:

```solidity
function() public payable {
    deposit();
}
```

但在较新的 `solidity` 中，我们需要修正为:

```solidity
receive() external payable {
    deposit();
}
```

如果一笔交易没有 `calldata` ，即没有调用任何函数，我们则认为该交易调用了 `receive` 函数。

> 更多内容请参考 [Receive Ether Function](https://docs.soliditylang.org/en/latest/contracts.html#receive-ether-function)

然后，我们发现包括 `Deposit` 在内的所有 `event` 都没有 `emit` 作为释放事件的前缀，读者应该增加此关键词。比如:

```solidity
emit Deposit(msg.sender, msg.value);
```

在 `withdraw` 函数中，原合约使用了 `msg.sender.transfer(wad);` 进行 ETH 的提取，但在新版本的 `solidity` 中，我们需要将 `msg.sender` 设置为 `payable address` 类型才可以进行此操作，所以修改后的代码如下:

```solidity
address payable withdrawAddress = payable(msg.sender);
withdrawAddress.transfer(wad);
```

> 此处使用 `transfer` 以防止被重入攻击，`transfer` 操作只能使用 2300 gas ，这意味着如果对方进行重入攻击等调用会因为 `Out of Gas` 而失败。

在 `transferFrom` 函数中，我们发现开发者使用了以下条件判断:

```solidity
src != msg.sender && allowance[src][msg.sender] != uint(-1)
```

此处的 `uint(-1)` 就是指 `uint256` 的最大值，但是在新版本的 `solidity` 中，该方法已被弃用，我们可以改为:

```
src != msg.sender && allowance[src][msg.sender] != type(uint256).max
```

至此，我们完成的大部分升级。其中，最可能影响 `WETH` 作用的变动是加入了 `receive` 函数和修正了 `withdraw`，所以在下文我们先从这些函数进行测试。

## Fuzz Testing

我们先给出一个对 `receive` 的正常的单元测试:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../src/WETH.sol";

contract WETHTest is Test {
    WETH9 private weth;

    receive() external payable {}
    
    function setUp() public {
        weth = new WETH9();
    }

    function testDesposit() public {
        (bool sent, ) = address(weth).call{value: 10 ether, gas: 1 ether}("");
        require(sent, "Send ETH");
        assertEq(weth.balanceOf(address(this)), 10 ether);
        weth.withdraw(10 ether);
        assertEq(weth.balanceOf(address(this)), 0);
    }
}
```

在此处，我们补充一点关于合约测试的调用问题，默认情况下，我们编写的所有函数调用的发起者都是 `WETHTest` 合约。当然，我们有办法使用 EOA 作为合约调用发起者，这需要使用 `vm.prank` 函数，我们会在后文展示。

值得注意到是此处我们没有使用 `transfer` 函数进行 ETH 转移而使用了 `call` 函数并指定了 `gas` 。这是因为追求简单的原因。在此处，我们给出三种合约转账函数的区别:

1. `transfer` 要求 `paybable address` 且 `gas` 限制为 2300 ，不可调整。在此处，由于 WETH 合约运行需要一定的 gas 所以如果使用此函数会出现 `FAIL. Reason: EvmError: Revert` 错误，原因是 gas 耗尽
2. `send` 所有性质与 `transfer` 一致，但会返回代表是否转账成功的布尔值
3. `call` 最底层的转账函数，允许用户指定转账金额、gas和 calldata。该函数会代表返回请求是否成功的布尔值和请求返回数据。该函数较为底层不建议直接在生产环境使用，但我们经常在测试环境种使用

读者可以通过 [Solidity by Example](https://solidity-by-example.org/sending-ether/) 获得更多示例。

> 如果读者阅读过我早期关于代理合约架构的一系列文章，就会发现
> 我在测试过程中大量使用了 `call` 进行合约调用

显然，我们的 `testDesposit` 证明了 `WETH` 提款是在 `10 ether` 的情况下是安全的，但我们需要证明 `withdraw` 函数在其他金额下也是安全的，为达成这一目标，我们需要进行 `fuzz` 测试。

> 我们给出的 `testDesposit` 是一个单元测试，单元测试可以证明
> 程序的有效性，但对于程序是否存在漏洞并无法证明，这是因为单
> 元测试往往是 **乐观测试**，仅仅对很小一部分情况进行测试，而无法大范围遍历测试空间

简单来说，`fuzz` 测试允许我们测试某一个函数在不同输入下是否均有效，理论上，我们可以使用 `fuzz` 测试函数所有可行的输入，最终证明函数的完全正确性。但在实际中，限于计算机性能，我们往往只能进行有限次数的测试。

一个典型的 fuzz 测试流程如下:

![Fuzz](https://files.catbox.moe/s7684t.svg)

关于 `fuzz` 测试，Google 有有个 [专题仓库](https://github.com/google/fuzzing) ，但需要注意的此仓库是对一般软件的 `fuzz` 测试而不是针对于智能合约开发。

在此处，我们给出 `testDesposit` 的 `fuzz` 版本:

```solidity
function testDesposit(uint256 amount) public {
    (bool sent, ) = address(weth).call{value: amount, gas: 1 ether}("");
    require(sent, "Send ETH");
    assertEq(weth.balanceOf(address(this)), amount);
    weth.withdraw(amount);
    assertEq(weth.balanceOf(address(this)), 0);
}
```

使用 `forge test -vvv` 启动测试，读者应该会看到一个报错:

```
[FAIL. Reason: Send ETH Counterexample: calldata=0xd9685cc20000000000000000000000000000000000000001000000000000000000000000, args=[79228162514264337593543950336]] testDesposit(uint256) (runs: 57, μ: 33716, ~: 33712)
```

这是因为测试合约自身在测试环境中仅有 `2**96 wei` 的 ETH，而 `fuzz` 测试中会随机选择 `amount` 的值，这可能导致转移的 ETH 大于合约所拥有的 ETH。一个可行的解决方案是限制 `amount` 的大小，修正后的测试函数如下:

```solidity
function testDesposit(uint96 amount) public
```

此测试可以通过，输出如下:

```
[PASS] testDesposit(uint96) (runs: 256, μ: 32626, ~: 33778)
```

其中，`runs` 代表合约测试运行测试，`μ` 代表合约测试的平均 gas 费用，`~` 代表合约测试 gas 费用的中位数。

可能有读者发现 `fuzz` 测试运行了 256 次，而我们的合约上链后可能运行几万次，我们是否可以调整测试次数？事实是可以的，我们可以通过修改 `foundry.toml` 文件实现。
 
在 `foundry.toml` 内加入:

```toml
[fuzz]
runs = 1024
```

通过修改 `runs` 参数，我们可以修改 `fuzz` 测试次数。当然，测试次数越多，合约测试运行速度越慢，另一方面，也意味着合约更加安全，部分 `fuzz` 测试工具将此值设置为 50000 ，读者可根据需求自行修改此参数。

> [Fuzz Config](https://book.getfoundry.sh/reference/config/testing#fuzz) 内包含其他参数的含义，读者可根据需求自行设置

在 `fuzz` 测试中，一个有用的 `cheatcode` 是 `vm.assume`，该函数可以限制 `fuzz` 随机生成的参数，如果参数不满足 `vm.assume` 的限制会被丢弃。下文展示了一个带有 `assume` 的测试:

```solidity
function testDesposit(uint256 amount) public {
    vm.assume(amount > 0.1 ether);
    // snip
}
```

更多关于 `vm.assume` 的内容请参考 [assume 文档](https://book.getfoundry.sh/cheatcodes/assume)

可能有读者发现这些内容都可以在 [Foundry 文档](https://book.getfoundry.sh/forge/fuzz-testing) 里找到。是的，`foundry` 文档时看时新，建议大家经常翻阅。接下来，我会介绍一些没有写在 `foundry` 文档中的内容，即讨论任何科学的对一份合约进行 `fuzz testing`。我也一直疑惑这一问题，但这一问题目前已经有了部分解。

`fuzz` 测试的核心是判断函数是否符合某个属性，属性应该是我们定义的合约运行必须要满足的某个数学条件，比如 `deposit` 函数本质上是以下数学限制:

1. 用户 WETH 余额 = 原 WETH 余额 - 提款数量
2. 用户 ETH 余额 = 原 ETH 余额 + 提款数量

当我们获得这两个数学限制后，我们首先判断在此问题中的变量有以下两个:

1. 原 WETH 余额，记为 `preWETHBalance`
2. 原 ETH 余额，记为 `preETHBalance`
3. 提款数量，记为 `amount`

即等式右侧的变量。

基于上述变量和等式关系，我们可以设计如下 `fuzz` 测试:

```solidity
function testDesposit(uint96 amount, uint96 preBalance, uint96 preETHBalance) public {
    vm.assume(preBalance > amount);

    address payable tester = payable(address(0x42));
    
    deal(tester, preETHBalance);
    deal(address(weth), tester, preBalance);

    deal(address(weth), uint256(amount) + 2300);
    
    vm.prank(tester);
    weth.withdraw(amount);

    assertEq(weth.balanceOf(tester), preBalance - amount);
    assertEq(tester.balance, uint(preETHBalance) + uint(amount));
}
```

此测试代码的核心是最后的 `assertEq` 函数:

```solidity
assertEq(weth.balanceOf(tester), preBalance - amount);
assertEq(tester.balance, uint(preETHBalance) + uint(amount));
```

这两个函数就是我们的对 `desposit` 的数学要求。

> 此处对于所有测试中的进行加法操作的变量都进行 `uint` 操作，这是为了避免溢出

在此处，我们也简述一下上述测试代码的具体含义:

1. 设置限制条件

    ```solidity
    vm.assume(preBalance > amount);
    ```
    该限制条件是一个硬性条件，我不建议在测试代码中增加过多的硬性条件
1. 初始化测试者参数
    ```solidity
    deal(tester, preETHBalance);
    deal(address(weth), tester, preBalance);
    ```

    此处使用的 `deal` 用于直接改变账户 ETH 或 ERC20代币余额，具体请参考 [文档](https://book.getfoundry.sh/cheatcodes/deal)
1. 初始化 WETH 合约参数
    
    ```solidity
    deal(address(weth), uint256(amount) + 2300);
    ```

    此处更改 WETH 合约余额为 `uint256(amount) + 2300` ，增加 2300 作为 `gas` 费用
1. 函数调用

    ```solidity
    vm.prank(tester);
    weth.withdraw(amount);
    ```

    此处使用 `vm.prank` 是为了修改 `weth.withdraw` 函数调用的 `msg` 为 `tester`

对于开发者而言，我们在编写智能合约前最好使用尽可能数学的形式描述相关功能，将函数完全抽象成数学关系。当我们可以把函数抽象为纯数学关系后，我们就可以使用 `fuzz testing` 对数学关系进行随机测试，以实现验证安全性的目的。

> 事实上，形式化证明也是对数学关系的验证，但形式化证明更进一步，可以在数学上遍历整个输出空间以完全证明函数的正确性。显然，通过简单的抽取元素的方法遍历输出空间是不显示的，所以我们往往需要一个数学求解器，这就是为什么大部分形式化证明工具都依赖 [Z3](https://github.com/Z3Prover/z3) 等求解器的原因，读者可以参考 [Satisfiability modulo theories](https://en.wikipedia.org/wiki/Satisfiability_modulo_theories) 词条。

如果读者不想自己总结某些 EIP 标准的数学原理及测试，可以参考 Trail of Bits 安全公司开源的 [Properties](https://github.com/crytic/properties/blob/main/PROPERTIES.md) 仓库。此仓库总结了以下标准合约的 `fuzzing` 测试项目:

1. ERC20 代币
2. ERC4626 金库
3. ABDKMath64x64 浮点运算

> 更多内容请参考 [Reusable properties for Ethereum contracts](https://blog.trailofbits.com/2023/02/27/reusable-properties-ethereum-contracts-echidna/) 一文

我们以较简单的 `ERC20-BASE-007` 为示例为大家展示然后进行 `fuzzing` 测试。在表格中，我们可以看到 `ERC20-BASE-007` 的 `Invariant tested` 描述为 `Self transfers should not break accounting.`

此处的 `Invariant tested` 就是不变量，即一个等式关系，我们可以根据此内容编写出对应的 `assertEq` 等断言。读者也发现此处已经给出了一个 [测试函数](https://github.com/crytic/properties/blob/main/contracts/ERC20/internal/properties/ERC20BasicProperties.sol#L70) ，但此函数是用于 `echidna` 而不是 `foundry` 的，所以我们需要重写此函数。原测试函数如下:

```solidity
// Self transfers should not break accounting
function test_ERC20_selfTransfer(uint256 value) public {
    uint256 balance_sender = balanceOf(address(this));
    require(balance_sender > 0);

    bool r = this.transfer(address(this), value % (balance_sender + 1));
    assertWithMsg(r == true, "Failed self transfer");
    assertEq(balance_sender, balanceOf(address(this)), "Self transfer breaks accounting");
}
```

主题逻辑是测试用户进行自转账是否会导致余额增加，对于很多编写不严谨的 ERC20 代币合约可能会发生此情况。

基于以上逻辑可以编写出以下测试:

```solidity
function test_ERC20_selfTransfer(uint256 value, uint96 preBalance) public {
    vm.assume(preBalance > value);

    address payable tester = payable(address(0x1337));

    deal(address(weth), tester, preBalance);

    vm.prank(tester);
    weth.transfer(tester, value);

    assertEq(weth.balanceOf(tester), preBalance);
}
```

大部分代码都与上文的 `testDesposit` 一致。可能有读者好奇这个测试为什么存在？为证明这一问题，请读者在 `WETH` 合约内加入以下代码:

```solidity
function errorTansfer(address dst, uint256 wad) public {
  uint256 srcBalance = balanceOf[msg.sender];
  uint256 dstBalance = balanceOf[dst];
  balanceOf[msg.sender] = srcBalance - wad;
  balanceOf[dst] = dstBalance + wad;
}
```

读者可以预判以下此代币转账会发生什么？

复用上述测试，并把 `transfer` 更改为 `errorTansfer` 进行测试，运行 `forge test -vvv` 读者可能会获得以下报错:

```
Logs:
  Error: a == b not satisfied [uint]
    Expected: 10310
      Actual: 19557
```

显然，用户进行自转账后账户余额增加了，这显然不符合常理。

进行完上述测试后，请将 `errorTansfer` 函数删除避免影响后续测试。


## Invarient Testing

在前文我们介绍了较为复杂的 `Fuzz Testing`，但在真实的链上合约运行环境中，我们所遇到的情况比 `fuzzing testing` 更加复杂，我们不清楚被测试的函数会在何种情况下被调用，我们需要为测试环境引入更大的复杂性。在较新版本的 `foundry` 中，开发者引入了一种全新的测试函数，即 `Invarient Testing` 。

在正式引入 `Invarient Testing` 前，我们首先介绍该测试方法的实现原理:

1. 测试器根据合约函数生成多次随机的 `calldata` ，构成一个随机调用序列。在此流程中，选择的被调用函数和输入的参数都是完全随机的且发起交易的地址也是随机的。我们定义随机调用序列的长度为 **depth** 。
2. 使用上文生成的随机调用序列对合约进行调用 
3. 等待随机请求结束后，根据我们的测试文件中的断言判断当前合约状态是否符合预期

上述流程可以使用下图概括:

![Invarient Testing Graph](https://files.catbox.moe/fr5jgk.svg)

值得注意的是，该流程会进行多次，已达到尽可能的随机性。上述流程重复的次数被记为 **runs** 。

后文部分内容参考了 [Invariant Testing WETH With Foundry](https://mirror.xyz/horsefacts.eth/Jex2YVaO65dda6zEyfM_-DXlXhOWCAoSpOx5PLocYgw) 文章，但并没有完全一致。相对于原文，本文减少了部分测试，并对代码进行部分修改。

为了更加详细的解释这一过程，我们编写以下测试: 

```solidity
function invariant_wethSupplyIsAlwaysZero() public {
    assertEq(0, weth.totalSupply());
}
```

我们得到了如下输出结果:

```solidity
[PASS] invariant_wethSupplyIsAlwaysZero() (runs: 1024, calls: 15360, reverts: 8832)
```

其中个参数含义如下:

- `runs` 进行判断前随机调用合约的次数
- `calls` 调用的总次数，相当于 `runs` 与 `depth` 的乘积
- `reverts` 因不合法调用而失败的次数

> 显然，在上述随机调用序列生成时，我们仅对调用函数进行了限制，而没有限制函数参数和函数调用的先后顺序，所以出现一定失败调用是正常的。

有读者可能好奇为什么 `totalSupply = 0` 这种离谱条件可以通过测试。这是因为正如上文所述，我们仅随机生成了 `calldata` 而没有生成 `msg.value` 等交易参数，所以事实上，没有一笔交易成功铸造出 WETH ，所以才产生了如此离谱断言也被通过。

除了上述问题外，我们也发现近一半的调用出现 `reverts` 的情况。众所周知，`revert` 不影响合约状态，那么产生 `revert` 的调用没有起到复杂化合约测试环境的作用。我们可以认为这些调用被浪费了。简单计算 `invariant_wethSupplyIsAlwaysZero` 中 `reverts` 与 `calls` 之比，我们发现一半以上的调用都被浪费。

> 我们假设读者熟悉 `revert` 的作用，简单来说，`revert` 会在合约调用失败后回滚 EVM 状态，使其回复调用前的状态。

根据上文的描述，读者可能感觉 `Invarient Testing` 过于随机的属性反而带来了一些坏处。我们需要将其随机性，保证其生成的调用序列符合一定规则。

> 当然，我们也要保证调用序列的一定的随机性，否则就丧失了测试的作用。读者需要自行权衡。

如何解决调用序列的缺陷？我们可以使用一种特殊工具 `Handler` 合约。简单来说，这是一种对待测试合约的封装。请读者在 `test` 文件夹内创建 `handlers/Handler.sol` 文件，写入以下内容:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import {WETH9} from "../../src/WETH.sol";
import {StdCheats} from "forge-std/StdCheats.sol";

contract Handler is StdCheats {
    WETH9 public weth;

    constructor(WETH9 _weth) {
        weth = _weth;
        deal(address(this), 10 ether);
    }

    function deposit(uint256 amount) public {
        weth.deposit{ value: amount }();
    }
}
```

该合约较为简单，唯一值得注意的是在我们构造合约时使用了 `deal` 函数为合约准备了 10 ether (`deal` 函数是 `StdCheats` 引入的 `cheatcode` ，不可在非测试环境下使用) 。此行为的目的是方便后期 `desposit` 操作。

> 请注意，当我们编写 `handler` 时，我们已经引入了一些预设，比如上述 `handler` 就预设用户会在持有 ETH 情况下与合约交互。我们需要认为判断每一个预设是否合理，以及该预设不成立情况下会产生的后果。这些预设可能导致合约通过了测试，但在真实链上环境中因为预设被打破而产生风险。

我们将对 WETH 的 `invariant testing` 更改为对 Handler 合约的测试。我们需要编写一个新的测试合约，将其命名为 `WETH.invariants.t.sol` ，该合约应置于 `test` 文件夹内。创建合约后，我们写入以下内容:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../src/WETH.sol";
import "./handlers/Handler.sol";

contract WETHInvariants is Test {
    WETH9 public weth;
    Handler public handler;

    function setUp() public {
        weth = new WETH9();
        handler = new Handler(weth);

        targetContract(address(handler));
    }

    function invariant_wethSupplyIsAlwaysZero() public {
        assertEq(0, weth.totalSupply());
    }
}
```

其中，`targetContract` 是一个指定待测试合约的函数。因为此处出现了 WETH 及其 Handler 两个合约，所以指定测试合约是必要的。使用 `foundry test` 运行测试，我们得到了以下报错:

```
Encountered 1 failing test in test/WETH.invariants.t.sol:WETHInvariants
[FAIL. Reason: Assertion failed.]
        [Sequence]
                sender=0x00000000000000000000000000000000000005e3 addr=[test/handlers/Handler.sol:Handler]0x2e234dae75c793f67a35089c9d99245e1c58470b calldata=deposit(uint256), args=[4070422560964460496250185987525772964564029773985053444012266177280]
                sender=0x0000000000000000000000000000000000000936 addr=[test/handlers/Handler.sol:Handler]0x2e234dae75c793f67a35089c9d99245e1c58470b calldata=deposit(uint256), args=[429415003]

 invariant_wethSupplyIsAlwaysZero() (runs: 1, calls: 2, reverts: 1)
```

正如我们预期，测试果然没有通过。关于此处为什么运行了两次才判断失败，可能是因为测试过程的并行化导致的。

> 我们可以看到 `sender` 有所不同，这是因为函数调用者 `sender` 也是随机生成的。

接下来，我们检验 WETH 合约的一个基础属性，即 WETH 合约是否具有合格的偿付能力。换言之，即 WETH 的数量与其合约所持有的 ETH 数量是否一致。

该问题可以抽象为以下数学等式是否成立:

WETH 持有的 ETH 数量 = 存入数量 - 提款数量

我们需要一个遍历来跟踪测试期间的存入数量和提款数量。在 `invariant testing` 中，这种用于跟踪状态的变量被称为幽灵变量 `ghost variable`(暂未见到合适的中文翻译，故直接直译)。

继续分析上述等式，其中存入数量可以拆分为 `deposit` 和回退存入(即不带有 calldata 情况下合约自动调用 `receive` 的存入)，而提款数量只可以拆分为 `withdraw` 提款。

我们将存款数量定义为 `ghost_depositSum` 而提款数量定义为 `ghost_withdrawSum`。那么基于上述数学等式，我们会得到如下断言:

```solidity
function invariant_solvencyDeposits() public {
    assertEq(
      address(weth).balance,
      handler.ghost_depositSum() - handler.ghost_withdrawSum()
    );
}
```

由此，我们可以编写出以下 Handler :

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import {WETH9} from "../../src/WETH.sol";
import {StdCheats} from "forge-std/StdCheats.sol";
import {StdUtils} from "forge-std/StdUtils.sol";

contract Handler is StdUtils, StdCheats {
    WETH9 public weth;

    uint256 public constant ETH_SUPPLY = 120_500_000 ether;
    uint256 public ghost_depositSum;
    uint256 public ghost_withdrawSum;

    constructor(WETH9 _weth) {
        weth = _weth;
        deal(address(this), ETH_SUPPLY);
    }

    function deposit(uint256 amount) public {
        amount = bound(amount, 0, address(this).balance);
        weth.deposit{ value: amount }();
        ghost_depositSum += amount;
    }

    function withdraw(uint256 amount) public {
        amount = bound(amount, 0, weth.balanceOf(address(this)));
        weth.withdraw(amount);
        ghost_withdrawSum += amount;
    }

    function sendFallback(uint256 amount) public {
        amount = bound(amount, 0, address(this).balance);
        (bool success,) = address(weth).call{ value: amount }("");
        require(success, "sendFallback failed");
        ghost_depositSum += amount;
    }
}
```

在上述合约中，函数包装关系如下:

- `deposit` 包装 WETH 的 `desposit` 函数
- `withdraw` 包装 WETH 的 `withdraw` 函数
- `sendFallback` 包装 WETH 的 `receive` 回退，注意 `receive` 回退严格上说不是一个函数，只是一种在没有 `calldata` 情况下运行机制。

在上述合约中，我们使用了一个有意思的测试用函数 `bound`，该函数被定义在 `StdUtils` 中，用于生成限定范围内的参数，具体用途请参考 [文档](https://book.getfoundry.sh/reference/forge-std/bound)。需要注意该函数会减少测试环境复杂度，请谨慎使用。

> 在 Foundry 文档内，`bound` 函数被列在 `std-cheats` 目录下，而事实上，`StdCheats.sol` 内没有此函数。建议读者不要完全相信 Foundry 文档内的函数位置。

完成上述所有内容后，使用 `foundry test` 进行测试，果然测试全部通过。

我们还有另一种方法证明 WETH 正确性，公式如下:

WETH 的 ETH 数量 = 各存款地址的 WETH 数量之和

显然，该条件是更加细粒化的。我们目前面临以下问题:

1. 随机地址生成方法
2. 随机地址的初始化，我们需要对随机地址赋予 ETH 等操作
3. 使用随机地址调用函数，在此流程中，我们需要记录各地址的 WETH 数量之和
4. 判断所有随机地址的 WETH 之和与 WETH 持有的 ETH 数量是否相同

首先，根据 `invarient testing` 的流程，我们会发现在测试时，`invarient testing` 进行随机调用时就生成了随机地址。

对于随机地址的 ETH 赋予，我们可以使用 `deal` 函数实现。而使用随即地址调用函数，则可以使用 `vm.prank`，该函数会改变下一次 `call` 的 `msg.sender`。

而最后一步判断则可以使用 `ghost variable` 实现。

为了充分解耦随即地址生成、设置与智能合约测试的关系，我们决定使用 Foundry Book 中给出的 [Actor Management](https://book.getfoundry.sh/forge/invariant-testing#actor-management) 模式。此模式使用一个 `modifier` 实现了地址的相关设置，而主体调用逻辑不需要修改。

为了尽可能降低代码的耦合性，我们将地址列表作为一个专门的数据结构分离出来，请读者创建 `test/helpers/AddressSet.sol` 合约，并写入以下内容:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

struct AddressSet {
    address[] addrs;
    mapping(address => bool) saved;
}

library LibAddressSet {
    function add(AddressSet storage s, address addr) internal {
        if (!s.saved[addr]) {
            s.addrs.push(addr);
            s.saved[addr] = true;
        }
    }

    function rand(AddressSet storage s, uint256 seed) internal view returns (address) {
        if (s.addrs.length > 0) {
            return s.addrs[seed % s.addrs.length];
        } else {
            return address(0);
        }
    }
}
```

我们构造了 `AddressSet` 特殊数据结构，该数据结构追踪并汇总合约调用者。为了实现集合的性质，我们使用了 `mapping` 映射。其中 `add` 函数用于向 `AddressSet` 内增加地址，而 `rand` 函数用于根据 `seed` 返回一个地址。

> 此处使用 `%` 取余来保证输入的变量位于 `s.addrs` 长度内。这样可以尽可能重用已有存款地址，与实际情况更加符合。

接下来，我们调整 `Handler.sol` 合约，增加以下 `import` 语句:

```solidity
import {CommonBase} from "forge-std/Base.sol";
import {LibAddressSet, AddressSet} from "../helpers/AddressSet.sol";
```

前者用于导入 `vm` 系列函数，后者用于导入定义的 `LibAddressSet` 和 `AddressSet` 数据结构及函数。为了使用导入的数据结构和函数，请在合约内加入以下代码:

```solidity
using LibAddressSet for AddressSet;

AddressSet internal _actors;

uint256 constant MAX_TRANSFER = 12_000_000 ether;
```

> 您可以在 [此仓库](https://github.com/wangshouh/WETHTest/blob/master/test/handlers/Handler.sol) 找到对应的代码。

在 Handler 合约内，我们需要加入以下 `modifier`：

```solidity
modifier addActor(uint256 amount) {
    _actors.add(msg.sender);
    deal(msg.sender, amount);
    vm.startPrank(msg.sender);
    _;
    vm.stopPrank();
}

modifier useActor(uint256 seed) {
    address addr = _actors.rand(seed);
    vm.startPrank(addr);
    _;
    vm.stopPrank();
}
```

其中，`addActor` 用于向 `_actors` 增加地址，该修饰器用于 `deposit` 和 `fallback` 函数。而 `useActor` 则是返回 `_actors` 的随即地址，用于 `withdraw` 函数。修改后的 `deposit` 、 `withdraw` 和 `fallback` 函数如下:

```solidity
function deposit(uint256 amount) public addActor(amount) {
    amount = bound(amount, 0, MAX_TRANSFER);
    weth.deposit{ value: amount }();
    ghost_depositSum += amount;
}

function withdraw(uint256 amount) public useActor(amount) {
    amount = bound(amount, 0, weth.balanceOf(msg.sender));
    weth.withdraw(amount);
    ghost_withdrawSum += amount;
}

function sendFallback(uint256 amount) public addActor(amount) {
    amount = bound(amount, 0, MAX_TRANSFER);
    (bool success,) = address(weth).call{ value: amount }("");
    require(success, "sendFallback failed");
    ghost_depositSum += amount;
}
```

为了模拟真实世界中的交易，我们使用 `bound` 将存款输入进行了限制，此处使用了上文定义的 `MAX_TRANSFER` 常量。该常量是我对以太坊整体交易数据集检索后的结果，使用了 `Dune` 作为数据源:

![Dune Max](https://img.gejiba.com/images/6f5187dc1154794511d41366c0b54be4.png)

> 读者可以使用 [此链接](https://dune.com/queries/2186276) 访问搜索结果和代码

对于函数的具体分析，我们不再此处详细分析，总体上与之前的测试基本一致。使用 `forge test` 可以发现测试通过，但存在少量的 `reverts` ，此处不再研究其产生原因。

此处使用了幽灵变量进行累加计算，但我们可以尝试去掉 `ghost_depositSum` 和 `ghost_withdrawSum` 变量，而是在测试断言时通过遍历 `_actors` 累加实现。为此，我们需要修正 `LibAddressSet` 库，为其增加一些简单的函数。在此处，我们将展示一些较为少用的 `solidity` 函数式编程。在 `LibAddressSet` 内增加以下函数:

```solidity
function forEach(
    AddressSet storage s,
    function(address) external returns (uint256) func
) internal returns (uint256[] memory) {
    uint256[] memory arr = new uint256[](s.addrs.length);
    for (uint256 i; i < s.addrs.length; ++i) {
        arr[i] = func(s.addrs[i]);
    }
    return arr;
}
```

其中，`forEach` 类似 [函数式编程](https://en.wikipedia.org/wiki/Functional_programming) 中的 `map` 函数，由于结构体本身的封闭性，我们需要在 Handler 合约内增加一个 `public` 的包装函数，如下:

```solidity
function actorForEach(
    function(address) external returns (uint256) func
) public returns (uint256[] memory) {
    return _actors.forEach(func);
}
```

最后，我们修改 `WETHInvariants` 合约，增加并修改部分函数:

```solidity
function invariant_solvencyDeposits() public {
    uint256[] memory balanceArray = handler.actorForEach(weth.balanceOf);
    uint256 actorWETHSum = sum(balanceArray);
    assertEq(
        address(weth).balance,
        // handler.ghost_depositSum() - handler.ghost_withdrawSum()
        actorWETHSum
    );
}

function sum(uint256[] memory uintList) internal pure returns (uint256) {
    uint256 sumResult = 0;
    for (uint256 i; i < uintList.length; ++i) {
        sumResult += uintList[i];
    }
    return sumResult;
}
```

其中，`sum` 函数用于聚合 `actorForEach` 产生的结果。此处使用了函数式编程的思路，在一定程度上增加了代码的可重用性。

读者可能发现我们在 Handler 合约内增加了 `actorForEach` 函数，此函数可能会被随机调用，但显然该函数仅是一个辅助函数，为了避免这一问题，我们需要引入一个函数用来限定 `invarient test` 的测试范围，此处我们仅使用了 `targetSelector` 函数，事实上，Foundry 提供了包括合约、发生者限定的多个限定函数，具体请参考 [Invariant Test Helper Functions](https://book.getfoundry.sh/forge/invariant-testing#invariant-test-helper-functions) 文档。

在 `WETHInvariants` 的 `setUp` 函数内增加以下代码:

```solidity
function setUp() public {
    weth = new WETH9();
    handler = new Handler(weth);

    bytes4[] memory selectors = new bytes4[](3);
    selectors[0] = Handler.deposit.selector;
    selectors[1] = Handler.withdraw.selector;
    selectors[2] = Handler.sendFallback.selector;

    targetSelector(FuzzSelector({
        addr: address(handler),
        selectors: selectors
    }));

    excludeSender(address(weth));

    targetContract(address(handler));
}
```

请注意，此处也使用了 `excludeSender` 辅助函数，该函数用于排除某些 `msg.sender` ，我们认为 WETH 合约不可能自操作，所以排除了此地址。

> 这也是为了避免 BUG 的发生，如果读者自行研究过 `addActor` 函数，会发现我们直接使用 `deal(msg.sender, amount);` 来初始化资金，一旦 `msg.sender` 为 WETH 合约，那么就意味着 WETH 合约的 ETH 余额会被 `deal` 修改。在编写这部分代码时，我遇到了这一问题，由于该 BUG 与输入有关，所以我进行了较长时间的 DEBUG。

到目前为止，我们都仅测试了部分函数而没有测试 `transfer` 等函数，我们可以继续设计 Handler 合约中的包装函数使其纳入我们的测试范围:

```solidity
function approve(uint256 amount, uint256 approverSeed) public useActor(amount) {
    address approver = _actors.rand(approverSeed);
    weth.approve(approver, amount);
}

function transfer(uint256 amount, address receiver) public useActor(amount) {
    amount = bound(amount, 0, weth.balanceOf(msg.sender));
    if (!_actors.saved[receiver]) {
        _actors.addrs.push(receiver);
        _actors.saved[receiver] = true;
        weth.transfer(receiver, amount);
    } else {
        weth.transfer(receiver, amount);
    }
}

function transferFrom(uint256 amount, uint256 senerSeed, uint256 receiverSeed) public {
    address receiver = _actors.rand(receiverSeed);
    address sender = _actors.rand(senerSeed);
    amount = bound(amount, 0, weth.balanceOf(sender));
    weth.transferFrom(sender, receiver, amount);
}
```

此处考虑到 `transfer` 可能将自身资产转移给一个新的 `actor` 所以使用了 `if-else` 结构进行判断。此处的 `transferFrom` 写的比较简单，没有考虑很多情形，事实上，读者应该建立一个保存 `approve` 关系的数据结构以方便后文测试。
 
然后，我们也需要修正测试范围:

```solidity
bytes4[] memory selectors = new bytes4[](6);
selectors[0] = Handler.deposit.selector;
selectors[1] = Handler.withdraw.selector;
selectors[2] = Handler.sendFallback.selector;
selectors[3] = Handler.approve.selector;
selectors[4] = Handler.transfer.selector;
selectors[5] = Handler.transferFrom.selector;
```

最后，我们讨论一个较为特殊的情况，即使用 `selfdestruct` 发生 ETH 的情况。该情况较为特殊，而且目前以太坊官方准备废弃此操作码。读者阅读本文时，可能该操作码已被废弃。此操作码会在上海升级中移除，具体可以参考 [EIP-4758](https://eips.ethereum.org/EIPS/eip-4758) 和 [Ethereum Magicians forum](https://ethereum-magicians.org/t/eip-4758-deactivate-selfdestruct/8710) 上的讨论。

为了进行此测试，我们需要定义一个用于自毁发送 ETH 的合约 `ForcePush` ，该合约可以直接定义在 Handler.sol 中:

```solidity
contract ForcePush {
    constructor(address dst) payable {
        selfdestruct(payable(dst));
    }
}
```

然后，我们在 `Handler` 合约内加入以下函数:

```solidity
function forcePush(
    uint256 amount
) public {
    amount = bound(amount, 0, MAX_TRANSFER);
    deal(address(this), amount);
    new ForcePush{ value: amount }(address(weth));
}
```

最后，我们需要在 `WETHInvariants` 合约的 `setUp` 函数中加入此函数的 `selectors`。代码如下:

```solidity
bytes4[] memory selectors = new bytes4[](7);
selectors[0] = Handler.deposit.selector;
selectors[1] = Handler.withdraw.selector;
selectors[2] = Handler.sendFallback.selector;
selectors[3] = Handler.approve.selector;
selectors[4] = Handler.transfer.selector;
selectors[5] = Handler.transferFrom.selector;
selectors[6] = Handler.forcePush.selector;
```

运行测试后，我们发现测试失败。使用 `forge -vvv` 显示测试过程中的函数调用关系，我们发现直接引起测试失败的是以下交易:

```solidity
 [49972] Handler::forcePush(530046953286835462105160314255)
    ├─ [0] console::log(Bound Result, 6953286835462105160270085) [staticcall]
    │   └─ ← ()
    ├─ [0] VM::deal(Handler: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], 6953286835462105160270085)
    │   └─ ← ()
    ├─ [7813] → new <Unknown>@0xffD4505B3452Dc22f8473616d50503bA9E1710Ac
    │   └─ ← 0 bytes of code
    └─ ← ()
```

显然，该笔交易调用了 `forcePush` 函数直接将 ETH 推入了 WETH ，但是该笔 ETH 汇入没有触发 WETH 铸造函数，导致没有新的 WETH 铸造，出现了 `address(weth).balance` 大于 `actorWETHSum` 的情况。

为了进一步验证此结果，我们决定在 Handler 中增加一个幽灵变量 `ghost_forcePushSum` ，并对 `forcePush` 进行以下修改:

```solidity
function forcePush(
    uint256 amount
) public {
    amount = bound(amount, 0, MAX_TRANSFER);
    deal(address(this), amount);
    new ForcePush{ value: amount }(address(weth));
    ghost_forcePushSum += amount;
}
```

对 `WETHInvariants` 合约中的 `invariant_solvencyDeposits` 中的断言进行如下修正:

```solidity
assertEq(
    address(weth).balance - handler.ghost_forcePushSum(),
    actorWETHSum
);
```

测试可以顺利通过。

## 形式化证明

`solidity` 语言通过了良好的形式化语言证明支持，而 Foundry 在不久前刚刚跟进了此配置。但根据 solidity 开发组的 [2022 年调查](https://blog.soliditylang.org/2023/03/10/solidity-developer-survey-2022-results/) ，约有 81% 的开发者没有听说过此技术。本文希望可以尽可能推广这一技术。

`solidity` 形式化证明工具有一个较小的依赖，即 `z3` 。这是由微软开发的求解工具，性能较高，且与大部分工具不同，微软团队提供了预编译产物并在各大 Linux 平台下都有相应的包可以安装。读者可前往 [z3](https://github.com/Z3Prover/z3) 的 github 仓库直接下载预编译产物。

下文可能使用一些神奇操作，如果您感觉此操作比较蠢可以考虑直接通过源代码编译安装。

如果您与我一样使用 Ubuntu 则可以使用以下命令安装:

```bash
apt-get install libz3-4
```

安装完成后，请前往 `/lib/x86_64-linux-gnu/` 内运行以下命令:

```bash
ls | grep "libz3.so"
```

如果安装成功，则返回以下内容:

```bash
libz3.so.4
```

但此处有一个神奇的地方，即 ubuntu 安装后的名字与正常情况不符，Solidity 需要 `libz3.so.4.8` 作为链接库。但事实上，`libz3.so.4` 和 `libz3.so.4.8` 是一致的，所以我们直接进行改名。命令如下:

```bash
mv libz3.so.4 libz3.so.4.8
```

再次运行 `ls | grep "libz3.so"` 获得的结果为 `libz3.so.4.8` 。

> 上述内容均假设用户为 `root` 用户。

限于本文篇幅，笔者可能不会给出对形式化证明的完整介绍，我们会在为了推出文章详细介绍这一主题。本文只是浅尝辄止的分析其中的一些内容。

目前来看，形式化证明在社群内讨论较少，本文无法找到现实中的最佳实践，本文中给出的内容均为笔者的一家之言，如果未来出现较官方推荐的最佳实践，本文会及时更新。

在开始编写测试前，我们需要理解什么是形式化证明？正如前文所言，我们可以将 solidity 函数抽象为一个逻辑关系，使用 `fuzz test` 可以对部分变量进行测试，而形式化证明则会对全集进行测试。可能有读者好奇，全集是庞大的，进行遍历式搜索显然不可能，一个合适的方法是使用一些特殊算法实现这一功能，上文安装的 `z3` 就是一个这样的库。

更加详细的说，`solidity` 语言的编译器除了支持将 `solidity` 编译为 EVM 字节码，也支持将 `solidity` 编译成 SMT 表达式，以下给出了一个例子:

![Solidity SMT](https://img.gejiba.com/images/876fb2f900ac79ab81ee2c58ca2ed13e.png)

关于其具体的编译逻辑，我们不再进行详细讨论。如果读者对这一部分感兴趣，可以参考 [SMT-based Verification of Solidity Smart Contracts](https://github.com/chriseth/solidity_isola/blob/master/main.pdf) 论文。

当然，读者也可根据 Solidity 代码的含义自行编写 SMT 表达式，这样自由度更高。关于手写 SMT 表达式的相关内容，我们未来可能会进行专题讲解，目前读者可以参考 [Formally Verifying The World’s Most Popular Smart Contract](https://www.zellic.io/blog/formal-verification-weth) 此文。

而有了此 SMT 表达式，我们就可以使用 SMT 求解器进行求解，而 `z3` 是一个高性能的 SMT 求解器。

solidity 使用的 SMT 求解器可以自动分析以下问题:

1. 算术溢出(由于较新版本的 Solidity 在语言层面内置了此功能，所以在 Solidity >= 0.8.7 版本后不会进行此问题的分析)
2. 除 0 错误
3. 不可达语句(即因为 `if-else` 条件配置错误而导致无法被访问的代码)
4. 空数组弹出
5. 越界访问
6. 转账金额不足

上述问题会被自动分析，我们不需要额外的代码编写，但需要额外的配置。

我们以算术溢出为例向大家展示如何使用该工具。我们首先写出一个案例合约 `overflow.sol` ，该合约应放在 `src` 文件内，内容如下:

```
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.8.0;

contract Overflow {
    uint public x;
    uint public y;

    constructor(uint x_, uint y_) {
        (x, y) = (x_, y_);
    }

    function add(uint x_, uint y_) public pure returns (uint) {
        return x_ + y_;
    }
}
```

对 `foundry.toml` 进行配置以启用此功能。配置如下:

```toml
[profile.default.model_checker]
contracts = {'src/overflow.sol' = ['Overflow']}
engine = 'chc'
timeout = 20000
targets = ['overflow']
```

其中各参数含义如下:
- `contracts` 指待测试目标合约
- `engine` 指测试引擎，具体参数会在后文讨论，
- `timeout` 指超时时间，部分 SMT 表达式可能长时间无法被求解，所以需要设置此参数中止求解
- `targets` 测试目标，此处为 `overflow` 上溢出

在命令行内运行 `forge build` ，结果如下:

```bash
warning[4984]: Warning: CHC: Overflow (resulting value larger than 2**256 - 1) happens here.
Counterexample:
x = 0, y = 0
x_ = 1
y_ = 115792089237316195423570985008687907853269984665640564039457584007913129639935
 = 0

Transaction trace:
Overflow.constructor(0, 0)
State: x = 0, y = 0
Overflow.add(1, 115792089237316195423570985008687907853269984665640564039457584007913129639935)
  --> src/overflow.sol:13:16:
   |
13 |         return x_ + y_;
   |                ^^^^^^^
```

显然，我们的溢出被检查到了。在此处，我们一并给出各问题对应的 `targets`，如下:

- `underflow` 和 `overflow` 均为数据溢出
- `divByZero` 除 0 异常
- `constantCondition` 不可达语句
- `popEmptyArray` 空数组弹出
- `outOfBounds` 索引越界
- `default` 测试所有情况

我们再演示一个 `divByZero` 测试，首先在 `foundry.toml` 内增加:

```toml
targets = ['overflow', 'divByZero']
```

修改 `Overflow` 合约，增加:

```solidity
function divide(uint x_, uint y_) public pure returns (uint) {
    return x_ / y_;
}
```

运行 `fogre build` 获得了一个额外结果:

```bash
warning[4281]: Warning: CHC: Division by zero happens here.
Counterexample:
x = 0, y = 0
x_ = 0
y_ = 0
 = 0

Transaction trace:
Overflow.constructor(0, 0)
State: x = 0, y = 0
Overflow.divide(0, 0)
  --> src/overflow.sol:17:16:
   |
17 |         return x_ / y_;
   |                ^^^^^^^

```

由此可见对于一些常见的数学问题，SMT求解器可以较快的找出错误。

最后，我们开始引入一个最特殊的检验目标，即 `assert`。`assert` 可以实现较为复杂的形式化证明，这也是唯一个需要开发者介入而非自动化的形式化证明。

我们首先给出一个例子，假如我们需要对以下错误的函数进行形式化证明:

```solidity
function errorTansfer(address dst, uint256 wad) public {
    require(balanceOf[msg.sender] > wad);
    uint256 srcBalance = balanceOf[msg.sender];
    uint256 dstBalance = balanceOf[dst];
    balanceOf[msg.sender] = srcBalance - wad;
    balanceOf[dst] = dstBalance + wad;
}
```

我们需要对其进行一些改写以使其符合形式化证明的格式，读者可以建立 `test\SMTChecker` 文件夹，并创建 `formalVerify.sol` 合约文件，写入以下内容:

```solidity
contract FormalVerify {

    mapping(address => uint256) public balanceOf;

    function errorTansfer(address dst, uint256 wad, uint256 initBalance) public {
        balanceOf[msg.sender] = initBalance;
        require(balanceOf[msg.sender] > wad);

        uint256 srcBalance = balanceOf[msg.sender];
        uint256 dstBalance = balanceOf[dst];

        uint256 sumBefore = srcBalance + dstBalance;

        balanceOf[msg.sender] = srcBalance - wad;
        balanceOf[dst] = dstBalance + wad;

        
        uint256 sumAfter = balanceOf[msg.sender] + balanceOf[dst];

        assert(sumBefore == sumAfter);
    }   
}
```

注意形式化证明的测试需要引入一些合约逻辑外的代码，我们建议直接创建一个合约专门用于测试。而且我们目前使用的 CHC 引擎不支持合约外函数导入，所以此处我们需要构造一个最近可运行合约，此处引入了 `mapping(address => uint256) public balanceOf;` 语句。我们也需要进行一些初始化，所以此处引入了 `initBalance` 变量对 `msg.sender` 的余额进行初始化。读者可能还记得我们需要论证的数学关系，简单来说，就是转账前和转账后，`msg.sender` 和 `dst` 的余额之和相同，在此处我们构造 `sumBefore` 代表转帐前余额之和而 `sumAfter` 代表转账后余额。

此处引入了 `require` 和 `assert`，前者在正常合约内用于检验变量是否符合某些标准，而在形式化证明中，该函数表示限定输入范围。如果没有此 `require` 函数，形式化证明过程中会尝试测试 `uint256` 的全集，这是浪费时间的且与实际逻辑不符，此处我们限定 `balanceOf[msg.sender] > wad` 。

> 注意此处仅限制了 `wad` 的取值范围，而 `dst` 和 `initBalance` 仍会全集搜索。一般来说，形式化证明中的 `require` 可以直接保留原合约的 `require` 即可。

`assert` 则用于最后的判断，如果不符合 `sumBefore == sumAfter` 的条件，则会抛出异常。

完成上述合约后，我们修正 `foundry.toml`，修正后如下:

```toml
[profile.default.model_checker]
contracts = {'test/SMTChecker/formalVerify.sol' = ['FormalVerify']}
engine = 'chc'
timeout = 20000
targets = ['assert']
```

运行 `forge build` 得到如下结果:

```bash
dst = 0x0
wad = 1
initBalance = 20978
srcBalance = 20978
dstBalance = 20978
sumBefore = 41956
sumAfter = 41958

Transaction trace:
FormalVerify.constructor()
FormalVerify.errorTansfer(0x0, 1, 20978){ msg.sender: 0x0 }
  --> test/SMTChecker/formalVerify.sol:25:3:
   |
25 |            assert(sumBefore == sumAfter);
   |            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

```

该形式化证明成功发现了我们代码的错误。

> 此处读者就会发现，如果需要编写形式化证明合约，我们就需要动原合约代码，但是显然直接在原合约上做形式化证明可能会被写合约的同学打死，所以此处我还是建议大家使用一个新合约专门进行形式化证明。

此处，我们应该简单讨论关于求解引擎的问题，在 `solidity` 中，支持两种求解引擎:

- CHC 上述案例一直使用的求解引擎，该引擎在外部函数调用方面存在一点问题，该引擎会视外部调用函数具有任意的转化能力，即外部函数可以将参数进行任意转化。在此前提下，CHC 推导 `assert` 是否正确。
- BMC 性能较好，但具有一定的误报率，适合快速使用。该引擎会将外部函数调用内化，尊重外部函数作用。但如果读者根据此教程可能会发现无法使用此引擎，具体原因未知。

可能有读者好奇 CHC 在不知道函数功能情况下如何进行判断 `assert` 条件呢？假如目前有一个函数 `f()`，其功能未知，但我们仍可以基于 `a = b` 的前提推测出 `f(a) = f(b)` ，CHC 引擎也是如此工作的。

> 这类似数学考试中屡见不鲜的抽象函数问题，抽象函数没有具体定义，但我们仍可以进行解题。

笔者目前使用形式化证明的经验并不是很多，但已经明显发现形式化证明存在一些比较严重的问题，对于较大范围的函数证明很容易无法产生证明结果。就目前而言，在较大范围内的形式化证明上，使用 solidity 自带的形式化证明工具往往无能为力，会返回以下错误:

```bash
warning[5840]: Warning: CHC: 1 verification condition(s) could not be proved. Enable the model checker option "show unproved" to see all of them. Consider choosing a specific contract to be verified in order to reduce the solving problems. Consider increasing the timeout per query.bash
```

对于这种整体的形式化证明，我建议手写相关逻辑的 SMT 表达式并进行求解，而 solidity 自带的形式化证明工具用以证明较小函数的正确性。

## 总结

本文主要介绍了以 foundry 为基础的三种测试方法:

- fuzz testing
- invarient testing
- 形式化证明

其中，`fuzz testing` 和形式化证明对于单个函数的证明都较为友好，而 `invarient testing` 则适用于复杂情况的测试。在介绍 `invarient testing` 的过程中，我们引入了以 Handler 为基础的测试方法，并介绍了 Actor 的测试方法。

本文仅给出了以 WETH 为例子的各类型测试，对于形式化证明的分析较为简单。在未来的博客中，我们较聚焦于以下问题:

1. 以 `invarient tesing`  为基础的更多的漏洞测试案例
2. 手写形式化证明表达式以解决整体合约验证问题