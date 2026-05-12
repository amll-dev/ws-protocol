# AMLL WebSocket 协议规范

> [!WARNING]
> 本文档为 V1 版本的纯二进制规范。为获得更好的扩展性和开发体验（如逐字音译、播放模式控制等），建议使用基于 JSON 的 **[V2 协议规范 (PROTOCOL_V2.md)](./PROTOCOL_V2.md)**。

## 1. 协议概述

本协议基于二进制 WebSocket 消息实现音乐播放状态同步，包含播放控制、元数据传输、歌词同步等功能。协议使用**小端字节序**进行二进制序列化。

## 2. 数据结构定义

以下数据结构按照从上到下的顺序定义各个属性值。

### 2.1 基础类型

为便于阅读理解，定义以下常见基础数据结构：

* `NullString`: 一个以 `\0` 结尾的 UTF-8 编码字符串。
* `Vec<T>`: 一个以 `u32` 类型开头（表示数据结构 `T` 的数量）的线性数据结构，后紧跟指定数量的 `T` 数据结构。

### 2.2 Artist 艺术家信息

```rust
struct Artist {
    id: NullString,   // 艺术家的唯一标识字符串
    name: NullString  // 艺术家名称
}

```

### 2.3 LyricWord 歌词单词

```rust
struct LyricWord {
    start_time: u64,  // 单词开始时间，单位为毫秒
    end_time: u64,    // 单词结束时间，单位为毫秒
    word: NullString  // 单词内容
}

```

### 2.4 LyricLine 歌词行

```rust
struct LyricLine {
    start_time: u64,                // 歌词行开始时间，单位为毫秒
    end_time: u64,                  // 歌词行结束时间，单位为毫秒
    words: Vec<LyricWord>,          // 单词列表
    translated_lyric: NullString,   // 翻译歌词，如无提供请留空白字符串
    roman_lyric: NullString,        // 罗马音歌词，如无提供请留空白字符串
    flag: u8,                       // 有关该歌词行的属性标记值，可用标记如下：
                                    // - 0b01 : 是否为背景歌词行，等同于 isBG
                                    // - 0b10 : 是否为对唱歌词行，等同于 isDuet
}

```

## 3. 消息主体 (Body)

所有消息都通过 Body 枚举类型进行封装，使用类型为 `u16` 的 Magic Number 区分消息类型。

每个主体都以 Magic Number 开头，随后紧跟需要的其他数据结构体（若有注明）。如无注明，则仅需一个 Magic Number。

以下列出了所有可用的主体信息（标题后的括号内为该 Magic Number 的值）：

### 3.1 状态与同步 (接收/发送)

#### Ping (0)

* **方向**：任意一方发送
* **说明**：心跳请求包。接收方收到后可选择发回 Pong 信息以确认通信情况。
* **附加数据**：无。

#### Pong (1)

* **方向**：任意一方发送
* **说明**：心跳响应包。通常在接收到 Ping 信息后发送。
* **附加数据**：无。

### 3.2 播放器状态同步 (接收端接收)

#### SetMusicInfo (2)

* **说明**：报告当前歌曲的主要信息。
* **数据结构**：

```rust
struct SetMusicInfo {
    music_id: NullString,   // 歌曲的唯一标识字符串
    music_name: NullString, // 歌曲名称
    album_id: NullString,   // 歌曲所属的专辑ID，如果没有可以留空
    album_name: NullString, // 歌曲所属的专辑名称，如果没有可以留空
    artists: Vec<Artist>,   // 歌曲的艺术家/制作者列表
    duration: u64,          // 歌曲的时长，单位为毫秒
}

```

#### SetMusicAlbumCoverImageURI (3)

* **说明**：以 URI 形式报告当前歌曲的专辑图片数据。
* **数据结构**：

```rust
struct SetMusicAlbumCoverImageURI {
    img_url: NullString,   // 歌曲专辑图片对应的资源链接，可以为 HTTP URL 或 Base64 Data URI
}

```

#### SetMusicAlbumCoverImageData (4)

* **说明**：以二进制原始数据形式报告当前歌曲的专辑图片数据。
* **数据结构**：

