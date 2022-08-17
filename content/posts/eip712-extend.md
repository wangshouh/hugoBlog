---
title: "EIP712的扩展使用"
date: 2022-08-16T10:27:17Z
tags: [EIP-2770,EIP-2771,EIP-2612,EIP-712,solidity]
aliases: ["/2022/08/11/eip712-extend"]
---

## 概述

本文在[上一篇文章](https://hugo.wongssh.cf/posts/ecsda-sign-chain/)介绍的`EIP712`的基础上进一步讨论了`EIP712`结构化哈希的进一步应用:

1. Meta-transactions，解决用户`gas`费用问题
2. ERC20-Permit

## Meta-transactions

`meta-transactions`指在交易中包含另一个实际交易。具体流程为用户签署实际交易，将交易提交给区块链运营商，此过程不需要`gas`费用和与区块链交互。运营商收到用户提交的交易后，由运营商较此交易提交给区块链。此过程实现了的意义在于将用户与区块链交易的`gas`费转移到运营商身上，有效降低了用户使用区块链的门槛。当然，运营商可以直接与矿池合作降低交易费用。

流程图如下:

![Meta transactions flow](https://s-bj-3358-blog.oss.dogecdn.com/svg/mermaid-diagram-2022-08-11-100952.svg)

此流程已被`EIP2771`标准化。

### 合约调用概述

由于此过程设计合约调用的流程，所以我们在此处简单介绍然后实现以太坊合约调用。此处我们仅仅讨论合约交互的基本原理，关于具体实现，读者可自行使用`ethers.js`等库实现。

第一步，生成`calldata`。

我们与合约交互需要生成`calldata`数据。此过程是根据我们输入的数据和`abi`进行编码得到的，在`solidity`中，一般使用`abi.encodeWithSignature`得到，我们在[Foundry教程：使用多种方式编写可升级的智能合约(上)](https://hugo.wongssh.cf/posts/foundry-contract-upgrade-part1/)测试中已经多次使用此函数。当然，我们可以使用`Foundry`提供的`cast`完成此步骤，具体可以参考[cast calldata](https://book.getfoundry.sh/reference/cast/cast-calldata)。在下图中，我们给出一个例子。

![castCalldata.png](https://img.gejiba.com/images/37034a598cffe242e7220aa4c0182a14.png)

除此之外，我们也可以在[网页](https://abi.hashex.org/)中进行操作，示意图如下:
![calldataweb.png](https://img.gejiba.com/images/859a1d3530aa96a81dfe1a4ef7a48784.png)

在`ethers.js`中，此过程在我们进行合约调用时隐形进行。

第二步，生成并签名交易数据。

我们在[上一篇](https://hugo.wongssh.cf/posts/ecsda-sign-chain/)中已经介绍了以太坊交易签名。在此处，我们给出标准交易签名的内容:
```
rlp(
    [
        chain_id, 
        nonce, 
        max_priority_fee_per_gas, 
        max_fee_per_gas, 
        gas_limit, 
        destination, 
        amount, 
        data, 
        access_list
    ]
)
```
此处，我们主要需要将`data`设置为`calldata`，将`destination`设置为合约地址。为了简洁，我们不再详细介绍其他参数。

第三步，发送交易至以太坊节点。当交易发送到以太坊节点后，以太坊节点检验交易的有效性。当以太坊节点查询到交易内的`destination`为合约地址后，以太坊节点将`calldata`内的数据发送到`EVM`中。EVM获得`calldata`数据后，会首先提取`calldata`前4 byte(即函数选择器)，并查询本地的函数选择器映射表选择需要运行的堆栈。完整过程见下图:

![EVM Opcodes](https://img.gejiba.com/images/b922b9236e6a25af7d18d568dfb3d0ef.png)

*上图来自[Deconstructing a Solidity Contract — Part III: The Function Selector](https://blog.openzeppelin.com/deconstructing-a-solidity-contract-part-iii-the-function-selector-6a9b6886ea49/)

完成上述流程后，EVM会输出结果并广播结果实现区块确认。当然，如果合约运行过程中出现代码错误在gas费充足的情况下会抛出异常。

> 注意抛出异常操作仍需要gas费用，所以在下文编写合约时我们使用了`1/64`规则以避免`call`操作耗尽所有gas导致异常无法抛出。

其中，第二三步可以使用`eth_sendtransaction`接口实现，也可以使用`cast send`实现这一过程，具体文档参考[cast send](https://book.getfoundry.sh/reference/cast/cast-send)。

上述过程完整描述了在以太坊区块链中如何完成合约调用。

### 基本流程

从上文中，我们已经知道了合约调用基本流程。我们需要在用户与合约正常的交互中插入运营商的合约，具体步骤可参考下图:
![EIP2771](https://s-bj-3358-blog.oss.dogecdn.com/svg/mermaid-diagram-2022-08-11-181624.svg)

第一步，交易用户需要签名以下数据(由`EIP2770`规定):
```solidity
struct ForwardRequest {
   address from;
   address to;
   uint256 value;
   uint256 gas;
   uint256 nonce;
   bytes data;
   uint256 validUntil;
}
```
其中，各个参数的意义为:

- `from` -  签名者的地址
- `to` - 目标合约地址
- `value` - 向目标合约转移的eth数量
- `gas` - gas 费用
- `nonce` - 交易`nonce`，防止重放攻击
- `data` - 具体的`calldata`，我们需要安装上文的方法进行生成
- `validUntil` - 有效期，一般设置为区块高度。在下文中，我们不会进行实现此参数

将上述参数使用`EIP712`进行架构化哈希得到最终的数据并进行签名。完成签名后将请求发送给运营商。

第二步，运营商得到用户发送的数据和签名后，验证用户签名。验证成功后，提取用户发送的`ForwardRequest`结构体，并使用运营商私钥对其签名并将其发送给可信转发合约。这一步运营商才真正与区块链进行交互，所以运营商需要缴纳`gas`费用。

第三步，可信转发合约会检测运营商发送的请求是否由运营商签名，如果签名确认正确，则会提取出`ForwardRequest`中的各个参数，对`to`地址使用`call`进行调用。

第四步，接受合约接受可信转发合约发送的`call`请求执行合约。

### 合约实现

此处的合约实现主要实现可信转发合约和接受合约。

#### 可信转发合约

此处我们主要基于`openzepplin`库提供的`Meta Transactions`合约为大家介绍合约实现。值得注意的是，目前此合约为最简形式，如果需要使用合约构建大规模系统，请参考[GSN](https://github.com/opengsn/gsn)的合约代码。

首先，我们编写`Forwarder`合约，本合约的具体实现较长，我们在此处仅分析核心函数:

`verify`函数，此函数用于校验请求是否正确:
```solidity
    function verify(ForwardRequest calldata req, bytes calldata signature)
        public
        view
        returns (bool)
    {
        address signer = _hashTypedDataV4(
            keccak256(
                abi.encode(
                    _TYPEHASH,
                    req.from,
                    req.to,
                    req.value,
                    req.gas,
                    req.nonce,
                    keccak256(req.data)
                )
            )
        ).recover(signature);
        return _nonces[req.from] == req.nonce && signer == req.from;
    }
```

此代码较为简单，我们已在[基于链下链上双视角深入解析以太坊签名与验证](https://hugo.wongssh.cf/posts/ecsda-sign-chain/)此文中进行了相关介绍。此函数的功能是验证`ForwardRequest`的请求是否由运营商签名。当然，此处也验证了`nonce`是否正确。设置`nonce`的目的是避免重放攻击。如果不设置此参数，攻击者可以在链上查找到用户的签名并重复使用。加入`nonce`并设置每次运行改变，可以有效避免签名被重复使用。在以太坊正常交易中，用户的`nonce`也会在每次进行交易后自加`1`，也是为了避免重放攻击。

`execute`函数，此函数用于将请求转发给接受合约，并由接受合约运行，代码如下:
```solidity
function execute(ForwardRequest calldata req, bytes calldata signature)
    public
    payable
    returns (bool, bytes memory)
{
    require(
        _senderWhitelist[msg.sender],
        "AwlForwarder: sender of meta-transaction is not whitelisted"
    );
    require(
        verify(req, signature),
        "AwlForwarder: signature does not match request"
    );
    _nonces[req.from] = req.nonce + 1;

    (bool success, bytes memory returndata) = req.to.call{
        gas: req.gas,
        value: req.value
    }(abi.encodePacked(req.data, req.from));

    if (!success) {
        assembly {
            let p := mload(0x40)
            returndatacopy(p, 0, returndatasize())
            revert(p, returndatasize())
        }
    }

    assert(gasleft() > req.gas / 63);

    emit MetaTransactionExecuted(req.from, req.to, req.data);

    return (success, returndata);
}
```
此函数在运行`data`之前进行了一系列检查:

1. 判断请求发送者是否为运营商节点
1. 验证用户签名

完成以上步骤后，进入`data`运行阶段，具体代码如下:
```solidity
(bool success, bytes memory returndata) = req.to.call{
    gas: req.gas,
    value: req.value
}(abi.encodePacked(req.data, req.from));
```
与一般的`call`调用不同，我们没有直接将`req.data`，即`calldata`作为请求体，而是在`req.data`后增加了`req.from`，即用户地址。这样设计是为了被调用的接受合约可以从非标准的`calldata`中获得用户地址，我们会在下文向大家展示提取函数。如果你的接受合约内需要`msg.sender`,则需要对接受合约设计此提取函数。

如果你接受合约不涉及`msg.sender`，此时已经可以宣告合约开发完成。因为当合约接受到非标准的`req.data`后它已经会按照正常方式读取固定部分进行运行，而附加的地址则不再解析范围内，所以不会对函数的正常运行产生影响。

> 在增加地址的过程中，我们使用了`abi.encodePacked`，此函数会实现非标准的abi字节，具体参考[solidity 文档](https://docs.soliditylang.org/en/latest/units-and-global-variables.html#abi-encoding-and-decoding-functions)或[Solidity Tutorial: all about ABI](https://coinsbench.com/solidity-tutorial-all-about-abi-46da8b517e7)，后文可能较为易读。

此处在合约运行失败后使用了内联汇编的方式处理错误，核心为`revert(offset, size)`。此函数可以返回错误，但不消耗所有`gas`，由`EIP140`引入，可以参考[EVM Codes](https://www.evm.codes/#fd)。

此处较难理解的为`assert(gasleft() > req.gas / 63);`，此函代码涉及到`gas`的一些复杂机制，具体可以参考[Ethereum, The Concept of Gas and its Dangers](https://ronan.eth.link/blog/ethereum-gas-dangers/)。总结来说，正如上文所述，此操作为了在`call`运行后需要在本地留下`1/64`的`gas`费以保证转发合约抛出可能异常。

完整代码可以参考[我的仓库](https://github.com/wangshouh/upgradeContractLearn/blob/master/src/metatx/Forwarder.sol)，生产级代码可以参考[GSN](https://github.com/opengsn/gsn/blob/master/packages/contracts/src/forwarder/Forwarder.sol)的转发合约。

#### 接受合约

我们需要特定的合约接受转发合约转发的`req.data`(在原有`calldata`的基础上增加了用户的`address`)并提取出`sender`。为达成此目的，我们使用了以下函数:
```solidity
function _msgSender() internal view virtual returns (address ret) {
    if (msg.data.length >= 20 && isTrustedForwarder(msg.sender)) {
        assembly {
            ret := shr(96, calldataload(sub(calldatasize(), 20)))
        }
    } else {
        ret = msg.sender;
    }
}
```

其中的核心代码为汇编代码部分`ret := shr(96, calldataload(sub(calldatasize(), 20)))`。此代码的作用原理如下图:
![msgaddress.drawio.png](https://img.gejiba.com/images/3b3676fcd7299731ca4855302ba3bcb5.png)

简单来说，可以将`calldata`视为一个长度为`calldatasize()`的列表。我们需要获得此列表中最后`20 byte`的数据，即用户地址。已知`calldataload(i)`会加载`calldata[i, -1]`的数据。我们通过`sub(calldatasize(), 20)`获得了`calldata`中用户地址的起始索引，并进一步使用`calldataload`将其加载到内存中。但在`EVM`中，一个标准不可变变量应占用`32 byte`的完整地址槽，而此处获得用户地址作为`address`类型变量占用的内存长度与规定不符。为了符合变量标准，我们使用`shr`操作码将`20 byte`的用户地址向左移`96 bit`(即 12 byte)实现了用户地址占用`32 byte`的条件，保证了在后期读取用户地址时不会出现错误。

如果你无法理解上述内容，建议参考:

- [EVM Code](https://www.evm.codes/)，此网站可以查询所有`EVM`汇编指令的参数及作用；
- [A Practical Introduction To Solidity Assembly: Part 0](https://mirror.xyz/0xB38709B8198d147cc9Ff9C133838a044d78B064B/nk40v2MJKSHXXNSlbqqhpwJf4MtZ9V2Vp8P_bSNwjYc)，此文章可以帮助读者理解`EVM`底层数据结构
- [Understanding Ethereum Smart Contract Storage ](https://programtheblockchain.com/posts/2018/03/09/understanding-ethereum-smart-contract-storage)，此文章可以帮助读者理解底层数据结构
当然，此合约也提供了一个不太常用的提取用户发送的`data`的方法，代码如下:
```solidity
function _msgData() internal view virtual returns (bytes calldata ret) {
    if (msg.data.length >= 20 && isTrustedForwarder(msg.sender)) {
        return msg.data[0:msg.data.length - 20];
    } else {
        return msg.data;
    }
}
```
此代码较为简单，不再解释。

你可以在[这里](https://github.com/opengsn/gsn/blob/master/packages/contracts/src/ERC2771Recipient.sol)找到`GSN`的实现。

以上内容就是标准化提取合约的内容，我们自行编写了合约逻辑部分需要继承上述函数，在此处，我编写了一个最为简单的`Box`合约，代码如下:
```solidity
contract Box is ERC2771Recipient {
    constructor(address trustedForwarder) ERC2771Recipient(trustedForwarder) {}

    uint256 private _value;

    event NewValue(uint256 newValue);
    event Sender(address sender);

    function store(uint256 newValue) public {
        _value = newValue;
        emit NewValue(newValue);
        emit Sender(_msgSender());
    }

    function retrieve() public view returns (uint256) {
        return _value;
    }
}
```

### 合约测试

在此处，我们依旧使用[上一篇](https://hugo.wongssh.cf/posts/ecsda-sign-chain/)提出的前往浏览器获得签名结果，再将签名结果手动输入测试合约的方法。

我们需要收集一些构建结构体所需要的数据:

1. `verifyingContract`、`from`和`to`都可以通过在测试合约中编写`console2.log`获得，具体代码参见[代码仓库](https://github.com/wangshouh/upgradeContractLearn/blob/master/test/metatx/metatx.t.sol#L29)
1. `data`，可以通过`cast calldata`命令获得，如`cast calldata "store(uint256)" 20`

获取签名结果的方法基本和上一篇相同，此处简单进行说明: 打开浏览器，前往[此网站](https://metamask.github.io/test-dapp/)，完成钱包链接等操作。按下`F12`打开终端`Console`。

与上次相同，首先构建结构体，如下:
```javascript
const msgParams = JSON.stringify({
    domain: {
        name: 'Forwarder',
        chainId: 4,
        version: '1',
        verifyingContract: '0xce71065d4017f316ec606fe4422e11eb2c47c246',
    },

    message: {
        from: '0x11475691c2caa465e19f99c445abb31a4a64955c',
        to: '0x185a4dc360ce69bdccee33b3784b0282f7961aea',
        value: 0,
        gas: 500000000000,
        nonce: 0,
        data: '0x6057361d0000000000000000000000000000000000000000000000000000000000000014'
    },
    primaryType: 'ForwardRequest',
    types: {
        EIP712Domain: [
            { name: 'name', type: 'string' },
            { name: 'version', type: 'string' },
            { name: 'chainId', type: 'uint256' },
            { name: 'verifyingContract', type: 'address' },
        ],

        ForwardRequest: [
            { name: 'from', type: 'address' },
            { name: 'to', type: 'address' },
            { name: 'value', type: 'uint256' },
            { name: 'gas', type: 'uint256' },
            { name: 'nonce', type: 'uint256' },
            { name: 'data', type: 'bytes' },
        ],
    },
});
```

读者应该可以理解此结构体的含义，我们在此不再赘述。

构建结构体后，我们可以直接调用`MetaMask`的签名接口，输入以下命令:
```javascript
const sign = await ethereum.request({
	method: 'eth_signTypedData_v4',
	params: ["0x11475691C2CAA465E19F99c445abB31A4a64955C", msgParams],
});
```
代码中`0x11475691C2CAA465E19F99c445abB31A4a64955C`应该替换为自己的地址。

完成`MetaMask`的交互后，在浏览器终端中输入`sign`，应该得到输出的签名结果。

为了方便进行`Nonce`的测试，我们需要将上文中给出的结构体中的`nonce`设置为`1`再进行一次签名获得签名结果以方便后文进行测试。

对于`Meta-transactions`含义的完整测试，我们不再本文继续讨论，读者可自行查阅[代码](https://github.com/wangshouh/upgradeContractLearn/blob/master/test/metatx/metatx.t.sol)。

## ERC20-Permit

`ERC20`标准成功的一个重要原因在于此标准引入了`approve`和`transferFrom`函数。前者用于用户授权`ERC20`合约使用自己所拥有的代币的权利，后者用于`ERC20`合约在授权范围内转移代币，包括`DAI`和`Uniswap`在内的大量合约使用了此函数，但此函数要求用户至少与合约进行两次交互，消耗的`gas`较多。而由`EIP2612`规定的`ERC20-Permit`完美解决了此类问题。

本节合约实现和测试相关内容大量参考了[Testing EIP-712 Signatures](https://book.getfoundry.sh/tutorials/testing-eip712)。这是`Foundry`文档中的一节，如果读者英文水平较好，可以直接参考此文章。
### 运行流程

我们首先给出不实现`EIP2612`情况下的进行交互的步骤:

1. 签署对ERC20合约的`approve(address,amount)`交易
1. 等待交易确认
1. 签署对ERC20合约中含有`transferFrom`函数的特定函数调用的交易

显然，为了调用含有`transferFrom`函数的目标函数，我们进行了两次交易，这也意味着我们需要缴纳两次`gas`费用。对于一般的用户而言，体验并不是很好。

> 当然，如果你想调用的函数中不含有`transferFrom`函数则不需要上述流程，可以直接调用。

如果合约实现`EIP2612`，步骤如下:

1. 对`Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)`结构体进行`EIP712`结构化哈希和签名
1. 向ERC20合约中的目标函数发送此签名
1. 目标函数运行并返回结果

我们在此流程内仅进行了一次交易，所以用户仅需要缴纳一笔`gas`费用，用户体验更好。当然，理论上使用合约钱包也可以实现目的，但由于合约钱包在目前以太坊生态系统中未广泛使用，而且用户体验与正常的账户钱包不同，而使用`EIP2612`则不存在此类问题。

总体而言，`EIP2612`的流程较为简单。

由于此EIP标准较为简单，所以在此处我们也给出EIP标准的具体内容。

该标准最重要的规定是哈希结构体:
```json
{
  "types": {
    "EIP712Domain": [
      {
        "name": "name",
        "type": "string"
      },
      {
        "name": "version",
        "type": "string"
      },
      {
        "name": "chainId",
        "type": "uint256"
      },
      {
        "name": "verifyingContract",
        "type": "address"
      }
    ],
    "Permit": [{
      "name": "owner",
      "type": "address"
      },
      {
        "name": "spender",
        "type": "address"
      },
      {
        "name": "value",
        "type": "uint256"
      },
      {
        "name": "nonce",
        "type": "uint256"
      },
      {
        "name": "deadline",
        "type": "uint256"
      }
    ],
    "primaryType": "Permit",
    "domain": {
      "name": erc20name,
      "version": version,
      "chainId": chainid,
      "verifyingContract": tokenAddress
  },
  "message": {
    "owner": owner,
    "spender": spender,
    "value": value,
    "nonce": nonce,
    "deadline": deadline
  }
}}
```

除了规定结构体外，标准还规定了一系列的函数，如下:
```solidity
function permit(address owner, address spender, uint value, uint deadline, uint8 v, bytes32 r, bytes32 s) external
function nonces(address owner) external view returns (uint)
function DOMAIN_SEPARATOR() external view returns (bytes32)
```

作用如下:

1. `permit`函数验证用户的发送的授权签名是否正确并修改用户的授权。其中`owner`为代币持有者而`spender`为被授权的代币使用者
1. `nonces`函数返回用户的`nonce`，在后文中我们没有进行实现
1. `DOMAIN_SEPARATOR`返回需要签名的`domain`字段

### 合约实现

为了增加文章多样性，此处我们选择完全使用`foundry`完成合约编写和测试。

此处我们选择`solmate`的`ERC20`[合约](https://github.com/transmissions11/solmate/blob/main/src/tokens/ERC20.sol)作为基准为大家解释相关代码。与`openzeppelin`提供的`ERC20`合约，`solmate`提供的合约原生支持`EIP2612`标准，而且`solmate`的合约在`gas`方面更有优势且实现更加简单。

关于`EIP2612`的[代码](https://github.com/transmissions11/solmate/blob/main/src/tokens/ERC20.sol#L116)如下:
```
function permit(
    address owner,
    address spender,
    uint256 value,
    uint256 deadline,
    uint8 v,
    bytes32 r,
    bytes32 s
) public virtual {
    require(deadline >= block.timestamp, "PERMIT_DEADLINE_EXPIRED");

    // Unchecked because the only math done is incrementing
    // the owner's nonce which cannot realistically overflow.
    unchecked {
        address recoveredAddress = ecrecover(
            keccak256(
                abi.encodePacked(
                    "\x19\x01",
                    DOMAIN_SEPARATOR(),
                    keccak256(
                        abi.encode(
                            keccak256(
                                "Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)"
                            ),
                            owner,
                            spender,
                            value,
                            nonces[owner]++,
                            deadline
                        )
                    )
                )
            ),
            v,
            r,
            s
        );

        require(recoveredAddress != address(0) && recoveredAddress == owner, "INVALID_SIGNER");

        allowance[recoveredAddress][spender] = value;
    }

    emit Approval(owner, spender, value);
}
```

此代码与我们之前使用的验证`EIP712`签名的函数基本类似，但此处为了方便从前端进行调用，将结构体拆分为了多个参数，当然此处也没有直接使用合并的签名而是使用`v r s`部分进行签名验证。此处为了使人疑惑的是`unchecked`。`unchecked`是`solidity 0.8.0`后引入的，在`0.8.0`后，`solidity`语言会自动检测计算是否导致溢出，如果溢出则抛出异常。但使用`unchecked`部分的代码不会进行溢出检测，当然删除溢出检查一方面增加了合约计算溢出的风险，另一方面减少了`gas`费用。此处，`nonces[owner]++`是唯一的计算，而且我们可以确认此数值不可能产生溢出情况，为减少`gas`使用了`unchecked`标识。

> `nonce`在每一次交易后自增，其数据类型为`uint256`，用户不能实现如此多次的交易。

在`DOMAIN_SEPARATOR`计算方面，此合约中与此相关的有以下部分:
```
constructor(
    string memory _name,
    string memory _symbol,
    uint8 _decimals
) {
    name = _name;
    symbol = _symbol;
    decimals = _decimals;

    INITIAL_CHAIN_ID = block.chainid;
    INITIAL_DOMAIN_SEPARATOR = computeDomainSeparator();
}

function DOMAIN_SEPARATOR() public view virtual returns (bytes32) {
    return block.chainid == INITIAL_CHAIN_ID ? INITIAL_DOMAIN_SEPARATOR : computeDomainSeparator();
}

function computeDomainSeparator() internal view virtual returns (bytes32) {
    return
        keccak256(
            abi.encode(
                keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
                keccak256(bytes(name)),
                keccak256("1"),
                block.chainid,
                address(this)
            )
        );
}
```

在合约初始化阶段选择当前的`chainId`进行初始化`DOMAIN_SEPARATOR`，但为了方便合约在不同链内复用，此处增加了`block.chainid == INITIAL_CHAIN_ID ? INITIAL_DOMAIN_SEPARATOR : computeDomainSeparator();`语句，利用三目表达式实现在不同的链内不同的`DOMAIN_SEPARATOR`。详细来说，此函数会首先检测目前的链是否为初始化时的链，如果是则返回初始化时已经计算好的`INITIAL_DOMAIN_SEPARATOR`，如果不是则利用当前的链ID重新计算`DOMAIN_SEPARATOR`。

我们基于此合约开发了一个极为简单的`Deposit`存款合约，该合约较为简单，代码如下:
```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {ERC20} from "./ERC20.sol";

contract Deposit {
    event TokenDeposit(address user, address tokenContract, uint256 amount);
    event TokenWithdraw(address user, address tokenContract, uint256 amount);

    mapping(address => mapping(address => uint256)) public userDeposits;

    function deposit(address _tokenContract, uint256 _amount) external {
        ERC20(_tokenContract).transferFrom(msg.sender, address(this), _amount);

        userDeposits[msg.sender][_tokenContract] += _amount;

        emit TokenDeposit(msg.sender, _tokenContract, _amount);
    }

    function depositWithPermit(
        address _tokenContract,
        uint256 _amount,
        address _owner,
        address _spender,
        uint256 _value,
        uint256 _deadline,
        uint8 _v,
        bytes32 _r,
        bytes32 _s
    ) external {
        ERC20(_tokenContract).permit(
            _owner,
            _spender,
            _value,
            _deadline,
            _v,
            _r,
            _s
        );

        ERC20(_tokenContract).transferFrom(_owner, address(this), _amount);

        userDeposits[_owner][_tokenContract] += _amount;

        emit TokenDeposit(_owner, _tokenContract, _amount);
    }
    function withdraw(address _tokenContract, uint256 _amount) external {
        require(
            _amount <= userDeposits[msg.sender][_tokenContract],
            "INVALID_WITHDRAW"
        );

        userDeposits[msg.sender][_tokenContract] -= _amount;

        ERC20(_tokenContract).transfer(msg.sender, _amount);

        emit TokenWithdraw(msg.sender, _tokenContract, _amount);
    }
}
```

上述合约并不复杂，我们通过用户操作的完整流程介绍每一个函数的作用:

1. 用户对需要进行存款的ERC20合约代币的`permit`进行签名操作，此过程在链下完成
1. 完成签名后，将签名内容输入`depositWithPermit`函数，授权合约访问你的代币并将授权的代币转入存款合约内
1. 当用户需要代币资产时使用`withdraw`函数将代币从存款合约转移到自己名下

上述流程在代币合约实现`permit`函数时可以使用。但部分代币合约没有实现此函数，则需要对代币合约调用`approve(address spender, uint256 amount)`进行手动授权，再调用存款合约内的`deposit`函数进行存款，最后可以使用`withdraw`函数提取存款。

显然，对于用户而言使用实现`EIP2612`的代币合约进行存款操作更加方便，只需要签署一笔交易就可以实现存款。

> 部分读者可能对ERC20合约的本质理解不清楚，实际上ERC20合约的核心就是`mapping(address => uint256) public balanceOf;`，此映射关系实现了对地址拥有代币的记录，我们进行的转账等操作都是对这一映射中数值的改变。

为了方便后文的合约测试，我们在此处实现一个用于`EIP712`结构化哈希的合约，代码如下:
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

contract SigUtils {
    bytes32 internal DOMAIN_SEPARATOR;

    constructor(bytes32 _DOMAIN_SEPARATOR) {
        DOMAIN_SEPARATOR = _DOMAIN_SEPARATOR;
    }

    // keccak256("Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)");
    bytes32 public constant PERMIT_TYPEHASH =
        0x6e71edae12b1b97f4d1f60370fef10105fa2faae0126114a169c64845d6126c9;

    struct Permit {
        address owner;
        address spender;
        uint256 value;
        uint256 nonce;
        uint256 deadline;
    }

    // computes the hash of a permit
    function getStructHash(Permit memory _permit)
        internal
        pure
        returns (bytes32)
    {
        return
            keccak256(
                abi.encode(
                    PERMIT_TYPEHASH,
                    _permit.owner,
                    _permit.spender,
                    _permit.value,
                    _permit.nonce,
                    _permit.deadline
                )
            );
    }

    // computes the hash of the fully encoded EIP-712 message for the domain, which can be used to recover the signer
    function getTypedDataHash(Permit memory _permit)
        public
        view
        returns (bytes32)
    {
        return
            keccak256(
                abi.encodePacked(
                    "\x19\x01",
                    DOMAIN_SEPARATOR,
                    getStructHash(_permit)
                )
            );
    }
}
```

上述合约较为简单，我们已经实现过多次类似的合约，此处不再赘述。

### 合约测试

本部分介绍的代码来自`Testing EIP-712 Signatures`的[代码仓库](https://github.com/kulkarohan/deposit/tree/main/test)。

除了上文给出的`SigUtils.sol`文件，我们也创建了`MockERC20`合约用于测试(位于`test/EIP2612/utils`)，此合约代码如下:
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {ERC20} from "../../../src/EIP-2612/ERC20.sol";

contract MockERC20 is ERC20 {
    constructor() ERC20("Mock Token", "MOCK", 18) {}

    function mint(address _to, uint256 _amount) public {
        _mint(_to, _amount);
    }
}
```
此合约仅仅实现了`mint`函数以方便后文进行测试。

在[Testing EIP-712 Signatures](https://book.getfoundry.sh/tutorials/testing-eip712)中详细解释了此项目测试的所有流程，在本文中我们仅仅进行简单的解释。

初始化阶段的代码为:
```solidity
MockERC20 internal token;
SigUtils internal sigUtils;

uint256 internal ownerPrivateKey;
uint256 internal spenderPrivateKey;

address internal owner;
address internal spender;

function setUp() public {
    token = new MockERC20();
    sigUtils = new SigUtils(token.DOMAIN_SEPARATOR());

    ownerPrivateKey = 0xA11CE;
    spenderPrivateKey = 0xB0B;

    owner = vm.addr(ownerPrivateKey);
    spender = vm.addr(spenderPrivateKey);

    token.mint(owner, 1e18);
}
```
主要完成了各行业的初始化和用户的初始化。在用户初始化中，使用了`vm.addr()`函数，此函数会返回指定私钥的地址。此处也使用`mint`函数为`owenr`铸造了代币。

> 通过私钥计算地址的方法我们已在[上一篇文章](https://hugo.wongssh.cf/posts/ecsda-sign-chain/)中给出

完成初始化后，我们首先实现对核心功能`Permit`的测试，具体代码如下:

```solidity
function test_Permit() public {
    SigUtils.Permit memory permit = SigUtils.Permit({
        owner: owner,
        spender: spender,
        value: 1e18,
        nonce: 0,
        deadline: 1 days
    });

    bytes32 digest = sigUtils.getTypedDataHash(permit);

    (uint8 v, bytes32 r, bytes32 s) = vm.sign(ownerPrivateKey, digest);

    token.permit(
        permit.owner,
        permit.spender,
        permit.value,
        permit.deadline,
        v,
        r,
        s
    );

    assertEq(token.allowance(owner, spender), 1e18);
    assertEq(token.nonces(owner), 1);
}
```
此处的`vm.sign()`函数可以实现使用私钥对哈希摘要进行签名。

完成核心功能的测试后，我们需要确认各个限制条件是否都能实现，主要的限制条件为:

- deadline
- signer
- nonce

对于`deadline`的测试核心使用了`vm.warp()`，此函数可以实现改变区块时间。对于其他限制因素的测试，我们在前文基本进行过描述，读者可以自行阅读源代码。

> 在源代码中使用了`vm.expectRevert`函数，此函数会判断下一次调用抛出的异常是否符合预期。如果不符合，则测试失败

最后，我们测试`transferFrom`函数，基本思路与上文类似，先测试函数是否可以正常运行，再测试各个限制条件是否可以发挥作用。在此处，我们主要使用了`vm.prank`用于切换测试环境中的`msg.sender`的地址。

我们也需要对`deposit`合约进行测试，基本思路类似。请读者自行阅读源代码进行理解。

## 总结

本文主要介绍了`EIP712`的拓展使用方法，涉及以下EIP标准:

- EIP2770
- EIP2771
- EIP2612

其中，前两者解决了合约交互方在没有ETH的情况下与行业交互的问题，后者解决了在`ERC20`合约内通过结构化哈希签名通过一次交易完成授权操作，优化了用户体验。

前者的优秀实践可以参考`GSN`项目，而后者在大量`ERC20`合约内都有所实现，而且在`solmate`项目实现的`ERC20`合约被默认实现。
