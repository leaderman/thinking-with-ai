# 抖音用户主页作品列表 API 调研

## 背景

需求：给定一个抖音博主的主页 URL，获取该博主发布的全部作品列表（视频 ID、标题、发布时间等）。

博主主页 URL 形如：
```
https://www.douyin.com/user/MS4wLjABAAAA...
```

## 调研方式

通过 Chrome CDP（Chrome DevTools Protocol）监听抖音用户主页的网络请求，滚动页面触发分页加载，拦截并分析所有 XHR/Fetch 请求，定位作品列表接口。

## 发现的接口

### 接口地址

```
GET https://www-hj.douyin.com/aweme/v1/web/aweme/post/
```

> 注意：实际请求打到的是 `www-hj.douyin.com`（CDN 分发域名），与页面域名 `www.douyin.com` 不同，但两个域名均可用。

### 核心请求参数

| 参数 | 说明 |
|---|---|
| `sec_user_id` | 博主 ID，即主页 URL 路径中 `/user/` 后面的字符串 |
| `max_cursor` | 分页游标，**第一页传 `0`**，后续传上一页返回的 `max_cursor` |
| `count` | 每页条数，页面默认使用 `18` |
| `aid` | 固定值 `6383` |
| `device_platform` | 固定值 `webapp` |

### 分页机制

抖音使用**基于时间戳的游标翻页**，游标值本质是发布时间的毫秒时间戳：

```
第 1 次请求: max_cursor=0
响应: { has_more: 1, max_cursor: 1761806726000, aweme_list: [...21条] }

第 2 次请求: max_cursor=1761806726000
响应: { has_more: 1, max_cursor: 1755698701000, aweme_list: [...18条] }

...

最后一页: { has_more: 0, aweme_list: [...N条] }  ← 结束条件
```

### 响应结构

```json
{
  "status_code": 0,
  "has_more": 1,
  "max_cursor": 1755698701000,
  "min_cursor": 1761806726000,
  "aweme_list": [
    {
      "aweme_id": "7630049734840962367",
      "desc": "视频标题",
      "create_time": 1776509400,
      "statistics": {
        "comment_count": 12,
        "digg_count": 340,
        "play_count": 8900
      },
      "video": { ... },
      "share_url": "https://..."
    }
  ]
}
```

## 关键约束

**签名参数问题**：请求中带有 `a_bogus`、`msToken`、`verifyFp` 等动态签名参数，**无法在浏览器外独立构造**。

解决方案：通过 CDP 在已登录的浏览器 Tab 内执行 `fetch`，由浏览器自动携带 Cookie 和签名，完全绕过签名问题。

## 实现方案

### 前提条件

1. Chrome 以 `--remote-debugging-port=9222` 启动
2. Chrome 中已登录抖音（任意抖音页面保持打开即可）

### 通用脚本

`fetch_user_posts.js`：传入博主主页 URL，自动翻页获取全部作品 ID。