```rust
struct SetMusicAlbumCoverImageData {
    data: Vec<u8>,   // 歌曲专辑图片对应的原始二进制数据
}

```

#### OnPlayProgress (5)

> 注：该信息不限制报告间隔，因此接收端在进行播放进度展现的时候需要做好数值过渡补偿

* **数据结构**：

```rust
struct OnPlayProgress {
    progress: u64,   // 当前歌曲的播放进度，单位为毫秒
}

```

#### OnVolumeChanged (6)

* **说明**：当发送端音量改变时报告。
* **数据结构**：

```rust
struct OnVolumeChanged {
    volume: f64,   // 音量大小，值域在 [0-1] 内
}

```

#### OnPaused (7)

* **说明**：当歌曲播放被暂停时报告。
* **附加数据**：无。

#### OnResumed (8)

> 注：如果处于歌曲切换的状态下，在发送完歌曲信息后应该发送一次本消息

* **说明**：当歌曲播放被恢复时报告。
* **附加数据**：无。

#### OnAudioData (9)

* **说明**：返回当前正在播放的音频 PCM 数据（2 通道 48000hz i16 采样率交错编码），可用于音频可视化等特效。
* **数据结构**：

```rust
struct OnAudioData {
    data: Vec<u8>,   // 音频数据
}

```

#### SetLyric (10)

* **说明**：设置当前歌曲的歌词数据（结构化数据），接收端可自行决定是否使用。
* **数据结构**：

```rust
struct SetLyric {
    data: Vec<LyricLine>,   // 歌词数据
}

```

#### SetLyricFromTTML (11)

* **说明**：设置当前歌曲的歌词数据（纯 TTML 字符串），接收端可自行决定是否使用。
* **数据结构**：

```rust
struct SetLyricFromTTML {
    data: NullString,   // 歌词数据
}

```

### 3.3 播放控制指令 (发送端发送给播放器)

#### Pause (12)

* **说明**：请求暂停音乐播放。
* **附加数据**：无。

#### Resume (13)

* **说明**：请求恢复音乐播放。
* **附加数据**：无。

#### ForwardSong (14)

* **说明**：请求跳转到下一首歌曲。
* **附加数据**：无。

#### BackwardSong (15)

* **说明**：请求跳转到上一首歌曲。
* **附加数据**：无。

#### SetVolume (16)

* **说明**：请求设置播放音量。
* **数据结构**：

```rust
struct SetVolume {
    volume: f64,   // 音量大小，值域在 [0-1] 内
}

```

#### SeekPlayProgress (17)

* **说明**：请求设置当前播放进度。
* **数据结构**：

```rust
struct SeekPlayProgress {
    progress: u64,   // 播放进度时间位置，单位为毫秒
}

```

## 4. 使用示例 (TypeScript)

### 设置音乐信息

```typescript
import { toBody } from "@applemusic-like-lyrics/ws-protocol";

const encoded = toBody({
    type: "setMusicInfo",
    value: {
        musicId: "1",
        musicName: "2",
        albumId: "3",
        albumName: "4",
        artists: [
            {
                id: "5",
                name: "6",
            },
        ],
        duration: 7,
    },
});

console.log(encoded); // Uint8Array

```

### 处理播放进度更新

```typescript
import { parseBody } from "@applemusic-like-lyrics/ws-protocol";

const body = new Uint8Array(
    [
        0x02, 0x00, // SetMusicInfo
        0x31, 0x00, // musicId: "1"
        0x32, 0x00, // musidName: "2"
        0x33, 0x00, // albumId: "3"
        0x34, 0x00, // albumName: "4"
        0x01, 0x00, 0x00, 0x00, // artists: size 1
        0x35, 0x00, // artist.id: "5"
        0x36, 0x00, // artist.name: "6"
        0x07, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00 // duration: 7
    ]
);

const parsed = parseBody(body);

console.log(parsed);

/* 预期输出：
{
    type: "setMusicInfo",
    value: {
        musicId: "1",
        musicName: "2",
        albumId: "3",
        albumName: "4",
        artists: [
            {
                id: "5",
                name: "6"
            }
        ],
        duration: 7
    }
}
*/
```
