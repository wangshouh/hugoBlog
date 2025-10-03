---
title: "基于Python与GraphQL的链上数据分析实战"
date: 2022-12-06T19:47:33Z
tags: [Python,GraphQL,Data]
aliases: ["/2022/12/04/on-chain-data-basement/"]
---

## 概述

本文主要介绍如何通过免费且高效的的基于GraphQL的[basement](https://basement.dev/)进行部分链上数据分析实战。本文不要求读者具有`GraphQL`相关经验，但要求读者会使用`Python`中的`Pandas`库，这是本文主要使用的数据分析工具。换言之，本文适用于了解数据分析而不了解链上数据获取的读者。本文会涵盖以下内容:

1. `GraphQL`检索数据基础入门
1. `Basement`的基础API实战

在阅读本文前，读者最好安装一个支持`GraphQL`请求方法的API调试工具，在此处，我个人使用的是[Postman](https://www.postman.com/)软件，但读者选择其他软件亦可。本文使用了新兴 Web3 链上数据API提供商[basement](https://basement.dev/)，此处我们使用的是免费版，无需 API Key 等配置，具体限制参考下图:

![Basement Price](https://img.gopic.xyz/509e0ded2829287c1754f84fb83bb787.png)

关于`Basement`的优势可参考[Mirror 文章](https://mirror.xyz/0x25B2B8458BAB283d465996df38305333C75982B6/uYsldHeef7FxVcBI233QSYzje4ejiQu0SMVdY74vf1s)。

## 第一个请求

在进行第一个请求前，我们需要了解关于`GraphQL`最基础的一些概念，首先`GraphQL`实质上是一种类似`SQL`的数据检索语言，当然其学习难度低于`SQL`。当我们进行一次`GraphQL`请求时，我们将`GraphQL`语言编写的检索方法放在`POST`的`body`中，直接发送`POST`请求到终端API节点即可。

我们给出一个来自[GraphQL 官网](https://graphql.org/learn/queries/)的检索示例，索引请求如下:
```graphl
{
  hero {
    name
  }
}
```
返回如下:
```json
{
  "data": {
    "hero": {
      "name": "R2-D2"
    }
  }
}
```
由此可见，`GraphQL`的检索是简单且易读的，基本遵从以下规则:
```
{
    所需要的对象 {
        对象属性
    }
}
```
值得注意的是在现实世界中存在大量对象嵌套的情况，比如以下数据:
```json
{
    "data": {
        "address": {
            "address": "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
            "profile": {
                "name": "vitalik.eth"
            }
        }
    }
}
```
可抽象化为以下情况:
```
{
    data对象 {
        address 对象 {
            address 属性
            profile 对象 {
                name 属性
            }
        }
    }
}
```
关于如何获得这些信息，一个方法是查询文档，本次实战使用的`Basement`在[它的文档](https://docs.basement.dev/schema/objects)中给出了这些信息，如下图:
![Basement Object Doc](https://img.gopic.xyz/28efc32c6927da42a6bfd9758a033d15.png)

我们可以通过点击其中的蓝色链接确定每个`Object`中的属性是否嵌套了另一个`Object`。

显然，在以太坊区块链上遍历获得`address`数据是不显示的，我们在此处需要引入一种方法筛选我们所需要的`address`，这就是参数机制，读者可在[此处](https://graphql.org/learn/queries/#arguments)找到相关文档。在`Basement`中大量的对象必须与参数一同使用。在`Basement`文档中参数被表示为`Arguments`。

最后我们介绍一个执行检索的入口，即`Query`对象，这些对象用于作为检索的入口与`query`关键词配合。这些作为检索入口的`Query`对象类型列表可以在[此处文档](https://docs.basement.dev/schema/queries)找到。

有了上述简单的基础学习，我们就可以开始构建我们第一个`GraphQL`索引，如下:
```graphql
query Test {
    address(address: "vitalik.eth") { 
        address
        profile { 
            name 
            avatar
        } 
        tokens(limit: 3) {
            contract
            name
            tokenId
        }
    }
}
```

此处我们选择了入口检索对象为`address`，此对象的参数仅有 1 个，即为`address`，`address`参数必须为字符串类型，可以为正常的 16 进制编码的以太坊地址，也可以是ENS地址。我任选了几个属于`address`的属性进行检索，完整的属性列表可以在[此处](https://docs.basement.dev/schema/objects#address)查询。

> `query`后的`Test`是为本次查询所取的名字，并不重要，读者甚至可以删去此字段，直接使用`query`关键词

请求的API的URL为`https://beta.basement.dev/v2/graphql`，读者使用了`Postman`进行请求的截图如下:

![Graphql Postman](https://img.gopic.xyz/9a41b288866997fed1a04afc5e2364c3.png)

返回的结果如下:
```json
{
    "data": {
        "address": {
            "address": "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
            "profile": {
                "avatar": "eip155:1/erc1155:0xb32979486938aa9694bfc898f35dbed459f44424/10063",
                "name": "vitalik.eth"
            },
            "tokens": [
                {
                    "contract": "0x000386e3f7559d9b6a2f5c46b4ad1a9587d59dc3",
                    "name": "BoredApeNikeClub #1",
                    "tokenId": "0x01"
                },
                {
                    "contract": "0x000386e3f7559d9b6a2f5c46b4ad1a9587d59dc3",
                    "name": null,
                    "tokenId": "0x0160"
                },
                {
                    "contract": "0x000386e3f7559d9b6a2f5c46b4ad1a9587d59dc3",
                    "name": null,
                    "tokenId": "0x01bd"
                }
            ]
        }
    }
}
```

至此我们完成了第一个请求。

在阅读完上述内容后，我们进行第一个实战，探索 V神 拥有NFT的数量和种类，我们首先进行数据索引:
```python
import pandas as pd

from gql import gql, Client
from gql.transport.requests import RequestsHTTPTransport

transport = RequestsHTTPTransport(
    url="https://beta.basement.dev/v2/graphql"
)
client = Client(transport=transport,fetch_schema_from_transport=True)
query = gql(
    """
    query vitalikNFT{
      address(address: "vitalik.eth") { 
          tokens(limit: 100000) {
              contract
              name
              tokenId
          }
      }
  }
"""
)

result = client.execute(query)
```
注意我们改变了`tokens`的`limit`参数，因为此参数默认为 50，但V神的NFT远远大于此值，所以我们通过设置调高此参数。

![Vitalik NFT data](https://img.gopic.xyz/61bb99873822519b487d19521e458046.png)

使用以下代码可以获得排序后前十个结果，即V神持有量最大的 10 种NFT:
```python
vitalik_nft.groupby("contract").count().tokenId.sort_values(ascending=False)[:10]
```
结果如下:
![Vitalik NFT](https://img.gopic.xyz/a0213d40d5622e30663f9c0477511317.png)

## ERC721数据获取

对于`Basement`而言，其提供关于NFT的数据主要关于以下三个方面:

1. NFT MetaData
1. NFT转账
1. NFT 交易

我们会在下文逐一介绍以上内容。

### NFT MetaData

NFT 元数据是 NFT 最重要的属性之一，使用`basement`的API，我们可以获得NFT的基本属性和一些拓展属性。

如果想获得单一NFT的 MetaData，我们需要以下参数:

1. `conrtact` NFT合约地址
1. `tokenId` 单一NFT的 id

我们可以声明获取属性请参考[此列表](https://docs.basement.dev/schema/interfaces#nonfungibletoken)，个人猜测大部分属性来自 Opensea 平台。以下给出一个以`Bored Ape Yacht Club`(BAYC) 为参数的请求示例。

使用 Opensea 网站的搜索功能，搜索`Bored Ape`，我们获得[此网页](https://opensea.io/collection/boredapeyachtclub)，如下:

![Ape Opensea](https://img.gopic.xyz/d4d0ce782127d84b8f71054da13df43a.png)

通过点击`etherscan`标识按钮，我们可以获得此NFT的合约地址，即`0xbc4ca0eda7647a8ab7c2061c2e118a18a936f13d`

我们可以通过`basement`获得非常多的数据，以下是我准备的请求
```graphql
query NFTFirst {
    token(
        contract: "0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D"
        tokenId: "7292"
    ){
        name
        description
        image{
            url
        }
        tokenUri
        sales{
            transactionHash
            price
            marketplace
        }
        mintPrice
    }
}
```
读者应该可以很简单的凭借语义了解此处请求获取的数据，由于长度问题，此处仅给出部分返回值:
```json
{
    "data": {
        "token": {
            "description": null,
            "image": {
                "url": "https://cdn.basement.dev/291e642509d5516da28e0ce746e69d99/original"
            },
            "mintPrice": "80000000000000000",
            "name": null,
            "sales": [
                {
                    "marketplace": "OPENSEA",
                    "price": "53600000000000000000",
                    "transactionHash": "0x922910f07d81d91074f938b2c98d4eb89c065f720241f030f492a53f28499715"
                },
                ...
            ],
            "tokenUri": {
                "attributes": [
                    {
                        "trait_type": "Mouth",
                        "value": "Phoneme Vuh"
                    },
                    ...
                ],
                "image": "ipfs://QmXgtpxm5rMLkBqj9xbQb5w4GSy8vrLWvUP8kgenonYa4n"
            }
        }
    }
}
```

对于部分用户而言，您请求的NFT数据可能并不存在于`basement`数据库，如在编写此文时，无聊猿 #1 仍不存在，如下图:

![APE#1 Not Exist](https://img.gopic.xyz/371194690ebde64fcd3e9b448254b9e8.png)

此时我们可以通过一个特殊请求要求`basement`获取相关数据，请求如下:
```graphql
mutation {
    nonFungibleTokenRefresh(
        contract: "0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D"
        tokenId: "1"
    )
}
```
此处`mutation`关键词标识用户通过此请求可以修改服务器的相关数据。

返回如下:
```json
{
    "data": {
        "nonFungibleTokenRefresh": "This token has been queued for refresh, check back in a few minutes."
    }
}
```
再次进行请求，就可以获得NFT的相关数据，如下图:

![Add APE#1](https://img.gopic.xyz/6cd7bd262eaa5ff9476c313558c444a5.png)

最后，有部分读者可以有对一系列地址所拥有的NFT进行统一检索的需要，此时我们可以使用`tokens`索引方法，此方法需要以下参数:

1. `filter` 目前仅支持通过地址进行过滤
1. `limit` 单次请求返回的限额
1. `after`和`before` 均为分割点，一般不需要此参数

在编写此文时，此API无法实现真正的多用户NFT检索，返回如下:
```json
{
    "data": null,
    "errors": [
        {
            "locations": [
                {
                    "column": 5,
                    "line": 2
                }
            ],
            "message": "Providing multiple owner addresses is not supported right now but will be in the near future.",
            "path": [
                "tokens"
            ]
        }
    ]
}
```

### NFT 转账

通过 [Opensea 分析](https://opensea.io/collection/boredapeyachtclub/analytics)网页，我们可以发现`bayc.benddao.eth`是最大的BAYC持有人。

![BAYC Owner Rank](https://img.gopic.xyz/a30b5924f8679a45369b98e83236f30d.png)

我们希望获得此人所有`BAYC`的来源，我们可以通过`erc721Transfers`进行检索，此检索需要以下参数:

1. `filter` 过滤器，下文会详细介绍
1. `limit` 返回值限定数量
1. `after`和`before` 不常用参数，用于分页

此处的`filter`是一个对象，包含以下属性:

1. `fromAddresses` 限定发送者地址，仅允许ENS地址
1. `toAddresses` 限定接收者地址，仅允许ENS地址
1. `blockNumbers` 一个由需要查询交易的情况构成的列表
1. `toBlock` 检索区块的上限
1. `fromBlock` 检索区块下限
1. `contractAddresses` 合约地址
1. `tokenIds` 转移的NFT的id值

我们可以通过以下代码进行索引:
```graphql
query NFTtransferTest {
    erc721Transfers(
        filter: {
            toAddresses: "bayc.benddao.eth"
            contractAddresses: "0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D"
        }
        limit: 100
    ){
        erc721Transfers{
            blockNumber
            token{
                name
                tokenId
            }
            from{
                address
            }
        }
        totalCount
    }
}
```
当然，此处可以检索出所有转移给`bayc.benddao.eth`的`BAYC`。最后发现`totalCount`为`1897`，这是因为仅计入转入而未计入转出导致的。

> 此处的`benddao`实际上是一个 NFTfi 机构，提供质押蓝筹 NFT 进行贷款的服务，所以此地址会持有大量NFT

![NFT transfer Graphql](https://img.gopic.xyz/08675d5ad503dbb0efa7f58755f180b9.png)

### NFT 交易

实际上，此数据并不是一个单独的`Query`而是`Object`对象。由于很多用户特别需要此数据，所以在此处详细给出其属性，当然，读者亦可以直接参考[文档](https://docs.basement.dev/schema/objects#nonfungibletokensale)。

目前此API似乎仅支持 `Opensae` 平台，且数据似乎与平台给出的数据数量不同，一个简单的检索如下:
```garphql
query NFTSale {
    token(
        contract: "0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D",
        tokenId: "7116"
    ){
        sales{
            price
            marketplace
            maker{
                ...simpleAddress
            }
            taker{
                ...simpleAddress
            }
        }
    }
}

fragment simpleAddress on Address{
    address
    reverseProfile{
        name
    }
}
```

其中的`maker`和`taker`分别为NFT的卖家和买家，均属于`Address`属性数据，为避免重复，我们将此部分抽离出为`fragment`，其中的`on`标识此`fragment`用于何种对象。


## 交易数据检索

我认为此功能才是`basement`最强的功能，但需要读者具有一定以太坊的基础知识，读者应至少了解过交易的`Receipt Event Logs`。简单来说，在以太坊中进行的合约交互交易会抛出一系列事件，这些事件也会被包含在以太坊区块中，我们可以通过检索事件来寻找交易。

此处我们通过一个实战案例为大家介绍如何进行一次交易数据检索。此实战案例是获取[Dori Samurai](https://opensea.io/collection/dori-samurai)NFT项目的所有铸造交易。为达成这一目标，我们需要首先获得代表性交易，进入任一NFT详情页，点击`Item Activity`栏目内的`Minted`链接，如下图:

![Opensea Mint](https://img.gopic.xyz/25ebc73c9546e0fc78c945c7d0c2f5d1.png)

读者进入[Etherscan](https://etherscan.io/tx/0x40c9a4d8f3b4f40e8f9ecf6264b6693eb892f0ce1251f648536d86558589f35b#eventlog)页面，点击`Logs`选项卡，如下图:

![Mint Logs](https://img.gopic.xyz/6511d8598d37dd73423e22b4f55961ea.png)

我们发现此处抛出的交易事件为`Transfer(address,address,uint256)`，其中第一个参数为转账来源而第二个参数为接受方，最后一个参数为NFT的`tokenId`。对于大部分`event`，我们可以通过`etherscan`给出的参数判断含义，当然，查询对应的EIP也可以获得参数含义。如此处，我们可以查询[EIP721 Specification](https://eips.ethereum.org/EIPS/eip-721#specification)阅读注释，大部分基于EIP的含义均遵循此处给出的注释。

> 如果您发现此合约没有对应的EIP标准且`etherscan`未给出参数，我们只能通过阅读源代码获取`event`含义

对于`NFT`的`mint`操作，第一个参数始终为`0x0000000000000000000000000000000000000000`，即空地址。(对于`EIP20代币`的铸造也是从空地址转账)

第三个参数是接收方的地址，此地址可以随机改变。

基于以上内容，我们可以尝试构建检索交易过程中最重要的参数`filter`，此参数属于[TransactionLogFilter](https://docs.basement.dev/schema/inputObjects#transactionlogfilter)对象，最重要的参数为`topics`，其构造为`[[event][args1][args2]...]`，比如此处构造的检索`mint`的`topics`应构造为:
```
[["Transfer(address,address,uint256)"], ["0x0000000000000000000000000000000000000000"], [],[]]
```
其中`[]`代表匹配任一内容。

我们需要此处还需要一个参数为`transaction`，属于`TransactionLogTransactionFilter`对象，此对象仅存在两个属性，如下:

1. `toAddresses` 交易接受者，此处为NFT合约地址
1. `fromAddresses` 交易发起者，此处无需设置

我们可以构造出如下检索:
```graphql
query GetApprovalsForAddress {
    transactionLogs(
        filter: {
            topics: [["Transfer(address,address,uint256)"], ["0x0000000000000000000000000000000000000000"], [],[]]
            transaction: {
                toAddresses: ["0x6d9c17bc83a416bb992ccc671bebd98d7a76cfc3"]
            }
        }
        limit: 20
    ) {
        totalCount
        transactionLogs {
            transactionHash
            address {
                address
            }
            topics
            data
        }
  }
}
```
但是此检索会出现一个极其致命的错误，即`This query timed out.`。这是因为以太坊的区块数据规模过于庞大，所以无法在规定的时间内检索数据。一个简单的方法是限定扫描的区块范围，即设置`fromBlock`(扫描区块的起始)和`toBlock`(扫描区块的结束)。我们可以通过查询第一个NFT的`mint`区块作为起始，此处为`16117573`和第 888 个NFT的`mint`区块作为终止，此处为`16119963`，修正后的检索如下:
```graphql
query GetApprovalsForAddress {
    transactionLogs(
        filter: {
            topics: [["Transfer(address,address,uint256)"], ["0x0000000000000000000000000000000000000000"], [],[]]
            transaction: {
                toAddresses: ["0x6d9c17bc83a416bb992ccc671bebd98d7a76cfc3"]
            }
            fromBlock: 16117573 
            toBlock: 16119963
        }
        limit: 20
    ) {
        totalCount
        transactionLogs {
            transactionHash
            address {
                address
            }
            topics
            data
        }
  }
}
```
返回如下:

![Mint Graphql res](https://img.gopic.xyz/d34613ca63753371728b8ec12955cdc4.png)

读者可以自行构造一系列的复杂的交易查询以获得一些更加具有研究价值的数据。

## 总结

在此篇文章内，我们主要介绍了以下内容:

1. `Graphql`的基础知识
1. 查询地址数据
1. 查询NFT相关数据
1. 查询交易数据
