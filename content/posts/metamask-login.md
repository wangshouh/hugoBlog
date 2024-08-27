---
title: MetaMask一键登录设计
date: 2022-06-06T22:52:13Z
tags: [Vue,MetaMask,Web3,CloudflareWorker,EIP-712]
aliases: ["/2022/06/06/metamask-login/"]
---

## 概述

在目前的网站用户体系搭建中，社会化登录主要依赖于Google、QQ等服务商，中心化趋势较强。在Web3中，作为网站建设者的我们应该考虑使用去中心化的登录方式。在此篇博客中，我们将以使用[MetaMask](https://metamask.io/)钱包中的API为例介绍去中心化登录的基本方式。

我会介绍前端页面的搭建和后端服务的设计。我选择了`Vue`作为前端页面的框架，同时使用了`MetaMask`插件提供的API接口。在后端为降低成本，我采用了[CloudFlare Worker](https://workers.cloudflare.com/)作为后端，主要使用`Worker`和`KV`服务。

本文的主要思路来自[这篇文章](https://www.toptal.com/ethereum/one-click-login-flows-a-metamask-tutorial)，但根据最新的MetaMsk的API和cloudflare worker的新特性，我对此文章的内容进行了一些改进，但总体思路是相同的。

## 登录流程

由于以太坊等加密货币自身建立在非对称加密基础上，我们应该考虑使用使用非对称加密的功能来实现登录。登录的本质是用户对个人身份的证明，在过去的登录方式中，我们采用密码、手机或邮箱验证码实现。而在Web3中，以太坊等区块链天然的提供了一种工具实现这一过程，即签名。
[签名](https://en.wikipedia.org/wiki/Digital_signature)是指用户使用私钥对数据进行签名，签名后的数据可以用公钥来验证。我们可以使用MetaMask中的签名API实现此流程，您可以查阅[此页面](https://docs.metamask.io/guide/signing-data.html)来查看所有属于MetaMask的签名API，但在此教程中我们会选择最新的`signTypedData_v4`API，因为此签名方法更加安全且对用户友好。

用户对什么数据进行签名？这应该由开发者决定，签名内容应该在后端生成后发给前端，之后前端用户对其进行签名，再将签名后的数据与个人以太坊账户地址一同回传给后端，后端对数据进行验签，判断数据是否与以太坊账户地址相同。如果相同，则返回登录凭证，在此教程中，我们将返回JWT。如果不同，则返回登录失败消息。总体流程如下图所示：

![MetaMask Login](https://blogimage.4everland.store/metamask-login.svg)

## 登录按钮实现（前端）

我们首先实现登录的第一步，实现请求后端获取签名内容和获取用户的以太坊账户地址。至于后端如何实现签名内容生成我们将在后文介绍。

首先给出Vue的基本框架：

```Vue
<template>
  <div>
    <button @click="login" v-if="metaMaskSupport">
        Login
    </button>
  </div>
</template>

<script>
  import axios from "axios";
  export default {
      data() {
          return {
              metaMaskSupport: false,
              ethAccount: null,
              sign: null,
              nonces: null,
          }
      },
      mounted() {
          this.metaMaskSupport = window.ethereum && window.ethereum.isMetaMask;
      },
      methods: {
          login() {
              //具体实现方式将在下文给出
          }
      }
  }
</script>
```

为了方便后续代码编写，我们导入了知名网络请求库`axios`，您需要使用以下命令安装：

```bash
npm install axios
```

我们在`data()`中定义了一些数据，其中包括：

- `metaMaskSupport`：是否支持MetaMask，如果不支持，则不显示登录按钮

- `ethAccount`：用户的以太坊账户地址

- `sign`：签名结果

- `nonces`：用于防止重放攻击的nonce（将在后文介绍）

在`mounted()`中，我们完成了基本的初始化，使用`window.ethereum && window.ethereum.isMetaMask`赋值给`metaMaskSupport`，这段代码用于判断用户是否安装了MetaMask插件。

接下来，我们会在完善`login()`方法的第一部分，获取用户的以太坊账户地址。

```javascript
window.ethereum.request({ method: 'eth_requestAccounts' }).then(
    accounts => {
        this.ethAccount = accounts[0]
        console.log(this.ethAccount);
    }
)
```

该段代码主要基于MetaMask文档中的[此部分](https://docs.metamask.io/guide/getting-started.html#basic-considerations)。在此处，我们使用了JavaScript中的异步请求，并且使用MetaMask API`window.ethereum.request`方法获取用户的以太坊账户地址并将其赋值给`this.ethAccount`，最终在`console`中输出用户的以太坊账户地址。

以上基本完成了登录的第一步，接下来我们会介绍后端实现签名内容生成的部分。

## 签名内容生成（后端）

### 基础环境搭建

为了代码内容的简单化，我们在此处假设您已经安装了Cloudflare Wrangler并已经搭建了基本的`CloudFlare Worker`的开发环境。如果您对此不熟悉，您可以查阅[此文档](https://developers.cloudflare.com/workers/get-started/guide/)。

>下列代码的前提是您已经完成了wrangler的登录，具体内容可以参考[此文档](https://developers.cloudflare.com/workers/get-started/guide/)。

首先使用此命令创建开发环境：
```bash
wrangler init web3login
```

在完成项目初始化后，您获得的目录结构应该如下图所示：

![项目目录](https://i.imgtg.com/2022/06/08/1i0LX.png)

以下内容主要关于`kv`的绑定问题，如果您认为我的表述较为奇怪，您可以自行查阅cloudflare的[kv文档](https://developers.cloudflare.com/workers/wrangler/workers-kv)

然后您需要绑定您的`kv`到`wrangler`代码中使用`kv`数据库，可以使用下述命令：
```bash
wrangler kv:namespace create web3login
```
![kv bind](https://pic.rmb.bdstatic.com/bjh/069ca9e26924a587cfee946079feef3d.png)

根据提示，将此代码运行后的结果添加到`wrangler.toml`中，如下：
```toml
name = "bloguse"
main = "src/index.js"
compatibility_date = "2022-06-08"

kv_namespaces = [{ binding = "web3login", id = "自行替换" }]
```

由于Worker的自身的限制，为了在dev中使用`kv`，我们需要在终端键入以下命令：

```bash
wrangler kv:namespace create web3login --preview
```

![kv preview](https://pic.rmb.bdstatic.com/bjh/c78d4982275eb22c416cb93147c9fb5a.png)

根据提示，将此代码运行后的结果添加到`wrangler.toml`中，如下：
```toml
name = "bloguse"
main = "src/index.js"
compatibility_date = "2022-06-08"

kv_namespaces = [{ binding = "web3login", id = "自行替换", preview_id = "自行替换" }]
```

我们也需要安装一些必要的npm包，主要需要`eth-sig-util`，您可以通过[此链接](https://github.com/MetaMask/eth-sig-util)查阅它的开源仓库，你也可以通过[这个链接](https://metamask.github.io/eth-sig-util/index.html)查阅它的文档。

使用以下命令，您可以安装此库：
```bash
npm install @metamask/eth-sig-util
```
或
```bash
yarn add @metamask/eth-sig-util
```

### nonces生成及存储

在前述内容中，我们已经得到了用户的`ethAccount`和基础开发环境的搭建，接下来我们考虑如何生成签名所需要的`nonces`。为了验证用户签名是否正确，我们需要存储用户的签名`nonces`和以太坊地址，这是一个简单的key-value数据，我们可以直接使用`CloudFlare`开发的`kv`数据库。

此处我们假设前端返回的数据结构如下：
```json
{ 
  "from": this.ethAccount 
}
```
即仅向后端返回`ethAccount`字段。

以此数据结构为基础，我们给出后端的实现。
```javascript
addEventListener("fetch", event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  if (request.method === "PUT") {
    let data = await request.json();
    let key = data.from;
    let nonces = Math.floor(Math.random() * 1000000)
    await loginKV.put(key, nonces, { expirationTtl: 120 })
    console.log(`${key} has been logged in`)
    return new Response(JSON.stringify({ "nonces": nonces, "key": key }), {
      headers: {
        'content-type': 'application/json;charset=UTF-8',
        'Access-Control-Allow-Origin': '*'
      },
    });
  } else if (request.method === "OPTIONS") {
    const responseHeaders = new Headers();
    responseHeaders.set('Access-Control-Allow-Origin', '*');
    responseHeaders.set('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
    responseHeaders.set('Access-Control-Allow-Headers', 'Origin, X-Requested-With, Content-Type, Accept');
    responseHeaders.set('Access-Control-Max-Age', '86400');
    return new Response("", {
      headers: responseHeaders
    })
  }
}
```

`addEventListener`功能为接受前端的请求，并将其转交给`handleRequest`函数进行处理

`handleRequest`函数的主要功能是：

1. 接受`PUT`请求返回`nonces`并将其存储在`KV`中，`nonces`使用随机数生成，如果您需要严格的密码学保证，您可以选择密码学随机数生成的`crypto.getRandomValues`函数，具体调用发送可以参考[CloudflareWorker文档](https://developers.cloudflare.com/workers/runtime-apis/web-crypto/#methods)。然后使用`Worker`直接调用`kv`的函数将此值直接推进数据库中，具体函数可参考[文档](https://developers.cloudflare.com/workers/runtime-apis/kv/#writing-key-value-pairs)，值得注意的是此处`expirationTtl`（过期删除时间）设置为120秒。


2. 接受`OPTIONS`请求处理跨域请求，跨域请求是个较为复杂的主题，您可以参考[此链接](https://www.ruanyifeng.com/blog/2016/04/cors.html)来了解更多关于跨域请求的内容。

完成代码编写后，我们需要进行一次测试以保证代码的正确运行，在此处我选择使用`Postman`作为调试工具，您可以选择其他工具进行调试。

首先，启动`wrangler`的开发功能，使用`wrangler dev`启动测试环境。使用Postman向`http://localhost:8787`发送`PUT`请求，要求`body`符合上述数据结构。

Postman截图如下：
![postmandev.png](https://img.gopic.xyz/postmandev.png)

wrangler终端输出如下
![wranglerdev.png](https://img.gopic.xyz/wranglerdev.png)

通过Postman或wrangler控制台输出，我们可以判断此段代码是可以正常运行的。

## 请求nonces与签名（前端）

在上述内容中，我们完成了后端`nonces`的生成，同时要求前端想后端发送`PUT`请求并规定了数据结构，在此处我们将实现该功能。

### 获取`nonces`

我们会在前述`login()`函数后进一步增加功能。
```javascript
window.ethereum.request({ method: 'eth_requestAccounts' }).then(
    accounts => {
        this.ethAccount = accounts[0]
        console.log(this.ethAccount);
    }
).then(
  () => {
      axios.put("http://localhost:8787", { "from": this.ethAccount }).then(res => {
          this.nonces = res.data.nonces;
      }).then(() => {
          console.log(this.nonces);
      )
  }
)
```

由于`window.ethereumh`和`axios`中的所有函数均为异步调用，此处为了确保登录逻辑的正常进行使用了大量的`Promise`内容，您可以参阅 **《JavaScript权威指南》第13章** 了解更多关于期约调用的内容。如果您对`axios.put`函数不熟悉，您可以参阅[此文档](https://axios-http.com/zh/docs/api_intro)。此处代码较为简单，不再给出详细解释。

### 进行签名

我们使用`MetaMask`的`signTypedData_v4`对数据进行签名，在此处我们给出简单的API解释，如果您需要更加详细的内容请参阅[MetaMask signTypedData_v4文档](https://docs.metamask.io/guide/signing-data.html#sign-typed-data-v4)

首先，我们需要知道`signTypedData_v4`的签名数据的定义来自`EIP-712`以太坊提案，详细内容可以参考[此文档](https://eips.ethereum.org/EIPS/eip-712)。在此处我们仅仅指出本项目所需要的内容。由于使用`json-schema`解释较为抽象，此处我们给出MetaMask文档同时也是以太坊EIP-712文档中给出的一个签名信息示例。

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

`domain`字段中需要包含以下内容：

- `chainId`，该字段由[EIP-155](https://eips.ethereum.org/EIPS/eip-155)规定，您可以前往[此网站](https://chainlist.org/)浏览更多关于`chainId`的信息，在代码实现中我们将直接使用`window.ethereum.chainId`属性

- `name`，该字段为增强人类的可读性而设计，我们简单的将其定义为`Login`，您可以根据自己的网站进行命名

- `verifyingContract`， 智能合约地址，由于此处我们未使用智能合约，所以我们不会使用此字段

- `version`，顾名思义即版本号，在此处我们将其简单定义为1，可根据实际情况更改

`message`字段可以自行定义，我们在此处仅对以下内容签名：

```javascript
{
  contents: "Login",
  nonces: this.nonces,
}
```

`primaryType`该字段可以简单理解为`message`字段的名字，可以进行自定义，此处我们将其命名为`Login`。该字段的具体功能未找到权威解释，但此字段必须存在。

`types`规定各个字段的具体类型，这些类型与一般编程语言的数据结构不太相同，您可以参考`Soildty`语言的数据类型，可以参考[此文档](https://docs.soliditylang.org/en/v0.8.15/types.html)。该字段要求对上述所有字段的类型进行定义。

最终，我们给出在我的代码中的签名内容：

```javascript
{
    domain: {
        chainId: window.ethereum.chainId,
        name: 'Login',
        version: '1'
    },
    message: {
        contents: 'Login',
        nonces: this.nonces,
    },
    primaryType: 'Login',
    types: {
        EIP712Domain: [
            { name: 'name', type: 'string' },
            { name: 'version', type: 'string' },
            { name: 'chainId', type: 'uint256' },
        ],
        Login: [
            { name: 'contents', type: 'string' },
            { name: 'nonces', type: 'uint256' },
        ],
    },
}
```

获得此签名内容后，我们可以通过API十分简单的进行签名，具体代码如下：
```javascript
window.ethereum.request({ method: 'eth_requestAccounts' }).then(
      accounts => {
          this.ethAccount = accounts[0]
          console.log(this.ethAccount);
      }
  ).then(
      () => {
          axios.put("http://localhost:8787", { "from": this.ethAccount }).then(res => {
              this.nonces = res.data.nonces;
          }).then(() => {
              console.log(this.nonces);
              const msgParams = {
                  domain: {
                      chainId: window.ethereum.chainId,
                      name: 'Login',
                      version: '1'
                  },
                  message: {
                      contents: 'Login',
                      nonces: this.nonces,
                  },
                  primaryType: 'Login',
                  types: {
                      EIP712Domain: [
                          { name: 'name', type: 'string' },
                          { name: 'version', type: 'string' },
                          { name: 'chainId', type: 'uint256' },
                      ],
                      Login: [
                          { name: 'contents', type: 'string' },
                          { name: 'nonces', type: 'uint256' },
                      ],
                  },
              };
              const from = this.ethAccount
              ethereum.request({
                  method: 'eth_signTypedData_v4',
                  params: [from, JSON.stringify(msgParams)],
              }).then((res) => {
                  this.sign = res;
                  console.log(res);
              })
          })
        })
```

如前所述，在此代码中存在着丑陋的期约调用，如何您有兴趣可以将其更改为`async/await`结构。

```javascript
ethereum.request({
    method: 'eth_signTypedData_v4',
    params: [from, JSON.stringify(msgParams)],
}).then((res) => {
    this.sign = res;
    console.log(res);
})

```

上述代码是签名计算的核心，较为简单。

### 回传签名内容

当完成前端签名后，我们需要将签名结果回传给后端，此处我们暂时不加解释的给出回传签名内容的具体结构（在下一节内容中，我们将解释为什么需要这些字段）。

```javascript
{
  "chainId": window.ethereum.chainId,
  "from": this.ethAccount,
  "signature": this.sign
}
```

为了与之前的`PUT`方法有所区分，此处使用`POST`方法作为回传的方法。在此给出完整的前端代码。

```javascript
window.ethereum.request({ method: 'eth_requestAccounts' }).then(
    accounts => {
        this.ethAccount = accounts[0]
        console.log(this.ethAccount);
    }
).then(
    () => {
        axios.put("http://localhost:8787", { "from": this.ethAccount }).then(res => {
            this.nonces = res.data.nonces;
        }).then(() => {
            console.log(this.nonces);
            const msgParams = {
                domain: {
                    chainId: window.ethereum.chainId,
                    name: 'Login',
                    version: '1'
                },
                message: {
                    contents: 'Login',
                    nonces: this.nonces,
                },
                primaryType: 'Login',
                types: {
                    EIP712Domain: [
                        { name: 'name', type: 'string' },
                        { name: 'version', type: 'string' },
                        { name: 'chainId', type: 'uint256' },
                    ],
                    Login: [
                        { name: 'contents', type: 'string' },
                        { name: 'nonces', type: 'uint256' },
                    ],
                },
            };
            const from = this.ethAccount
            ethereum.request({
                method: 'eth_signTypedData_v4',
                params: [from, JSON.stringify(msgParams)],
            }).then((res) => {
                this.sign = res;
                console.log(res);
            }).then(
                () => {
                    axios.post("http://localhost:8787", {
                        "chainId": window.ethereum.chainId,
                        "from": this.ethAccount,
                        "signature": this.sign
                    }).then(
                        res => {
                            if (res.data.verify) {
                                localStorage.setItem("token", res.data.token);
                                localStorage.setItem("expire", Date.now() + 3600000);
                                localStorage.setItem("userName", this.ethAccount);
                                console.log("登录成功");
                            } else {
                                console.log("登录失败，请重新登录");
                            }
                        }
                    )
                })
        })
    }
)
```
核心代码如下：
```javascript
axios.post("http://localhost:8787", {
    "chainId": window.ethereum.chainId,
    "from": this.ethAccount,
    "signature": this.sign
}).then(
    res => {
        if (res.data.verify) {
            localStorage.setItem("token", res.data.token);
            localStorage.setItem("expire", Date.now() + 3600000);
            localStorage.setItem("userName", this.ethAccount);
            console.log("登录成功");
        } else {
            console.log("登录失败，请重新登录");
        }
    }
)
```

向后端使用`POST`方法回传数据，同时接受后端数据，此处依旧不加解释的给出后端回传数据的结构：

登录成功的结果如下：
```javascript
{
  "verify": true, 
  "token": tokenLogin
}
```
登录失败的结果如下：
```javascript
{
  "verify": false
}
```

同时本段代码也实现了在`localStorage`中设置`userName`、`token`、`expire`等字段，具体含义如下：

- userName， 用户名。在此处为用户的以太坊地址
- token，登录凭证，在下一节内容中我们将使用`JWT`实现此功能
- expire，登录过期时间，与`token`类似将使用`JWT`在后端实现

上述内容将直接存储在`localStorage`中，也可以根据您的需求自行更改。

在下一节中，我们将处理客户端发送的数据并返回认证后的数据

## 后端验证签名并生成token

### 验证签名

在此过程中主要使用了`recoverTypedSignature`函数，该函数的文档在[这](https://metamask.github.io/eth-sig-util/modules.html#recoverPersonalSignature)。其主要作用是接受数据结构、签名、版本号三个参数返回用户的地址。

在此我们直接给出该部分的代码：

```javascript
import { recoverTypedSignature } from '@metamask/eth-sig-util';

const data = await request.json();
const chainId = data.chainId;
const from = data.from;
const nonces = await loginKV.get(from);
const msgParams = {
  domain: {
    chainId: chainId,
    name: 'Login',
    version: '1'
  },
  message: {
    contents: 'Login',
    nonces: nonces,
  },
  primaryType: 'Login',
  types: {
    EIP712Domain: [
      { name: 'name', type: 'string' },
      { name: 'version', type: 'string' },
      { name: 'chainId', type: 'uint256' },
    ],
    Login: [
      { name: 'contents', type: 'string' },
      { name: 'nonces', type: 'uint256' },
    ],
  },
};
const signature = data.signature;
const version = "V4";
const recoveredAddr = recoverTypedSignature({
  data: msgParams,
  signature: signature,
  version: version,
});
```

该部分代码首先使用`await request.json()`获取前端回传数据，并将数据赋值给变量。然后，使用`kv`中自带的函数`get`在键值数据库获取到该用户所签名的`nonces`值。最后，根据前端定义的结构化数据形式编写后端数据，并直接调用`recoverTypedSignature`函数获得用户的地址。

### 生成`JWT`凭证

```javascript
import { toChecksumAddress } from 'ethereumjs-util';
import { SignJWT } from 'jose';

if (toChecksumAddress(recoveredAddr) === toChecksumAddress(from)) {

  const tokenLogin = await new SignJWT({ "user_id": from })
    .setProtectedHeader({ alg: 'HS256', typ: 'JWT' })
    .setExpirationTime('1h')
    .sign(Buffer.from(SECRET_KEY, "utf8"));
  console.log(`${from} has been ${tokenLogin}`)
  return new Response(JSON.stringify({ "verify": true, "token": tokenLogin }), {
    headers: {
      'content-type': 'application/json;charset=UTF-8',
      'Access-Control-Allow-Origin': '*'
    },
  });

} else {
  return new Response(JSON.stringify({ "verify": false }), {
    headers: {
      'content-type': 'application/json;charset=UTF-8',
      'Access-Control-Allow-Origin': '*'
    },
  });
}
```
上述代码展示了获取用户的地址地址后，我们可以通过`ethereumjs-util`中的`toChecksumAddress`计算通过签名获得的地址与用户回传的地址是否相同。由于worker此类`serverless`应用的无状态性，我们采用了一种无状态的登录授权方式，即`JWT`。此处使用了`jose`库中的`SignJWT`函数，使用了`HS256`作为签名方式。如果您想了解更多关于`SignJWT`函数的相关内容，您可以参考[文档](https://github.com/panva/jose/blob/main/docs/classes/jwt_sign.SignJWT.md#readme)。


此处使用的`SECRET_KEY`应存储在`worker`的系统变量中，您可以在`wrangler.toml`增加下列内容：

```
[vars]
SECRET_KEY = 密钥
```

上述代码也显示了对前端的回传结果，直接使用if-else逻辑可以简单的完成此项工作。

## 总结

通过上述内容，您基本可以完成一个简单的`metamask`登录系统，较为简单，如果您有任何不了解的内容，可以向我发送邮件。此项目的所有代码可以在[这里](https://github.com/wangshouh/CloudflareWorkerWeb3Login)找到。
