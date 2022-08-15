---
title: "基于链下链上双视角深入解析以太坊签名与验证"
date: 2022-08-08T11:27:17Z
tags: [ECDSA,secp256k1,EIP-712,solidity]
aliases: ["/2022/08/02/ecsda-sign-chain"]
---

## 概述

本文主要介绍在以太坊中的签名问题，主要涵盖以下内容:

1. ECDSA公钥密码学的数学原理与代码实现解析
1. 以太坊客户端对交易信息签名的基本流程与源代码分析
1. 智能合约内签名的验证

## ECDSA公钥密码学

为了方便读者理解和实战本文中的内容，本文将结合一个可以使用`Typescript`编写用于生产环境的`noble-secp256k1`库作为实战案例解析。你可以在[这里](https://github.com/paulmillr/noble-secp256k1)找到源代码。当然，为了节省篇幅，本文不会对此库中的所有代码进行解析。
### 公钥生成

以下内容部分参考了[比特币 P2TR 交易详解](https://www.btcstudy.org/2022/06/13/part-2-bitcoin-p2tr-transaction-breakdown)。

在椭圆密码学中，许多不同种类的曲线都可以用于生成公钥。以太坊选择了与比特币相同的曲线类型，形式为`y² = x³ + 7`，被称为`secp256k1`。具体的图像如下图:

![secp256k1 Img](https://img.wang.232232.xyz/img/2022/08/15/rGv48gca4b516a4994fc8f.jpg)

在此图像上，我们可以选择一个点作为生成点`G`，使用`陷门函数`计算获得公钥。陷门函数特点是正向计算简单，我们可以快速从私钥求出公钥，而逆向计算难度巨大。

比特币与以太坊均选择了一种被称为`点倍增`的陷门函数。如下图为我们选择的生成点`G`：
![G point](https://img.wang.232232.xyz/img/2022/08/15/n7jg2Qa3eda935ae1d924b.jpg)

我们画出过点`G`的切线与曲线交与一点，我们选择此点关于`x`轴的对称点作为`2G`点。下图展示了进行第一次点倍增后的结果`2G`:

![2G point](https://img.wang.232232.xyz/img/2022/08/15/DiglQw30e2fb984acc31e3.jpg)

连结`G`与`2G`与曲线交与一点，我们选择与此点关于x轴对称的点作为`3G`。如下图:

![3G point](https://img.wang.232232.xyz/img/2022/08/15/elbrtQ04528b18e77aa66c.jpg)

依次类推，我们可以得到`4G`的图像如下:

![4G point](https://img.wang.232232.xyz/img/2022/08/15/jlNwag7e27300948e58efc.jpg)

显然上述操作是直觉上是无法逆向的，关于严格的数学证明，读者可以自行查阅相关论文。以上过程可以进行算法上的优化，读者可以自行阅读[noble-secp256k1](https://github.com/paulmillr/noble-secp256k1)的开发者的写的关于加速`secp256k1`计算的[博客](https://paulmillr.com/posts/noble-secp256k1-fast-ecc/)。

在此给出比特币规定的`secp256k1`的`G`的数值:
```
G.x = 0x79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798
G.y = 0x483ada7726a3c4655da4fbfc0e1108a8fd17b448a68554199c47d08ffb10d4b8
```

综上所述，当我们生成一个私钥`d`后，我可以通过计算`P=dG`获得公钥`P`，而以太坊账户就是选择公钥最后`20 Bytes`进行`keccak-256`计算得到的。

在`noble-secp256k1`[实现](https://github.com/paulmillr/noble-secp256k1/blob/main/index.ts#L440)如下:
```typescript
static fromPrivateKey(privateKey: PrivKey) {
    return Point.BASE.multiply(normalizePrivateKey(privateKey));
}
```
当然，上述代码中的`multiply`是经过优化的。`Point.BASE`即上文给出的`G`点。在代码中使用了`bigint`表示，[定义](https://github.com/paulmillr/noble-secp256k1/blob/main/index.ts#L28)如下:
```typescript
Gx: BigInt('55066263022277343669578718895168534326250603453777594175500187360389116729240')
Gy: BigInt('32670510020758816978083085130507043184471273380659243275938904335757337482424')
```

当然，`secp256k1`也存在定义域，其最大值被记为`n`，任何有效的点都应在`n`之内。具体[定义](https://github.com/paulmillr/noble-secp256k1/blob/main/index.ts#L25)如下:
```
n: BigInt('0xfffffffffffffffffffffffffffffffebaaedce6af48a03bbfd25e8cd0364141')
```

我们也可以在`go-ethereum`找到以下[定义](https://github.com/ethereum/go-ethereum/blob/master/crypto/crypto.go#L48):
```go
var (
	secp256k1N, _  = new(big.Int).SetString("fffffffffffffffffffffffffffffffebaaedce6af48a03bbfd25e8cd0364141", 16)
	secp256k1halfN = new(big.Int).Div(secp256k1N, big.NewInt(2))
)
```

### 签名
为了方便读者阅读，给出变量说明:
| 变量缩写 | 含义 |
| ------ | ---- |
| d | 私钥 |
| G | 生成点 |
| n | 曲线最大值 |


标准的`ECDSA`签名由两个整数`r`和`s`构成，签名流程如下:

1. 对待签数据进行哈希计算获得哈希值`m`
1. 生产随机数`k`，并使用`k`与`G`相乘获得点`R`
1. 计算`r = R.x mod n`。如果`r = 0`则需要重新生产随机数`k`
1. 计算`s = (1/k * (m + dr) mod n`。如果`s = 0`则需要重新生产随机数`k`

上述过程中进行`mod n`是为了确保计算出的数值在我们所规定的定义域内。

在以太坊中，为了避免签名字段被其他应用使用，对哈希值`m`计算进行特别规定，即使用`Keccak256("\x19Ethereum Signed Message:\n32" + Keccak256(message))`进行哈希计算，使用`go`实现如下:
```go
func signHash(data []byte) []byte {
   msg := fmt.Sprintf("\x19Ethereum Signed Message:\n%d%s", len(data), data)
   return crypto.Keccak256([]byte(msg))
}
```

对其签名的具体的代码[实现](https://github.com/paulmillr/noble-secp256k1/blob/main/index.ts#L973)方式如下:
```typescript
function kmdToSig(kBytes: Uint8Array, m: bigint, d: bigint): RecoveredSig | undefined {
  const k = bytesToNumber(kBytes);
  if (!isWithinCurveOrder(k)) return;
  // Important: all mod() calls in the function must be done over `n`
  const { n } = CURVE;
  const q = Point.BASE.multiply(k);
  // r = x mod n
  const r = mod(q.x, n);
  if (r === _0n) return;
  // s = (1/k * (m + dr) mod n
  const s = mod(invert(k, n) * mod(m + d * r, n), n);
  if (s === _0n) return;
  const sig = new Signature(r, s);
  const recovery = (q.x === sig.r ? 0 : 2) | Number(q.y & _1n);
  return { sig, recovery };
}
```
上述代码给出了签名必要的两个元素`r`和`s`，通过这两个元素我们就可以得到一个完整的比特币签名。根据[BIP66](https://github.com/bitcoin/bips/blob/master/bip-0066.mediawiki)规定，比特币的签名组成如下:
```
 0x30 [total-length] 0x02 [R-length] [R] 0x02 [S-length] [S]
```
我们可以通过上述代码给出的`sig`变量中的信息填充上述签名。

但在以太坊中，以太坊要求的签名格式如下:
```
[R][S][V]
```
其中增加了变量`v`，此值已在上文的代码中给出，为`recovery`值。但与尚未给出的通用`recover`不同，以太坊交易签名中的`v`为`27`或`28`，分别对应`0`和`1`。这个奇怪的增加`27`的规则是从比特币的设计中继承来的。此值可以保证在从签名中恢复密钥时尽可以获得一个正确的公钥。如果不包含此值，可以恢复出两个公钥，需要程序通过其他方式判断。

### 验证签名

验证签名需要对签名者的公钥进行恢复，具体的流程如下:

1. 计算签名信息的哈希值`m`
1. 计算点`R = (x, y)`。其中，当`v=0`时，`x=r`; 当`v=1`时，`x=r+n`
1. 计算`u1 = hs^-1 mod n`，其中`h`为经过调整的哈希值，调整算法参考[truncateHash](https://github.com/paulmillr/noble-secp256k1/blob/main/index.ts#L882)
1. 计算`u2 = sr^-1 mod n`
1. 计算`Q = u1 * G + u2 * R`，`Q`即签名者的公钥。

我们在此给出对应的[实现](https://github.com/paulmillr/noble-secp256k1/blob/main/index.ts#L454)代码:
```typescript
static fromSignature(msgHash: Hex, signature: Sig, recovery: number): Point {
    msgHash = ensureBytes(msgHash);
    const h = truncateHash(msgHash);
    const { r, s } = normalizeSignature(signature);
    if (recovery !== 0 && recovery !== 1) {
        throw new Error('Cannot recover signature: invalid recovery bit');
    }
    if (h === _0n) throw new Error('Cannot recover signature: msgHash cannot be 0');
    const prefix = recovery & 1 ? '03' : '02';
    const R = Point.fromHex(prefix + numTo32bStr(r));
    const { n } = CURVE;
    const rinv = invert(r, n);
    // Q = u1⋅G + u2⋅R
    const u1 = mod(-h * rinv, n);
    const u2 = mod(s * rinv, n);
    const Q = Point.BASE.multiplyAndAddUnsafe(R, u1, u2);
    if (!Q) throw new Error('Cannot recover signature: point at infinify');
    Q.assertValidity();
    return Q;
}
```

对于上述代码，基本逻辑与我们介绍的流程是相同的。但开发者为了优化代码使用了许多函数，这些函数大多包含位移、算法优化和增强安全性的内容，我们不在此深入研究。开发者将所有的代码都放在了`index.ts`中，读者可以仅[下载](https://raw.githubusercontent.com/paulmillr/noble-secp256k1/main/index.ts)`index.ts`，然后自行使用`vscode`阅读代码，请善用函数定义跳转功能`F12`。

通过上述流程，我们可以获得信息签名者的公钥，进一步可以获得签名者的以太坊地址。

## 以太坊交易签名

在`go-ethereum`中，我们可以查到以下关于交易签名的[源代码](https://github.com/ethereum/go-ethereum/blob/master/core/types/transaction_signing.go#L94)：
```go
// SignTx signs the transaction using the given signer and private key.
func SignTx(tx *Transaction, s Signer, prv *ecdsa.PrivateKey) (*Transaction, error) {
	h := s.Hash(tx)
	sig, err := crypto.Sign(h[:], prv)
	if err != nil {
		return nil, err
	}
	return tx.WithSignature(s, sig)
}
```
代码中最为关键的部分是`s.Hash(tx)`。从函数的参数中，我们可以得到`s`为`Signer`类型。跳转定义，我们发现`Signer`为接口类型，[具体代码](https://github.com/ethereum/go-ethereum/blob/master/core/types/transaction_signing.go#L156)如下:
```go
type Signer interface {
	// Sender returns the sender address of the transaction.
	Sender(tx *Transaction) (common.Address, error)

	// SignatureValues returns the raw R, S, V values corresponding to the
	// given signature.
	SignatureValues(tx *Transaction, sig []byte) (r, s, v *big.Int, err error)
	ChainID() *big.Int

	// Hash returns 'signature hash', i.e. the transaction hash that is signed by the
	// private key. This hash does not uniquely identify the transaction.
	Hash(tx *Transaction) common.Hash

	// Equal returns true if the given signer is the same as the receiver.
	Equal(Signer) bool
}
```

出现`Signer`接口的原因是为了适配以太坊的升级，在以太坊升级过程中，开发者升级过多次签名流程中的`tx`交易数据结构。为了保证代码的简洁性，引入了接口类型。在代码中，对此接口的实现是根据区块高度决定的:
```go
// MakeSigner returns a Signer based on the given chain config and block number.
func MakeSigner(config *params.ChainConfig, blockNumber *big.Int) Signer {
	var signer Signer
	switch {
	case config.IsLondon(blockNumber):
		signer = NewLondonSigner(config.ChainID)
	case config.IsBerlin(blockNumber):
		signer = NewEIP2930Signer(config.ChainID)
	case config.IsEIP155(blockNumber):
		signer = NewEIP155Signer(config.ChainID)
	case config.IsHomestead(blockNumber):
		signer = HomesteadSigner{}
	default:
		signer = FrontierSigner{}
	}
	return signer
}
```

> `London`、`BerLin`都是以太坊的阶段代号，具体可以参考[The history of Ethereum](https://ethereum.org/en/history)

由于我们没有必要使用以前`Signer`，在此处我们仅介绍最新的实现`londonSigner`。此签名器满足以下标准:

- EIP-1599，详情参看[以太坊文档 Gas和费用](https://ethereum.org/zh/developers/docs/gas/#post-london)
- EIP-2930，增加了`accessList`参数，预支付存储费用
- EIP-155，增加`CHAIN_ID`参数，防止重放攻击

我们所需要研究的函数如下:
```go
func (s londonSigner) Hash(tx *Transaction) common.Hash {
	if tx.Type() != DynamicFeeTxType {
		return s.eip2930Signer.Hash(tx)
	}
	return prefixedRlpHash(
		tx.Type(),
		[]interface{}{
			s.chainId,
			tx.Nonce(),
			tx.GasTipCap(),
			tx.GasFeeCap(),
			tx.Gas(),
			tx.To(),
			tx.Value(),
			tx.Data(),
			tx.AccessList(),
		})
}
```
其中，`tx.Type()`有以下几种情况:
```go
const (
	LegacyTxType = iota
	AccessListTxType
	DynamicFeeTxType
)
```
为了降低复杂度，我们自此不再讨论`tx.Type() != DynamicFeeTxType`的情况，这种情况并不符合`EIP-1599`。从代码中可以看出核心函数为`prefixedRlpHash`，其定义如下:
```
func prefixedRlpHash(prefix byte, x interface{}) (h common.Hash) {
	sha := hasherPool.Get().(crypto.KeccakState)
	defer hasherPool.Put(sha)
	sha.Reset()
	sha.Write([]byte{prefix})
	rlp.Encode(sha, x)
	sha.Read(h[:])
	return h
}
```
简化来说，此代码首先完成了在`sha`变量内写入`0x02`标识符，此标识符表示该交易符合`EIP1599`。

然后，继续在`sha`变量内写入交易数据的`rlp`编码，如下:
```
rlp([chain_id, nonce, max_priority_fee_per_gas, max_fee_per_gas, gas_limit, destination, amount, data, access_list])
```
最后进行哈希计算。

> `rlp`编码相对复杂，此处不再解释。一个较好的学习材料是[Ethereum Doc](https://ethereum.org/en/developers/docs/data-structures-and-encoding/rlp)，它的源代码位于[这里](https://github.com/ethereum/go-ethereum/tree/master/rlp)

获得符合要求的哈希值后，我们只需要对此哈希值按照上述方法进行签名即可，注意我们需要`V`、`R`、`S`依次进行编码加入交易数据即可。其中， `V`和`R`长度为32 bytes, `S`长度为1 bytes.

所有流程可以用下图表示:
![sign](https://s-bj-3358-blog.oss.dogecdn.com/svg/mermaid-diagram-2022-08-04-192726.svg)

验证签名只需要将上述过程反过来进行，较为简单，此处不再赘述。
## 链上签名验证

### 基本代码

请读者阅读以下代码:
```solidity
function recoverSignerFromSignature(uint8 v, bytes32 r, bytes32 s, bytes32 hash) external {
    address signer = ecrecover(hash, v, r, s);
    require(signer != address(0), "ECDSA: invalid signature");
}
```

这是使用`ecrecover`的一个代码示例，基本展示了我们使用`ecrecover`的基本方法，但在实际中，我们需要更多的代码确保安全。**不要将此代码用于生产环境**。我们会在本文的最后给出生产环境中应使用的代码。

通过上文学习，相信读者很快就能判断出`ecrecover`的作用，该指令接受`hash`、`v`、`r`和`s`用于恢复签名者公钥。

特殊的一点是`ecrecover`是一个预编译的EVM指令。预编译意味着智能合约的通用功能已被EVM底层实现，运行EVM的节点可以更加有效地运行含有此类代码的智能合约，而且与自己实现的代码相比，性能更高且消耗的`gas`费更少。

你可以在[EVM Codes](https://www.evm.codes/precompiled#0x01)查询到此指令。

### 适用范围

`ecrecover`适合所用使用标准`secp256k1`曲线签名的数据，但我们建议使用以太坊目前最常用的签名API进行签名:

- eth_sign
- personal_sign
- EIP-712

以上API均在`metamask`中进行了实现，具体使用方法可以参考[文档](https://docs.metamask.io/guide/signing-data.html)。

其中`EIP712`属于一种较新的签名标准，在`metamask`中的实现为`signTypedData_v4`。我们会在后文详细讨论此标准及其应用。

`eth_sign`用于对任何数据进行签名，但不建议开发者使用它完成非交易数据签名。这会导致严重的安全问题。

`personal_sign`也可以对任一数据进行签名，但使用此API会自动在代签数据前加入前缀`\x19Ethereum Signed Message:\n`，以防止此签名被用于交易。目前常用于网站的登录。

目前，`metamask`推荐开发者使用`EIP712`进行数据签名，而且`metamask`还对这种方法进行特殊支持，如UI显示等。另一方面，`EIP712`在复杂数据交互方面具有先天优势，索引开发者应尽可能使用`EIP712`。

## EIP712

对于`DApp`签名相关问题，以太坊标准已经有`EIP712`进行了相关规范。我们之前在[MetaMask一键登录设计](https://hugo.wongssh.cf/posts/metamask-login/)已经讨论过链下`EIP712`。但其实`EIP712`主要解决了链上智能合约签名与验证的相关问题。
### 签名

`EIP712`的核心内容是结构化数据的哈希计算，与一般的交易数据不同，`EIP712`主要面向`DApp`等相关产品，如果采用与交易数据相同的签名模式，可能导致在不同`DApp`之间签名被盗用。比如A产品使用一组数据进行哈希签名，而B产品也选择了与A产品相同的数据结构，这意味着你在A产品内进行的非交易签名可以在B产品内使用。这极有可能造成严重的财产问题。

为了解决此问题，`EIP712`对结构化数据的哈希使用了以下公式:
```
hashStruct(s : 𝕊) = keccak256(typeHash ‖ encodeData(s))
typeHash = keccak256(encodeType(typeOf(s)))
```
上述公式中`𝕊`代表结构化类型数据集中的所有实例

为了方便读者理解`EIP712`进行结构化数据的流程，我们在此给出一个结构化数据:
```solidity
struct Mail {
    address from;
    address to;
    string contents;
}
```

下述给出的`‖`代表字节拼接。

`encodeType`要求数据被编码为`type ‖ " " ‖ name`。上述示例应被编码为`Mail(address from,address to,string contents)`，与我们在[使用多种方式编写可升级的智能合约(下)](https://hugo.wongssh.cf/posts/foundry-contract-upgrade-part2/)中讨论的函数选择器字符串编码的规则类似。此处应该注意`type`必须是`solidity`规定的[数据类型](https://docs.soliditylang.org/en/latest/types.html)。

> 上述对`type`的描述较为简陋，但一般情况下可以理解为就是`solidity`中的数据类型，实际上，不是所有的数据类型都可以编码在`EIP712`中，具体情况可以参考[标准定义](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-712.md#definition-of-typed-structured-data-%F0%9D%95%8A)

`encodeData`定义为`enc(value₁) ‖ enc(value₂) ‖ … ‖ enc(valueₙ)`。`enc`代表编码函数，要求数据编码成`32 bytes`，编码要求大致如下:

- 布尔值使用`uint256`表示，`0`代表`false`， `1`代表`true`
- `address`编码为`uint160`
- `uint`统一编码为`uint256`，并使用[大端顺序](https://zh.wikipedia.org/wiki/%E5%AD%97%E8%8A%82%E5%BA%8F#%E5%A4%A7%E7%AB%AF%E5%BA%8F)排序
- `bytes1`到`bytes32`数据类型使用0填充为`bytes32`
- `bytes`和`string`均进行`keccak256`后编码
- `array`类型数据进行`keccak256`后编码
- `struct`类型数据递归调用`hashStruct`函数

上述规则与`abi`[编码规则](https://docs.soliditylang.org/en/latest/abi-spec.html#formal-specification-of-the-encoding)是基本一致的，但在`string`等类型上存在区别，请注意对比。

> 注意`bytes1`,`bytes2`,...,`bytes32`等数据类型属于不可变的长度固定[静态类型](https://docs.soliditylang.org/en/latest/types.html#fixed-size-byte-arrays)，而`bytes`则属于长度可变的[动态类型](https://docs.soliditylang.org/en/latest/types.html#dynamically-sized-byte-array)，注意区分。

经过上述步骤我们其实仍没有解决签名盗用问题，为了解决这一问题，`EIP712`要求最终的需要待签数据应该为以下形式:
```
encode(domainSeparator : 𝔹²⁵⁶, message : 𝕊) = "\x19\x01" ‖ domainSeparator ‖ hashStruct(message)
```
在此处，`EIP712`引入了一个重要的待签字段`domainSeparator`，它的定义如下:
```
domainSeparator = hashStruct(eip712Domain)
```
`eip712Domain`规定由以下字段构成:

- `string name`，具有可读性的合约名称等
- `string version`，当前交互的`dapp`或合约的版本，不同版本之间的签名不能混用
- `uint256 chainId`，区块链链ID，参考[此网站](https://chainlist.org/)
- `address verifyingContract`，验证签名的合约，可以保证签名仅被单一合约使用，选填
- `bytes32 salt`，加盐，可选填

> 注意不应更改或删除上述字段，如果你有新的建议，请提出`EIP`请求。

我们在此处列出一个简单的流程图解释上述过程:

![EIP712 Flow](https://s-bj-3358-blog.oss.dogecdn.com/svg/mermaid-diagram-2022-08-08-110929.svg)

在上图中，我们省略了`encodeData`的详细情况。

标准中也给出了与对应接口`eth_signTypedData`交互时应使用的`json-schema`:
```javascript
{
  type: 'object',
  properties: {
    types: {
      type: 'object',
      properties: {
        EIP712Domain: {type: 'array'},
      },
      additionalProperties: {
        type: 'array',
        items: {
          type: 'object',
          properties: {
            name: {type: 'string'},
            type: {type: 'string'}
          },
          required: ['name', 'type']
        }
      },
      required: ['EIP712Domain']
    },
    primaryType: {type: 'string'},
    domain: {type: 'object'},
    message: {type: 'object'}
  },
  required: ['types', 'primaryType', 'domain', 'message']
}
```
我们在此给出一个完整的示例:
```javascript
{
	domain: {
		// Defining the chain aka Rinkeby testnet or Ethereum Main Net
		chainId: 1,
		// Give a user friendly name to the specific contract you are signing for.
		name: 'Ether Mail',
		// If name isn't enough add verifying contract to make sure you are establishing contracts with the proper entity
		verifyingContract: '0xCcCCccccCCCCcCCCCCCcCcCccCcCCCcCcccccccC',
		// Just let's you know the latest version. Definitely make sure the field name is correct.
		version: '1',
	},

	// Defining the message signing data content.
	message: {
		/*
		- Anything you want. Just a JSON Blob that encodes the data you want to send
		- No required fields
		- This is DApp Specific
		- Be as explicit as possible when building out the message schema.
		*/
		contents: 'Hello, Bob!',
		attachedMoneyInEth: 4.2,
		from: {
		name: 'Cow',
		wallets: [
			'0xCD2a3d9F938E13CD947Ec05AbC7FE734Df8DD826',
			'0xDeaDbeefdEAdbeefdEadbEEFdeadbeEFdEaDbeeF',
		],
		},
		to: [
		{
			name: 'Bob',
			wallets: [
			'0xbBbBBBBbbBBBbbbBbbBbbbbBBbBbbbbBbBbbBBbB',
			'0xB0BdaBea57B0BDABeA57b0bdABEA57b0BDabEa57',
			'0xB0B0b0b0b0b0B000000000000000000000000000',
			],
		},
		],
	},
	// Refers to the keys of the *types* object below.
	primaryType: 'Mail',
	types: {
		// TODO: Clarify if EIP712Domain refers to the domain the contract is hosted on
		EIP712Domain: [
			{ name: 'name', type: 'string' },
			{ name: 'version', type: 'string' },
			{ name: 'chainId', type: 'uint256' },
			{ name: 'verifyingContract', type: 'address' },
		],
		// Not an EIP712Domain definition
		Group: [
			{ name: 'name', type: 'string' },
			{ name: 'members', type: 'Person[]' },
		],
		// Refer to PrimaryType
		Mail: [
			{ name: 'from', type: 'Person' },
			{ name: 'to', type: 'Person[]' },
			{ name: 'contents', type: 'string' },
		],
		// Not an EIP712Domain definition
		Person: [
			{ name: 'name', type: 'string' },
			{ name: 'wallets', type: 'address[]' },
		],
	},
}
```

另一个示例可以参考我之前的博客[MetaMask一键登录设计](https://hugo.wongssh.cf/posts/metamask-login/)。此博客使用了`EIP712`开发了一个链下登陆系统。

对于`EIP712`标准，大多数钱包都进行了实现，此处我们主要介绍`MetaMask`钱包。该钱包提供了`signTypedData_v4`方法以支持`EIP712`，读者可自行阅读(文档)[https://docs.metamask.io/guide/signing-data.html#sign-typed-data-v4]

### 验证

通过上文，我们得到了`EIP712`签名的基本形式:
```
"\x19\x01" ‖ domainSeparator ‖ hashStruct(message)
```
一般情况下，智能合约接收到的并不是完整的签名而是签名后的`v, r, s`数据和发送的结构体。故而我们需要先进行签名重建，一个简单的例子如下，你可以在[这里](https://github.com/wangshouh/upgradeContractLearn/blob/master/src/EIP-712/simple.sol)找到完整代码:
```
function verify(Mail mail, uint8 v, bytes32 r, bytes32 s) internal view returns (bool) {
	// Note: we need to use `encodePacked` here instead of `encode`.
	bytes32 digest = keccak256(abi.encodePacked(
		"\x19\x01",
		DOMAIN_SEPARATOR,
		hash(mail)
	));
	return ecrecover(digest, v, r, s) == mail.from.wallet;
}
```
用户将`Mail`结构体和`v, r, s`数据发送给智能合约，智能合约使用以上数据调用`verify`进行验证。在验证时，首先进行签名重建，在检验签名者是否为`mail`的发送者。

关于`hash`函数读者可自行查阅源代码，注意此文中给出的`hash`函数均符合以下公式:
```
hashStruct(s : 𝕊) = keccak256(typeHash ‖ encodeData(s))
typeHash = keccak256(encodeType(typeOf(s)))
```
在此举出一个例子:
```solidity
struct Person {
	string name;
	address wallet;
}
bytes32 constant PERSON_TYPEHASH = keccak256( 
	"Person(string name,address wallet)"
);
function hash(Person memory person) internal pure returns (bytes32) {
	return keccak256(abi.encode(
		PERSON_TYPEHASH,
		keccak256(bytes(person.name)),
		person.wallet
	));
}
```
在进行`hash`时，首先使用`PERSON_TYPEHASH`完成`typeHash`定义。然后再结构化各个数据。比如对`string`类型的`person.name`进行`keccak`转换。 读者可能发现似乎没有对`person.wallet`进行转换，这是因为`abi.encode`会自动进行此类转换。

> 默认的`abi.encode`会对`string`计算`utf-8`编码，而`EIP712`规定为计算`keccak`，所以此处需要手动转换。判断是否需要手动转换的方法就是对比`abi`的`encode`规则与`EIP712`的规定是否一致。前者请查阅[Formal Specification of the Encoding](https://docs.soliditylang.org/en/latest/abi-spec.html#formal-specification-of-the-encoding)，后者请查阅[Definition of encodeData](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-712.md#definition-of-encodedata)

上述代码使用了`abi.encode`方法，该方法会自动转换所有变量使其符合编码规则并进行拼接操作。但是，根据以下公式:
```
encode(domainSeparator : 𝔹²⁵⁶, message : 𝕊) = "\x19\x01" ‖ domainSeparator ‖ hashStruct(message)
```
最终的代签数据中包含`\x19\x01`数据，如果使用`abi.encode`进行拼接，那么`\x19\x01`会被转换为`bytes32`形式，进而导致生成的哈希值错误。为了避免这一问题，我们使用了`abi.encodePacked`方法。此方法不会对`string`进行标准`abi`编码，而仅仅将其转换为`utf-8`。使用此函数可以保证最终生成的哈希值正确的。

如果你仍无法理解上述内容，请参考[Solidity Tutorial: all about ABI](https://coinsbench.com/solidity-tutorial-all-about-abi-46da8b517e7)

> 有读者可能疑惑为什么不直接发送完整的签名，因为智能合约往往需要`mail`中的数据进行下一步操作。如果直接发送签名，则是哈希后的数据，无法实现数据的重现。

## 一个例子

我们在上述流程中，我们已经完成了一个简单的项目。但我们会在此节使用`openzeppelin`简化这一流程并提高安全性。

本项目的所有代码均放在[Github仓库](https://github.com/wangshouh/upgradeContractLearn/tree/master/src/EIP-712)中找到。

我们此处仍使用此数据作为待签数据:
```javascript
{
	contents: 'Hello, Bob!',
	from: {
		name: 'Cow',
		wallet: '0xCD2a3d9F938E13CD947Ec05AbC7FE734Df8DD826',
	},
	to: {
		name: 'Bob',
		wallet: '0xbBbBBBBbbBBBbbbBbbBbbbbBBbBbbbbBbBbbBBbB',
	}
}
```

### 合约编写

首先编写接口`IProduct.sol`。首先考虑数据结构，我们可以把`from`和`to`中的结构化数据抽离为`Person`结构体，把整体数据抽象为`Mail`结构体。然后，我们需要设计函数接口，在此处我们仅需要`verify`函数对外暴露，此函数仅需要`Mail`和`signature`参数。

具体代码如下:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

interface IProduct {
    struct Person {
        string name;
        address wallet;
    }

    struct Mail {
        Person from;
        Person to;
        string contents;
    }

    function verify(Mail memory mail, bytes memory signature) external;
}
```

我们需要对此接口进行实现。首先完成合约的初始化，具体代码如下:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import "openzeppelin-contracts/contracts/utils/cryptography/draft-EIP712.sol";
import "openzeppelin-contracts/contracts/utils/cryptography/ECDSA.sol";
import "./IProduct.sol";

contract ProductEIP712 is EIP712, IProduct {
	constructor(string memory _name, string memory _version)
        EIP712(_name, _version)
    {}
}
```
上述代码主要实现了`EIP712Domain`的初始化。值得注意的是，`openzeppelin`提供的`EIP712Domain`仅提供以下属性:
```
bytes32 typeHash = keccak256(
	"EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"
);
```
在`EIP712`合约中，最关键的函数为:
```solidity
function _hashTypedDataV4(bytes32 structHash) internal view virtual returns (bytes32) {
	return ECDSA.toTypedDataHash(_domainSeparatorV4(), structHash);
}
```
此函数的返回值正是完整的`EIP712`哈希值结果。

我们需要生成此函数所需要的全部参数，`_domainSeparatorV4()`函数是库提供的，定义如下:
```solidity
function _domainSeparatorV4() internal view returns (bytes32) {
	if (address(this) == _CACHED_THIS && block.chainid == _CACHED_CHAIN_ID) {
		return _CACHED_DOMAIN_SEPARATOR;
	} else {
		return _buildDomainSeparator(_TYPE_HASH, _HASHED_NAME, _HASHED_VERSION);
	}
}
```
对此函数，我们不需要提供额外参数。

对于`structHash`参数，我们则需要自己进行计算。

> 读者可能疑惑为什么智能合约中的函数和`MetaMask`的函数带有`V4`后缀，这是因为`EIP712`在形成标准期间进行过多次变化，`V4`是其最后标准。你可以查阅[此注释](https://github.com/MetaMask/eth-sig-util/blob/main/src/sign-typed-data.ts#L36)了解更多。如果没有特殊情况，建议使用`V4`标准。

综上所述，我们需要定义一个符合`EIP712`规定的`hashStruct`函数。参考上文给出的计算方法，由于递归属性，我们需要首先定义`Person`的`hashStruct`，然后定义`Mail`的`hashStruct`，代码如下:
```solidity
bytes32 constant PERSON_TYPEHASH =
	keccak256("Person(string name,address wallet)");

bytes32 constant MAIL_TYPEHASH =
	keccak256(
		"Mail(Person from,Person to,string contents)Person(string name,address wallet)"
	);
function hash(Person memory person) internal pure returns (bytes32) {
	return
		keccak256(
			abi.encode(
				PERSON_TYPEHASH,
				keccak256(bytes(person.name)),
				person.wallet
			)
		);
}

function hash(Mail memory mail) public pure returns (bytes32) {
	return
		keccak256(
			abi.encode(
				MAIL_TYPEHASH,
				hash(mail.from),
				hash(mail.to),
				keccak256(bytes(mail.contents))
			)
		);
}
```

上述代码符合以下公式:
```
hashStruct(s : 𝕊) = keccak256(typeHash ‖ encodeData(s))
typeHash = keccak256(encodeType(typeOf(s)))
```
有了上述参数，我们最终可以编写`verify`函数，具体定义如下:
```solidity
function verify(Mail memory mail, bytes memory signature) public {
	bytes32 digest = _hashTypedDataV4(hash(mail));
	address signer = ECDSA.recover(digest, signature);
	require(signer == msg.sender, "Not sender");
}
```
在此处，我们使用了安全性更高的`ECDSA.recover`函数。

### 编写测试

一种方案是由`foundry`官方文档提供的，读者可以自行参考[文档](https://book.getfoundry.sh/tutorials/testing-eip712)。

另一种方案比较神奇，我们从浏览器端获得签名结果，然后前往测试合约中进行测试。

编写测试合约，在`test/EIP712/product.t.sol`中输入:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../../src/EIP-712/product.sol";
import "../../src/EIP-712/IProduct.sol";

contract ProductTest is Test {
    ProductEIP712 private product;

    function setUp() public {
        product = new ProductEIP712("Ether Mail", "1");
    }

    function testArg() public {
        vm.chainId(4);
        console2.log(address(product));
    }
}
```

上述合约主要为我们提供一些必要的参数，主要是`verifyingContract`参数。当然，在此处，我们使用`vm.chainId(4);`将链ID改为了`4`，即`Rinkeby`网络。

在终端内使用`forge test -vvv`，获得以下输出:
![Contract Address](https://img.wang.232232.xyz/img/2022/08/15/verifyContractba8deb0427d1b34b.png)

前往[此网页](https://metamask.github.io/test-dapp/)，首先点击`Connected`链接钱包，然后点击`ETH_ACCOUNTS`查看地址是否正确。按下`F12`打开开发者工具，进入`Console`终端。

键入`window.ethereum`，返回值应不为空。

首先设置待签数据，输入下文代码:
```javascript
const msgParams = JSON.stringify({
  domain: {
    name: 'Ether Mail',
	chainId: 4,
	version: '1',
    verifyingContract: '0xce71065d4017f316ec606fe4422e11eb2c47c246',
  },

  message: {
    contents: 'Hello, Bob!',
    from: {
      name: 'Cow',
      wallet: '0xCD2a3d9F938E13CD947Ec05AbC7FE734Df8DD826',
    },
    to: {
        name: 'Bob',
        wallet:
          '0xbBbBBBBbbBBBbbbBbbBbbbbBBbBbbbbBbBbbBBbB',
      },
  },
  primaryType: 'Mail',
  types: {
    EIP712Domain: [
      { name: 'name', type: 'string' },
      { name: 'version', type: 'string' },
      { name: 'chainId', type: 'uint256' },
      { name: 'verifyingContract', type: 'address' },
    ],

    Mail: [
      { name: 'from', type: 'Person' },
      { name: 'to', type: 'Person' },
      { name: 'contents', type: 'string' },
    ],
    Person: [
      { name: 'name', type: 'string' },
      { name: 'wallet', type: 'address' },
    ],
  },
});
```
其中，`chainId`请根据页面显示确定。`verifyingContract`地址根据上文终端输出决定。

使用下述命令进行签名:
```javascript
const sign = await ethereum.request({
	method: 'eth_signTypedData_v4',
	params: ["0x11475691C2CAA465E19F99c445abB31A4a64955C", msgParams],
});
```
其中，`params`第一个参数应改为个人地址。

键入`sign`，输出结果如下:
```
0x70aa843f69e5d32252c65011b34831e79c9c64752134d9318cdefb7f8d7a04ac08a2193aedb8f329a8d80f5390c7f661fe447ccc9337ebed15b578c01d7dc71e1c
```
获得签名结果后，我们继续进行测试合约的编写，加入以下函数:
```solidity
function testVerify() public {
	vm.chainId(4);
	vm.prank(0x11475691C2CAA465E19F99c445abB31A4a64955C);

	bytes
		memory Signature = hex"70aa843f69e5d32252c65011b34831e79c9c64752134d9318cdefb7f8d7a04ac08a2193aedb8f329a8d80f5390c7f661fe447ccc9337ebed15b578c01d7dc71e1c";
	IProduct.Person memory PersonFrom = IProduct.Person({
		name: "Cow",
		wallet: address(0xCD2a3d9F938E13CD947Ec05AbC7FE734Df8DD826)
	});
	IProduct.Person memory PersonTo = IProduct.Person({
		name: "Bob",
		wallet: address(0xbBbBBBBbbBBBbbbBbbBbbbbBBbBbbbbBbBbbBBbB)
	});
	IProduct.Mail memory mail = IProduct.Mail({
		contents: "Hello, Bob!",
		from: PersonFrom,
		to: PersonTo
	});
	product.verify(mail, Signature);
	vm.stopPrank();
}
```
其中，`vm.prank(address)`用于切换地址。

最后，在终端内输入`forge test`，结果如下:
![signTestResult.png](https://img.wang.232232.xyz/img/2022/08/15/signTestResult0019f7aabd49eab5.png)

## 总结

本文介绍了基本的`EIP712`的原理和相关链上和链下合约。我们会在未来编写一篇介绍了`EIP712`的进一步应用的文章。

## 拓展阅读

1. [比特币 P2TR 交易详解](https://www.btcstudy.org/2022/06/13/part-2-bitcoin-p2tr-transaction-breakdown)，介绍了比特币使用的`ECDSA`和`schnorr`多签名等内容
1. [A (Relatively Easy To Understand) Primer on Elliptic Curve Cryptography](https://blog.cloudflare.com/a-relatively-easy-to-understand-primer-on-elliptic-curve-cryptography)，由Cloudflare编写的博客，介绍了部分历史和椭圆函数密码学在网络安全中的应用
1. [以太坊技术与实现 签名与校验](https://learnblockchain.cn/books/geth/part3/sign-and-valid.html)，完整介绍了以太坊的签名和验证等相关问题，同时给出`go`编写的以太坊底层代码
1. [以太坊签名验签原理揭秘](https://mirror.xyz/0x9B5b7b8290c23dD619ceaC1ebcCBad3661786f3a/jU9qUqkhF5PAG_TXIB0Mb481-cGaaaaTAvAF8FaHt40)，此文较短但也基本全面介绍了以太坊内的签名和验证，可以作为上一篇文章的缩减版
1. [Elliptic Curve Digital Signature Algorithm](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm)，维基百科，给出了完整的数学证明
