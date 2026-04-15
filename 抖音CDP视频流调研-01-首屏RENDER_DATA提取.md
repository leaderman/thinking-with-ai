# 抖音视频流获取方案调研

> 调研日期：2026-04-15  
> 目标视频：`https://www.douyin.com/jingxuan?modal_id=7597065321907817755`  
> Chrome CDP：`192.168.31.112:9223`（Chrome 146）

---

## 一、技术架构概述

抖音 PC Web 播放器使用 **MSE（Media Source Extensions）** 技术：

- `<video>` 标签的 `src` 是 `blob:https://www.douyin.com/xxx`（内存对象，无法直接使用）
- JS 播放器（xgplayer）通过 `fetch` 分片请求（HTTP Range）向 `douyinvod.com` 拉取视频数据
- 数据写入 `MediaSource` 对象后交由浏览器渲染

**因此不能直接从 `<video>` 元素获取下载地址，需要从其他入口获取原始 URL。**

---

## 二、获取地址的最佳入口：RENDER_DATA

页面 HTML 中内嵌了一个 `<script id="RENDER_DATA">` 元素，内容是 **URL 编码的 JSON 字符串**，包含当前视频的完整播放信息，包括所有画质的下载地址。

### 提取方式（CDP Runtime.evaluate）

```javascript
const raw = document.getElementById('RENDER_DATA').textContent;
const data = JSON.parse(decodeURIComponent(raw));
```

### 关键数据结构

```
data.app.videoDetail.video
├── width / height / duration          # 视频基本信息
├── playAddr[]                         # 默认画质播放地址（镜像列表）
├── playAddrH265[]                     # H.265 版本地址
├── bitRateList[]                      # 所有画质选项（完整列表）
│   ├── gearName                       # 画质名称（如 normal_1080_0）
│   ├── bitRate                        # 码率（bps）
│   ├── dataSize                       # 文件大小（bytes）
│   ├── isH265                         # 是否 H.265
│   ├── playAddr[].src                 # 下载 URL（可直接使用）
│   ├── audioFileId                    # 存在则为 DASH 分离流（独立音频）
│   ├── indexRange / initRange         # DASH 分片索引范围
│   └── audioChannels / audioSampleRate
└── bitRateAudioList[]                 # 独立音频流列表（配合 DASH 视频使用）
    ├── urlList[].src                  # 音频下载 URL（media-audio-und-mp4a）
    ├── bitrate / size
    └── codecType
```

---

## 三、视频流格式说明

抖音同时提供两种格式：

### 格式 A：合流 MP4（视频+音频打包）

- `bitRateList` 中**没有** `audioFileId` 字段的条目
- URL 路径形如：`/video/tos/cn/tos-cn-ve-15/xxxxx/?...&mime_type=video_mp4`
- 视频和音频合并在同一个 mp4 文件，**直接下载即可，无需 ffmpeg**
- 编码：H.264 或 H.265

### 格式 B：DASH 分离流（视频/音频独立）

- `bitRateList` 中**有** `audioFileId` 字段的条目
- 视频 URL 路径含 `media-video-avc1` 或 `media-video-hvc1`
- 音频 URL 路径含 `media-audio-und-mp4a`（在 `bitRateAudioList` 中）
- 需要 ffmpeg 合并：`ffmpeg -i video.mp4 -i audio.m4a -c copy output.mp4`

---

## 四、实测画质列表（视频 7597065321907817755）

| # | gearName | 码率 | 文件大小 | 编码 | 格式 |
|---|----------|------|---------|------|------|
| 0 | normal_1080_0 | 871k bps | **101.8 MB** | H.264 | 合流 |
| 1 | normal_1080_0 | 814k bps | — | H.264 | **DASH分离** |
| 2 | normal_540_0 | 674k bps | 78.8 MB | H.264 | 合流 |
| 3 | normal_720_0 | 667k bps | 78.0 MB | H.264 | 合流 |
| 4 | low_720_0 | 593k bps | 69.4 MB | H.264 | 合流 |
| 5 | low_540_0 | 539k bps | 63.1 MB | H.264 | 合流 |
| 6 | low_720_0 | 537k bps | — | H.264 | **DASH分离** |
| 7 | 1080_1_1 | 498k bps | 58.2 MB | **H.265** | 合流 |
| 8 | lower_540_0 | 444k bps | 52.0 MB | H.264 | 合流 |
| 9 | adapt_low_540_0 | 384k bps | 45.0 MB | H.264 | 合流 |
| 10 | low_540_0 | 483k bps | — | H.264 | **DASH分离** |
| 11 | 1080_1_1 | 442k bps | — | **H.265** | **DASH分离** |
| ... | 720_1_1 / 540_1_1 / 540_2_1 等 | ... | ... | H.265 | DASH分离 |

独立音频（配合 DASH 流）：
- AAC，44100 Hz，立体声，约 **22.7 MB**，190k bps

---

## 五、完整实现流程

```
1. CDP Page.navigate(url)
2. 等待 Page.loadEventFired
3. Runtime.evaluate → 提取 RENDER_DATA
4. decodeURIComponent + JSON.parse
5. 按需选择 bitRateList 中的画质
6. 取 playAddr[0].src 作为下载地址
7. curl / axios 下载（需带 Referer 和 User-Agent）
```

### 关键 JS 代码片段

```javascript
// 在 CDP Runtime.evaluate 中执行
(function() {
  const raw = document.getElementById('RENDER_DATA').textContent;
  const data = JSON.parse(decodeURIComponent(raw));
  const video = data.app.videoDetail.video;

  // 取最高质量合流版本（无 audioFileId）
  const best = video.bitRateList.find(item => !item.audioFileId);
  const videoUrl = best.playAddr[0].src;

  // 取独立音频（如需 DASH 方案）
  const audioUrl = video.bitRateAudioList[0].urlList[0].src;

  return JSON.stringify({ videoUrl, audioUrl });
})()
```

---

## 六、下载注意事项

### 必要的请求头

```
Referer: https://www.douyin.com/
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
```

不需要 Cookie。

### URL 有效期

URL 中含时间戳参数（`dy_q`），**有效期约 1~2 小时**，需在提取后尽快下载。

### curl 下载示例

```bash
# 方案 A：直接下载合流 mp4（推荐）
curl -L -g \
  -H "Referer: https://www.douyin.com/" \
  -H "User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36" \
  -o output.mp4 \
  "<playAddr[0].src>"

# 方案 B：DASH 分离流 + ffmpeg 合并
curl -L -H "Referer: https://www.douyin.com/" -o video_only.mp4 "<media-video-avc1 URL>"
curl -L -H "Referer: https://www.douyin.com/" -o audio_only.m4a "<media-audio-und-mp4a URL>"
ffmpeg -i video_only.mp4 -i audio_only.m4a -c copy output.mp4
```

---

## 七、待进一步调研

- [ ] URL 有效期精确测量（`dy_q` + 路径中的 token 何时失效）
- [ ] 是否可以通过 API（如 `/aweme/v1/feed/`）直接获取播放地址，绕过页面加载
- [ ] 不同类型页面（用户主页、搜索结果等）的 RENDER_DATA 结构是否一致
- [ ] `playApi` 字段的作用（可能是动态刷新 URL 的接口）
- [ ] `playerAccessKey` 字段的作用（可能用于防盗链校验）
