# WebSocket 协议 V2 规范

V2 协议是一个基于 JSON 的结构化通信协议。消息被明确区分为“控制指令”和“状态更新”，并提供了针对高频/大体积数据的独立二进制传输通道。

## 1. 顶层消息结构

所有 V2 的 JSON 消息在顶层都被包装为一个统一的 Payload 结构。

### 1.1 基础控制消息

* **Initialize**: `{"type": "initialize"}` - 用于初始化连接。
* **Ping**: `{"type": "ping"}` - 心跳请求。
* **Pong**: `{"type": "pong"}` - 心跳响应。

---

## 2. 状态更新 (State)

由播放器发送给其他端，用于同步当前的播放状态、媒体元数据和歌词等。
结构示例：`{"type": "state", "value": { "update": "...", ... }}`

| Update 类型 (update) | 参数字段                                                                           | 说明                        |
| -------------------- | ---------------------------------------------------------------------------------- | --------------------------- |
| `setMusic`           | `musicId`, `musicName`, `albumId`, `albumName`, `artists` (Artist数组), `duration` | 设置当前播放的歌曲元数据    |
| `setCover`           | 见下方“专辑封面结构”                                                               | 设置歌曲的专辑封面          |
| `setLyric`           | 见下方“歌词内容结构”                                                               | 设置当前歌曲的歌词          |
| `progress`           | `progress` (毫秒)                                                                  | 同步当前播放进度            |
| `volume`             | `volume` (0.0 - 1.0)                                                               | 同步当前音量                |
| `paused`             | 无                                                                                 | 播放已暂停                  |
| `resumed`            | 无                                                                                 | 播放已恢复                  |
| `modeChanged`        | `repeat` (Off/All/One), `shuffle` (布尔值)                                         | 播放模式（循环/随机）已更改 |
| `audioData`          | `data` (数字数组)                                                                  | 传输 PCM 音频数据           |

### 2.1 专辑封面结构

通过 `source` 字段区分封面数据类型：

```jsonc
// URI 格式
{
    "update": "setCover",
    "source": "uri",
    "url": "https://..."
}

// 原始数据格式 (Base64)
{
    "update": "setCover",
    "source": "data",
    "image": {
        "mimeType": "image/jpeg", // 可选
        "data": "..."             // Base64 编码的图片数据
    }
}

```

### 2.2 歌词内容结构

通过 `format` 字段区分歌词格式：

```jsonc
// 结构化逐词歌词
{
    "update": "setLyric",
    "format": "structured",
    "lines": [ /* LyricLine 数组 */ ]
}

// 纯文本 TTML 格式
{
    "update": "setLyric",
    "format": "ttml",
    "data": "..." // TTML 字符串
}

```

---

## 3. 控制指令 (Command)

从控制端发送给播放器，用于控制播放行为。
结构示例：`{"type": "command", "value": { "command": "...", ... }}`

| Command 类型 (command) | 参数字段                     | 说明              |
| ---------------------- | ---------------------------- | ----------------- |
| `pause`                | 无                           | 暂停播放          |
| `resume`               | 无                           | 恢复播放          |
| `forwardSong`          | 无                           | 切换到下一首      |
| `backwardSong`         | 无                           | 切换到上一首      |
| `setVolume`            | `volume` (0.0 - 1.0)         | 设置音量          |
| `seekPlayProgress`     | `progress` (毫秒)            | 跳转播放进度      |
| `setRepeatMode`        | `mode` ("off", "all", "one") | 设置循环模式      |
| `setShuffleMode`       | `enabled` (布尔值)           | 设置/取消随机播放 |

---

## 4. 二进制扩展通道 (BinaryV2)

为了降低通过网络传输大数据（如高频 PCM 数据或庞大的高清图片封面）时的编解码开销及延迟，V2 协议保留了一个纯二进制通道。
该通道采用**小端字节序**序列化。

所有二进制消息以一个 `u16` 的 Magic Number 开头：

### 4.1 OnAudioData (Magic: 0x0000)

