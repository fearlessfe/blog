---
title: "Tendermint abci spec"
date: 2022-05-25T14:07:36+08:00
draft: true
---


## 方法和类型

### 连接

tendermint(区块链)和application(状态机)可以运行在不同的线程，两者通过 socket 连接。

连接可以分为一下四类

1. Consensus connection： InitChain, BeginBlock, DeliverTx, EndBlock, Commit
2. Mempool connection： CheckTx
3. Info connection： Info， Query
4. Snapshot connection: ListSnapshots, LoadSnapshotChunk, OfferSnapshot, ApplySnapshotChunk

### CheckTx

send from Tendermint to the application. 无效的交易会被移出 mempool

### DeliverTx

Deliver tx from Tendermint to the application. 收到非零的回复后，会记录日志。由于交易已经在区块中了，错误不会影响Tendermint 的共识

## proxy 模块

有四类连接： appConnConsensus，appConnMempool， appConnQuery， appConnSnapshot

每类 conn 都是一个 abcicli.Client

client 分为 local 和 remote，remote又分为 socket 和 grpc

## NewNode

1. 初始化DB，blockStore 和 stateStore，
2. 读取 State，from DB or GenesisFile
3. 创建启动App的proxy，根据 config，分为 local 和 remote
4. 启动 EventBus
5. 启动 Indexer
6. 创建 mempoolReactor
7. 启动 evidenceReactor
8. 启动 BlockchainReactor
9. 创建 ConsensusReactor

## Service

Service 定义了一个可以 start, stopped and reset 的接口

```golang
// Service defines a service that can be started, stopped, and reset.
type Service interface {
 // Start the service.
 // If it's already started or stopped, will return an error.
 // If OnStart() returns an error, it's returned by Start()
 Start() error
 OnStart() error

 // Stop the service.
 // If it's already stopped, will return an error.
 // OnStop must never error.
 Stop() error
 OnStop()

 // Reset the service.
 // Panics by default - must be overwritten to enable reset.
 Reset() error
 OnReset() error

 // Return true if the service is running
 IsRunning() bool

 // Quit returns a channel, which is closed once service is stopped.
 Quit() <-chan struct{}

 // String representation of the service
 String() string

 // SetLogger sets a logger.
 SetLogger(log.Logger)
}
```

BaseService 实现了 Service 接口，New 一个 BaseService 需要传一个实现了 Service 的实例，BaseService 只是包装，实际还是调用 实例的方法

```golang
type BaseService struct {
 Logger  log.Logger
 name    string
 started uint32 // atomic
 stopped uint32 // atomic
 quit    chan struct{}

 // The "subclass" of BaseService
 impl Service
}
```

channel 和 reactor 一一对应

## Config

一个tendermint node 配置文件分为一下几类

* BaseConfig: 链id，根目录，创世区块配置，节点证书，DB类型，DB路径，ABCI连接方式和地址等。主要和链相关
* RPC：Tendermint RPC server，监听地址，跨域，RPC 最大连接数，最大 body size 等
* P2P：节点ip，监听地址等
* Mempool：最大交易数，最大交易size，缓存，单个交易最大数，wal地址等
* StateSync：状态同步服务。节点之间通过 RPC 同步状态？
* FastSync：Tendermint fast sync service
* Consensus：共识相关，各阶段时间，wal目录
* TxIndex：交易索引。 null or kv
* Instrumentation：metrics reporting

## erhai-app

默认目录

～/.dcd

```golang

erhai-app/hcd/cmd/root 

initAppConfig 与 app 相关的配置

customAppTemplate  默认模版
customAppConfig  默认的config配置

erhai-chain/server/util

InterceptConfigsPreRunHandler 初始化配置

serverCtx := {
  Viper: viper.New(),
  Config: tmcfg.DefaultConfig(),
  Logger: ZeroLogWrapper{log.Logger},
}

interceptConfigs 解析配置

startInProcess 启动node
```

## newNode 与配置的关系

```golang
1. initDBs
初始化 id 为 blockstore 和 state 的两个数据库。id 需要加上 zone 的前缀

相关配置
Config.DBBackend
Config.DBPath
Config.ArchiveDBAddress
Config.ArchiveCertPath
Config.ArchiveRelativePath

2. LoadStateFromDBOrGenesisDocProvider
从数据库或者Provider读取 Genesis 文件
相关配置
Config.Genesis

3. createAndStartProxyAppConns

baseAppCreator.newApp
// flags
viper.GET("inter-block-cache")
viper.GET("unsafe-skip-upgrades")
("home")
("inv-check-period")
("zone")

4. createAndStartIndexerService

config.TxIndex.Indexer

tx_index 数据库需要加上 zoneId 前缀

5. createAndStartPrivValidatorSocketClient

config.PrivValidatorListenAddr

6. createMempoolAndMempoolReactor

7. createEvidenceReactor

config.FastSync.Version

8. createConsensusReactor

config.Consensus
```

