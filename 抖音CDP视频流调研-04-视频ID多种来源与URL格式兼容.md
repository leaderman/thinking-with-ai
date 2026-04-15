# 抖音视频流获取方案调研（四）：视频 ID 的多种来源

> 调研日期：2026-04-15（更新：2026-04-15）  
> 核心问题：视频 ID（aweme_id）可以从哪里获取？哪个最可靠？  
> 补充验证：`/video/<id>` 格式 URL 下各来源的表现

---

## 一、名称统一说明

抖音的视频 ID 在不同上下文中有不同的叫法，但指向同一个值：

| 上下文 | 字段名 |
|--------|--------|
| 页面 URL 参数 | `modal_id` |
| API 请求/响应 | `aweme_id` |
| xgplayer 内部 | `vid` |

> `modal_id` 这个名字来自 PC 网页的交互形式——视频以弹窗（modal）叠加在页面上，所以路由参数叫 `modal_id`，本质就是视频 ID。

---

## 二、抖音 PC 网页的两种 URL 格式

| 格式 | 示例 | 场景 |
|------|------|------|
| `/jingxuan?modal_id=<id>` | `douyin.com/jingxuan?modal_id=7597065321907817755` | 精选/推荐 feed 页，视频以弹窗叠加 |
| `/video/<id>` | `douyin.com/video/7627277196940524671` | 视频直链，独立页面 |

两种格式下各来源的可用性不同（见下节）。

---

## 三、已知的 vid 来源及各格式下的可用性

### 来源 1：`window.player.config.vid`（最准确，**两种格式均有效**）

```javascript
// 注意：window.player.vid 直接访问是 undefined
// 实际路径是 window.player.config.vid
const vid = window.player?.config?.vid;
```

| URL 格式 | 是否可用 |
|----------|---------|
| `/jingxuan?modal_id=` | ✓ |
| `/video/<id>` | ✓ |

- 优点：播放器确认正在播的视频，与实际播放强绑定，两种格式均有效
- 缺点：字段不在顶层，直接访问 `player.vid` 是 `undefined`，需用 `player.config.vid`

---

### 来源 2：URL path 末段（`/video/<id>` 格式专用）

```javascript
const pathVid = window.location.pathname.split('/').filter(Boolean).pop();
// 需验证是否为纯数字
if (/^\d+$/.test(pathVid)) return pathVid;
```

| URL 格式 | 是否可用 |
|----------|---------|
| `/jingxuan?modal_id=` | ✗（path 末段是 `jingxuan`，非数字） |
| `/video/<id>` | ✓ |

---

### 来源 3：URL 参数 `modal_id`（`/jingxuan` 格式专用）

```javascript
const vid = new URLSearchParams(window.location.search).get('modal_id');
```

| URL 格式 | 是否可用 |
|----------|---------|
| `/jingxuan?modal_id=` | ✓ |
| `/video/<id>` | ✗（无 query 参数） |

- 缺点：视频切换动画进行中时可能与实际播放短暂不同步（待验证）

---

### 来源 4：`window.SSR_RENDER_DATA`（仅部分页面有效）

```javascript
const vid = window.SSR_RENDER_DATA?.app?.videoDetail?.aweme_id;
```

| URL 格式 | 是否可用 |
|----------|---------|
| `/jingxuan?modal_id=` | ✓（仅首屏，切换后不更新） |
| `/video/<id>` | ✗（该页面无此数据） |

---

## 四、推荐优先级

```
window.player.config.vid        ← 首选：两种格式均有效，与实际播放强绑定
  ↓ 取不到
URL path 末段（纯数字）          ← 次选A：/video/<id> 格式
URL modal_id 参数               ← 次选B：/jingxuan?modal_id= 格式
  ↓ 取不到
SSR_RENDER_DATA.app.videoDetail ← 兜底：仅 /jingxuan 首屏有效
```

---

## 五、健壮的取 vid 代码（支持两种 URL 格式）

```javascript
function getCurrentVid() {
  // 首选：player 内部（两种格式均有效）
  const playerVid = window.player?.config?.vid;
  if (playerVid) return playerVid;

  // 次选A：/video/<id> 格式，取 path 最后一段
  const pathVid = window.location.pathname.split('/').filter(Boolean).pop();
  if (/^\d+$/.test(pathVid)) return pathVid;

  // 次选B：/jingxuan?modal_id=<id> 格式
  const urlVid = new URLSearchParams(window.location.search).get('modal_id');
  if (urlVid) return urlVid;

  // 兜底：SSR 数据（仅 /jingxuan 首屏）
  const ssrVid = window.SSR_RENDER_DATA?.app?.videoDetail?.aweme_id;
  if (ssrVid) return ssrVid;

  return null;
}
```

---

## 六、实测验证结果汇总

### `/jingxuan?modal_id=` 格式（vid=7598562874448940328）

| 来源 | 结果 |
|------|------|
| `player.config.vid` | ✓ |
| URL path 末段 | ✗（末段是 `jingxuan`） |
| URL `modal_id` | ✓ |
| `SSR_RENDER_DATA` | ✓（仅首屏） |

### `/video/<id>` 格式（vid=7627277196940524671）

| 来源 | 结果 |
|------|------|
| `player.config.vid` | ✓ |
| URL path 末段 | ✓ |
| URL `modal_id` | ✗（无此参数） |
| `SSR_RENDER_DATA` | ✗（页面无此数据） |

两种格式下调用详情 API 均返回 200，下载地址完全可用。

---

## 七、待验证

- [ ] 视频切换动画进行中时，`modal_id` 与 `player.config.vid` 是否存在短暂不一致？
- [ ] 是否还有其他 URL 格式（如分享短链 `v.douyin.com/xxx`）需要兼容？
