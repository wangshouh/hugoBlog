---
title: "图数据分析:使用 cozodb 分析以太坊交易数据"
date: 2023-02-22T23:47:33Z
tags: [graph,data]
---

## 概述

在之前的两篇文章中，我们讨论了对以太坊常规数据的导入和分析。文章链接如下:

1. [Clickhouse 以太坊分析:基础交易数据清洗](https://blog.wssh.trade/posts/clickhouse-with-blockchain/)
1. [Clickhouse 以太坊分析:交易日志分析](https://blog.wssh.trade/posts/clickhouse-eth-logs/)

> 如果您未阅读过以上文章并不影响您继续阅读本篇文章，本文内所有数据集均提供下载链接。

事实上，以太坊内存在复杂的交易关系，这些交易关系构成了一个庞大的图。如果需要分析这些复杂交易，我们需要进行图数据分析。

本文主要介绍如何进行基于以太坊数据进行图数据分析，主要包括以下内容:

1. 图可视化
1. 图检索

对于图可视化，我选择了 [CosmosGraph](https://cosmograph.app/) 作为主要工具，而对于图检索，我选择了 [cozodb](https://www.cozodb.org/) 图数据库。当然，读者也可以使用 `neo4j` 或其他图数据库。

## 图可视化

在本节中，我们主要针对 ERC-20 代币的转账进行可视化处理。此处我们选择使用基于 `WebGL` 的可在浏览器内运行的高性能图可视化工具 `CosmosGraph` 。`CosmosGraph` 要求我们使用如下格式导入数据:

```
time, source, target, value
2/4/2022, node1, node2, 2
2/5/2022, node1, node3, 10
```

其中，`time` 和 `value` 是可选的，`time` 用于添加时间轴，如下图:

![CosmosGraph TimeLine](https://img-blog.csdnimg.cn/13cec5ef722548f4a11d4dd2926d7e70.webp)

最下方的即为根据 `time` 生成的时间轴。我们可以在 `csv` 中增加任意数据用于生成节点大小或指定线条颜色等。

我们需要在 `clickhouse` 中导出相关数据:

```sql
SELECT
	reinterpretAsUInt64(reverse(unhex(txblockNumber))) as `time`,
	substr(topics[2], 27) as txFrom, 
	substr(topics[3], 27) as txTo,
	reinterpretAsUInt128(reverse(unhex(txlogsData))) as txValue,
	symbol,
	concat('#', substr(hex(cityHash64(symbol)), 11)) as linkColor
FROM
	logsTemp lt
LEFT OUTER JOIN tokenInfo ti ON
	lt.contractAddress = LOWER(ti.address)
WHERE
	topics[1] == '0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef'
	AND
	topics.size0 == 3
```

由于 `topics` 中的 `from` 和 `to` 均为 `uint256` 数据类型，而实际上 `address` 为 `uint160` 类型，这导致原始数据存在大量前导 0 ，此处我们使用 `substr` 函数去除前导 0 以方便阅读。为了区分不同代币转账，我们使用 `concat('#', substr(hex(cityHash64(symbol)), 11))` 拼接得到形如 `#5EC1A6` 的 16 进制 RGB 颜色。

我们可以通过 `DBeaver` 附带的工具进行数据导出:

![Clickhouse export data](https://img-blog.csdnimg.cn/img_convert/f14a8fc02c90b35d12edc8ddff1fc803.png)

导出时所有对话框均可为默认配置即可，只需要记住 `csv` 导出地址即可。

> 导出数据可以使用 [此链接](https://void.cat/d/Aup1RKAzzmqv8x9LK1Q8Sv) 进行下载

打开 [此网站](https://cosmograph.app/run/) ，点击 `Select data file` ，选择之前导出的数据集，如下:

![CosmosGraph Setting](https://img-blog.csdnimg.cn/img_convert/abf070082ac9b057eca0eb5ce55a30b5.png)

在此处，我们设置节点大小由链接数目决定，节点颜色由转账总价值决定，而链接颜色由 `linkColor` 决定，点击 `Launch` 启动渲染。最后，我们可以得到如下图像:

![ComosGraph Result](https://img-blog.csdnimg.cn/img_convert/d8503975dbd4d2193e2518ba638536bb.png)

读者可以根据自身需求研究可视化图像。如果读者希望对 `CosmosGraph` 有更多了解，可以参考 [How to Visualize a Graph with a Million Nodes](https://nightingaledvs.com/how-to-visualize-a-graph-with-a-million-nodes/) 

## 图分析

目前市面上存在了大量图数据库，其中的代表是 `neo4j` 图数据库，但在本文中，我们准备使用 `cozodb` 作为图数据库。原因有以下几点:

1. 嵌入式数据库，易于安装且轻量化
1. 可以处理百万量级内的图数据集，对于当下的项目是完全够用的
1. 开发者是国人，其全部文档均有汉语版本
1. 使用 Rust 编写

但是，就目前体验而言，`cozodb` 的缺点如下:

1. 使用特定的数据重新语言(`CozoScript`)，而不是常用的 `Cypher` 等常用的图数据查询语言
1. 查询语言的错误显示较差，不会具体显示查询语言错误的具体位置，Debug 比较令人烦恼

如果读者希望获得更多关于 `cozodb` 的信息，可以访问 [文档](https://docs.cozodb.org/zh_CN/latest/tutorial.html)。

### 安装 cozodb

`cozodb` 的开发者在 [Github Realse](https://github.com/cozodb/cozo/releases) 中提供了已经编译好的二进制文件，
读者可根据自身要求选择合适架构的编译产物，使用 `curl` 或 `wget` 下载到服务器后，命令如下:

```bash
wget https://github.com/cozodb/cozo/releases/download/v0.5.1/cozo-0.5.1-x86_64-unknown-linux-musl.gz
```

使用以下命令进行解压缩:

```bash
gunzip cozo-0.5.1-x86_64-unknown-linux-musl.gz
```

然后进行重命名和权限调整:

```bash
mv cozo-0.5.1-x86_64-unknown-linux-musl cozoserver
chmod 777
```

可以使用以下命令测试是否安装成果:

```bash
./cozoserver -V
```

### 数据清洗

我们首先需要准备数据导出的 `SQL` 文件，使用 `vi export.sql` 创建并键入以下代码:

```sql
SELECT
        txHash,
        substr(topics[2], 27) as txFrom,
        substr(topics[3], 27) as txTo,
        reinterpretAsUInt128(reverse(unhex(txlogsData))) / 1000 as txValue,
        symbol
FROM
        logsTemp lt
LEFT OUTER JOIN tokenInfo ti ON
        lt.contractAddress = LOWER(ti.address)
WHERE
        topics[1] == '0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef'
        AND
        topics.size0 == 3
```

在将数据导入图数据之前，我们需要在 `clickhouse` 中导出需要的数据，我们直接在服务器内使用以下命令导出数据并保存为 `ERC20cozo.csv` 文件:

```bash
clickhouse-client --queries-file export.sql -f CSVWithNames > ERC20cozo.csv
```

> 使用此命令可能会出现 `. (AUTHENTICATION_FAILED)` 报错，此时读者需要增加 `--password` 参数手动输入用户(默认为 `deafult`)密码

> 如果您没有完成之前的系列教程，但对于图数据库感兴趣，您可以直接使用 [此链接](https://void.cat/d/D2kDpUQqyywFpU8uxJSCyW) 下载数据集。

使用以下命令检查数据情况:

```bash
head ERC20cozo.csv
```

接下来，我们需要执行数据导入工作，首先使用以下命令启动 `cozodb` 命令行:

```bash
./cozoserver -r
```

由于我们需要多行输入导入指令，所以我们首先键入空格后进行回车进入多行模式，然后输入以下内容:

```
res[txHash,txFrom,txTo,txValue,symbol] <~
    CsvReader(types: ['String', 'String', 'String', 'Float', 'String?'],
              url: 'file://ERC20cozo.csv',
              has_headers: true)
?[txHash, txFrom, txTo, txValue, symbol] := res[txHash,txFrom,txTo,txValue,symbol]
:replace ERC20trans {
    txHash: String,
    txFrom: String,
    txTo: String,
    txValue: Float,
    symbol: String?
}
```

> 由于终端比较难以进行大量内容键入，我建议读者在文本编辑器内编写相关指令，然后复制到终端内运行

如果没有问题，读者会受到如下回复:

```
 status
--------
 "OK"
```

接下来，我们讨论上述指令是如何导入数据的。

此处我们讨论的 `cozoscript` 是 `Datalog` 的一种方言，所以其规则与 `SQL` 差异较大。

> 关于 `Datalog` 语言的一些基础问题，读者可阅读 [An introduction to Datalog](https://blogit.michelin.io/an-introduction-to-datalog) 博客

在数据导入过程中，我们使用了 `csv` 文件阅读函数 `CsvReader`，其具体的相关参数请自行参考 [文档](https://docs.cozodb.org/zh_CN/latest/algorithms.html#Algo.CsvReader)。其中 `types` 用来指定数据类型，`cozodb` 支持的数据类型请参考 [此文档](https://docs.cozodb.org/zh_CN/latest/datatypes.html#id4)

此处使用了 `res` 作为暂时存储数据的表(事实上，在文档中一般使用 **规则** 描述)。我们也可以看到此处使用了 `<~` 连接 `CsvReader` ，原因在于 `CsvReader` 为 **固定规则**(可以理解为是一张不可变的表)。

> 此处的操作简单来说，就是将数据从不可变的 `CsvReader` 通过 `<~` 操作符转移到可修改的 `res` 表格以方便数据清洗

`?[...] := res[...]` 事实上就是以下 SQL 代码:

```sql
SELECT ...
FROM res
```

所有使用 `?` 开始的语句都是数据查询语句，并会将结果进行输出，我们会在后文更加详细的展示此类语句的用法。

最后使用 `:replace` 实现对查询数据的存储，关于 `:replace` 的更多信息请参考 [存储表](https://docs.cozodb.org/zh_CN/latest/stored.html#id2)

### 数据查询

在完成数据导入后，我们开始聚焦于从原始数据中进行查询以提取获得更多有效的信息。我们首先展示进行一些常规的数据查询，这些查询在关系型数据库中也可以高效完成。

第一个任务是获取 ERC20 代币最大转移量的前五个交易的哈希值、数量和代币名称:

```
?[txHash, txValue, symbol] := *ERC20trans{txHash, txValue, symbol}
:order -txValue
:limit 5
```

输出如下:

```
 txHash                                                               | txValue               | symbol
----------------------------------------------------------------------+-----------------------+--------
 "0xf13b0604f2e4b76fda01a58d307b13b2be746370fb4743031e4566d99191845d" | 1.0148631219570396e28 | ""
 "0x502a52913e7f9fa7fdd046e2b96ea0b95ccb2d2d748d3c737b3af0a4eae08c00" | 9.73730152319995e27   | ""
 "0x9a17ec13d8d782dc598b0c27c3de26df13d08420fa89b6bc302ecb395bad3ddb" | 8.358515999253435e27  | ""
 "0x119d05dd92a73414fd994c76417f4bc533c70b3228fdb9744ac122a69abcba06" | 8.274930839260901e27  | ""
 "0xd73951fec0265eabc430fae11fdc6fa4e471e7e9ef7c3c86e74051f4d3570a67" | 6.052257673869336e27  | ""
```

此处的 `*ERC20trans{...}` 意为从 `ERC20trans` 表格内进行数据读取，同时在花括号内指定了列的读取情况。我们也使用了 `:order` 进行数据排序和 `:limit` 限制输出范围。我们可以看到该语法与 SQL 在思想上有一定相似性。

接下来，我们基于以上检索进行优化，我们希望输出的交易中 `symbol` 都不为空值:

```
?[txHash, txValue, symbol] :=
    *ERC20trans{txHash, txValue, symbol}, symbol != ""
:order -txValue
:limit 5
```

此处我们使用了 `symbol != ""` 作为限制条件对检索结果进行限制。结果如下:

```
 txHash                                                               | txValue               | symbol
----------------------------------------------------------------------+-----------------------+--------
 "0xde28c7ccab402c5f760df71392e2925fc7f9bd267d2d6694c93225fea7cf6b54" | 4.3234586859142595e25 | "SHIB"
 "0xb889ff3313fb60277f30716fabe84301eb976a2c64418324d6b773142c36404f" | 4.06112211860191e25   | "SHIB"
 "0xa96612457c1ca1dd2a58c2a03575dd5584e1539a16e54d88a466a744b6d3dece" | 2.6586093083999996e25 | "ELON"
 "0x89626f945d9a97075e8b81ec97e77351dbd72d40cc09217c140b87282b3a038b" | 2.1260568085000002e25 | "ELON"
 "0x309d3e8483ca521c48dba6caffa66db62eb2b01916d16d7644f73e62c176c726" | 6.771431572110383e24  | "ELON"
```

获取 ERC20 代币转出最多的地址:

```
?[txFrom, count(txFrom)] := *ERC20trans{txFrom}
:order -count(txFrom)
:limit 5
```

此处相当于进行了 `GROUP BY` 操作，此操作被称为 **聚合**，具体的介绍可以参考 [文档](https://docs.cozodb.org/zh_CN/latest/tutorial.html#%E8%81%9A%E5%90%88)

此查询返回值如下:

```
 txFrom                                     | count(txFrom)
--------------------------------------------+---------------
 "a5025faba6e70b84f74e9b1113e5f7f4e7f4859f" | 781
 "ef1c6e67703c7bd7107eed8303fbe6ec2554bf6b" | 638
 "0000000000000000000000000000000000000000" | 295
 "a9d1e08c7793af67e9d92fe308d5697fb81d3e43" | 223
 "74de5d4fcbf63e00296fd95d33236b9794016631" | 220
```

> 笔者使用 `clickhouse` 进行聚合操作的结果与之相同

此处的 `a5025faba6e70b84f74e9b1113e5f7f4e7f4859f` 是一个 [multisender.app](https://multisender.app/#/) 的地址，该应用用来大批量转账。而 `a9d1e08c7793af67e9d92fe308d5697fb81d3e43` 是知名交易所 `Coinbase` 的地址。我们接下来主要分析 `Coinbase` 地址的一系列特征。

检索 `Coinbase` 地址转出的最多的五种代币:

```
?[symbol, count(symbol), sum(txValue)] :=
    *ERC20trans{txFrom, symbol, txValue},
    txFrom == "a9d1e08c7793af67e9d92fe308d5697fb81d3e43"
:order -sum(txValue)
```

输出如下:

```
 symbol  | count(symbol) | sum(txValue)
---------+---------------+-----------------------
 "SHIB"  | 19            | 6.190227701038489e23
 ""      | 9             | 1.680186087552655e20
 "GRT"   | 3             | 6.776611658788e19
 "ANKR"  | 1             | 4.995122396136e19
 "AMP"   | 2             | 3.116081336054e19
 "LCX"   | 4             | 1.019582581807e19
 "CRV"   | 1             | 9.91008e18
 "RAD"   | 1             | 8.86793e18
 "MATIC" | 11            | 3.9549959397700004e18
 "BUSD"  | 4             | 1.06720634928e18
 "CHZ"   | 1             | 7.1663566256e17
 "FARM"  | 1             | 6.1458777923e17
 "PRQ"   | 2             | 5.6033880077e17
 "APE"   | 4             | 3.7451645757e17
 "AAVE"  | 2             | 2.516280078e17
 "LRC"   | 1             | 1.6091394989e17
 "LQTY"  | 1             | 3.802376091e16
 "RNDR"  | 1             | 1.707504482e16
 "LINK"  | 1             | 5666448920000000.0
 "QNT"   | 2             | 1454150950000000.0
 "LUNA"  | 1             | 1000000000000000.0
 "SAND"  | 1             | 250293580000000.0
 "GALA"  | 4             | 90313496065.979
 "USDC"  | 54            | 5163323390.892002
 "USDT"  | 89            | 1854740686.8849998
 "CRO"   | 3             | 510447040.74799997
```

在此处，我们需要说明所有 `:=` 后的语句会构成一个规则集，`cozodb` 会使用此规则集查询我们需要的数据，在上文中，我们使用了 `,` 连接规则集，这意味各条规则之间术语 **并** 的关系。

上述给出的一系列查询在关系型数据库中使用 SQL 也可以简单的完成，始终没有涉及 `cozodb` 的关键功能，即对于图检索的支持。继续分析上述 `coinbase` 地址，我们需要知道从 `coinbase` 转出资产的人又在哪里消费了这些资产，如下图:

<img src="https://files.catbox.moe/v9elbc.svg">

我们的目标是找出 `interactiveAddress` 地址，查询代码如下:

```
?[interactiveAddress, count(interactiveAddress)] := 
    *ERC20trans{txFrom: "a9d1e08c7793af67e9d92fe308d5697fb81d3e43", txTo: coinbaseReciver},
    *ERC20trans{txFrom: coinbaseReciver, txTo: interactiveAddress}
:order -count(interactiveAddress)
:limit 5
```

返回结果如下:

```
interactiveAddress                         | count(interactiveAddress)
--------------------------------------------+---------------------------
 "56178a0d5f301baf6cf3e1cd53d9863437345bf9" | 45
 "28c6c06298d514db089934071355e5743bf21d60" | 9
 "6dfc34609a05bc22319fa4cce1d1e2929548c0d7" | 5
 "21202591a02ee22ecc9a4e3a11ab2cf8956f5b07" | 2
 "55fe002aeff02f77364de339a1292923a15844b8" | 2
```

在上述检索中，我们使用了所谓的 **绑定** ，即将某一个数据与变量名绑定以方便后文调用。在检索出的结果中，`56178a0d5f301baf6cf3e1cd53d9863437345bf9` 是一个巨鲸，但其身份未被揭露，该地址持有大量资产，地址可前往 [此页面](https://bloxy.info/address/0x56178a0d5f301baf6cf3e1cd53d9863437345bf9) 看到该地址的链上财务信息。而 `28c6c06298d514db089934071355e5743bf21d60` 是 Binance 的地址，从交易记录来看，此地址主要用于入账。

我们希望对 `56178a0d5f301baf6cf3e1cd53d9863437345bf9` 的间接提款交易进行追查，因为中介方可能包含更多信息。我们构造的查询如下:

```
?[transferMeida, meidaToken, meidaHash, transferHash] :=
    *ERC20trans{
        txHash: meidaHash, 
        txFrom: "a9d1e08c7793af67e9d92fe308d5697fb81d3e43", 
        txTo: transferMeida,
        symbol: meidaToken
    },
    *ERC20trans{
        txHash: transferHash,
        txFrom: transferMeida, 
        txTo: "56178a0d5f301baf6cf3e1cd53d9863437345bf9",
        symbol: meidaToken
    }
```

最终获得如下输出:


| transferMeida                            | meidaToken | meidaHash                                                          | transferHash                                                       |
|:----------------------------------------:|:----------:|:------------------------------------------------------------------:|:------------------------------------------------------------------:|
| fa103c21ea2df71dfb92b0652f8b1d795e51cdef | AAVE       | 0x3fe283fc31c2746e4cb76609bd9c5bbcaeb40dc05c0573e4d6406acd8745199e | 0x99aa41dfa8847e6dfd9d487d464a81470a7daf426bb81445e595f7e2c5fccd50 |
| fa103c21ea2df71dfb92b0652f8b1d795e51cdef | AAVE       | 0x3fe283fc31c2746e4cb76609bd9c5bbcaeb40dc05c0573e4d6406acd8745199e | 0xe8b31a052cb39c4c983781c621d7529f19bef60d1cda6b825639e16925fead23 |
| fa103c21ea2df71dfb92b0652f8b1d795e51cdef | RAD        | 0x0da42a3520d73474325bb625c90c9349ea3cd19b2935381a4740502c024f0e6c | 0xa584e28537fa845d52f07df1302e5dc2f50c759fb049a52dbe2e24ceeb6c4dab |
| fa103c21ea2df71dfb92b0652f8b1d795e51cdef | USDC       | 0x14fc352b81aa947785134e521fc4534e14b98806f4e2ea9e4f20256f71d4d967 | 0x02260768f2a62bc103c08455f1159d7d2a6eeab91201a08153f7d6c305b4931e |
| fa103c21ea2df71dfb92b0652f8b1d795e51cdef | USDC       | 0x14fc352b81aa947785134e521fc4534e14b98806f4e2ea9e4f20256f71d4d967 | 0x0b269c40c0f1677521d8c8019e3455d33df83fb5c4e83e297d15fbe0323a74c0 |
| fa103c21ea2df71dfb92b0652f8b1d795e51cdef | USDC       | 0x14fc352b81aa947785134e521fc4534e14b98806f4e2ea9e4f20256f71d4d967 | 0xcdda0e83cd76549dcba3fda75ae357a5fbca5ee0b9c9519bc34872216ff9c304 |
| fa103c21ea2df71dfb92b0652f8b1d795e51cdef | USDC       | 0x30f487d24fb6767a10c8d520eeb8a8580180d25bdd2164e877ecc9d79d0d0074 | 0x02260768f2a62bc103c08455f1159d7d2a6eeab91201a08153f7d6c305b4931e |
| fa103c21ea2df71dfb92b0652f8b1d795e51cdef | USDC       | 0x30f487d24fb6767a10c8d520eeb8a8580180d25bdd2164e877ecc9d79d0d0074 | 0x0b269c40c0f1677521d8c8019e3455d33df83fb5c4e83e297d15fbe0323a74c0 |
| fa103c21ea2df71dfb92b0652f8b1d795e51cdef | USDC       | 0x30f487d24fb6767a10c8d520eeb8a8580180d25bdd2164e877ecc9d79d0d0074 | 0xcdda0e83cd76549dcba3fda75ae357a5fbca5ee0b9c9519bc34872216ff9c304 |
| fa103c21ea2df71dfb92b0652f8b1d795e51cdef | USDC       | 0x3429d25645e66e9539aafd52b65b322a0ab02e0ee6dfc6caaad00af8f9026585 | 0x02260768f2a62bc103c08455f1159d7d2a6eeab91201a08153f7d6c305b4931e |
| fa103c21ea2df71dfb92b0652f8b1d795e51cdef | USDC       | 0x3429d25645e66e9539aafd52b65b322a0ab02e0ee6dfc6caaad00af8f9026585 | 0x0b269c40c0f1677521d8c8019e3455d33df83fb5c4e83e297d15fbe0323a74c0 |
| fa103c21ea2df71dfb92b0652f8b1d795e51cdef | USDC       | 0x3429d25645e66e9539aafd52b65b322a0ab02e0ee6dfc6caaad00af8f9026585 | 0xcdda0e83cd76549dcba3fda75ae357a5fbca5ee0b9c9519bc34872216ff9c304 |

我们可以看到 `fa103c21ea2df71dfb92b0652f8b1d795e51cdef` 将许多从 `Coinbase` 中提取的资产转移给了 `56178a0d5f301baf6cf3e1cd53d9863437345bf9` 地址，这说明该地址与 `56178a0d5f301baf6cf3e1cd53d9863437345bf9` 的实控人可能是同一人。但我们仍不确认相关情况，我们需要检查 `fa103c21ea2df71dfb92b0652f8b1d795e51cdef` 的交易情况，如果该地址仅与 `56178a0d5f301baf6cf3e1cd53d9863437345bf9` 进行转账则说明该地址与上述地址时控人为同一人。因此，我们可以构造以下查询:

```
?[txTo, symbol] := *ERC20trans{txFrom, txTo, symbol}, txFrom="fa103c21ea2df71dfb92b0652f8b1d795e51cdef"
```

获得如下检索结果:

 txTo                                       | symbol
--------------------------------------------+--------
 "56178a0d5f301baf6cf3e1cd53d9863437345bf9" | "AAVE"
 "56178a0d5f301baf6cf3e1cd53d9863437345bf9" | "LDO"
 "56178a0d5f301baf6cf3e1cd53d9863437345bf9" | "RAD"
 "56178a0d5f301baf6cf3e1cd53d9863437345bf9" | "USDC"
 "56178a0d5f301baf6cf3e1cd53d9863437345bf9" | "WETH"

我们基本可以确认 `fa103c21ea2df71dfb92b0652f8b1d795e51cdef` 和 `56178a0d5f301baf6cf3e1cd53d9863437345bf9` 为同一人的两个地址，其中 `fa103c21ea2df71dfb92b0652f8b1d795e51cdef` 作为资产暂存地址。

> 该部分内容事实上属于 `OSINT` ，即开源情报调查，读者可以通过 [此篇文章](https://medium.com/@ibederov_en/ethereum-eth-osint-investigations-tools-7d1ec5deab1e) 获得更多关于以太坊开源情报调查的工具

### 图算法

#### 路径搜索

在本节中，我们主要讨论一些较为常见的图算法，包括路径查找算法和社群发现算法，`cozodb` 实现了大量的图算法，具体可以参考 [文档](https://docs.cozodb.org/zh_CN/latest/algorithms.html) 。但文档内缺少部分示例，所以本文会手把手介绍如何使用 `cozodb` 自带图算法。

> 关于图算法，读者可以阅读 [《数据分析之图算法：基于Spark和Neo4j》](https://book.douban.com/subject/35217091/)，此书内大部分图算法均已被 `cozodb` 实现。如果读者希望更加深入的了解这一领域，可以关注 **网络科学** 这一学科主题，我认为 [《巴拉巴西网络科学》](https://book.douban.com/subject/34970365/) 是一本很好的入门书

我们首先通过一个简单的示例讨论路径搜索算法的使用。在上文中，我们发现存在部分代币从 Coinbase 被转移到了 Binance 中，这种交易所转换行为是令人较为好奇的，我们希望获得所有从 Coinbase 转换到 Binance 的用户地址。这是一个简单的路径查询问题，我们已经确认起点是 Coinbase 的地址`a9d1e08c7793af67e9d92fe308d5697fb81d3e43` ，而终点是 Binance 的存款地址，我们只需要一个算法以 Coinbase 地址作为起点沿交易进行搜索直到 Binance 的地址即可。显然，这是一个寻路问题，如果读者熟悉一般的寻路算法，就会发现宽度优先搜索(BFS)和深度优先搜索(DFS)都可以完成此过程。

下图展示了两者搜索的基本过程:

![DFS and BFS](https://img.gejiba.com/images/e7e06e0b76600656bdfcc06f1bc7a0e3.png)

在正式进行搜索前，我们需要获得 Binance 的存款地址。为方便后文讨论，我们仅假设用户只使用 [Binance 14](https://etherscan.io/address/28c6c06298d514db089934071355e5743bf21d60) 进行存款操作。

> 事实上， Binance 应该拥有 42 个用于交互的地址。这些地址可以在此包含所有 Etherscan Tag 标签的[JSON 文件]([https://github.com/brianleect/etherscan-labels/raw/main/combined/combinedLabels.json) 内获得。

我们首先通过文档发现以下定义:

```javascript
BreadthFirstSearch(edges[from, to], nodes[idx, ...], starting?[idx], condition: expr, limit: 1)
```
此算法为 BFS 算法的定义，而 DFS 定义与之完全相同。该函数需要以下参数:

1. `edges` 由边构成的集合。
    在此案例中即由 `txFrom` 和 `txTo` 转账关系。注意此处使用了 `[from, to]` 说明要求输入表格的有且仅有两列且必须为 `from` 和 `to` 。此处涉及到 `[]` 的提取问题，该操作符会按顺序提取数据，且要求列与`[]` 内的元素数量相同，具体参考 [文档](https://docs.cozodb.org/zh_CN/latest/queries.html#id3)。

    简单来说，输入的 `edges` 进入函数后，函数会首先检查列数是否为 2 列，然后将第一列作为 `from` ，而第二列作为 `to` 。如果我们输入的表格为三列，则会直接报错。

    显然，我们的 `ERC20trans` 不符合要求，我们需要建立一张临时表，如下:

    ```
    transfer[from, to] := *ERC20trans{txFrom: from, txTo: to}
    ```

    该表格仅存在两列，且符合要求。
1. `nodes` 由节点构成的集合
    注意此处使用了 `[idx, ...]` 限制，该限制要求 `nodes` 表第一列是与 `edges` 对应的标识符，在本案例中，`idx` 应为交易者地址。而 `...` 表示 `nodes` 表格可具有其他任意列，这些列可以用于 `condition` 参数进行条件判断。

    我们可以使用以下代码构造 `nodoes` :

    ```
    nodeAddress[address] := *ERC20trans{txFrom: address}, *ERC20trans{txTo: address};
    ```

    `cozodb` 严格遵循集合语义，所以此处的检索会自动去重
1. `starting` 开始进行搜索的节点
    `?` 说明本参数可以置为空，但一旦置为空就会在整个图数据集内进行搜索，显然，这导致大量运算。而 `[idx]` 参数也说明该表格只能存在一列，该列与 `edges` 对应。

    在此案例中，我们构造如下检索即可:

    ```
    coinBase[address] := nodeAddress[address], address="a9d1e08c7793af67e9d92fe308d5697fb81d3e43";
    ```
1. `condition` 中止搜索的条件。该条件应该为一个布尔表达式
1. `limit` 产生结果的数量。默认为 `1` ，即找到一个符合 `condition` 的结果后立即停止搜索。


综上所述，我们最终构造的检索如下:

```
transfer[from, to] := *ERC20trans{txFrom: from, txTo: to}
nodeAddress[address] := *ERC20trans{txFrom: address}, *ERC20trans{txTo: address};
coinBase[address] := nodeAddress[address], address="a9d1e08c7793af67e9d92fe308d5697fb81d3e43";
?[] <~ BFS(
    transfer[],
    nodeAddress[address],
    coinBase[],
    condition: address == "28c6c06298d514db089934071355e5743bf21d60",
    limit: 200
);
```

搜索结果如下:


| _0                                         | _1                                         | _2 |
|-------------------------------------------|--------------------------------------------|-------------------|
| "a9d1e08c7793af67e9d92fe308d5697fb81d3e43" | "28c6c06298d514db089934071355e5743bf21d60" | ["a9d1e08c7793af67e9d92fe308d5697fb81d3e43","4d078ccdf62e287514adb078fdfddaddb1890067","28c6c06298d514db089934071355e5743bf21d60"] |

上述结果中，各列含义如下:

1. `_0` 搜索的起始节点
1. `_1` 搜索的终点
1. `_2` 搜索的路径

我们可以看到 `4d078ccdf62e287514adb078fdfddaddb1890067` 进行了交易所转换，我们可以查找一下该地址转移代币的详细情况:

```
?[txHash, txValue, symbol] := 
    *ERC20trans{txHash, txValue, symbol, txFrom, txTo},
    (txFrom = "a9d1e08c7793af67e9d92fe308d5697fb81d3e43", txTo = "4d078ccdf62e287514adb078fdfddaddb1890067")
    or
    (txFrom = "4d078ccdf62e287514adb078fdfddaddb1890067", txTo = "28c6c06298d514db089934071355e5743bf21d60")
```

我们通过 `or` 连接多条筛选规则，实现了交易的提取。由于 `cozodb` 是大小写敏感的数据库，所以此处的 `or` 只能使用小写。在此处，我们也给出 `cozodb` 中各算符的优先级，即算符 `and` 的算符优先级比 `or` 高， `or` 又比逗号高，逗号在语义上与 `and` 一致，但优先级不同。

最后的搜索结果如下:

```
 txHash                                                               | txValue     | symbol
----------------------------------------------------------------------+-------------+--------
 "0x35dc94f4af4a7b4dd825b6f6b46531f9d77f883556945395d80093cdb6f1b0df" | 1422436.829 | "USDC"
 "0x99b993493cae0a4edfa36b2cae924d708c875cc176d359c483e61a1f7132e657" | 1422436.829 | "USDC"
```

该用户提取了 USDC 并将其转账到 Binance 。

如果读者使用 `DFS` 算法进行搜索，则会得到一个极其离谱的代币转移路径，如下:

```
["a9d1e08c7793af67e9d92fe308d5697fb81d3e43","fa103c21ea2df71dfb92b0652f8b1d795e51cdef",
"56178a0d5f301baf6cf3e1cd53d9863437345bf9","f8a95b2409c27678a6d18d950c5d913d5c38ab03",
"74de5d4fcbf63e00296fd95d33236b9794016631","fca9090d2c91e11cc546b0d7e4918c79e0088194",
"1111111254eeb25477b68fb85ed929f73a960582","e3d3551bb608e7665472180a20280630d9e938aa",
"fc25a25afb6f86bcb37825ae4d63ccaef6fae213","e66b31678d6c16e9ebf358268a790b763c133750",
"f7d31825946e7fd99ef07212d34b9dad84c396b7","def1c0ded9bec7f1a1670819833240f027b25eff",
"e5d028350093a743a9769e6fd7f5546eeddaa320","9008d19f58aabd9ed0d60971565aa8510560ab41",
"f2f12b364f614925ab8e2c8bfc606edb9282ba09","99a58482bd75cbab83b27ec03ca68ff489b5788f",
"fd049161e80f5e665af09e4e23b8e78ac44f11d8","aa0c3f5f7dfd688c6e646f66cd2a6b66acdbe434",
"3fe65692bfcd0e6cf84cb1e7d24108e434a7587e","0be55326919f08af4d14a42aafb3b68f95738355",
"def171fe48cf0115b1d80b88dc8eab59176fee57","ede8dd046586d22625ae7ff2708f879ef7bdb8cf",
"e592427a0aece92de3edee1f18e0157c05861564","e0554a476a092703abdb3ef35c80e0d76d32939f",
"f71530c1f043703085b42608ff9dcccc43210a8e","a69945a0f0addd9953092b5b5df5692909d2c588",
"a5ef2a6bbe8852bd6fd2ef6ab9bb45081a6f531c","d119c9dabe94d67485fef94c98b10602dc24fad1",
"69d91b94f0aaf8e8a2586909fa77a5c2c89818d5","e4000004000bd8006e00720000d27d1fa000d43e",
"3328ca5b535d537f88715b305375c591cf52d541","f02dcfaa7c5d0260a81e81b9ccb2ce73e42b66ef",
"ef1c6e67703c7bd7107eed8303fbe6ec2554bf6b","f7ef5352de45a20d8c8565cd94a4bd6c8831f749",
"a9cb710be0f7d4091f7185d46008c4b12c6ab203","11b815efb8f581194ae79006d24e0d814b7697f6",
"c438655d0d1e83fd714d02acf7a6797fb657d32f","4c54ff7f1c424ff5487a32aad0b48b19cbaf087f",
"98c3d3183c4b8a650614ad179a1a98be0a8d6b8e","e56c60b5f9f7b5fc70de0eb79c6ee7d00efa2625",
"53222470cdcfb8081c0e3a50fd106f0d69e63f20","fad57d2039c21811c8f2b5d5b65308aa99d31559",
"1111111254fb6c44bac0bed2854e76f90643097d","919ddd07650f2b6fcd0f7178e19fc3d737cbbab6",
"61aae18051498d255a92cc15106085f933d8fb3a","28c6c06298d514db089934071355e5743bf21d60"]
```

简单查看了最后一笔和第一笔交易都是正确的，但中间过程没有进行详细研究，读者如果对此感兴趣可以自己研究此代币转移流程。

#### 社群发现

可能有部分对于图算法不熟悉的用户不清楚 **社群发现** 的含义。社群发现指在大规模图数据库中找到一系列属于同一集团的节点。发现社团的一般性原则是，社团成员在群组内部的关系要多于其与群组外部节点的关系。理论上，我们可以使用社群发现算法识别属于同一实体的地址。值得注意的是，使用社群发现算法识别属于同一实体的节点是存在一些问题的。正如上文所述，社群发现的基本原则是 `社团成员在群组内部的关系要多于其与群组外部节点的关系` ，某些实体可以通过创建一系列钱包并在长时期内不进行实体内钱包交互实现来避免被社群发现算法发现。

社群发现算法主要包括以下几类:

1. 三角形计数和聚类系数
1. 强连通分量算法
1. 连通分量算法
1. 标签传播算法
1. Louvain模块度算法

我们不会在本文内详细讨论这些算法的实现等，读者可以自行阅读 《数据分析之图算法：基于Spark和Neo4j》 查看算法的一些细节。

`cozodb` 对应的函数如下:

1. `ClusteringCoefficients` 三角形计数
1. `CommunityDetectionLouvain` Louvain模块度算法
1. `LabelPropagation` 标签传播算法

在这些算法中，`LabelPropagation` 具有以下特点:

> 标签传播算法通常用于发现大规模网络中的初始社团，当权重可用时尤其适用。该算法可以并行化处理，因此在分割图时速度非常快。

在本案例中，我们主要使用此算法。

为方便后文判断社群发现的效果，我们导入 `etherscan` 标签数据集:

```
res[address, name] <~ 
    CsvReader(types: ['String', 'String?'],
    url: 'https://void.cat/d/6QpPmFCHz42RrBGD9EQbEs',
    has_headers: false)
?[address, name] := res[address, name]
:replace etherscanTags {
    address: String
    =>
    name: String?
}
```

此处使用的 CSV 文件是我使用 Python 对 [Etherscan CombinedLables JSON](https://github.com/brianleect/etherscan-labels/blob/main/combined/combinedLabels.json) 文件清洗出的。基本格式如下:

```
0x0e8ba001a821f3ce0734763d008c9d7c957f5852,AmadeusRelay
0xc898fbee1cc94c0ff077faa5449915a506eff384,Bamboo Relay
0x58a5959a6c528c5d5e03f7b9e5102350e24005f1,ERC dEX
```

在文档中，`LabelPropagation` 定义如下:

```javascript
LabelPropagation(edges[from, to, weight?], undirected: false, max_iter: 10)
```

在上文中，我们已经讨论了如何理解这些图算法定义，所以此处我们构造一个简单的查询(直接查询返回的行数过多，所以此处使用 `limit` 限制返回数):

```
transfer[from, to, weight] := *ERC20trans{txFrom: from, txTo: to, txValue: weight};
?[] <~ LabelPropagation(transfer[]);
:limit 5
```

但是此查询产生的结果较为简单，我们无法进行一些有效的分析，所以我们使用以下复杂检索来补充地址标签:

```
transfer[from, to, weight] := *ERC20trans{txFrom: from, txTo: to, txValue: weight};
addressGroup[groupId, address] <~ LabelPropagation(transfer[]);
?[groupId, suffixAddress, tags] := 
    addressGroup[groupId, notSuffixAddress],
    suffixAddress = concat("0x", notSuffixAddress),
    *etherscanTags{address: suffixAddress, name: tags},
    tags != ""
```

与上文的简单检索类似，此处最重要的是与 `etherscanTags` 表格进行了合并查询。由于 `ERC20trans` 中的地址没有 `0x` 前缀，而 `etherscanTags` 中的地址包含 `0x` 前缀，所以此处使用了 `concat` 拼接字符串。我们截取部分结果，如:

```
 4356    | "0x000000000005af2ddc1a93a03e9b7014064d3b8d" | "MEV Bot: 0x00...b8D"
 4356    | "0x00000000500e2fece27a7600435d0c48d64e0c00" | "MEV Bot: 0x000...C00"
 4356    | "0x03f7724180aa6b939894b5ca4314783b0b36b329" | "Shiba Inu: ShibaSwap"
 4356    | "0xe8c060f8052e07423f71d445277c61ac5138a2e5" | "MEV Bot: 0xE8c...2e5"
```
可以发现标签传播算法成功发现了 `MEV Bot` 的社群关系，此处出现 `Shiba Inu: ShibaSwap` 的原因可能是 `Shib` 等 `meme` 币单次转账数额巨大，赋权过高导致的。读者可以通过调整权重(比如使用归一化的 `txValue`) 等方法获得更好的社群发现。

当然，以下社群也是值得关注的:

```
 3902    | "0x05767d9ef41dc40689678ffca0608878fb3de906" | "SushiSwap: CVX"
 3902    | "0x130f4322e5838463ee460d5854f5d472cfc8f253" | "SushiSwap: APE"
 3902    | "0x1bec4db6c3bc499f3dbf289f5499c30d541fec97" | "SushiSwap: MANA"
 3902    | "0x53162d78dca413d9e28cf62799d17a9e278b60e8" | "SushiSwap: APW"
 3902    | "0x7ee3be9a82f051401ca028db1825ac2640884d0a" | "SushiSwap: DAI-SUSHI"
 3902    | "0x7f8f7dd53d1f3ac1052565e3ff451d7fe666a311" | "SushiSwap: MATIC"
 3902    | "0xc40d16476380e4037e6b1a2594caf6a6cc8da967" | "SushiSwap: LINK"
 3902    | "0xe592427a0aece92de3edee1f18e0157c05861564" | "Uniswap V3: Router"
```

该社区成功聚合了各大 DEX 尤其是 SushiSwap 的各个地址。但此处为什么会混入 `Uniswap V3` 的原因不明。读者可以考虑输出全社群地址集合(即包含未包含标签的集合)进行进一步研究。这正是社群发现算法的一大用途，将大规模图数据集进行分解以是我们可以聚集于子图中。

读者可以记得上文我们分析了 `56178a0d5f301baf6cf3e1cd53d9863437345bf9` 与 `fa103c21ea2df71dfb92b0652f8b1d795e51cdef` 为同一实控人，我们可以查看在两者是否被社群发现算法分为同一类，为保证后期查询，我们首先创建一个表格存储社群发现算法结果，如下:

```
transfer[from, to, weight] := *ERC20trans{txFrom: from, txTo: to, txValue: weight};
?[groupId, address] <~ LabelPropagation(transfer[]);
:create addressCluster {
    groupId: Int,
    address: String
}
```

`:create` 用于创建数据库，该命令直接使用上文 `?[groupId, address]` 的查询结果。

运行以下算法查询结果:

```
?[groupId, address] :=
    *addressCluster{groupId, address},
    address in ["56178a0d5f301baf6cf3e1cd53d9863437345bf9", "fa103c21ea2df71dfb92b0652f8b1d795e51cdef"]
```

返回如下:

```
 groupId | address
---------+--------------------------------------------
 3048    | "56178a0d5f301baf6cf3e1cd53d9863437345bf9"
 3048    | "fa103c21ea2df71dfb92b0652f8b1d795e51cdef"
```

社群发现算法认为这两个地址属于同一实控人。我们可以查看 `3048` 群体内还有哪些地址:

```
allAddress[groupId, suffixAddress, tag] := 
    *addressCluster[groupId, notSuffixAddress],
    suffixAddress = concat("0x", notSuffixAddress),
    tag = "",
    groupId = 3048;
tagAddress[groupId, suffixAddress, tag] :=
    allAddress[groupId, suffixAddress, _],
    *etherscanTags{address: suffixAddress, name: tag};
allMinusTagAddress[groupId, address, tag] :=
    allAddress[groupId, address, tag],
    not tagAddress[groupId, address, _];
?[groupId, address, tag] :=
    allMinusTagAddress[groupId, address, tag] or
    tagAddress[groupId, address, tag];
```

此处为了实现外连接(`Outer join`) 的逻辑，我们使用了大量检索，接下来，我们会逐一进行分析。首先读者需要明确相对于 `SQL` 中的高度抽象化的布尔代数，`cozoscript` 的抽象程度较低。简单来说，我们键入的每一条命令都是一个抽象程度不高的布尔代数式，这些式子构成了一个过滤器对数据库进行过滤。所以此处如果实现外连接，我们必须知道外连接的布尔表达式，如下:

```
(Table_A ∩ Table_B) + (Table_A - Table_B)
```

设此处的 `Table_A` 为检索出的不带有标签的所有属于 `groupId = 3048` 的数据，我们将其命名为 `allAddress` ，而 `Table B` 为包含标签的所以属于 `groupId = 3048` 的数据，我们将其命名为 `tagAddress` 。关于这两个表的具体构造过程，上文代码已经给出了分析。使用上述定义替换布尔表达式内的变量，得到:

```
(allAddress ∩ tagAddress) + (allAddress - tagAddress)
```

我们发现 `tagAddress` 内的数据都在 `allAddress` 内出现过，即 `tagAddress ⊂ allAddress`，那么上述布尔表达式可以改写为:

```
allAddress ∩ tagAddress = tagAddress
tagAddress + (allAddress - tagAddress)
```

接下来，我们着重构造 `allAddress - tagAddress` 使用 `not` 操作符即可，即 `not tagAddress[groupId, address, _]` ，此处使用 `_` 表示 `tag` 列被忽略，不参与最终的代数运算。最后构造出的表格为 `allMinusTagAddress` 。

> `_` 的含义简单来说就是该行可以在操作中被忽略。为什么我们在上文没有使用过此操作符？原因是上文使用的 `{}` 表格可以进行任意列的提取，而此处使用的 `[]` 表格必须给出每一个列绑定的变量名，具体请参考 [文档](https://docs.cozodb.org/zh_CN/latest/queries.html#id3)。

最后实现 `+` 加法，使用 `and` 操作符即可。最终，我们构造出了上文给出的复杂查询。

> 此处应感谢 [teradata 文档](https://docs.teradata.com/r/2_MC9vCtAJRlKle2Rpb0mA/waN6ZnVI~AtIQ~KDNcGtDA) 中给出的外连接的关系代数。在该数据库文档内也包含了一些其他 SQL 检索的布尔代数表达式，如果读者有兴趣可以简单阅读。

返回值如下:

```
 groupId | address                                      | tag
---------+----------------------------------------------+---------------------------------
 3048    | "0x0087bb802d9c0e343f00510000729031ce00bf27" | ""
 3048    | "0x1ae3b4cee159c2a75190d2f89d7fca249c5dad03" | ""
 3048    | "0x1d1661cb61bf5e3066f17f82099786d0fcc49d46" | ""
 3048    | "0x24ee2c6b9597f035088cda8575e9d5e15a84b9df" | ""
 3048    | "0x2f62f2b4c5fcd7570a709dec05d68ea19c82a9ec" | ""
 3048    | "0x31503dcb60119a812fee820bb7042752019f2355" | "SushiSwap: COMP"
 3048    | "0x4585fe77225b41b697c938b018e2ac67ac5a20c0" | ""
 3048    | "0x4a137fd5e7a256ef08a7de531a17d0be0cc7b6b6" | "MEV Bot: 0x4a1...6b6"
 3048    | "0x54afce9e356878190fd6dd1968979477fa73db74" | ""
 3048    | "0x56178a0d5f301baf6cf3e1cd53d9863437345bf9" | ""
 3048    | "0x5991190a8069fbb62da5b91a219e23bab2781282" | ""
 3048    | "0x60594a405d53811d3bc4766596efd80fd545a270" | ""
 3048    | "0x632e675672f2657f227da8d9bb3fe9177838e726" | ""
 3048    | "0x713d1e08d077281ba20ece3bde5c522dcf7c5a26" | ""
 3048    | "0x789c8ab61f73b2dc3a45dec67b63e46de1666503" | ""
 3048    | "0x82c427adfdf2d245ec51d8046b41c4ee87f0d29c" | ""
 3048    | "0x8c1c499b1796d7f3c2521ac37186b52de024e58c" | ""
 3048    | "0x8d90113a1e286a5ab3e496fbd1853f265e5913c6" | "Tokenlon: PMM"
 3048    | "0x8dbb0b95e7966849ec8e2711280b313b0d9f234c" | ""
 3048    | "0x92560c178ce069cc014138ed3c2f5221ba71f58a" | ""
 3048    | "0x9799b475dec92bd99bbdd943013325c36157f383" | "Fund: 0x979...383"
 3048    | "0xa57bd00134b2850b2a1c55860c9e9ea100fdd6cf" | "MEV Bot: 0xa57...6CF"
 3048    | "0xa6cc3c2531fdaa6ae1a3ca84c2855806728693e8" | ""
 3048    | "0xbae401d98c797c7955c751efcb5cea0ddd548e2d" | ""
 3048    | "0xc558f600b34a5f69dd2f0d06cb8a88d829b7420a" | "SushiSwap: LDO"
 3048    | "0xc6093fd9cc143f9f058938868b2df2daf9a91d28" | ""
 3048    | "0xc697051d1c6296c24ae3bcef39aca743861d9a81" | ""
 3048    | "0xd51a44d3fae010294c616388b506acda1bfaae46" | "Curve.fi: USDT/WBTC/WETH Pool"
 3048    | "0xd8de6af55f618a7bc69835d55ddc6582220c36c0" | ""
 3048    | "0xdbc8d2ffc94da7a6c33f34faac8c4e6dd3730f2d" | ""
 3048    | "0xf4ad61db72f114be877e87d62dc5e7bd52df4d9b" | ""
 3048    | "0xf8a95b2409c27678a6d18d950c5d913d5c38ab03" | ""
 3048    | "0xf8e3e331676e47d679ed5c3973c8a49d679bb5a0" | ""
 3048    | "0xfa103c21ea2df71dfb92b0652f8b1d795e51cdef" | ""
 3048    | "0xfda0097b9830f85df9a64ef695f784ecdaf9c53b" | ""
```

读者可以尝试进行可视化来观察这些地址之间的关系，并进行进一步分析。

#### 中心性算法

中心性算法主要勇于寻找图数据集中最重要的节点。涉及以下话题时，我们可能需要中心性算法:

1. DeFi 项目热度分析
1. 区块链中最重要的节点

下表展示了常见的中心性算法:

| 算法类型 | 作用 | 函数 |
| ------- | --- | --- |
| 度中心性算法 | 度量节点拥有的关系数量 | `DegreeCentrality` |
| 接近中心性算法 | 计算哪些节点具有到其他各节点的最短路径 | `ClosenessCentrality` |
| PageRank 算法 | 通过与当前节点相连的节点以及它们的邻节点来估计当前节点的重要性 | `PageRank` |
| 中间中心性算法 | 度量通过某个节点的最短路径的数量 | `BetweennessCentrality` |

> 中间中心性算法复杂度极高，请勿在大规模图数据集上使用，可以先使用上文的社群发现算法分割图数据集，然后使用此算法

也可以使用下图总结:

![Graph Centeral](https://img.gejiba.com/images/6512ee20a9442ea48df83ee5ac9c9824.png)

> 这些内容均来自 《数据分析之图算法：基于Spark和Neo4j》 第五章，详细内容请阅读此书

我们在此处仅展示 `DegreeCentrality` 的用法:

```
transfer[from, to] := *ERC20trans{txFrom: from, txTo: to};
?[address, allLink, toLink, fromLink] <~ DegreeCentrality(transfer[]);
:order -allLink
:limit 5

```

返回结果如下:

```
 address                                    | allLink | toLink | fromLink
--------------------------------------------+---------+--------+----------
 "a5025faba6e70b84f74e9b1113e5f7f4e7f4859f" | 782     | 781    | 1
 "28c6c06298d514db089934071355e5743bf21d60" | 416     | 83     | 333
 "a9d1e08c7793af67e9d92fe308d5697fb81d3e43" | 342     | 204    | 138
 "0000000000000000000000000000000000000000" | 306     | 199    | 107
 "74de5d4fcbf63e00296fd95d33236b9794016631" | 252     | 126    | 126
```

读者可以自行探索其他中心性算法的用法和作用。

## 总结

本文简单介绍了如何使用 `CosmosGraph` 进行大规模图数据可视化和使用 `cozodb` 进行图数据分析的相关内容，介绍了如何使用 `cozodb` 进行图数据检索和运行图算法以获得需要的结果。

本文没有详细介绍图算法的相关内容，读者可自行参考 《数据分析之图算法：基于Spark和Neo4j》 和 《巴拉巴西网络科学》等书籍研究图算法的背景知识和数学原理。本文没有涉及到 `cozodb` 的数据备份问题，而全程使用了 `mem` 内存数据引擎，该数据引擎虽然可以快速进行图数据分析，但一旦关闭就意味着数据丢失，读者可以自行研究如何使用其他数据引擎实现数据备份。

本文尽可能使用通俗的语言解释了 `cozoscript` 的相关内容，相信对于读者理解 `cozodb` 的文档是有帮助的。`cozodb` 的文档中大量使用了专有名词，理解难度较高，希望此文可以降低读者使用 `cozodb` 的门槛。
