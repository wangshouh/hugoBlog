---
title: 区块链大数据分析:导入 xatu 数据集
date: 2024-12-06T23:47:30Z
tags: [clickhouse]
---

## 概述

在笔者之前的几篇使用 Clickhouse 进行数据分析的文章内使用了 Clickhouse 作为数据分析工具，并采用了 ifast 作为数据源。但 ifast 在很久前就已经停止运营。而且依赖 ifast 并不是一个很好的选择。本文将摆脱对 ifast 数据源的依赖。但是我们依旧保留之前的文章，因为这些文章内的数据清洗工作具有较高价值。

而在本文内，我们使用了 [cryo](https://github.com/paradigmxyz/cryo) + clickhouse 的分析组合。cryo 是 paradigm 推出的使用 rust 抽取全节点数据的数据导出工具，目前基本上是以太坊数据导出的最重要的工具。而本文使用 Clickhouse 原因在于 Clickhouse 是目前性能最高的 OLAP 数据库之一。而且 Clickhouse 也被很多平台作为数据分析的核心工具。甚至 Clickhouse 官方也推出过用于展示目的的区块链分析平台，具体可以参考 [Announcing CryptoHouse: Free Blockchain Analytics powered by ClickHouse and Goldsky](https://clickhouse.com/blog/announcing-cryptohouse-free-blockchain-analytics)。

本文并没有从自建节点开始，正如标题所述，我们使用了 ethpandaops 使用 cryo 导出的数据集，具体可以参考 [Xatu Execution Layer data now available](https://ethpandaops.io/posts/xatu-execution-layer/)。

## 环境搭建

在本系列 [第一篇文章](https://blog.wssh.trade/posts/clickhouse-with-blockchain/) 内，笔者介绍了如果在服务器直接部署。但在本文内，我们准备使用 docker 进行部署。我们首先需要使用 `docker pull clickhouse` 下载相关镜像。读者可以使用以下方法创建项目所在的目录:

```bash
mkdir xatu
cd xatu
mkdir data
```

然后在 `xatu` 文件夹内使用以下方法编写 `compose.yaml`:

```yaml
version: '3'

services:
  clickhouse:
    image: clickhouse
    volumes:
      - ./data:/var/lib/clickhouse
    ports:
      - "9000:9000"
      - "8123:8123"
```

然后读者应该可以直接使用 `http://127.0.0.1:8123/play` 访问本地的 clickhouse 服务器。在上述配置内，我们没有配置密码等，读者需要注意在生产环境下的配置。

## 数据导入

### 数据来源

在最新的以太坊数据分析框架内，一般推荐使用 [cryo](https://github.com/paradigmxyz/cryo) 提取节点内的数据。在大部分情况下，我们都使用 `parquet` 作为导出数据集的优先格式。笔者手边并没有直接可用的全节点服务器，所以此处不在演示导出的方法。非常有幸的是，ethpandaops 开源了他们使用 cryo 导出的数据集，具体可以参考 [Xatu Execution Layer data now available](https://ethpandaops.io/posts/xatu-execution-layer/)。

简单来说，我们可以提供以下格式的链接获取数据：

```
https://data.ethpandaops.io/xatu/NETWORK/databases/default/TABLE_NAME/1000/CHUNK_NUMBER.parquet
```

上文内的 `NETWORK` 可以被替换为 `mainnet` \ `holesky` \ `sepolia` 。而 `TABLE_NAME` 则用很多选择，我们在本文内会用到以下几个:

1. `canonical_execution_transaction` 交易的基本数据
2. `canonical_execution_logs` 交易的日志数据

xatu 对 cryo 导出的数据集进行重命名，如果读者直接使用 cryo 的数据集请注意名称区别。而 `CHUNK_NUMBER` 则代表分块，如使用 `50000` 作为 `CHUNK_NUMBER`，则 xatu 会返回 `50000 - 50999` 的数据。

> 如果读者希望获得更多关于 xatu 数据集的 schema 信息，可以阅读 [此文档](https://ethpandaops.io/data/xatu/schema/canonical_execution_/)

### 数据导入

我们首先根据 [canonical_execution_block 文档](https://ethpandaops.io/data/xatu/schema/canonical_execution_/#canonical_execution_block) 编写创建表格的 SQL:

```sql
CREATE TABLE canonical_execution_block (
    updated_date_time DateTime,
    block_date_time DateTime64(3),
    block_number UInt64,
    block_hash FixedString(66),
    author Nullable(String),
    gas_used Nullable(UInt64),
    extra_data Nullable(String),
    extra_data_string Nullable(String),
    base_fee_per_gas Nullable(UInt64),
    meta_network_id Int32,
    meta_network_name LowCardinality(String)
) ENGINE = MergeTree
ORDER BY
    block_number
```

注意，此处在文档里的 `block_hash` 有错误，上述 SQL 代码内有所修复。并且我们对 `meta_network_name` 使用 `LowCardinality`  类型，该类型可以视为自动化调整的枚举类型。

在检索引擎方面，此处我们使用了 `MergeTree` 这种通用引擎。由于笔者并不是非常熟悉 Clickhouse 的调优策略，所以此处只是简单选择了一种通用引擎。如果您使用 Clickhouse 作为数据流的接受对象以便进行实时数据分析的话，官方建议使用 [ReplacingMergeTree](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/replacingmergetree) 以便在数据库层面保持数据的自动替换。

> 更多策略可以参考 [Announcing CryptoHouse: Free Blockchain Analytics powered by ClickHouse and Goldsky](https://clickhouse.com/blog/announcing-cryptohouse-free-blockchain-analytics) 一文。比较重要的策略包括使用 Null 数据引擎来快速存储数据，并使用物化视图(`Materialized Views`) 来进行数据的格式化入库

然后，读者可以使用以下 SQL 代码导入 xatu 提供的数据集:

```sql
INSERT INTO
    canonical_execution_block
SELECT
    *
FROM
    url(
        'https://data.ethpandaops.io/xatu/mainnet/databases/default/canonical_execution_block/1000/{16400..16450}000.parquet',
        'Parquet'
    )
```

上述代码导入了 `16400 000 - 16450 999` 范围内的 51000 个区块。我们可以使用以下`SQL`进行数据查询:

```sql
SELECT * FROM canonical_execution_block  LIMIT 5;
```
获得的结果如下图:

![Blocks Example](https://img.gopic.xyz/ClickhouseBlockSelect.png)

我们发现每一个区块的数据变成了数据库中的一行，这对于我们后期处理是极为重要的。而且由于 cryo 已经将数据进行了格式化，所以我们后期不需要进行进一步的数据处理。

接下来，我们尝试导入另一个重要的数据源，即交易日志数据源。如果读者不熟悉交易日志的作用，请自行阅读本系列的[第二篇文章](https://blog.wssh.trade/posts/clickhouse-eth-logs/)。

我们首先创建相关数据库:

```sql
CREATE TABLE canonical_execution_logs (
    updated_date_time DateTime,
    block_number UInt64,
    transaction_index UInt32,
    transaction_hash FixedString(66),
    internal_index UInt32,
    log_index UInt32,
    address FixedString(42),
    topic0 String,
    topic1 Nullable(String),
    topic2 Nullable(String),
    topic3 Nullable(String),
    meta_network_id Int32,
    meta_network_name LowCardinality(String)
) ENGINE = MergeTree
ORDER BY
    block_number
```

然后，我们可以使用以下命令下载一部分交易日志:

```sql
INSERT INTO
    canonical_execution_logs
SELECT
    *
FROM
    url(
        'https://data.ethpandaops.io/xatu/mainnet/databases/default/canonical_execution_logs/1000/16400000.parquet',
        'Parquet'
    )
```

在此处下载的 1000 个区块内包含 10244474 行日志数据，如果不建议读者使用 url 的方式一次性导入过多数据，一旦下载数据过程中出现网络问题就会回滚。使用 [crypto clickhouse](https://crypto.clickhouse.com/) 查询获得的结果为以太坊目前总计有 43 亿行日志数据。更加优雅的方式应该是将 `parquet` 文件下载到本地然后导入。关于交易日志的分析，我们在之前也有所介绍。在本文编写时，笔者仍未发现完全开源的事件签名数据集。

最后，我们介绍一个更加庞大的数据集，也是我们之前没有介绍过的数据集——以太坊交易 trace 数据集。在大部分情况下，读者都会使用到此数据集，使用此数据集大部分是为了进行交易的 debug 或者进行交易的深入分析。该数据集一般不用于数据分析。

我们还是首先介绍如何创建表格，读者可以使用以下 SQL 代码:

```sql
CREATE TABLE canonical_execution_traces (
    updated_date_time DateTime,
    block_number UInt64,
    transaction_index UInt32,
    transaction_hash FixedString(66),
    internal_index UInt32,
    action_from String,
    action_to Nullable(String),
    action_value String,
    action_gas UInt32,
    action_input Nullable(String),
    action_call_type LowCardinality(String),
    action_type LowCardinality(String),
    action_init Nullable(String),
    action_reward_type String,
    result_output Nullable(String),
    result_code Nullable(String),
    result_address Nullable(String),
    trace_address Nullable(String),
    subtraces UInt32,
    error Nullable(String),
    address FixedString(42),
    meta_network_id Int32,
    meta_network_name LowCardinality(String)
) ENGINE = MergeTree
ORDER BY
    block_number
```

然后使用以下代码从 xatu 数据集内下载并插入数据:

```sql
INSERT INTO
    canonical_execution_traces
SELECT
    *
FROM
    url(
        'https://data.ethpandaops.io/xatu/mainnet/databases/default/canonical_execution_traces/1000/16400000.parquet',
        'Parquet'
    )
```
最终，读者可以获得大约 76 万行数据。

![Clickhouse Trace Table](https://img.gopic.xyz/ClickhouseTraceTable.png)

我们可以看到上述数据集内包含 `action_call_type` 列，此列的内容有以下三种:

1. `call` 操作，常规的合约调用形式，用于调用其他合约触发其他合约数据变化
2. `delegate_call` 操作，一种特殊的合约调用形式，其目的是调用对方的代码来修改自己的状态，主要用于代理合约以及调用库等目的
3. `static_call` 操作，主要用于调用其他合约的 `view` 函数

我们可以看看那些合约被 `static_call` 调用的比较多：

```sql
SELECT
    action_to,
    count(action_to) AS times
FROM
    canonical_execution_traces
WHERE
    action_call_type = 'static_call'
GROUP BY
    action_to
ORDER BY
    times DESC;
```

结果如下:

![Clickhouse trace type](https://img.gopic.xyz/ClickhouseTraceType.png)

目前被 `static_call` 调用最多的是 `0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2`。该合约实际上就是 WETH 合约。直接上，我们感觉调用最多的方法应该是 `balanceOf` 方法，我们可以使用 SQL 检索一下：

```sql
SELECT
    substr(action_input, 1, 10) AS selector,
    count(selector) as times
FROM
    canonical_execution_traces
WHERE
    action_call_type = 'static_call'
    AND action_to = '0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2'
GROUP BY selector
```

最后的结论为:

| selector   | times |
| ---------- | ----- |
| 0x70a08231 | 41201 |
| 0xdd62ed3e | 363   |
| 0x313ce567 | 65    |
| 0x95d89b41 | 2     |

读者可以自行使用 `cast 4byte 0x70a08231` 等命令查询对应的函数定义。其中 `0x70a08231` 确实是 `balanceOf(address)` 函数。这是符合我们直觉的。

接下来，我们可以看看有哪些合约被 `delegate_call` 最多。因为目前 `delegate_call` 往往用于代理合约，我们可以通过查找被 `delegate_call` 最多的合约来获知在数据范围内最火的合约实现：

```sql
SELECT
    action_to,
    count(action_to) AS times
FROM
    canonical_execution_traces
WHERE
    action_call_type = 'delegate_call'
GROUP BY
    action_to
ORDER BY
    times DESC;
```

返回的前五行数据如下：

| action_to                                  | Times |
| ------------------------------------------ | ----- |
| 0xa2327a938febf5fec13bacfb16ae10ecbc4cbdcf | 27453 |
| 0x68b3465833fb72a70ecdf485e0e4c7bd8665fc45 | 12567 |
| 0x034031e261e00e1ae1f749724e4673ab30a7ca35 | 3386  |
| 0xd9db270c1b5e3bd161e8c8503c55ceabee709552 | 3052  |
| 0x6bf5ed59de0e19999d264746843ff931c0133090 | 2223  |

其中排名第一的 `0xa2327a938febf5fec13bacfb16ae10ecbc4cbdcf` 是一个代币实现合约。那么，我们希望知道这个代币合约被某一个工厂合约绑定造成的巨量交互还是单纯某一个代币导致了巨量交互。我们可以通过检索 `delegate_call` 的发起人解决此问题。我们可以使用以下 SQL 代码进行检索:

```sql
SELECT
    action_from,
    count(action_from) AS times
FROM
    canonical_execution_traces
WHERE
    action_call_type = 'delegate_call'
    AND action_to = '0xa2327a938febf5fec13bacfb16ae10ecbc4cbdcf'
GROUP BY
    action_from
ORDER BY
    times DESC;
```

结论如下:

| action_from                                | Times |
| ------------------------------------------ | ----- |
| 0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48 | 27446 |
| 0x1abaea1f7c830bd89acc67ec4af516284b1bc33c | 7     |

事实证明这是依靠一个单一合约调用了此实现实现了大量交互。那么假如我们希望知道哪一个合约被作为不同地址 `delegate_call` 的最多，我们该如何编写 SQL 代码?

```sql
SELECT
    action_to,
    uniq(action_from) AS caller_count
FROM
    canonical_execution_traces
WHERE
    action_call_type = 'delegate_call'
GROUP BY
    action_to
ORDER BY
    caller_count DESC;
```

此处使用了 `uniq` 聚合函数来进行数据聚合。最终的结果表明 `0xd9db270c1b5e3bd161e8c8503c55ceabee709552` 是被最多作为实现合约的地址。该合约是 Safe 多签钱包的实现合约，这也符合我们的直觉。

一般我们使用 trace 数据集都出于以下目的:

1. 希望拿到一些跨合约的信息数据，比如代理模型中最常用的实现合约
2. 拿到一些合约交互过程中的内部数据。某些合约有价值数据可能并没有对外抛出事件，此时就需要使用 trace 数据集根据 calldata 反向获得一些内部数据

## 总结

相比起本系列的前几篇文章，本篇文章可能内容稍有单薄，核心内容是介绍如何导入 xatu 的数据集，并补充了 trace 数据集的使用。本文缺失如何直接使用 cryo 从全节点抽取数据的内容，关于此部分内容，可能会在未来进行补充。(我的 homelab 硬盘不够跑全节点是最大阻碍)
