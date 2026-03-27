# HTTP-FLV 直播流格式规范（C/C++ 封装库开发指南）

本文档面向需要开发 C/C++ 后端推流服务的开发者，详细描述了 mpegts.js 所支持的 HTTP-FLV 流的完整格式规范，包括 H.264、H.265 视频封装和 AAC 音频封装方式。

---

## 一、mpegts.js 支持的流类型

mpegts.js 支持以下几种输入流类型：

| `type` 值       | 协议 / 格式            | 说明                         |
|-----------------|------------------------|------------------------------|
| `flv`           | HTTP-FLV               | 标准 FLV 容器通过 HTTP 流式传输 |
| `mpegts` / `m2ts` | MPEG2-TS over HTTP   | TS 流通过 HTTP 传输           |
| `mse`           | WebSocket              | FLV 或 TS 通过 WebSocket 传输 |

本文重点描述 **HTTP-FLV** 格式，这也是直播场景中最常用的格式。

---

## 二、FLV 文件 / 流整体结构

一个完整的 FLV 直播流结构如下：

```
[FLV Header]          9 字节
[PreviousTagSize0]     4 字节 (固定为 0x00000000)
[Tag 1: ScriptData]   onMetaData (可选，但推荐)
[Tag 2: Video]        AVCDecoderConfigurationRecord / HEVCDecoderConfigurationRecord (Sequence Header)
[Tag 3: Audio]        AudioSpecificConfig (Sequence Header)
[Tag 4: Video]        关键帧 IDR
[Tag 5: Audio]        AAC 裸帧
[Tag 6: Video]        非关键帧
...                   循环
```

---

## 三、FLV Header（9 字节）

| 偏移 | 大小 | 描述 |
|------|------|------|
| 0    | 3    | Magic：`0x46 0x4C 0x56`（ASCII "FLV"） |
| 3    | 1    | Version：`0x01` |
| 4    | 1    | Flags：`bit2` = HasAudio，`bit0` = HasVideo |
| 5    | 4    | DataOffset：大端 uint32，固定为 `0x00000009` |

示例（仅视频）：

```
46 4C 56 01 01 00 00 00 09
```

示例（音视频都有）：

```
46 4C 56 01 05 00 00 00 09
```

---

## 四、FLV Tag 通用结构

每个 Tag 的格式如下（共 11 字节头 + 数据 + 4 字节尾）：

| 偏移    | 大小 | 描述 |
|---------|------|------|
| 0       | 1    | TagType：`8`=Audio，`9`=Video，`18`=ScriptData |
| 1       | 3    | DataSize：大端 uint24，Tag 数据体字节数 |
| 4       | 3    | Timestamp（低 24 位）：大端 uint24，单位毫秒 |
| 7       | 1    | TimestampExtended（高 8 位）：uint8 |
| 8       | 3    | StreamID：大端 uint24，始终为 `0x000000` |
| 11      | N    | Data：具体负载 |
| 11 + N  | 4    | PreviousTagSize：大端 uint32 = `11 + DataSize` |

> **时间戳说明：** 实际时间戳（ms）= `(TimestampExtended << 24) | Timestamp低24位`，为 32 位有符号整数，直播流从 0 开始单调递增。

---

## 五、ScriptData Tag（TagType = 18）—— onMetaData

用 AMF0 编码，在流开头发送，包含媒体元信息：

```
[AMF0 String]  "onMetaData"
[AMF0 Object]
  "width"           => Number  (视频宽度，px)
  "height"          => Number  (视频高度，px)
  "framerate"       => Number  (帧率，如 25.0)
  "videodatarate"   => Number  (视频码率，Kbps)
  "audiodatarate"   => Number  (音频码率，Kbps)
  "hasAudio"        => Boolean
  "hasVideo"        => Boolean
  "duration"        => Number  (直播流填 0 或省略)
```

mpegts.js 会解析 `width`、`height`、`framerate`、`duration`、`hasAudio`、`hasVideo` 等字段。

---

## 六、Video Tag（TagType = 9）

### 6.1 H.264 (AVC) 视频 Tag

#### 6.1.1 Tag 数据结构

