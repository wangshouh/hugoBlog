---
title: "深入解析AAVE智能合约:计算和利率"
date: 2022-12-21T13:47:33Z
tags: [aave,defi]
math: true
---

## 概述

本文主要讨论`AAVE V3`中的数学计算模块，该模块位于`src/protocol/libraries/math`文件夹内，基础合约为`WadRayMath`。

本文主要包含以下内容:

1. 浮点数的在`solidity`内的表示及四则运算
1. 单利和复利的计算
1. 百分比的乘除

相比于上一篇文章，本文较为简单且篇幅较短。

## 浮点数表示与计算

在AAVE中，我们使用定点浮点数进行浮点数表示，具体单位如下:
```solidity
uint256 internal constant WAD = 1e18;
uint256 internal constant RAY = 1e27;
```
显然，使用`wad`表示的浮点数精度为小数点后18位，而使用`ray`进行标识则精度为小数点后27位。

在`WayRayMath`中，我们也定义了一些必要的常量以用于后续的计算，如下:
```solidity
uint256 internal constant HALF_WAD = 0.5e18;

uint256 internal constant HALF_RAY = 0.5e27;

uint256 internal constant WAD_RAY_RATIO = 1e9;
```

我们定义了基础的数据类型后，我们需要完善其基础的乘除操作。此处仅完善乘除操作是因为自行定义的方法可以保证乘除操作可以实现四舍五入。

我们首先分析`wad`的乘法操作:
```solidity
function wadMul(uint256 a, uint256 b) internal pure returns (uint256 c) {
    // to avoid overflow, a <= (type(uint256).max - HALF_WAD) / b
    assembly {
        if iszero(
            or(iszero(b), iszero(gt(a, div(sub(not(0), HALF_WAD), b))))
        ) {
            revert(0, 0)
        }

        c := div(add(mul(a, b), HALF_WAD), WAD)
    }
}
```

此函数首先进行了条件判断，要求同时满足以下条件:

1. `b`的数值不为`0`
1. `a` 小于 `(type(uint256).max - HALF_WAD) / b`

这些条件存在的原因为:

1. 避免相除时分母为 0 
1. 避免`a * b`溢出

> 为什么使用`(type(uint256).max - HALF_WAD) / b`而不是`type(uint256).max / b` 作为判断是否溢出的条件？正如前文所述，此乘法会进行四舍五入，所以如果使用`type(uint256).max / b`进行判断，可能导致进一后溢出。

此处，对于第二个条件的判定稍有复杂，我们在此处特别分析`iszero(gt(a, div(sub(not(0), HALF_WAD), b)))`部分代码。`not(0)`的数值大小即为`type(uint256).max`(`0`使用`not`全部取反后即为`type(uint256).max`)。`sub(a, b)`操作符的含义即为`a - b`，而`gt(a, b)`则会判断`a > b`，如不符合条件，则返回`0`，符合条件则返回`1`。

