# 抖音视频流获取方案调研（五）：CDP 注入按钮 + 点击即获取

> 调研日期：2026-04-15  
> 核心问题：如何在不修改页面代码的情况下，通过 CDP 在页面上创建一个按钮，点击后获取当前播放视频的下载地址？

---

## 一、核心思路

```
CDP Runtime.evaluate
  → 执行 JS 在页面 DOM 中创建 <button>
  → 按钮绑定 click 事件
  → 点击时读 window.player.config.vid（当前值，无需监听）
  → 在页面内 fetch 详情 API（自动带 cookie）
  → 拿到下载 URL，复制到剪贴板
```

整个过程**不修改任何页面源码**，不需要安装浏览器插件，只通过 CDP 注入一段 JS 即可。

---

## 二、实现原理

### 为什么不需要监听 vid 变化？

用户切换视频后，`window.player` 会被替换为新的播放器实例，`window.player.config.vid` 自然就是新视频的 ID。

用户点击按钮的那一刻，直接读取即为当前值，**天然正确，无需任何监听机制**。

```javascript
// 点击时只需这一行
const vid = window.player?.config?.vid;
```

### 按钮注入方式

通过 `CDP Runtime.evaluate` 执行 JS，在 `document.body` 上 `appendChild` 一个 `position: fixed` 的按钮，悬浮在页面右下角，不影响原有布局。

---

## 三、完整注入代码

```javascript
(function() {
  // 避免重复创建
  if (document.getElementById('__dl_btn')) return '按钮已存在';

  const btn = document.createElement('button');
  btn.id = '__dl_btn';
  btn.textContent = '⬇ 下载当前视频';

  Object.assign(btn.style, {
    position:     'fixed',
    bottom:       '40px',
    right:        '40px',
    zIndex:       '99999',
    padding:      '12px 20px',
    fontSize:     '15px',
    fontWeight:   'bold',
    color:        '#fff',
    background:   '#fe2c55',    // 抖音红
    border:       'none',
    borderRadius: '8px',
    cursor:       'pointer',
    boxShadow:    '0 4px 12px rgba(0,0,0,0.3)',
  });

  btn.addEventListener('click', async () => {
    // Step 1：读取当前视频 ID
    const vid = window.player?.config?.vid;
    if (!vid) { alert('未找到视频 ID'); return; }

    btn.textContent = '获取中...';
    btn.disabled = true;

    try {
      // Step 2：动态获取 API 参数（从页面已有请求中提取，自动跟随版本）
      const baseParams = (() => {
        const entry = performance.getEntriesByType('resource')
          .find(e => e.name.includes('www.douyin.com/aweme') && e.name.includes('aid='));
        if (entry) {
          const u = new URL(entry.name);
          return {
            device_platform:     u.searchParams.get('device_platform'),
            aid:                 u.searchParams.get('aid'),
            channel:             u.searchParams.get('channel'),
            update_version_code: u.searchParams.get('update_version_code'),
          };
        }
        return { device_platform: 'webapp', aid: '6383', channel: 'channel_pc_web', update_version_code: '170400' };
      })();

      const params = new URLSearchParams({ ...baseParams, aweme_id: vid });
      const resp = await fetch(
        'https://www.douyin.com/aweme/v1/web/aweme/detail/?' + params,
        { headers: { 'Referer': 'https://www.douyin.com/' }, credentials: 'include' }
      );
      const data = await resp.json();

      // Step 3：取最高质量合流版本
      const best = (data.aweme_detail.video.bit_rate || [])
        .filter(b => !b.audio_file_id)
        .sort((a, b) => b.bit_rate - a.bit_rate)[0];

      const url = best?.play_addr?.url_list?.[0];

      if (url) {
        // Step 4：复制到剪贴板
        await navigator.clipboard.writeText(url);
        btn.textContent = '✓ 链接已复制！';
        console.log('[下载地址] vid=' + vid);
        console.log(url);
      } else {
        btn.textContent = '获取失败';
      }
    } catch(e) {
      btn.textContent = '出错: ' + e.message;
    }

    // 3 秒后恢复按钮
    setTimeout(() => {
      btn.textContent = '⬇ 下载当前视频';
      btn.disabled = false;
    }, 3000);
  });

  document.body.appendChild(btn);
  return '按钮创建成功';
})()
```

---

## 四、API 参数说明与动态获取

### 参数含义

| 参数 | 含义 | 示例值 |
|------|------|--------|
| `aweme_id` | 视频 ID，**唯一必须参数** | `7598562874448940328` |
| `aid` | App ID，标识抖音 PC Web 端 | `6383` |
| `device_platform` | 设备平台 | `webapp` |
| `channel` | 渠道来源 | `channel_pc_web` |
| `update_version_code` | 客户端版本号 | `170400` |

实测：**只有 `aweme_id` 是必须的**，其余参数省略后接口仍正常返回数据。保留它们是为了更贴近正常请求，降低被识别为异常请求的风险。

### 动态获取参数（推荐）

这些参数硬编码存在版本过期的风险。更好的方式是从**页面自己已发出的请求**中提取，与当前运行版本保持一致：

```javascript
function getApiParams() {
  // 从 performance entries 中找一条抖音 API 请求，解析其参数
  const entry = performance.getEntriesByType('resource')
    .find(e => e.name.includes('www.douyin.com/aweme') && e.name.includes('aid='));

  if (entry) {
    const u = new URL(entry.name);
    return {
      device_platform:     u.searchParams.get('device_platform'),
      aid:                 u.searchParams.get('aid'),
      channel:             u.searchParams.get('channel'),
      update_version_code: u.searchParams.get('update_version_code'),
    };
  }

  // 兜底：硬编码（极少变化）
  return {
    device_platform: 'webapp',
    aid: '6383',
    channel: 'channel_pc_web',
    update_version_code: '170400',
  };
}
```

`performance.getEntriesByType('resource')` 包含页面加载以来所有网络请求的 URL，抖音自己的 API 调用都带着这些参数，直接从中解析即可。这样即使抖音升级了版本号，代码也不需要改动。

---

## 五、通过 CDP 注入的 Node.js 代码

```javascript
await cdp.send('Runtime.evaluate', {
  expression: `/* 上方完整注入代码 */`,
  returnByValue: true,
});
```

---

## 五、按钮行为说明

| 状态 | 按钮文字 |
|------|---------|
| 默认 | ⬇ 下载当前视频 |
| 点击后请求中 | 获取中... |
| 成功 | ✓ 链接已复制！ |
| 3 秒后自动恢复 | ⬇ 下载当前视频 |

点击成功后：
- 下载 URL 自动写入**剪贴板**，可直接粘贴使用
- 同时在浏览器 **console** 中打印完整 URL 和 vid

---

## 六、下载命令（拿到 URL 后）

```bash
curl -L -g \
  -H "Referer: https://www.douyin.com/" \
  -H "User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36" \
  -o video_<vid>.mp4 \
  "<剪贴板中的 URL>"
```

> 注意：`-g` 参数防止 curl 对 URL 中的 `{}`、`[]` 做 glob 展开。

---

## 七、注意事项

- 按钮注入是**临时的**，页面刷新后消失，需重新注入
- `position: fixed` + 高 `zIndex` 确保按钮悬浮在抖音播放器之上
- 用 `id="__dl_btn"` 做去重判断，避免重复注入
- 详情 API 返回的 URL 有效期约 1~2 小时（含时间戳签名）
