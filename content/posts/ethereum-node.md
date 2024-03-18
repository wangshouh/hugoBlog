---
title: "以太坊节点快速部署指南"
date: 2024-03-17T22:47:33Z
tags: [ethereum]
---

## 概述

作为以太坊开发者，拥有自己的以太坊节点是一个常规操作。而且大量的以太坊项目也依赖于以太坊节点，本文将向读者介绍如何最快的部署以太坊的完整节点，包括执行层节点与共识层节点。我选择使用 [Reth](https://github.com/paradigmxyz/reth) 作为执行层节点，选择使用 [Lighthouse](https://github.com/sigp/lighthouse) 作为共识层节点。

本文使用了 Docker 作为节点部署方式，相比于其他方式，使用 Docker 比较方便进行节点升级工作。而且为了减少执行层节点的同步时间，我使用了 [Reth Snapshots](https://snapshots.merkle.io/) 直接下载了执行层节点的快照。

## 基本配置

[Reth Snapshots](https://snapshots.merkle.io/) 作为执行层节点的快照，其解压缩后占据了 2.4TB 存储，Lighthouse 建议用户使用了 4 核心 32 GB 内存，所以笔者使用了 4 核心 32 GB 内存与 4 TB 硬盘配置的 AWS 服务器作为节点的部署硬件。

![AWS Ethereum Storage](https://blogimage.4everland.store/ethereumNodeStorage.png)

## 快照下载

**我发现使用快照可能导致同步问题，这可能是由于 merkle 压缩镜像时出现的错误。我不建议使用此方案**

由于快照下载需要消耗大量时间，所以我建议首先安装 `tmux` ，然后在 `tmux` 的终端内进行下载。使用以下命令安装 `tmux` 工具:

```bash

# Ubuntu 或 Debian
$ sudo apt-get install tmux

# CentOS 或 Fedora
$ sudo yum install tmux

# Mac
$ brew install tmux
```

使用以下命令创建用于下载快照的终端:

```bash
tmux new -s snap
```

创建以下 `/mainnet_data/db` 文件夹来存储快照数据库，在该文件夹内运行以下命令:

```bash
wget -O - https://downloads.merkle.io/reth-2024-02-15.tar.lz4 | tar -I lz4 -xvf -
```

>  上述命令需要 `lz4` 解压缩工具，如果您当前系统内不能存在此工具，可以使用 `apt-get install lz4` 命令安装。

![AWS Download Snap](https://blogimage.4everland.store/AWSDowloadSnap.png)

按下 `Ctrl+b d` 退出当前终端。

退出终端后，您可以使用 `tmux ls` 命令查询当前系统后台内运行的 tmux 终端，并可以使用 `tmux attach -t snap` 命令进入下载终端。

## 节点部署

为了方便后期节点升级与快速节点部署，我准备使用 Docker 容器技术，将 Reth 和 Lighthouse 运行在容器环境内。所以在进行后续节点部署前，我们需要安装 Docker 软件:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
```

接下来，我们需要配置 JWT 密钥，使用以下命令:

```bash
sudo mkdir secrets
openssl rand -hex 32 | tr -d "\n" | sudo tee secrets/jwt.hex
```

我们可以进入 `secrets` 文件夹内观察到 `jwt.hex` 文件。

我们使用 `docker compose` 进行节点的部署，在服务器内上传 `docker-compose.yml` 文件，该文件内容如下:

```yaml
version: '3.9'

services:
  reth:
    restart: unless-stopped
    image: ghcr.io/paradigmxyz/reth
    ports:
      - '30303:30303' # eth/66 peering
      - '8545:8545' # rpc
      - '8551:8551' # engine
    volumes:
      - mainnet_data:/root/.local/share/reth/mainnet
      - ./logs:/root/logs
      - ./secrets:/root/jwt
    command: >
      node
      --chain mainnet
      --log.file.directory /root/logs
      --authrpc.addr 0.0.0.0
      --authrpc.port 8551
      --authrpc.jwtsecret /root/jwt/jwt.hex
      --http --http.addr 0.0.0.0 --http.port 8545
      --http.api "eth,net,web3"
  
  lighthouse:
    restart: unless-stopped
    image: sigp/lighthouse
    depends_on:
      - reth
    ports:
      - '5052:5052/tcp' # rpc
      - '9000:9000/tcp' # p2p
      - '9000:9000/udp' # p2p
    volumes:
      - ./lighthousedata:/root/.lighthouse
      - ./secrets:/root/jwt
    command: >
      lighthouse bn
      --network mainnet
      --http --http-address 0.0.0.0
      --execution-endpoint http://reth:8551
      --execution-jwt /root/jwt/jwt.hex
      --checkpoint-sync-url https://mainnet.checkpoint.sigp.io
      --disable-deposit-contract-sync
  
  volumes:
      mainnet_data:
        driver: local
```

上述配置文件启动了 `8545` 端口作为 RPC 请求端口，我们可以通过调用 `127.0.0.1:8545` 进行 `eth_call` 等操作，而 `5052` 为共识层节点的 RPC 请求接口。我们在 `lighthouse` 内使用了 `--disable-deposit-contract-sync` 选项，这是因为我们不使用此节点进行质押操作。如果您是个人质押者，请启用此选项，具体质押流程可以参考 [lighthouse](https://lighthouse-book.sigmaprime.io/mainnet-validator.html) 文档。

在 `docker-compose.yml` 内写入以上内容后，使用 `sudo docker compose up -d` 运行上述 docker 镜像。运行完成后，您可以使用 `docker ps -a` 查询当前系统内运行的 docker 容器情况。

上述配置为一个最简单的方案，我们没有部署节点的监控页面等，在 RETH 的 [etc](https://github.com/paradigmxyz/reth/tree/main/etc) 文件夹内包含一些带有完整监控设施的 docker compose 配置文件。具体内容可以自己查阅 docker compose，而使用方法在 [Reth 文档](https://paradigmxyz.github.io/reth/installation/docker.html#using-docker-compose)内有完整描述。

如果您是个人质押者，而且希望启动 `mev-boost` 提高质押奖励，可以参考 [此 docker compose 配置文件](https://github.com/clifton/reth-lighthouse/tree/main)。

## 等待同步

我们可以使用 `sudo docker logs ubuntu-reth-1 -n 20` 来确定 Reth 节点的运行情况，一般会打印以下日志:

![Reth logs](https://blogimage.4everland.store/RETHLogs.png)

在使用快照的情况下，您可能在日志内看到以下报错，请尝试将 `/mainnet_data/db` 内我们下载的快照版本 `mdbx.dat` 重命名为 `mdbx.dat1`。等待 Reth 重启后生成一个新的 `mdbx.dat`，然后删除 Reth 生成的 `mdbx.dat` 将我们前文重命名的 `mdbx.dat1` 改回 `mdbx.dat` 名称。我们就可以发现 Reth 可以正常启动。

```
thread 'main' panicked at /project/crates/node-core/src/node_config.rs:684:14:
the total difficulty for the latest block is missing, database is corrupt
stack backtrace:
   0: rust_begin_unwind
   1: core::panicking::panic_fmt
   2: core::option::expect_failed
   3: reth_node_core::node_config::NodeConfig::lookup_head
   4: reth_node_builder::builder::NodeBuilder<DB,reth_node_builder::builder::ComponentsState<Types,Components,reth_node_builder::components::traits::FullNodeComponentsAdapter<reth_node_builder::node::FullNodeTypesAdapter<Types,DB,reth_provider::providers::BlockchainProvider<DB,reth_blockchain_tree::shareable::ShareableBlockchainTree<DB,reth_revm::factory::EvmProcessorFactory<<Types as reth_node_builder::node::NodeTypes>::Evm>>>>,<Components as reth_node_builder::components::traits::NodeComponentsBuilder<reth_node_builder::node::FullNodeTypesAdapter<Types,DB,reth_provider::providers::BlockchainProvider<DB,reth_blockchain_tree::shareable::ShareableBlockchainTree<DB,reth_revm::factory::EvmProcessorFactory<<Types as reth_node_builder::node::NodeTypes>::Evm>>>>>>::Pool>>>::launch::{{closure}}
   5: reth_node_core::cli::runner::run_to_completion_or_panic::{{closure}}
   6: reth::cli::Cli<Ext>::run
   7: reth::main
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
```

上述方法可能是一个玄学方案，可能不处理 `mdbx.dat` 文件多重启几次 Reth 节点也可以成功。

当然，我们也可以使用 `sudo docker logs ubuntu-lighthouse-1 -n 20` 来查看 `lighthouse` 的日志。我们可以看到由于 Reth 还没有同步完成，所以此处会显示 `WARN Head is optimistic`。含义为 lighthouse 乐观的接受区块头。

![Lighthouse logs](https://blogimage.4everland.store/lighthouseLogs.png)

## 总结

这篇文章实际上是我边部署节点边写的，所以目前仅能保证节点可以正常跑起来。简单来说，部署一个节点只需要两步:

1. 下载 RETH 快照
2. 运行 docker compose 配置

如果您是一个个人质押者，请参考 [EETH Full Home Staking Setup Guide](https://stakesaurus.gitbook.io/eth-full-home-staking-setup-guide) 文档或者参考 [Staking Directory](https://www.staking.directory/)。由于在质押过程中节点掉线会导致资金损失，所以您可能不能直接使用本文这种部署方案。