```javascript
const WebSocket = require('ws');
const http = require('http');

// 用法: node fetch_user_posts.js <douyin_user_url>
const userUrl = process.argv[2];
if (!userUrl) {
  console.error('Usage: node fetch_user_posts.js <douyin_user_url>');
  process.exit(1);
}

const match = userUrl.match(/douyin\.com\/user\/([^/?#]+)/);
if (!match) {
  console.error('Invalid Douyin user URL');
  process.exit(1);
}
const SEC_USER_ID = decodeURIComponent(match[1]);

// 找第一个已打开的抖音 Tab，借用其 Cookie
function findDouyinTab() {
  return new Promise((resolve, reject) => {
    http.get('http://localhost:9222/json', (res) => {
      let raw = '';
      res.on('data', d => raw += d);
      res.on('end', () => {
        const tabs = JSON.parse(raw);
        const tab = tabs.find(t => t.type === 'page' && t.url && t.url.includes('douyin.com'));
        tab ? resolve(tab) : reject(new Error('No Douyin tab found in Chrome'));
      });
    }).on('error', reject);
  });
}

async function main() {
  const tab = await findDouyinTab();
  process.stderr.write(`Using tab: ${tab.title}\nsec_user_id: ${SEC_USER_ID}\n\n`);

  const ws = new WebSocket(tab.webSocketDebuggerUrl);
  let msgId = 1;
  const pending = new Map();

  ws.on('message', (data) => {
    const msg = JSON.parse(data);
    if (msg.id && pending.has(msg.id)) { pending.get(msg.id)(msg); pending.delete(msg.id); }
  });

  function evaluate(expression) {
    return new Promise((resolve) => {
      const id = msgId++;
      pending.set(id, (msg) => resolve(msg.result?.result?.value));
      ws.send(JSON.stringify({ id, method: 'Runtime.evaluate', params: { expression, returnByValue: true, awaitPromise: true } }));
    });
  }

  await new Promise(r => ws.on('open', r));

  let cursor = 0;
  let page = 0;
  const all = [];

  while (true) {
    page++;
    const raw = await evaluate(`(async () => {
      const p = new URLSearchParams({
        device_platform:'webapp', aid:'6383', channel:'channel_pc_web',
        sec_user_id:'${SEC_USER_ID.replace(/'/g, "\\'")}',
        max_cursor:'${cursor}',
        locate_query:'false', show_live_replay_strategy:'1',
        need_time_list:'${cursor === 0 ? 1 : 0}', time_list_query:'0',
        whale_cut_token:'', cut_version:'1', count:'18',
        publish_video_strategy_type:'2', from_user_page:'1',
        pc_client_type:'1', version_code:'290100', version_name:'29.1.0',
      });
      const res = await fetch('https://www-hj.douyin.com/aweme/v1/web/aweme/post/?'+p.toString(), { credentials:'include' });
      const j = await res.json();
      return JSON.stringify({
        status_code: j.status_code,
        has_more: j.has_more,
        max_cursor: j.max_cursor,
        items: (j.aweme_list || []).map(v => ({
          id: v.aweme_id,
          desc: v.desc ? v.desc.slice(0, 50) : '',
          create_time: v.create_time,
        })),
      });
    })()`);

    const data = JSON.parse(raw);
    if (data.status_code !== 0) {
      process.stderr.write(`Page ${page}: API error status_code=${data.status_code}\n`);
      break;
    }

    all.push(...data.items);
    process.stderr.write(`Page ${page}: +${data.items.length} | has_more=${data.has_more} | total=${all.length}\n`);

    if (!data.has_more || data.items.length === 0) break;
    cursor = data.max_cursor;
    await new Promise(r => setTimeout(r, 500));
  }

  process.stderr.write(`\nDone. Total posts: ${all.length}\n`);
  console.log(JSON.stringify(all, null, 2));
  ws.close();
}

main().catch(e => { console.error(e.message); process.exit(1); });
```

### 使用方式

```bash
# 直接输出到终端
node fetch_user_posts.js "https://www.douyin.com/user/MS4wLjABAAAA..."

# 保存为 JSON 文件（进度信息输出到 stderr，不污染文件内容）
node fetch_user_posts.js "https://www.douyin.com/user/MS4wLjABAAAA..." > posts.json
```

### 输出格式

```json
[
  {
    "id": "7630049734840962367",
    "desc": "飞书云文档和 Markdown 的那些事儿 #AI #Claude",
    "create_time": 1776509400
  },
  ...
]
```

## 实测数据

| 博主 | 作品数 | 发布时间跨度 |
|---|---|---|
| 发脾气🍀 | 310 条 | 2018-05 至今 |
| dada 不讲废话 | 188 条 | 2023-06 至今 |

## 附：喜欢列表接口

同样结构，可获取用户公开点赞的作品：

```
GET https://www-hj.douyin.com/aweme/v1/web/aweme/favorite/
```

参数与分页机制与作品列表接口完全一致，`sec_user_id` 传目标用户 ID 即可（需对方公开喜欢列表）。
