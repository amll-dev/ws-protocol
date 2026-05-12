# AMLL WebSocket Protocol

本模块定义了一个用于跨端同步播放媒体信息的数据传输协议，在双方结构情况支持的情况下，可以以任意形式传输并同步包括歌词在内的音频媒体播放状态。

## 核心文档

* **[AMLL WebSocket 协议 V2 规范 (PROTOCOL_V2.md)](./PROTOCOL_V2.md)** - **(推荐)** 基于 JSON 的结构化协议，支持随机和循环播放模式切换、逐字音译等新特性，并包含二进制通道用来传输高频或大体积数据。
* **[AMLL WebSocket 协议 V1 规范 (PROTOCOL.md)](./PROTOCOL.md)** - **(旧版)** 基于纯二进制序列化的协议，适用于通过互联网传输数据、需要降低延迟的场景。

## 快速入门 (TypeScript)

本库提供了统一的工具函数用于序列化和反序列化消息：

```typescript
// V1 协议使用
import { toBody, parseBody } from "@applemusic-like-lyrics/ws-protocol";

// V2 协议二进制通道使用
import { toBinaryV2, parseBinaryV2 } from "@applemusic-like-lyrics/ws-protocol";

```

具体的调用示例请参阅上述核心文档。

