---
title: "加密货币衍生品研究:开放衍生品交易基本数据获取"
date: 2022-11-17T11:47:33Z
tags: [Finance,Data]
aliases: ["/2022/11/16/crypto-derivatives-data/"]
---
## 概述

最近笔者在写一篇关于加密货币方面的金融小论文，由于在写作过程中涉及到了部分加密货币衍生品，如期货、期权等，所以笔者研究了加密货币衍生品的交易数据获取。本文主要涉及以下衍生品的数据获取:

- 期货
- 期权

除此之外，本文也将介绍如何获取ETH现货价格数据。同时，本文会给出使用`python`进行数据清洗的相关步骤，方便读者使用数据集。

笔者是一个穷学生，所以本文介绍的方法都是通过开放API或数据库获取相关数据，如果读者可以支付每月 100 美元以上的费用可以考虑放弃阅读本文直接前往[Tardis](https://tardis.dev/)获取相关衍生品交易数据。

## ETH现货价格数据

虽然本文主要考虑加密货币衍生品相关内容，但衍生品是基于现货的，所以在讨论衍生品交易数据获取前，我们首先讨论ETH现货价格的获取。

ETH现货价格是可以非常方便的获得基础数据。在此处，我们给出 5 种获取方法。

### Ethereumprice 数据源

前往[ethereumprice](https://ethereumprice.org/history/)，读者可以下载ETH价格历史价格数据。

![EthereumPrice](https://img.gejiba.com/images/19d8c888a9d57c9b9246fb74050617b4.png)

从网站最大可下载 2015 年 8 月 7 日(August 7, 2015)至今的ETH价格历史数据，数据样例如下:

|timestamp |open              |high              |low               |close             |
|----------|------------------|------------------|------------------|------------------|
|1668556800|1251.8444409081906|undefined         |undefined         |undefined         |
|1668470400|1241.6217648171023|1286.153135295928 |1234.150703742326 |1251.8002966763943|
|1668384000|1219.7753518829538|1287.0985307186104|1175.2757062112164|1240.81844865221  |
|1668297600|1255.215288170864 |1272.0209432301174|1204.746716406467 |1220.0310931112967|

值得注意的是早期数据存在大量缺失，样例如下:

|timestamp |open              |high              |low               |close             |
|----------|------------------|------------------|------------------|------------------|
|1439078400|1.2               |undefined         |undefined         |undefined         |
|1438992000|1.2               |undefined         |undefined         |undefined         |
|1438905600|3                 |undefined         |undefined         |undefined         |

此数据源优点如下:

- 可获取数据范围广，是所有数据源内获取时间范围最大的
- 完全无门槛，不需要注册等流程直接下载相关数据

缺点如下:

- 数据存在重复，时常出现同日期出现两行数据且时间间隔仅为 1 分钟
- 早期时间存在大量缺失情况，事实上只有 2019 年 2 月 27 日后的收据完整

如果读者需要早期以太坊价格数据可以考虑使用此数据集。

使用`python`中的`pandas`对其进行清洗的相关代码如下:
```python
// 数据读取和时间序列转化
ethereum_price = pd.read_csv("data\ethereumPrice.csv", index_col="timestamp")
ethereum_price.index = pd.to_datetime(ethereum_price.index, unit="s")
//数据类型转变，此处将 undefined 转化为 nan
ethereum_price = ethereum_price.apply(lambda x: pd.to_numeric(x, errors="coerce"))
//保留旧数据
old_ethereum = ethereum_price.loc[: "2019-2-26"]
//删除 nan
ethereum_price.dropna(inplace=True)
//将新数据与旧数据拼接
ethereum_price = pd.concat([ethereum_price, old_ethereum])
//使用开盘价绘制图像
ethereum_price["open"].plot()
```

最终读者可以获得下图:
![EthPrice Plot](https://img.gejiba.com/images/d55396a6965c03f26162f7fbf8f52343.png)

### Investing Crypto

[Investing Crypto Data](https://www.investing.com/crypto/ethereum/historical-data)也提供了 ETH 历史价格数据下载服务，此网站也提供了获取 BTC 等加密货币现货历史价格的服务。

> 此方法获取历史数据需要登陆，读者可以直接通过 google 账户或其他登录方式登录，如果读者不希望注册或登录请跳过此方法。

![Investing Crypto Data](https://img.gejiba.com/images/ea3ea5a77e2cd6d21cb68d87cc830912.png)

该数据源可获得 2016 年 5 月 10 日至今的ETH价格数据，样例如下:

|Date        |Price   |Open    |High    |Low     |Vol.   |Change %|
|------------|--------|--------|--------|--------|-------|--------|
|Nov 16, 2022|1,258.92|1,253.45|1,267.62|1,248.88|636.07K|0.44%   |
|Nov 15, 2022|1,253.45|1,242.64|1,289.21|1,235.39|686.18K|0.87%   |
|Nov 14, 2022|1,242.64|1,221.47|1,289.01|1,175.42|1.01M  |1.73%   |

值得注意的是，上述所有列的数据类型均为字符(`str`)型。

此数据源优点如下:

- 数据较为全面，包含多个属性
- 涉及时间范围较广
- 无缺失值

缺点如下:

- 没有良好的数据结构类型，统一保存为`str`，清洗难度较大
- 数据来源未知，似乎为多家交易所现货价格的平均

如果读者较为在意数据来源，`investing`提供了各交易所的数据来源接口，读者可前往[ETH USD Historical Data](https://www.investing.com/crypto/ethereum/eth-usd-historical-data)网页，如下图:

![Eth Cex Source](https://img.gejiba.com/images/13fb113570cbc4ff734e3892e0563d74.gif)

使用`Pandas`进行数据清洗的流程如下:

1. 读取文件
    ```python
    investing = pd.read_csv("data\Ethereum Historical Data - Investing.com.csv", index_col="Date")
    investing.index = pd.to_datetime(investing.index)
    ```
1. 处理`Price`、`Open`、`Hign`、`Low`四列数据
    ```python
    investing.iloc[:, 0:4] = investing.iloc[:, 0:4].applymap(lambda x: x.replace(",", ""))
    investing.iloc[:, 0:4] = investing.iloc[:, 0:4].apply(pd.to_numeric)
    ```
1. 处理`Change %`列
    ```python
    investing["Change"] = investing["Change %"].str.rstrip('%').astype('float') / 100.0
    ```
1. 处理`Vol.`列
    ```python
    investing["Vol."] = investing["Vol."].replace({'B': '*1e9', 'K': '*1e3', 'M': '*1e6', '-': "0"}, regex=True).map(pd.eval)
    ```

使用`investing.info()`获得输出如下:
```
DatetimeIndex: 2443 entries, 2022-11-16 to 2016-03-10
Data columns (total 7 columns):
 #   Column    Non-Null Count  Dtype  
---  ------    --------------  -----  
 0   Price     2443 non-null   float64
 1   Open      2443 non-null   float64
 2   High      2443 non-null   float64
 3   Low       2443 non-null   float64
 4   Vol.      2443 non-null   float64
 5   Change %  2443 non-null   object 
 6   Change    2443 non-null   float64
dtypes: float64(6), object(1)
memory usage: 152.7+ KB
```

根据`Change`绘制的ETH价格波动分布图:

![Change Hist](https://img.gejiba.com/images/ed3434e485b50e22af9f367e31a6fdd2.png)

### yahoo ETH数据

前往[Yahoo](https://finance.yahoo.com/quote/ETH-USD/history/)网站，读者可以很明显的发现数据下载按钮，如下图:

![Yahoo Data](https://img.gejiba.com/images/a8b3c5e864273967b104f874f47da174.png)

此处可下载2017 年 11 月 9 日至今的数据，数据样例如下:

|Date      |Open      |High      |Low       |Close     |Adj Close |Volume   |
|----------|----------|----------|----------|----------|----------|---------|
|2017-11-09|308.644989|329.451996|307.056000|320.884003|320.884003|893249984|
|2017-11-10|320.670990|324.717987|294.541992|299.252991|299.252991|885985984|
|2017-11-11|298.585999|319.453003|298.191986|314.681000|314.681000|842300992|

此数据来源为[CoinmarketCap](https://coinmarketcap.com/currencies/ethereum/)网站，根据其披露的[计算方法](https://support.coinmarketcap.com/hc/en-us/articles/360015968632-How-are-prices-calculated-on-CoinMarketCap)，此数据对所有交易此加密货币的交易所的价格根据其成交量加权平均得到。

此数据源的优点如下:

- 数据权威性高，根据多个市场计算得到
- 数据较好处理，与 `Investing` 给出的数据源不同，此数据集可以直接读取为`float`数据类型

缺点如下:

- 涉及时间范围较短，但对于大部分研究者来说此数据范围已经足够研究

由于此数据源已经经过了数据清洗，所以可以较为简单的的读取，代码如下:
```python
yahoo_data = pd.read_csv("data\yahoo.csv", index_col="Date")
yahoo_data.index = pd.to_datetime(yahoo_data.index)
```
调用`yahoo_data.info()`可以得到其数据结构如下:
```
DatetimeIndex: 1834 entries, 2017-11-09 to 2022-11-16
Data columns (total 6 columns):
 #   Column     Non-Null Count  Dtype  
---  ------     --------------  -----  
 0   Open       1834 non-null   float64
 1   High       1834 non-null   float64
 2   Low        1834 non-null   float64
 3   Close      1834 non-null   float64
 4   Adj Close  1834 non-null   float64
 5   Volume     1834 non-null   int64  
dtypes: float64(5), int64(1)
memory usage: 100.3 KB
```

使用此数据源绘制ETH成交量图像如下:

![Yahoo Vol](https://img.gejiba.com/images/357d5750535ed64f8554971742ee9c60.png)

### Binance Spot

Binance 交易所作为全球最大的加密货币交易所提供的数据具有较高的权威性。Binance 通过了专门的[网站](https://www.binance.com/en/landing/data)列出了其公开的各项历史交易数据，这些数据中除`Futures Order Book Data`需要请求权限外，其他数据均可以直接下载。

![Binance Open Data](https://img.gejiba.com/images/58543b1d3dd5638ec4843092184c5e78.png)

其中各项含义如下:

- Spot 现货交易
- USDⓈ-M Futures Data U本位合约
- COIN-M Futures Data 币本位合约

下图展示了进入 Binance 具体交易数据下载的页面截图:

![Binance Spot Data](https://img.gejiba.com/images/91b2c6c5199ad17c21c42bfb1dca582d.png)

> 上图来自[Binance Spot Mothly Data](https://data.binance.vision/?prefix=data/spot/monthly/klines/)

对于数据下载，一个最简单的方法是直接通过上述给出的网页进行下载，但此种下载方法显然过慢，且一次最多下载到一个月的数据。

除在网页上下载外，Binance 交易所提供了一套开源代码，读者访问[Binance Public Data](https://github.com/binance/binance-public-data)仓库获得此代码。

> 上述开源仓库也给出了各种数据的基本示例。

为方便读者使用，本文已获取 2022 年 1 月 至 6 月六个月取样频度为 1 分钟的 ETH/USDT 现货交易K线数据为例为大家介绍如何下载相关数据。

下载源代码，如下图:

![Binance Code Save](https://img.gejiba.com/images/78ad95889f9024c9fb060bf1403c52aa.gif)

> 读者也可通过点击[此链接](https://codeload.github.com/binance/binance-public-data/zip/refs/heads/master)下载`zip`压缩的源代码

将压缩包内的`python`文件夹解压缩到您的数据工作目录，此工具需要`pandas`库支持，但相信读者都已经安装了相关库，所以此处我们不再安装依赖库。打开`cmd`命令行:

![Cmd Open](https://img.gejiba.com/images/a1b01aa5e3304fa883ce74da6b7be46c.gif)

在`cmd`内键入以下命令:
```bash
python download-kline.py -t spot -s ETHUSDT -i 1m -y 2022 -startDate 2022-01-01 -endDate 2022-07-01
```

各参数含义如下:

| 标识符 | 作用 |
| ----- | ----- |
| -t | 品种，可选择`spot`, `um`(USD-M Futures), `cm`(COIN-M Futures) |
| -s | 货币对，可参考[Symbols](https://github.com/binance/binance-public-data/blob/master/data/symbols.txt)文件 |
| -i | 时间间隔，见下文 |
| -y | 选择具体下载的数据的年份 |
| -startDate | 下载数据的起始日期 |
| -endDate | 下载数据的结束日期 |

> 关于时间间隔问题，读者可选择的具体项目可参考[此网页](https://data.binance.vision/?prefix=data/spot/monthly/klines/ETHUSDT/)，此处`m`表示分钟，而`mo`表示月

> 更多参数含义请参考[文档](https://github.com/binance/binance-public-data/blob/master/python/README.md)

此命令输入后会出现如下输出:

![Download Output](https://img.gejiba.com/images/c4d70c5198e31829f0e66f5e4fb8648f.png)

当出现`[1/1] - start download daily ETHUSDT klines`时，请同时按下`ctrl + C`键停止`python`脚本运行。

> 此脚本默认下载完月度数据后下载每日数据，前者是后者的集合，所以我们不需要下载日度数据。

最终读者在`python`文件夹内会出现`data`文件夹，目录树如下:
```
.
├── README.md
├── __pycache__
│   ├── enums.cpython-39.pyc
│   └── utility.cpython-39.pyc
├── data
│   └── spot
│       ├── daily
│       └── monthly
├── download-aggTrade.py
├── download-futures-indexPriceKlines.py
├── download-futures-markPriceKlines.py
├── download-futures-premiumIndexKlines.py
├── download-kline.py
├── download-trade.py
├── enums.py
├── requirements.txt
└── utility.py
```
数据位于`data\spot\monthly\klines\ETHUSDT\1m\2022-01-01_2022-07-01`文件夹内，且均为压缩包格式。

解压`ETHUSDT-1m-2022-01.zip`，获得`ETHUSDT-1m-2022-01.csv`，打开后样例如下:
|Open time    |Open         |High         |Low          |Close        |Volume      |
|-------------|-------------|-------------|-------------|-------------|------------|
|1640995200000|3676.22000000|3687.05000000|3676.22000000|3684.84000000|504.30200000|
|1640995260000|3684.85000000|3694.20000000|3681.33000000|3691.55000000|273.01800000|
|1640995320000|3692.50000000|3694.42000000|3687.49000000|3693.62000000|216.08240000|

| Close time   |Quote asset volume|Number of trades|Taker buy base asset volume|Taker buy quote asset volume|Ignore|
| -------------|------------------|----------------|---------------------------|----------------------------|------|
| 1640995259999|1856131.65059700  |749             |271.35540000               |998619.71581700             |0     |
| 1640995319999|1006818.04395100  |580             |181.67450000               |670095.91242200             |0     |
| 1640995379999|797656.33206100   |460             |80.15550000                |295925.00790500             |0     |
> 注意，读者的`csv`表格没有表头，此表头为参考存开源仓库内的 [K-lines](https://github.com/binance/binance-public-data#klines) 文档获得。

显然，相比于其他数据来源，Binance 数据更加丰富且频繁，此数据为每分钟采样一次得到的，所以最终得到的单文件内包含`44640`行数据。

在开始进行数据分析前，请读者将上文下载的所有压缩包解压至`unzip`文件夹内。

> 笔者使用的`7z`包含此功能，如下图:
> ![7z Exact](https://img-blog.csdnimg.cn/img_convert/2f519b63e65505a3056527e655697bd5.png)

由于此数据较为特殊，具体数据清洗代码如下:

1. 读取文件
    ```python
    import glob
    csv_dir = glob.glob(r"python/data/spot/monthly/klines/ETHUSDT/1m/2022-01-01_2022-07-01/unzip/*/*.csv")
    binance = pd.concat((pd.read_csv(f, names=["Open time", "Open", "High", "Low", "Close", "Volume", "Close time", "Quote asset volume",
                    "Number of trades", "Taker buy base asset volume", "Taker buy quote asset volume", "Ignore"]) for f in csv_dir), ignore_index=True)
    ```
1. 修正索引
    ```python
    binance.index = pd.to_datetime(binance["Open time"], unit="ms")
    ```

> 此处使用了`glob.glob`获取文件列表，然后使用`pd.read_csv`读取，最后使用`pd.concat`进行合并

使用`binance.info()`获得如下输出结果:
```
<class 'pandas.core.frame.DataFrame'>
DatetimeIndex: 305280 entries, 2022-01-01 00:00:00 to 2022-07-31 23:59:00
Data columns (total 12 columns):
 #   Column                        Non-Null Count   Dtype  
---  ------                        --------------   -----  
 0   Open time                     305280 non-null  int64  
 1   Open                          305280 non-null  float64
 2   High                          305280 non-null  float64
 3   Low                           305280 non-null  float64
 4   Close                         305280 non-null  float64
 5   Volume                        305280 non-null  float64
 6   Close time                    305280 non-null  int64  
 7   Quote asset volume            305280 non-null  float64
 8   Number of trades              305280 non-null  int64  
 9   Taker buy base asset volume   305280 non-null  float64
 10  Taker buy quote asset volume  305280 non-null  float64
 11  Ignore                        305280 non-null  int64  
dtypes: float64(8), int64(4)
memory usage: 30.3 MB
```

此数据源优势如下:

1. 直接来自 Binance 交易所，数据来源权威
1. 数据取样频度高，精确到每分钟数据
1. 数据丰富，可以获得其他数据源没有的属性

缺点如下:

1. 数据源单一
1. 数据清洗处理将为复杂

### 数据源对比

在数据源对比上，我们抛弃 Binance 的数据，此数据源较为权威，主要对比其他三种公开数据源的偏差情况。

此处对各个数据源的开盘价绘制在同一张图标上，结果如下:

![Ethereum Data Contrast](https://img.gejiba.com/images/3dcf567c82a89527c6d9d92176930d05.png)

直观来看差距不大。

使用`(ethereum_price['open'] - investing['Open']).dropna().describe()`对`EthereumPrice`和`Investing`数据对比，结果如下:
```
count    1356.000000
mean        0.527483
std         4.415048
min       -43.669255
25%        -0.387686
50%         0.172739
75%         1.186820
max        34.037451
dtype: float64
```

使用`(ethereum_price["open"] - yahoo_data["Open"]).dropna().describe()`对`EthereumPrice`和`Yahoo`数据对比，结果如下:
```
count    1356.000000
mean       -0.360980
std         5.012673
min       -41.017079
25%        -1.441817
50%        -0.348343
75%         0.872702
max        49.083426
dtype: float64
```

使用`(investing.Open - yahoo_data.Open).dropna().describe()`对`investing`和`Yahoo`数据对比如下:
```
count    1834.000000
mean       -1.541900
std         5.063060
min       -53.559995
25%        -1.811721
50%        -0.731606
75%         0.064915
max        46.371660
Name: Open, dtype: float64
```

总体来说，差别不大，读者可根据自身情况选择不同的数据源。

### 特殊数据

如果您是一位金融市场技术分析专家，您不需要上述数据而只需要K线图和其他指标，那么[Tradingview](https://www.tradingview.com/)是一个非常好的选择，作为知名K线图技术提供商，TradingView提供了一站式查看目前几乎所有交易所的相关加密货币对交易K线图的方法，且其允许在原K线图上增加各种元素，以及增加新的技术指标。下图展示了 Binance 合约及其市场费率的组合图，其中下半图为市场费率:

![Binance With Funding](https://img.gejiba.com/images/485deaf7af9652db4fac6a3673e3feb9.png)

当然，读者可以选择来自任何市场的任何数据进行观察而不需要手动收集数据和绘制相关图像。

> 此方法也适用于期货数据，下文不再说明

## 期货

期货是加密货币市场中最为重要的衍生品，大部分散户交易的均为期货而非现货。在各交易所中，一般将期货称为合约，期货交易也称合约交易。与正常期货市场不同，加密货币领域交易流动性最强的品种为永续合约，其显著特点为永不交割。当然，加密货币合约另一个著名特点就是高杠杆，最多可开出 250 倍杠杆，根据笔者观察，大部分散户均使用 20 倍以上杠杆，这在传统期货市场上是很难看到的。加密货币合约交易一般仅要求用户完成基本KYC即可，不会查验用户是否属于合格投资者，所以在合约市场内充斥了大量散户。

> 关于永续合约如何实现与现货价格的匹配属于另一个话题，交易所实际上使用了资金费率概念保证合约价值与现货价格的匹配，具体可参考[A Beginner’s Guide To Funding Rates](https://www.binance.com/en/blog/futures/a-beginners-guide-to-funding-rates-421499824684900382)。

本文主要介绍目前两家不同交易所的期货交易数据获取，这两个交易所分别是:

1. Binance
1. MEXC

在本文写作时，根据[coinmarketcap](https://coinmarketcap.com/rankings/exchanges/derivatives/)的统计，两者交易量分别为全球第一和第二，如下图:

![Derivatives Top](https://img.gejiba.com/images/6900ac686eb402562ceb6405b5c55f8d.png)

这两家的数据获取方式有明显不同，前者通过数据库下载而后者通过API请求，这两种方式基本可以涵盖目前大部分交易所的数据获取方法。

为方便传统金融研究者进行相关数据处理，本节在最后介绍 CME 交易所的 期货合约交易数据的获取方法。作为传统交易所，CME提供的均为可交割期货合约，具体提供的品种请参考[此链接](https://www.cmegroup.com/markets/cryptocurrencies.html#explore-our-cryptocurrency-products)。

### MEXC 期货数据获取

由于我个人对此交易所并不熟悉，简单了解了一下，分析[MEXC 期货交易](https://www.mexc.com/futures)似乎仅包含永续合约交易。

在正式开始期货交易数据获取前，读者应当确认此期货品种是否正在交易，[此页面](https://www.coingecko.com/en/exchanges/mexcglobal_futures)列出来所有可交易期货品种。

我们此次仍以`ETHUSDT`期货交易对为示例。

> `ETHUSDT`的含义为以USDT(一种美元稳定币)为保证金进行ETH期货交易，此种交易对一般被称为`USDT-M`交易对。而如`BTCUSD`等交易对则以BTC为保证金进行美元期货交易，此种交易对被称为`COIN-M`。

MEXC 交易所仅能使用API获取历史K线数据，无法获得其他历史数据，且MEXC也没有提供其他下载接口。

此处使用的API为`https://www.mexc.com/open/api/v2/market/kline`，此API支持以下参数:

| 参数名        | 数据类型    | 是否必须 | 说明              | 取值范围                                     |
|:----------:|:-------:|:----:|:---------------:|:----------------------------------------:|
| symbol     | string  | 是    | 交易对名称           |
| interval   | string  | 是    | 时间间隔            | 分钟制:1m，5m，15m，30m，60m。小时制:4h，天制:1d，月制:1M |
| start_time | long    | 否    | 起始时间。以秒记的10位时间戳 |
| limit      | integer | 否    | 返回条数            | 1~1000，默认值100                            |

为方便后文数据读取，我们定义两个函数:
```python
def split_interval(start: datetime, end: datetime) -> list:
    interval_days = (end - start).days
    split_freq = interval_days // 1000
    remainder = interval_days % 1000

    split_list = []
    for i in range(split_freq + 1):
        split_timestamp = start_date + timedelta(1000) * i
        split_list.append(split_timestamp.timestamp())

    return split_list

def get_data(split_list: list, symbol: str) -> list:
    k_lines_api = "https://www.mexc.com/open/api/v2/market/kline"
    result_list = []
    for i in split_list:
        params = {
            "symbol": symbol,
            "interval": "1d",
            "start_time": f"{i:.0f}",
            "limit": 1000
        }

        data_json = requests.get(k_lines_api, params=params).json()
        result_list.extend(data_json["data"])

    return result_list
```
由于`mexc`API仅允许返回 1000 条信息，所以我们此处通过`split_interval`将获取数据的区间进行分段(此处以 1d 为单位获取，如果读者需要其他间隔请自行修改代码)，划分为各间断点构成的列表。然后使用`get_data`获取以各间断点为起点的数据。

进行完基础的数据爬取后，我们可以进行常规的数据清洗工作:

1. 爬取数据
    ```python
    import requests
    from datetime import datetime
    from datetime import timedelta

    split_list = split_interval(start_date, end_date)
    mexc_data = get_data(split_list, "ETH_USDT")
    ```
1. 列表转化
    ```python
    mexc = pd.DataFrame(mexc_data, columns=[
                    "time", "open", "close", "high", "low", "vol", "amount"])
    ```
1. 时间序列化
    ```python
    mexc.index = pd.to_datetime(mexc.time, unit="s")
    ```
1. 数据格式规整
    ```python
    mexc.iloc[:, 1:7] = mexc.iloc[:, 1:7].apply(pd.to_numeric)
    ```

各参数含义如下表:

| 参数名    | 说明                 |
|:------:|:------------------:|
| time   | 开始时间 (以秒表示的10位时间戳) |
| open   | 开盘价                |
| close  | 收盘价                |
| high   | 最高价                |
| low    | 最低价                |
| vol    | 成交量                |
| amount | 计价货币成交量            |

使用`mexc.vol.plot()`绘图如下:

![Mexc Future Vol](https://img.gejiba.com/images/ec54959ca8fde2d8f04c9ad818ad9994.png)

> 如果读者想了解更多关于`mexc`的`api`的信息，请自行查阅[文档](https://mxcdevelop.github.io/APIDoc/open.api.v2.en.html)

### Binance 期货数据获取

与 MEXC 仅有永续合约不同， Binance 交易所提供部分加密货币的可交割期货合约。Binance 提供季度可交割期货合约，具体品种可在[可交割合约交易页面](https://www.binance.com/en/futures/btcusdt_quarter)获得。

读者也可在[Binance Data](https://data.binance.vision/?prefix=data/futures/um/daily/klines/)查询到具体可交割期货的期货名称，如下图:

![Binace Contract](https://img.gejiba.com/images/573a8967f9d30ddc283c2e79ba1a5fe3.png)

关于 Binance 期货交易数据的下载其实与现货交易数据下载方法相同，仅需要更改部分参数即可，如下命令:
```bash
python download-kline.py -t um -s ETHUSDT -i 1m -y 2022
```
上述命令即可下载`ETHUSDT`期货合约的相关数据，数据清洗流程也是类似的。

### CME 期货交易数据获取

本文以 BTC 期货交易数据获取为例，进入 BTC 品种[页面](https://www.cmegroup.com/markets/cryptocurrencies/bitcoin/bitcoin.html)，点击`calendar`按钮获取具体的交易品种和交易日期，如下图:

![CME Calendar](https://img.gejiba.com/images/d15d9026e9df4c3ce59665bdde42a91c.png)

此处以在 30 May 2022 至 25 Nov 2022 交易的 BTCX22 为例进行相关数据获取，由于CME官网并没有免费公开其交易数据，此处我们通过第三方获取其交易数据。

为简化数据获取流程，请读者使用以下命令安装相关库:
```bash
pip install pandas-datareader
```

具体数据处理代码如下:
```python
import pandas_datareader as pdr
BTCX22 = pdr.yahoo.daily.YahooDailyReader("BTCX22.CME").read()
```
上述代码默认获取`BTCX22`从发售到目前所有的交易数据并且交易数据均清洗良好。关于上述代码更加详细的介绍请自行参考[pandas_datareader](https://pandas-datareader.readthedocs.io/en/latest/readers/yahoo.html)的文档。

> 注意上述代码使用了 [yahoo finance](https://finance.yahoo.com/)，此网站已拒绝大陆访问，请读者自行解决此问题。

### Deribit 可交割期货交易数据获取

作为全球第五大加密货币衍生品交易所， Deribit 提供了大量的期货和期权，此处我们主要介绍 [Deribit](https://www.deribit.com/) 的可交割期货的数据获取。 

Deribit 一般存在以下上市品种:

1. 当月交割品种
1. 当月交割后 7 日后
1. 下一年 3 月、6 月 和 9 月

具体的交割日期为当月最后一个周五。命名规则为`[标的资产]-[交割日期]`

以作者写作本文时的 11 月为例，此时应存在当月交割的BTC期货，即以 11 月 最后一个周五为交割日的期货，查询日历，发现此日期为 11 月 25 日，则该期货名称为`BTC-25Nov22`。第二个可交割期货应在11月25日后 7 日，即 12 月 2 日，名称为`BTC-2DEC22`。

从上述品种外，还存在:

1. 2023 年 3 月的`BTC-31MAR23`
1. 2023 年 6 月的`BTC-30JUN23`
1. 2023 年 9 月的`BTC-29SEP23`

由于 Deribit 并未给出详细的品种规则，上述数据为我个人总结，目前来看下一年 3 月、6 月和 9 月 的期货的产生与具体的月份有一定关系，暂时未找到规律。

读者可以提供[此链接](https://history.deribit.com/api/v2/public/get_instruments?currency=BTC&kind=future&expired=true)获得到历史上所有BTC期货品种的详细情况，示例数据如下:
```json
{
    "tick_size": 0.01,
    "taker_commission": 0.0001,
    "settlement_period": "week",
    "settlement_currency": "BTC",
    "rfq": false,
    "quote_currency": "USD",
    "price_index": "btc_usd",
    "min_trade_amount": 10.0,
    "max_liquidation_commission": 0.002,
    "max_leverage": 50,
    "maker_commission": 0.0,
    "kind": "future",
    "is_active": false,
    "instrument_name": "BTC-15JUL16",
    "instrument_id": 200878,
    "future_type": "reversed",
    "expiration_timestamp": 1468594800000,
    "creation_timestamp": 1468431060000,
    "counter_currency": "USD",
    "contract_size": 10.0,
    "block_trade_commission": 0.0001,
    "base_currency": "BTC"
}
```

此`JSON`数据格式化后有 3921 行，建议使用 VSCode 等查看，亦可转化为 DataFrame 进行高级查询。关于链接构造参数和返回参数的详细含义请参考[文档](https://docs.deribit.com/#public-get_instruments)。


> 此处使用了`history.deribit.com/api/v2/`网址以获取全部历史数据，注意此前缀可调用的接口有限，具体请参考[Deribit Twitter](https://twitter.com/DeribitExchange/status/1570135191972085764)

此处我们仅展示如何获取历史交易数据，包含开盘价、收盘价等数据，代码如下:
```python
def get_deribit_data(symbol: str, interval: str, start: int, end: int) -> pd.DataFrame:
    base_url = "https://asia.deribit.com/api/v2/public/get_tradingview_chart_data"
    params = {
        "instrument_name": symbol,
        "start_timestamp": start,
        "end_timestamp": end,
        "resolution": interval
    }
    funture_json = requests.get(base_url, params=params).json()["result"]
    del funture_json["status"]
    funture_data = pd.DataFrame(funture_json)
    funture_data.index = pd.to_datetime(funture_data.ticks, unit="ms")
    funture_data.drop("ticks", inplace=True, axis=1)

    return funture_data
```

其中各参数取值请参考[Deribit API文档](https://docs.deribit.com/#public-get_tradingview_chart_data)。

在此处以获取`BTC-4NOV22`的每分钟交易数据为例，代码如下:
```python
deribit_funture = get_deribit_data("BTC-4NOV22", "60", 1666339207000, 1667548800000)
```

此处使用的`start`和`end`均来自上文给出的期货品种历史数据中的`creation_timestamp`(期货品种上市日期)和`expiration_timestamp`(期货品种交割日期)。

使用`deribit_funture.info()`获得的结果如下:
```
<class 'pandas.core.frame.DataFrame'>
DatetimeIndex: 337 entries, 2022-10-21 08:00:00 to 2022-11-04 08:00:00
Data columns (total 6 columns):
 #   Column  Non-Null Count  Dtype  
---  ------  --------------  -----  
 0   volume  337 non-null    float64
 1   open    337 non-null    float64
 2   low     337 non-null    float64
 3   high    337 non-null    float64
 4   cost    337 non-null    float64
 5   close   337 non-null    float64
dtypes: float64(6)
memory usage: 18.4 KB
```

绘制`open`开盘价折线图，如下:

![Deribit Funture Plot](https://img.gejiba.com/images/c68da3a3577f2f6a877cc5baaea6ad31.png)

## 期权

由于作者本人对于期权的相关统计和分析并不是非常清楚，本文会尽可能给出期权的相关信息。在加密货币领域最重要的期权交易所为 [deribit](https://www.deribit.com/) ，此交易所提供BTC、ETH和SOL的期权。本文仅介绍此交易所，当然CME也提供期货期权交易，但此交易数据不太好获得，所以此处不进行介绍。

`Deribit` 的期权具有以下规定:

1. 名称格式为`[标的资产]-[到期日][到期年份]-[执行价]-[C/P]`
    如`ETH-14SEP22-1700-C`指 2022 年 9 月 14 日到期的执行价格为 1700 USD/ETH 的ETH看涨期权
1. 期权在每天 8:00 UTC 时间到期
1. 均为欧式期权到期自动行权，采用美元交割

> 对于上述规定的更加具体版本请参考[交易规则](https://www.deribit.com/kb/options)

Deribit 可以保证每日都存在交割的期权品种，读者可以通过以下链接下载期权品种列表:
```
https://history.deribit.com/api/v2/public/get_instruments?currency=ETH&kind=option&expired=true
```
此链接返回的JSON文件在2022年11月18日已达到 20.771 MB 的大小，格式化后长度为 98 万行，请读者不要尝试使用浏览器直接打开链接。样例数据如下:
```json
{
    "tick_size": 0.0005,
    "taker_commission": 0.0003,
    "strike": 1700.0,
    "settlement_period": "day",
    "settlement_currency": "ETH",
    "rfq": false,
    "quote_currency": "ETH",
    "price_index": "eth_usd",
    "option_type": "call",
    "min_trade_amount": 1,
    "maker_commission": 0.0003,
    "kind": "option",
    "is_active": false,
    "instrument_name": "ETH-14SEP22-1700-C",
    "instrument_id": 231300,
    "expiration_timestamp": 1663142400000,
    "creation_timestamp": 1662969660000,
    "counter_currency": "USD",
    "contract_size": 1.0,
    "block_trade_commission": 0.0003,
    "base_currency": "ETH"
}
```

我们在此处仍是获取期权交易的历史数据，使用上文给出的`get_deribit_data`函数，代码如下:
```python
deribit_option = get_deribit_data("ETH-14SEP22-1700-C", "60", 1662969660000, 1663142400000)
```

最后使用期权交易数据的交易量可以绘制如下图像:

![Deribit Option Vol](https://img.gejiba.com/images/9dc05d773ec797fdfbecb462308bae3d.png)

## 总结

本文介绍了大量关于加密货币金融衍生品的数据获取方法，当然也包含了现货交易数据的获取方法，读者可以前往[此仓库](https://github.com/wangshouh/cryptoFinanceData)获得仅包含代码的 Jupyter NoteBook 文件。注意开源仓库内的 Jupyter NoteBook 不包含相关数据文件夹，请读者根据本文内容自行下载。
