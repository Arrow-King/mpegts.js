# HTTP-FLV 直播流格式工程规范

**文档定位：** 供 AI Agent 直接用于 C/C++ 代码审查与开发的精确工程规范。所有字段约束、合法值范围、错误处理要求、参考实现均来自 mpegts.js 源码（`src/demux/flv-demuxer.js`）的实际解析逻辑，具有权威性。

**适用版本：** mpegts.js v1.8.0+（支持 H.264 / H.265 / AV1 / AAC / Opus / FLAC over HTTP-FLV）

---

## 目录

1. [数据类型与字节序约定](#1-数据类型与字节序约定)
2. [FLV 流整体结构](#2-flv-流整体结构)
3. [FLV Header](#3-flv-header)
4. [FLV Tag 通用结构](#4-flv-tag-通用结构)
5. [ScriptData Tag（onMetaData）](#5-scriptdata-tagonmetadata)
6. [Video Tag — H.264 (AVC)](#6-video-tag--h264-avc)
7. [Video Tag — H.265 (HEVC) 旧式](#7-video-tag--h265-hevc-旧式)
8. [Video Tag — Enhanced RTMP（H.265 / AV1）](#8-video-tag--enhanced-rtmph265--av1)
9. [Audio Tag — AAC](#9-audio-tag--aac)
10. [Audio Tag — Enhanced FLV（Opus / FLAC）](#10-audio-tag--enhanced-flvopus--flac)
11. [HTTP 服务器响应头要求](#11-http-服务器响应头要求)
12. [直播流时序规则与约束](#12-直播流时序规则与约束)
13. [完整 C/C++ 参考实现](#13-完整-cc-参考实现)
14. [测试向量（十六进制）](#14-测试向量十六进制)
15. [Agent 代码审查 Checklist](#15-agent-代码审查-checklist)

---

## 1. 数据类型与字节序约定

| 类型名     | 位宽  | 字节序 | 说明 |
|-----------|-------|--------|------|
| `uint8`   | 8     | —      | 无符号字节 |
| `uint16`  | 16    | 大端   | 高字节在低地址 |
| `uint24`  | 24    | 大端   | 3 字节，高字节在低地址 |
| `uint32`  | 32    | 大端   | 高字节在低地址 |
| `int24`   | 24    | 大端   | 3 字节有符号整数（符号扩展至 32 位用于计算） |
| `int32`   | 32    | 大端   | 32 位有符号整数 |

> **规则：** FLV 格式中**所有多字节字段一律使用大端序**，没有例外。

---

## 2. FLV 流整体结构

```
┌─────────────────────────────────────┐
│  FLV Header (9 bytes)               │
├─────────────────────────────────────┤
│  PreviousTagSize0 = 0x00000000      │  (4 bytes)
├─────────────────────────────────────┤
│  Tag: ScriptData / onMetaData       │  (可选，但推荐，必须在所有媒体 Tag 之前)
├─────────────────────────────────────┤
│  Tag: Video — Sequence Header       │  (AVCDecoderConfigurationRecord 或 HEVCDecoderConfigurationRecord)
├─────────────────────────────────────┤
│  Tag: Audio — Sequence Header       │  (AudioSpecificConfig)
├─────────────────────────────────────┤
│  Tag: Video — IDR (Keyframe)        │
│  Tag: Audio — AAC Raw               │
│  Tag: Video — Inter Frame           │
│  Tag: Audio — AAC Raw               │
│  ...  (循环)                         │
├─────────────────────────────────────┤
│  (直播流中每个 GOP 开头重发 Seq Header) │
└─────────────────────────────────────┘
```

**约束：**
- Sequence Header（视频 + 音频）**必须**出现在任何同类型媒体帧之前。
- 直播场景中，**每个 IDR GOP 开头必须重发** Video Sequence Header（以支持中途接入的播放器）。
- 若流仅含视频，无需发送 Audio Sequence Header，反之亦然。

---

## 3. FLV Header

### 字段定义

| 偏移（字节） | 大小（字节） | 字段名          | 类型     | 合法值 / 约束                          |
|------------|------------|-----------------|----------|---------------------------------------|
| 0          | 3          | Signature       | uint8[3] | **必须**为 `0x46 0x4C 0x56`（"FLV"）  |
| 3          | 1          | Version         | uint8    | **必须**为 `0x01`                      |
| 4          | 1          | TypeFlagsAudio  | uint8    | bit2：HasAudio（1=有音频轨道）          |
|            |            | TypeFlagsVideo  |          | bit0：HasVideo（1=有视频轨道）          |
|            |            | （保留位）       |          | 其余 bit **必须**为 0                  |
| 5          | 4          | DataOffset      | uint32   | **必须**为 `0x00000009`（9）           |

### 错误处理

mpegts.js `probe()` 函数的实际校验逻辑（必须全部满足，否则拒绝）：
- `data[0..2] == {0x46, 0x4C, 0x56}` （Signature）
- `data[3] == 0x01` （Version）
- `DataOffset >= 9`

### 示例（音视频流）

```
46 4C 56 01 05 00 00 00 09
│           │  └──────────── DataOffset = 9
│           └─────────────── Flags: bit2=1(Audio), bit0=1(Video)
└─────────────────────────── "FLV" + Version=1
```

---

## 4. FLV Tag 通用结构

每个 Tag 由 **11 字节头 + N 字节数据 + 4 字节尾** 组成，共 `15 + N` 字节。

### 字段定义

| 偏移 | 大小 | 字段名              | 类型   | 合法值 / 约束 |
|------|------|---------------------|--------|--------------|
| 0    | 1    | TagType             | uint8  | `8`=Audio，`9`=Video，`18`=ScriptData；其他值被 mpegts.js 跳过（logged warning） |
| 1    | 3    | DataSize            | uint24 | Tag 数据体字节数，**不含** 11 字节头和 4 字节尾 |
| 4    | 3    | Timestamp           | uint24 | 时间戳低 24 位，单位**毫秒**，大端 |
| 7    | 1    | TimestampExtended   | uint8  | 时间戳高 8 位；实际 ts(ms) = `(TimestampExtended << 24) \| Timestamp` |
| 8    | 3    | StreamID            | uint24 | **必须**为 `0x000000`；mpegts.js 遇到非 0 值会输出 warning 但不拒绝 |
| 11   | N    | Data                | —      | 具体负载，见各节 |
| 11+N | 4    | PreviousTagSize     | uint32 | **必须**等于 `11 + DataSize`；mpegts.js 会校验并 warning，但不中止 |

### 时间戳语义

```
int32_t timestamp_ms = (int32_t)((TimestampExtended << 24) | Timestamp_24bit);
// 直播流：从 0 开始，单调递增，单位毫秒
// 不得出现时间戳回绕（除非 int32 溢出后的自然回绕）
```

---

## 5. ScriptData Tag（onMetaData）

### 数据编码

使用 **AMF0** 格式编码，TagType = 18：

```
[AMF0 Type=0x02][uint16 length=10]["onMetaData"]   ← AMF0 String
[AMF0 Type=0x08][uint32 count]                      ← AMF0 ECMA Array
  key: "duration"        value: AMF0 Number (直播填 0.0)
  key: "width"           value: AMF0 Number (像素宽度)
  key: "height"          value: AMF0 Number (像素高度)
  key: "framerate"       value: AMF0 Number (帧率，如 25.0 / 30.0)
  key: "videodatarate"   value: AMF0 Number (视频码率 Kbps)
  key: "audiodatarate"   value: AMF0 Number (音频码率 Kbps)
  key: "hasAudio"        value: AMF0 Boolean
  key: "hasVideo"        value: AMF0 Boolean
[0x00 0x00 0x09]                                    ← AMF0 Object End Marker
```

### mpegts.js 解析的字段

| 字段             | 类型    | 用途说明 |
|------------------|---------|----------|
| `hasAudio`       | Boolean | 覆盖 FLV Header 的 HasAudio 标志 |
| `hasVideo`       | Boolean | 覆盖 FLV Header 的 HasVideo 标志 |
| `audiodatarate`  | Number  | 填充 MediaInfo.audioDataRate |
| `videodatarate`  | Number  | 填充 MediaInfo.videoDataRate |
| `width`          | Number  | 填充 MediaInfo.width |
| `height`         | Number  | 填充 MediaInfo.height |
| `duration`       | Number  | 填充 MediaInfo.duration（直播填 0） |
| `framerate`      | Number  | 填充参考帧率（fps_num = floor(framerate * 1000)） |

---

## 6. Video Tag — H.264 (AVC)

TagType = 9，CodecID = 7（旧式，bit7 = 0）。

### 6.1 FrameInfo 字节（Tag Data 第 0 字节）

```
bit7   : 0 = 旧式（legacy）FLV，非 Enhanced
bit6-4 : FrameType
           1 = 关键帧 (keyframe / IDR)          ← 必须与实际 IDR NALU 对应
           2 = 非关键帧 (inter frame / P/B)
           3 = 可丢弃非关键帧
           4 = 生成关键帧 (generated keyframe)
           5 = 命令帧 (info/command frame)
bit3-0 : CodecID = 7 (AVC/H.264)
```

### 6.2 Sequence Header（AVCPacketType = 0）

Tag Data 布局（最小 12 字节）：

```
偏移  大小  字段
0     1    FrameInfo = 0x17  (FrameType=1|CodecID=7)
1     1    AVCPacketType = 0x00
2     3    CompositionTime = 0x000000
5     N    AVCDecoderConfigurationRecord
```

**AVCDecoderConfigurationRecord** 字段（ISO 14496-15，最小 7 字节）：

| 偏移 | 大小 | 字段                      | 约束 |
|------|------|---------------------------|------|
| 0    | 1    | configurationVersion      | **必须**为 `1` |
| 1    | 1    | avcProfileIndication      | 来自 SPS[1]；**不得**为 `0` |
| 2    | 1    | profile_compatibility     | 来自 SPS[2] |
| 3    | 1    | AVCLevelIndication        | 来自 SPS[3] |
| 4    | 1    | `0xFC \| lengthSizeMinusOne` | 低 2 位：0=1字节, 1=2字节, 2=3字节, **3=4字节**（推荐） |
| 5    | 1    | `0xE0 \| numSPS`           | 低 5 位：SPS 个数，**不得**为 0 |
| 6    | 2    | spsLength                 | 大端 uint16，SPS NALU 字节数 |
| 8    | N    | SPS NALU                  | **不含**起始码 `00 00 00 01` |
| 8+N  | 1    | numPPS                    | PPS 个数，**不得**为 0 |
| 9+N  | 2    | ppsLength                 | 大端 uint16 |
| 11+N | M    | PPS NALU                  | **不含**起始码 |

**mpegts.js 的实际校验：**
- `configurationVersion != 1` → 报 FORMAT_ERROR
- `avcProfileIndication == 0` → 报 FORMAT_ERROR
- `lengthSizeMinusOne` 不等于 2 或 3（即 NALU 长度前缀不是 3 或 4 字节）→ 报 FORMAT_ERROR
- `numSPS == 0` → 报 FORMAT_ERROR（"No SPS"）
- `numPPS == 0` → 报 FORMAT_ERROR（"No PPS"）

### 6.3 视频帧（AVCPacketType = 1）

Tag Data 布局（最小 5 字节 + NALU 数据）：

```
偏移  大小  字段
0     1    FrameInfo = 0x17(关键帧) 或 0x27(非关键帧)
1     1    AVCPacketType = 0x01
2     3    CompositionTime (int24)：CTS，毫秒
             PTS = DTS(TagTimestamp) + CTS
             纯 I/P 帧填 0；B 帧时 CTS > 0
5     N    NALU 数据（AVCC 格式，见下）
```

**AVCC 格式 NALU 布局**（`lengthSizeMinusOne=3` 即 4 字节长度前缀）：

```
┌──────────────────────────────────────────────┐
│ [4字节大端长度 naluLen1][NALU1 数据 naluLen1字节] │
│ [4字节大端长度 naluLen2][NALU2 数据 naluLen2字节] │
│ ...                                           │
└──────────────────────────────────────────────┘
```

**严格约束：**
- NALU 数据中**不得**包含起始码 `00 00 00 01`（这是 Annex-B，**不兼容**）。
- `naluLen` 必须等于后续 NALU 字节数，不包含自身的 4 字节。
- `naluLen > (DataSize - 5 - 已解析字节数 - lengthSize)` → mpegts.js 报 warning 并丢弃。

### 6.4 序列结束（AVCPacketType = 2）

```
偏移  大小  字段
0     1    FrameInfo = 0x17
1     1    AVCPacketType = 0x02
2     3    CompositionTime = 0x000000
```

DataSize = 5，无额外数据。

---

## 7. Video Tag — H.265 (HEVC) 旧式

TagType = 9，CodecID = 12，bit7 = 0（旧式），结构与 H.264 相似。

### 7.1 FrameInfo 字节

```
bit7   : 0 (旧式)
bit6-4 : FrameType（含义同 H.264）
bit3-0 : CodecID = 12 (HEVC)
```

### 7.2 Sequence Header（HEVCPacketType = 0）

Tag Data 布局：

```
偏移  大小  字段
0     1    FrameInfo = 0x1C (FrameType=1|CodecID=12)
1     1    HEVCPacketType = 0x00
2     3    CompositionTime = 0x000000
5     N    HEVCDecoderConfigurationRecord（最小 23 字节）
```

**HEVCDecoderConfigurationRecord** 字段（ISO 14496-15:2014，固定头 23 字节）：

| 偏移 | 大小 | 字段 | 约束 |
|------|------|------|------|
| 0    | 1    | configurationVersion | `0` 或 `1`（mpegts.js 同时接受） |
| 1    | 1    | `general_profile_space(2)\|general_tier_flag(1)\|general_profile_idc(5)` | `general_profile_idc` **不得**为 0 |
| 2    | 4    | general_profile_compatibility_flags | 大端 |
| 6    | 6    | general_constraint_indicator_flags | |
| 12   | 1    | general_level_idc | |
| 13   | 2    | `0xF000 \| min_spatial_segmentation_idc` | 大端 |
| 15   | 1    | `0xFC \| parallelismType` | |
| 16   | 1    | `0xFC \| chromaFormat` | |
| 17   | 1    | `0xF8 \| bitDepthLumaMinus8` | |
| 18   | 1    | `0xF8 \| bitDepthChromaMinus8` | |
| 19   | 2    | avgFrameRate | 大端 |
| 21   | 1    | `constantFrameRate(2)\|numTemporalLayers(3)\|temporalIdNested(1)\|lengthSizeMinusOne(2)` | `lengthSizeMinusOne` 同 AVC：推荐 3（4 字节前缀） |
| 22   | 1    | numOfArrays | NALU 类型数组个数 |
| 23   | …    | arrays | 见下 |

**arrays 格式**（每组）：

```
1字节  array_completeness(1) | reserved(1) | NAL_unit_type(6)
         常见类型：32=VPS, 33=SPS, 34=PPS
2字节  numNalus (大端)
foreach nalu:
  2字节  naluLength (大端)
  N字节  NALU 数据（不含起始码）
```

**mpegts.js 的实际校验：**
- 总 DataSize < 22 → warning，丢弃
- `(version != 0 && version != 1) || general_profile_idc == 0` → 报 FORMAT_ERROR
- `lengthSizeMinusOne` 转换后不等于 2 或 3 → 报 FORMAT_ERROR

### 7.3 视频帧（HEVCPacketType = 1）

Tag Data 布局同 AVC，区别在于 FrameInfo 的 CodecID = 12，以及 NALU 类型判断：

```c
// HEVC NALU 类型提取（取 NALU 第 0 字节 bits[14:9]）
uint8_t nal_unit_type = (nalu_data[0] >> 1) & 0x3F;
// 关键帧类型（IRAP）：
//   19 = IDR_W_RADL
//   20 = IDR_N_LP
//   21 = CRA_NUT
```

---

## 8. Video Tag — Enhanced RTMP（H.265 / AV1）

适用于 mpegts.js v1.7.3+（H.265）和 v1.8.0+（AV1），TagType = 9，**bit7 = 1**。

### 8.1 ExFrameInfo 字节

```
bit7   : 1 = Enhanced FLV（IsExHeader 标志）
bit6-4 : FrameType（同旧式）
bit3-0 : PacketType
           0 = SequenceStart（Decoder Config）
           1 = CodedFrames（含 CTS）
           2 = SequenceEnd
           3 = CodedFramesX（CTS 隐含为 0，无 3 字节 CTS 字段）
```

### 8.2 公共头（Tag Data 偏移 0~4）

```
偏移  大小  字段
0     1    ExFrameInfo（bit7=1, FrameType, PacketType）
1     4    FourCC（ASCII）
             "hvc1" = HEVC / H.265
             "av01" = AV1
```

### 8.3 PacketType = 0（SequenceStart）

```
偏移  大小  字段
0     1    ExFrameInfo = (1<<7)|(FrameType<<4)|0x00
1     4    FourCC ("hvc1" 或 "av01")
5     N    Decoder Configuration Record
             hvc1: HEVCDecoderConfigurationRecord（同第 7 节，offseted by 0）
             av01: AV1CodecConfigurationRecord（见下）
```

**AV1CodecConfigurationRecord（最小 4 字节）：**

```
bit7   : marker = 1
bit6-0 : version = 1         ← mpegts.js 校验，不等于 1 则报 FORMAT_ERROR
bit7-5 : seq_profile
bit4-0 : seq_level_idx_0
bit7   : seq_tier_0
bit6-1 : high_bitdepth(1)|twelve_bit(1)|monochrome(1)|chroma_subsampling_x(1)|chroma_subsampling_y(1)|chroma_sample_position(2)
bit0   : initial_presentation_delay_present
...
[剩余字节: configOBUs —— 包含 Sequence Header OBU]
```

### 8.4 PacketType = 1（CodedFrames）

```
偏移  大小  字段
0     1    ExFrameInfo = (1<<7)|(FrameType<<4)|0x01
1     4    FourCC
5     3    CompositionTime int24（毫秒）
8     N    NALU 数据（AVCC 格式，4 字节长度前缀）
```

### 8.5 PacketType = 3（CodedFramesX，无 CTS）

```
偏移  大小  字段
0     1    ExFrameInfo = (1<<7)|(FrameType<<4)|0x03
1     4    FourCC
5     N    NALU 数据（AVCC 格式，CTS 隐含为 0）
```

### 8.6 PacketType = 2（SequenceEnd）

```
偏移  大小  字段
0     1    ExFrameInfo = (1<<7)|(FrameType<<4)|0x02
1     4    FourCC
```

DataSize = 5，无额外数据。

---

## 9. Audio Tag — AAC

TagType = 8，SoundFormat = 10（AAC），这是最常用的音频格式。

### 9.1 SoundSpec 字节（Tag Data 第 0 字节）

```
bit7-4 : SoundFormat
           10 = AAC（最常用）
           2  = MP3
           3  = PCM (little-endian)
           9  = Enhanced FLV Audio（见第 10 节）
           其他值 → mpegts.js 报 CODEC_UNSUPPORTED

bit3-2 : SoundRate（当 SoundFormat=10 时，实际采样率取自 AudioSpecificConfig）
           0 = 5500 Hz
           1 = 11025 Hz
           2 = 22050 Hz
           3 = 44100 Hz（推荐填写此值）
           4 = 48000 Hz

bit1   : SoundSize  0=8bit, 1=16bit（推荐 1）
bit0   : SoundType  0=Mono, 1=Stereo（推荐 1）
```

> **注意：** 当 SoundFormat=10(AAC) 时，SoundRate/SoundSize/SoundType 字段仅作提示，实际参数从 AudioSpecificConfig 读取，mpegts.js 会用后者覆盖前者。推荐固定填写 `0xAF`。

### 9.2 Sequence Header（AACPacketType = 0）

Tag Data 布局（最小 4 字节）：

```
偏移  大小  字段
0     1    SoundSpec = 0xAF（SoundFormat=10|Rate=3|Size=1|Type=1）
1     1    AACPacketType = 0x00
2     N    AudioSpecificConfig（通常 2 字节，HE-AAC 扩展时 4 字节）
```

**AudioSpecificConfig** 位级字段（ISO 14496-3，按比特对齐）：

```
bits  字段                    合法值
5     audioObjectType         2=AAC-LC, 5=HE-AAC SBR, 29=PS
4     samplingFrequencyIndex  查表（见下）；值 0xF 时后跟 24 位直接频率值
4     channelConfiguration    1=Mono, 2=Stereo, 3=3ch, 4=4ch, 5=5ch, 6=5.1ch, 7=7.1ch
                               0=从 bitstream 读取（需要额外 PCE 数据）
```

**samplingFrequencyIndex 对照表：**

| 索引 | 0     | 1     | 2     | 3     | 4     | 5     | 6     | 7     | 8     | 9     | 10    | 11   | 12   |
|------|-------|-------|-------|-------|-------|-------|-------|-------|-------|-------|-------|------|------|
| Hz   | 96000 | 88200 | 64000 | 48000 | 44100 | 32000 | 24000 | 22050 | 16000 | 12000 | 11025 | 8000 | 7350 |

**常用 2 字节 AudioSpecificConfig 速查：**

| 配置                   | 字节值（十六进制） | 二进制拆解 |
|------------------------|------------------|-----------|
| AAC-LC, 44100Hz, Stereo | `0x12 0x10`     | OT=2, SFI=4, CH=2 |
| AAC-LC, 48000Hz, Stereo | `0x11 0x90`     | OT=2, SFI=3, CH=2 |
| AAC-LC, 32000Hz, Stereo | `0x14 0x10`     | OT=2, SFI=5, CH=2 |
| AAC-LC, 22050Hz, Stereo | `0x16 0x10`     | OT=2, SFI=6, CH=2 |
| AAC-LC, 44100Hz, Mono   | `0x12 0x08`     | OT=2, SFI=4, CH=1 |
| AAC-LC, 48000Hz, Mono   | `0x11 0x88`     | OT=2, SFI=3, CH=1 |

**mpegts.js 的实际校验：**
- `samplingIndex < 0 || samplingIndex >= 13` → 报 FORMAT_ERROR（"AAC invalid sampling frequency index"）
- `channelConfig < 0 || channelConfig >= 8` → 报 FORMAT_ERROR（"AAC invalid channel configuration"）

### 9.3 AAC Raw Frame（AACPacketType = 1）

Tag Data 布局（最小 3 字节）：

```
偏移  大小  字段
0     1    SoundSpec = 0xAF
1     1    AACPacketType = 0x01
2     N    裸 AAC 帧数据（不含 ADTS 头）
```

**严格约束：**
- **不得**包含 ADTS 头（`0xFF 0xF?` 开头的 7 或 9 字节头）。
- AAC 帧数据直接为 MPEG-4 AAC bitstream。

---

## 10. Audio Tag — Enhanced FLV（Opus / FLAC）

适用于 mpegts.js v1.8.0+，SoundFormat = 9。

### 10.1 SoundSpec 字节

```
bit7-4 : SoundFormat = 9 (Enhanced FLV Audio 标志)
bit3-0 : PacketType
           0 = SequenceStart
           1 = CodedFrames
           2 = SequenceEnd
```

### 10.2 FourCC（Tag Data 字节 1~4）

| FourCC | 编解码器 |
|--------|---------|
| `Opus` | Opus 音频（`0x4F 0x70 0x75 0x73`） |
| `fLaC` | FLAC 音频（`0x66 0x4C 0x61 0x43`） |

### 10.3 PacketType = 0（SequenceStart）

Opus：`SoundSpec(1) + "Opus"(4) + Opus Identification Header(≥8字节)`

FLAC：`SoundSpec(1) + "fLaC"(4) + FLAC STREAMINFO block`

### 10.4 PacketType = 1（CodedFrames）

`SoundSpec(1) + FourCC(4) + 音频编码数据`

---

## 11. HTTP 服务器响应头要求

```http
HTTP/1.1 200 OK
Content-Type: video/x-flv
Transfer-Encoding: chunked
Connection: keep-alive
Cache-Control: no-cache
Access-Control-Allow-Origin: *
Access-Control-Allow-Headers: Range
Access-Control-Expose-Headers: Content-Length
```

| 头字段 | 要求 | 违反后果 |
|--------|------|---------|
| `Content-Type` | 必须为 `video/x-flv` | 浏览器可能拒绝 |
| `Transfer-Encoding: chunked` | 直播流必须使用，不设置 `Content-Length` | 流无法持续输出 |
| `Connection: keep-alive` | 必须保持长连接 | 流被中断 |
| `Cache-Control: no-cache` | 强烈建议，防止代理缓存 | 播放延迟增加 |
| `Access-Control-Allow-Origin` | 跨域时**必须**设置，直接赋值 `*` 或指定域名 | mpegts.js 无法发起 fetch 请求 |

---

## 12. 直播流时序规则与约束

### 规则总览

| # | 规则 | 来源 |
|---|------|------|
| 1 | Video Sequence Header 必须在第一个视频媒体帧之前发送 | flv-demuxer.js `_videoInitialMetadataDispatched` |
| 2 | Audio Sequence Header 必须在第一个音频媒体帧之前发送 | flv-demuxer.js `_audioInitialMetadataDispatched` |
| 3 | 每个 IDR GOP 开头**必须重发** Video Sequence Header | mpegts.js 中途接入支持 |
| 4 | 关键帧的 FrameType **必须**为 1 | flv-demuxer.js `keyframe = (frameType === 1)` |
| 5 | 时间戳单位为**毫秒**，从 0 开始，单调递增 | 全部时间戳计算逻辑 |
| 6 | 音视频时间戳偏差**不应**超过 500ms | 避免 buffer 累积 |
| 7 | StreamID **必须**为 0 | flv-demuxer.js 校验 |
| 8 | PreviousTagSize = 11 + DataSize | flv-demuxer.js 校验 |
| 9 | FLV Header 后紧跟 PreviousTagSize0 = 0 | flv-demuxer.js `_firstParse` 逻辑 |
| 10 | AAC 数据为裸帧，不含 ADTS 头 | flv-demuxer.js AAC 解析 |
| 11 | NALU 使用 AVCC 格式（4 字节长度前缀），不使用 Annex-B | flv-demuxer.js NALU 解析 |

---

## 13. 完整 C/C++ 参考实现

以下为可直接编译的 C99 参考实现，涵盖所有核心功能。

### 13.1 头文件 `flv_muxer.h`

```c
#ifndef FLV_MUXER_H
#define FLV_MUXER_H

#include <stdint.h>
#include <stddef.h>
#include <stdio.h>

#ifdef __cplusplus
extern "C" {
#endif

/* ──────────────────────────────────────────────
 * 常量定义（来自 FLV 规范 + mpegts.js 源码）
 * ────────────────────────────────────────────── */

/* TagType */
#define FLV_TAG_AUDIO       8
#define FLV_TAG_VIDEO       9
#define FLV_TAG_SCRIPTDATA  18

/* Video CodecID（旧式，bit7=0） */
#define FLV_CODEC_AVC       7   /* H.264 */
#define FLV_CODEC_HEVC      12  /* H.265（旧式） */

/* Video FrameType */
#define FLV_FRAME_KEY       1
#define FLV_FRAME_INTER     2

/* AVC/HEVC PacketType */
#define FLV_PKT_SEQ_HEADER  0
#define FLV_PKT_NALU        1
#define FLV_PKT_SEQ_END     2

/* Enhanced RTMP PacketType */
#define FLV_EX_PKT_SEQ_START    0
#define FLV_EX_PKT_CODED_FRAMES 1
#define FLV_EX_PKT_SEQ_END      2
#define FLV_EX_PKT_CODED_FRAMESX 3

/* Audio SoundFormat */
#define FLV_SOUND_MP3       2
#define FLV_SOUND_PCM_LE    3
#define FLV_SOUND_AAC       10
#define FLV_SOUND_ENHANCED  9   /* Enhanced FLV audio */

/* AAC PacketType */
#define FLV_AAC_SEQ_HEADER  0
#define FLV_AAC_RAW         1

/* ──────────────────────────────────────────────
 * 数据结构
 * ────────────────────────────────────────────── */

typedef struct {
    uint8_t  profile;          /* SPS[1] */
    uint8_t  compat;           /* SPS[2] */
    uint8_t  level;            /* SPS[3] */
    const uint8_t *sps;        /* SPS NALU，不含起始码 */
    uint32_t sps_len;
    const uint8_t *pps;        /* PPS NALU，不含起始码 */
    uint32_t pps_len;
} FlvAVCConfig;

typedef struct {
    const uint8_t *hvcc_data;  /* 完整 HEVCDecoderConfigurationRecord */
    uint32_t       hvcc_len;   /* 必须 >= 23 字节 */
} FlvHEVCConfig;

typedef struct {
    uint8_t  audio_object_type;   /* 2=AAC-LC, 5=HE-AAC */
    uint8_t  sampling_freq_index; /* 0-12，见采样率对照表 */
    uint8_t  channel_config;      /* 1=Mono, 2=Stereo */
} FlvAACConfig;

/* ──────────────────────────────────────────────
 * 函数声明
 * ────────────────────────────────────────────── */

/* FLV Header + PreviousTagSize0 (共 13 字节) */
int flv_write_header(FILE *fp, int has_audio, int has_video);

/* H.264 Sequence Header */
int flv_write_avc_seq_header(FILE *fp, const FlvAVCConfig *cfg);

/* H.264 视频帧（AVCC 格式输入） */
int flv_write_avc_frame(FILE *fp, uint32_t ts_ms, int is_key,
                         int32_t cts_ms,
                         const uint8_t *avcc_data, uint32_t avcc_len);

/* H.265 Sequence Header（旧式 CodecID=12） */
int flv_write_hevc_seq_header(FILE *fp, const FlvHEVCConfig *cfg);

/* H.265 视频帧（旧式） */
int flv_write_hevc_frame(FILE *fp, uint32_t ts_ms, int is_key,
                          int32_t cts_ms,
                          const uint8_t *avcc_data, uint32_t avcc_len);

/* H.265 Sequence Header（Enhanced RTMP, FourCC="hvc1"） */
int flv_write_enhanced_hevc_seq_header(FILE *fp, const FlvHEVCConfig *cfg);

/* H.265 视频帧（Enhanced RTMP） */
int flv_write_enhanced_hevc_frame(FILE *fp, uint32_t ts_ms, int is_key,
                                   int32_t cts_ms,
                                   const uint8_t *avcc_data, uint32_t avcc_len);

/* AAC Sequence Header */
int flv_write_aac_seq_header(FILE *fp, const FlvAACConfig *cfg);

/* AAC Sequence Header（使用原始 AudioSpecificConfig 字节） */
int flv_write_aac_seq_header_raw(FILE *fp, const uint8_t *asc, uint32_t asc_len);

/* AAC 音频帧（裸 AAC，不含 ADTS） */
int flv_write_aac_frame(FILE *fp, uint32_t ts_ms,
                         const uint8_t *aac_data, uint32_t aac_len);

/* Annex-B 转 AVCC（调用方负责 free() 返回值） */
uint8_t *flv_annexb_to_avcc(const uint8_t *annexb, uint32_t annexb_len,
                              uint32_t *out_len);

/* 去除 ADTS 头，返回裸 AAC 帧指针（指向原始缓冲区内部，无需 free） */
const uint8_t *flv_strip_adts(const uint8_t *adts, uint32_t adts_len,
                                uint32_t *raw_len);

/* 构造 2 字节 AudioSpecificConfig */
int flv_make_asc(const FlvAACConfig *cfg, uint8_t out[2]);

#ifdef __cplusplus
}
#endif
#endif /* FLV_MUXER_H */
```

### 13.2 实现文件 `flv_muxer.c`

```c
#include "flv_muxer.h"
#include <stdlib.h>
#include <string.h>
#include <assert.h>

/* ────────────────────────────────────────
 * 内部辅助：大端写入
 * ──────────────────────────────────────── */

static int w8(FILE *fp, uint8_t v) {
    return fwrite(&v, 1, 1, fp) == 1 ? 0 : -1;
}

static int w24be(FILE *fp, uint32_t v) {
    uint8_t b[3] = { (uint8_t)(v >> 16), (uint8_t)(v >> 8), (uint8_t)v };
    return fwrite(b, 1, 3, fp) == 3 ? 0 : -1;
}

static int w32be(FILE *fp, uint32_t v) {
    uint8_t b[4] = {
        (uint8_t)(v >> 24), (uint8_t)(v >> 16),
        (uint8_t)(v >> 8),  (uint8_t)v
    };
    return fwrite(b, 1, 4, fp) == 4 ? 0 : -1;
}

static int wbuf(FILE *fp, const uint8_t *data, uint32_t len) {
    return (len == 0 || fwrite(data, 1, len, fp) == len) ? 0 : -1;
}

/* ────────────────────────────────────────
 * 写通用 Tag（11字节头 + 数据 + 4字节尾）
 * ──────────────────────────────────────── */

static int write_tag(FILE *fp, uint8_t tag_type, uint32_t ts_ms,
                     const uint8_t *data, uint32_t data_len) {
    if (w8(fp, tag_type) < 0) return -1;
    if (w24be(fp, data_len) < 0) return -1;
    if (w24be(fp, ts_ms & 0x00FFFFFF) < 0) return -1;   /* 低 24 位 */
    if (w8(fp, (ts_ms >> 24) & 0xFF) < 0) return -1;    /* 高 8 位 */
    if (w24be(fp, 0) < 0) return -1;                     /* StreamID = 0 */
    if (wbuf(fp, data, data_len) < 0) return -1;
    if (w32be(fp, 11 + data_len) < 0) return -1;        /* PreviousTagSize */
    return 0;
}

/* ────────────────────────────────────────
 * FLV Header
 * ──────────────────────────────────────── */

int flv_write_header(FILE *fp, int has_audio, int has_video) {
    uint8_t flags = 0;
    if (has_audio) flags |= 0x04;  /* bit2 */
    if (has_video) flags |= 0x01;  /* bit0 */

    if (fwrite("FLV", 1, 3, fp) != 3) return -1;   /* Signature */
    if (w8(fp, 0x01) < 0) return -1;                /* Version = 1 */
    if (w8(fp, flags) < 0) return -1;               /* TypeFlags */
    if (w32be(fp, 9) < 0) return -1;                /* DataOffset = 9 */
    if (w32be(fp, 0) < 0) return -1;                /* PreviousTagSize0 = 0 */
    return 0;
}

/* ────────────────────────────────────────
 * H.264 Sequence Header
 * ──────────────────────────────────────── */

int flv_write_avc_seq_header(FILE *fp, const FlvAVCConfig *cfg) {
    /* 最大 buf 大小：5(tag data prefix) + 7(AVCC fixed) + sps_len + 2 + pps_len + 2 + 1 */
    uint32_t buf_len = 5 + 7 + cfg->sps_len + 3 + cfg->pps_len;
    uint8_t *buf = (uint8_t *)malloc(buf_len);
    if (!buf) return -1;

    uint32_t off = 0;
    buf[off++] = 0x17;                              /* FrameType=1|CodecID=7 */
    buf[off++] = FLV_PKT_SEQ_HEADER;
    buf[off++] = 0x00;                              /* CTS = 0 */
    buf[off++] = 0x00;
    buf[off++] = 0x00;

    /* AVCDecoderConfigurationRecord */
    buf[off++] = 0x01;                              /* configurationVersion */
    buf[off++] = cfg->profile;                      /* avcProfileIndication */
    buf[off++] = cfg->compat;                       /* profile_compatibility */
    buf[off++] = cfg->level;                        /* AVCLevelIndication */
    buf[off++] = 0xFF;                              /* lengthSizeMinusOne=3 (4字节) */
    buf[off++] = 0xE1;                              /* numSPS=1 */
    buf[off++] = (uint8_t)(cfg->sps_len >> 8);
    buf[off++] = (uint8_t)(cfg->sps_len);
    memcpy(buf + off, cfg->sps, cfg->sps_len); off += cfg->sps_len;
    buf[off++] = 0x01;                              /* numPPS=1 */
    buf[off++] = (uint8_t)(cfg->pps_len >> 8);
    buf[off++] = (uint8_t)(cfg->pps_len);
    memcpy(buf + off, cfg->pps, cfg->pps_len); off += cfg->pps_len;

    int ret = write_tag(fp, FLV_TAG_VIDEO, 0, buf, off);
    free(buf);
    return ret;
}

/* ────────────────────────────────────────
 * H.264 视频帧
 * ──────────────────────────────────────── */

int flv_write_avc_frame(FILE *fp, uint32_t ts_ms, int is_key,
                         int32_t cts_ms,
                         const uint8_t *avcc_data, uint32_t avcc_len) {
    uint32_t buf_len = 5 + avcc_len;
    uint8_t *buf = (uint8_t *)malloc(buf_len);
    if (!buf) return -1;

    uint32_t off = 0;
    buf[off++] = (uint8_t)(is_key ? 0x17 : 0x27);  /* FrameType|CodecID=7 */
    buf[off++] = FLV_PKT_NALU;
    /* CompositionTime: 24位有符号，大端 */
    buf[off++] = (uint8_t)((cts_ms >> 16) & 0xFF);
    buf[off++] = (uint8_t)((cts_ms >> 8)  & 0xFF);
    buf[off++] = (uint8_t)( cts_ms        & 0xFF);
    memcpy(buf + off, avcc_data, avcc_len); off += avcc_len;

    int ret = write_tag(fp, FLV_TAG_VIDEO, ts_ms, buf, off);
    free(buf);
    return ret;
}

/* ────────────────────────────────────────
 * H.265 Sequence Header（旧式 CodecID=12）
 * ──────────────────────────────────────── */

int flv_write_hevc_seq_header(FILE *fp, const FlvHEVCConfig *cfg) {
    uint32_t buf_len = 5 + cfg->hvcc_len;
    uint8_t *buf = (uint8_t *)malloc(buf_len);
    if (!buf) return -1;

    uint32_t off = 0;
    buf[off++] = 0x1C;                              /* FrameType=1|CodecID=12 */
    buf[off++] = FLV_PKT_SEQ_HEADER;
    buf[off++] = 0x00;
    buf[off++] = 0x00;
    buf[off++] = 0x00;
    memcpy(buf + off, cfg->hvcc_data, cfg->hvcc_len); off += cfg->hvcc_len;

    int ret = write_tag(fp, FLV_TAG_VIDEO, 0, buf, off);
    free(buf);
    return ret;
}

/* ────────────────────────────────────────
 * H.265 视频帧（旧式）
 * ──────────────────────────────────────── */

int flv_write_hevc_frame(FILE *fp, uint32_t ts_ms, int is_key,
                          int32_t cts_ms,
                          const uint8_t *avcc_data, uint32_t avcc_len) {
    uint32_t buf_len = 5 + avcc_len;
    uint8_t *buf = (uint8_t *)malloc(buf_len);
    if (!buf) return -1;

    uint32_t off = 0;
    buf[off++] = (uint8_t)(is_key ? 0x1C : 0x2C);  /* FrameType|CodecID=12 */
    buf[off++] = FLV_PKT_NALU;
    buf[off++] = (uint8_t)((cts_ms >> 16) & 0xFF);
    buf[off++] = (uint8_t)((cts_ms >> 8)  & 0xFF);
    buf[off++] = (uint8_t)( cts_ms        & 0xFF);
    memcpy(buf + off, avcc_data, avcc_len); off += avcc_len;

    int ret = write_tag(fp, FLV_TAG_VIDEO, ts_ms, buf, off);
    free(buf);
    return ret;
}

/* ────────────────────────────────────────
 * H.265 Sequence Header（Enhanced RTMP）
 * ──────────────────────────────────────── */

int flv_write_enhanced_hevc_seq_header(FILE *fp, const FlvHEVCConfig *cfg) {
    /* ExFrameInfo: bit7=1(ExHeader), FrameType=1(key), PacketType=0(SeqStart) */
    uint32_t buf_len = 5 + cfg->hvcc_len;
    uint8_t *buf = (uint8_t *)malloc(buf_len);
    if (!buf) return -1;

    uint32_t off = 0;
    buf[off++] = 0x80 | (FLV_FRAME_KEY << 4) | FLV_EX_PKT_SEQ_START; /* 0x90 */
    buf[off++] = 'h';  /* FourCC "hvc1" */
    buf[off++] = 'v';
    buf[off++] = 'c';
    buf[off++] = '1';
    memcpy(buf + off, cfg->hvcc_data, cfg->hvcc_len); off += cfg->hvcc_len;

    int ret = write_tag(fp, FLV_TAG_VIDEO, 0, buf, off);
    free(buf);
    return ret;
}

/* ────────────────────────────────────────
 * H.265 视频帧（Enhanced RTMP，PacketType=1，含 CTS）
 * ──────────────────────────────────────── */

int flv_write_enhanced_hevc_frame(FILE *fp, uint32_t ts_ms, int is_key,
                                   int32_t cts_ms,
                                   const uint8_t *avcc_data, uint32_t avcc_len) {
    uint8_t frame_type = (uint8_t)(is_key ? FLV_FRAME_KEY : FLV_FRAME_INTER);
    uint32_t buf_len = 5 + 3 + avcc_len;
    uint8_t *buf = (uint8_t *)malloc(buf_len);
    if (!buf) return -1;

    uint32_t off = 0;
    buf[off++] = (uint8_t)(0x80 | (frame_type << 4) | FLV_EX_PKT_CODED_FRAMES);
    buf[off++] = 'h'; buf[off++] = 'v'; buf[off++] = 'c'; buf[off++] = '1';
    /* CompositionTime int24 */
    buf[off++] = (uint8_t)((cts_ms >> 16) & 0xFF);
    buf[off++] = (uint8_t)((cts_ms >> 8)  & 0xFF);
    buf[off++] = (uint8_t)( cts_ms        & 0xFF);
    memcpy(buf + off, avcc_data, avcc_len); off += avcc_len;

    int ret = write_tag(fp, FLV_TAG_VIDEO, ts_ms, buf, off);
    free(buf);
    return ret;
}

/* ────────────────────────────────────────
 * AAC: 构造 AudioSpecificConfig
 * ──────────────────────────────────────── */

int flv_make_asc(const FlvAACConfig *cfg, uint8_t out[2]) {
    if (cfg->audio_object_type == 0 || cfg->audio_object_type > 31) return -1;
    if (cfg->sampling_freq_index > 12) return -1;
    if (cfg->channel_config == 0 || cfg->channel_config > 7) return -1;

    /* 5bit OT | 4bit SFI | 4bit CH | 3bit padding */
    out[0]  = (uint8_t)((cfg->audio_object_type & 0x1F) << 3);
    out[0] |= (uint8_t)((cfg->sampling_freq_index >> 1) & 0x07);
    out[1]  = (uint8_t)((cfg->sampling_freq_index & 0x01) << 7);
    out[1] |= (uint8_t)((cfg->channel_config & 0x0F) << 3);
    return 0;
}

/* ────────────────────────────────────────
 * AAC Sequence Header（使用原始 ASC 字节）
 * ──────────────────────────────────────── */

int flv_write_aac_seq_header_raw(FILE *fp, const uint8_t *asc, uint32_t asc_len) {
    uint32_t buf_len = 2 + asc_len;
    uint8_t *buf = (uint8_t *)malloc(buf_len);
    if (!buf) return -1;

    buf[0] = 0xAF;              /* SoundFormat=10|Rate=3|Size=1|Type=1 */
    buf[1] = FLV_AAC_SEQ_HEADER;
    memcpy(buf + 2, asc, asc_len);

    int ret = write_tag(fp, FLV_TAG_AUDIO, 0, buf, buf_len);
    free(buf);
    return ret;
}

int flv_write_aac_seq_header(FILE *fp, const FlvAACConfig *cfg) {
    uint8_t asc[2];
    if (flv_make_asc(cfg, asc) < 0) return -1;
    return flv_write_aac_seq_header_raw(fp, asc, 2);
}

/* ────────────────────────────────────────
 * AAC 音频帧（裸帧，不含 ADTS）
 * ──────────────────────────────────────── */

int flv_write_aac_frame(FILE *fp, uint32_t ts_ms,
                         const uint8_t *aac_data, uint32_t aac_len) {
    uint32_t buf_len = 2 + aac_len;
    uint8_t *buf = (uint8_t *)malloc(buf_len);
    if (!buf) return -1;

    buf[0] = 0xAF;
    buf[1] = FLV_AAC_RAW;
    memcpy(buf + 2, aac_data, aac_len);

    int ret = write_tag(fp, FLV_TAG_AUDIO, ts_ms, buf, buf_len);
    free(buf);
    return ret;
}

/* ────────────────────────────────────────
 * Annex-B → AVCC 转换
 * ──────────────────────────────────────── */

uint8_t *flv_annexb_to_avcc(const uint8_t *annexb, uint32_t annexb_len,
                              uint32_t *out_len) {
    /* 最坏情况：每个字节都被当成独立 NALU，输出不超过 annexb_len + 4*NALU数量 */
    uint8_t *out = (uint8_t *)malloc(annexb_len + 64);
    if (!out) { *out_len = 0; return NULL; }

    uint32_t out_off = 0;
    uint32_t i = 0;

    while (i < annexb_len) {
        /* 检测起始码 00 00 00 01（4字节）或 00 00 01（3字节） */
        int sc = 0;
        if (i + 3 < annexb_len &&
            annexb[i]==0 && annexb[i+1]==0 && annexb[i+2]==0 && annexb[i+3]==1)
            sc = 4;
        else if (i + 2 < annexb_len &&
                 annexb[i]==0 && annexb[i+1]==0 && annexb[i+2]==1)
            sc = 3;

        if (!sc) { i++; continue; }

        uint32_t nalu_start = i + sc;
        uint32_t j = nalu_start;

        /* 找下一个起始码 */
        while (j < annexb_len) {
            if (j + 3 < annexb_len &&
                annexb[j]==0 && annexb[j+1]==0 && annexb[j+2]==0 && annexb[j+3]==1)
                break;
            if (j + 2 < annexb_len &&
                annexb[j]==0 && annexb[j+1]==0 && annexb[j+2]==1)
                break;
            j++;
        }

        uint32_t nalu_len = j - nalu_start;
        if (nalu_len == 0) { i = j; continue; }

        /* 写 4 字节大端长度 + NALU 数据 */
        out[out_off++] = (uint8_t)(nalu_len >> 24);
        out[out_off++] = (uint8_t)(nalu_len >> 16);
        out[out_off++] = (uint8_t)(nalu_len >> 8);
        out[out_off++] = (uint8_t)(nalu_len);
        memcpy(out + out_off, annexb + nalu_start, nalu_len);
        out_off += nalu_len;
        i = j;
    }

    *out_len = out_off;
    return out;
}

/* ────────────────────────────────────────
 * 去除 ADTS 头
 * ──────────────────────────────────────── */

const uint8_t *flv_strip_adts(const uint8_t *adts, uint32_t adts_len,
                                uint32_t *raw_len) {
    if (!adts || adts_len < 7) return NULL;
    /* ADTS syncword: 0xFFF? (12bit) */
    if (adts[0] != 0xFF || (adts[1] & 0xF0) != 0xF0) return NULL;
    /* protection_absent: adts[1] bit0；1=无 CRC(7字节头)，0=有 CRC(9字节头) */
    uint32_t hdr_len = (adts[1] & 0x01) ? 7 : 9;
    if (adts_len <= hdr_len) return NULL;
    *raw_len = adts_len - hdr_len;
    return adts + hdr_len;
}
```

### 13.3 使用示例（完整直播流）

```c
#include "flv_muxer.h"
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    FILE *fp = fopen("live.flv", "wb");
    if (!fp) return 1;

    /* 1. 写 FLV Header（含音频和视频） */
    flv_write_header(fp, 1, 1);

    /* 2. 写 H.264 Sequence Header */
    /* sps/pps 来自编码器（不含起始码，如从 AVCodecContext->extradata 中提取） */
    static const uint8_t sps[] = { 0x67, 0x42, 0xC0, 0x1E, /* ... */ };
    static const uint8_t pps[] = { 0x68, 0xCE, 0x38, 0x80 };
    FlvAVCConfig avc_cfg = {
        .profile = sps[1], .compat = sps[2], .level = sps[3],
        .sps = sps, .sps_len = sizeof(sps),
        .pps = pps, .pps_len = sizeof(pps)
    };
    flv_write_avc_seq_header(fp, &avc_cfg);

    /* 3. 写 AAC Sequence Header（44100Hz, Stereo, AAC-LC） */
    FlvAACConfig aac_cfg = { .audio_object_type = 2, .sampling_freq_index = 4, .channel_config = 2 };
    flv_write_aac_seq_header(fp, &aac_cfg);

    /* 4. 循环写入媒体帧 */
    uint32_t ts_ms = 0;
    while (/* 有数据 */ 0) {
        /* 视频帧（假设编码器输出 Annex-B） */
        uint8_t *annexb_frame = NULL; /* 从编码器获取 */
        uint32_t annexb_len = 0;
        uint32_t avcc_len = 0;
        int is_key = 0; /* 从编码器获取 */
        uint8_t *avcc = flv_annexb_to_avcc(annexb_frame, annexb_len, &avcc_len);
        if (avcc) {
            flv_write_avc_frame(fp, ts_ms, is_key, 0, avcc, avcc_len);
            free(avcc);
        }

        /* 音频帧（假设编码器输出 ADTS） */
        uint8_t *adts_frame = NULL; /* 从编码器获取 */
        uint32_t adts_len = 0;
        uint32_t raw_len = 0;
        const uint8_t *raw = flv_strip_adts(adts_frame, adts_len, &raw_len);
        if (raw) {
            flv_write_aac_frame(fp, ts_ms, raw, raw_len);
        }

        ts_ms += 40; /* 25fps 时每帧 40ms */
    }

    fclose(fp);
    return 0;
}
```

---

## 14. 测试向量（十六进制）

### 14.1 最小合法 FLV Header（仅视频）

```
46 4C 56 01 01 00 00 00 09 00 00 00 00
│           │  │  └──────────── DataOffset=9
│           │  └─────────────── Flags: HasVideo
│           └────────────────── Version=1
└────────────────────────────── "FLV"
最后 4 字节: PreviousTagSize0=0
```

### 14.2 AAC Sequence Header Tag（44100Hz, Stereo）

```
Tag Header (11字节):
  08               TagType=8 (Audio)
  00 00 04         DataSize=4
  00 00 00         Timestamp=0
  00               TimestampExtended=0
  00 00 00         StreamID=0
Tag Data (4字节):
  AF               SoundSpec (Format=10|Rate=3|Size=1|Type=1)
  00               AACPacketType=0 (AudioSpecificConfig)
  12 10            AudioSpecificConfig: OT=2,SFI=4,CH=2 (AAC-LC,44100,Stereo)
PreviousTagSize (4字节):
  00 00 00 0F      = 11 + 4 = 15
```

### 14.3 H.264 Sequence Header Tag 片段

```
Tag Header:
  09               TagType=9 (Video)
  00 00 XX         DataSize (取决于 SPS/PPS 长度)
  00 00 00         Timestamp=0
  00               TimestampExtended=0
  00 00 00         StreamID=0
Tag Data:
  17               FrameInfo: FrameType=1(Key)|CodecID=7(AVC)
  00               AVCPacketType=0 (SequenceHeader)
  00 00 00         CompositionTime=0
  01               configurationVersion=1
  XX               avcProfileIndication (SPS[1])
  XX               profile_compatibility (SPS[2])
  XX               AVCLevelIndication (SPS[3])
  FF               lengthSizeMinusOne=3 (4字节前缀)
  E1               0xE0|numSPS=1
  00 XX            spsLength
  [SPS NALU bytes without start code]
  01               numPPS=1
  00 XX            ppsLength
  [PPS NALU bytes without start code]
```

---

## 15. Agent 代码审查 Checklist

以下规则供 AI Agent 在审查 C/C++ HTTP-FLV 封装代码时逐条核对。

### 15.1 FLV 格式合规性

- [ ] **FLV Header** Signature 是否为 `0x46 0x4C 0x56`（"FLV"）
- [ ] **FLV Header** Version 是否为 `0x01`
- [ ] **FLV Header** DataOffset 是否为 `0x00000009`（9）
- [ ] **FLV Header** TypeFlags 中保留 bit 是否为 0
- [ ] **PreviousTagSize0** 是否紧跟在 Header 后，值是否为 `0x00000000`
- [ ] 每个 **Tag** 的 `PreviousTagSize` 是否等于 `11 + DataSize`
- [ ] 每个 **Tag** 的 `StreamID` 是否为 `0x000000`
- [ ] 所有多字节字段是否使用**大端序**

### 15.2 时间戳

- [ ] 时间戳单位是否为**毫秒**
- [ ] 直播流时间戳是否从 0 开始单调递增
- [ ] `TimestampExtended`（高 8 位）是否正确分离并写入偏移 7 处
- [ ] 计算 `PTS = DTS + CTS` 时，`CTS` 是否为 24 位**有符号整数**

### 15.3 Video Tag

- [ ] 关键帧（IDR/IRAP）的 `FrameType` 是否为 `1`，非关键帧是否为 `2`
- [ ] NALU 是否使用 **AVCC 格式**（4 字节大端长度前缀），而非 Annex-B
- [ ] NALU 数据中是否**不含**起始码 `00 00 00 01`
- [ ] `naluLength` 是否等于后续 NALU 数据字节数（**不含**自身 4 字节）
- [ ] `AVCDecoderConfigurationRecord` 中 `configurationVersion` 是否为 `1`
- [ ] `AVCDecoderConfigurationRecord` 中 `avcProfileIndication` 是否非 0
- [ ] `AVCDecoderConfigurationRecord` 中 `lengthSizeMinusOne` 是否为 `3`（4 字节前缀）
- [ ] `AVCDecoderConfigurationRecord` 中 `numSPS` 和 `numPPS` 是否均 ≥ 1
- [ ] SPS/PPS 是否**不含**起始码
- [ ] `HEVCDecoderConfigurationRecord` 总长度是否 ≥ 23 字节
- [ ] 直播流每个 IDR GOP 开头是否重发 Video Sequence Header
- [ ] Video Sequence Header 时间戳是否为 0（或等于首帧时间戳）

### 15.4 Audio Tag

- [ ] AAC `SoundSpec` 字节高 4 位是否为 `10`（0xA_）
- [ ] AAC Sequence Header 的 `AACPacketType` 是否为 `0`
- [ ] AAC Raw Frame 的 `AACPacketType` 是否为 `1`
- [ ] AAC 数据是否**不含 ADTS 头**（首字节 `0xFF` 是 ADTS 特征，不应出现）
- [ ] `AudioSpecificConfig` 的 `samplingFrequencyIndex` 是否在 0~12 范围内
- [ ] `AudioSpecificConfig` 的 `channelConfig` 是否在 1~7 范围内

### 15.5 流时序

- [ ] Video Sequence Header 是否在第一个视频媒体 Tag 之前发送
- [ ] Audio Sequence Header 是否在第一个音频媒体 Tag 之前发送
- [ ] 若包含 ScriptData（onMetaData），是否在 Sequence Header 之前发送

### 15.6 HTTP 服务器

- [ ] `Content-Type` 是否为 `video/x-flv`
- [ ] 是否使用 `Transfer-Encoding: chunked`（**不设**`Content-Length`）
- [ ] 跨域场景是否设置 `Access-Control-Allow-Origin`
- [ ] 连接是否维持长连接（`Connection: keep-alive`）

### 15.7 内存与安全

- [ ] 所有 `malloc` 返回值是否检查 NULL
- [ ] 写入 `buf` 前是否检查缓冲区大小（防越界）
- [ ] `flv_annexb_to_avcc` 返回值是否在使用后 `free()`
- [ ] `naluLen` 与实际数据大小的一致性是否校验（防止 buffer overread）

---

## 参考资料

| 资料 | 说明 |
|------|------|
| `src/demux/flv-demuxer.js` | 本文档的直接来源，权威参考 |
| Adobe FLV/F4V Spec v10.1 | FLV 容器格式官方规范 |
| ISO 14496-15:2014 | AVC/HEVC in MP4/FLV 封装规范 |
| ISO 14496-3 | AAC AudioSpecificConfig 编码规范 |
| [Enhanced RTMP Spec](https://github.com/veovera/enhanced-rtmp) | H.265/AV1/Opus over FLV 规范 |
| [SRS (Simple Realtime Server)](https://github.com/ossrs/srs/) | 开源 C++ 媒体服务器参考实现 |