传输音频 PCM 数据，通常用于音频可视化。

* **结构**：
* `magic` (u16): `0`
* `size` (u32): 数据字节数组的长度
* `data`: `size` 长度的 `u8` 数组

### 4.2 SetCoverData (Magic: 0x0001)

传输专辑封面的原始二进制图像数据。

* **结构**：
* `magic` (u16): `1`
* `size` (u32): 数据字节数组的长度
* `data`: `size` 长度的 `u8` 数组

---

## 5. TypeScript 类型定义参考

```typescript
export interface Artist {
	id: string;
	name: string;
}

export type RepeatMode = "off" | "all" | "one";

export type StateUpdate =
	| {
			update: "setMusic";
			musicId: string;
			musicName: string;
			albumId: string;
			albumName: string;
			artists: Artist[];
			duration: number;
	  }
	| {
			update: "setCover";
			source: "uri";
			url: string;
	  }
	| {
			update: "setCover";
			source: "data";
			image: {
				mimeType?: string;
				data: string; // Base64 编码
			};
	  }
	| {
			update: "progress";
			progress: number;
	  }
	| {
			update: "volume";
			volume: number;
	  }
	| {
			update: "paused";
	  }
	| {
			update: "resumed";
	  }
	| {
			update: "modeChanged";
			repeat: RepeatMode;
			shuffle: boolean;
	  }
    | {
			update: "audioData";
			data: number[] | Uint8Array;
	  }
	| ({ update: "setLyric" } & LyricContent);

export interface LyricWord {
	startTime: number;
	endTime: number;
	word: string;
	romanWord?: string;
}

export interface LyricLine {
	startTime: number;
	endTime: number;
	words: LyricWord[];
	translatedLyric?: string;
	romanLyric?: string;
	isBG?: boolean;
	isDuet?: boolean;
}

export type LyricContent =
	| {
			format: "structured";
			lines: LyricLine[];
	  }
	| {
			format: "ttml";
			data: string;
	  };

export type Command =
	| { command: "pause" }
	| { command: "resume" }
	| { command: "forwardSong" }
	| { command: "backwardSong" }
	| { command: "setVolume"; volume: number }
	| { command: "seekPlayProgress"; progress: number }
	| { command: "setRepeatMode"; mode: RepeatMode }
	| { command: "setShuffleMode"; enabled: boolean };

export type Message =
	| { type: "initialize" }
	| { type: "ping" }
	| { type: "pong" }
	| { type: "command"; value: Command }
	| { type: "state"; value: StateUpdate };

```

## 6. 使用示例 (TypeScript)

### 示例 A：发送控制指令 (JSON 格式)

V2 协议的普通指令和状态更新通过标准 JSON 传输，无需额外的二进制包装。

```typescript
// 让播放器暂停
const pauseCommand = {
    type: "command",
    value: {
        command: "pause"
    }
};

// 设置循环模式为单曲循环
const repeatModeCommand = {
    type: "command",
    value: {
        command: "setRepeatMode",
        mode: "one"
    }
};

```

### 示例 B：使用二进制发送音频数据

对于高频的音频 PCM 数据，使用 `toBinaryV2` 进行编码以降低性能开销。

```typescript
import { toBinaryV2 } from "@applemusic-like-lyrics/ws-protocol";

const pcmData = new Uint8Array([/* 原始采样数据 */]);

const encoded = toBinaryV2({
    type: "onAudioData",
    value: {
        data: pcmData
    }
});

// 结果前两个字节为 0x00 0x00 (OnAudioData 的 Magic Number)

```

### 示例 C：发送结构化歌词

```typescript
const lyricUpdate = {
    type: "state",
    value: {
        update: "setLyric",
        format: "structured",
        lines: [
            {
                startTime: 1000,
                endTime: 5000,
                words: [{ startTime: 1000, endTime: 2000, word: "Hello" }],
                translatedLyric: "你好",
                isBG: false
            }
        ]
    }
};

```