```
1 字节  FrameInfo:
          bit7-4: FrameType
            1 = 关键帧 (keyframe / IDR)
            2 = 非关键帧 (inter frame)
          bit3-0: CodecID = 7  (AVC / H.264)

1 字节  AVCPacketType:
          0 = AVCDecoderConfigurationRecord (Sequence Header)
          1 = AVC NALU (普通视频帧)
          2 = AVC end of sequence

3 字节  CompositionTime (CTS)：24 位有符号整数，单位毫秒
          PTS = DTS(TagTimestamp) + CTS
          纯 I/P 帧时 CTS = 0，B 帧时 CTS > 0

N 字节  VideoData（见下文）
```

#### 6.1.2 AVCDecoderConfigurationRecord（ISO 14496-15）

在 `AVCPacketType = 0` 时发送，格式如下：

```
1 字节  configurationVersion = 1
1 字节  avcProfileIndication  (来自 SPS[1])
1 字节  profile_compatibility (来自 SPS[2])
1 字节  AVCLevelIndication    (来自 SPS[3])
1 字节  0xFC | (lengthSizeMinusOne & 0x03)  通常为 0xFF (4 字节长度前缀)
1 字节  0xE0 | numSPS                        通常为 0xE1 (1 个 SPS)
2 字节  spsLength (大端)
N 字节  SPS NALU 数据（不含起始码 00 00 00 01）
1 字节  numPPS                               通常为 0x01
2 字节  ppsLength (大端)
M 字节  PPS NALU 数据（不含起始码）
```

#### 6.1.3 AVC NALU 数据帧（AVCPacketType = 1）

使用 **AVCC 格式**（4 字节大端长度前缀），**不是** Annex-B（起始码 `00 00 00 01`）：

```
4 字节  naluLength (大端 uint32)
N 字节  NALU 数据（不含起始码）

// 多个 NALU 连续拼接（如同一帧包含 SEI + IDR）
4 字节  naluLength2
M 字节  NALU2 数据
...
```

> **关键区别：** FLV 中的 H.264 NALU 使用 AVCC 格式，**不能**直接放 Annex-B 数据。x264 等编码器输出的 Annex-B 需要先转换（去除起始码，加 4 字节长度前缀）。

---

### 6.2 H.265 (HEVC) 视频 Tag — 旧式写法（CodecID = 12）

```
1 字节  FrameInfo:
          bit7-4: FrameType（同上）
          bit3-0: CodecID = 12  (HEVC / H.265)

1 字节  HEVCPacketType:
          0 = HEVCDecoderConfigurationRecord
          1 = HEVC NALU
          2 = HEVC end of sequence

3 字节  CompositionTime（同 H.264）

N 字节  VideoData
```

**HEVCDecoderConfigurationRecord（ISO 14496-15:2014）格式：**

```
1 字节  configurationVersion = 1
1 字节  general_profile_space(2) | general_tier_flag(1) | general_profile_idc(5)
4 字节  general_profile_compatibility_flags (大端)
6 字节  general_constraint_indicator_flags
1 字节  general_level_idc
2 字节  0xF000 | min_spatial_segmentation_idc (大端)
1 字节  0xFC | parallelismType
1 字节  0xFC | chromaFormat
1 字节  0xF8 | bitDepthLumaMinus8
1 字节  0xF8 | bitDepthChromaMinus8
2 字节  avgFrameRate (大端)
1 字节  constantFrameRate(2) | numTemporalLayers(3) | temporalIdNested(1) | lengthSizeMinusOne(2)
1 字节  numOfArrays
foreach array:
  1 字节  array_completeness(1) | reserved(1) | NAL_unit_type(6)
  2 字节  numNalus (大端)
  foreach nalu:
    2 字节  naluLength (大端)
    N 字节  NALU 数据
```

---

### 6.3 H.265 (HEVC) 视频 Tag — Enhanced RTMP 写法（推荐，mpegts.js v1.7.3+）

