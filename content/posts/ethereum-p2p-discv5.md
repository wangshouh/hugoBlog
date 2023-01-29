---
title: "以太坊执行层P2P网络架构与设计:Discv5"
date: 2022-09-11T11:47:33Z
tags: [P2P,RLP,ethereum,EIP-778,EIP-1459,Kademlia]
aliases: ["/2022/09/11/ethereum-p2p-discv5/"]
math: true
---

## 概述

关于以太坊的`P2P`网络问题，目前的资料较为零散，本文尝试结合具体的`go-ethereum`源代码，尽可能为读者完整介绍以太坊所使用的`ÐΞVp2p`(`devp2p`)网络架构和运转流程。

`devp2p`各协议栈之间的关系可以参考下图:

![DevP2P stack](https://img-blog.csdnimg.cn/img_convert/7ae6b746c9822e8edfba97202ef66525.png)

其中，`Node Discovery Protocol v5`运行在`UDP`上，其余均运行在`TCP`协议上。

本文会以节点第一次进入以太坊P2P网络的流程为主线，介绍`Node Discovery Protocol v5`中各个模块的构成和作用。

## 前置知识

本文将介绍的很多组件均依赖于RLP，所以在此处我们对其首先进行介绍

### RLP 编码

在以太坊中，我们使用的最基本的编码格式就是`RLP`编码(递归长度前缀解码)，这是一种特殊的序列化形式。`RLP`可以对以下内容进行编码:

- 字符串(`String`)，在编码过程中会被转换为纯粹的二进制数据字节
- 列表(`list`)

为了方便读者理解`RLP`编码的具体功能和实现，我们将采用先介绍解码，再介绍编码的形式。

对于`RLP`的解码，流程如下:

1. 根据输入的第一个字节(即前缀)判断解码的数据类型、实际数据的长度和偏移量
1. 根据数据类型和偏移量进行解码
1. 根据第一步返回的实际数据长度寻找下一个前缀并重复解码流程

上述流程也说明了此编码协议的名称的由来。读者可以通过此工具为[RLP online](https://toolkit.abdk.consulting/ethereum#rlp)辅助学习。

接下来我们介绍二进制字节解码的规则:

1. 前缀在`[0x00, 0x7f]`范围内，我们认为其自身就是数据，且解码类型为单个二进制字节
1. 前缀为`0x80`，解码数据为空值(如`uint(0)`、`""`等)
1. 前缀为`[0x81, 0xb7]`，解码数据为字符串且前缀后跟长度等于第一个字节减去 0x80 的二进制字节
1. 前缀为`[0xb8, 0xbf]`，解码数据为二进制字节，读取前缀后将前缀与`0xb7`相减获得第二部分的长度，然后读取第二部分的长度即二进制字节

接下来，我们给出具体的实例:
```
0x8b68656c6c6f20776f726c64
```
对于上述内容进行解码，首先我们读取第一字节`0x8b`，发现此字节符合条件3，我们协议计算字符串长度为`0x8b - 0x80`，即 `11 bytes`，这意味着字符串的具体长度为`11 bytes`。我们依照此长度读取`11 bytes`，就获得了解码后的数据`0x68656c6c6f20776f726c64`。

> 如果你可以确认此数据的编码格式就可以进行下一步编码，在此处我们给出的数据使用了`UTF-8`编码，[转换](https://onlineutf8tools.com/convert-hexadecimal-to-utf8)后为`hello world`。

另一个例子如下:
```
0xb90400(61)*1024
```
`(61)*1024`的意思为`61`连续重复1024次。依照上述流程，我们首先读取前缀`0xb9`，发现其大于`0xb8`，相减后获得`0xb9 - 0xb7`，即`2 bytes`。我们读取后`2 bytes`，即`0400`(1024)，这是最终的二进制字节长度，我们再读取`1024 bytes`，获得`(61) * 1024`。

> 同理，此处我们也使用了`utf-8`编码，转换后为A

综合使用上述规则，我们可以编码长度低于`65536 bytes`的二进制字节。

接下来，我们介绍列表的解码规则:

1. 前缀为`0xc0`，解码数据为空列表
1. 前缀为`[0xc1, 0xf7]`，解码数据为列表，且长度为前缀与`0xc0`的差
1. 前缀为`[0xf8, 0xff]`， 解码数据为列表，读取前缀后将前缀与`0xf7`相减获得第二部分的长度，然后读取第二部分的长度即列表长度

此规则与二进制字节的解码规则类似。

一个简单的例子:
```
0xcc 8568656c6c6f 85776f726c64
```

>上述示例加空格只为了更好划分结构

我们首先读取前缀`0xcc`，发现此数据结构为列表且长度为`0xcc - 0xc0`，即`12`。获得此消息后我们对后文数据进行读取，后文数据为两个字符串，读者可以根据上文给出的方法自行解码。将最后的数据使用`utf-8`解码后获得`["hello","world"]`

如果读者对代码实现感兴趣，我非常建议阅读`go-ethereum`的[源代码](https://github.com/ethereum/go-ethereum/tree/master/rlp)。

### ECDH

`ECDH`是一种基于椭圆函数曲线的`Diffie–Hellman`密钥协商算法。通过此算法，两个节点可以在不安全的信道中交换生成对称密码学的密钥。为方便对算法实现进行介绍，我们假设节点A与节点B需要进行`ECDH`生成对称密码学密钥，同时节点A和节点B都可以简单获得对方的公钥。

交换流程如下:

1. 节点查询对方的公钥，并与自己的私钥相乘
1. 获得的乘法结果即为对称密码学密钥

证明如下:

节点A的公钥为 $Q_A = d_A\cdot G$ ，其中 $d_A$ 为节点A的私钥

节点B的公钥为 $Q_B = d_B\cdot G$ ，其中 $d_B$ 为节点A的私钥

依照上述给出的流程，结果如下:

节点A生成的密钥为 $d_A\cdot Q_B = d_A\cdot d_B\cdot G$

节点B生成的密钥为 $d_B\cdot Q_A = d_B\cdot d_A\cdot G$

显然，生成的密钥是相同的，完成了密钥的交换。

### HKDF

`HKDF`是一种基于`HMAC`的密钥推导算法，由[RFC5869](https://www.rfc-editor.org/rfc/rfc5869)进行了相关定义。其功能为给定随机密钥生成材料(如上文获得的`ECDH`对称密钥)，经过`HKDF`函数可以获得给定长度的具有高密码学强度的密钥。简单来说，可以将此算法看作密钥长度转换算法。

在了解`HKDF`前，我们首先需要知道`HMAC`的运作原理，如下图:

![HMAC](https://img-blog.csdnimg.cn/img_convert/7e1031135ab9ed0bc753d2fc9f866f1a.png)

我们在此处不详细介绍具体流程。读者若感兴趣，建议阅读[《图解密码技术》 第三版](https://book.douban.com/subject/26822106/)的 194 页部分，非常详细介绍了相关的计算流程。上图即来自此书。

在了解了`HMAC`流程后，我们就可以介绍`HKDF`的相关流程。我们以上文生成的密钥作为基础材料，记为`secret`，并使用 SHA256 作为哈希算法，目标是获得长度为 256 位的标准密钥。

第一步，将`secret`作为信息，使用给定或随机生成的`salt`作为密钥进行`HMAC`计算获得`MAC`值
第二步，我们需要获得 256 位的密钥，计算密钥长度与哈希算法输出值之间的比值并向上取整，此数值记为`n`。由于此处我们选择的哈希算法与密钥长度正好相同，所以计算结果为`n = 1`。接下来，我们进行以下计算:
```
T(0) = empty string (zero length)
T(1) = HMAC-Hash(PRK, T(0) | info | 0x01)
T(2) = HMAC-Hash(PRK, T(1) | info | 0x02)
T(3) = HMAC-Hash(PRK, T(2) | info | 0x03)
...
T(n) = HMAC-Hash(PRK, T(n-1) | info | hex(n))
```
将上述结果进行拼接`T = T(1) | T(2) | T(3) | ... | T(N)`，最后选择`T`的前 256 位作为密钥。

在`Go`语言中，我们可以使用以下`golang.org/x/crypto`中的官方实现:
```
func New(hash func() hash.Hash, secret, salt, info []byte) io.Reader
```
具体文档可参考[这里](https://pkg.go.dev/golang.org/x/crypto/hkdf#New)

## Node Discovery Protocol v5

作为一个全新的节点，加入以太坊网络最重要的一步就是尽可能与其他节点建立链接，完善自己的节点列表。下图表示了发现以太坊网络节点的流程:

![Peer Discovery](https://s-bj-3358-blog.oss.dogecdn.com/svg/mermaid-diagram-2022-09-13-224506.svg)

我们会逐一解释其中每一个过程使用的具体协议和协议定义。

### 初始节点获得

当我们启动一个全新节点时，我们会与`go-ethereum`规定的启动节点进行通信，这些节点的地址被硬编码在`go-ethereum`源代码中，读者可以前往[此处](https://github.com/ethereum/go-ethereum/blob/master/params/bootnodes.go)查看。我们可以看到其中有几种典型的编码格式，如下:
1. 形如`enode://d860a01f9722d78051619d1e2351aba3f43f943f6f00718d1b9baa4101932a1f5011f16bb2b1bb35db20d6fe28fa0bf09636d26a87d31de9ec6203eeedb1f666@18.138.108.67:30303`，其中`d860a01f9722d78051619d1e2351aba3f43f943f6f00718d1b9baa4101932a1f5011f16bb2b1bb35db20d6fe28fa0bf09636d26a87d31de9ec6203eeedb1f666`为节点运营者的未压缩公钥，其余数据为IP地址和端口号

1. 形如`enr:-KG4QOtcP9X1FbIMOe17QNMKqDxCpm14jcX5tiOE4_TyMrFqbmhPZHK_ZPG2Gxb1GE2xdtodOfx9-cgvNtxnRyHEmC0ghGV0aDKQ9aX9QgAAAAD__________4JpZIJ2NIJpcIQDE8KdiXNlY3AyNTZrMaEDhpehBDbZjM_L9ek699Y7vhUJ-eAdMyQW_Fil522Y0fODdGNwgiMog3VkcIIjKA`，此形式使用了`ENR`编码格式，由[EIP778](https://eips.ethereum.org/EIPS/eip-778)进行了相关定义。

1. 形如`enrtree://AKA3AM6LPBYEUDMVNU3BSVQJ5AD45Y7YPOHJLEF6W26QOE4VTUDPE`，此形式使用了`enrtree`的编码格式，由[EIP1459](https://eips.ethereum.org/EIPS/eip-1459)进行定义，主要用于通过DNS查询对等节点

上述不同的编码格式中`enode`格式最简单且具有自解释性，我们不会在后文进行讨论。后文主要讨论`enr`和`enrtree`形式。读者可以通过`enode`中的公钥在[EtherNode](https://ethernodes.org/nodes)查询到所有节点的信息。

#### ENR

`ENR`相较于较好理解的`enode`格式由以下优势:

1. 可拓展性高，由于`ENR`使用了以太坊中通用的`RLP`编码，我们可以在其中加入其他数据
1. 安全性好，`ENR`要求用户用户在数据内加入签名，避免了伪造节点信息的情况

在编码中，我们需要依次编码以下数据:

- `signature` 签名，对于记录信息的签名
- `seq` 序列号，每当记录重新修改发布后一个增加此值
- 其他键值数据

此处的其他键值数据理论上可以随意填写，填写时应保证键在前、值在后，同时尽可以保证字符使用`ASCII`编码。但`EIP1459`对于以下字段进行了预定义:

| 键 | 值 |
| -- | --- |
| `id`(0x6964) | 当前节点身份方案的名称，如`v4` |
| `secp256k1`(0x736563703235366b31) | 压缩后的公钥( 33 bytes) |
| `ip`(0x6970) | 节点的 IP 地址 |
| `tcp`(0x746370) | 节点的 TCP 端口号|
| `udp`(0x756470) | 节点的 UDP 端口号|
| `ip6`(0x697036) | 节点的 IPv6 地址( 16 bytes ) |
| `tcp6`(0x74637036) | 节点 IPv6 地址使用的 TCP 端口号|
| `udp6`(0x75647036) | 节点 IPv6 地址使用的 UDP 端口号|

> 其中键后的括号表示其名称的`ASCII`编码结果

接下来，我们给出以下参数:

- 私钥 0xb71c71a67e1177ad4e901695e1b4b9ee17ae16c6668d313eac2f96dbcda3f291
- seq 01
- id "v4"
- ip 127.0.0.1
- udp port 30303

我们使用上述参数构建此节点的`enr`。

首先我们需要获得公钥，通过[此网站](https://toolkit.abdk.consulting/ethereum#key-to-address)，我们可以获得压缩公钥为`ca634cae0d49acb401d8a4c6b6fe8c55b70d115bf400769cc1400f3258cd3138`，未压缩公钥为`ca634cae0d49acb401d8a4c6b6fe8c55b70d115bf400769cc1400f3258cd31387574077f301b421bc84df7266c44e9e6d569fc56be00812904767bf5ccd1fc7f`

![Private Key to Pk](https://img-blog.csdnimg.cn/img_convert/9be49ea2db336c4f8fbbe681929a39e0.png)

> 在此处，我们删去`0x03`和`0x04`是因为此两者仅作为标识符，无实际意义。

获得公钥后，我们对上述信息进行编码，格式如下:
```
[
    "0x01", #此数值即`seq`
    "id",
    "v4",
    "ip",
    7f000001,
    "secp256k1",
    03ca634cae0d49acb401d8a4c6b6fe8c55b70d115bf400769cc1400f3258cd3138,
    "udp",
    765f, # `303030`的16进制形式
]
```

获得上述数据后，根据我们已经介绍过的的`RLP`进行编码(字符应使用`ASCII`转换为`16 进制`)，结果如下:
```
0xf84201826964827634826970847f00000189736563703235366b31a103ca634cae0d49acb401d8a4c6b6fe8c55b70d115bf400769cc1400f3258cd31388375647082765f
```
使用私钥对上述结果进行签名，获得签名结果`0x7098ad865b00a582051940cb9cf36836572411a47278783077011599ed5cd16b76f2635f4e234738f30813a89eb9137e3e3df5266e3a1f11df72ecf1145ccb9c27`。

![Recover signature](https://img-blog.csdnimg.cn/img_convert/e2e3e774006ec355286143763e63602a.png)

> 上图显示了自签名中恢复公钥

获得公钥后，我们需要重新进行信息编码，如下:
```
[
    7098ad865b00a582051940cb9cf36836572411a47278783077011599ed5cd16b76f2635f4e234738f30813a89eb9137e3e3df5266e3a1f11df72ecf1145ccb9c,
    01,
    "id",
    "v4",
    "ip",
    7f000001,
    "secp256k1",
    03ca634cae0d49acb401d8a4c6b6fe8c55b70d115bf400769cc1400f3258cd3138,
    "udp",
    765f,
]
```
进行`RLP`编码，如下:
```
f884b8407098ad865b00a582051940cb9cf36836572411a47278783077011599ed5cd16b76f2635f4e234738f30813a89eb9137e3e3df5266e3a1f11df72ecf1145ccb9c01826964827634826970847f00000189736563703235366b31a103ca634cae0d49acb401d8a4c6b6fe8c55b70d115bf400769cc1400f3258cd31388375647082765f
```

![RLP Decode](https://img-blog.csdnimg.cn/img_convert/a6a0cb5c310bad15ab61946f65c53946.png)

完成上述任务后，我们需要对最终结果进行`base64`编码，注意此处使用了特殊的`URL and Filename safe`[映射表](https://www.rfc-editor.org/rfc/rfc4648#section-5)，与一般的`base64`编码不同。如在`Python`中，普通的的`base64`使用`base64.b64encode`，而`URL and Filename safe`使用了`base64.urlsafe_b64encode`。

!["URL and Filename safe" Base 64 Alphabet](https://img-blog.csdnimg.cn/img_convert/cd3fcd3556b794de58a6e4d8dee755cc.png)

使用以下`Python`代码即可实现编码:
```python
>>> data = bytes.fromhex("f884b8407098ad865b00a582051940cb9cf36836572411a47278783077011599ed5cd16b76f2635f4e234738f30813a89eb9137e3e3df5266e3a1f11df72ecf1145ccb9c01826964827634826970847f00000189736563703235366b31a103ca634cae0d49acb401d8a4c6b6fe8c55b70d115bf400769cc1400f3258cd31388375647082765f
")
>>> base64.urlsafe_b64encode(data)
b'-IS4QHCYrYZbAKWCBRlAy5zzaDZXJBGkcnh4MHcBFZntXNFrdvJjX04jRzjzCBOonrkTfj499SZuOh8R33Ls8RRcy5wBgmlkgnY0gmlwhH8AAAGJc2VjcDI1NmsxoQPKY0yuDUmstAHYpMa2_oxVtw0RW_QAdpzBQA8yWM0xOIN1ZHCCdl8='
```
我们会丢掉最后结果中的占位`=`，同时在前加上`enr:`，最终达到以下结果:
```
enr:-IS4QHCYrYZbAKWCBRlAy5zzaDZXJBGkcnh4MHcBFZntXNFrdvJjX04jRzjzCBOonrkTfj499SZuOh8R33Ls8RRcy5wBgmlkgnY0gmlwhH8AAAGJc2VjcDI1NmsxoQPKY0yuDUmstAHYpMa2_oxVtw0RW_QAdpzBQA8yWM0xOIN1ZHCCdl8
```

读者可以前往[此处](https://sourcegraph.com/github.com/ethereum/go-ethereum/-/blob/p2p/enr/enr.go)阅读`go-ethereum`中的`ENR`实现。

一个有用的工具是[ENR to Enode](https://enr2maddr.herokuapp.com/)将`ENR`转换为更好理解的`Enode`形式。
#### DNSDisc

与固定的`Enode`和`ENR`相比，`DNSDisc`所代表的`Node Discovery via DNS`开创了一种全新的方法——通过DNS分发对等节点，使用这种方法有效避免了硬编码无法进行升级的情况。此方法由[EIP1459](https://eips.ethereum.org/EIPS/eip-1459)进行定义和规范化。

我们可以在[bootnodes](https://github.com/ethereum/go-ethereum/blob/master/params/bootnodes.go)中找到以下代码:
```go
const dnsPrefix = "enrtree://AKA3AM6LPBYEUDMVNU3BSVQJ5AD45Y7YPOHJLEF6W26QOE4VTUDPE@"

func KnownDNSNetwork(genesis common.Hash, protocol string) string {
	var net string
	switch genesis {
	case MainnetGenesisHash:
		net = "mainnet"
	case RopstenGenesisHash:
		net = "ropsten"
	case RinkebyGenesisHash:
		net = "rinkeby"
	case GoerliGenesisHash:
		net = "goerli"
	case SepoliaGenesisHash:
		net = "sepolia"
	default:
		return ""
	}
	return dnsPrefix + protocol + "." + net + ".ethdisco.net"
}
```
此处我们以主网为例进行介绍，使用主网，同时将协议`protocol`设置为`all`，调用`KnownDNSNetwork`函数，我们可以获得`enrtree://AKA3AM6LPBYEUDMVNU3BSVQJ5AD45Y7YPOHJLEF6W26QOE4VTUDPE@all.mainnet.ethdisco.net`输出。

我们会使用`parseLink`[函数](https://sourcegraph.com/github.com/ethereum/go-ethereum/-/blob/p2p/dnsdisc/tree.go?L342:6)解析此链接：
```go
func parseLink(e string) (*linkEntry, error) {
	if !strings.HasPrefix(e, linkPrefix) {
		return nil, fmt.Errorf("wrong/missing scheme 'enrtree' in URL")
	}
	e = e[len(linkPrefix):]
	pos := strings.IndexByte(e, '@')
	if pos == -1 {
		return nil, entryError{"link", errNoPubkey}
	}
	keystring, domain := e[:pos], e[pos+1:]
	keybytes, err := b32format.DecodeString(keystring)
	if err != nil {
		return nil, entryError{"link", errBadPubkey}
	}
	key, err := crypto.DecompressPubkey(keybytes)
	if err != nil {
		return nil, entryError{"link", errBadPubkey}
	}
	return &linkEntry{e, domain, key}, nil
}
```
以`@`为分界线，我们将`enrtree://AKA3AM6LPBYEUDMVNU3BSVQJ5AD45Y7YPOHJLEF6W26QOE4VTUDPE@all.mainnet.ethdisco.net`分解为以下三部分:

- `e` \- `AKA3AM6LPBYEUDMVNU3BSVQJ5AD45Y7YPOHJLEF6W26QOE4VTUDPE@all.mainnet.ethdisco.net`，即除`enrtree`外的完整字符串

- `domin` \- `all.mainnet.ethdisco.net`，即域名部分

- `key` \- `AKA3AM6LPBYEUDMVNU3BSVQJ5AD45Y7YPOHJLEF6W26QOE4VTUDPE`使用`base32`解码(`b32format.DecodeString`)后并解压缩(`DecompressPubkey`)的`0x0281b033cb78704a0d956d36195609e807cee3f87b8e9590beb6bd0713959d06f2`

相信读者也可以通过上述解析过程理解`enrtree//`的生成过程。简单来说，就是将`base32`编码的公钥与提供服务的域名结合起来。

当我们解析获得服务域名后，我们需要使用以下`resolveRoot`[函数](https://sourcegraph.com/github.com/ethereum/go-ethereum/-/blob/p2p/dnsdisc/client.go?L140):
```go
func (c *Client) resolveRoot(ctx context.Context, loc *linkEntry) (rootEntry, error) {
	e, err, _ := c.singleflight.Do(loc.str, func() (interface{}, error) {
		txts, err := c.cfg.Resolver.LookupTXT(ctx, loc.domain)
		c.cfg.Logger.Trace("Updating DNS discovery root", "tree", loc.domain, "err", err)
		if err != nil {
			return rootEntry{}, err
		}
		for _, txt := range txts {
			if strings.HasPrefix(txt, rootPrefix) {
				return parseAndVerifyRoot(txt, loc)
			}
		}
		return rootEntry{}, nameError{loc.domain, errNoRoot}
	})
	return e.(rootEntry), err
}
```
此函数完成以下功能:
1. 向DNS服务器发出`LookupTXT`请求
1. 检测返回值内是否包含`enrtree-root:v1`(`rootPrefix`)
1. 解析并验证(`parseAndVerifyRoot`函数)返回值

由于我们不知道`LookupTXT`的返回值，进行进一步讨论时极其抽象的，所以在此处，我们可以使用`dig all.mainnet.ethdisco.net TXT`命令进行一次真实的请求。如果读者没有此命令，也可以前往[DNSLookup Online](https://dnslookup.online/txt.html)进行测试，返回值如下:
```bash
;; ANSWER SECTION:
all.mainnet.ethdisco.net. 266   IN      TXT     
"enrtree-root:v1 e=WWP2ZHNHHZO3K5Y5T2VMGV7GXU l=FDXN3SN67NA5DKA4J2GOK7BVQI seq=3315 sig=YvkxX3lGYBRwpasJ7a73acOtQg_kbK51uFt2_1RQ4uR8Jq9dFD_fxE9WiZKNBZjgo8hmNcsqVoP82xHo3vos3gE"
```

> 读者可能无法获得的返回值与此处不同，因为此服务的DNS一直再进行更新。

其中，
| 缩写 | 全称 | 作用 |
| ---- | ---- | ---- |
| `e` | `enr-root` | 此DNS服务器维护的`enr merkle tree`的根节点
| `l` | `link-root` | 链接的其他DNS服务器的`enr merkle tree`的根节点
| `seq` | `sequence-number` | 服务器更新的次数
| `sig` | `signature` | 服务器维护者使用公钥的对`enrtree-root:v1 e=WWP2ZHNHHZO3K5Y5T2VMGV7GXU l=FDXN3SN67NA5DKA4J2GOK7BVQI seq=3315`的哈希值签名的结果

有了以上认识，我们可以分析`parseAndVerifyRoot`[函数](https://sourcegraph.com/github.com/ethereum/go-ethereum/-/blob/p2p/dnsdisc/client.go?L157)，代码如下:
```go
func parseAndVerifyRoot(txt string, loc *linkEntry) (rootEntry, error) {
	e, err := parseRoot(txt)
	if err != nil {
		return e, err
	}
	if !e.verifySignature(loc.pubkey) {
		return e, entryError{typ: "root", err: errInvalidSig}
	}
	return e, nil
}
```
其中，`e`结构体的定义如下:
```go
struct {
    eroot string
    lroot string
    seq uint
    sig []byte
}
```
我们使用DNS返回的TXT值中的`sig`和其他参数的哈希值进行公钥恢复操作，如果发现恢复出的公钥与`enrtree`中规定的不同，我们则认为此DNS不可信。

![enrtree Verify](https://img-blog.csdnimg.cn/img_convert/9fe28cabc4b6ddf03e0afc85c1050935.png)

> 关于公钥恢复的详细介绍可以参考我写的[基于链下链上双视角深入解析以太坊签名与验证]({{<ref "ecsda-sign-chain#%E9%AA%8C%E8%AF%81%E7%AD%BE%E5%90%8D" >}})

> 为什么需要设计`enr-root`和`link-root`两种根？
> 设计`link-root`的目的是方便大型列表的维护。由所有以太坊节点构成的`merkle tree`是巨大的，提供`link-root`指向其他DNS服务器可以有效分割大型`merkle tree`。但目前根节点给出的`link-root`似乎是失效的。

我们对`e=WWP2ZHNHHZO3K5Y5T2VMGV7GXU`代表的`Enr-root`为例，我们使用`dig WWP2ZHNHHZO3K5Y5T2VMGV7GXU.all.mainnet.ethdisco.net txt`对其查询`TXT`解析:
```
WWP2ZHNHHZO3K5Y5T2VMGV7GXU.all.mainnet.ethdisco.net. 1 IN TXT
"enrtree-branch:DT4G4IMIRQLGT55XIRYTC2JWQM,2TDZ2XP33G656AAUOJ3GVKUHJE"
```
此处获得了`enrtree-branch`前缀的返回值，此返回值为`merkle tree`的子树根节点。我们可以递归查询:
`DT4G4IMIRQLGT55XIRYTC2JWQM.all.mainnet.ethdisco.net` =>
```
enrtree-branch:
    3AZRES3OYSPP74C6OGTEEY3TBY, 57MTMJGUCEDELSGUBH65ZS54HE,33OL5MHULIAPCZBPSLICSLGEGU, 2TU2RXC2SQ5WQGNVD2QNDW45PI,HWPZ7Y56NWL7J7ZQFBNM4OSV6Y, IHZ66V3WAQLJVQAAQJTPCNK5NM,5NTCRHGPNY7CDOFCG2G44L7NCY, J4C7XS534WNRLWVWJ6LGYDKVMI,3ZWH74DECJXURPAKZQPA4BO5YA, NQDK5ZKHFVT25NBYJACBLKNYGA,OV3QA3PIGBGMHYMVZA2RES3Q7U, 3X5KEPA2RJ7MFBD7ZB4R52ZANI,47ALRD6236JBOYOHLAZTHY4VFA
```
我们选择第一个分支进行查询:
`3AZRES3OYSPP74C6OGTEEY3TBY.all.mainnet.ethdisco.net` =>
```
enrtree-branch:
    PS2VTC25LB55L6QRIB4Z3DLKY4,GLWRNB5ZJYRBGI7FMKD44PT5AQ,3M273RBSBOBVDEBEVJT7MVC46Y,NYYGSM2JJD5GPY4A2SEDZOJOPM,LCHQX7DYYHZ4FT2YKMU55SPTOM,RSSYWFUH5YSHFWSHMPCOZEBMFE,2EG45KXXIREOYP2XFTWEDIOTSU,EQ2JFJQGM424ME55MQU2PGWU7Y,IQ2MQX655ZGGJYXFFGKVJS6JEI,OS6X2RCMSLIIJLM3TTH4SNPXLY,OKKX647BKCSKW5IW66FZLJEHVE,SHYCCNLQ56LVZN3SBNZ4BTCAZA,OJXE42P53GW5DREZ3ORGIQ735E
```
仍选择第一个节点:
`PS2VTC25LB55L6QRIB4Z3DLKY4.all.mainnet.ethdisco.net` =>
```
enrtree-branch:
    EQB6BANBDY7S6FB3CXOAKAKHPQ,N6FVBKKHHHRT627FZUHSFRE7WE,JSTWYETUXFWA77QY2INZERLNLQ,JBHMNHDUML7RLNT5A543V3TMOM,DH37OCQEZKYK5BEUBQQQCXRTDM,4N4AGKDNCLK5ROKJRIU5PF4Z4U,Z5NSNKIVJ3QUSWV6GOVRGCZK5E,WXIGMPPKXEU7LBDCF4VJMFXCXQ,PA5RYZUMUF3ICVQ7ARODPK3R5A,2E2CA3EIDS2ICERHIFETVSCDVY,AYDMO4ZAUNSAWM2SI5LOX3SZZ4,EKA7L5PRXNDAAHLPMZPHIMKHIY,YHXXS4MU5RVBRZDZZMKLJ542LE
```

`EQB6BANBDY7S6FB3CXOAKAKHPQ.all.mainnet.ethdisco.net` => 
```
enr:-Je4QONq94Aa-VkvtRb0klXhGpVGW4mH1BwrfJU9chEjpSviCq8YThCiAD5oZz4UCdexfhLMXMV4kgaz_oOkti2TB5EHg2V0aMfGhCDDJ_yAgmlkgnY0gmlwhIjzL2CJc2VjcDI1NmsxoQLJV1XQ65-I37gAQi3zDisSClBqJ2u9Zrz3HxC8rW3kRoN0Y3CCdl-DdWRwgnZf
```
此输出正是上文介绍的`ENR`编码的节点信息。

> 为了方便读者阅读，我在`enrtree-branch:`与具体节点之前加入了换行。

在此处，我们也介绍一下如何生成此类`merkle tree`。首先，我们需要一个以太坊节点信息，以上文最后一个返回值为例，对其进行`keecak256`计算(可以使用[此网站](https://emn178.github.io/online-tools/keccak_256.html))，我们可以得到`2403e081a11e3f2f143b15dc0501477c5d6a656198eaed1119aefcfac38d4d2f`，取其前32位`2403e081a11e3f2f143b15dc0501477c`进行`base32`编码(可以使用[此网站](https://cryptii.com/pipes/base32))获得`EQB6BANBDY7S6FB3CXOAKAKHPQ`。

当我们获得此叶节点地址后，我们需要聚合13个叶节点，聚合成字符串`enrtree-branch:EQB6BANBDY7S6FB3CXOAKAKHPQ,N6FVBKKHHHRT627FZUHSFRE7WE,JSTWYETUXFWA77QY2INZERLNLQ,JBHMNHDUML7RLNT5A543V3TMOM,DH37OCQEZKYK5BEUBQQQCXRTDM,4N4AGKDNCLK5ROKJRIU5PF4Z4U,Z5NSNKIVJ3QUSWV6GOVRGCZK5E,WXIGMPPKXEU7LBDCF4VJMFXCXQ,PA5RYZUMUF3ICVQ7ARODPK3R5A,2E2CA3EIDS2ICERHIFETVSCDVY,AYDMO4ZAUNSAWM2SI5LOX3SZZ4,EKA7L5PRXNDAAHLPMZPHIMKHIY,YHXXS4MU5RVBRZDZZMKLJ542LE`，对此字符串进行`keecak256`哈希计算取前32位并进行`base32`编码即可得到`PS2VTC25LB55L6QRIB4Z3DLKY4`。

循环进行上述过程，我们就可以得到最终的形式。而DNS运行者仅需要对此`merkle tree`的根节点`WWP2ZHNHHZO3K5Y5T2VMGV7GXU`进行签名即可。

关于此部分的`go-ethereum`代码较为复杂，因为涉及到大量的抽象操作以构建对应的`merkle tree`。如果读者感兴趣，可以以`SyncTree`[函数](https://sourcegraph.com/github.com/ethereum/go-ethereum/-/blob/p2p/dnsdisc/client.go?L113)为主干开始探索。

如果读者认为使用DNS服务器同步复杂的`merkle tree`是麻烦的，也可以直接前往[Discv4 DNS List](https://github.com/ethereum/discv4-dns-lists)获得直接的地址列表。

### 节点握手认证

当节点获得对等节点后，向节点发送任何信息都需要经过此握手阶段，所以在此处，我们首先介绍如何进行节点之间的握手操作。此处涉及到一系列的数据包格式定义和具体的数据交换流程。

我们会先介绍不同数据包的格式，再介绍具体的握手流程。此处，我们对一些不是非常重要的定义问题进行省略，如果读者对此感兴趣，可以自行阅读[官方文档](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-theory.md)。

#### 数据包格式

此节主要介绍在在`Node Discovery Protocol v5`中所使用的各个数据包的具体格式。为后文介绍具体的握手等网络流程奠定基础。

值得注意的是，此节介绍的所有网络包均基于`UDP`且仅用于节点发现和绑定工作，并不用于交易数据交换等场景。后者由`rlpx`协议栈规定，且基于`TCP`。

读者可以跳过此节，当下文提及具体的包名时，再阅读本节介绍的包格式。此节主要参考了[Node Discovery Protocol v5 - Wire Protocol](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-wire.md)。

本节主要涵盖以下三种数据包:

1. `Ordinary message packets` 普通信息包，主要用于携带加密/认证信息
1. `WHOAREYOU packets` `WHOAREYOU`包，当普通信息数据包的接收者不能解密/认证数据包的信息时发送，主要规定加密的所需要的一系列数据
1. `Handshake message packets` 握手信息包，在`WHOAREYOU`包之后发送。这些数据包建立了一个新的会话，除了加密/认证的消息外，还携带与握手有关的数据。

在每次创建新的会话时，需要建立新会话的节点需要依次使用上述数据包进行交互，具体的交互流程，我们会在下一节进行介绍。

在此协议中，数据包的具体构成如下:
```
packet = masking-iv || masked-header || message
```
其中，各参数含义为:

- masking-iv 随机生成的数据，对称加密过程中使用的`初始化向量`
- masked-header 使用目标节点`nodeID`的前16位作为密钥，使用`masking-iv`作为计数器使用`AES`对称加密算法的`CTR`模式获得的对`header`进行加密的密文
- message 具体的信息

> 在此处进行加密的原因是为了避免被防火墙通过头部判断出数据包的类型。

我们首先介绍`masked-header`字段，此字段构成如下:
```
header        = static-header || authdata
static-header = protocol-id || version || flag || nonce || authdata-size
protocol-id   = "discv5"
version       = 0x0001
authdata-size = uint16    -- byte length of authdata
flag          = uint8     -- packet type identifier
nonce         = uint96    -- nonce of message
```
其中的参数`authdata`和`flag`的定义与具体的数据包类型有关，我们会在下文进行介绍。

`nonce`的生成代码如下:
```go
func generateNonce(counter uint32) (n Nonce, err error) {
	binary.BigEndian.PutUint32(n[:4], counter)
	_, err = crand.Read(n[4:])
	return n, err
}
```
其中的`counter`为发包计数器，每次进行通信后会自增。

在`go-ethereum`项目的`p2p/discover/v5wire/encoding.go`中可以找到此定义的源代码:
```go
type StaticHeader struct {
	ProtocolID [6]byte
	Version    uint16
	Flag       byte
	Nonce      Nonce
	AuthSize   uint16
}
```
有了上述字段，我们可以对`header`进行`AES/CTR`加密以获得`masked-header`参数，代码如下:
```go
func (h *Header) mask(destID enode.ID) cipher.Stream {
	block, err := aes.NewCipher(destID[:16])
	if err != nil {
		panic("can't create cipher")
	}
	return cipher.NewCTR(block, h.IV[:])
}
```
即使用`packet`中的`masking-iv`作为计数器并使用目标节点`nodeID`的前16位作为密钥进行`AES`加密。

在`ordinary message packets`和`handshake message packets`中，我们需要在上文介绍的`header`后加入`message`字段。具体定义如下:
```
message       = aesgcm_encrypt(initiator-key, nonce, message-pt, message-ad)
message-pt    = message-type || message-data
message-ad    = masking-iv || header
```
此定义较为抽象，我们直接给出具体的代码:
```go
func (c *Codec) encryptMessage(s *session, p Packet, head *Header, headerData []byte) ([]byte, error) {
	// Encode message plaintext.
	c.msgbuf.Reset()
	c.msgbuf.WriteByte(p.Kind())
	if err := rlp.Encode(&c.msgbuf, p); err != nil {
		return nil, err
	}
	messagePT := c.msgbuf.Bytes()

	// Encrypt into message ciphertext buffer.
	messageCT, err := encryptGCM(c.msgctbuf[:0], s.writeKey, head.Nonce[:], messagePT, headerData)
	if err == nil {
		c.msgctbuf = messageCT
	}
	return messageCT, err
}
```

在下文中，我们大量使用了`Codec`，此结构体内包含所有关于包的内容，定义如下:
```go
type Codec struct {
	sha256    hash.Hash
	localnode *enode.LocalNode
	privkey   *ecdsa.PrivateKey
	sc        *SessionCache

	// encoder buffers
	buf      bytes.Buffer // whole packet
	headbuf  bytes.Buffer // packet header
	msgbuf   bytes.Buffer // message RLP plaintext
	msgctbuf []byte       // message data ciphertext

	// decoder buffer
	reader bytes.Reader
}
```

接下来，我们介绍三种类型的数据包的构成:

**Ordinary Message Packet(`flag=0`)**

对于此格式的数据包，`authdata`的定义如下:
```
authdata      = src-id
authdata-size = 32
```
包的结构图如下:
![Message Packet](https://img-blog.csdnimg.cn/img_convert/858c50f1af6026431ae5cb0cc0f59400.png)

在此处，我们给出一个`message data`为随机数据的包的代码:
```go
func (c *Codec) encodeRandom(toID enode.ID) (Header, []byte, error) {
	head := c.makeHeader(toID, flagMessage, 0)

	// Encode auth data.
	auth := messageAuthData{SrcID: c.localnode.ID()}
	if _, err := crand.Read(head.Nonce[:]); err != nil {
		return head, nil, fmt.Errorf("can't get random data: %v", err)
	}
	c.headbuf.Reset()
	binary.Write(&c.headbuf, binary.BigEndian, auth)
	head.AuthData = c.headbuf.Bytes()

	// Fill message ciphertext buffer with random bytes.
	c.msgctbuf = append(c.msgctbuf[:0], make([]byte, randomPacketMsgSize)...)
	crand.Read(c.msgctbuf)
	return head, c.msgctbuf, nil
}
```
在此处，`makeHeader(toID enode.ID, flag byte, authsizeExtra int)`函数用于设置`header`字段中的`Flag`和`AuthSize`参数。我们在此处完成了对`head`的初始化并返回了了包含随机内容的`message`。该具有随机内容的数据包用于在不清楚对等节点密钥的情况下进行握手请求。代码如下:
```go
// No keys, send random data to kick off the handshake.
head, msgData, err = c.encodeRandom(id)
```

我们后文介绍的各种数据包类型只是对`message data`进行修改。

**WHOAREYOU Packet(`flag = 1`)**
 
此数据包在握手流程中有着极其重要的作用，主要用于与对等节点进行一系列信息交换。该数据包的定义如下:
```
authdata      = id-nonce || enr-seq
authdata-size = 24
id-nonce      = uint128   -- random bytes
enr-seq       = uint64    -- ENR sequence number of the requesting node
```
结构图如下:
![WHOAREYOU Packet](https://img-blog.csdnimg.cn/img_convert/ed96680762e49b0938c71da5c516ce65.png)

值得注意的是，我们`WHOAREYOU`包的`nonce`与引起握手的数据包的`nonce`是一致的，不可改变的。在下文握手阶段，我们会使用这一知识点。

我们在此处给出`go-ethereum`中的实现代码:
```go
func (c *Codec) encodeWhoareyou(toID enode.ID, packet *Whoareyou) (Header, error) {
	// Sanity check node field to catch misbehaving callers.
	if packet.RecordSeq > 0 && packet.Node == nil {
		panic("BUG: missing node in whoareyou with non-zero seq")
	}

	// Create header.
	head := c.makeHeader(toID, flagWhoareyou, 0)
	head.AuthData = bytesCopy(&c.buf)
	head.Nonce = packet.Nonce

	// Encode auth data.
	auth := &whoareyouAuthData{
		IDNonce:   packet.IDNonce,
		RecordSeq: packet.RecordSeq,
	}
	c.headbuf.Reset()
	binary.Write(&c.headbuf, binary.BigEndian, auth)
	head.AuthData = c.headbuf.Bytes()
	return head, nil
}
```
核心代码为`binary.Write(&c.headbuf, binary.BigEndian, auth)`，此代码向`headbuf`内写入`authdata`内容。代码`head.Nonce = packet.Nonce`表示`WHOAREYOU`的`nonce`与引起握手的数据包的`nonce`相同。

**Handshake Message Packet (`flag = 2`)**

此数据包的内容较为复杂，包含多个字段，也是在握手环节内最为重要的一个数据包类型。具体定义如下:
```
authdata      = authdata-head || id-signature || eph-pubkey || record
authdata-head = src-id || sig-size || eph-key-size
authdata-size = 34 + sig-size + eph-key-size + len(record)
sig-size      = uint8     -- value: 64 for ID scheme "v4"
eph-key-size  = uint8     -- value: 33 for ID scheme "v4"
```
其中，各参数含义如下:

- eph-key-size 临时公钥的长度
- id-signature 使用临时私钥签名的数据

此处的所有参数的含义及其获得，我们会在下一节进行详细介绍。此处给出其结构图如下:

![Handshake Packet layout](https://img-blog.csdnimg.cn/img_convert/ac846f667d43e01a02c6c09bb682157f.png)

代码如下:
```go
func (c *Codec) encodeHandshakeHeader(toID enode.ID, addr string, challenge *Whoareyou) (Header, *session, error) {
	// Ensure calling code sets challenge.node.
	if challenge.Node == nil {
		panic("BUG: missing challenge.Node in encode")
	}

	// Generate new secrets.
	auth, session, err := c.makeHandshakeAuth(toID, addr, challenge)
	if err != nil {
		return Header{}, nil, err
	}

	// Generate nonce for message.
	nonce, err := c.sc.nextNonce(session)
	if err != nil {
		return Header{}, nil, fmt.Errorf("can't generate nonce: %v", err)
	}

	// TODO: this should happen when the first authenticated message is received
	c.sc.storeNewSession(toID, addr, session)

	// Encode the auth header.
	var (
		authsizeExtra = len(auth.pubkey) + len(auth.signature) + len(auth.record)
		head          = c.makeHeader(toID, flagHandshake, authsizeExtra)
	)
	c.headbuf.Reset()
	binary.Write(&c.headbuf, binary.BigEndian, &auth.h)
	c.headbuf.Write(auth.signature)
	c.headbuf.Write(auth.pubkey)
	c.headbuf.Write(auth.record)
	head.AuthData = c.headbuf.Bytes()
	head.Nonce = nonce
	return head, session, err
}
```

以上内容就是在进行握手中所使用的主要数据包。我们在此处省略了`Ordinary Message Packet`中`message`的其他情况，读者可以自行参考[文档](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-wire.md#protocol-messages)。当然，我们会在后文介绍部分其他包的结构，读者可以暂不阅读文档而继续阅读此文。

#### 握手

在此处，我们会尽可能展示流程图帮助大家理解整个握手过程。在下文中，我们假设节点A向节点B发送`FINDNODED`数据包。

第一步，节点A发送数据包。

![Step 1](https://s-bj-3358-blog.oss.dogecdn.com/svg/step1.drawio.svg)

当节点A需要与节点B进行通信时，节点A应当首先查询自己是否具有之前与B通信的会话密钥，如果有密钥则按照密钥对数据包加密。如果没有，则发送一个带有随机内容的[普通数据包](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-wire.md#ordinary-message-packet-flag--0)(`Ordinary Message Packet`)启动握手。

第二步，节点B进行回应。

![Step2](https://s-bj-3358-blog.oss.dogecdn.com/svg/step2.drawio.svg)

当节点B获得发送的数据包后，节点B会在数据包`header`部分中的`authdata`提取出发送方`src-id`字段，并查询自己是否具有此`src-id`的会话密钥。如果有，则使用密钥进行解密。如果解密正确并能验证发送者身份正确，则直接进行响应。

> 正如上文所述，`header`使用目标节点`nodeID`的前16位作为密钥，使用`masking-iv`作为计数器使用`AES`对称加密算法的`CTR`模式获得的对`header`进行加密的密文。而目标节点B收到数据包后，可以通过提取未加密的`masking-iv`和自身的节点ID解密数据包。

如果没有会话密钥或会话密钥无法正确解密数据(会话密钥过期)，B节点会发送[ WHOAREYOU 数据包](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-wire.md#whoareyou-packet-flag--1)。B节点会在`WHOAREYOU`数据包内填入`id-nonce`字段。节点B会检测自己是否有节点A的`ENR`，如果有，则会在数据包内加入`enr-seq`字段。

> `enr-seq`字段就是我们在上文介绍的`ENR`格式中的`seq`字段。此字段会在`ENR`每次更改时自加`1`。此处在`WHOAREYOU`数据包内加入此字段是为了方便节点A判断节点B的`ENR`是否为最新，如果不是会进行反馈。

此过程的源代码如下:
```go
func (t *UDPv5) handleUnknown(p *v5wire.Unknown, fromID enode.ID, fromAddr *net.UDPAddr) {
	challenge := &v5wire.Whoareyou{Nonce: p.Nonce}
	crand.Read(challenge.IDNonce[:])
	if n := t.getNode(fromID); n != nil {
		challenge.Node = n
		challenge.RecordSeq = n.Seq()
	}
	t.sendResponse(fromID, fromAddr, challenge)
}
```
其中，`v5wire.Unknown`即无法解密的数据包。此处使用的`crand.Read`是一个具有密码学安全的随机数生成器。上述代码其实仅在`WHOAREYOU`数据包内设置了`id-nonce`和`enr-seq`参数，其余参数的生成通过上文给出的`encodeWhoareyou`等函数。

第三步，节点A处理`WHOAREYOU`请求

![Step 3](https://s-bj-3358-blog.oss.dogecdn.com/svg/step3.drawio.svg)

节点A收到节点B发送的`WHOAREYOU`数据包后，节点A首先查询`WHOAREYOU`的`nonce`字段追溯到自己的发起握手的请求包。具体代码如下:
```go
func (t *UDPv5) matchWithCall(fromID enode.ID, nonce v5wire.Nonce) (*callV5, error) {
	c := t.activeCallByAuth[nonce]
	if c == nil {
		return nil, errChallengeNoCall
	}
	if c.handshakeCount > 0 {
		return nil, errChallengeTwice
	}
	return c, nil
}
```
其中，`activeCallByAuth`为`map[v5wire.Nonce]*callV5`映射。

> 因为`WHOAREYOU`数据包内不包含`src-id`字段，为了获得此`WHOAREYOU`数据包的发送者的`NodeID`，我们只能通过`nonce`匹配获得发送者的`NodeID`

节点A需要构建`Handshake Message Packet`数据包，即握手数据包。此数据包的构成可参考上文，需要注意的是用户可以设置`message data`字段，实现握手后立即响应请求。在此处，我们将`message data`字段填充为`FINDNODED`规定的内容。

此处核心在于构建`authdata`和`authdata-size`，为方便读者阅读，我们在此处再次给出有关定义:
```
authdata      = authdata-head || id-signature || eph-pubkey || record
authdata-head = src-id || sig-size || eph-key-size
authdata-size = 34 + sig-size + eph-key-size + len(record)
sig-size      = uint8     -- value: 64 for ID scheme "v4"
eph-key-size  = uint8     -- value: 33 for ID scheme "v4"
```

为了构建此数据包，节点A首先将节点B发送的`WHOAREYOU`数据包头部进行解密，得到:
```
challenge-data     = masking-iv || static-header || authdata
```
在此解密后的头部中，节点A可以获得节点B使用的`protocol-id`情况，根据此情况，节点A生成会话密钥。目前，`protocol-id`在以太坊官方中仅有`discv5`情况，使用了`secp256k1`曲线。节点A需要在`secp256k1`曲线上生成会话密钥。在此处，我们定义如下变量:
```
ephemeral-key      = 节点A随机生成的私钥
ephemeral-pubkey   = 与上述私钥对应的公钥
```

接下来，我们需要生成会话密钥，步骤如下:

1. 在目标节点`ENR`中获得目标节点的公钥`pub`
1. 使用`ECDH`([Elliptic-curve Diffie–Hellman](https://en.wikipedia.org/wiki/Elliptic-curve_Diffie%E2%80%93Hellman))获得`secret`变量。简单来说，ECDH只是将节点A生成私钥`ephemeral-key`与节点B的公钥`pub`相乘。
1. 将`"discovery v5 key agreement"`、节点A的`NodeID`、节点B的`NodeID`拼接在一起，作为变量`info`
1. 使用`HKDF`算法获得随机二进制数据
1. 读取 16 bytes 数据作为`writeKey`，再读取 16 bytes 作为`readKey`

上述流程的代码如下:
```go
func deriveKeys(hash hashFn, priv *ecdsa.PrivateKey, pub *ecdsa.PublicKey, n1, n2 enode.ID, challenge []byte) *session {
	const text = "discovery v5 key agreement"
	var info = make([]byte, 0, len(text)+len(n1)+len(n2))
	info = append(info, text...)
	info = append(info, n1[:]...)
	info = append(info, n2[:]...)

	eph := ecdh(priv, pub)
	if eph == nil {
		return nil
	}
	kdf := hkdf.New(hash, eph, challenge, info)
	sec := session{writeKey: make([]byte, aesKeySize), readKey: make([]byte, aesKeySize)}
	kdf.Read(sec.writeKey)
	kdf.Read(sec.readKey)
	for i := range eph {
		eph[i] = 0
	}
	return &sec
}
```

其中，`priv`即节点随机生成的私钥，其余变量含义已在上文给出。为了提高可拓展性，此处我们可以自行选择`hash`函数，但目前执行层仅支持`sha256`哈希算法。

接下来，我们生成`id-signature`字段，步骤如下:

1. 将`"discovery v5 identity proof"`、解密后的`WHOAREYOU`数据包内容、`ephemeral-key`、节点B的`NodeID`拼接在一起后进行`hash`，作为`input`
1. 使用私钥`ephemeral-key`进行签名。

具体代码如下:
```go
func idNonceHash(h hash.Hash, challenge, ephkey []byte, destID enode.ID) []byte {
	h.Reset()
	h.Write([]byte("discovery v5 identity proof"))
	h.Write(challenge)
	h.Write(ephkey)
	h.Write(destID[:])
	return h.Sum(nil)
}

func makeIDSignature(hash hash.Hash, key *ecdsa.PrivateKey, challenge, ephkey []byte, destID enode.ID) ([]byte, error) {
	input := idNonceHash(hash, challenge, ephkey, destID)
	switch key.Curve {
	case crypto.S256():
		idsig, err := crypto.Sign(input, key)
		if err != nil {
			return nil, err
		}
		return idsig[:len(idsig)-1], nil // remove recovery ID
	default:
		return nil, fmt.Errorf("unsupported curve %s", key.Curve.Params().Name)
	}
}
```
此处允许用户选择其他类型的椭圆函数，但目前以太坊执行层仅支持`secp256k1`曲线。

有了上述字段，我们可以非常简单的拼接出`Handshake Message Packet`的头部，而具体的`message`则由发送者决定，并使用`writeKey`对`message`中的数据进行`AES-GCM`加密。在此处，我们将其设置为[FINDNODED](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-wire.md#findnode-request-0x03)

```go
func (c *Codec) decodeHandshake(fromAddr string, head *Header) (n *enode.Node, auth handshakeAuthData, s *session, err error) {
	if auth, err = c.decodeHandshakeAuthData(head); err != nil {
		return nil, auth, nil, err
	}

	// Verify against our last WHOAREYOU.
	challenge := c.sc.getHandshake(auth.h.SrcID, fromAddr)
	if challenge == nil {
		return nil, auth, nil, errUnexpectedHandshake
	}
	// Get node record.
	n, err = c.decodeHandshakeRecord(challenge.Node, auth.h.SrcID, auth.record)
	if err != nil {
		return nil, auth, nil, err
	}
	// Verify ID nonce signature.
	sig := auth.signature
	cdata := challenge.ChallengeData
	err = verifyIDSignature(c.sha256, sig, n, cdata, auth.pubkey, c.localnode.ID())
	if err != nil {
		return nil, auth, nil, err
	}
	// Verify ephemeral key is on curve.
	ephkey, err := DecodePubkey(c.privkey.Curve, auth.pubkey)
	if err != nil {
		return nil, auth, nil, errInvalidAuthKey
	}
	// Derive sesssion keys.
	session := deriveKeys(sha256.New, c.privkey, ephkey, auth.h.SrcID, c.localnode.ID(), cdata)
	session = session.keysFlipped()
	return n, auth, session, nil
}
```

第四步，节点B收到握手后返回信息。

![Step4 Flow](https://s-bj-3358-blog.oss.dogecdn.com/svg/step4.drawio.svg)

当节点B收到节点A发送的握手包后，首先加载之前存储的`WHOAREYOU`数据包内容，并对握手包的头部进行解密。如果握手包内存在`record`记录，则节点B根据握手包中的`record`字段更新对A节点的记录。对`id-signature`字段进行提取公钥操作，将获得公钥与`ephemeral-pubkey`字段的记录进行比较。

经过上述测试后，节点B使用`ECDH`算法基于自身私钥与对方的短暂公钥`ephemeral-pubkey`进行计算通信使用的会话密钥并对`message`进行解密。如果`message`可以顺利解密，节点B会对`message`中的请求进行响应，在此处为对`FINDNODED`的响应，即返回节点信息。

这里有一点比较神奇的是，我们在上文给出了`writeKey`和`readKey`作为密钥，根据名称，我们可以判断出`writeKey`用于向对方发送数据时加密，`readKey`用于解密对方的数据包。但`ECDH`会输出一个相同的密钥，在对称加密的情况下，我们使用相同的密钥对数据进行加密或解密，但显然`writeKey`与`readKey`是不同的。为解决这一问题，以太坊使用了一种较为简单的方案，即收到握手包的节点B会对`ECDH`产生的密钥进行调换，实现自身`readKey`与节点A的`writeKey`相同的条件。

上述过程的具体代码如下:
```go
func (c *Codec) decodeHandshake(fromAddr string, head *Header) (n *enode.Node, auth handshakeAuthData, s *session, err error) {
	if auth, err = c.decodeHandshakeAuthData(head); err != nil {
		return nil, auth, nil, err
	}

	// Verify against our last WHOAREYOU.
	challenge := c.sc.getHandshake(auth.h.SrcID, fromAddr)
	if challenge == nil {
		return nil, auth, nil, errUnexpectedHandshake
	}
	// Get node record.
	n, err = c.decodeHandshakeRecord(challenge.Node, auth.h.SrcID, auth.record)
	if err != nil {
		return nil, auth, nil, err
	}
	// Verify ID nonce signature.
	sig := auth.signature
	cdata := challenge.ChallengeData
	err = verifyIDSignature(c.sha256, sig, n, cdata, auth.pubkey, c.localnode.ID())
	if err != nil {
		return nil, auth, nil, err
	}
	// Verify ephemeral key is on curve.
	ephkey, err := DecodePubkey(c.privkey.Curve, auth.pubkey)
	if err != nil {
		return nil, auth, nil, errInvalidAuthKey
	}
	// Derive sesssion keys.
	session := deriveKeys(sha256.New, c.privkey, ephkey, auth.h.SrcID, c.localnode.ID(), cdata)
	session = session.keysFlipped()
	return n, auth, session, nil
}
```

### 节点查询

此节主要介绍节点在进行握手之后，如果通过`FINDNODED`初始化节点列表。为了达成此目标，此节介绍了节点的存储方式和`FINDNODED`查询节点的基本原理。

#### 存储

关于节点存储的源代码位于[Table.go](https://github.com/ethereum/go-ethereum/blob/master/p2p/discover/table.go)内，本文对其部分内容进行介绍。

在介绍具体的节点存储前，我们需要了解一个比较重要的概念——`距离`。与我们认为的物理距离不同，使用`Kademlia`算法的以太坊使用了两个节点的`NodeID`进行`XOR`的结果并进行一些其他计算作为最终的距离。计算方法如下:
```go
func LogDist(a, b ID) int {
	lz := 0
	for i := range a {
		x := a[i] ^ b[i]
		if x == 0 {
			lz += 8
		} else {
			lz += bits.LeadingZeros8(x)
			break
		}
	}
	return len(a)*8 - lz
}
```
当我们计算出与其他节点的距离后，我们需要将其划分到`k-buckets`中，每个桶包含 16 个节点。我们通过计算`LogDist(self, other) = i`将`other`节点分配到第`i`个桶。理论上，我们需要 256 个桶。但在实际情况中，由于我们计算距离的算法问题和节点`NodeID`充分的随机性，我们选择不可能发现距离小于`LogDist(self, other) < 239`的情况，所以在实际情况中，我们规定节点被分配到第`LogDist(self, other) - 239`个桶，这样的话，我们仅需要`17`个桶。

在每一个桶中，我们将节点按活跃频率进行排序，最不活跃的节点位于桶的最后，最活跃的节点防止桶的最前面。

当我们发现一个新的节点后，我们需要计算此节点应该被分配的桶的序号，并将其推入桶中。在此过程中分为以下两种情况:

1. 被分配的桶没有满则直接将其推入桶的末尾
1. 被分配的桶已经满了，我们需要对桶的最后节点进行有效性检测(即`PING`)，如果没有收到回复则删除此节点并将其替换为新节点

我们主要关注第二种情况，此情况更具有技术复杂性。如果我们每次找到一个新节点就对桶中16个节点均进行`PING`，会出现`DoS`攻击的效果，以太坊网络可能因为过量的有效性检测请求而变得十分低效。

一种较好的解决方案是在进行异步的检测，我们将新发现的节点放入对应桶的替换缓存(缓存最多 10 个节点)中。节点会每隔 10 秒钟就对随机桶中的所有节点进行`PING`操作，将进行响应的节点排序到桶的最前方。如果发现节点失效，我们可以直接使用替换缓存中的节点对其进行替换。此流程代码如下:
```go
func (tab *Table) doRevalidate(done chan<- struct{}) {
	defer func() { done <- struct{}{} }()

	last, bi := tab.nodeToRevalidate()
	if last == nil {
		// No non-empty bucket found.
		return
	}

	// Ping the selected node and wait for a pong.
	remoteSeq, err := tab.net.ping(unwrapNode(last))

	// Also fetch record if the node replied and returned a higher sequence number.
	if last.Seq() < remoteSeq {
		n, err := tab.net.RequestENR(unwrapNode(last))
		if err != nil {
			tab.log.Debug("ENR request failed", "id", last.ID(), "addr", last.addr(), "err", err)
		} else {
			last = &node{Node: *n, addedAt: last.addedAt, livenessChecks: last.livenessChecks}
		}
	}

	tab.mutex.Lock()
	defer tab.mutex.Unlock()
	b := tab.buckets[bi]
	if err == nil {
		// The node responded, move it to the front.
		last.livenessChecks++
		tab.log.Debug("Revalidated node", "b", bi, "id", last.ID(), "checks", last.livenessChecks)
		tab.bumpInBucket(b, last)
		return
	}
	// No reply received, pick a replacement or delete the node if there aren't
	// any replacements.
	if r := tab.replace(b, last); r != nil {
		tab.log.Debug("Replaced dead node", "b", bi, "id", last.ID(), "ip", last.IP(), "checks", last.livenessChecks, "r", r.ID(), "rip", r.IP())
	} else {
		tab.log.Debug("Removed dead node", "b", bi, "id", last.ID(), "ip", last.IP(), "checks", last.livenessChecks)
	}
}
```

#### 查询

在介绍具体的查找之前，我们首先介绍当节点收到`FINDNODED`数据包后的操作流程。`FINDNODED`数据包由以下部分构成:
```
message-data = [request-id, [distance₁, distance₂, ..., distanceₙ]]
message-type = 0x03
distanceₙ    = requested log2 distance, a positive integer
```
在此数据包内，`request-id`是一个RLP数组，此参数由请求者进行分配，数据包接受者当收到此数据包后，需要在返回的数据包内包含请求的`request-id`。`[request-id, [distance₁, distance₂, ..., distanceₙ]]`表示使用`RLP`进行数据编码。

正如上文所述，节点第一次启动时一般仅与初始化节点`bootnodes`进行通信，此时的节点中的桶基本都是空的。我们需要向`bootnodes`发起`FINDNODED`请求获得节点填充自己的桶。但需要注意，我们只能查询到对方节点指定距离的桶内的节点。

当我们需要填充节点的桶时，我们会使用`lookupRandom()`函数，此函数会随机在目前的以太坊P2P网络中进行随机搜索以达到快速填充桶的目的。我们首先需要了解如何在以太坊中进行搜索操作，当我们开始搜索时，我们会在桶内随机筛选最多 3 个与目标节点距离最近的节点，对这些节点发起`FINDNODED`请求。当获得发送节点的反馈后，节点再次挑选 3 个节点进行`FINDNODED`查询。部分代码如下:
```go
func (t *UDPv5) newRandomLookup(ctx context.Context) *lookup {
	var target enode.ID
	crand.Read(target[:])
	return t.newLookup(ctx, target)
}

func (t *UDPv5) newLookup(ctx context.Context, target enode.ID) *lookup {
	return newLookup(ctx, t.tab, target, func(n *node) ([]*node, error) {
		return t.lookupWorker(n, target)
	})
}
```
关于递归查询的代码逻辑位于`lookup.go`内部，此部分代码较难理解，涉及部分`go`的通道等特性，为了方便读者阅读，此处不再给出代码。

当接受查询的节点收到`FINDNODED`后，节点需要返回符合距离条件的节点。此处的距离指节点与自己的距离而不是节点与请求者的距离。具体代码如下:
```go
func (t *UDPv5) collectTableNodes(rip net.IP, distances []uint, limit int) []*enode.Node {
	var nodes []*enode.Node
	var processed = make(map[uint]struct{})
	for _, dist := range distances {
		// Reject duplicate / invalid distances.
		_, seen := processed[dist]
		if seen || dist > 256 {
			continue
		}

		// Get the nodes.
		var bn []*enode.Node
		if dist == 0 {
			bn = []*enode.Node{t.Self()}
		} else if dist <= 256 {
			t.tab.mutex.Lock()
			bn = unwrapNodes(t.tab.bucketAtDistance(int(dist)).entries)
			t.tab.mutex.Unlock()
		}
		processed[dist] = struct{}{}

		// Apply some pre-checks to avoid sending invalid nodes.
		for _, n := range bn {
			// TODO livenessChecks > 1
			if netutil.CheckRelayIP(rip, n.IP()) != nil {
				continue
			}
			nodes = append(nodes, n)
			if len(nodes) >= limit {
				return nodes
			}
		}
	}
	return nodes
}
```
我们使用了`bucketAtDistance`函数直接查询在桶内满足条件的节点并进行返回。

### 主题检索

此部分的主要功能为节点可以为自己设定一个特殊的关键词帮助其他节点进行检索。一个节点可以为自己设置多个或不设定关键词。参与网络的每一个节点都可以响应根据主题的检索请求。当然，此部分似乎仍属于建设阶段，为了优化读者阅读体验，我们对此模块不进行深入研究。如果读者对其感兴趣，可以自行阅读[文档](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-theory.md#topic-advertisement)了解相关知识。

## 总结

本文基本介绍了P2P网络初次进入的基本流程，包含以下内容:

1. `ENR`、`Encode`等以太坊节点表示格式和`DNSDisc`查询节点的方式
1. `Discv5`中`Ordinary Message Packet`、`WHOAREYOU Packet`、`Handshake Message Packet`等数据包的构成和对应的加密算法
1. `Discv5`规定的繁琐的握手流程和加密流程(`ECDH`、`HKDF`)
1. 基于`Kademlia`节点信息的存储方式和查询方式

由于我本人并不是P2P网络方面的研究人员，而且对于以太坊P2P源代码并未进行深入探索，如果读者发现本文出现的任何问题，可以随时通过[我的博客](https://hugo.wongssh.cf/)中提供的邮件与我联系交流。

对于下篇，可能会在不久后推出。
