---
title: "Rfc Http2"
date: 2022-07-04T16:40:09+08:00
draft: true
---

## http2 详解

### Frame

所有的帧都以固定的 9 字节报头开始，后面是可变长度的payload。
帧的结构如下

```
HTTP Frame {
  Length(24),
  Type(8),
  Flags(8),
  Reserved(1),
  Stream Identifier(31),
  Frame Payload (..)
}
```

Frame 字段定义如下:

1. Length: Payload 字段的长度（不包含9字节header的长度），超过 2**14（16384）字节的数据不能被发送，除非将 SETTINGS_MAX_FRAME_SIZE 设置为更大的值。
2. Type: Frame的类型。位置类型的Frame必须被舍弃
3. Flags: 为特定帧类型布尔标志的保留字段
4. Reserved: 1 bit 保留字段，直接忽略
5. Stream Identifier: 流的标识符，0x00 保留给与整个连接相关的帧，而不是单个流

帧 payload 的结构和内容和帧类型相关


### Stream

流是在 HTTP/2 连接中客户端与服务端之间独立，双向的帧序列。

客户端是偶数，服务端是奇数。 0x00 用于连接的控制信息

新的 stream 的标识符必须大于当前端口已打开或者保留的流的编号