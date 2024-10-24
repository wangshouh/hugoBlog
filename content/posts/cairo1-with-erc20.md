---
title: "Cairo 实战入门:编写测试部署ERC-20代币智能合约"
date: 2023-07-06T11:47:33Z
tags: [cario,ERC-20]
math: true
---

## 概述

Cairo 是 ZK Rollup 的领域专用语言，目前仅用于 [StarkNet](https://www.starknet.io/en) 项目。随着 Rollup 叙事的发展，我们认为 cairo 在未来一定会成为智能合约开发的核心语言。

本文类似我之前编写的 [Foundry教程：编写测试部署ERC-20代币智能合约](https://blog.wssh.trade/posts/foundry-with-erc20/) ，介绍了使用 cairo 1 v2 版本(该版本也可称为 `Cairo 2`) 进行编程、测试和部署的全流程。由于缺乏易用工具，本文放弃了本地测试网部署。

本文仅使用 Rust 的部分基础语法，并进行了详细说明，所以读者可以没有 rust 开发基础，但如果读者熟悉 rust 基础语法，那么阅读代码会更加容易。

本文的重点在于介绍 cairo 1 的合约语法部分，理论上，rust 开发者阅读完本文后，就可以熟练编写基础 cairo 1 合约。

本文的部分内容为 solidity 与 cairo 的对比，如果读者不熟悉 solidity 可以直接跳过。由于笔者对 rust 了解不多，所以本文没有给出 cairo 与 rust 的对比。

值得注意的是，笔者没有详细介绍 ERC20 各函数的功能，读者可以参考 [EIP 文档](https://eips.ethereum.org/EIPS/eip-20) 或者 [SNIP 文档](https://github.com/starknet-io/SNIPs/blob/main/SNIPS/snip-2.md) 。
 
## 安装

在开始进行 Cairo 编程前，我们需要安装准备相关环境。笔者使用的是 WSL Ubuntu 22.04 系统。但事实上，使用 macOS 也可达到相同的开发体验。

本文使用了 Cairo 1 语言，相比于大量依赖于 Python 的 Cairo 0 语言，Cairo 1 语言相关的开发工具基本都使用了 Rust 。这意味着我们可以通过直接下载编译后的二进制安装包进行安装。

在开发环境部署上，我们主要依赖于 Cairo 包管理器 [scarb](https://docs.swmansion.com/scarb)

Scarb的安装方法与 cairo 基本一致，读者可以参考 [文档](https://docs.swmansion.com/scarb/download)。使用以下命令测试安装是否成功:

```bash
scarb -V
```

> 此处存在一个 scarb 自带的 cairo 版本与 starknet 区块链支持版本的差异问题，读者可以通过 [Starknet environments](https://docs.starknet.io/documentation/starknet_versions/version_notes/) 确定当前测试网和主网支持的版本，可以在 [Scarb Releases](https://github.com/software-mansion/scarb/releases) 中查看 scarb 自带的 cairo 版本

最后，我们安装 `vscode` 中的开发插件，值得注意的是，目前开发插件已可以通过直接在 VSCode 拓展市场下载，用户可前往 [此链接](https://marketplace.visualstudio.com/items?itemName=starkware.cairo1) 下载。

安装完成后，进入插件的设置页面，如下图:

![Cairo1 Setting](https://acjgpfqbqr.cloudimg.io/_img1_/f9e7827a82197e71da78e63f0a865274.png)

在插件设置页面内，在 `Language Server Path` 内填入 `cairo-language-server` 二进制文件地址，可以使用 `which cairo-language-server` 命令获得。在 `Scarb Path` 内填入 `scarb` 二进制文件地址，可以使用 `which scarb` 命令获得。完成上述设置后，请重启 VSCode 软件。

一个示例配置如下(请勿直接抄写文件地址):

![Cairo Setting Example](https://acjgpfqbqr.cloudimg.io/_img1_/f0564492611f17938f4170eb697b9823.png)

## Cairo vs. Solidity

考虑到本文大部分读者具有 `solidity` 编程背景，本文将梳理 `EVM` 与 `cairoVM` 的区别。本节内容对于 Cairo 0 的开发者而言有阅读必要，但对于 Cairo 1 的开发者而言，理论上可以跳过本文。

在数据类型方面，事实上，EVM 的原生数据类型仅有 `uint256` ，其他类型都是由 solidity 编程器在编译过程中实现的。

而在 cairo 中，原生数据类型仅有 `felt` 类型，读者可简单认为该类型为 `uint252`。需要注意的是，该类型定义在 **有限域** 上，更加准确的定义为 $0 \leq x < P$ ，而 $P = 2^{251}+17 \cdot 2^{192} + 1$ 。其他数据类型都是由 `corelib` 标准库和编译器实现的。

与 solidity 提供的常规计算机代数不同，cairo 的所有计算都定义在域上，简单来说，就是所有计算完成后都需要与 $P$ 进行模除。当然，这似乎与常规的计算机代数相同。但 `felt` 类型的除法是令人惊奇的。在 solidity 中，我们认为 `x / y` 的结果为 $\lfloor x / y \rfloor$ ，设 $x = 7$ 和 $y=3$ ，那么在 solidity 中计算结果为 2 ，但在 cairo 中，计算结果为 `1,206,167,596,222,043,737,899,107,594,365,023,368,541,035,738,443,865,566,657,697,352,045,290,673,496`

这是因为 cairo 对 `felt` 的除法做出了以下要求，设 $z = x / y$ ，那么 $z * y = x$ 是恒成立的。该保证使上述离谱结果的出现。更加详细的解释，请参考 [Field elements](https://www.cairo-lang.org/docs/how_cairo_works/cairo_intro.html#field-elements)。请读者在进行 `felt` 数据类型除法时注意。

在研究完基础数据类型后，我们需要考虑运行环境，众所周知，EVM 是一个基于栈的虚拟机，所有运算都发生在栈上，但 carioVM 则是直接在内存上进行计算。当然，cario 虚拟机也存在寄存器，但功能都较为底层，在正常开发时较少使用。如果读者对此感兴趣请参考 Cairo 0 的 [文档](https://www.cairo-lang.org/docs/how_cairo_works/cairo_intro.html#registers)。

在内存模型上，solidity 使用了可变稀疏内存，我们可以使用 `mstore` 等操作符在内存的任意位置写入数据，且可以在同一地址内进行覆写操作，但 cairo 的内存模型为不可变连续内存，这意味着我们不能在内存地址内任意写入数据且不能改变已写入的内存数据。该特性使编写循环结构变得几乎不可行，我们只能使用递归的方式编写循环。当然，Cairo 1 引入了 `loop` 结构也可以更好的实现循环效果，但对于 `for` 循环语句，目前仍未见到相关语法结构，可能暂未实现。

> 如果读者熟悉 [elixir](https://elixir-lang.org/) 等函数式编程语言，应该对于递归代替循环的编程逻辑较为熟悉。如果读者有空闲时间，可以考虑学习一下。

## Cairo 0 vs Cairo 1 vs Cairo 2

本文在编写时，Starknet [量子跃迁](https://medium.com/starkware/starknet-quantum-leap-major-throughput-improvements-are-here-3e4e294ad8cd) 已部署，使用 Cairo 2 编写的合约可以进行部署。考虑到读者不一定了解 Cairo 语言的发展历程且了解 Cairo 0 是有意义的，所以本节主要介绍 Cairo 0、Cairo 1 和 Cairo 2 之间的区别。

我们首先介绍 Cairo 1 和 Cairo 2 之间的区别，事实上，两者区别较小，只在部分语法上存在差异，其中差异最大的部分在于合约编程部分。正是此部分的破坏性更新导致 Cairo 1 升级为 Cairo 2。关于两者具体不同，请参考 [此文章](https://community.starknet.io/t/cairo-1-contract-syntax-is-evolving/94794)。

接下来，我们介绍 Cairo 0 与 Cairo 1 的区别。

在语法方面，`cairo 1` 与 Rust 语法几乎完全一致，但可能有部分语法由于 CairoVM 的限制无法实现。而 `cairo 0` 则与 `golang` 等语言类似。另一方面，Cairo 0 支持一些底层编程方法，允许开发者直接调整寄存器和内存。同时，cairo 0 要求开发者手动维护内存，没有自动的内存分配系统。一个并不恰当的类比是 Cairo 0 类似 huff 语言，而 cairo 2 类似 solidity 语言。

在本文编写时，`cairo 0` 并没有语法文档，只有官方提供的两个教程：

1. [Hello, Cairo](https://docs.cairo-lang.org/0.12.0/hello_cairo/index.html)
2. [How Cairo Works](https://docs.cairo-lang.org/0.12.0/how_cairo_works/index.html)

前者属于实战入门，而后者则是自底向上的分析。读者可根据自身爱好选择教程。我推荐读者阅读后者，因为后者涉及大量对 CairoVM 的底层分析，这些内容是不会随语言特性改变而改变的。

如果读者希望更加详细的了解 Cairo 1 和 Cairo 2 的语法，可以参考 [Cairo book](https://cairo-book.github.io/title-page.html) 。该文档是目前最为详细和系统的 Cairo 教程。读者也可参考以下文章:

1. [Starknet Cairo 101 Automated Workshop](https://github.com/starknet-edu/starknet-cairo-101/tree/main)
2. [A First Look at Cairo 1.0: A Safer, Stronger & Simpler Provable Programming Language](https://medium.com/nethermind-eth/a-first-look-at-cairo-1-0-a-safer-stronger-simpler-provable-programming-language-892ce4c07b38)
3. [The Starknet Book](https://book.starknet.io/index.html)
4. [Awesome Cairo](https://github.com/auditless/awesome-cairo) 该仓库给出了很多 cairo 1 的资源，建议参考

值得注意的是，上述资料部分仍使用了 cairo 1 语法，可能会在 Cairo 2 的编译环境内报错，请读者注意。上述资料都处于快速变化中，读者应随时参考官方最新动态。

在编译上，Cairo 1 引入了中间编译层，该表示层被称为 `Sierra` ，而最终的编译结果被称为 `casm` ，更多信息可以参考 [Under the hood of Cairo 1.0: Exploring Sierra](https://medium.com/nethermind-eth/under-the-hood-of-cairo-1-0-exploring-sierra-7f32808421f5) 。如下图:

![cairo 1 complie](https://acjgpfqbqr.cloudimg.io/_img1_/f1424e7997da290ddced38e7f9eb9595.png)

> solidity 的编译也是用了中间表示层方案，大家熟悉的 yul 即中间表示层

## Hello World 与测试

本节所有代码都可以在 [hello_erc20](https://github.com/wangshouh/helloERC20) github 仓库内找到。

在了解基本的 CairoVM 的基础知识后，我们进入真正的编程阶段。首先，我们需要初始化项目，使用以下命令初始化包:

```bash
scarb new hello_erc20
```

使用 `cd hello_erc20` 进入项目目录，我们可以看到以下目录结构:

```bash
.
├── Scarb.toml
└── src
    └── lib.cairo
```

打开 `lib.cairo` ，读者会看到如下代码:

```rust
fn main() -> felt252 {
    fib(16)
}

fn fib(mut n: felt252) -> felt252 {
    let mut a: felt252 = 0;
    let mut b: felt252 = 1;
    loop {
        if n == 0 {
            break a;
        }
        n = n - 1;
        let temp = b;
        b = a + b;
        a = temp;
    }
}

#[cfg(test)]
mod tests {
    use super::fib;

    #[test]
    fn it_works() {
        assert(fib(16) == 987, 'it works!');
    }
}
```

这是一个斐波那契数列计算函数，并附带了一个简单测试。

其中宏 `#[test]` 标识 `it_works` 为测试函数，`assert` 代表测试相等条件，`'it works!'` 为测试失败后的提示。

此处，我们主要需要讨论 `use super::fib;` ，这是一个路径导入语句，作用是将位于 `lib.cairo` 中的 `fib` 函数导入。我们可以看到 `tests` 的上一级模块内包含 `fib` 函数，所以此处使用了 `use super::fib;` 导入 `fib` 函数。

`cairo` 的导入与 Rust 有所不同。我们需要以编译器的视角看问题， cairo 编译器在启动编译后，进入 `src` 目录并记 `src` 目录名称为 `hello_erc20` 。然后进入 `lib.cairo` 文件寻找待编译文件。

根据上述流程，我们可以认为 `use super::fib` 等价于导入 `src/lib.cairo` 中的 `fib` 作用域，所以此处我们可以将 `tests` 中的 `use super::fib;` 替换为 `use hello_erc20::fib;`。可能有读者不理解 `use` 关键词含义，该关键词会将 `super::fib` 导入作用域，然后我们可以直接调用 `fib` 函数。值得注意的是，`use` 不止可以导入函数，也可以导入一个模块，我们会在后文进行展示。

> 如果读者无法理解，请继续阅读，我会对后文每一个路径导入进行详细分析。当然，读者也可以尝试分析 [quaireaux](https://github.com/keep-starknet-strange/quaireaux/tree/main) 复杂项目的路径导入问题，如果读者可以理解 `quaireaux` 的路径导入，那么就基本可以理解大部分项目的路径导入方法。
> 
> 此部分最好的学习材料是 [Cairo Book Chapter 7](https://book.cairo-lang.org/ch07-00-managing-cairo-projects-with-packages-crates-and-modules.html)，建议读者参考

我们可以使用 `scarb cairo-run` 运行 `main` 函数，输出如下:

```bash
   Compiling hello_erc20 v0.1.0 (hello_erc20/Scarb.toml)
    Finished release target(s) in 1 second
     Running hello_erc20
Run completed successfully, returning [987]
```

也可以使用 `scarb test` 执行项目内的所有测试，输出如下:

```bash
     Running cairo-test hello_erc20
   Compiling test(hello_erc20_unittest) hello_erc20 v0.1.0 (hello_erc20/Scarb.toml)
    Finished release target(s) in 2 seconds
testing hello_erc20 ...
running 1 tests
test hello_erc20::tests::it_works ... ok (gas usage est.: 46860)
test result: ok. 1 passed; 0 failed; 0 ignored; 0 filtered out;
```

我们可以看到命令行内也输出了函数的 gas 消耗。

完成上述流程后，我们希望使我们的系统更加模块化，我们希望将所有的测试放在 `tests` 文件夹内，我们首先创建 `tests` 文件夹，并创建同名的 cairo 文件 `tests.cairo` 。我们需要在 `tests` 文件夹内创建用于斐波那契数列测试的 `fib_test.cairo`，并且在 `tests.cairo` 中键入以下内容:

```cairo
mod fib_test;
```

正如前文所述，`tests.cairo` 是一个测试入口文件，我们使用 `mod fib_test;` 在此文件内标识待测试文件。我们可以认为 `mod fib_test;` 相当于告诉测试工具请将 `tests/fib_test.cairo` 文件中的测试函数运行。更加正式的说，`mod fib_test;` 的作用是声明模块。

我们将 `lib.cairo` 中的测试代码进行转移，如下:

```rust
use hello_erc20::fib;

#[test]
fn it_works() {
    assert(fib(16) == 987, 'it works!');
}
```

此时 `lib.cairo` 内的测试部分需要修改为:

```rust
#[cfg(test)]
mod tests;
```

此处更改会将 `tests.cairo` 引入 `lib.cairo` ，这样编译器和测试工具都可以编写和运行测试函数。再次运行 `scarb test` 命令，我们发现编译器可以寻找到所有的测试并且完成测试。在之后，我们增加测试时，会在 `tests` 内编写测试代码，并修改 `tests.cairo` 文件增加测试声明。

在 Hello World 的最后，我们进一步模块化项目，我们将 `main` 函数分离出来，删除 `lib.cairo` 中的 `main` 函数，并增加 `mod main;` 声明，最终 `lib.cairo` 的构成如下:

```rust
mod main;
fn fib(mut n: felt252) -> felt252 {
    let mut a: felt252 = 0;
    let mut b: felt252 = 1;
    loop {
        if n == 0 {
            break a;
        }
        n = n - 1;
        let temp = b;
        b = a + b;
        a = temp;
    }
}

#[cfg(test)]
mod tests;
```

创建 `src/main.cairo` 并写入以下内容:

```rust
use hello_erc20::fib;

fn main() {
    println!("FIB16 is {}", fib(16))
}
```

此处引入了 `println!` 用于终端打印输出结果。

运行 `scarb cairo-run` 命令，输出如下:

```bash
FIB16 is 987
Run completed successfully, returning []
```

我们可以看到输出了结果 `987` 。

如果读者对底层感兴趣，可以尝试使用 `scarb cairo-run --print-full-memory` 此处使用 `--print-full-memory` 可以打印出内存结构。之前介绍 `cairoVM` 时，我们已经支出 cairoVM 的内存结构是不可变的，所以我们可以根据运行结束后的内存情况来推测运行过程中的事件。当然，直接输出的内存可能很难读懂，如果读者经过 cairo 0 的相关训练，可能可以读懂一部分。

## ERC20 合约编程

关于 `cairo` 智能合约编程最为核心文档是 [Cairo Contracts](https://github.com/starkware-libs/cairo/blob/main/docs/reference/src/components/cairo/modules/language_constructs/pages/contracts.adoc) 和 [Cairo book](https://book.cairo-lang.org/title-page.html)，请读者务必阅读这两份文档内容。本文的 ERC20 代币合约主要参考了 [starkware 官方实现](https://github.com/starkware-libs/cairo/blob/main/crates/cairo-lang-starknet/cairo_level_tests/contracts/erc20.cairo)。需要注意的是，starknet 已有 ERC20 代币规范被称为 [SNIP 2](https://github.com/starknet-io/SNIPs/blob/main/SNIPS/snip-2.md) 。

本文主要基于 solmate 版本的 [ERC20 智能合约](https://github.com/transmissions11/solmate/blob/main/src/tokens/ERC20.sol) 的具体逻辑。

在了解基础的知识后，我们开始进行 ERC20 代币合约编程。在 `src` 下创建 `ERC20.cairo` ，并在 `src/tests` 下创建 `ERC20_test.cairo`。最终，目录结构如下:

```
.
├── Scarb.toml
├── src
│   ├── ERC20.cairo
│   ├── lib.cairo
│   ├── main.cairo
│   ├── tests
│   │   ├── ERC20_test.cairo
│   │   └── fib_test.cairo
│   └── tests.cairo
```

请读者在 `tests.cairo` 中写入以下内容:

```rust
mod fib_test;
mod ERC20_test;
```

由于我们需要在 `tests` 文件夹内引入 `ERC20` 合约，所以我们需要对 `lib.cairo` 进行修改，请加入以下内容:

```rust
mod ERC20;
```

> 如果读者感觉上述初始化云里雾里，请参考 [github 仓库](https://github.com/wangshouh/helloERC20)

最后，我们需要配置 `Scarb.toml` 使其支持智能合约，请写入以下配置:

```toml
[package]
name = "hello_erc20"
version = "0.1.0"
edition = "2023_10"

# See more keys and their definitions at https://docs.swmansion.com/scarb/docs/reference/manifest.html

[dependencies]
starknet = ">=2.4.0"
```

完成上述任务后，我们开始编写 ERC20 合约，我们使用了编写和测试的逻辑，编写完部分函数后就会立即进行测试，所以后文代码中的编写和测试会交替出现，请读者仔细观察。

我们首先给出类似接口的定义的 `trait` 定义，如下:

```rust
use starknet::ContractAddress;

#[starknet::interface]
trait IERC20<TContractState> {
    fn name(self: @TContractState) -> felt252;
    fn symbol(self: @TContractState) -> felt252;
    fn decimals(self: @TContractState) -> u8;
    fn total_supply(self: @TContractState) -> u256;
    fn balanceOf(self: @TContractState, account: ContractAddress) -> u256;
    fn allowance(self: @TContractState, owner: ContractAddress, spender: ContractAddress) -> u256;
    fn transfer(ref self: TContractState, to: ContractAddress, amount: u256) -> bool;
    fn transferFrom(
        ref self: TContractState, from: ContractAddress, to: ContractAddress, amount: u256
    ) -> bool;
    fn approve(ref self: TContractState, spender: ContractAddress, amount: u256) -> bool;
    fn mint(ref self: TContractState, amount: u256);
}
```

此处引入了 `starknet::ContractAddress` 数据类型，该数据类型表示地址，类似 solidity 中的 `address` 类型。此接口参考了 [SNIP 2](https://github.com/starknet-io/SNIPs/blob/main/SNIPS/snip-2.md) 中的内容。此处，我们使用了 `TContractState` 类型，该类型代表合约的状态空间(或简单认为是合约的存储空间)。我们可以看到使用了 `self: @TContractState` 和 `ref self: TContractState` 两种不同的参数。其中 `@TContractState` 象征当前合约状态的快照(`snapshots`)，这是一种不可变视图，使用此类型意味着该函数不能改变合约的状态，即该函数不可以向合约内写入数据。而 `ref self: TContractState` 代表可变引用，使用此类型的函数具有改变合约状态的能力，即函数可以向合约内写入数据。显然，`total_supply` 等函数只需要读取合约状态而不需要改变合约状态，所以我们使用了 `self: @TContractState`，而 `mint` 等函数显然会改变合约状态(进行用户余额的增加)，所以使用了 `ref self: TContractState` 参数。

> 关于 `snapshots` 与 `ref` 的不同，读者可以参考 [References and Snapshots](https://cairo-book.github.io/ch03-02-references-and-snapshots.html)

接下来，我们进行合约主体的开发，我们首先定义合约内的存储，如下:

```rust
#[starknet::contract]
mod ERC20 {
    use starknet::get_caller_address;
    use starknet::ContractAddress;
    #[storage]
    struct Storage {
        _name: felt252,
        _symbol: felt252,
        _decimals: u8,
        _total_supply: u256,
        _balances: LegacyMap::<ContractAddress, u256>,
        _allowances: LegacyMap::<(ContractAddress, ContractAddress), u256>,
    }
}
```

此处的 `#[starknet::contract]` 告知编译器以下模块 `ERC20` 为合约模块，需要编译时特殊处理。

我们使用 `use` 导入的各模块的作用如下:

1. `get_caller_address` 导入获取请求者地址的模块，类似 solidity 中的 `msg.sender`
1. `ContractAddress` 导入 `starkNet` 地址类型

结构体 `Storage` 是一类特殊的结构体，声明在此结构体内的变量会被写入存储。此处需要注意 `LegacyMap` 数据类型，此数据类型类似 `solidity` 中的 `mapping` 映射类型。在上述存储变量中，最难理解的是 `_allowances` ，在 solidity 中，该变量一般定义如下:

```solidity
mapping(address => mapping(address => uint256)) public allowance;
```

上述多重映射在 cairo 并不好表达，所以此处使用了元组 `(ContractAddress, ContractAddress)` 与 `uint256` 的映射关系。

> cairo 原生不支持 uint256 类型，仅支持 felt252 类型，uint256 本质上是由 felt128 拼接获得的。此处使用 uint256 是为了保持兼容性。

完成上述定义后，我们开始定义事件，如下:

```rust
#[event]
#[derive(Drop, PartialEq, starknet::Event)]
enum Event {
    Transfer: Transfer,
    Approval: Approval,
}
#[derive(Drop, PartialEq, starknet::Event)]
struct Transfer {
    #[key]
    from: ContractAddress,
    #[key]
    to: ContractAddress,
    value: u256,
}
#[derive(Drop, PartialEq, starknet::Event)]
struct Approval {
    #[key]
    owner: ContractAddress,
    #[key]
    spender: ContractAddress,
    value: u256,
}
```

最后，我们利用 `#[event]` 宏声明了两个事件。此处需要在 `enum Event` 枚举类型内写入合约内所有 event 的名字。接下来，我们使用结构体具体定义了 event 包含的数据，此处可以使用 `#[key]` 标识可检索变量，类似 solidity 中的 `index` 关键词。在此处，我们也使用了 `#[derive(Drop, PartialEq, starknet::Event)]` 宏为 event 增加了一些接口的默认实现，此处的 `PartialEq` 是为后文进行测试准备的。

> `PartialEq` 可以为事件结构体自动衍生一个相等判断属性，这意味着我们后面可以使用 `assert` 进行测试断言

接下来，我们编写构造器和基础的 `view` 函数，如下:

```rust
    #[constructor]
    fn constructor(ref self: ContractState, name: felt252, symbol: felt252, decimals: u8, ) {
        self._name.write(name);
        self._symbol.write(symbol);
        self._decimals.write(decimals);
    }
```

构造器是合约初始化函数，我们在此处对 ERC20 代币的基本参数进行初始化，显然构造器函数需要对合约状态进行改变，所以此处我们使用了 `ref self: ContractState` 参数。在此处，我们也使用了 `self._name.write` 形式的函数对合约内的变量进行了写入。

接下来，我们编写 `view` 函数，如下:

```rust
#[external(v0)]
impl IERC20Impl of super::IERC20<ContractState> {
    fn name(self: @ContractState) -> felt252 {
        self._name.read()
    }

    fn symbol(self: @ContractState) -> felt252 {
        self._symbol.read()
    }

    fn decimals(self: @ContractState) -> u8 {
        self._decimals.read()
    }

    fn total_supply(self: @ContractState) -> u256 {
        self._total_supply.read()
    }

    fn balanceOf(self: @ContractState, account: ContractAddress) -> u256 {
        self._balances.read(account)
    }

    fn allowance(
        self: @ContractState, owner: ContractAddress, spender: ContractAddress
    ) -> u256 {
        self._allowances.read((owner, spender))
    }
}
```

该部分也较为简单，基本都是 `read` 读取操作。此处，我们对上文定义的 `super::IERC20<ContractState>` 进行了实现。此处允许 `IERC20Impl` 对多个接口进行实现，但需要注意的是不允许 `IERC20Impl` 实现的多接口内的存在重名函数。

> 此处使用了 `#[external(v0)]` 宏对 `IERC20Impl` 进行修饰。目前 starknet 已支持多种修饰符，具体可以参考 [ABI 的一些写法](https://blog.wssh.trade/posts/sn-foundry-cairo/#abi-%E7%9A%84%E4%B8%80%E4%BA%9B%E5%86%99%E6%B3%95)。

完成上述构造器后，我们可以尝试编写测试函数，但由于目前合约没有实现全部的接口，所以会出现编译报错，请读者在合约内增加无实现函数来避免编译报错。如下:

```rust
fn mint(ref self: ContractState, amount: u256) {}

fn approve(ref self: ContractState, spender: ContractAddress, amount: u256) -> bool {
    true
}

fn transfer(ref self: ContractState, to: ContractAddress, amount: u256) -> bool {
    true
}

fn transferFrom(
    ref self: ContractState, from: ContractAddress, to: ContractAddress, amount: u256
) -> bool {
    true
}
```

请读者在 `src/tests/ERC20_test.cairo` 中输入以下内容:

```rust
use hello_erc20::ERC20::ERC20;
use hello_erc20::ERC20::IERC20Dispatcher;
use hello_erc20::ERC20::IERC20DispatcherTrait;
use hello_erc20::ERC20::ERC20::{Event, Approval};

use starknet::contract_address::ContractAddress;
use starknet::syscalls::deploy_syscall;

use core::test::test_utils::assert_eq;

const NAME: felt252 = 'Test';
const SYMBOL: felt252 = 'TET';
const DECIMALS: u8 = 18_u8;

#[test]
fn test_initializer() {
    let mut calldata = array![NAME, SYMBOL, DECIMALS.into()];

    let (erc20_address, _) = deploy_syscall(
        ERC20::TEST_CLASS_HASH.try_into().unwrap(), 0, calldata.span(), false
    )
        .unwrap();

    let erc20_token = IERC20Dispatcher { contract_address: erc20_address };

    assert_eq(@erc20_token.name(), @NAME, 'Name should be NAME');
    assert_eq(@erc20_token.symbol(), @SYMBOL, 'Symbol should be SYMBOL');
    assert_eq(@erc20_token.decimals(), @18_u8, 'Decimals should be 18');
}
```

在文件头部，我们定义了一系列后文所需要的常量，主要集中在 ERC20 构造部分。

> 此处利用了 cairo 对短字符串的支持，使用了 `'Test'` 等进行字符串定义，这些字符串会被直接转化为 `felt252` 类型。

为了进行测试，我们导入了大量依赖，其中最核心的依赖为 `hello_erc20::ERC20::IERC20Dispatcher` 和 `hello_erc20::ERC20::IERC20DispatcherTrait` 。读者可能感觉我们似乎没有在 `ERC20` 合约内实现这两个模块，实际上，这两个模块是由 `IERC20` 接口衍生获得的，是 cairo 语言自动生成的。此模块用于后文对部署合约的函数调用。此处导入的 `use hello_erc20::ERC20::ERC20::{Event, Approval};` 是为后文抛出事件测试准备的

> 可能有读者好奇如果写出这么多依赖，对于我来说，大部分都是写完代码观察编译器报错后补充的，当然还有一部分来自参考代码

对于 Cairo 合约测试来说，我们需要部署合约然后调用部署合约的函数观察结果来判断合约运行是否符合要求。此处，我们使用 `deploy_syscall` 函数进行合约部署，该函数的参数如下:

```rust
extern fn deploy_syscall(
    class_hash: ClassHash,
    contract_address_salt: felt252,
    calldata: Span<felt252>,
    deploy_from_zero: bool,
) -> SyscallResult<(ContractAddress, Span<felt252>)> implicits(GasBuiltin, System) nopanic;
```

此处涉及到关于合约部署的相关知识，我们会在后文进行专题介绍，简单来说，在 starknet 上部署合约分为两步，第一步是 `declare` 合约，此过程中会将合约注册到 StarkNet 合约库中获得唯一标识 `classhash` ，第二步是根据 Classhash 部署合约。此处的 `deploy_syscall` 函数即用于合约部署，其中 `class_hash` 参数即为 declare 后获得的 classhash ，而 `calldata` 参数则用于构造器函数。

其余参数均有特殊作用，`contract_address_salt` 参数用于调整和计算合约地址，合约地址的计算方法为:

```javascript
contract_address := pedersen(
    “STARKNET_CONTRACT_ADDRESS”,
    caller_address,
    salt,
    class_hash,
    pedersen(constructor_calldata))
```

而 `deploy_from_zero` 则是 StarkNet 账户抽象的核心，其允许用户使用零用户部署合约，即不在 EOA 帮助的情况下部署合约，我们会在后文内详细讨论此问题。

在上述测试代码中，我们使用 `array![NAME, SYMBOL, DECIMALS.into()]` 获得数组类型，完成 calldata 构造后，我们使用 `deploy_syscall` 函数进行合约部署。最后，我们将部署的合约包装在 `IERC20Dispatcher` 内以方便后文直接调用。

此处使用了 `DECIMALS.into()` 实现类型转换，`calldata` 为 `Array<felt252>` 而 `DECIMALS` 为 `u8` 类型，所以 `DECIMALS` 无法直接 `append` 到 `calldata` 中。在 Cairo 中，存在一类 `trait` 被称为 `into` ，该 `trait` 的功能是将不符合标准的类型转化为函数要求的类型，所以此处我们调用 `DECIMALS.into()` 实现了 `u8` 到 `felt252` 的自动转换。但需要注意的是，不是任意类型都可以使用 `into` 进行转换。而后文使用的 `try_into` 功能类似，但其会在转换失败后返回 `Option` 类型，我们可以使用 `unwrap` 函数获取 `Option` 内包装的数据。当然，如果转换失败，`unwrap`方法也会抛出异常。

> `try_into` 往往用于一些危险的类型转换，如 `u16` 转 `u8` 等

在 `deploy_syscall` 函数中，我们也使用 `span` 函数实现了 `Array<felt252>` 到 `Span<felt252>` 的转化。此处使用的 `Span<felt252>` 是 `Array<felt252>` 的快照(`snapshots`)。在 Cairo 中，官方建议函数之间传递数组使用 `Span<T>` 类型以避免变量借代等问题。

对于具体的 `assert_eq` 相等判断部分较为简单，但需要注意的是 `assert_eq` 需要使用 `use core::test::test_utils::assert_eq;` 单独导入，且其定义如下:

```rust
#[inline]
fn assert_eq<T, +PartialEq<T>>(a: @T, b: @T, err_code: felt252) {
    assert(a == b, err_code);
}
```

此处我们可以看到 `assert_eq` 接受的类型为 `@T` 类型，这意味我们需要手动使用 `@` 符号手动转换一些类型，如 `@erc20_token.name()` 和 `@NAME` 等

在项目根目录下运行 `scarb test` 命令，输出如下:

```bash
testing hello_erc20 ...
running 2 tests
test hello_erc20::tests::fib_test::it_works ... ok (gas usage est.: 46860)
test hello_erc20::tests::ERC20_test::test_initializer ... ok (gas usage est.: 467510)
test result: ok. 2 passed; 0 failed; 0 ignored; 0 filtered out;
```

我们首先编写较为容易测试的 `approve` 函数，编写代码如下:

```rust
fn approve(ref self: ContractState, spender: ContractAddress, amount: u256) -> bool {
    let owner = get_caller_address();
    self._allowances.write((owner, spender), amount);
    self.emit(Event::Approval(Approval { owner, spender, value: amount }));

    true
}
```

该函数较为简单，在此处，我们采用 `self._allowances.write((owner, spender), amount);` 函数对数据进行写入。该方法等同于以下 solidity 代码:

```solidity
_allowance[msg.sender][spender] = amount;
```

然后，我们编写测试代码，我们首先增加一个特殊测试辅助函数 `setUp` ，该函数用于初始化 ERC20 合约，并设置一个用于合约调用的地址，如下:

```rust
fn setUp() -> (ContractAddress, IERC20Dispatcher, ContractAddress) {
    let caller = contract_address_const::<1>();
    set_contract_address(caller);

    let mut calldata = array![NAME, SYMBOL, DECIMALS.into()];

    let (erc20_address, _) = deploy_syscall(
        ERC20::TEST_CLASS_HASH.try_into().unwrap(), 0, calldata.span(), false
    )
        .unwrap();

    let mut erc20_token = IERC20Dispatcher { contract_address: erc20_address };

    (caller, erc20_token, erc20_address)
}
```

此处的 `set_contract_address` 函数需要使用 `use starknet::testing::set_contract_address;` 语句导入。该函数的作用是将该语句后的所有函数调用的测试合约地址修正为 `caller` 。而 `contract_address_const` 用于生成固定地址，需要使用 `use starknet::contract_address_const;` 导入。

> 此处读者需要理解 cairo 的测试基本原理，cairo 的测试可以认为是我们将测试代码和 ERC20 代币合约一起部署到测试环境中，测试代码对 ERC20 代币合约进行调用，所以此处我们使用 `set_contract_address` 实现修改 ERC20 代币合约内的 `get_caller_address` 的值

在 `src/tests/ERC20_test.cairo` 中，我们使用 `ERC20_test.cairo` 为基础对部署的 ERC20 代币合约进行测试，所以我们需要修改 `ERC20_test.cairo` 的地址来改变 ERC20 代币合约调用者的地址，此处我们使用 `set_contract_address` 函数将 `ERC20_test.cairo` 的地址修正为 `contract_address_const::<1>()` 实现了对代币合约调用者的修改。

最后，我们给出测试函数，如下:

```rust
#[test]
fn test_approve() {
    let (caller, erc20_token, erc20_address) = setUp();

    let spender: ContractAddress = contract_address_const::<2>();
    let amount: u256 = u256_from_felt252(2000);

    erc20_token.approve(spender, amount);

    assert(erc20_token.allowance(caller, spender) == amount, 'Approve should eq 2000');
    assert_eq(
        @starknet::testing::pop_log(erc20_address).unwrap(),
        @Event::Approval(Approval { owner: caller, spender: spender, value: amount }),
        'Approve Emit'
    )
}
```

注意，此处 `assert_eq` 需要使用 `use test::test_utils::assert_eq;` 进行导入，而 `u256_from_felt252` 需要使用 `use core::integer::u256_from_felt252;` 导入。我们使用了 `@starknet::testing::pop_log` 实现了对合约抛出事件的监控，我们使用 `assert_eq` 对监控到的事件和我们预计抛出的事件进行了相等性测试。这也是为什么我们要在 ERC20 合约代码中实现 `PartialEq` 宏。

接下来，我们编写 `transfer` 系列代码，但在编写 `transfer` 系列代码前。为了方便后期测试，我们引入 `mint` 函数，如下:

```rust
fn mint(ref self: ContractState, amount: u256) {
    let sender = get_caller_address();
    self._total_supply.write(self._total_supply.read() + amount);
    self._balances.write(sender, self._balances.read(sender) + amount);
}
```

由于此函数测试较为简单，不再给出测试代码，读者可以前往 [github 仓库](https://github.com/wangshouh/hello_erc20/blob/main/src/tests/ERC20_test.cairo) 阅读。

我们给出 `transfer` 的最简实现，如下:

```rust
fn transfer(ref self: ContractState, to: ContractAddress, amount: u256) -> bool {
    let from = get_caller_address();

    self._balances.write(from, self._balances.read(from) - amount);
    self._balances.write(to, self._balances.read(to) + amount);

    self.emit(Event::Transfer(Transfer { from, to, value: amount }));

    true
}
```

此处的 `self.emit` 用于 `event` 的释放。

对于此函数的正向测试，请读者自行参考 [仓库](https://github.com/wangshouh/hello_erc20/blob/main/src/tests/ERC20_test.cairo#L55)。此函数是本合约中第一个可能会抛出异常的函数，我们认为该函数在用户转账数额大于其余额时应该产生报错。我们尝试编写此测试:

```rust
#[test]
fn test_err_transfer() {
    let (from, erc20_token, _) = setUp();
    let to = contract_address_const::<2>();
    let amount: u256 = u256_from_felt252(2000);

    erc20_token.mint(amount);
    erc20_token.transfer(to, u256_from_felt252(3000));
}
```

进行测试，结果如下:

```bash
running 7 tests
test hello_erc20::tests::fib_test::it_works ... ok (gas usage est.: 46860)
test hello_erc20::tests::ERC20_test::test_mint ... ok (gas usage est.: 589950)
test hello_erc20::tests::ERC20_test::test_err_transfer ... fail (gas usage est.: 744260)
test hello_erc20::tests::ERC20_test::test_initializer ... ok (gas usage est.: 467510)
test hello_erc20::tests::ERC20_test::test_approve ... ok (gas usage est.: 582860)
test hello_erc20::tests::ERC20_test::test_FailtransferFrom ... ok (gas usage est.: 1048330)
test hello_erc20::tests::ERC20_test::test_transfer ... ok (gas usage est.: 1060820)
failures:
   hello_erc20::tests::ERC20_test::test_err_transfer - Panicked with (0x753235365f737562204f766572666c6f77 ('u256_sub Overflow'), 0x454e545259504f494e545f4641494c4544 ('ENTRYPOINT_FAILED')).
```

但是问题来了，`fail` 测试看上去不太好看，而且这个错误是我们已知的，该怎么办？答案是引入 `should_panic` 宏，用法如下:

```rust
#[test]
#[should_panic(expected: ('u256_sub Overflow', 'ENTRYPOINT_FAILED', ))]
fn test_err_transfer() {
    ...
}
```

此处使用 `expected` 指明报错原因即可。再次运行测试，会发现所有测试均通过。

> 此处的错误就是 rust 中的 `panic` 运行时恐慌，目前所见合约基本都使用 `panic` 抛出错误。值得注意的，Cairo 合约所有的错误除原本的错误外都会带有 `ENTRYPOINT_FAILED` 错误。

接下来，我们实现 `transferFrom` 函数，代码如下:

```rust
fn transferFrom(
    ref self: ContractState, from: ContractAddress, to: ContractAddress, amount: u256
) -> bool {
    let caller = get_caller_address();
    let allowed: u256 = self._allowances.read((from, caller));

    if !(allowed == BoundedInt::max()) {
        self._allowances.write((from, caller), allowed - amount);
        self
            .emit(
                Event::Approval(
                    Approval { owner: from, spender: caller, value: allowed - amount }
                )
            );
    }

    self._balances.write(from, self._balances.read(from) - amount);
    self._balances.write(to, self._balances.read(to) + amount);

    self.emit(Event::Transfer(Transfer { from, to, value: amount }));

    true
}
```

此处涉及到 `u256` 即 `uint256` 的最大值判断问题，我们使用 `BoundedInt::max()` 来获取任意数字类型的最大值，使用此函数需要使用 `use core::integer::BoundedInt;` 进行导入，此处进行了类型推断，自动推断出此处需要 `u256` 的最大值。

当然，由于 u256 实际上是通过两个 `u128` 拼接获得的，我们可以通过以下函数生成 u256 的最大值，如下:

```rust
fn MAX_U256() -> u256 {
    u256 {
        low: 0xffffffffffffffffffffffffffffffff_u128, high: 0xffffffffffffffffffffffffffffffff_u128
    }
}
```

该函数会在测试过程中使用。

最后，我们给出相关测试，这些测试都较为简单，读者可以自行参考仓库。

此处我们给出一些编程规范，事实上，本文以上代码并没有完全遵循此规范。

关于命名问题，很幸运，此部分的文档处于完成状态，我们可以参考 [Naming conventions](https://github.com/starkware-libs/cairo/blob/main/docs/reference/src/components/cairo/modules/language_constructs/pages/naming-conventions.adoc) 文档。

关于项目组织问题，我们可以发现使用 rust 作为语法来源，遵从 [组合优于继承](https://en.wikipedia.org/wiki/Composition_over_inheritance) 原则的 cairo 1 语言无法实现 solidity 那样的合约继承关系。而且 cairo 1 中的合约属于特殊模块。目前较为通用的做法是将大部分不涉及存储变量的操作抽离为库，即不包含 `#[contrat]` 宏的普通模块，而合约则调用库中的函数。由于 ERC20 合约较为简单，所以我们没有采取这种复杂方式，但随着项目的拓展，我们有必要将较为复杂的逻辑独立出来写进库中。当然，这一法则也不是我提出的，在 cairo 0 时期就已有对此问题的讨论，具体可以参考 [Cairo Coding Guidelines](https://medium.com/nethermind-eth/cairo-coding-guidelines-74eb6f4ee264) 。

## ERC20 合约部署

本文使用 [argent](https://www.argent.xyz/argent-x/) 钱包作为浏览器钱包，但需要注意的是，在部署阶段，我们使用了命令行工具，`argent` 钱包的功能是完成一些简单的 faucet 等操作。请读者完成插件安装等步骤，并设置账户。

设置完成后，读者可以获得账户地址。读者需要注意在 starknet 上，所有账户均为合约账户，没有 EOA 账户的存在，所以理论上获得一个账户就是我们在 starknet 上的第一次合约部署。

当读者完成设置密码等步骤后，会获得钱包地址，但此时账户仍处于未部署状态。点击右上角⚙图标，如下图:

![Agent Deploy](https://acjgpfqbqr.cloudimg.io/_img1_/0c7c1bbc8e7f7ab12d8c69c8be19ba50.jpg)

我们可以看到 `Deploy account` 的选项。此处的部署需要消耗一笔资产，请读者前往 [此处](https://faucet.goerli.starknet.io/) 获取第一笔 ETH 资产。当交易进入 `Pending` 状态后，并可在 agent 钱包中查询到存在 ETH 资产，读者可以点击 `Deploy account` 进行账户合约部署。

如果读者担心资产过少，可以前往 [starkgate](https://goerli.starkgate.starknet.io/) 进行跨链，将 geroli 测试网中的 ETH 进行跨链。

作为开发者，我们应该了解这一流程是如何实现的。与以太坊不同，在 starknet 上存在一种特殊的交易类型，被称为 `declare` 声明。该交易的用途是将合约字节码注册到 starknet 状态仓库中，注册完成后，我们可以获得 `class hash` 标识符。关于 `class hash` 的具体计算方法，读者可自行参考 [文档](https://docs.starknet.io/documentation/architecture_and_concepts/Contracts/class-hash/#computing_the_cairo_1_class_hash)。

合约部署时，我们需要知道合约的 `class hash` ，使用 [公式](https://docs.starknet.io/documentation/architecture_and_concepts/Contracts/contract-address/) 可以计算得到待部署合约地址。获得待部署合约地址后，我们对其进行转账。该笔转账汇入的资产作为合约的部署费用，最后，我们发起 `deploy_account` 交易实现合约部署。总结如下:

![StarkNet Account Deploy](https://blogimage.4everland.store/StarkNetAccountDeploy.svg)

此流程中，我们没有使用 EOA 账户进行合约部署，此做法需要将 `deploy_syscall` 函数的 `deploy_from_zero` 设置为 `True` 。在此模式下，会使用部署合约地址上的 ETH 支付手续费。我们可以通过计算提前获得此地址，使用跨链桥等工具对其充值 ETH 即可。

完成上述流程后，我们需要安装一个用于 StarkNet 的 CLI 工具 [starkli](https://github.com/xJonathanLEI/starkli)。该工具是目前 StarkNet 生态系统内最新的交互工具，其功能对标 Foundry 中的 `cast` 工具。

我们首先使用以下命令进行 starkli 的安装工具 `starkliup` 的安装:

```bash
curl https://get.starkli.sh | sh
```

接下来运行 `starkliup` 命令安装 starkli，如下:

```bash
root@LAPTOP ~# starkliup
Installing the latest version of starkli...
Fetching the latest release from GitHub...
####################################################################################################### 100.0%
Latest release found: v0.1.3
Detected host triple: x86_64-unknown-linux-gnu
Downloading latest release from GitHub...
####################################################################################################### 100.0%
Successfully installed starkli v0.1.3

Generating shell completion files...
- Bash ... Done
- Zsh ... Done
Note that shell completions might not work until you start a new session.
```

接下来，我们需要部署一个用于合约部署的账户，首先我们需要生成账户私钥，如下:

```bash
starkli signer keystore new ~/.starknet_accounts/key.json
```

其中 `~/.starknet_accounts/key.json` 可以修改为任一文件位置。

在生成私钥过程中，需要用户输入一个密码来加密密钥，建议开发者使用一个较为复杂的密码。

根据上文对 argent 钱包的介绍，读者应该发现部署一个账户实际上分为以下三步:

1. 计算账户地址
2. 向账户地址内转入 ETH
3. 部署

使用 `starkli account oz init ~/.starknet_accounts/starkli.json --keystore ~/.starknet_accounts/key.json` 计算账户地址:

```bash
root@LAPTOP ~# starkli account oz init ~/.starknet_accounts/starkli.json --keystore ~/.starknet_accounts/key.json
Enter keystore password: Created new account config file: /root/.starknet_accounts/starkli.json

Once deployed, this account will be available at:
    0x02306d22bc46bc9399e705094a496c134a0cf8fe872d72ec9fe45ca9ccfd4851

Deploy this account by running:
    starkli account deploy /root/.starknet_accounts/starkli.json
```

使用 [faucet](https://faucet.goerli.starknet.io/) 向账户地址进行充值。

后续操作需要调用 starknet 的 RPC 操作，所以建议读者此处使用环境变量设置 RPC ，如下:

```bash
export STARKNET_RPC=https://free-rpc.nethermind.io/goerli-juno
```

> 读者可根据 [Full nodes & API services](https://docs.starknet.io/documentation/tools/api-services/) 文档自行选择其他服务商或搭建节点服务，注意节点服务商的 RPC 版本将会影响命令执行

使用 `starkli account deploy ~/.starknet_accounts/starkli.json --keystore ~/.starknet_accounts/key.json` 部署账户:

```bash
root@LAPTOP # starkli account deploy ~/.starknet_accounts/starkli.json --keystore ~/.starknet_accounts/key.json

Enter keystore password: The estimated account deployment fee is 0.000005063280307272 ETH. However, to avoid failure, fund at least:
    0.000007594920460908 ETH
to the following address:
    0x02306d22bc46bc9399e705094a496c134a0cf8fe872d72ec9fe45ca9ccfd4851
Press [ENTER] once you've funded the address.
Account deployment transaction: 0x06b17658670c9d3c6c1d3ef8e235249eb0cc66cf40bd172335faff98b0b636bc
Waiting for transaction 0x06b17658670c9d3c6c1d3ef8e235249eb0cc66cf40bd172335faff98b0b636bc to confirm. If this process is interrupted, you will need to run `starkli account fetch` to update the account file.
Transaction not confirmed yet...
Transaction 0x06b17658670c9d3c6c1d3ef8e235249eb0cc66cf40bd172335faff98b0b636bc confirmed
```

至此，我们完成了部署合约账户的配置，这些账户都存储在 `~/.starknet_accounts/starkli.json` 中，如下:

```json
{
  "version": 1,
  "variant": {
    "type": "open_zeppelin",
    "version": 1,
    "public_key": "0x7b62815cb338983d49827cb1d359859ca57642803b9cb2f894cdaea748e7437"
  },
  "deployment": {
    "status": "deployed",
    "class_hash": "0x48dd59fabc729a5db3afdf649ecaf388e931647ab2f53ca3c6183fa480aa292",
    "address": "0x2306d22bc46bc9399e705094a496c134a0cf8fe872d72ec9fe45ca9ccfd4851"
  }
}
```

接下来，我们需要修正项目使其可以被正确编译。在 `Scarb.toml` 中写入以下内容:

```toml
[package]
name = "hello_erc20"
version = "0.1.0"
edition = "2023_10"

# See more keys and their definitions at https://docs.swmansion.com/scarb/docs/reference/manifest.html

[dependencies]
starknet = ">=2.4.0"

[[target.starknet-contract]]
allowed-libfuncs = true
```

此处我们加入了 `dependencies` 和 `target.starknet-contract`。前者为当前项目增加依赖项，此处我们增加了 `starknet` 作为依赖项，此模块提供了大量合约相关的内容。后者代表编译出的合约所含有的特殊选项，此处的特殊选项为 `allowed-libfuncs = true` ，该选项的含义为允许合约调用库函数。如果不使用此选项，合约不能调用标准库内的函数。读者应当注意测试网和主网启用的标注库函数种类不同，具体可以参考 [allowed_libfuncs_lists](https://github.com/starkware-libs/cairo/tree/main/crates/cairo-lang-starknet/src/allowed_libfuncs_lists) 内的内容。

在项目中加入 `.gitgnore` 文件，写入以下内容:

```
/target
```

此处的 `target` 即编译产物的存储文件夹，我们一般不将编译结果上传到 github 中。

使用 `scarb build` 对合约进行编译，我们可以在 `target` 文件夹内获得以下内容:

```bash
.
├── CACHEDIR.TAG
└── dev
    ├── hello_erc20.sierra.json
    ├── hello_erc20.starknet_artifacts.json
    ├── hello_erc20_ERC20.contract_class.json
    └── hello_erc20_unittest.test.json
```

在 `dev` 目录中使用以下命令进行 `declare` 操作，如下:

```bash
starkli declare --keystore ~/.starknet_accounts/key.json --account ~/.starknet_accounts/starkli.json hello_erc20_ERC20.contract_class.json
```

此命令输出如下:

```bash
Declaring Cairo 1 class: 0x03a03848d08e2eb8ebffa1da26ef8f64175ad73aed6ed5c75bd7b0c4f8ebed4b
Compiling Sierra class to CASM with compiler version 2.4.0...
CASM class hash: 0x061cffc0fe65e0e74d78845e691e731a21a0b81f7263adca4617efc78a512456
Contract declaration transaction: 0x01495ea4f1ad4978513ca70a9650cc59ea47e7b26081ef2efc9c6fb33a9d05af
Class hash declared:
0x03a03848d08e2eb8ebffa1da26ef8f64175ad73aed6ed5c75bd7b0c4f8ebed4b
```

值得注意的是，如果您完全照抄了我的代码，可能会出现如下内容:

```bash
Not declaring class as it's already declared. Class hash:
0x03a03848d08e2eb8ebffa1da26ef8f64175ad73aed6ed5c75bd7b0c4f8ebed4b
```

您可以通过修改 `mint` 函数的名字，或者增加部分函数解决这一问题。

> 不要任意修改除 `mint` 外的函数的名字，否则就会被钱包识别无效代币

正如上文所述，我们通过 `declare` 获得了 `Class hash` ，下一步我们可以使用此 `Class hash` 进行合约部署。

我们尝试使用此命令部署合约:

```bash
starkli deploy --keystore ~/.starknet_accounts/key.json --account ~/.starknet_accounts/starkli.json 0x015470d854ba85f29cde01fd3943698c33ffcb92c5ebdb6048b9547cd6a03eb9 str:HELLO str:HE 18
```

> 如果读者感觉每次输入 `--keystore` 和 `--account` 较为麻烦，可以尝试使用 `export STARKNET_KEYSTORE=~/.starknet_accounts/key.json` 和 `export STARKNET_ACCOUNT=~/.starknet_accounts/starkli.json` 将配置写入环境变量

上述命令输出如下:

```bash
The contract will be deployed at address 0x05007818fb1d449b205ba6043d3bb3819684a091e841fbd90d36d490f4978ebe
Contract deployment transaction: 0x06dbbcb3b30dd66d93f2e86738791579d563160f823c4f9a8dbd7739098dfcd4
Contract deployed:
0x05007818fb1d449b205ba6043d3bb3819684a091e841fbd90d36d490f4978ebe
```

读者可以使用 `Transaction hash` ，前往任一区块链浏览器查看交易状态。较为著名的区块链浏览器有:

1. [starkscan](https://testnet.starkscan.co/)
2. [voyager](https://goerli.voyager.online/)

由于本文使用了较新的 Cairo 2 编程语言，在本文编写时，StarkScan 并没有兼容，所以建议使用 voyager 进行合约操作。

合约最终部署位置为 `0x05007818fb1d449b205ba6043d3bb3819684a091e841fbd90d36d490f4978ebe`。点击 [此网址](https://testnet.starkscan.co/contract/0x05007818fb1d449b205ba6043d3bb3819684a091e841fbd90d36d490f4978ebe#read-write-contract-sub-read) 可以前往交互，读者可以调用 `mint` 函数进行代币铸造。读者也可以将此代币加入钱包，由于代币符合 SNIP-2 标准，所以钱包可以很好的兼容代币。

![Hello ERC20](https://acjgpfqbqr.cloudimg.io/_img1_/880d7d4e22758bcc9e788c51792b3535.png)

## 总结

相信读者完成本文的所有代码编程后，就可以基本掌握 cairo 合约编程技术。在编写本文时，笔者多次因缺乏资料而意欲放弃，最后凭借 [quaireaux](https://github.com/keep-starknet-strange/quaireaux) 等项目走出来困境。事实证明，在没有文档的情况下，还是可以写代码的，只是需要消耗大量时间。

在完成本节内容后，我建议还没有学习过 rust 的 solidity 工程师抓紧时间学习 rust 。目前 rust 几乎成为了区块链领域中的主导语言。虽然在 cairo 0 时期，starknet 开发团队使用 python 构建了大部分工具，但进入 cairo 1 时期，不仅将 cairo 语法完全迁移至 rust ，也将开发工具使用了 rust 重写。而 sui 等公链支持的 move 语言也被认为是 rust 系语言。
