---
title: "基于 Bitcast 模型的 KV 数据库"
date: 2022-07-04T22:09:42+08:00
draft: true
---

一个 Bitcast 实例是一个目录，任一时刻只有一个文件是活跃状态，当一个文件到达阈值后，会创建一个新的文件，旧的文件会被关闭，不在变化。

新增，修改和删除操作都只会增加一条新的操作日志

活动状态的文件只能追加新的 entry 日志，entry 的结构如下

```go
type entry struct {
  crc int32  // 下面所有字段的 crc 值
  tstamp int32  // 32位的时间戳
  ksz uint32 // key 的长度
  value_sz uint32  // value 的长度
  key []byte 
  value []byte
}
```

在文件中追加 entry 记录后，需要更新内存中方的 `keydir` 结构，这是一个 hash table，用来映射每个 key 对应 value 的位置。

`keydir` 的值的结构如下

```go
type value struct {
  file_id string  // 文件名称
  value_sz uint32 // 值的长度
  value_pos uint32 // 值所在文件的偏移量
  tstamp int32 // 32位的时间戳
}
```

随着写入或修改操作越来越多，日志文件会变得越来越多。可以 merge 非活动状态下的文件，压缩日志文件。

遍历所有非活跃的文件，生成一个只包含每个key最新版本的文件。遍历完成后，为每个数据文件生成一个 `hint file`,该文件和数据文件类型，但是只记录值的位置

```go
type entry struct {
  tstamp int32  // 32位的时间戳
  ksz uint32 // key 的长度
  value_sz uint32  // value 的长度
  value_pos uint32 // 值所在的偏移量
  key []byte 
  
}
```