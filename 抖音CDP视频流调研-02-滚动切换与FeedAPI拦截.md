# 抖音视频流获取方案调研（二）：滚动切换与 Feed API

> 调研日期：2026-04-15  
> 承接：RESEARCH.md（首屏视频提取方案）  
> 核心问题：滚动到下一个视频后，如何获取新视频的下载地址？

---

## 一、关键发现：RENDER_DATA 不会动态更新

滚动切换视频后：
- 页面 URL 中的 `modal_id` 会更新（如 `7597065321907817755` → `7605237006045547785`）
- 但 `document#RENDER_DATA` 和 `window.SSR_RENDER_DATA` **始终保持首屏内容不变**
- 因此 RESEARCH.md 中的 RENDER_DATA 方案**只对第一个视频有效**

---

## 二、关键发现：切换视频时没有新的 API 请求

实测观察：滚动切换视频时，页面**不会发出新的视频信息请求**。

原因是：**页面初始加载时已经批量预拉取了大量后续视频的完整数据**（包括所有画质的播放地址），存放在内存中，切换时直接使用。

---

## 三、负责批量预加载视频数据的 API

页面加载期间会调用以下接口，每个接口返回一批视频的完整信息：

### 接口一：主推荐流
```
GET https://www.douyin.com/aweme/v2/web/module/feed/
    ?device_platform=webapp&aid=6383&channel=channel_pc_web&...
```
- 每次返回 10~20 个视频
- 页面加载期间会连续调用多次（懒加载）
- 字段路径：响应体 `aweme_list[]`

### 接口二：合集视频列表（精选频道专用）
```
GET https://www.douyin.com/aweme/v1/web/mix/aweme/
    ?device_platform=webapp&aid=6383&channel=channel_pc_web&...
```
- 返回当前合集的所有视频（实测 15 个）
- 字段路径：响应体 `aweme_list[]`

### 其他相关接口
```
GET https://www.douyin.com/aweme/v1/web/aweme/favorite/   # 收藏视频列表
```

---

## 四、响应体数据结构

```
aweme_list[]
└── [i]
    ├── aweme_id                          # 视频 ID
    ├── video
    │   ├── width / height / duration
    │   ├── bit_rate[]                    # 所有画质（注意：下划线命名，区别于 RENDER_DATA）
    │   │   ├── gear_name                 # 画质名称
    │   │   ├── bit_rate                  # 码率 bps
    │   │   ├── data_size                 # 文件大小 bytes
    │   │   ├── audio_file_id             # 存在则为 DASH 分离流
    │   │   └── play_addr
    │   │       └── url_list[]            # CDN 镜像列表，取 [0] 即可
    │   └── bit_rate_audio[]              # 独立音频流（配合 DASH）
    │       └── url_list[]
    └── ...
```

> **注意字段命名差异**：  
> - RENDER_DATA 中用驼峰命名：`bitRateList`、`playAddr`、`audioFileId`  
> - Feed API 响应中用下划线命名：`bit_rate`、`play_addr`、`audio_file_id`

---

## 五、提取逻辑（JS 代码片段）

```javascript
// 从 Feed API 响应体中提取指定视频的最高质量合流下载地址
function extractDownloadUrl(awemeList, targetVid = null) {
  const items = targetVid
    ? awemeList.filter(item => item.aweme_id === targetVid)
    : awemeList;

  return items.map(item => {
    const bitRates = item.video?.bit_rate || [];
    // 合流（无独立音频）的最高码率版本
    const best = bitRates
      .filter(b => !b.audio_file_id)
      .sort((a, b) => b.bit_rate - a.bit_rate)[0];

    return {
      vid: item.aweme_id,
      gearName: best?.gear_name,
      bitRate: best?.bit_rate,
      dataSize: best?.data_size,
      url: best?.play_addr?.url_list?.[0],
    };
  });
}
```

---

## 六、CDP 实现方案：拦截 Feed API 响应体

```javascript
// 1. 启用 Network 域
await cdp.send('Network.enable');

// 2. 监听响应，找到 feed 接口
cdp.on('Network.responseReceived', ({ requestId, response }) => {
  const url = response.url;
  if (/aweme\/v[12]\/web\/(module\/feed|mix\/aweme)/.test(url)) {
    feedRequestIds.push(requestId);
  }
});

// 3. 页面加载完成后读取响应体
const body = await cdp.send('Network.getResponseBody', { requestId });
const data = JSON.parse(body.body);
const urls = extractDownloadUrl(data.aweme_list);
```

---

## 七、两种方案对比总结

| 场景 | 方案 | 接口 / 字段 |
|------|------|------------|
| 获取当前首屏视频 | RENDER_DATA | `app.videoDetail.video.bitRateList[].playAddr[].src` |
| 获取切换后的视频 | 拦截 Feed API 响应 | `aweme_list[].video.bit_rate[].play_addr.url_list[0]` |
| 批量获取所有预加载视频 | 拦截 Feed API 响应 | 同上，遍历整个 `aweme_list` |

---

## 八、其他观察

- `window.player` — 当前视频的 xgplayer 实例（含 `vid`、`currentTime`、`duration`）
- `window.nextPlayer` — 下一个视频已预初始化的播放器实例（可读取 `vid`）
- 两个播放器的 `src` 均为 `blob:` URL，不是直接下载地址
- `window.playerPreloader` 和 `window.playerActxs` 在视频播放中途为空（已消费）

---

## 九、待进一步调研

- [ ] Feed API 是否需要登录态（Cookie）才能调用？
- [ ] `aweme/v2/web/module/feed/` 的参数规律，能否直接构造请求批量拉取？
- [ ] 滚动到底部时会触发新的 feed 请求，规律是否一致？
