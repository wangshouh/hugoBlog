---
title: "Clickhouse 以太坊分析:基础交易数据清洗"
date: 2023-01-21T11:47:33Z
tags: [Data,Clickhouse]
---

## 概述

笔者最近遇到了许多关于数据分析的文章，大部分都使用了 [Dune](https://dune.com/browse/dashboards) 等 `SaaS` 工具，这些工具往往提供了清洗后的区块链数据和数据库分析工具。对于大部分数据分析师而言，这些工具可以应对一系列复杂的数据分析问题，而且免去了搭建数据处理平台的苦恼。

但作为一个爱折腾的工程师，我决定几乎从零开始搭建一套区块链历史数据数据分析系统。在此项目中，我们仅使用了 `0xfast` 作为数据提供商，如果读者追求纯粹的自建，可以考虑自建以太坊节点抓取历史数据。

为尽可能提高分析系统的效率，我使用了 [Clickhouse](https://clickhouse.com/) 作为数据存储和分析工具。本文仅使用 `Clickhouse` 以完成以下工作：

1. 环境搭建。讨论在服务器内安装 `Clickhouse` 可能会遇到的问题并在个人计算机上安装客户端。
1. 数据导入。使用纯粹的 SQL 代码导入数据，此处会涉及到使用 `Clickhouse` 进行 JSON 处理的内容。
1. 数据分析。完成一些基础的数据分析任务，这并不是本文的核心。

注意，`Clickhouse` 对服务器似乎有着严格的性能要求，但由于本文分析的样本仅为 100 个区块，笔者使用 3 核心 3.5 GB 内存的服务器已经可以完成本博客中的任务。VPS负载如下：

![VPS Load](https://img.gopic.xyz/0a3349d10e72b77bf04dbbc61fa0d8a5.png)

当然，读者也可以考虑使用个人计算机完成此博客中的内容。如果涉及以太坊全区块链数据分析，建议使用高规格服务器，毕竟以太坊全节点数据集近 650GB。

## 环境搭建

### 服务端

如果你是一个时间价值比较高的工程师，且不想折腾`clickhouse`的安装，你可以选择 [Clickhouse Cloud](https://clickhouse.com/cloud)，似乎有一个月的试用期，价格数据如下：

![Clickhouse Cloud](https://img.gopic.xyz/bddd57d076b5f2665aa019902b15e530.png)

如果你选择此方案，就无需阅读本节。

以下介绍如何在服务器上安装`Clickhouse`，本部分主要参考了 [Clickhouse install](https://clickhouse.com/docs/en/install)，同时本文也会补充一些部分错误的修复方法。

> 由于笔者并不是一边搭建一边编写的博客，错误的截图并未给出，如果读者遇到错误，可在本文的评论区内留下截图。

安装`Clickhouse`相当简单，步骤如下：

1. 下载脚本并执行
    ```bash
    curl https://clickhouse.com/ | sh
    ```
1. 执行安装程序
    ```bash
    sudo ./clickhouse install
    ```
    执行此安装程序过程中，请耐心等待，在前期安装过程中，不会有任何输出提示。
1. 在安装过程的最后，会要求输入默认密码，建议键入复杂密码：
    ```bash
    Creating log directory /var/log/clickhouse-server.
    Creating data directory /var/lib/clickhouse.
    Creating pid directory /var/run/clickhouse-server.
    chown -R clickhouse:clickhouse '/var/log/clickhouse-server'
    chown -R clickhouse:clickhouse '/var/run/clickhouse-server'
    chown  clickhouse:clickhouse '/var/lib/clickhouse'
    Enter password for default user:
    ```
    读者可能已经发现此处使用了 `clickhouse:clickhouse` 用户进行了一系列授权操作，而不是`root`用户，所以进行应用操作时请使用`clickhouse`用户
1. 启动`clickhouse`:
    ```bash
    sudo clickhouse start
    ```
1. 校验状态:
    ```bash
    sudo clickhouse status
    ```
    合理的输出如下:
    ```bash
    /var/run/clickhouse-server/clickhouse-server.pid file exists and contains pid = 10818.
    The process with pid = 10818 is running.
    ```
    如果您的输出格式与之不符，就说明启动失败。

一旦启动失败，我们就要进入痛苦的`debug`阶段，读者可以使用以下命令启动`clickhouse`:
```bash
sudo clickhouse server --config-file /etc/clickhouse-server/config.xml
```

> 此处我们指定了启动配置文件`/etc/clickhouse-server/config.xml`，否则会使用默认文件，造成启动失败的问题。

此命令会将`log`输出到终端，读者可以自行查看进行`debug`，大部分问题都出现在用户权限配置问题上。

下面列出一个仅用于学习的配置方法，为方便使用，我直接将`root`用户置为`clickhouse`的管理者，并手动启动服务器，相关命令如下：

1. 配置权限：
    ```bash
    sudo chown -R root /var/lib/clickhouse /var/log/clickhouse-server /etc/clickhouse-server
    ```
1. 手动启动服务器:
    ```bash
    sudo /usr/bin/clickhouse-server --config-file /etc/clickhouse-server/config.xml --pid-file /var/run/clickhouse-server/clickhouse-server.pid --daemon
    ```

> 如果读者遇到其他问题，请直接放在评论区，或加入我的 [telegram 讨论群](https://web3l.t.me)

最后，请放行相关端口，此过程需要配置系统防火墙，读者请根据自身系统来进行配置。读者可尝试开启 `http://服务器IP:配置端口/play` 查看是否可以配置正确，若配置正确且放行端口正确，会出现如下显示:

![Clickhouse play](https://img.gopic.xyz/06656248e4d8c9cb1c46054fd0f0bfa6.png)

### 客户端

使用一个优秀的本地`SQL`客户端会大幅提高工作效率，由于不想在服务端安装一套`clickhouse`，所以我选择了其他的`sql`客户端，读者可以查看自己常用的 SQL 客户端是否支持 `clickhouse`。在本博客中，我使用了 [dbeaver](https://dbeaver.io/download/) 作为客户端，原因在于:

1. 社区版开源且免费
1. 压缩后体积仅为 110 MB，较为小巧
1. 主流操作系统均支持

读者可以根据自己的系统选择下载对应的版本。

关于 `dbeaver` 与 `clickhouse` 服务器的链接问题，由于`clickhouse`的 [官方文档](https://clickhouse.com/docs/en/integrations/sql-clients/dbeaver) 提供了图文并茂的教程，此处不再赘述。

## 数据导入

### 数据来源

此处我们使用了`0xfast`提供的相关服务，读者可提供我写的 [智能合约开发效率工具](https://blog.wssh.trade/posts/smart-contract-tool#区块数据获取) 了解此服务。

简单来说，我们可以提供以下链接获取数据:

1. https://eth-uswest.0xfast.com/stream/free?range=latest 获取最新区块数据
1. https://eth-uswest.0xfast.com/stream/free?range=1 获取单一区块数据
1. https://eth-uswest.0xfast.com/stream/free?range=1-10 获取系列区块数据

我们需要观察数据的具体结构，读者可在任一浏览器内打开`https://eth-uswest.0xfast.com/stream/free?range=latest`链接，返回值如下:

![0xfast json](https://img.gopic.xyz/1364f86d9cd75ada0a6b9858f67d9fff.png)

我们会经常查阅此`json`数据以确定数据导入的格式等。

### 基础导入

使用以下命令创建临时表格:
```sql
CREATE TABLE jsonTemp
(
	field String
)
ENGINE = Memory
```

导入基础数据并将`json`列表中的每一项解析为一个`json`行：
```sql
INSERT INTO jsonTemp 
SELECT * FROM url('https://eth-uswest.0xfast.com/stream/free?range=16448580-16448680', 'JSONAsString', 'field String');
```
此处使用了`clickhouse`可以直接使用`url`作为数据来源的方法，其中`JSONAsString`可以初步拆分形如`[{"a": "b"}, {"a": "c"}]`，将每一项列为表格中的一行。读者可查阅[相关文档](https://clickhouse.com/docs/en/sql-reference/formats/#jsonasstring)了解使用。

最后一项为表结构，事实上，我们将`url`提供的数据视为一张表，最后的`field String`用于规定表结构，具体请参考[文档](https://clickhouse.com/docs/en/sql-reference/table-functions/url/)

键入以下`SQL`进行查询:
```sql
SELECT * FROM jsonTemp jt  LIMIT 5;
```
获得的结果如下图:

![JsonTemp Limit](https://img.gopic.xyz/6bd5965caf494240d5d97eee4f6f9205.png)

我们发现每一个区块的数据变成了数据库中的一行，这对于我们后期处理是极为重要的。

### 清洗数据

将数据导入完成后，我们需要对数据进行进一步清洗，这涉及到`JSON`提取问题，在这一方面的资料一直较少，笔者认为目前分析最好的文章是[JSONExtract to parse many attributes at a time](https://kb.altinity.com/altinity-kb-queries-and-syntax/jsonextract-to-parse-many-attributes-at-a-time)，虽然此文章仅给出了一个示例，但对于笔者有很大的启发作用。

目前，`clickhouse`已支持`JSON`数据类型，但由于我们的目标是提取出`transactions`交易列表并进行进一步分析，所以`JSON`格式并不适用当前方案。对于`clickhouse`的`JSON`数据结构的分析，读者可以参考[ClickHouse Newsletter April 2022: JSON, JSON, JSON](https://clickhouse.com/blog/clickhouse-newsletter-april-2022-json-json-json)

继续我们的项目，为方便后文的讨论，我们首先进行一些简单的`JSON`检索。首先，我们尝试获得区块数据中的`number`和`gasUsed`:

```sql
WITH JSONExtract(
	field, 
	'Tuple(number String, gasUsed String)'
) AS parsed_json
SELECT 
	tupleElement(parsed_json, 'number') as blockNumber,
	tupleElement(parsed_json, 'gasUsed') as gasUsed
FROM jsonTemp jt LIMIT 5;
```
输出结果如下:
```
blockNumber|gasUsed  |
-----------+---------+
0xfafc44   |0xcd7e49 |
0xfafc45   |0xf3bf2  |
0xfafc46   |0x1c98c36|
0xfafc47   |0xce5243 |
0xfafc48   |0x156b2a8|
```

其中的核心在于我们构造的`Tuple(number String, gasUsed String)`字符串，该字符串指示我们需要提取的部分及其数据类型。接下来，我们尝试交易部分，请读者注意构造的提取字符串:

```sql
WITH JSONExtract(
	field, 
	'Tuple(transactions Nested(value String, type String))'
) AS parsed_json
SELECT 
	tupleElement(parsed_json, 'transactions') as tx
FROM jsonTemp jt LIMIT 5;
```

我们可以使用`Nested`指示提取列表内的数据，对于很多使用`Juila`等编程语言进行数据分析的程序员可能对其很熟悉。返回值如下:

![Nested](https://img.gopic.xyz/f1d2eac4005140f4e9bc8ba6a907d96c.png)

返回值为多层列表嵌套，如下:
```
['[0x0, 0x0]','[0x1fef237e4e8800, 0x0]','[0x62f1b62f7235000, 0x0]' ...]
['[0x78ff1d2e7a12d, 0x0]','[0x0, 0x2]','[0x0, 0x2]','[0x5b2413b88e9400, 0x0]' ...]
['[0x14d809f18123, 0x2]','[0x68f59ce9e20fd57, 0x2]','[0x1643684073ce, 0x2]' ...]
['[0xb4e5452324, 0x2]','[0x7c9e43a16382df2, 0x0]','[0xc0a2b45ba0, 0x2]' ... ]
['[0x0, 0x2]','[0x2353376319bb8968, 0x2]','[0x85d5077a8c000, 0x0]' ...]
```
每行为一个区块，内部为嵌套的提取结果。我们希望其中的每一个交易数据转化为数据库中的一行。在`clickhouse`中，存在一个函数 [arrayJoin](https://clickhouse.com/docs/en/sql-reference/functions/array-join/) 可以解决这一问题。修改提取代码，如下:

```sql
WITH JSONExtract(
	field, 
	'Tuple(transactions Nested(value String, type String))'
) AS parsed_json
SELECT 
	arrayJoin(tupleElement(parsed_json, 'transactions')) as tx
FROM jsonTemp jt LIMIT 5;
```

返回值如下:

```
tx                       |
-------------------------+
[0x0, 0x0]               |
[0x1fef237e4e8800, 0x0]  |
[0x62f1b62f7235000, 0x0] |
[0x1943c033140be000, 0x0]|
[0x1f10c26adf13df8, 0x0] |
```

我们成功将交易数据分解成立数据库中的一行，虽然现在仍属于嵌套列表的形式。使用另一个函数 [untuple](https://clickhouse.com/docs/en/sql-reference/functions/tuple-functions/#untuple) 可以解决这一问题，`sql`代码如下:

```sql
WITH JSONExtract(
	field, 
	'Tuple(transactions Nested(value String, type String))'
) AS parsed_json
SELECT 
	untuple(arrayJoin(tupleElement(parsed_json, 'transactions'))) as tx
FROM jsonTemp jt LIMIT 5;
```

返回值如下:
```
tx.1              |tx.2|
------------------+----+
0x0               |0x0 |
0x1fef237e4e8800  |0x0 |
0x62f1b62f7235000 |0x0 |
0x1943c033140be000|0x0 |
0x1f10c26adf13df8 |0x0 |
```

这与我们的目标的表结构基本一致。接下来，我们编写所需要数据的完整`sql`代码，此处提取的数据是我所需要的，读者可根据自身需求自行更改:

```sql
WITH JSONExtract(
	field, 
	'Tuple(transactions Nested(blockNumber String, value String, type String, gasUsed String, from Tuple(address String), to Tuple(address String)))'
) AS parsed_json
SELECT 
	untuple(arrayJoin(tupleElement(parsed_json, 'transactions'))) as tx
FROM jsonTemp jt LIMIT 5;
```

此处我们在提取字符串内部`Nested`再次嵌套了一个`tuple`以处理以下字段:
```json
{
    ...
    "from": {
        "@type": "Account",
        "address": "0x30741289523c2e4d2a62c7d6722686d14e723851"
    },
    "to": {
        "@type": "Account",
        "address": "0x525a8f6f3ba4752868cde25164382bfbae3990e1"
    }
    ...
}
```
请读者注意，此数据也会影响我们后期搭建数据库。

运行上述代码后，返回值如下:

```
tx.1    |tx.2              |tx.3|tx.4   |tx.5         |tx.6         |
--------+------------------+----+-------+-------------+-------------+
0xfafc44|0x0               |0x0 |0x1f10c|[0x6001a7...]|[0x255538...]|
0xfafc44|0x1fef237e4e8800  |0x0 |0x5208 |[0x05cdb1...]|[0xa20f70...]|
0xfafc44|0x62f1b62f7235000 |0x0 |0x5208 |[0x75e89d...]|[0xbc514e...]|
0xfafc44|0x1943c033140be000|0x0 |0x5208 |[0x75e89d...]|[0x680f68...]|
0xfafc44|0x1f10c26adf13df8 |0x0 |0x5208 |[0xb08cd4...]|[0x0e747e...]|
```

此处`tx.5`(即`from`)和`tx.6`(即`to`)都属于`tuple`类型，这会影响我们创建数据库。

### 创建数据库

在`clickhouse`中允许我们创建含有名称的`tuple`，这正好用于处理上文的 `tx.5` 和 `tx.6` 列。

数据库创建代码如下:
```sql
CREATE TABLE tx 
(
	txblockNumber String,
	txValue String,
	type Enum8('0x0' = 0, '0x1' = 1, '0x2' = 2),
	gasUsed String,
	txFrom Tuple 
	(
		address String
	),
	txTo Tuple 
	(
		address String
	)
)
ENGINE = MergeTree
ORDER BY txblockNumber
```

由于考虑长时间存储数据和检索效率，我们使用了`MergeTree`数据引擎，并将交易类型`type`设置为`Enum8`以方便检索。我们可以看到`txFrom`和`txTo`均为`tuple`类型以方便数据输入和分析。

> 使用`tuple`与`JSON`在功能上基本类似，但前者要求严格的数据顺序，后者则没有要求，在本项目中，使用`tuple`是合适的。

插入数据的`sql`代码:

```sql
INSERT INTO tx
WITH JSONExtract(
	field, 
	'Tuple(transactions Nested(blockNumber String, value String, type String, gasUsed String, from Tuple(address String), to Tuple(address String)))'
) AS parsed_json
SELECT 
	untuple(arrayJoin(tupleElement(parsed_json, 'transactions'))) as tx
FROM jsonTemp jt;
```

结果如下:

![Insert Clickhouse](https://img.gopic.xyz/02da01387c3e8a1c3a6868700bb7d2de.png)

我们写入了 15774 行数据，但数据库仅占用了 1014 KB ，可见数据引擎对数据压缩的影响。

> 我们在此处仅对数据进行了简单清洗，我们会在此**系列文章**的后期，增加一些更加复杂的数据抽取和分析

## 数据分析

由于笔者暂时不知道分析那些内容，本节仅讨论以下问题:

1. 交易的`type`占比情况
1. 各区块的交易数量受`gas`的影响情况
1. 接受和发起交易最多的地址
1. 交易平均转移 ETH 数量

### 交易类型占比

这是一个比较简单的任务，我们使用的代码如下:

```sql
SELECT 
	COUNT()
FROM tx t 
GROUP BY `type` 
```

输出如下:

```
count()|
-------+
   2306|
     17|
  13451|
```

显然，目前大部分用户都在使用`type 3`类型的交易，即符合`EIP1559`类型的交易。

### 交易数量影响

在 [以太坊机制详解:Gas Price计算](https://blog.wssh.trade/posts/ethereum-gas#base-fee) 中，我们介绍了`baseFee`对交易数量的影响并绘制了简单的图像。简单来说，区块内交易数量会在某均值范围内波动，简单来说，就是当前区块交易数量较多时，以太坊会提高交易手续费以抑制下一区块内的交易。在本项目中，我们会再次验证这一事实。

使用的`sql`代码如下:

```sql
SELECT 
	txblockNumber,
	bar(COUNT(), 0, 1250),
	SUM(reinterpretAsUInt128(reverse(unhex(`gasUsed`))))
FROM tx
GROUP BY txblockNumber;
```

输出结果为:

![Clickhouse Bar](https://img.gopic.xyz/6ba19311b6f8a62cba2ad4f7f495ceb3.png)

其中的`bar`用于绘制用于绘制柱状图，其参数格式为`bar(x, min, max, width)`，此处我们省略了`width`参数。此处我们也使用了``reinterpretAsUInt128(reverse(unhex(`gasUsed`)))``来转化 16 进制格式的 `gasUsed`，使用`UInt128`是为了防止数据溢出。

`GROUP BY`的用法对于数据分析师而言是一个极为熟悉的函数，此处不再赘述。

显然，以太坊基于交易手续费的策略有效避免了长时间区块链拥堵的问题出现。

### 交易发起和接受

这其实也是一个较为简单的问题，我们使用以下代码获取交易接受方次数最大的地址:

```SQL
SELECT 
	COUNT(),
	txTo.address
FROM tx t 
GROUP BY txTo.address 
ORDER BY COUNT() DESC
LIMIT 5
```
此处使用`ORDER BY`进行数据排序，其中`DESC`代表降序而`ASC`代表升序。

返回值如下:

```
count()|txTo.address                              |
-------+------------------------------------------+
   1747|0xdac17f958d2ee523a2206206994597c13d831ec7|
    589|0xef1c6e67703c7bd7107eed8303fbe6ec2554bf6b|
    540|0x00000000006c3852cbef3e08e8df289169ede581|
    485|0xc24f7c14ec350a7dbcc08673aeef4e87c726fe11|
    481|0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48|
```

使用 [Etherscan](https://etherscan.io) 可以查询到这些地址的标签，交互最多的合约`0xdac17f958d2ee523a2206206994597c13d831ec7`其实是`USDT`的代币合约。此处，我们没有详细讨论交易的其他细节，我们可能会在此系列中的下一篇文章内进行分析。

同理，我们可以获得发起交易最多的地址:

```sql
count()|txFrom.address                            |
-------+------------------------------------------+
    164|0x21a31ee1afc51d94c2efccaa2092ad1028285549|
    148|0xdfd5293d8e347dfe59e90efd55b2956a1343963d|
    138|0x28c6c06298d514db089934071355e5743bf21d60|
    131|0x46340b20830761efd32832a74d7169b29feb9758|
    116|0x56eddb7aa87536c09ccc2793473599fd21a8b17f|
```

发起交易最多的地址`0x21a31ee1afc51d94c2efccaa2092ad1028285549`隶属于`Binance`，用于链上提现。

### 交易平均价值

注意此处仅讨论了 ETH 转账的数量而不包含其他`ERC-20`代币，关于`ERC-20`代币及其他讨论，我们会在后文中讨论。

使用如下代码:

```sql
SELECT 
	AVG(reinterpretAsUInt128(reverse(unhex(txValue)))) as avgTxValue
FROM tx t 
```

此处使用了`AVG`函数以计算交易均值。

返回如下:

```
avgTxValue           |
---------------------+
546686958851271800000|
```

此返回值以`wei`为单位，转换为`ETH`单位为`546.6869588512718`，这个过大的数值可能受一些极端值影响，读者可以使用`Python`或`Juila`等工具进行归因分析。

## 拓展思路

此处介绍一些笔者脑洞打开的设想:

1. `Clickhouse`集群与`kafka`流处理拼接已实现高效率数据更新
1. `Clickhouse`与图数据、大规模机器学习门槛拼接已实现复杂基于链上数据的风控系统
1. 基于`Clickhouse`的区块链浏览器网站

这些设想可用作大数据专业的毕业设计。在本系列的后续文章中，我们会讨论`Clickhouse`与图数据库的组合以及尝试搭建基于`Clickhouse`的区块链浏览器。但由于笔者财力有限，对于`clickhouse`集群搭建等问题则无能为力。

## 总结

本文重在基础数据的清理，我们没有涉及一些`logs`等复杂数据的分析，关于一些更加复杂的情况，我们已在 [基于Python与GraphQL的链上数据分析实战](https://blog.wssh.trade/posts/on-chain-data-basement/) 进行过过一系列讨论，笔者也计划在本系列未来的文章讨论更加复杂的数据清洗和提取。

本文的一大遗憾是由于编写本文时笔者已完成了`Clickhouse`的安装，所以在本文中缺失了大量的安装截图，读者可通过评论区反馈您遇到的问题，我会尝试解决并将其纳入正文。

本文最杰出的贡献是完整阐述了在`Clickhouse`中使用纯`SQL`导入嵌套(Nested)`JSON`数据并分析的全流程，介绍此技术的文章在国内外都是缺少的，笔者也是经过数天摸索在完成了这一任务。