> 上述操作符可以在[EVM Codes](https://www.evm.codes/?fork=merge)内找到其具体内容

如果不满足上述条件则会`revert`抛出异常。部分读者可以对`revert(offset, size)`操作符不熟悉，简单来说，此操作符会中止函数运行并回滚，退回未使用`gas`，并将从`offset`开始的长度为`size`大小的内容作为异常信息抛出。

> `revert`的更加详细的介绍请参考[evm codes](https://www.evm.codes/#fd?fork=merge)

介绍了对参数的具体要求后，我们接下来介绍真正的计算步骤`c := div(add(mul(a, b), HALF_WAD), WAD)`，使用数学语言描述为`(a * b + HALF_WAD) // WAD`，其中`//`即为整除符号，但此整除仅会向下取整。

我们通过将`a * b`的实际值与`HALF_WAD`(相当于 `0.5`)相加再与`WAD`(相当于`1`)进行整除获得四舍五入的结果。

> 此处除去了一个`WAD`的原因有以下两点:
> 1. 保证单位仍未`WAD`
> 1. 通过整除实现四舍五入

此处，我们可以讨论一下此函数的函数类型，此函数的函数类型为:

1. `internal` 表示此函数仅能用于合约内部调用
1. `pure` 表示此函数不读取合约内的状态变量也不修改合约变量

在介绍完乘法相关的运算的方法后，我们介绍除法的具体实现，代码如下:
```solidity
function wadDiv(uint256 a, uint256 b) internal pure returns (uint256 c) {
    // to avoid overflow, a <= (type(uint256).max - halfB) / WAD
    assembly {
        if or(
            iszero(b),
            iszero(iszero(gt(a, div(sub(not(0), div(b, 2)), WAD))))
        ) {
            revert(0, 0)
        }

        c := div(add(mul(a, WAD), div(b, 2)), b)
    }
}
```

在讨论具体的代码限制条件前，我们首先分析其具体的逻辑代码部分即`c := div(add(mul(a, WAD), div(b, 2)), b)`，改写为数学表达式为`((a * WAD) + b // 2) // b`

具体推导过程如下:
$$
\begin{align}
    \lbrack{a / b}\rbrack &= \lfloor{a / b + \frac{1}{2}}\rfloor \\\
    &= ((a / b) + \frac{1}{2}) // 1 \\\
    &= (a * 1 + \frac{b}{2}) // b
\end{align}
$$

最终获得的 $(3)$ 与我们代码中的公式是一致的。注意此处的的`1`不可以直接省去，原因是当 $a // b$ 后即意味着整个数学式子的单位(即`WAD`)的丢失，此处我们需要补齐此单位。

从式 $(2)$ 到 式 $(3)$ 进行转化的原因是在`solidity`中不存在正常的除法，仅存在整除，我们只能进行分子分母同乘`b`的操作使除法消失。

通过最后的公式，我们可以得到如下限制条件:

$$
a * {WAD} + \frac{b}{2} \leqq {MAX}_{uint256}
$$

上式等同于:

$$
a \leqq \frac{{MAX}_{uint256} - \frac{b}{2}}{WAD}
$$

翻译为以下代码:
```solidity
iszero(iszero(gt(a, div(sub(not(0), div(b, 2)), WAD))))
```
由于`solidity`没有提供小于等于的运算符，此处使用了`iszero(iszero(gt`的方法进行了实现。

对于另一个使用`RAY`为单位的计算方法逻辑完全一致，代码也几乎相同，此处不再赘述。

在介绍完基本运算后，我们介绍如何实现两者的互相转换，首先介绍`rayToWad`函数，代码如下:
```solidity
function rayToWad(uint256 a) internal pure returns (uint256 b) {
    assembly {
        b := div(a, WAD_RAY_RATIO)
        let remainder := mod(a, WAD_RAY_RATIO)
        if iszero(lt(remainder, div(WAD_RAY_RATIO, 2))) {
            b := add(b, 1)
        }
    }
}
```
此函数使用以下三个步骤完成了转换:

1. 将`ray`与转换因子`WAD_RAY_RATIO`整除获得一个向下取整的结果
1. 使用`mod`取模运算获得上述除法的余数
1. 判断余数与`0.5`的关系以确定结果是否需要加 1 (即完成四舍五入的步骤)

另一个转换函数为`wadToRay`，代码如下:
```solidity
function wadToRay(uint256 a) internal pure returns (uint256 b) {
    // to avoid overflow, b/WAD_RAY_RATIO == a
    assembly {
        b := mul(a, WAD_RAY_RATIO)

        if iszero(eq(div(b, WAD_RAY_RATIO), a)) {
            revert(0, 0)
        }
    }
}
```
此处直接使用`b`与转换因子`WAD_RAY_RATIO`进行乘法计算转换后的结果，最后通过判断`b / WAD_RAY_RATIO`是否等于`a`判断上述乘法是否溢出。

## 利率相关计算

在介绍完各类计算的基石类型后，我们介绍本节中最重要的利率计算这一话题。

首先，我们介绍最为简单的单利计算，代码如下:
```solidity
function calculateLinearInterest(uint256 rate, uint40 lastUpdateTimestamp)
    internal
    view
    returns (uint256)
{
    //solium-disable-next-line
    uint256 result = rate *
        (block.timestamp - uint256(lastUpdateTimestamp));
    unchecked {
        result = result / SECONDS_PER_YEAR;
    }

    return WadRayMath.RAY + result;
}
```
注意此处的利率`rate`为年利率。但我们将`block.timestamp - uint256(lastUpdateTimestamp)`与`rate`相乘，前者是以秒为单位，所以在进行完运算后，我们需要将结果与`SECONDS_PER_YEAR`相除，获得以年为单位的最终结果。

> 在 `solidity 8.0` 版本后，`solidity`原生支持运算溢出保护，我们不再需要引入`safeMath`等合约。但溢出保护会增加`gas`消耗，如果确定运算不会溢出，可以使用`unchecked`进行包裹以减少`gas`消耗，这就是上述代码使用`unchecked`的原因

复利计算是本节中最为复杂的计算，在介绍代码之前，我们首先相关的数学基础，即[Binomial series](https://en.wikipedia.org/wiki/Binomial_series)，公式如下:

$$
(1+x)^{n }=1+n x+{\frac {n (n -1)}{2!}}x^{2}+{\frac {n (n -1)(n -2)}{3!}}x^{3}+\cdots 
$$

基于此公式，我们可以将乘方运算转化为多项式运算。为减少`gas`消耗，我们仅构造到上述公式的第 4 项为止。

我们首先构造 $1+n x$ 部分，转化为代码如下:
```
WadRayMath.RAY + (rate * exp) / SECONDS_PER_YEAR
```

此处的`exp`即公式中的`n`。其在复利计算中的实际意义是时间间隔，定义如下:
```solidity
uint256 exp = currentTimestamp - uint256(lastUpdateTimestamp);
```

接下来构造 $ {\frac {n (n -1)}{2!}}x^{2} $ 部分，代码如下:
```solidity
// 构造 n - 1
expMinusOne = exp - 1;
// 构造 x²
basePowerTwo = rate.rayMul(rate) / (SECONDS_PER_YEAR * SECONDS_PER_YEAR);
// 整体构造
uint256 secondTerm = exp * expMinusOne * basePowerTwo;
unchecked {
    secondTerm /= 2;
}
```

同理，我们使用以下代码构造最后一部分:
```solidity
expMinusTwo = exp > 2 ? exp - 2 : 0;
basePowerThree = basePowerTwo.rayMul(rate) / SECONDS_PER_YEAR;
uint256 thirdTerm = exp * expMinusOne * expMinusTwo * basePowerThree;
unchecked {
    thirdTerm /= 6;
}
```
最后将所有数据加和返回即可。上述代码汇总起来就是`calculateCompoundedInterest`函数，此处不再给出其完整代码。

## 百分比的乘除

对于百分数的相关计算与`wad`相关运算基本一致，仅在数值定义上有所不同，百分数相关数据的定义如下:
```solidity
// Maximum percentage factor (100.00%)
uint256 internal constant PERCENTAGE_FACTOR = 1e4;

// Half percentage factor (50.00%)
uint256 internal constant HALF_PERCENTAGE_FACTOR = 0.5e4;
```
## 总结

本文介绍了`AAVE`中用于数学计算的模块，此模块代码量较少，且较易理解。我们依次介绍了以下内容:

1. 浮点数的表示及相关运算
1. 单利复利的计算

希望此篇文章对读者进行合约编程有所帮助。