遵循 [Enhanced RTMP](https://github.com/veovera/enhanced-rtmp) 规范，首字节最高位置 1：

```
1 字节  ExFrameInfo:
          bit7:   IsExHeader = 1（标志这是 Enhanced FLV 包）
          bit6-4: FrameType（同上）
          bit3-0: PacketType:
            0 = SequenceStart (decoder config)
            1 = CodedFrames   (含 CTS)
            2 = SequenceEnd
            3 = CodedFramesX  (不含 CTS，隐含 CTS=0)

4 字节  FourCC:
          "hvc1" = HEVC / H.265
          "av01" = AV1

// PacketType = 0:
  N 字节  HEVCDecoderConfigurationRecord（同上）

// PacketType = 1:
  3 字节  CompositionTime（24 位有符号，毫秒）
  N 字节  HEVC NALU 数据（AVCC 格式，4 字节长度前缀）

// PacketType = 3:
  N 字节  HEVC NALU 数据（CTS 隐含为 0）
```

---

## 七、Audio Tag（TagType = 8）

### 7.1 AAC 音频 Tag（最常用）

#### 7.1.1 Tag 数据结构

```
1 字节  SoundSpec:
          bit7-4: SoundFormat = 10  (AAC)
          bit3-2: SoundRate         (建议 3 = 44100 Hz)
          bit1:   SoundSize = 1     (16-bit)
          bit0:   SoundType = 1     (stereo)
          → 通常固定为 0xAF

1 字节  AACPacketType:
          0 = AudioSpecificConfig（Sequence Header）
          1 = AAC Raw Frame Data

N 字节  AudioData
```

#### 7.1.2 AudioSpecificConfig（ISO 14496-3）

在 `AACPacketType = 0` 时发送，最简格式 2 字节：

```
bits  描述
----  --------
5     audioObjectType:  2 = AAC-LC，5 = HE-AAC SBR
4     samplingFrequencyIndex:
        0=96000, 1=88200, 2=64000, 3=48000, 4=44100,
        5=32000, 6=22050, 7=16000, 8=12000, 9=11025, 10=8000
4     channelConfiguration:  1=mono，2=stereo
```

常用示例：

| 配置                 | 2 字节值  |
|----------------------|-----------|
| AAC-LC，44100Hz，立体声 | `0x12 0x10` |
| AAC-LC，48000Hz，立体声 | `0x11 0x90` |
| AAC-LC，32000Hz，立体声 | `0x14 0x10` |
| AAC-LC，22050Hz，立体声 | `0x16 0x10` |

#### 7.1.3 AAC Raw Frame Data（AACPacketType = 1）

直接放**裸 AAC 帧数据**，**不含 ADTS 头**（即去掉 `0xFF 0xF1 ...` 开头的 7 字节 ADTS 头）。

---

### 7.2 Enhanced FLV 音频（Opus / FLAC，mpegts.js v1.8.0+）

```
1 字节  SoundSpec:
          bit7-4: SoundFormat = 9  (Enhanced FLV 音频标志)
          bit3-0: PacketType:
            0 = SequenceStart
            1 = CodedFrames
            2 = SequenceEnd

4 字节  FourCC:
          "Opus" = Opus 音频
          "fLaC" = FLAC 音频

N 字节  音频数据
```

---

## 八、HTTP 响应头要求

后端 HTTP 服务器的响应头必须满足以下要求：

```http
HTTP/1.1 200 OK
Content-Type: video/x-flv
Transfer-Encoding: chunked
Connection: keep-alive
Cache-Control: no-cache
Access-Control-Allow-Origin: *
Access-Control-Allow-Headers: *
Access-Control-Expose-Headers: *
```

> - **不要设置 `Content-Length`**（直播流长度未知）
> - 使用 **`Transfer-Encoding: chunked`** 持续写入数据
> - `Access-Control-Allow-Origin: *` 是跨域播放的必要条件

---

## 九、直播流 Tag 时序规则

1. **Sequence Header 必须在任何媒体帧之前发送。**
2. **直播中每个 GOP（IDR 帧开头）应重发 Sequence Header**，以便后接入的客户端能初始化解码器。
3. **关键帧（IDR）的 FrameType 必须为 1**；其他帧 FrameType = 2。
4. **时间戳从 0 开始**，以毫秒为单位单调递增。
5. **音视频时间戳需对齐**，不应有大的偏差。
6. AAC 数据发送**裸 AAC 帧**（无 ADTS 头），发前先发 AudioSpecificConfig。

---

## 十、C/C++ 参考实现

以下伪代码展示了各类 FLV Tag 的写入方法，适用于 C/C++ 封装库开发。

### 10.1 辅助写入函数

```c
#include <stdint.h>
#include <string.h>
#include <stdio.h>

static void write_uint8(FILE* fp, uint8_t v) {
    fwrite(&v, 1, 1, fp);
}

static void write_uint16_be(FILE* fp, uint16_t v) {
    uint8_t buf[2] = { (uint8_t)(v >> 8), (uint8_t)(v & 0xFF) };
    fwrite(buf, 1, 2, fp);
}

static void write_uint24_be(FILE* fp, uint32_t v) {
    uint8_t buf[3] = { (uint8_t)(v >> 16), (uint8_t)(v >> 8), (uint8_t)(v & 0xFF) };
    fwrite(buf, 1, 3, fp);
}

static void write_uint32_be(FILE* fp, uint32_t v) {
    uint8_t buf[4] = {
        (uint8_t)(v >> 24), (uint8_t)(v >> 16),
        (uint8_t)(v >> 8),  (uint8_t)(v & 0xFF)
    };
    fwrite(buf, 1, 4, fp);
}
```

### 10.2 写 FLV Header

```c
void write_flv_header(FILE* fp, int has_audio, int has_video) {
    uint8_t flags = 0;
    if (has_audio) flags |= 0x04;
    if (has_video) flags |= 0x01;

    fwrite("FLV", 1, 3, fp);         // Magic
    write_uint8(fp, 0x01);            // Version
    write_uint8(fp, flags);           // Flags
    write_uint32_be(fp, 9);           // DataOffset = 9
    write_uint32_be(fp, 0);           // PreviousTagSize0 = 0
}
```

### 10.3 写通用 FLV Tag

```c
void write_flv_tag(FILE* fp, uint8_t tag_type, uint32_t timestamp_ms,
                   const uint8_t* data, uint32_t data_size) {
    write_uint8(fp, tag_type);
    write_uint24_be(fp, data_size);
    write_uint24_be(fp, timestamp_ms & 0x00FFFFFF);   // 低 24 位
    write_uint8(fp, (timestamp_ms >> 24) & 0xFF);      // 高 8 位（扩展）
    write_uint24_be(fp, 0);                            // StreamID = 0
    fwrite(data, 1, data_size, fp);
    write_uint32_be(fp, 11 + data_size);               // PreviousTagSize
}
```

### 10.4 写 H.264 Sequence Header

```c
// sps / pps: 不含起始码的裸 NALU 数据
void write_avc_sequence_header(FILE* fp,
                                const uint8_t* sps, uint32_t sps_len,
                                const uint8_t* pps, uint32_t pps_len) {
    uint8_t buf[4096];
    uint32_t offset = 0;

    buf[offset++] = 0x17;  // FrameType=1(keyframe) | CodecID=7(AVC)
    buf[offset++] = 0x00;  // AVCPacketType=0 (Sequence Header)
    buf[offset++] = 0x00;  // CompositionTime = 0
    buf[offset++] = 0x00;
    buf[offset++] = 0x00;

    // AVCDecoderConfigurationRecord
    buf[offset++] = 0x01;           // configurationVersion
    buf[offset++] = sps[1];         // avcProfileIndication
    buf[offset++] = sps[2];         // profile_compatibility
    buf[offset++] = sps[3];         // AVCLevelIndication
    buf[offset++] = 0xFF;           // lengthSizeMinusOne = 3 (4 字节前缀)
    buf[offset++] = 0xE1;           // numSPS = 1
    buf[offset++] = (uint8_t)(sps_len >> 8);
    buf[offset++] = (uint8_t)(sps_len & 0xFF);
    memcpy(buf + offset, sps, sps_len); offset += sps_len;
    buf[offset++] = 0x01;           // numPPS = 1
    buf[offset++] = (uint8_t)(pps_len >> 8);
    buf[offset++] = (uint8_t)(pps_len & 0xFF);
    memcpy(buf + offset, pps, pps_len); offset += pps_len;

    write_flv_tag(fp, 9, 0, buf, offset);
}
```

### 10.5 写 H.264 视频帧

```c
// nalu_data: 一个或多个 NALU（已为 AVCC 格式，含 4 字节长度前缀）
// 若输入为 Annex-B，需先调用 annexb_to_avcc() 转换
void write_avc_video_frame(FILE* fp, uint32_t timestamp_ms, int is_keyframe,
                            int32_t cts_ms,
                            const uint8_t* nalu_data, uint32_t nalu_size) {
    uint8_t* buf = (uint8_t*)malloc(nalu_size + 5);
    uint32_t offset = 0;

    buf[offset++] = is_keyframe ? 0x17 : 0x27;  // FrameType | CodecID=7
    buf[offset++] = 0x01;                         // AVCPacketType=1 (NALU)
    // CompositionTime（24 位有符号）
    buf[offset++] = (uint8_t)((cts_ms >> 16) & 0xFF);
    buf[offset++] = (uint8_t)((cts_ms >> 8)  & 0xFF);
    buf[offset++] = (uint8_t)(cts_ms         & 0xFF);
    memcpy(buf + offset, nalu_data, nalu_size); offset += nalu_size;

    write_flv_tag(fp, 9, timestamp_ms, buf, offset);
    free(buf);
}
```

### 10.6 Annex-B 转 AVCC

```c
// 将 x264/x265 输出的 Annex-B 格式转换为 AVCC（4 字节大端长度前缀）
// 返回值需 free()
uint8_t* annexb_to_avcc(const uint8_t* in, uint32_t in_size,
                         uint32_t* out_size) {
    uint8_t* out = (uint8_t*)malloc(in_size + 64);
    uint32_t out_off = 0;
    uint32_t i = 0;

    while (i < in_size) {
        // 检测起始码 00 00 01 或 00 00 00 01
        int sc_len = 0;
        if (i + 3 < in_size && in[i]==0 && in[i+1]==0 && in[i+2]==0 && in[i+3]==1)
            sc_len = 4;
        else if (i + 2 < in_size && in[i]==0 && in[i+1]==0 && in[i+2]==1)
            sc_len = 3;

        if (sc_len == 0) { i++; continue; }

        uint32_t nalu_start = i + sc_len;
        // 找下一个起始码
        uint32_t j = nalu_start;
        while (j < in_size) {
            if (j + 3 < in_size && in[j]==0 && in[j+1]==0 && in[j+2]==0 && in[j+3]==1) break;
            if (j + 2 < in_size && in[j]==0 && in[j+1]==0 && in[j+2]==1) break;
            j++;
        }
        uint32_t nalu_len = j - nalu_start;
        if (nalu_len == 0) { i = j; continue; }

        // 写入 4 字节大端长度 + NALU 数据
        out[out_off++] = (uint8_t)(nalu_len >> 24);
        out[out_off++] = (uint8_t)(nalu_len >> 16);
        out[out_off++] = (uint8_t)(nalu_len >> 8);
        out[out_off++] = (uint8_t)(nalu_len);
        memcpy(out + out_off, in + nalu_start, nalu_len);
        out_off += nalu_len;
        i = j;
    }

    *out_size = out_off;
    return out;
}
```

### 10.7 写 AAC Sequence Header

```c
// asc: AudioSpecificConfig 字节数组（通常 2 字节）
void write_aac_sequence_header(FILE* fp, const uint8_t* asc, uint32_t asc_len) {
    uint8_t buf[asc_len + 2];
    buf[0] = 0xAF;  // SoundFormat=10(AAC) | SoundRate=3 | SoundSize=1 | SoundType=1
    buf[1] = 0x00;  // AACPacketType=0 (AudioSpecificConfig)
    memcpy(buf + 2, asc, asc_len);
    write_flv_tag(fp, 8, 0, buf, (uint32_t)(asc_len + 2));
}
```

### 10.8 写 AAC 音频帧

```c
// aac_data: 裸 AAC 帧（不含 ADTS 头）
void write_aac_audio_frame(FILE* fp, uint32_t timestamp_ms,
                            const uint8_t* aac_data, uint32_t aac_size) {
    uint8_t* buf = (uint8_t*)malloc(aac_size + 2);
    buf[0] = 0xAF;  // SoundFormat=10(AAC)
    buf[1] = 0x01;  // AACPacketType=1 (Raw)
    memcpy(buf + 2, aac_data, aac_size);
    write_flv_tag(fp, 8, timestamp_ms, buf, aac_size + 2);
    free(buf);
}
```

### 10.9 去除 ADTS 头（FFmpeg/编码器输出处理）

```c
// ADTS 头长度：固定部分 7 字节；若有 CRC 则 9 字节
// ADTS Header: bit 28 (protection_absent) = 1 → 无 CRC，header = 7 字节
//                                           = 0 → 有 CRC，header = 9 字节
const uint8_t* strip_adts_header(const uint8_t* adts_data, uint32_t adts_size,
                                   uint32_t* raw_size) {
    if (adts_size < 7) return NULL;
    if (adts_data[0] != 0xFF || (adts_data[1] & 0xF0) != 0xF0) return NULL;
    int header_len = (adts_data[1] & 0x01) ? 7 : 9;  // protection_absent
    *raw_size = adts_size - header_len;
    return adts_data + header_len;
}
```

---

## 十一、关键注意事项

| 事项 | 要求 |
|------|------|
| 字节序 | FLV 所有多字节字段使用**大端序** |
| NALU 格式 | FLV 中使用 **AVCC**（4 字节长度前缀），**不能**用 Annex-B |
| Sequence Header | 必须在任何媒体帧之前发送；直播中每个 GOP 开头重发 |
| 时间戳 | 毫秒单位，32 位有符号整数，直播从 0 单调递增 |
| HTTP 响应头 | 必须有 `Transfer-Encoding: chunked` + `Access-Control-Allow-Origin: *` |
| H.265 编码标识 | CodecID=12（旧式）或 Enhanced FLV FourCC=`hvc1`（推荐） |
| AAC 数据 | 发送裸 AAC 帧（无 ADTS 头），流开头先发 AudioSpecificConfig |
| StreamID | 永远为 0 |
| PreviousTagSize | = `11 + DataSize` |

---

## 十二、常用 AudioSpecificConfig 速查表

| 采样率   | 声道 | HE-AAC (2 字节) | AAC-LC (2 字节) |
|----------|------|-----------------|-----------------|
| 44100 Hz | 立体声 | `0x12 0x10` | `0x12 0x10` |
| 48000 Hz | 立体声 | `0x11 0x90` | `0x11 0x90` |
| 32000 Hz | 立体声 | `0x14 0x10` | `0x14 0x10` |
| 22050 Hz | 立体声 | `0x16 0x10` | `0x16 0x10` |
| 44100 Hz | 单声道 | `0x12 0x08` | `0x12 0x08` |

---

## 十三、推荐参考资源

- **SRS (Simple Realtime Server)**：开源 C++ 媒体服务器，mpegts.js 官方推荐用于测试 HTTP-FLV 直播
  https://github.com/ossrs/srs/
- **nginx-rtmp-module**：可通过 RTMP 接收推流，转为 HTTP-FLV 输出
- **Adobe FLV/F4V 规范**：`video_file_format_spec_v10_1.pdf`（可在 Adobe 官网或开源镜像获取）
- **Enhanced RTMP 规范**（H.265 / AV1 over FLV）：https://github.com/veovera/enhanced-rtmp
- **ISO 14496-15**：AVC/HEVC 在 MP4/FLV 中的封装规范（AVCDecoderConfigurationRecord / HEVCDecoderConfigurationRecord）
- **ISO 14496-3**：AAC AudioSpecificConfig 编码规范

---

*文档生成自 mpegts.js 源码分析（`src/demux/flv-demuxer.js`），版本参考 v1.8.0+*
