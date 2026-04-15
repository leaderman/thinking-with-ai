# 抖音视频流获取方案调研（三）：以当前播放视频为起点

> 调研日期：2026-04-15  
> 承接：RESEARCH_02_feed_api.md  
> 核心问题：不依赖用户行为（滚动方向），以当前播放视频本身为起点获取下载地址

---

## 一、核心思路

不监听网络请求，不依赖 RENDER_DATA，不关心用户滚动方向。  
只做两件事：

```
当前页面 URL → 取 modal_id → 调详情 API → 拿下载地址
```

无论用户向上滚还是向下滚，只要当前视频在播放，页面 URL 的 `modal_id` 就是当前视频的 vid，随时可取。

---

## 二、Step 1：获取当前视频 vid

### 首选：从页面 URL 取 modal_id（最可靠）

```javascript
const vid = new URLSearchParams(window.location.search).get('modal_id');
// 示例：7598562874448940328
```

### 兜底：从 player 内部递归查找

```javascript
// window.player.config.vid 存有当前 vid
// 但字段路径不是顶层，需递归查找
function findVid(obj, visited = new WeakSet(), depth = 0) {
  if (!obj || depth > 6 || typeof obj !== 'object' || visited.has(obj)) return null;
  visited.add(obj);
  for (const key of ['vid', 'aweme_id', 'awemeId']) {
    if (obj[key] && typeof obj[key] === 'string') return obj[key];
  }
  for (const k of Object.keys(obj)) {
    try {
      const r = findVid(obj[k], visited, depth + 1);
      if (r) return r;
    } catch {}
  }
  return null;
}
const vid = findVid(window.player); // 实测路径：player.config.vid
```

> **注意**：`window.player.vid` 直接访问是 `undefined`，必须递归查找。  
> `modal_id` 方案更简单，优先用它。

---

## 三、Step 2：在页面内调详情 API

用 `fetch` **在页面内**发请求，自动携带当前用户的 Cookie，无需额外处理认证。

```javascript
async function getVideoDetail(vid) {
  const params = new URLSearchParams({
    device_platform: 'webapp',
    aid: '6383',
    channel: 'channel_pc_web',
    aweme_id: vid,
    update_version_code: '170400',
    pc_client_type: '1',
    version_code: '170400',
    version_name: '17.4.0',
  });

  const resp = await fetch(
    'https://www.douyin.com/aweme/v1/web/aweme/detail/?' + params,
    {
      headers: { 'Referer': 'https://www.douyin.com/' },
      credentials: 'include',   // 关键：带上 cookie
    }
  );

  const data = await resp.json();
  const video = data.aweme_detail.video;
  const bitRates = video.bit_rate || [];

  // 合流（video+audio 合并）：过滤掉有独立音频的 DASH 条目，按码率降序
  const muxed = bitRates
    .filter(b => !b.audio_file_id)
    .sort((a, b) => b.bit_rate - a.bit_rate);

  return {
    vid: data.aweme_detail.aweme_id,
    width: video.width,
    height: video.height,
    duration: video.duration,          // 毫秒
    qualities: muxed.map(b => ({
      gearName: b.gear_name,
      bitRate: b.bit_rate,
      dataSize: b.data_size,
      url: b.play_addr.url_list[0],   // 取第一个 CDN 镜像
    })),
    audioUrl: video.bit_rate_audio?.[0]?.url_list?.[0] ?? null,
  };
}
```

---

## 四、API 响应数据结构

```
aweme_detail
├── aweme_id                            # 视频 ID
├── desc                                # 视频标题
└── video
    ├── width / height / duration
    ├── bit_rate[]                      # 所有画质（下划线命名）
    │   ├── gear_name                   # 画质档位名
    │   ├── bit_rate                    # 码率 bps
    │   ├── data_size                   # 文件大小 bytes（部分条目为空）
    │   ├── audio_file_id               # 有此字段 = DASH 分离流，无 = 合流
    │   └── play_addr.url_list[]        # CDN 镜像列表，取 [0]
    └── bit_rate_audio[]                # 独立音频（配合 DASH 视频）
        └── url_list[]
```

---

## 五、实测结果（vid=7598562874448940328）

- 标题：《纸牌屋》深度解析 第三期
- 分辨率：1920x1080，时长：1015.9s
- **返回合流画质数：21 种**（比 RENDER_DATA 的 11 种更全）

| # | gearName | 码率 |
|---|----------|------|
| 0 | normal_1080_0 | 1013k bps |
| 1 | normal_1080_0 | 957k bps |
| 2 | normal_720_0 | 771k bps |
| 3 | normal_540_0 | 751k bps |
| 4 | 1080_1_1 | 688k bps |
| … | … | … |
| 20 | 540_3_1 | 205k bps |

---

## 六、完整 CDP 实现流程

```javascript
// 在 CDP Runtime.evaluate 中执行（awaitPromise: true）
const result = await cdp.send('Runtime.evaluate', {
  expression: `
    (async function() {
      // Step 1: 取 vid
      const vid = new URLSearchParams(window.location.search).get('modal_id');
      if (!vid) return JSON.stringify({ error: 'no modal_id in URL' });

      // Step 2: 调详情 API
      const params = new URLSearchParams({
        device_platform: 'webapp', aid: '6383',
        channel: 'channel_pc_web', aweme_id: vid,
        update_version_code: '170400',
      });
      const resp = await fetch(
        'https://www.douyin.com/aweme/v1/web/aweme/detail/?' + params,
        { headers: { 'Referer': 'https://www.douyin.com/' }, credentials: 'include' }
      );
      const data = await resp.json();
      const video = data.aweme_detail.video;
      const best = (video.bit_rate || [])
        .filter(b => !b.audio_file_id)
        .sort((a, b) => b.bit_rate - a.bit_rate)[0];

      return JSON.stringify({
        vid,
        url: best?.play_addr?.url_list?.[0],
        gearName: best?.gear_name,
        bitRate: best?.bit_rate,
      });
    })()
  `,
  returnByValue: true,
  awaitPromise: true,
});

const { vid, url } = JSON.parse(result.result.value);
```

---

## 七、下载命令

```bash
curl -L -g \
  -H "Referer: https://www.douyin.com/" \
  -H "User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36" \
  -o video_<vid>.mp4 \
  "<url>"
```

> 注意加 `-g` 参数，防止 curl 对 URL 中的 `{}`、`[]` 做 glob 展开。

---

## 八、方案优势对比

| | RESEARCH.md（RENDER_DATA） | RESEARCH_02（Feed API 拦截） | **本方案（详情 API）** |
|---|---|---|---|
| 适用范围 | 仅首屏第一个视频 | 所有预加载视频 | **任意当前播放视频** |
| 依赖网络拦截 | 否 | 是（需页面加载时监听） | **否** |
| 依赖用户行为 | 否 | 否 | **否** |
| 实现复杂度 | 低 | 高 | **低** |
| 画质选项数量 | 11 种 | 与 API 一致 | **21 种（最全）** |
| URL 实时性 | 页面加载时生成 | 页面加载时生成 | **调用时实时生成（最新）** |

**结论：本方案是三种方案中最简单、最通用、最可靠的。**

---

## 九、待进一步调研

- [ ] 详情 API 是否需要登录态？（当前测试有登录，需验证未登录场景）
- [ ] `modal_id` 不存在时的兜底方案（如直接从分享链接进入的场景）
- [ ] API 返回的 URL 有效期是否与 RENDER_DATA 中的一致？