## init 命令

tendermint init 初始化目录

* config
  * config.toml
  * genesis.json
  * node_key.json  // node_key_file
  * priv_validator_key.json  // priv_validator_key_file
* data
  * priv_validator_state.json

BaseConfig 中只有 node_key_file, priv_validator_key_file 属性是不需要每个zone 都生成


## core/store

```golang
// BlockStore struct
// 实现了 core/state 模块中的 BlockStore 接口
// 主要存储以下的数据类型
// - BlockMeta:   Meta information about each block
// - Block part:  Parts of each block, aggregated w/ PartSet
// - Commit:      The commit part of each block, for gossiping precommit votes

NewBlockStore 从数据库中读取key为 blockStore 的数据

type BlockStore struct{
  Size BlockStore中的区块数量
  LoadBaseMeta 读取base block meta,数据库中读取key为 H:1231 的数据
  LoadBlock 读取对应高度的区块，首先读取 BlockMeta，根据meta中的total读取对应的part，数据库中方对应的key是 "P:%v:%v", height, partIndex
  LoadBlockByHash 数据库对应的key是 "BH:%x", hash ，对应的值是区块高度，然后根据高度去获取区块
  LoadBlockPart 根据区块高度和index获取part
  LoadBlockMeta 获取对应高度的区块Meta
  LoadBlockCommit 对应高度的Commit，数据库的key为 "C:%v", height。当前高度 Commit包含大于 2/3 的 Precommit-votes，height+1区块的 LastCommit 引用当前的Commit
  LoadSeenCommit 本地可见的Commit，key为"SC:%v", height，还没被加到下一个区块的commit
  PruneBlocks 移除 base - (height-1) 的区块，返回移除的区块数量。通过 Batch 删除
  SaveBlock 保存 Block， blockParts, Commit
  SaveSeenCommit 
}
```

## core/state

State 结构体描述了 tendermint共识最新commit的区块，能够验证新的区块，

```golang

type State struct {
  MakeBlock 根据当前状态，txs，commit和evidence构造区块，还需要知道proposer的地址。 区分区块的时候，计算默克尔证明 // todo 去掉proposer的地址
  MakeGenesisStateFromFile 从文件中读取 Genesis 数据
}
// 执行block，更新state
// 依赖 Store, App, eventBus, mempool, evpool
type BlockExecutor struct {
  CreateProposalBlock  调用 state.makeBlock,
  ValidateBlock  验证区块是否有效
  ApplyBlock 验证区块，通过app执行交易，触发对应的事件，更新state和response
  Commit lock mempool,运行abci的commit接口，更新mempool
}

// 存储 state 和 ABCI response
type store struct {

}

```


## core/mempool

```golang
type Mempool interface {
  // 调用 abci 接口，检查交易是否有效
  CheckTx(tx types.Tx, callback func(*abci.Response), txInfo TxInfo) error
  // 根据 maxBytes 和 maxGas 从 mempool 读取交易
  ReapMaxBytesMaxGas(maxBytes, maxGas int64) types.Txs
  // 从 mempool 获取最大的交易数
  ReapMaxTxs(max int) types.Txs

  Lock()
	Unlock()
  // 区块 commit 后，去掉对应的tx
  Update(
		blockHeight int64,
		blockTxs types.Txs,
		deliverTxResponses []*abci.ResponseDeliverTx,
		newPreFn PreCheckFunc,
		newPostFn PostCheckFunc,
	) error
  // flushes mempool连接，保证异步的 reqResCb 被调用
  FlushAppConn() error
  // Flush removes all transactions from the mempool and cache
	Flush()
  //  每个height fire 一次
  TxsAvailable() <-chan struct{}
  Size() int

	// TxsBytes returns the total size of all txs in the mempool.
	TxsBytes() int64

	// InitWAL creates a directory for the WAL file and opens a file itself. If
	// there is an error, it will be of type *PathError.
	InitWAL() error

	// CloseWAL closes and discards the underlying WAL file.
	// Any further writes will not be relayed to disk.
	CloseWAL()
}
```


## core/blockchain

### v0