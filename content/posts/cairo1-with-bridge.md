---
title: "Cairo 1 实战入门:可升级合约与跨链信息发送"
date: 2023-04-21T14:45:33Z
tags: [cario,solidity]
---

## 概述

本文正在根据 Cairo 1.2 进行更新，请读者注意。

如果读者阅读过笔者之前的文章就会发现，我在 solidity 中使用了 ERC20 代币 -> 可实升级合约的学习路径。为了保持文章的统一性，我准备在此文中介绍 cairo 的可升级合约编程。

此处我没有使用代理合约一词，因为 cairo 1 的可升级配置不需要代理模式。实现起来相当简单，所以为了保持文章的长度，我在代理合约基础上增加了跨链信息发送这一主题。

StarkNet 作为以太坊 L2 项目，其部署在以太坊 [核心合约](https://goerli.etherscan.io/address/0xde29d060D45901Fb19ED6C6e959EB22d8626708e) 提供了跨链信息传输功能，而在 StarkNet 链上，原生支持向以太坊通信的系统调用。

显然，使用这些函数可以构造一个沟通 StarkNet 和 Ethereum 的跨链应用。本文将在 [Cairo 1 实战入门:编写测试部署ERC-20代币智能合约](https://blog.wssh.trade/posts/cairo1-with-erc20/) 基础上构造一个原生支持跨链的 ERC20 代币。

本文出现了大量难度不高的 solidity 代码并使用了 foundry 开发框架，如果读者不熟悉 solidity 合约编程或不熟悉 foundry 框架，请参考 [Foundry教程：编写测试部署ERC-20代币智能合约](https://blog.wssh.trade/posts/foundry-with-erc20/)。

我们可以将跨链任务拆解为两部分:

1. 部署以太坊 ERC20 代币并实现 `transfer_to_L2` 函数
2. 在 HelloERC20 cairo 智能合约基础上实现 `transferToL1` 函数

本文所有代码都可在 [helloCairoBridge](https://github.com/wangshouh/helloCairoBridge) 仓库内找到。

## 可升级合约

在 [Cairo 1 实战入门:编写测试部署ERC-20代币智能合约](https://blog.wssh.trade/posts/cairo1-with-erc20/) 中，我们曾介绍过 `class hash`。事实上，我们编写的 cairo 代码上链时都会被注册到 starknet 区块链的 `class` 状态仓库中，而合约只是 `class` 的运行时，用于存储运行状态。 `class` 和 合约的分离彻底实现了逻辑和状态的分离。

> 简单来看，`class` 相当于逻辑合约，而合约相当于代理合约。

众所周知，可升级合约的基础就是状态和逻辑的分离。为了方便开发者进行可升级合约编程，starknet 为 cairo 语言提供了一个特殊的用于合约升级函数 `replace_class_syscall(new_class_hash)`。正如函数名，此函数用于替换合约背后的 `class`。这意味着，我们可以随时通过调用此函数实现运行逻辑的改变。

有很多读者可以阅读过笔者之前编写的 solidity 代理合约系列，一定对存储槽冲突问题有所耳闻。但在 starknet 中，存储槽冲突问题也消失了。

在 starknet 智能合约中，我们使用以下方法进行数据写入:

```rust
_name::write(name);
```

但这其实使用语法糖后的写法，其底层实现为:

```rust
        fn address() -> starknet::StorageBaseAddress {
            starknet::storage_base_address_const::<0x3a858959e825b7a94eb8d55c738f59c7bf4685267af5064bed5fd9c6bbc26de>()
        }
        fn write(value: felt252) {
            // Only address_domain 0 is currently supported.
            let address_domain = 0_u32;
            starknet::StorageAccess::<felt252>::write(
                address_domain,
                address(),
                value,
            ).unwrap_syscall()
        }
```

其中，`starknet::StorageAccess::<felt252>::write(address_domain, address, value)` 是写入的核心函数，而写入地址为 变量名的 `sn-keccak` 哈希值 `0x3a858959e825b7a94eb8d55c738f59c7bf4685267af5064bed5fd9c6bbc26de` 。这与 solidity 的线性存储排布产生了鲜明对比。显然，这种依靠变量名哈希值进行存储的方式完全避免了存储槽冲突问题。我们可以进行任意的合约升级，不需要专门考虑升级前后存储变量是否会丢失或无法检索等问题。

> 关于此处的变量存储地址 `address` 的计算问题，读者可以参考 [文档](https://docs.starknet.io/documentation/architecture_and_concepts/Contracts/contract-storage/#storage_variables)

综上所述，可升级合约在 cairo 1 的极易实现，只需要在合约内加入以下内容:

```rust
    use starknet::syscalls::replace_class_syscall;

    #[external]
    fn upgrade(new_class_hash:  core::starknet::class_hash::ClassHash) -> bool {
        replace_class_syscall(new_class_hash);
        true
    }
```

请注意此处没有对 `upgrade` 进行权限控制，读者在使用时应当进行增加。

> 读者在参与 starknet 项目交互时，应当注意风险，可升级合约在 starknet 上可能会成为主流，这意味着项目方更容易通过可升级合约的方式跑路


## 跨链流程

在编写代码前，我们需要了解跨链的基本流程，该部分内容主要参考 [L1-L2 messaging](https://docs.starknet.io/documentation/architecture_and_concepts/L1-L2_Communication/messaging-mechanism/) 文档。

### L2 -> L1

从 starknet 跨链到 ethereum 主网是比较简单的。

一个简单的例子如下:

```rust
        let mut message_payload: Array<felt252> = ArrayTrait::new();
        message_payload.append(WITHDRAW_MESSAGE);
        message_payload.append(l1_recipient.into());
        message_payload.append(amount.low.into());
        message_payload.append(amount.high.into());

        send_message_to_l1_syscall(
            to_address: read_initialized_l1_bridge(), payload: message_payload.span()
        );
```

当我们在 cairo 合约中调用 `send_message_to_l1_syscall` 函数时， starknet 节点会收到 `to_address` 和 `payload` 的信息，其中 `to_address` 是 L1 信息接收地址，而 `payload` 为发送的信息内容。节点收到上述信息后，会使用以下公式计算 hash 值:

```solidity
keccak256(
    abi.encodePacked(
        FromAddress,
        ToAddress,
        Payload.length,
        Payload
    )
);
```

其中各参数含义如下:

- FromAddress L2 发送方地址
- ToAddress L1 接收方地址
- Payload 发送信息

计算完成后会将上述内容使用 `abi.encodePacked` 进行序列化，以其作为参数请求 L1 的 [核心合约](https://goerli.etherscan.io/address/0xde29d060D45901Fb19ED6C6e959EB22d8626708e) 中 `updateState` 函数，该函数有很多作用，此处我们仅关注 L2 -> L1 跨链信息传递功能。此功能是通过 `processMessages` 函数实现的，其中核心部分如下:

```solidity
if (isL2ToL1) {
    bytes32 messageHash = keccak256(
        abi.encodePacked(programOutputSlice[offset:endOffset])
    );

    emit LogMessageToL1(
        // from=
        programOutputSlice[offset + MESSAGE_TO_L1_FROM_ADDRESS_OFFSET],
        // to=
        address(programOutputSlice[offset + MESSAGE_TO_L1_TO_ADDRESS_OFFSET]),
        // payload=
        (uint256[])(programOutputSlice[offset + MESSAGE_TO_L1_PREFIX_SIZE:endOffset])
    );
    messages[messageHash] += 1;
}
```

此处的 `programOutputSlice` 即节点打包的请求参数，我们使用数组检索来实现请求参数的反序列化。值得注意的是，这些请求参数并没有被保存在以太坊状态中，而仅作为 `LogMessageToL1` 事件抛出。该事件定义为:

```solidity
event LogMessageToL1(uint256 indexed fromAddress, address indexed toAddress, uint256[] payload);
```

故在上述代码中，只有 `messages[messageHash] += 1;` 进行了以太坊状态修改，所以归根到底，只有 L2 -> L1 的信息的哈希值被记录了。

> 如果读者对此部分感兴趣，可以阅读 [源代码](https://github.com/starkware-libs/cairo-lang/blob/master/src/starkware/starknet/solidity/Starknet.sol#L157)。

这意味着如果我们在 L2 向 L1 发送信息，我们需要自己在链下维护 `from_address` 和 `payload` 信息。当然，从核心合约抛出的事件中检索也可以。

当 L2 节点在 L1 完成数据写入后，就完成所有任务。接下来，我们需要使用 L1 合约去消费数据。我们一般会直接使用 `consumeMessageFromL2` 函数。为了使读者了解该函数的底层原理，我截取了其 [源代码](https://github.com/starkware-libs/cairo-lang/blob/master/src/starkware/starknet/solidity/StarknetMessaging.sol#L132):

```solidity
function consumeMessageFromL2(uint256 fromAddress, uint256[] calldata payload)
    external
    override
    returns (bytes32)
{
    bytes32 msgHash = keccak256(
        abi.encodePacked(fromAddress, uint256(msg.sender), payload.length, payload)
    );

    require(l2ToL1Messages()[msgHash] > 0, "INVALID_MESSAGE_TO_CONSUME");
    emit ConsumedMessageToL1(fromAddress, msg.sender, payload);
    l2ToL1Messages()[msgHash] -= 1;
    return msgHash;
}
```

`consumeMessageFromL2` 实质就是检索存储中是否存在指定跨链信息的哈希值。如果该跨链信息存在，那么返回信息哈希，否则则抛出异常。在跨链应用开发过程中，我们往往直接丢弃 `consumeMessageFromL2` 返回的信息哈希值。

> 请注意此处消费不存在的信息会直接抛出异常

正如上文所述，我们需要自己维护 `from_address` 和 `payload` 信息，否则就无法调用 `consumeMessageFromL2` 以消费数据。

在真正的开发过程中，我们一般会使用 [IStarknetMessaging](https://github.com/starkware-libs/cairo-lang/blob/master/src/starkware/starknet/solidity/IStarknetMessaging.sol) 接口，我们会在后文展示其使用。

一个简单的流程如下:

![StarkNet Message L2 -> L1](https://img.gejiba.com/images/bc2ca14215bb38af77ab219d25835a5d.png)

当然，此流程中没有显示技术细节。一个具有更多技术细节的流程图如下:

![StarkNet L2 -> L1](https://files.catbox.moe/tkjaer.svg)

在上图中，出现之前没有给出过解释的 `StarknetOS` 名词。简单来说，我们可以认为其指 cairo 运行环境。但事实上，该词有更加深层的隐喻。我们所学习的 `cairo 1` 都是无状态的，无法进行数据存储等操作，而缺少数据存储等操作显然无法构造真正的应用。所以 `StarkNet` 开发者为 Cairo 提供了一系列函数用于拓展 Cairo 编程语言。当 cairo 调用到这些函数后，类似操作系统中的应用进行系统调用，控制权会被转移到 cairo 外的程序，这一过程也类似操作系统中的用户态到内核态的转移。基于这种相似性，我们称 starknet 项目组构造的 cairo 运行时环境为 `StarknetOS` 。

> 这样解释了在测试过程中，我们为什么使用 `cairo-test --starknet .` 。此处的 `--starknet` 就意味着测试过程中应为运行时增加 `StarknetOS` 提供的系统调用

所有 `StarknetOS` 提供的系统调用都可以在 [corelib/src/starknet/syscalls.cairo](https://github.com/starkware-libs/cairo/blob/main/corelib/src/starknet/syscalls.cairo) 中找到。如果读者访问此文件，会发现文件中的函数都没有定义，这是因为这些函数的逻辑代码都是在 cairo 运行环境中实现的。

最后，我们讨论费用问题。不难发现，L2 -> L1 的信息传递最大的花销在于 L1 中的核心合约将消息哈希值写入以太坊状态。这一部分会消耗 20k gas ，当然事件的抛出也会消耗一定 gas。starknet 在其 gas 计算方法上应该已经兼顾这一部分，读者不太用担心交易失败的问题。

### L1 -> L2

相比于 L2 -> L1 的信息传递需要 ciaro 发送和 solidity 接受，L1 -> L2 的跨链调用则更加简单，仅需要 L1 合约发起请求即可。但另一方面，L1 -> L2 的交易费用是重点，一旦计算操作很有可能出现 ETH 丢失的现象。

> 此处我们使用了 **跨链调用** 一词，L1 -> L2 的信息发送一定会触发 L2 合约函数，我们会在后文分析其具体原理。

跨链调用的第一步是 L1 合约调用 starknet 核心合约的 `sendMessageToL2` 函数，该函数 [源代码](https://github.com/starkware-libs/cairo-lang/blob/master/src/starkware/starknet/solidity/StarknetMessaging.sol#L110) 如下:

```solidity
function sendMessageToL2(
    uint256 toAddress,
    uint256 selector,
    uint256[] calldata payload
) external payable override returns (bytes32, uint256) {
    require(msg.value > 0, "L1_MSG_FEE_MUST_BE_GREATER_THAN_0");
    require(msg.value <= getMaxL1MsgFee(), "MAX_L1_MSG_FEE_EXCEEDED");
    uint256 nonce = l1ToL2MessageNonce();
    NamedStorage.setUintValue(L1L2_MESSAGE_NONCE_TAG, nonce + 1);
    emit LogMessageToL2(msg.sender, toAddress, selector, payload, nonce, msg.value);
    bytes32 msgHash = getL1ToL2MsgHash(toAddress, selector, payload, nonce);
    // Note that the inclusion of the unique nonce in the message hash implies that
    // l1ToL2Messages()[msgHash] was not accessed before.
    l1ToL2Messages()[msgHash] = msg.value + 1;
    return (msgHash, nonce);
}
```

简单来说，该函数的运行逻辑为:

1. 检查 `msg.value` 的范围，该笔转入的 ETH 就是 L2 合约调用的 gas 
2. 增加 `nonce`
3. 释放事件
4. 计算 `msgHash` 并进行数据写入，注意此处写入的是 `msg.value + 1` 参数

一旦事件 `LogMessageToL2` 抛出，节点就会捕获此消息，等待 L1 中的跨链调用达到一定数量后，节点会统一抽取 `toAddress` 等参数进行 L2 合约调用。但需要注意的是，合约调用不是任意的，其要求调用的函数具有 `#[l1_handler]` 注释，如下:

```rust
#[l1_handler]
fn handle_deposit(from_address: felt252, account: ContractAddress, amount: u256) {
    ...
}
```

只有带有 `#[l1_handler]` 的函数会被调用，否则会出现调用失败情况。此处的 `from_address` 是进行跨链调用的 L1 合约地址，通过此参数，我们可以避免函数被任意调用。

下图展示了 [voyager](https://goerli.voyager.online/txns) 区块链浏览器中对 `L1_HANDLER` 这种特殊的 L1 调用交易标识:

![L1 Handler Voyer](https://img.gejiba.com/images/2e20dae76562b3734d3bd43b6bab2cf4.png)

当节点在 L2 完成合约调用后，L2 节点会对所有 **请求成功** 的交易的参数进行进行序列化(类似 L2 -> L1 参数的序列化，即 `abi.encodePacked`)，并将其作为参数调用核心合约，核心合约会根据参数进行更新，[源代码](https://github.com/starkware-libs/cairo-lang/blob/master/src/starkware/starknet/solidity/Output.sol#L125) 如下:

```solidity
// 计算信息哈希并重置该消息的此存储槽
{
    bytes32 messageHash = keccak256(
        abi.encodePacked(programOutputSlice[offset:endOffset])
    );

    uint256 msgFeePlusOne = messages[messageHash];
    require(msgFeePlusOne > 0, "INVALID_MESSAGE_TO_CONSUME");
    totalMsgFees += msgFeePlusOne - 1;
    messages[messageHash] = 0;
}
// 释放事件
uint256 nonce = programOutputSlice[offset + MESSAGE_TO_L2_NONCE_OFFSET];
uint256[] memory messageSlice = (uint256[])(
    programOutputSlice[offset + MESSAGE_TO_L2_PREFIX_SIZE:endOffset]
);
emit ConsumedMessageToL2(
    // from=
    address(programOutputSlice[offset + MESSAGE_TO_L2_FROM_ADDRESS_OFFSET]),
    // to=
    programOutputSlice[offset + MESSAGE_TO_L2_TO_ADDRESS_OFFSET],
    // selector=
    programOutputSlice[offset + MESSAGE_TO_L2_SELECTOR_OFFSET],
    // payload=
    messageSlice,
    // nonce =
    nonce
);
```

此处的代码较为简单，复杂的部分在于反序列化，但即使不理解反序列化，我们也理解核心逻辑。最后，由于 `gas` 支付是在 L1 完成的，所以我们需要将该笔 gas 转给节点，[源代码](https://github.com/starkware-libs/cairo-lang/blob/master/src/starkware/starknet/solidity/Output.sol#L157) 如下:

```solidity
if (totalMsgFees > 0) {
    // NOLINTNEXTLINE: low-level-calls.
    (bool success, ) = msg.sender.call{value: totalMsgFees}("");
    require(success, "ETH_TRANSFER_FAILED");
}
```

如果我们的跨链调用成功，则会遵循上述流程。简单来说，就是我们向 L1 核心合约输入 L2 交易参数和交易 gas ，L2 节点会捕获这一请求，然后代替我们在 L2 上进行交易请求。所以此处我们称其为跨链调用，而不是跨链信息。

总结来看，L1 -> L2 的跨链调用如下图:

![L1 -> L2 call](https://files.catbox.moe/xxj9kt.svg)

但是上述只是交易成功的情况，但也有可能交易失败，一般来说，有以下失败原因:

1. L1 转入的 L2 交易 gas 费用不足，较常见
2. L1 传入的 L2 交易参数构造存在问题

这些失败的交易不会被纳入核心合约的更新。那是否意味着我们转入的 ETH 被永远锁定在了核心合约里？其实不是，starknet 开发者考虑到了这一点，所以提供了一个特殊的 `startL1ToL2MessageCancellation` 函数，其源代码如下:

```solidity
function startL1ToL2MessageCancellation(
    uint256 toAddress,
    uint256 selector,
    uint256[] calldata payload,
    uint256 nonce
) external override returns (bytes32) {
    emit MessageToL2CancellationStarted(msg.sender, toAddress, selector, payload, nonce);
    bytes32 msgHash = getL1ToL2MsgHash(toAddress, selector, payload, nonce);
    uint256 msgFeePlusOne = l1ToL2Messages()[msgHash];
    require(msgFeePlusOne > 0, "NO_MESSAGE_TO_CANCEL");
    l1ToL2MessageCancellations()[msgHash] = block.timestamp;
    return msgHash;
}
```

如果在一段时间内，我们没有观察到发起交易的 `ConsumedMessageToL2` 事件，那么可能意味着交易的失败。我们需要调用 `startL1ToL2MessageCancellation` 函数来取消这笔错误交易，该函数设定了 5 天的等待周期。在等待 5 天后，我们可以调用 `cancelL1ToL2Message` 函数来释放 `MessageToL2Canceled` 事件，[源代码](https://github.com/starkware-libs/cairo-lang/blob/master/src/starkware/starknet/solidity/StarknetMessaging.sol#L161) 如下:
 
```solidity
function cancelL1ToL2Message(
    uint256 toAddress,
    uint256 selector,
    uint256[] calldata payload,
    uint256 nonce
) external override returns (bytes32) {
    emit MessageToL2Canceled(msg.sender, toAddress, selector, payload, nonce);
    // Note that the message hash depends on msg.sender, which prevents one contract from
    // cancelling another contract's message.
    // Trying to do so will result in NO_MESSAGE_TO_CANCEL.
    bytes32 msgHash = getL1ToL2MsgHash(toAddress, selector, payload, nonce);
    uint256 msgFeePlusOne = l1ToL2Messages()[msgHash];
    require(msgFeePlusOne != 0, "NO_MESSAGE_TO_CANCEL");

    uint256 requestTime = l1ToL2MessageCancellations()[msgHash];
    require(requestTime != 0, "MESSAGE_CANCELLATION_NOT_REQUESTED");

    uint256 cancelAllowedTime = requestTime + messageCancellationDelay();
    require(cancelAllowedTime >= requestTime, "CANCEL_ALLOWED_TIME_OVERFLOW");
    require(block.timestamp >= cancelAllowedTime, "MESSAGE_CANCELLATION_NOT_ALLOWED_YET");

    l1ToL2Messages()[msgHash] = 0;
    return (msgHash);
}
```

注意是检查时间是否达到要求。此函数中没有退回 ETH 的函数，这是因为错误交易也消耗了 gas ，所以核心合约不会退回 gas ，此函数的意义在于开发者可以通过调用 `cancelL1ToL2Message` 来判断交易是否处于可取消状态。如果 `cancelL1ToL2Message` 调用成功，那么开发者可以在函数内编写代币铸造等功能帮助用户恢复资产。我们会在后文给出此过程的具体编程方法。

一般来说，退回 gas 都是因为 L2 gas 不足而交易失败导致的，所以如何准确评估 L2 交易 gas 消耗是一个问题，万幸的是，starknet 提供了用于评估交易 gas 消耗的 `estimate_fee` 接口，读者可以提供 `starknet` 的 CLI 进行调用，调用方法如下:

```bash
starknet estimate_fee
    --address <contract_address>
    --abi <contract_abi>
    --function <function_name>
    --inputs <arguments>
```

由于笔者没有安装此工具，所以读者可以自行参考 [文档](https://docs.starknet.io/documentation/tools/CLI/commands/#starknet_estimate_fee) 。

## 合约编程

在了解基本原理后，我们需要进行合约编程，正如本文开头所说，此任务可以分解为以下两部分:

1. 在 HelloERC20 cairo 智能合约基础上实现 `transferToL1` 函数
1. 部署以太坊 ERC20 代币并实现 `transfer_to_L2` 函数

在真实的编程环境中，我建议读者先完成 L2 -> L1 的跨链，因为此部分跨链是异步的，我们可以 fork 核心合约状态来进行测试。而 L1 -> L2 的跨链调用是自动的，该部分仅需要对 L2 cairo 合约进行简单修改即可。

本文将编写一个原生具有可跨链能力的 ERC20 合约。在其 L2 合约上提供以下函数:

1. `transfer_to_L1(l1_recipient, amount)` 调用此函数会烧毁调用会将资产发送到 L1 并在 L2 中烧毁用户的资产
2. `despoit_from_L1` 属于 `#[l1_handler]` 用于接受 L1 的存款请求。

而在 L1 的合约上，我们提供以下函数:

1. `transferToL2` 调用此函数将资产发送到 L2 且在 L1 烧毁资产
2. `despoitFromL2` 调用此函数时接受 L2 转入资产

### L2 -> L1

在本节中，我们首先介绍如何编写 L2 cairo 合约使其向 L1 发送信息，等待信息被 L1 确定后，我们再编写 L1 solidity 合约来消费此消息。值得注意的是，L2 -> L1 的信息跨链需要 4 小时或 8 小时时间，读者可以直接使用本文给出的 `messageHash` 进行测试。

> 本文选择将跨链信息的接受方设置为 EOA 地址，不可能被消费。在测试过程中，我们可以使用 `foundry` 的 `cheatcode` 调整合约部署地址

#### L2 cairo 合约
在此流程中，我们首先修改 cairo 合约。由于 starknet 的地址格式与以太坊不同，所以我们需要构造一个用于生成以太坊地址的模块，创建 `helloERC20/src/utils/eth_address.cairo` 文件，写入以下内容:

```rust
use serde::Serde;
use zeroable::Zeroable;
use traits::Into;

#[derive(Copy, Drop)]
struct EthAddress {
    address: felt252, 
}
trait EthAddressTrait {
    fn new(address: felt252) -> EthAddress;
}
impl EthAddressImpl of EthAddressTrait {
    // Creates a EthAddress from the given address, if it's a valid Ethereum address. If not,
    // panics with the given error.
    fn new(address: felt252) -> EthAddress {
        // TODO(yuval): change to a constant once u256 literals are supported.
        let ETH_ADDRESS_BOUND = u256 { high: 0x100000000_u128, low: 0_u128 }; // 2 ** 160

        assert(address.into() < ETH_ADDRESS_BOUND, 'INVALID_ETHEREUM_ADDRESS');
        EthAddress { address }
    }
}
impl EthAddressIntoFelt252 of Into<EthAddress, felt252> {
    fn into(self: EthAddress) -> felt252 {
        self.address
    }
}
impl EthAddressSerde of Serde<EthAddress> {
    fn serialize(ref output: Array<felt252>, input: EthAddress) {
        Serde::<felt252>::serialize(ref output, input.address);
    }
    fn deserialize(ref serialized: Span<felt252>) -> Option<EthAddress> {
        // Option::Some(EthAddressTrait::new(*serialized.pop_front()?))
        Option::Some(EthAddressTrait::new(Serde::<felt252>::deserialize(ref serialized)?))
    }
}
impl EthAddressZeroable of Zeroable<EthAddress> {
    fn zero() -> EthAddress {
        EthAddressTrait::new(0)
    }

    #[inline(always)]
    fn is_zero(self: EthAddress) -> bool {
        self.address.is_zero()
    }

    #[inline(always)]
    fn is_non_zero(self: EthAddress) -> bool {
        !self.is_zero()
    }
}
```

此代码看上去复杂，实际较为简单，核心逻辑是在 felt252 内存储 160 位长的以太坊地址，由于 felt 完全可以存储下一个以太坊地址，所以此处的核心函数是 `EthAddressImpl` 中的 `new` 函数。其他的代码都是对一些接口的实现。此处我们着重探讨以下代码:

```rust
impl EthAddressSerde of Serde<EthAddress> {
    fn serialize(ref output: Array<felt252>, input: EthAddress) {
        Serde::<felt252>::serialize(ref output, input.address);
    }
    fn deserialize(ref serialized: Span<felt252>) -> Option<EthAddress> {
        // Option::Some(EthAddressTrait::new(*serialized.pop_front()?))
        Option::Some(EthAddressTrait::new(Serde::<felt252>::deserialize(ref serialized)?))
    }
}
```

此处是对 `Serde` trait 的实现，`Serde` 是用于序列化与反序列化的模块，其功能是将定义的结构体序列化为数组或者进行相反操作。此处我们定义的 `EthAddress` 作为 felt252 类型本身不需要较多的序列化，所以此处的对 `EthAddress` 序列化与反序列化的定义都较为简单。关于 `Serde` 更多的信息可以参考 [Cairo Book](https://cairo-book.github.io/appendix-03-derivable-traits.html#serializing-with-serde) 中的内容

我们可以注意到 `deserialize` 返回了 `Option<T>` 数据类型，`Option<T>` 数据类型对 rust 工程师来说应该非常熟悉，该数据类型会返回一个包装结果，该结果有可能是返回的正确的结果，也有可能为空(Null)。此处的可能产生的空值来自 `Serde::<felt252>::deserialize(ref serialized)?` ，最后的 `?` 用于简化处理 `Serde::<felt252>::deserialize` 的返回值。

> `Serde::<felt252>::deserialize` 的返回值为 `Option::Some` ，使用 `?` 可以处理此返回值。关于 `Option<T>` 的更多内容，读者可以参考 [Rust 文档](https://doc.rust-lang.org/std/option/enum.Option.html)

此处另一个未见过的关键词是 `ref` ，该关键词表示传入变量为引用。

> 上述 `eth_address` 看似复杂，但由于 `EthAddress` 数据类型本质上就是 `felt252` 类型，所以大部分实现都是直接调用了 `felt252` 函数的实现。此处使用的 `trait` 都来自 `corelib` ，读者可以自行查阅其源代码。

完成上述模块编程后，创建 `helloERC20/src/utils.cairo` 文件，写入以下内容:

```rust
mod eth_address;
```

并在 `lib.cairo` 中写入以下内容:

```rust
mod ERC20;
mod utils;

#[cfg(test)]
mod tests;
```

在 cairo 设计的模块系统内，在检索模块时仅会在 `lib.cairo` 中检索，而 `lib.cairo` 中只能引用同级文件夹下的 `cairo` 文件，所以出现上述情况。我们通过创建与 `uitls/` 文件夹同名的 `utils.cairo` ，实现了在 `utils.cairo` 中引用 `eth_address` 模块，此时 `utils.cairo` 与 `lib.cairo` 位于同级文件夹下，所以 `lib.cairo` 可以通过 `mod utils;` 引用 `utils.cairo`。通过上述流程，我们实现了 `eth_address` 在命名空间内的注册。

首先，我们需要增加 `burn` 函数，代码如下:

```rust
fn burn(amount: u256) {
    let zero_address = contract_address_const::<0>();
    let sender = get_caller_address();
    _total_supply::write(_total_supply::read() - amount);
    _balances::write(sender, _balances::read(sender) - amount);
}
```

注意此函数没有 `#[external]` 标识。此处我们省略测试代码，该函数的测试与 `mint` 函数类似。

我们也需要修正 `stroage` 结构体，增加 `governor` 和 `l1_token` 变量，前者指 `owner` 而后者指 L1 上的代币合约。修改后的 `Storage` 如下:

```rust
struct Storage {
    ...,
    _allowances: LegacyMap<(ContractAddress, ContractAddress), u256>,
    governor: ContractAddress,
    l1_token: felt252,
}
```

由于增加了 `governor` 存储变量，我们也需要修改 `constructor` 构造器，修改后如下:

```rust
#[constructor]
fn constructor(name: felt252, symbol: felt252, decimals: u8, governor:ContractAddress) {
    _name::write(name);
    _symbol::write(symbol);
    _decimals::write(decimals);
    governor::write(governor);
}
```

此处不要使用 `governor::write(get_caller_address())` 设置 `owner` ，因为如果你使用的是 `ArgentX` 钱包，其合约部署是通过调用 `UniversalDeployer` 合约实现的，这意味如果使用 `get_caller_address()` 函数，则 `governor` 会被设置为 `UniversalDeployer` 合约地址。

更改 `constructor` 后，也需要修改一部分单元测试，请读者自行更改。

增加以下函数用于设置 L1 代币合约地址:

```rust
#[external]
fn set_l1_token(l1_token_address: EthAddress) {
    // The call is restricted to the governor.
    assert(get_caller_address() == governor::read(), 'GOVERNOR_ONLY');

    assert(l1_token::read().is_zero(), 'L1_token_ALREADY_INITIALIZED');
    assert(l1_token_address.is_non_zero(), 'ZERO_token_ADDRESS');

    l1_token::write(l1_token_address.into());
    l1_token_set(l1_token_address);
}
```

`set_l1_token` 函数较为简单，但我们暂时没有引入 `EthAddress` 类型，请读者使用以下代码引入:

```rust
use helloERC20::utils::eth_address::EthAddress;
```

读者可能发现报错中提示 `is_zero` 和 `is_non_zero` 方法也不存在，所以我们需要引入:

```rust
use zeroable::Zeroable;
use helloERC20::utils::eth_address::EthAddressZeroable;
```

最后，`l1_token_address` 为 `EthAddress` 类型，而 `l1_token` 为 `felt` 类型，所以此处需要使用 `l1_token_address.into()` 进行转换。此转换是我们之前在 `EthAddressIntoFelt252` 中定义的，而不是原生存在的方法。读者可能发现此处提示函数不存在，我们需要导入以下内容来使用 `into` 方法:

```rust
use traits::Into;
use helloERC20::utils::eth_address::EthAddressIntoFelt252;
```

关于测试部分，读者可以自行参考 [github 仓库](https://github.com/wangshouh/helloERC20/blob/bridge/src/tests/ERC20_test.cairo) 。

完成这些繁琐的基本任务后，我们开始真正编写跨链函数 `transfer_to_L1` ，其实也比较简单，代码如下:

```rust
#[external]
fn transfer_to_L1(l1_recipient: EthAddress, amount: u256) {
    burn(amount);
    // Call burn on l2_token contract.
    let caller_address = get_caller_address();

    // Send the message.
    let mut message_payload: Array<felt252> = ArrayTrait::new();
    message_payload.append(l1_recipient.into());
    message_payload.append(amount.low.into());
    message_payload.append(amount.high.into());

    send_message_to_l1_syscall(
        to_address: l1_token::read(), payload: message_payload.span()
    );
    TransferToL1(l1_recipient, amount, caller_address);
}
```

代码比较简单，在 L2 上烧毁用户资产，之后将接收方和发送资产金额作为跨链信息发送到核心合约中。

完成上述工作后，我们开始部署此合约。这种跨链合约显然无法较为简单的进行本地测试，直接使用测试网可能是一个更好的选择。

在部署过程中，直接使用 `nile-rs complie` 进行编译会出现编译报错，这可能是由于 `nile-rs` 的编译器版本问题，我们可以直接使用以下命令进行编译:

```bash
starknet-compile . target/release/bridgeERC20_ERC20.json
```

在部署过程中需要注意，由于 nile-rs 的 gas 计算似乎有些问题，所以如果使用默认 max-fee 会出现交易失败的情况，如 [这笔交易](https://testnet.starkscan.co/tx/0xe636d8e82b2a9cbe76bc7d227f843d788de53ba36e8b73cd907fa1765e597e)，手动调整 `max-fee` 可以避免这种情况，命令如下:

```bash
nile-rs declare -p PRIVATE_KEY -n goerli bridgeERC20_ERC20 -t
nile-rs deploy -p PRIVATE_KEY -m 1025377281411474 -n goerli bridgeERC20_ERC20 'BRIDGE' 'BE' 18 $OWNER_ADDRESS
```

请读者自行设置 `OWNER_ADDRESS` 变量。

> 不太清楚为什么 `nile-rs` 的 gas 变量计算存在问题，目前，gas 不足的错误交易会被不会被记录到区块链上，且不扣用户 gas 费用。当然，不上链也可以使用区块链浏览器进行检索。注意，官方文档中声明错误交易不消耗 gas 只是权宜之计，为了更新后会移除此特性。

部署完成后，读者可以调用 `set_l1_token` 函数设置一个 L1 地址，设置完成后，调用 `transfer_to_L1` 函数，读者可以点击 [此链接](https://testnet.starkscan.co/tx/0x36811d58cb4ef9bf804c0c22a0152797dd2bd69613fe4964dfd39756f82d68#overview) 查看示例交易。

点击 `Message Logs` 选项卡，我们可以看到 `L2 -> L1` 的信息，如下:

![StarkScan L2 -> L1](https://img.gejiba.com/images/1e148001157ceaaaf2cdb2fc37b7c62f.png)

读者可以点击 `Message Hash` 查看信息传递详情，或者点击 [此链接](https://testnet.starkscan.co/message/0x829b7b9b220945a1e3c40d04eb2b6c38b0ee7ff6f54049bbb4b9ea87d021b21a#overview) 。我们主要关注 `Status` ，该内容说明当前跨链信息是否被 L1 接受。一般来说，L2 -> L1 的跨链信息传递需要 4 个小时。然后，我们的重点在于 `Payload`，如下图:

![StarkNet L2 -> L1 Payload](https://img.gejiba.com/images/0ab82705f2b2901ca1e4263c8a804a34.png)

该 `payload` 与我们的代码是对应的，`Index 0` 为接收方地址，`Index 1` 为转移代币数量 `amount` 的低 128 位，而 `Index 2` 为转移代币数量的高 128 位。

> 此处又显示了为了使用 256 bit 而带来的复杂度，一个可行的解决方案是放弃使用 u256 数据类型，而使用 `felt252` 数据类型作为 `amount` 参数。在大部分情况下，使用 252 bit 的 `felt252` 作为 `amount` 是足够的。

我们可以观察到 `Message Hash
` 为 `0x829b7b9b220945a1e3c40d04eb2b6c38b0ee7ff6f54049bbb4b9ea87d021b21a`。之前，我们介绍过此值的计算方法:

```solidity
keccak256(
    abi.encodePacked(
        FromAddress,
        ToAddress,
        Payload.length,
        Payload
    )
);
```

此处，我们进行使用 `solidity` 进行一次验算。我们假设读者系统环境内包含 `foundry` 以太坊开发工具，在终端键入 `chisel` 以启动 `foundry` 的 solidity REPL 工具。分别键入以下命令:

```solidity
uint256 fromAddress = 0x03df9e4bb7dbee67fbfb50d5e1e8205df55c7b85789b1eb0dca618c390c0ffee;
uint256 toAddress = 0x00afd48f565e1ac63f3e547227c9ad5243990f3d40;
uint256 length = 3;
uint256[] memory payload = new uint256[](3);
payload[0] = 98643737269556690607045493661323739746101663068;
payload[1] = 100;
keccak256(abi.encodePacked(fromAddress, toAddress, length, payload))
```

我们可以看到如下输出:

![Chisel Message Hash](https://img.gejiba.com/images/b6f40e26f2e07c33a05d94589bb98d96.png)

此处使用了 `0x00afd48f565e1ac63f3e547227c9ad5243990f3d40` 而不是 `0xafd48f565e1ac63f3e547227c9ad5243990f3d40` 是因为使用后者会报错。

耐心等待 4 小时或 8 小时，我们可以在 [核心合约](https://goerli.etherscan.io/address/0xde29d060D45901Fb19ED6C6e959EB22d8626708e#readProxyContract) 查询到此信息，如下图:

![Message Hash Query](https://img.gejiba.com/images/0bcd2715d226d943e729e5157f377ad1.png)

#### solidity 合约

接下来，我们就可以编写 solidity 合约来消费此消息。此处我们需要实现 `despoitFromL2` 函数。

我们使用了 [solmate ERC20](https://github.com/transmissions11/solmate/blob/main/src/tokens/ERC20.sol) 作为底稿进行修改。为了保持合约的简单性，我们删除来 `EIP-2612` 部分代码。为了使合约具有跨链功能，我们需要在合约头部加入以下声明:

```solidity
address public starkNetAddress;
```

此变量用于存储核心合约地址。我们也需要在构造器中对此变量进行初始化:

```solidity
constructor(
    string memory _name,
    string memory _symbol,
    uint8 _decimals,
    address _starkNetAddress
) {
    name = _name;
    symbol = _symbol;
    decimals = _decimals;
    starkNetAddress = _starkNetAddress;
}
```

事实上，此处的构造器并不会在后文测试中使用。

接下来，我们编写 `payload` 解析器，代码如下:

```solidity
function payloadParese(uint256[] calldata payload) internal pure returns (address, uint256) {
    address receiver = address(uint160(payload[0]));
    uint256 amount = payload[2] << 128 | payload[1];
    return (receiver, amount);
}
```

在上文中，我们定义了 cairo 1 合约传递的 `payload` 的内容如下:

```rust
message_payload.append(l1_recipient.into());
message_payload.append(amount.low.into());
message_payload.append(amount.high.into());
```

此处的代码是对 `payload` 的解析，此处需要注意 `amount` 为两个 `uint128` 拼接得到的，我们在此处使用位移操作将 payload 的 `amount` 进行了重现拼接。

接下来，我们可以进行跨链函数 `despoitFromL2` 的编写，代码如下:

```solidity
function despoitFromL2(uint256 fromAddress, uint256[] calldata payload) external {
    IStarknetMessaging(starkNetAddress).consumeMessageFromL2(
        fromAddress,
        payload
    );
    (address receiver, uint256 amount) = payloadParese(payload);
    _mint(receiver, amount);
}
```

相对来说，合约实现上比较简单。核心合约提供的 `consumeMessageFromL2` 函数实现了一系列功能，特别是如果无法查询到对应跨链信息的哈希值直接 `revert` 的功能大大简化了我们的后期处理。当然，此处我们没有进行事件抛出，读者可根据自身业务需求增加此部分。

> 此处，我们需要引入 `IStarknetMessaging` 接口，限于篇幅限制，我们省略了此部分。

接下来，我们进行测试。测试较为简单，唯一的复杂度在于我们需要构造一个核心合约的 `mock` 版本。代码如下:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.15;

contract Core {

    mapping(bytes32 => uint256) public l2ToL1Messages;

    function consumeMessageFromL2(uint256 fromAddress, uint256[] calldata payload)
        external
        returns (bytes32)
    {
        l2ToL1Messages[0x829b7b9b220945a1e3c40d04eb2b6c38b0ee7ff6f54049bbb4b9ea87d021b21a] = 1;

        bytes32 msgHash = keccak256(
            abi.encodePacked(fromAddress, uint256(uint160(msg.sender)), payload.length, payload)
        );

        require(l2ToL1Messages[msgHash] > 0, "INVALID_MESSAGE_TO_CONSUME");
        // emit ConsumedMessageToL1(fromAddress, msg.sender, payload);
        l2ToL1Messages[msgHash] -= 1;
        return msgHash;
    }
}
```

总体来说，比较简单。在测试观察中，我们直接使用了 `0x829b7b9b220945a1e3c40d04eb2b6c38b0ee7ff6f54049bbb4b9ea87d021b21a` 信息，该该消息的接收方为 `0xAFD48f565e1aC63f3e547227c9AD5243990f3D40` ，所以我们需要在指定地址进行合约部署。此处，我们使用了 `vm.etch` cheatcode 函数，此函数可以直接向指定地址内写入合约字节码。我们编写测试代码如下:

```solidity
core = new Core();
uint256 L2Address = 0x03df9e4bb7dbee67fbfb50d5e1e8205df55c7b85789b1eb0dca618c390c0ffee;
token_code = new BridgeERC20("BRIDGE", "BE", 18, address(core), L2Address);

bytes memory code = address(token_code).code;
targetAddr = address(0xAFD48f565e1aC63f3e547227c9AD5243990f3D40);
vm.etch(targetAddr, code);

uint256 slot = stdstore
    .target(targetAddr)
    .sig("starkNetAddress()")
    .find();
bytes32 loc = bytes32(slot);
bytes32 mockedCore = bytes32(abi.encode(address(core)));
vm.store(targetAddr, loc, mockedCore);
```

此处使用 `vm.etch` 将 `BridgeERC20` 的字节码直接写入了 `0xAFD48f565e1aC63f3e547227c9AD5243990f3D40` 地址内，但是字节码内缺少部分参数的初始化，所以此处我们使用 `vm.store` 对指定变量进行了强行初始化。

### L1 -> L2

相比于 L2 -> L1 的缓慢，L1 -> L2 是高速的。我们首先构造 L1 合约中的核心函数 `transferToL2` ，此函数用于发送资产。但是正如上文所述，资产发送可能因为 L1 交易缴纳的 gas 不足而失败，所以我们也要实现交易取消和资产取回等函数。

#### cairo 合约

我们首先需要增加一个 `event` 方便后期判断:

```rust
 #[event]
fn DepositFromL1(account: ContractAddress, amount: u256) {}
```

接下来，我们构造 `despoit_from_L1` 函数，此函数也比较简单:

```rust
#[l1_handler]
fn despoit_from_L1(from_address: felt252, account: ContractAddress, amount: u256) {
    assert(from_address == l1_token::read(), 'EXPECTED_FROM_BRIDGE_ONLY');

    _total_supply::write(_total_supply::read() + amount);
    _balances::write(account, _balances::read(account) + amount);

    DepositFromL1(account, amount);
}
```

此处用到了 `u256` 类型，cairo 1 在传入参数时可以解析这一类型。

#### solidity 合约

此处主要实现 `transferToL2` 函数，此函数的核心在于调用核心合约的 `sendMessageToL2` 方法。一个简单的实现如下:

```solidity
function transferToL2(uint256 L2Address, uint256 amount) payable external returns (uint256) {
    _burn(msg.sender, amount);
    uint256[] memory payload = generatePayload(L2Address, amount);

    (bytes32 msgHash,uint256 nonce) = IStarknetMessaging(starkNetAddress).sendMessageToL2{value: msg.value}(
        L2TokenAddress, SELECTOR, payload
    );

    emit MessageHash(msgHash);

    nonceValue[nonce][msg.sender] = amount;

    return nonce;
}
```

此处使用 `generatePayload` 函数生成 Payload ，具体是实现如下:

```solidity
function generatePayload(
    uint256 L2Address, 
    uint256 amount
) internal pure returns (uint256[] memory payload) {
    payload = new uint256[](3);
    uint128 low = uint128(amount);
    uint128 high = uint128(amount >> 128);

    payload[0] = L2Address;
    payload[1] = low;
    payload[2] = high;
}
```

`transferToL2` 函数另一需要注意的是 `nonceValue` 映射，此映射定义为 `mapping(uint256 => mapping(address => uint256)) public nonceValue;`，用于存储用户的 `nonce` 与用户地址和转移至 L2 的代币的关系。该映射为后文 `startCancel` 和 `cancel` 函数使用，即用于取消 L2 交易以拿回资产。

上述 `startCancel` 和 `cancel` 函数定义如下:

```solidity
function startCancel(uint256 L2Address, uint256 nonce) external {
    uint256 amount = nonceValue[nonce][msg.sender];
    require(amount > 0, "NONCE_NOT_EXIST");
    uint256[] memory payload = generatePayload(L2Address, amount);
    IStarknetMessaging(starkNetAddress).startL1ToL2MessageCancellation(
        L2TokenAddress, SELECTOR, payload, nonce, SELECTOR, payload, nonce
    );
}

function cancel(uint256 L2Address, uint256 nonce) external {
    uint256 amount = nonceValue[nonce][msg.sender];
    require(amount > 0, "NONCE_NOT_EXIST");
    uint256[] memory payload = generatePayload(L2Address, amount);
    IStarknetMessaging(starkNetAddress).cancelL1ToL2Message(
        L2TokenAddress, SELECTOR, payload, nonce, SELECTOR, payload, nonce
    );
    nonceValue[nonce][msg.sender] = 0;
    _mint(msg.sender, amount);
}
```

此处将大量的安全检查委托给了 `startL1ToL2MessageCancellation` 和 `cancelL1ToL2Message` 函数。我们仅对 `msg.sender` 的 `nonce` 进行检查，这是为了避免用户恶意操作来取消他人的交易。

关于测试，我们不再进行详细讨论，读者可自行参考文档。值得注意的是，我们没有对 `cancel` 函数进行完整的测试。这是因为 核心合约的 `cancelL1ToL2Message` 实现包含大量参数，所以笔者没有在 `mock` 合约中进行实现，如果读者的项目对安全性要求比较高，建议进行完整测试。

> 此处，一个更好的测试方案是使用 `forking` 系列 cheatcode 直接将主网核心合约 fork 到本地进行测试，读者可以自行参考 [相关文档](https://book.getfoundry.sh/cheatcodes/forking)。此方法不需要在本地构造 `mock` 合约，但缺点是缓慢，测试会因为需要 `fork` 主网状态而变得极其缓慢。读者可根据自身需求使用。

## 跨链测试

我们主要进行 L1 -> L2 的跨链测试。

首先，我们需要部署最终的 cairo 合约，关于部署的流程，我们也在上文进行了讨论，简单来说，需要以下命令:

```bash
nile-rs declare -m 1528919118624 -p PRIVATE_KEY -n goerli bridgeERC20_ERC20 -t
nile-rs deploy -p PRIVATE_KEY -m 1025377281411474 -n goerli bridgeERC20_ERC20 'BRIDGE' 'BE' 18 0x046d703Df4Dd4E1B8196B294ec295C8C66097836Cd4085372299Ac53dFf5d478 -t
```

上述命令均增加 `-m` 直接指定交易的 `max-fee` ，这是因为 `nile-rs` 的 gas 设定机制存在一定问题，使用默认的 `max-fee` 容易失败。

部署 cairo 合约完成后，我们需要部署 solidity 合约，创建 `script/BridgeERC20.s.sol` 合约，输入以下命令:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.15;

import "forge-std/Script.sol";
import "../src/BridgeERC20.sol";

contract BridgeScript is Script {
    function run() public returns (BridgeERC20 token) {
        vm.startBroadcast();
        uint256 L2Address = 0x01333558b36dcba8bfdaf329a20a595958bfa2c97aa0332df4350199b740228f;
        address core = 0xde29d060D45901Fb19ED6C6e959EB22d8626708e;
        token = new BridgeERC20("BRIDGE", "BE", 18, core, L2Address);
        vm.stopBroadcast();
    }
}
```

最后，使用以下命令完成 solidity 合约部署:

```bash
forge script script/BridgeERC20.s.sol:BridgeScript --rpc-url $RPC_URL --private-key $PRIVATE_KEY --broadcast --verify --etherscan-api-key $ETHERSCAN_KEY -vvvv
```
 
此处的 `$RPC_URL` 为 goerli 测试网 RPC URL 地址，`$PRIVATE_KEY ` 为私钥，而 `$ETHERSCAN_KEY` 为 EtherScan 网站 API 密钥，主要用于验证合约。

> 关于 solidity 合约部署，比较完整的示例可以参考 [Foundry教程：编写测试部署ERC-20代币智能合约](https://blog.wssh.trade/posts/foundry-with-erc20/) 。

完成上述部署后，我们可以前往 cairo 合约的区块链浏览器页面，调用 `set_l1_token` 函数，如下图:

![Set L1 Token In starkSacn](https://img.gejiba.com/images/1786cbd8f5833f68bfc39b1ad72cdb42.png)

此处使用的区块链浏览器为 [starkscan](https://testnet.starkscan.co) 。

完成此设置后，读者需要在终端中与 L1 合约进行交互。首先，我们使用以下命令铸造代币，如下:

```bash
cast call $bridge "mint(uint256)" 1ether --rpc-url $RPC_URL --private-key $PRIVATE_KEY
cast send $bridge "mint(uint256)" 1ether --rpc-url $RPC_URL --private-key $PRIVATE_KEY
```

此处依旧使用了经典的先 `call` 后 `send` 的方法，`call` 会模拟交易，可以有效避免因填错参数而导致的交易失败情况。

然后，我们进行跨链转账:

```bash
export L2ADDRESS=0x046d703Df4Dd4E1B8196B294ec295C8C66097836Cd4085372299Ac53dFf5d478
cast call $bridge "transferToL2(uint256,uint256)" $L2ADDRESS 0.5ether --value 0.003ether --priate-key $PRIVATE_KEY --rpc-url $RPC_URL
cast send $bridge "transferToL2(uint256,uint256)" $L2ADDRESS 0.5ether --value 0.003ether --private-key $PRIVATE_KEY --rpc-url $RPC_URL
```

此处的 0.003 ether 是根据之前的在 L2 上进行的 `mint` 函数的 gas 花费随意给出的，如果读者可以使用其他工具进行更加准确的评估。建议在评估结果上增加一部分冗余。

读者可以在 etherscan 页面中的合约 `events` 内找到交易的 `msgHash`，如下:

![Etherscan Event MsgHash](https://img.gejiba.com/images/ccd5f038c8e203330e6a1d08d66e4e73.png)

稍等 1-2 分钟，前往 starknet 区块链浏览器直接搜索 `msgHash` ，可以前往此页面:

![StarkScan MessageHash](https://img.gejiba.com/images/b27e03069330ca54f63716feb8611a66.png)

一般来说，如果读者的 L2 代码没有问题，稍等 2 分钟就会变为 `Consumed On L2` 状态，如果 5 分钟内都没有状态变更，可能说明读者的 L2 代码存在问题。但是跨链调用不会给出报错信息，这会使 debug 异常复杂。

至此，我们就完成跨链的全部流程。

## 总结

本文首先介绍 starknet 的可升级合约模式，相比于 solidity 语言中的可升级合约模式，starknet 通过 cairo 1 语言完全基于 哈希地址的存储模式和可更改的 class 逻辑代码两个方面实现极其简单的可升级合约模式。由于此模式过于简单，所以本文没有给出相关实战。

接下来，我们介绍了 starknet 的跨链机制，该机制也比较简单。但是目前缺少很多文档，所以笔者在跨链上花费了不少时间。本文首先源代码介绍了跨链的基本原理，然后使用核心合约一系列函数进行跨链实战，编写了原生支持跨链的 ERC20 代币。