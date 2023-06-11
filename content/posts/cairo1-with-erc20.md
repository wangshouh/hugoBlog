---
title: "Cairo 1 实战入门:编写测试部署ERC-20代币智能合约"
date: 2023-04-17T11:47:33Z
tags: [cario,ERC-20]
math: true
---


## 概述

Cairo 是 ZK Rollup 的领域专用语言，目前仅用于 [StarkNet](https://www.starknet.io/en) 项目。随着 Rollup 叙事的发展，我们认为 cairo 在未来一定会成为智能合约开发的核心语言。

本文类似我之前编写的 [Foundry教程：编写测试部署ERC-20代币智能合约](https://blog.wssh.trade/posts/foundry-with-erc20/) ，介绍了使用 cairo 1 进行编程、测试和部署的全流程。由于缺乏易用工具，本文放弃了本地测试网部署。

本文仅使用 Rust 的部分基础语法，并进行了详细说明，所以读者可以没有 rust 开发基础，但如果读者熟悉 rust 基础语法，那么阅读代码会更加容易。

本文的重点在于介绍 cairo 1 的合约语法部分，理论上，rust 开发者阅读完本文后，就可以熟练编写基础 cairo 1 合约。

本文的部分内容为 solidity 与 cairo 的对比，如果读者不熟悉 solidity 可以直接跳过。由于笔者对 rust 了解不多，所以本文没有给出 cairo 与 rust 的对比。

值得注意的是，笔者没有详细介绍 ERC20 各函数的功能，读者可以参考 [EIP 文档](https://eips.ethereum.org/EIPS/eip-20) 或者 [SNIP 文档](https://github.com/starknet-io/SNIPs/blob/main/SNIPS/snip-2.md) 。

## 安装

在开始进行 Cairo 编程前，我们需要安装准备相关环境。笔者使用的是 WSL Ubuntu 22.04 系统。但事实上，使用 macOS 也可达到相同的开发体验。

本文使用了 Cairo 1 语言，相比于大量依赖于 Python 的 Cairo 0 语言，Cairo 1 语言相关的开发工具基本都使用了 Rust 。这意味着我们可以通过直接下载编译后的二进制安装包进行安装。

我们主要依赖于以下工具:

1. [Cairo 开发工具链](https://github.com/starkware-libs/cairo)
2. Cairo 包管理器 [scarb](https://docs.swmansion.com/scarb)

读者需要 `nodejs` 和 `npm` 工具，考虑到大部分读者在系统内应包含此工具，我们不再详细介绍。事实上，如果读者没有此工具，也可以继续阅读。

> 如果读者没有 `nodejs` 工具链，可以考虑使用 [nvm](https://github.com/nvm-sh/nvm) 工具进行安装

我们主要介绍 Cairo 开发工具链的安装，使用以下命令下载 `release` 中编译好的二进制压缩包:

```bash
curl -L -o cairo.zip https://github.com/starkware-libs/cairo/releases/download/v1.1.0/release-x86_64-unknown-linux-musl.tar.gz
``` 

上述命令中的 `v1.1.0` 是笔者编写时的最新版本，请读者根据 [releases](https://github.com/starkware-libs/cairo/releases/) 中的最新版本自行替换。

下载完成后，我们使用以下命令解压缩文件:

```bash
tar -xvf cairo.zip
```

最终，读者会获得一个 `cairo/` 文件夹，该文件内结构如下:

```
.
├── bin
│   ├── cairo-compile
│   ├── cairo-format
│   ├── cairo-language-server
│   ├── cairo-run
│   ├── cairo-test
│   ├── sierra-compile
│   ├── starknet-compile
│   └── starknet-sierra-compile
└── corelib
    ├── Scarb.toml
    ├── cairo_project.toml
    └── src
```

请读者将 `cairo/bin` 部分加入系统变量 `PATH` 中，即完成安装工作。

> 我一般直接修改 `.bashrc` 来永久性增加系统变量，可以在 `.bashrc` 内增加类似 `export PATH="$PATH:/root/.cairo/bin"` 的命令来添加系统变量。

使用以下命令测试安装是否成功:

```bash
cairo-compile -V
```

Scarb的安装方法与 cairo 基本一致，读者可以参考 [文档](https://docs.swmansion.com/scarb/download)。使用以下命令测试安装是否成功:

```bash
scarb -V
```

最后，我们安装 `vscode` 中的开发插件，值得注意的是，目前开发插件需要自行编译安装，使用 [download-directory](https://download-directory.github.io/?url=https%3A%2F%2Fgithub.com%2Fstarkware-libs%2Fcairo%2Ftree%2Fmain%2Fvscode-cairo) 工具下载 `vscode-cairo` 文件夹，并在其中运行以下命令:

```bash
sudo npm install --global @vscode/vsce
npm install
vsce package
code --install-extension cairo1*.vsix
```

如果读者遇到错误，请参考 [文档](https://github.com/starkware-libs/cairo/tree/main/vscode-cairo)，或者直接使用下文我编译好的插件。

如果读者不想自己编译 `cairo1*.vsix` 文件，我提供了一个编译后的插件，点击 [此处](https://files.catbox.moe/0reyvh.vsix) 进行下载。此插件版本对应 `v1.1.0`，请读者注意时效性。当然，我相信未来我们可以直接在拓展市场下载此插件。

下载后仅需要运行 `code --install-extension cairo1*.vsix`，使用以下方法导入安装:

![visx install](https://img.gejiba.com/images/a4c7fcaae3378cc3d8ba84e85d6fbf5f.png)

安装完成后，进入插件的设置页面，如下图:

![Cairo1 Setting](https://img.gejiba.com/images/f9e7827a82197e71da78e63f0a865274.png)

在插件设置页面内，在 `Language Server Path` 内填入 `cairo-language-server` 二进制文件地址，可以使用 `which cairo-language-server` 命令获得。在 `Scarb Path` 内填入 `scarb` 二进制文件地址，可以使用 `which scarb` 命令获得。完成上述设置后，请重启 VSCode 软件。

一个示例配置如下(请勿直接抄写文件地址):

![Cairo Setting Example](https://img.gejiba.com/images/17983ca61ab9a9dbfb81f94ac6ae2378.png)

## Cairo vs. Solidity

考虑到本文大部分读者具有 `solidity` 编程背景，本文将梳理 `EVM` 与 `cairoVM` 的区别。本节内容对于 Cairo 0 的开发者而言有阅读必要，但对于 Cairo 1 的开发者而言，理论上可以跳过本文。

在数据类型方面，事实上，EVM 的原生数据类型仅有 `uint256` ，其他类型都是由 solidity 编程器在编译过程中实现的。

而在 cairo 中，原生数据类型仅有 `felt` 类型，读者可简单认为该类型为 `uint252`。需要注意的是，该类型定义在 **有限域** 上，更加准确的定义为 $0 \leq x < P$ ，而 $P = 2^{251}+17 \cdot 2^{192} + 1$ 。其他数据类型都是由 `corelib` 标准库和编译器实现的。

与 solidity 提供的常规计算机代数不同，cairo 的所有计算都定义在域上，简单来说，就是所有计算完成后都需要与 $P$ 进行模除。当然，这似乎与常规的计算机代数相同。但 `felt` 类型的除法是令人惊奇的。在 solidity 中，我们认为 `x / y` 的结果为 `\lfloor x / y \rfloor` ，设 $x = 7$ 和 $y=3$ ，那么在 solidity 中计算结果为 2 ，但在 cairo 中，计算结果为 `1,206,167,596,222,043,737,899,107,594,365,023,368,541,035,738,443,865,566,657,697,352,045,290,673,496`

这是因为 cairo 对 `felt` 的除法做出了以下要求，设 $z = x / y$ ，那么 $z * y = x$ 是恒成立的。该保证使上述离谱结果的出现。更加详细的解释，请参考 [Field elements](https://www.cairo-lang.org/docs/how_cairo_works/cairo_intro.html#field-elements)。请读者在进行 `felt` 数据类型除法时注意。

在研究完基础数据类型后，我们需要考虑运行环境，众所周知，EVM 是一个基于栈的虚拟机，所有运算都发生在栈上，但 carioVM 则是直接在内存上进行计算。当然，cario 虚拟机也存在寄存器，但功能都较为底层，在正常开发时较少使用。如果读者对此感兴趣请参考 Cairo 0 的 [文档](https://www.cairo-lang.org/docs/how_cairo_works/cairo_intro.html#registers)。

在内存模型上，solidity 使用了可变稀疏内存，我们可以使用 `mstore` 等操作符在内存的任意位置写入数据，且可以在同一地址内进行覆写操作，但 cairo 的内存模型为不可变连续内存，这意味着我们不能在内存地址内任意写入数据且不能改变已写入的内存数据。该特性使编写循环结构变得几乎不可行，我们只能使用递归的方式编写循环。当然，Cairo 1 引入了 `loop` 结构也可以更好的实现循环效果，但对于 `for` 循环语句，目前仍未见到相关语法结构，可能暂未实现。

> 如果读者熟悉 [elixir](https://elixir-lang.org/) 等函数式编程语言，应该对于递归代替循环的编程逻辑较为熟悉。如果读者有空闲时间，可以考虑学习一下。

## Cairo 0 vs Cairo 1

本文在编写时，cairo 1 已经发布，笔者认为 Cairo 1 应该是进一步发展的重点，所以本文主要介绍 Cairo 1 。考虑到读者不一定了解 Cairo 语言的发展历程且了解 Cairo 0 是有意义的，所以本节主要介绍 Cairo 0 和 Cairo 1 之间的区别。

在语法方面，`cairo 1.0` 与 Rust 语法几乎完全一致，但可能有部分语法由于 CairoVM 的限制无法实现。而 `cairo 0` 则与 `golang` 等语言类似。另一方面，Cairo 0 支持一些底层编程方法，允许开发者直接调整寄存器和内存。同时，cairo 0 要求开发者手动维护内存，没有自动的内存分配系统。一个并不恰当的类比是 Cairo 0 类似 huff 语言，而 cairo 1 类似 solidity 语言。

在本文编写时，`cairo 0` 并没有语法文档，只有官方提供的两个教程：

1. [Hello, Cairo](https://www.cairo-lang.org/docs/hello_cairo/index.html)
2. [How Cairo Works](https://www.cairo-lang.org/docs/how_cairo_works/index.html)

前者属于实战入门，而后者则是自底向上的分析。读者可根据自身爱好选择教程。我推荐读者阅读后者，因为后者涉及大量对 CairoVM 的底层分析，这些内容是不会随语言特性改变而改变的。

很遗憾，`cairo 1` 似乎没有完整文档，语法特性几乎完全借鉴于 Rust ，但至于那些特性不被支持，暂且没有完整的语法文档。如果读者希望了解 `cairo 1` ，可以参考以下文档:

1. [Starknet Cairo 101 Automated Workshop](https://github.com/starknet-edu/starknet-cairo-101/tree/main)
2. [A First Look at Cairo 1.0: A Safer, Stronger & Simpler Provable Programming Language](https://medium.com/nethermind-eth/a-first-look-at-cairo-1-0-a-safer-stronger-simpler-provable-programming-language-892ce4c07b38)
3. [The Starknet Book](https://book.starknet.io/index.html)
4. [Awesome Cairo](https://github.com/auditless/awesome-cairo) 该仓库给出了很多 cairo 1 的资源，建议参考

上述资料都处于快速变化中，读者应随时参考官方最新动态。

在编译上，Cairo 1 引入了中间编译层，该表示层被称为 `Sierra` ，而最终的编译结果被称为 `casm` ，更多信息可以参考 [Under the hood of Cairo 1.0: Exploring Sierra](https://medium.com/nethermind-eth/under-the-hood-of-cairo-1-0-exploring-sierra-7f32808421f5) 。如下图:

![cairo 1 complie](https://img.gejiba.com/images/f1424e7997da290ddced38e7f9eb9595.png)

> solidity 的编译也是用了中间表示层方案，大家熟悉的 yul 即中间表示层

## Hello World 与测试

本节所有代码都可以在 [helloERC20](https://github.com/wangshouh/helloERC20) github 仓库内找到。

在了解基本的 CairoVM 的基础知识后，我们进入真正的编程阶段。首先，我们需要初始化项目，使用以下命令初始化包:

```bash
scarb new helloERC20
```

使用 `cd helloERC20` 进入项目目录，我们可以看到以下目录结构:

```bash
.
├── Scarb.toml
└── src
    └── lib.cairo
```

此目录结构内仍缺少一些项目配置，请读者增加 `cairo_project.toml` 文件，并在内部输入以下内容:

```toml
[crate_roots]
helloERC20 = "src"
```

该配置将为 `cairo` 编译器等工具指明项目入口和顶层包的名称。更多关于 `cairo_project.toml`  作用的详细内容，我们会在后文介绍 `use` 关键词时给出。

请读者在 `src` 文件夹下创建 `tests.cairo` 文件，此文件为测试入口，所有在此文件中给出的模块都会被测试。此文件暂时为空，但我们马上会向其内部输入内容。

在 `src` 文件夹下创建 `tests` 文件夹，该文件夹内放置编写后的单元测试。最终，我们可以获得以下项目结构:

```
.
├── Scarb.toml
├── cairo_project.toml
└── src
    ├── lib.cairo
    └── tests.cairo
```

这是目前最标准的项目结构。

我们接下来介绍每个文件和文件夹的作用，如下:

1. `lib.cairo` 作为 `create` 的根，是编译器查找需要编译的代码的起点
2. `tests.cairo` 作为测试的入口存在，内部所有给出的模块都会被测试，我们马上会展示其用法

此处出现了一个新概念 `create`，对于 Rust 开发者而言，这是一个熟悉的概念。`create` 是编译器一次编译的所有内容。

打开 `lib.cairo` ，读者会看到如下代码:

```rust
fn fib(a: felt252, b: felt252, n: felt252) -> felt252 {
    match n {
        0 => a,
        _ => fib(b, a + b, n - 1),
    }
}
```

这是一个斐波那契数列计算函数，我们可以对其进行测试。请读者在 `tests/` 文件夹下创建 `fib_test.cairo`，写入以下内容:

```rust
use helloERC20::fib;

#[test]
fn fib_test() {
    let fib5 = fib(0, 1, 5);
    assert(fib5 == 5, 'fib5 != 5')
}
```

其中宏 `#[test]` 标识 `fib_test` 为测试函数，`assert` 代表测试相等条件，`'fib5 = 5'` 为测试失败后的提示。

此处，我们主要需要讨论 `use helloERC20::fib;` ，这是一个路径导入语句，作用是将位于 `lib.cairo` 中的 `fib` 函数导入。

`cairo` 的导入与 Rust 有所不同。我们需要以编译器的视角看问题， cairo 编译器在启动编译后，会首先寻找 `cairo_project.toml` ，找到 `helloERC20 = "src"` 后，会进入 `src` 目录并记 `src` 目录名称为 `helloERC20` 。然后进入 `lib.cairo` 文件寻找待编译文件。

根据上述流程，我们可以认为 `use helloERC20::fib` 等价于导入 `src/lib.cairo` 中的 `fib` 作用域。可能有读者不理解 `use` 关键词含义，该关键词会将 `helloERC20::fib` 导入作用域，然后我们可以直接调用 `fib` 函数。值得注意的是，`use` 不止可以导入函数，也可以导入一个模块，我们会在后文进行展示。

> 如果读者无法理解，请继续阅读，我会对后文每一个路径导入进行详细分析。当然，读者也可以尝试分析 [quaireaux](https://github.com/keep-starknet-strange/quaireaux/tree/main) 复杂项目的路径导入问题，如果读者可以理解 `quaireaux` 的路径导入，那么就基本可以理解大部分项目的路径导入方法。
> 
> 此部分最好的学习材料是 [Cairo Book Chapter 6](https://cairo-book.github.io/ch06-00-managing-cairo-projects-with-packages-crates-and-modules.html)，建议读者参考

完成上述流程后，在 `tests.cairo` 中键入以下内容:

```cairo
mod fib_test;
```

正如前文所述，`tests.cairo` 是一个测试入口文件，我们使用 `mod fib_test;` 在此文件内标识待测试文件。我们可以认为 `mod fib_test;` 相当于告诉测试工具请将 `tests/fib_test.cairo` 文件中的测试函数运行。更加正式的说，`mod mod fib_test;` 的作用是声明模块。
 
在根目录允许 `cairo-test .` 命令(不要忽略 `.`)，然后，我们发现输入如下:

```bash
running 0 tests
test result: ok. 0 passed; 0 failed; 0 ignored; 0 filtered out;
```

显然，测试工具没有找到任何一个测试。正如上文所述，编译器仅编译 `lib.cairo` 中给出的模块，显然，`tests.cairo` 目前未被写入 `lib.cairo` 所以编译器没有编译测试函数，自然，测试工具也没有发现测试函数。我们需要修正 `lib.cairo`，请读者在文件末尾增加以下内容:

```rust
#[cfg(test)]
mod tests;
```

此处更改会将 `tests.cairo` 引入 `lib.cairo` ，这样编译器和测试工具都可以编写和运行测试函数。再次运行 `cairo-test .` 命令，我们发现一个报错:

```bash
running 1 tests
test helloERC20::tests::fib_test::fib_test ... fail
failures:
   helloERC20::tests::fib_test::fib_test - panicked with [375233589013918064796019 ('Out of gas'), ].

Error: test result: FAILED. 0 passed; 1 failed; 0 ignored
```

测试出现了臭名昭著的 `Out of gas` 报错，这是因为我们在测试过程中未加入可用 gas ，请读者修改 `tests/fib_test.cairo`，如下:

```rust
use helloERC20::fib;

#[test]
#[available_gas(2000000)]
fn fib_test() {
    let fib5 = fib(0, 1, 5);
    assert(fib5 == 5, 'fib5 != 5')
}
```

此处，我们使用 `#[available_gas(2000000)]` 宏为测试环境增加了 `2000000 gas`。再次运行测试命令，如下:

```bash
Error: Failed setting up runner.

Caused by:
    Failed calculating gas usage, it is likely a call for `gas::withdraw_gas` is missing.
```

这次报错更加恐怖，直接无法启动运行环境，这是因为 `fib` 函数未增加 `gas::withdraw_gas` 函数，使运行时无法计算 gas 相关信息。我们需要对 `lib,cairo` 中的 `fib` 函数进行修改，修改后如下:

```rust
use option::OptionTrait;

fn fib(a: felt252, b: felt252, n: felt252) -> felt252 {
    match n {
        0 => a,
        _ => fib(b, a + b, n - 1),
    }
}

#[cfg(test)]
mod tests;
```

最后，我们再次运行 `cairo_test .` 命令，输出如下:

```bash
running 1 tests
test helloERC20::tests::fib_test::fib_test ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 filtered out;
```

目前，测试功能虽然会记录 gas 消耗，但没有直接给出。

在 Hello World 的最后，我们可用尝试编译一下整个项目，为了更加直观的给出编译和运行的过程，请读者创建 `src/main.cairo` 并写入以下内容:

```rust
use debug::PrintTrait;
use helloERC20::fib;

fn main() {
    let fib5 = fib(0, 1, 5);
    fib5.print();
}
``` 

此处引入了 `debug::PrintTrait;` 该模块用于输出 `debug` 信息，此处用此函数充当 `print` 。在 `lib.cairo` 中，写入以下内容:

```rust
use option::OptionTrait;

fn fib(a: felt252, b: felt252, n: felt252) -> felt252 {
    gas::withdraw_gas_all(get_builtin_costs()).expect('Out of gas');
    match n {
        0 => a,
        _ => fib(b, a + b, n - 1),
    }
}

mod main;

#[cfg(test)]
mod tests;
```

正如上文所述，此处的 `mod main;` 是为了帮助编译器找到 `main.cairo` 文件。

运行 `cairo-run --available-gas 300000 .` 命令，输出如下:

```bash
[DEBUG]                                (raw: 5)

Run completed successfully, returning []
Remaining gas: 280410
```

我们可以看到输出了结果 `5` 。

> 值得注意的是，此处的 `--available-gas` 为必选项，否则会运行失败

如果读者对底层感兴趣，可以尝试使用 `cairo-run --available-gas 300000 --print-full-memory .` 此处使用 `--print-full-memory` 可以打印出内存结构。之前介绍 `cairoVM` 时，我们已经支出 cairoVM 的内存结构是不可变的，所以我们可以根据运行结束后的内存情况来推测运行过程中的事件。当然，直接输出的内存可能很难读懂，如果读者经过 cairo 0 的相关训练，可能可以读懂一部分。

> 目前来看 `cairo-run` 的功能远远低于 `cairo 0` 中的 `cairo-run` 的功能，我相信 cairo 1 的开发者团队会在为了进一步扩充其功能

## ERC20 合约编程

关于 `cairo` 智能合约编程最为核心文档是 [Cairo Contracts](https://github.com/starkware-libs/cairo/blob/main/docs/reference/src/components/cairo/modules/language_constructs/pages/contracts.adoc) ，请读者务必阅读此文档内容。本文的 ERC20 代币合约主要参考了 [starkware 官方实现](https://github.com/starkware-libs/cairo/blob/main/crates/cairo-lang-starknet/test_data/erc20.cairo) 和 [openzeppline 实现](https://github.com/OpenZeppelin/cairo-contracts/blob/cairo-1/src/openzeppelin/token/erc20.cairo) 。需要注意的是，starknet 已有 ERC20 代币规范被称为 [SNIP 2](https://github.com/starknet-io/SNIPs/blob/main/SNIPS/snip-2.md) 。

> openzeppline 目前的实现位于 `cairo-1` 分支，读者阅读时可能此分支已被合并进入主分支。此处需要注意 SNIP 2 的命名规范与 cairo 1 的命名规范不符，但大部分钱包都兼容于 SNIP 2 规范，所以后文我们仍使用了不符合 cairo 1 规范的 SNIP 2 规范进行命名

本文主要基于 solmate 版本的 [ERC20 智能合约](https://github.com/transmissions11/solmate/blob/main/src/tokens/ERC20.sol) 的具体逻辑。

在了解基础的知识后，我们开始进行 ERC20 代币合约编程。在 `src` 下创建 `ERC20.cairo` ，并在 `src/tests` 下创建 `ERC20_test.cairo`。最终，目录结构如下:

```
.
├── Scarb.toml
├── cairo_project.toml
├── src
│   ├── ERC20.cairo
│   ├── lib.cairo
│   ├── main.cairo
│   ├── tests
│   │   ├── ERC20_test.cairo
│   │   └── fib_test.cairo
│   └── tests.cairo
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

完成上述任务后，我们开始编写 ERC20 合约，我们使用了编写和测试的逻辑，编写完部分函数后就会立即进行测试，所以后文代码中的编写和测试会交替出现，请读者仔细观察。

我们首先给出类似接口的定义的 `trait` 定义，如下:

```rust
use starknet::ContractAddress;

trait IERC20 {
    fn name() -> felt252;
    fn symbol() -> felt252;
    fn decimals() -> u8;
    fn total_supply() -> u256;
    fn balanceOf(account: ContractAddress) -> u256;
    fn allowance(owner: ContractAddress, spender: ContractAddress) -> u256;
    fn transfer(recipient: ContractAddress, amount: u256) -> bool;
    fn transferFrom(sender: ContractAddress, recipient: ContractAddress, amount: u256) -> bool;
    fn approve(spender: ContractAddress, amount: u256) -> bool;
}
```

此处引入了 `starknet::ContractAddress` 数据类型，该数据类型表示地址，类似 solidity 中的 `address` 类型。此接口参考了 [SNIP 2](https://github.com/starknet-io/SNIPs/blob/main/SNIPS/snip-2.md) 中的内容。

接下来，我们进行合约主体的开发，我们首先定义合约模块，如下:

```rust
#[contract]
mod ERC20 {
    use helloERC20::ERC20::IERC20;
}
```

此处的 `#[contract]` 告知编译器以下模块 `ERC20` 为合约模块，需要编译时特殊处理。而 `use helloERC20::ERC20::IERC20;` 则为了导入上文定义到 `IERC20` 接口。此处的路径翻译导入 `src/ERC20.cairo` 中的 `IERC20` 部分。

读者可以考虑一下，在 `ERC20_test.cairo` 中如何导入 `ERC20` 模块，首先该模块位于 `src/ERC20.cairo`，所以导入语句前部分路径为 `helloERC20::ERC20`，而 `mod ERC20` 是导入部分，所以最终结果为 `use helloERC20::ERC20::ERC20` ，第一个 ERC20 指文件 `ERC20.cairo` 而第二个 `ERC20` 指 `ERC20.cairo` 中定义的 `mod ERC20`。

完成上述定义后，我们开始定义数据结构和事件，如下:

```rust
    use starknet::get_caller_address;
    use starknet::ContractAddress;

    struct Storage {
        _name: felt252,
        _symbol: felt252,
        _decimals: u8,
        _total_supply: u256,
        _balances: LegacyMap<ContractAddress, u256>,
        _allowances: LegacyMap<(ContractAddress, ContractAddress), u256>,
    }

    #[event]
    fn Transfer(from: ContractAddress, to: ContractAddress, value: u256) {}

    #[event]
    fn Approval(owner: ContractAddress, spender: ContractAddress, value: u256) {}
```

导入各模块的作用如下:

1. `get_caller_address` 导入获取请求者地址的模块，类似 solidity 中的 `msg.sender`
1. `ContractAddress` 导入 `starkNet` 地址类型

结构体 `Storage` 是一类特殊的结构体，声明在此结构体内的变量会被写入存储。此处需要注意 `LegacyMap` 数据类型，此数据类型类似 `solidity` 中的 `mapping` 映射类型。在上述存储变量中，最难理解的是 `_allowances` ，在 solidity 中，该变量一般定义如下:

```solidity
mapping(address => mapping(address => uint256)) public allowance;
```

上述多重映射在 cairo 并不好表达，所以此处使用了元组 `(ContractAddress, ContractAddress)` 与 `uint256` 的映射关系。

> cairo 原生不支持 uint256 类型，仅支持 felt252 类型，uint256 本质上是由 felt128 拼接获得的。此处使用 uint256 是为了保持兼容性。

最后，我们利用 `#[event]` 宏声明了两个事件。对于大部分熟悉 solidity 的读者而言，这些事件都应该较为熟悉。

接下来，我们编写构造器和基础的 `view` 函数，如下:

```rust
    #[constructor]
    fn constructor(name: felt252, symbol: felt252, decimals: u8, ) {
        _name::write(name);
        _symbol::write(symbol);
        _decimals::write(decimals);
    }
```

构造器是合约初始化函数，我们在此处对 ERC20 代币的基本参数进行初始化，较为简单不再赘述，接下来，我们编写 `view` 函数，如下:

```rust
    #[view]
    fn name() -> felt252 {
        _name::read()
    }

    #[view]
    fn symbol() -> felt252 {
        _symbol::read()
    }

    #[view]
    fn decimals() -> u8 {
        _decimals::read()
    }

    #[view]
    fn total_supply() -> u256 {
        _total_supply::read()
    }

    #[view]
    fn balanceOf(account: ContractAddress) -> u256 {
        _balances::read(account)
    }

    #[view]
    fn allowance(owner: ContractAddress, spender: ContractAddress) -> u256 {
        _allowances::read((owner, spender))
    }
```

该部分也较为简单，基本都是 `read` 读取操作。完成上述构造器后，我们可以尝试编写测试函数，请读者在 `src/tests/ERC20_test.cairo` 中输入以下内容:

```rust
use helloERC20::ERC20::ERC20;
use integer::u256;
use integer::u256_from_felt252;

const NAME: felt252 = 'Test';
const SYMBOL: felt252 = 'TET';
const DECIMALS: u8 = 18_u8;

#[test]
#[available_gas(2000000)]
fn test_initializer() {
    let initial_supply: u256 = u256_from_felt252(2000);
    ERC20::constructor(NAME, SYMBOL, DECIMALS, initial_supply);

    assert(ERC20::name() == NAME, 'Name should be NAME');
    assert(ERC20::symbol() == SYMBOL, 'Symbol should be SYMBOL');
    assert(ERC20::decimals() == 18_u8, 'Decimals should be 18');
    assert(ERC20::total_supply() == u256_from_felt252(2000), 'Supply should eq 2000');
}
```

在文件头部，我们定义了一系列后文所需要的常量，主要集中在 ERC20 构造部分。此处需要注意 `initial_supply` 由于其属于特殊类型 `u256` 无法定义为常量。

> 此处利用了 cairo 对短字符串的支持，使用了 `'Test'` 等进行字符串定义，这些字符串会被直接转化为 `felt252` 类型。

对于具体的 `assert` 相等判断部分较为简单，不再赘述。

在项目根目录下运行 `cairo-test --starknet .` 命令，输出如下:

```bash
running 2 tests
test helloERC20::tests::fib_test::fib_test ... ok
test helloERC20::tests::ERC20_test::test_initializer ... ok
test result: ok. 2 passed; 0 failed; 0 ignored; 0 filtered out;
```

此处我们使用 `--starknet` 标识符，该标识符意味着 cairo-test 在测试时引入 `starknet` 环境。一般来说，只要涉及到合约测试，`--starknet` 标识是必要的。

我们首先编写较为容易测试的 `approve` 函数，编写代码如下:

```rust
    #[external]
    fn approve(spender: ContractAddress, amount: u256) -> bool {
        let owner = get_caller_address();

        _allowances::write((owner, spender), amount);

        Approval(owner, spender, amount);

        true
    }
```

较为简单，但由于目前 cairo 1 没有语法糖直接对 `_allowances` 等存储变量进行操作，所以我们只能采用 `_allowances::write` 这种笨拙的方式。该方法等同于以下 solidity 代码:

```solidity
_allowance[msg.sender][spender] = amount;
```

然后，我们编写测试代码，我们首先增加一个特殊测试辅助函数 `setUp` ，该函数用于初始化 ERC20 合约，并设置一个用于合约调用的地址，如下:

```rust
fn setUp() -> ContractAddress {
    let caller = contract_address_const::<1>();
    set_caller_address(caller);
    ERC20::constructor(NAME, SYMBOL, DECIMALS);
    caller
}
```

此处的 `set_caller_address` 函数需要使用 `use starknet::testing::set_caller_address;` 语句导入。该函数的作用是将该语句后的所有函数调用的 `msg.sender` 修正为 `caller` 。

最后，我们给出测试函数，如下:

```rust
#[test]
#[available_gas(2000000)]
fn test_approve() {
    let caller: ContractAddress = setUp();
    let spender: ContractAddress = contract_address_const::<2>();
    let amount: u256 = u256_from_felt252(2000);

    ERC20::approve(spender, amount);

    assert(ERC20::allowance(caller, spender) == amount, 'Approve should eq 2000');
}
```

较为简单，不再赘述。

接下来，我们编写 `transfer` 系列代码，但在编写 `transfer` 系列代码前。为了方便后期测试，我们引入 `mint` 函数，如下:

```rust
    #[external]
    fn mint(amount: u256) {
        let sender = get_caller_address();

        _total_supply::write(_total_supply::read() + amount);
        _balances::write(sender, _balances::read(sender) + amount);
    }
```

由于此函数测试较为简单，不再给出测试代码，读者可以前往 [github 仓库](https://github.com/wangshouh/helloERC20/blob/main/src/tests/ERC20_test.cairo) 阅读。

我们给出 `transfer` 的最简实现，如下:

```rust
    #[external]
    fn transfer(to: ContractAddress, amount: u256) -> bool {
        let from = get_caller_address();

        _balances::write(from, _balances::read(from) - amount);
        _balances::write(to, _balances::read(to) + amount);

        Transfer(from, to, amount);
        
        true
    }
```

对于此函数的正向测试，请读者自行参考 [仓库](https://github.com/wangshouh/helloERC20/blob/main/src/tests/ERC20_test.cairo#L55)。此函数是本合约中第一个可能会抛出异常的函数，我们认为该函数在用户转账数额大于其余额时应该产生报错。我们尝试编写此测试:

```rust
#[test]
#[available_gas(2000000)]
fn test_err_transfer() {
    let from = setUp();
    let to = contract_address_const::<2>();
    let amount: u256 = u256_from_felt252(2000);

    ERC20::mint(amount);
    ERC20::transfer(to, u256_from_felt252(3000));

    assert(ERC20::balanceOf(from) == u256_from_felt252(0), 'Balance from = 0');
    assert(ERC20::balanceOf(to) == amount, 'Balance to = 2000');
}
```

进行测试，结果如下:

```bash
running 6 tests
test helloERC20::tests::ERC20_test::test_approve ... ok
test helloERC20::tests::fib_test::fib_test ... ok
test helloERC20::tests::ERC20_test::test_initializer ... ok
test helloERC20::tests::ERC20_test::test_mint ... ok
test helloERC20::tests::ERC20_test::test_err_transfer ... fail
test helloERC20::tests::ERC20_test::test_transfer ... ok
failures:
   helloERC20::tests::ERC20_test::test_err_transfer - panicked with [39879774624085075084607933104993585622903 ('u256_sub Overflow'), ].
```

但是问题来了，`fail` 测试看上去不太好看，而且这个错误是我们已知的，该怎么办？答案是引入 `should_panic` 宏，用法如下:

```rust
#[test]
#[available_gas(2000000)]
#[should_panic(expected: ('u256_sub Overflow', ))]
fn test_err_transfer() {
    ...
}
```

此处使用 `expected` 指明报错原因即可。再次运行测试，会发现所有测试均通过。

> 此处的错误就是 rust 中的 `panic` 运行时恐慌，目前所见合约基本都使用 `panic` 抛出错误。众所周知，在 EVM 生态系统中，我们都是以 `revert` 进行错误抛出，这是因为 `revert` 不仅可以抛出异常，还会退回 gas 。但尚未见到 cairo 提供此机制，似乎 cairo 错误会直接用户发送的所有 gas ，用户在合约交互时应注意此点。

接下来，我们实现 `transferFrom` 函数，代码如下:

```rust
#[external]
fn transferFrom(from: ContractAddress, to: ContractAddress, amount: u256) -> bool {
    let caller = get_caller_address();
    let allowed: u256 = _allowances::read((from, caller));

    let ONES_MASK = 0xffffffffffffffffffffffffffffffff_u128;

    let is_max = (allowed.low == ONES_MASK) & (allowed.high == ONES_MASK);

    if !is_max {
        _allowances::write((from, caller), allowed - amount);
        Approval(from, caller, allowed - amount);
    }

    _balances::write(from, _balances::read(from) - amount);
    _balances::write(to, _balances::read(to) + amount);

    Transfer(from, to, amount);

    true
}
```

此处涉及到 `u256` 即 `uint256` 的最大值判断问题，正如上文所述，`u256` 事实上是由 `u128` 拼接获得的，所以其本质是一个结构体，我们可以提供 `u256.low` 和 `u256.high` 的方法去访问其前 128 位和后 128 位。此处使用了 `(allowed.low == ONES_MASK) & (allowed.high == ONES_MASK);` 来判断 `allowed` 是否为 `u256` 的最大值。

当然，我们可以通过以下函数生成 u256 的最大值，如下:

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

关于存储变量访问问题，由于目前 cairo 不支持使用等号对存储变量进行重赋值操作，我们只能使用 `::read` 和 `::write` 函数，这些操作是复杂的。`openzepplin` 在自己的实现中使用了包装函数，即将所有的对存储变量的读写尽可能抽象为内部函数，而在具体编程过程中，涉及到对存储变量的读写时都使用包装函数。这是一个较好的思路。

> 在较新版本的 cairo 中，已经引入了一些函数用以解决此问题，但笔者使用的 `cairo-1.1.0` 尚不支持这些函数。

关于项目组织问题，我们可以发现使用 rust 作为语法来源，遵从 [组合优于继承](https://en.wikipedia.org/wiki/Composition_over_inheritance) 原则的 cairo 1 语言无法实现 solidity 那样的合约继承关系。而且 cairo 1 中的合约属于特殊模块。目前较为通用的做法是将大部分不涉及存储变量的操作抽离为库，即不包含 `#[contrat]` 宏的普通模块，而合约则调用库中的函数。由于 ERC20 合约较为简单，所以我们没有采取这种复杂方式，但随着项目的拓展，我们有必要将较为复杂的逻辑独立出来写进库中。当然，这一法则也不是我提出的，在 cairo 0 时期就已有对此问题的讨论，具体可以参考 [Cairo Coding Guidelines](https://medium.com/nethermind-eth/cairo-coding-guidelines-74eb6f4ee264) 。

## ERC20 合约部署

本文使用 [argent](https://www.argent.xyz/argent-x/) 钱包部署合约。请读者完成插件安装等步骤，并设置账户。

设置完成后，读者可以获得账户地址。读者需要注意在 starknet 上，所有账户均为合约账户，没有 EOA 账户的存在，所以理论上获得一个账户就是我们在 starknet 上的第一次合约部署。

当读者完成设置密码等步骤后，会获得钱包地址，但此时账户仍处于未部署状态。点击右上角⚙图标，如下图:

![Agent Deploy](https://img.gejiba.com/images/0c7c1bbc8e7f7ab12d8c69c8be19ba50.jpg)

我们可以看到 `Deploy account` 的选项。此处的部署需要消耗一笔资产，请读者前往 [此处](https://faucet.goerli.starknet.io/) 获取第一笔 ETH 资产。当交易进入 `Pending` 状态后，并可在 agent 钱包中查询到存在 ETH 资产，读者可以点击 `Deploy account` 进行账户合约部署。

如果读者担心资产过少，可以前往 [starkgate](https://goerli.starkgate.starknet.io/) 进行跨链，将 geroli 测试网中的 ETH 进行跨链。

作为开发者，我们应该了解这一流程是如何实现的。与以太坊不同，在 starknet 上存在一种特殊的交易类型，被称为 `declare` 声明。该交易的用途是将合约字节码注册到 starknet 状态仓库中，注册完成后，我们可以获得 `class hash` 标识符。关于 `class hash` 的具体计算方法，读者可自行参考 [文档](https://docs.starknet.io/documentation/architecture_and_concepts/Contracts/class-hash/#computing_the_cairo_1_class_hash)。

合约部署时，我们需要知道合约的 `class hash` ，使用 [公式](https://docs.starknet.io/documentation/architecture_and_concepts/Contracts/contract-address/) 可以计算得到待部署合约地址。获得待部署合约地址后，我们对其进行转账。该笔转账汇入的资产作为合约的部署费用，最后，我们发起 `deploy_account` 交易实现合约部署。总结如下:

![StarkNet Deploy](https://files.catbox.moe/bqu4fa.svg)

完成上述流程后，我们需要安装一个用于部署合约的 CLI 工具 [nile-rs](https://github.com/OpenZeppelin/nile-rs/tree/main)，非常不幸的是，该工具虽然使用 Rust 作为开发语言，但开发者可能认为工具仍较为早期所以没有给出预编译版本。我使用了 `github codespace` 进行了编译，编译后可以获得 `nile-rs` 二进制文件，将其放入 `PATH` 即可。

> `nile-rs` 的开发似乎进入了停滞，但对于简单的合约部署而言还是有用的

完成上述流程后，我们需要修正项目使其兼容 `nile-rs`。在 `Scarb.toml` 中写入以下内容:

```toml
[package]
name = "helloERC20" # the name of the package
version = "0.1.0"    # the current version, obeying semver

[[target.starknet-contract]]
allowed-libfuncs = true

[tool.nile_rs]
artifacts_dir = "./target/release"
contracts_dir = "./src"
```

此处我们加入了一个特殊的选项为 `allowed-libfuncs = true` ，该选项的含义为允许合约调用库函数。如果不使用此选项，合约不能调用标准库内的函数。读者应当注意测试网和主网启用的标注库函数种类不同，具体可以参考 [allowed_libfuncs_lists](https://github.com/starkware-libs/cairo/tree/main/crates/cairo-lang-starknet/src/allowed_libfuncs_lists) 内的内容。

在项目中加入 `.gitgnore` 文件，写入以下内容:

```
/target
/.env
/deployments
```

请读者前往 `Argent` 钱包中导出私钥，如下图:

![Argent Export private key](https://img.gejiba.com/images/0a6a9b974d989d9c20928881c0a915de.png)

导出私钥后，打开 `.env` 文件填入，如下:

```
PRIVATE_KEY=您的私钥
```

创建 `deployments/goerli.accounts.json` 填入以下内容:

```json
[
  {
    "name": "PRIVATE_KEY",
    "address": "您的钱包地址",
    "public_key": "公钥"
  }
]
```

其中 `address` 可以直接在钱包内复制获得，而钱包公钥则较为复杂，读者可以在区块链浏览器里打开钱包地址，如下图:

![View on starknet](https://img.gejiba.com/images/43222d17c1874829e7538a445489e903.png)

在合约交互中，使用 `getSigner` 函数，点击 `query` 就可以获得钱包公钥。

![getSigner](https://img.gejiba.com/images/953163bc321c7ac9bebd24d84ef59da0.png)

完成上述流程后，在终端运行 `nile-rs compile` 命令，我们可以发现项目目录中增加了 `target/release/helloERC20_ERC20.json` 文件。

使用以下命令进行 `declare` 操作，如下:

```bash
nile-rs declare -p PRIVATE_KEY -n goerli helloERC20_ERC20 -t
```

我们会获得如下输出:

```
⏳ Declaration successfully sent!

Transaction hash: 0x067abfeef634ba7755627cbf34921e262aa5e0c7efe3718a32f88278e125e011
Class hash: 0x06672e37dd3f0afe69354fd6243d99a9854f537aa21f607b419b2e67dc2589d0
⏳ Transaction status: Received
```

等待出现 `⏳ Transaction status: Pending` ，该过程一般来说耗时较长。目前，我们一般认为 `declare` 交易达到 `Pending` 状态就相当于交易完成。

值得注意的是，如果您完全照抄了我的代码，可能会出现 `already declared` 的错误，如下:

```bash
Error: Failed attempt to send the declare transaction

Caused by:
    Class with hash 0x6672e37dd3f0afe69354fd6243d99a9854f537aa21f607b419b2e67dc2589d0 is already declared.
    0x7cb3adf9071af26bd07ee9cfd5e94270590a53be25353f68948a7c373731466 != 0 (ClassAlreadyDeclared)
```

您可以通过修改 `mint` 函数的名字，或者增加部分函数解决这一问题。

> 不要任意修改除 `mint` 外的函数的名字，否则就会被钱包识别无效代币

正如上文所述，我们通过 `declare` 获得了 `Class hash` ，下一步我们可以使用此 `Class hash` 进行合约部署。但是，`nile-rs` 为了方便用户，没有设置直接通过 `class hash` 部署合约的功能，而是直接给出了 `nile-rs deploy` 命令。

我们尝试使用此命令部署合约:

```
nile-rs deploy -p PRIVATE_KEY -n goerli helloERC20_ERC20 'HELLO' 'HE' 18
```

此函数会直接使用 `helloERC20_ERC20` 的 `class hash` 进行合约部署，最后三个参数为构造器参数。此处，我们没有使用 `-t` 标识符来进行交易状态跟踪，读者可以使用 `Transaction hash` ，前往任一区块链浏览器查看交易状态。较为著名的区块链浏览器有:

1. [starkscan](https://testnet.starkscan.co/)
2. [voyager](https://goerli.voyager.online/)

当然，此处我们仅给出了 `nile-rs deploy` 的最简单使用，读者可以通过 `nile-rs deploy -h` 命令获得更多参数，如用于调整地址的 `salt` 参数等。

合约最终部署位置为 `0x0398dd27515818daa8dcbf57f18befefd42d4d98405a3a736394314d39c4c29e`。点击 [此网址](https://testnet.starkscan.co/contract/0x0398dd27515818daa8dcbf57f18befefd42d4d98405a3a736394314d39c4c29e#read-write-contract-sub-read) 可以前往交互，读者可以调用 `mint` 函数进行代币铸造。读者也可以将此代币加入钱包，由于代币符合 SNIP-2 标准，所以钱包可以很好的兼容代币。

![Hello ERC20](https://img.gejiba.com/images/e03b52057727eac7209afbf735468832.png)

事实上，目前 starknet 官方更推荐使用 `starknet` 命令行工具进行合约部署，该工具使用 Python 作为开发语言，需要部署虚拟环境且仅兼容 Python 3.9。关于 Python 3.9 安装及虚拟环境配置问题，读者可以参考我在 CSDN 上发表的 [Ubuntu Python 多版本安装](https://blog.csdn.net/WongSSH/article/details/130166679) 。其他部署相关与上文大同小异，具体可以参考 [官方文档](https://docs.starknet.io/documentation/getting_started/deploying_contracts/)。

> 本文其实最早准备使用 `starknet` 工具，我都完成了环境安装，但最后考虑到安装较为复杂，所以更换为了 `nile-rs` 。如果读者无法安装 `starknet` ，可以评论区留言。

读者可能发现我们使用了 `nile-rs` 的编译功能，而 `nile-rs` 开发较慢，其编译器为 `v1.0.0-alpha6` ，而上文我们使用的 `cairo-test` 等 cairo 系工具都基于 `v1.0.0-alpha7` ，由于我们编写 ERC20 代币合约仅在测试部分使用了较新的语法，所以没有出现编译问题。如果读者是狂热的语言新特性爱好者，请使用 `starknet` 命令行工具，因为该工具的编译部分是由 `starknet-compile` 完成的，该编译器属于 cairo 系工具，所以版本较新。

> 当然，最硬核的方法自然是直接与 RPC URL 交互来部署合约

## 总结

相信读者完成本文的所有代码编程后，就可以基本掌握 cairo 合约编程技术。在编写本文时，笔者多次因缺乏资料而意欲放弃，最后凭借 [quaireaux](https://github.com/keep-starknet-strange/quaireaux) 等项目走出来困境。事实证明，在没有文档的情况下，还是可以写代码的，只是需要消耗大量时间。

在完成本节内容后，我建议还没有学习过 rust 的 solidity 工程师抓紧时间学习 rust 。目前 rust 几乎成为了区块链领域中的主导语言。虽然在 cairo 0 时期，starknet 开发团队使用 python 构建了大部分工具，但进入 cairo 1 时期，不仅将 cairo 语法完全迁移至 rust ，也将开发工具使用了 rust 重写。而 sui 等公链支持的 move 语言也被认为是 rust 系语言。

我个人还是比较看好 cairo 语言发展的，其自带的测试框架是极其优秀的。但目前最大问题仍是开发工具的不足，我相信 starknet 团队未来一定会使用 rust 完全重写开发框架。

本文写于 2023 年 4 月 17 日，使用 `cairo 1.0.0-alpha7` 完成。在 2023 年 6 月 11 日进行了更新，使用了 `cairo 1.1.0` 。