---
title: "深入 ENS 系统"
date: 2023-06-26T14:45:33Z
tags: [ENS,solidity]
---

## 概述

ENS 是一个以太坊上的开放、可拓展的命名系统，也是目前在以太坊中最常用的命名系统。ENS 的功能是将人类较难判读的以太坊地址转化为人类可读的名字，如 `vitalik.eth` 。 

本博客内容较为易懂，但要求读者具有一定的 solidity 编程水平，如果您能看懂任意实现的 ERC20 合约源代码，即可以读懂本文。如果您不具有此前置知识，建议阅读 [Foundry教程：编写测试部署ERC-20代币智能合约](https://blog.wssh.trade/posts/foundry-with-erc20/)。

本文包含一个 Dune SQL 查询，但此 SQL 查询与正文内容关系不大，如果您不了解 SQL 语言，可以直接跳过。

## ENS 解析

ENS 解析的基本架构如下，我们会在后文详细讨论每一部分的作用以及代码实现:

![ENS Architecture](https://img.gopic.xyz/us4yb9.webp)

`Registry` 是 ENS 系统核心合约，也是进行 ENS 查询的起点。此合约中记录了以下信息:

1. 域名的持有人 `owner`
2. 域名的解析器 `resolver`
3. 解析记录的缓存时间 `TTL`

域名的所有者可以将将域名的修改权限授予 `operator`。

具体来看，此部分对应如下代码:

```solidity
struct Record {
    address owner;
    address resolver;
    uint64 ttl;
}

mapping(bytes32 => Record) records;
mapping(address => mapping(address => bool)) operators;
```

`Resolvers` 是真正执行域名解析的合约，任何符合 ENS 标准的智能合约都可以作为域名的解析器，这意味着用户可以自行增加一些域名的可解析内容。目前来说，`Resolvers` 支持以太坊地址、EVM兼容链地址、内容哈希值等解析。关于这部分更加详细的定义，可以参考 [ENSIP](https://docs.ens.domains/ens-improvement-proposals) 中的相关内容以及 [Resolver.sol](
https://github.com/ensdomains/ens-contracts/blob/dev/contracts/resolvers/Resolver.sol)合约。

> 一般来说，`Resolvers` 会实现 [ENSIP](https://docs.ens.domains/ens-improvement-proposals) 中规定所有解析内容

有了上述知识，我们就可以研究 ENS 域名解析的具体流程，流程如下:

![ENS flow](https://img.gopic.xyz/cee6e6dfe232afa72530a556491c8745.png)

当用户需要查询域名的某个域名对应的解析内容时，首先需要调用 `Registry` 的 `resolver` 函数获得域名对应的解析器，该函数代码如下:

```solidity
/**
 * @dev Returns the address of the resolver for the specified node.
 * @param node The specified node.
 * @return address of the resolver.
 */
function resolver(
    bytes32 node
) public view virtual override returns (address) {
    return records[node].resolver;
}
```

`node` 是域名在 `Registry` 合约中真正的记录方法，该变量实际上对应域名的 `namehash` 。`namehash` 可以使用以下方法进行计算:

1. 名称规范化。读者可能注意到 ENS 允许使用非 ascii 字符进行注册，但这些非 ascii 字符是极难处理的，ENS 使用了 [UTS46](http://unicode.org/reports/tr46/) 编码使所有域名都可以使用 ascii 进行表示。关于 UTS46 具体的编码规则，此处不进行深入讨论。但读者应当知道该编码方式对大小写不敏感，即 `FOO.eth` 和 `foo.eth` 在 ENS 系统内是等价的。
2. 规范命名哈希。
    对经过规范化后的域名，我们需要进行哈希以获得最终的 `namehash`。`namehash` 是一个较为奇怪的哈希方法，使用这种 hash 方法可以表现域名的层级关系。比如 `wallet.eth` 与 `vitalik.wallet.eth` 存在层级关系，前者是后者的根域，这种层级关系也会表现在 `namehash` 中，这使得根域管理者可以对子域进行管理，比如转域子域所有权。`Registry` 合约实现了该方法，实现代码如下:

    ```solidity
    function setSubnodeOwner(
        bytes32 node,
        bytes32 label,
        address owner
    ) public virtual override authorised(node) returns (bytes32) {
        bytes32 subnode = keccak256(abi.encodePacked(node, label));
        _setOwner(subnode, owner);
        emit NewOwner(node, label, owner);
        return subnode;
    }
    ```

    此处的 `label` 即子域名字的 `keccak` 哈希值 ，以 `vitalik.wallet.eth` 为例，此处的 `label` 为 `vitalik` 的 `keccak` 哈希值，而 `node` 是 `wallet.eth` 的 `namehash` 。

    而计算 `vitalik.wallet.eth` 的流程可以使用下图表示:

    ![vitalik namehash](https://img.gopic.xyz/8cb836b40d279cb82a82f5ad406ab080.png)

    上述流程图中的 `||` 表示拼接，在 `solidity` 中，我们一般使用 `abi.encodePacked` 实现。一般来说，我们称最外层的 `vitalik` 为 `label hash`，即 `setSubnodeOwner` 中的 `label` 参数。

    此流程可以使用以下 Python 代码表示:

    ```python
    from Crypto.Hash import keccak


    def namehash(name: str):
        label_list = name.split(".")
        label_list.reverse()

        node = "0" * 64
        for name_label in label_list:
            keccak_hash = keccak.new(digest_bits=256)

            label_hash = keccak_hash.update(name_label.encode("utf-8")).hexdigest()

            node += label_hash
            keccak_hash = keccak.new(digest_bits=256)
            node = keccak_hash.update(bytes.fromhex(node)).hexdigest()

        return node


    if __name__ == "__main__":
        print(namehash("foo.eth"))
    ```

    上述代码运行需要 [pycryptodome](https://pycryptodome.readthedocs.io/en/latest/src/hash/keccak.html) 库的支持，您可以将 `keccak_hash.update` 换成任意的实现 `keccak` 哈希算法的函数。注意上述代码使用了 `utf-8` 对域名进行编码，没有实现 `UTS46` 规范化过程，上述代码不建议在任何生产环境使用。您可以使用 [ENS Libraries](https://docs.ens.domains/dapp-developer-guide/ens-libraries) 中的任意 SDK 内的 `namehash` 计算函数

    虽然官方文档内指出 `namehash` 的计算是一个递归过程，但为了方便读者理解，上述示例代码将其改写为递推过程。

部分读者可能好奇，引入 `namehash` 这种计算方法除了便于在智能合约内部管理和表示域名外，还有什么其他作用？正如上文所述，`namehash` 的另一大作用是标志了根域所有者对其根域衍生子域的所有权，该功能主要通过 `Registry` 合约中的 `setSubnodeOwner` 函数体现。

> 限于篇幅，此处我们不会进一步深入讨论 `namehash` 的相关内容，关于 `namehash` 的详细介绍可以参考 [ERC-137](https://eips.ethereum.org/EIPS/eip-137) 和 [Name Processing](https://docs.ens.domains/contract-api-reference/name-processing) 文档。后者给出使用 `thegraph` 检索某一域名的 `namehash` 的流程。

> 如果读者安装有 `foundry` 智能合约开发框架，可以使用 `cast namehash foo.eth` 命令获得 `foo.eth` 的 namehash

接下来，我们继续讨论 ENS 域名的解析流程，当应用通过 `Registry` 拿到域名的 `Resolvers` 后，应用程序会调用 `Resolvers` 合约的 `addr` 函数，我们以最简单的以太坊地址解析为例介绍该函数:

```solidity
uint256 private constant COIN_TYPE_ETH = 60;

mapping(uint64 => mapping(bytes32 => mapping(uint256 => bytes))) versionable_addresses;

function addr(
    bytes32 node
) public view virtual override returns (address payable) {
    bytes memory a = addr(node, COIN_TYPE_ETH);
    if (a.length == 0) {
        return payable(0);
    }
    return bytesToAddress(a);
}

function addr(
    bytes32 node,
    uint256 coinType
) public view virtual override returns (bytes memory) {
    return versionable_addresses[recordVersions[node]][node][coinType];
}
```

上述代码展示了 `addr(bytes32 node)` 函数的实现，该函数的作用是用来查询某个 ENS 对应的以太坊地址，这应该是 ENS 使用最为频繁的场景。该函数的实现依赖于多链地址查询函数 `addr(bytes32 node, uint256 coinType)` 函数，此函数支持根据不同的 `coinType` 查询不同链上的用户地址。比如 ETH 链的 `coinType` 为 `60`，其他区块链的 `coinType` 可参考 [SLIP-0044](https://github.com/satoshilabs/slips/blob/master/slip-0044.md) 中的规定，关于存储地址的编码格式等情况，可以参考 [ENSIP-9](https://docs.ens.domains/ens-improvement-proposals/ensip-9-multichain-address-resolution) 中的规定。

此处还有一个较为奇怪的映射 `recordVersions`，该映射维护 ENS 解析表的版本.用户可以通过调用 `clearRecords` 函数增加版本号。一旦版本号增加，就是导致过去所有解析无效。相关代码如下:

```solidity
mapping(bytes32 => uint64) public recordVersions;

function clearRecords(bytes32 node) public virtual authorised(node) {
    recordVersions[node]++;
    emit VersionChanged(node, recordVersions[node]);
}
```

上文给出的复杂映射 `versionable_addresses` ，具体的定义如下:

```
verison -> namehash -> Coin Type -> address
```

继续回到主线域名解析，当用户向 `Resolvers` 调用 `addr` 函数后，`Resolvers` 合约会返回指定 ENS 对应的以太坊地址，完成解析的全流程。

## ENS 注册

有读者可能发现，在上文中至始至终都没出现所谓的“ ENS 是一个 NFT ”的内容。从技术上来说，ENS 的本质是 `namehash` 而 ENS 的 NFT 形式只是一个表象。理论上，ENS 系统可以摆脱当前 NFT 的叙事，但可能因为各种商业上的原因，ENS 还是继续走在了 NFT 的道路上，甚至进一步深入了与 NFT 的绑定，推出了 `ENS Name Wrapper` 功能，关于 `ENS Name Wrapper` 的详细内容，我们会在后文进行讨论。

ENS 的注册是由 `.eth Registrar` 合约完成的，该合约具有 `Registry` 的写入权限，当用户购买 ENS NFT 后，`.eth Registrar` 合约就会向 `Registry` 进行写入。

我们可以使用以下图片表示两者的关系:

![ENS registrar](https://img.gopic.xyz/6cd8992a1a19c09f9f2f963e33f6ab4a.jpg)

该图展示了 `.eth Registrar` 和 `Registry` 之间的关系。我们可以发现当前的 `.eth Registrar` 和 `Registry` 其实关系不大。对于一般用户而言，NFT 的所有者和 `Registry` 中记录的域名的所有者基本都是一个人，但两者实际上是可以分离的，出现域名的实际控制人和 ENS NFT 的所有者不是同一个的情况，而且此处读者也可以发现 ENS NFT 只会对形如 `name.eth` 的用户，而 `name.eth` 的子域，如 `sub1.name.eth` 则不持有 ENS NFT。我们会在后文讨论该问题的解决方案，即 `ENS Name Wrapper`

在上图中，我们可以观察到 `.eth Registrar` 存储了 `Labelhash(x)` ，此值即为 ENS NFT 对应的 `token id`。在上文中，我们已经介绍了 `label hash` 的定义与计算。以 `vitalik.eth` 为例，假如我们希望知道其 NFT 对应的 token id，只需要对 `vitalik` 进行 `keccak` 哈希，可以获得结果为 `0xaf2caa1c2ca1d027f1ac823b529d0a67cd144264b2789fa2ea4d63a67c7103cc`。

> ENS NFT 的 metadata URL 为 `https://metadata.ens.domains/mainnet/0x57f1887a8bf19b14fc0df6fd9b2acc9af147ea85/`，我们可以通过访问 [此链接](https://metadata.ens.domains/mainnet/0x57f1887a8bf19b14fc0df6fd9b2acc9af147ea85/0xaf2caa1c2ca1d027f1ac823b529d0a67cd144264b2789fa2ea4d63a67c7103cc) 获得 `vitalik.eth` 的 metadata。

接下来，我们深入了解 ENS 合约的内容，该合约部署地址为 `0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85` ，使用的合约为 `BaseRegistrarImplementation.sol`，您可以点击 [此处](https://github.com/ensdomains/ens-contracts/blob/dev/contracts/ethregistrar/BaseRegistrarImplementation.sol) 访问其源代码。

> 如何查找 ENS 系统内的各个合约部署地址？读者可以通过访问 [ens.eth subnames](https://app.ens.domains/ens.eth?tab=subnames) 获取，此处展示了 ENS 系统内所有部署的合约及其域名。

我们首先介绍用于 ENS 注册的 `register` 函数，该函数实现如下:

```solidity
function register(
    uint256 id,
    address owner,
    uint256 duration
) external override returns (uint256) {
    return _register(id, owner, duration, true);
}

function _register(
    uint256 id,
    address owner,
    uint256 duration,
    bool updateRegistry
) internal live onlyController returns (uint256) {
    require(available(id));
    require(
        block.timestamp + duration + GRACE_PERIOD >
            block.timestamp + GRACE_PERIOD
    ); // Prevent future overflow

    expiries[id] = block.timestamp + duration;
    if (_exists(id)) {
        // Name was previously owned, and expired
        _burn(id);
    }
    _mint(owner, id);
    if (updateRegistry) {
        ens.setSubnodeOwner(baseNode, bytes32(id), owner);
    }

    emit NameRegistered(id, owner, block.timestamp + duration);

    return block.timestamp + duration;
}
```

我们可以看到该函数使用了 `live` 和 `onlyController` 修饰器，其中 `live` 修饰器保证了当前合约为 ENS Registry 的管理者，而 `onlyController` 保证该函数的调用者有权限，相关代码如下:

```solidity
modifier live() {
    require(ens.owner(baseNode) == address(this));
    _;
}

modifier onlyController() {
    require(controllers[msg.sender]);
    _;
}
```

继续阅读代码，我们发现代码进行了以下逻辑:

1. 使用 `require(available(id))` 检验 `id` 的可用性，要求 `expiries[id] + GRACE_PERIOD < block.timestamp`，即当前 id 已过期且不处于宽限期(GRACE_PERIOD，当前设定为 90 日)内。
2. 使用 `require(block.timestamp + duration + GRACE_PERIOD >  block.timestamp + GRACE_PERIOD)` 保证在时间溢出的情况下中止合约运行
3. 使用 `expiries[id] = block.timestamp + duration;` 为注册用户设置 ENS 域名的过期期限
4. 在 `_exists(id)` 即当前 ENS NFT 存在的情况下，调用 `burn` 函数烧毁原 NFT
5. 使用 `_mint(owner, id);` 为用户铸造 NFT
6. 更新 ENS Registry 中的记录赋予用户修改 ENS 域名解析的权力
7. 释放事件并返回 ENS 到期时间

该合约较为简单，我们不再继续分析其他函数，读者可以自行研究用于续费的 `renew` 函数和用于申请ENS 域名解析的权力的 `reclaim` 函数。ENS NFT 的基础实现极为简单的，并没有实现一些复杂的逻辑。但读者可以发现核心的注册和续费函数并不允许一般的用户直接调用，而只允许 `Controller` 调用，这是否意味着 ENS 系统是相对不去中心化的？

为了解决这一个问题，我们需要查询出当前运作的所有 Controller 地址，这是一个难度一般的工作，我们此处使用了 [Dune](https://dune.com/home) 作为数据平台，使用以下查询即可获得当前在运行的 Controller 地址:

```sql
with
  controller_add as (
    SELECT
      tx_hash,
      topic1
    FROM
      ethereum.logs
    WHERE
      contract_address = 0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85
      AND topic0 = 0x0a8bb31534c0ed46f380cb867bd5c803a189ced9a764e30b3a4991a9901d7474
  ),
  controller_remove as (
    SELECT
      tx_hash,
      topic1
    FROM
      ethereum.logs
    WHERE
      contract_address = 0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85
      AND topic0 = 0x33d83959be2573f5453b12eb9d43b3499bc57d96bd2f067ba44803c859e81113
  )
SELECT
  ca.topic1
FROM
  controller_add ca
  LEFT JOIN controller_remove cr ON ca.topic1 = cr.topic1
WHERE
  cr.topic1 IS NULL
```

由于 ENS 系统没有已经解析完成的数据，所以此处我们直接使用了 `ethereum.logs` 数据集，其中 `topic0` 的对应的事件如下:

| topic0                                                             | event name                 |
|--------------------------------------------------------------------|----------------------------|
| 0x0a8bb31534c0ed46f380cb867bd5c803a189ced9a764e30b3a4991a9901d7474 | ControllerAdded(address)   |
| 0x33d83959be2573f5453b12eb9d43b3499bc57d96bd2f067ba44803c859e81113 | ControllerRemoved(address) |

在安装 `foundry` 的情况下，可以使用以下命令获得此表格:

```bash
cast sig-event "ControllerAdded(address)"
cast sig-event "ControllerRemoved(address)"
```

如果您可以理解上述 SQL 代码的含义，但仍无法理解其工作原理，请参考我的另一篇博客 [Clickhouse 以太坊分析:交易日志分析](https://blog.wssh.trade/posts/clickhouse-eth-logs/) 内的有关内容。

上述 [查询](https://dune.com/queries/2666918) 可以在此处访问。结果如下:

- [0x60c7c2a24b5e86c38639fd1586917a8fef66a56d](https://etherscan.io/address/0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85)，用于展示迁移信息的合约，没有实际作用
- [0x283af0b28c62c092c9727f1ee09c02ca627eb7f5](https://etherscan.io/address/0x283af0b28c62c092c9727f1ee09c02ca627eb7f5),注册器，目前仍可以使用，[源代码](https://github.com/ensdomains/ethregistrar/blob/master/contracts/ETHRegistrarController.sol)公开。该注册器的原理可以参考 [ENS 文档](https://docs.ens.domains/contract-api-reference/.eth-permanent-registrar/controller)。
- [0xd4416b13d2b3a9abae7acd5d6c2bbdbe25686401](https://etherscan.io/address/0xd4416b13d2b3a9abae7acd5d6c2bbdbe25686401), ENS Name Wrapper
- [0xcf60916b6cb4753f58533808fa610fcbd4098ec0](https://etherscan.io/address/0xcf60916b6cb4753f58533808fa610fcbd4098ec0), Safe 多签钱包

## ENS Name Wrapper

正如上文所说，在当前的 ENS 系统内，子域名无法获得 ENS NFT，而且存在一个较为严重的问题，即根域所有者不能对其子域用户进行一系列的权限管理，这对于某些组织而言是极其麻烦的。

为了解决这些问题，ENS 引入了 ENS Name Wrapper，该系统如下图所示:

![ENS Name Wrapper](https://img.gopic.xyz/93cd0014f42c2761af3bf82dc2788f21.jpg)

我们可以看到包装一个域名需要将域名在 `Registry` 中的控制人设置为 ENS Name Wrapper 合约。在这种设置下，用户不能直接与 `Registry` 交互而只能通过 ENS Name Wrapper 合约，这也保证了用户在 ENS Name Wrapper 合约中设置的权限是有效且不会被绕过的。在下文中，我们简称 `ENS Name Wrapper` 为 `ENW` 以方便叙述。

包装过程在最底层使用以下函数:

```solidity
function wrapETH2LD(
    string calldata label,
    address wrappedOwner,
    uint16 ownerControlledFuses,
    address resolver
) public returns (uint64 expiry) {
    uint256 tokenId = uint256(keccak256(bytes(label)));
    address registrant = registrar.ownerOf(tokenId);
    if (
        registrant != msg.sender &&
        !registrar.isApprovedForAll(registrant, msg.sender)
    ) {
        revert Unauthorised(
            _makeNode(ETH_NODE, bytes32(tokenId)),
            msg.sender
        );
    }

    // transfer the token from the user to this contract
    registrar.transferFrom(registrant, address(this), tokenId);

    // transfer the ens record back to the new owner (this contract)
    registrar.reclaim(tokenId, address(this));

    expiry = uint64(registrar.nameExpires(tokenId)) + GRACE_PERIOD;

    _wrapETH2LD(
        label,
        wrappedOwner,
        ownerControlledFuses,
        expiry,
        resolver
    );
}
```

与传统的 ENS 合约不同，ENW 合约接受 `string` 类型的参数，这大大方便了用户的调用，但也带来了 Gas 消耗的上升，而 `wrappedOwner` 则指定了最终 ERC1155 NFT 的所有者，`ownerControlledFuses` 参数涉及到 `fuse` 机制，我们会在后文讨论。具体实现较为简单，将用户的 ENS NFT 转移到当前包装器名下且使用 `reclaim` 函数重置 `Registry` 中域名的控制人为包装器合约。

限于篇幅，我们不再讨论 `_wrapETH2LD` 的具体实现，该函数的一大作用是铸造 ERC1155 NFT，读者可以自行研究。除 `wrapETH2LD` 函数外，还存在 `wrap` 处理非 `.eth` 域名，如 `.xyz` 等。

接下来，我们主要讨论 fuse 机制。`fuse` 的中文名为保险丝，具体来说，`fuse` 代表一种权限许可，当我们烧毁该保险丝就意味着在一定期限(该期限被称为 `Expiry`)内，该权限被撤销。如当我们烧毁某个 ERC 1155 域名的 `CANNOT_TRANSFER` 保险丝后，当前的 ERC 1155 NFT 则无法进行转移操作。

关于保险丝有效时间是一个较为复杂的问题，我们会在后文具体讨论。当到达有效期后，所有的 `fuse` 都会被重置。`fuse` 在智能合约内表现为 `uint32` 类型。每个 `fuse` 占据 1 bit 位置，其中 1-16 位可以被域名所有者烧毁。具体包含以下内容:

 name               | bit | function    
--------------------|-----|-------------
 CANNOT_UNWRAP      | 1   | 无法脱离包装状态    
 CANNOT_BURN_FUSES  | 2   | 无法烧毁其他 fuse 
 CANNOT_TRANSFER    | 4   | 无法转移当前域名 NFT
 CANNOT_SET_RESOLVER| 8   | 无法设置 `Resolver`
 CANNOT_SET_TTL     | 16  | 无法设置 `TTL`
 CANNOT_CREATE_SUBDOMAIN | 32 | 无法创建子域
 CANNOT_APPROVE     | 64 | 无法进行 NFT 授权操作

而 19-32 位部分仅能被根域所有者烧毁。具体包含以下内容:

 name               | bit | function    
--------------------|-----|-------------
 PARENT_CANNOT_CONTROL | 65536   | 放弃对子域的所有权
 IS_DOT_ETH         | 131072 | 无法被用户设置，烧毁后表示该域名为 `.eth` 域名
 CAN_EXTEND_EXPIRY  | 262144 | 子域名可以自行延长到期时间

在上述 fuse 基础上，ENW 系统对其中 `PARENT_CANNOT_CONTROL`、`CANNOT_UNWRAP` 和 `CANNOT_BURN_FUSES` 有着以下特殊规定:

- 在根域名未烧毁 `CANNOT_UNWRAP` 情况下，不能烧毁子域名 `PARENT_CANNOT_CONTROL`，参考 `_checkParentFuses` 函数
- 在 `PARENT_CANNOT_CONTROL` 被烧毁前，不能烧毁 `CANNOT_UNWRAP`
- 在未烧毁 `CANNOT_UNWRAP` 前，域名所有者不能烧毁其他用户所有者可以控制的 `fuse`
- 在未烧毁 `PARENT_CANNOT_CONTROL` 前，域名所有者不能烧毁根域可以控制的 `fuse`

基于上述 `fuse` ，我们可以通过烧毁不同的 `fuse` 进入以下状态:

1. `Unregistered` 未被注册
2. `Unwrapped` 未包装
3. `Wrapped` 域名已被 ENW 包装，但在此状态下，根域所有者仍保持对当前域名的权限管理，可以进行烧毁 `fuse` 操作，甚至修改子域的所有人
4. `Emancipated` 域名已被 ENW 包装，且根域所有者不能烧毁当前域名的 `fuse`。所有 `.eth` 的二级域名注册后自动成为此状态
5. `Locked` 域名无法脱离包装状态，且不受域所有者和根域所有者的控制，注意处于 `Locked` 状态的域名可以控制其子域名

各状态的关系可以参考下图:

![ENW State](https://files.catbox.moe/8s99tx.svg)

我们以一个简单的示例展示 `name.eth` 的所有者如何通过烧毁 `sub.name.eth` 使其进入不同状态。

由于 `name.eth` 是 `sub.name.eth` 的根域所有者，在 `name.eth` 烧毁 `PARENT_CANNOT_CONTROL` 前，`sub.name.eth` 无法手动烧毁任何 fuse 。以下叙述假设 `name.eth` 已烧毁 `PARENT_CANNOT_CONTROL`(包装时自动烧毁) 和 `CANNOT_UNWRAP`(所有者手动烧毁)，另一方面，我们假设 `sub.name.eth` 已被铸造，处于 `Wrapped` 状态。

此时，`name.eth` 的所有者可以完全控制 `sub.name.eth` ，可以使用 `setChildFuses` 函数烧毁 `sub.name.eth` 任意的 `fuse`

假如 `name.eth` 烧毁 `sub.name.eth` 的 `CAN_EXTEND_EXPIRY` 即意味着子域所有者可以自己延长子域的有效期。当 `name.eth` 烧毁子域的 `PARENT_CANNOT_CONTROL` 后，子域便从 `Wrapped` 转变为了 `Emancipated` 状态，此时子域所有者可以自己烧毁剩余的 fuse 进行权限控制，子域所有者也可以进行自己子域的创建(前提是 `CANNOT_CREATE_SUBDOMAIN` 未被根域所有者烧毁)。但由于此时子域尚未烧毁 `CANNOT_UNWRAP` ，`sub.name.eth` 的所有者无法对自己的子域进行管理。

当 `sub.name.eth` 烧毁 `CANNOT_UNWRAP` 后，其进入 `Locked` 状态，此时可以对自己的子域名进行管理。

上文较为简单的介绍了 ENS Name Wrapper 系统，如果读者希望获得更多关于此部分的信息，请阅读以下文章:

- [NameWrapper docs](https://github.com/ensdomains/ens-contracts/tree/dev/contracts/wrapper)
- [A Deep Dive into the ENS Name Wrapper](https://ens.mirror.xyz/0M0fgqa6zw8M327TJk9VmGY__eorvLAKwUwrHEhc1MI)