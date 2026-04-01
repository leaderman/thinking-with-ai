# 通过 Chrome CDP 抓取 X 用户推文

## 核心思路

X 是 SPA，推文数据不在 HTML 里，而是通过 GraphQL 接口异步加载。整体实现分三层：

1. **建立 CDP 连接** — 通过 Chrome 的 HTTP 接口开一个新 Tab，用 WebSocket 连接 Chrome DevTools Protocol
2. **拦截 `UserTweets` 接口** — 监听网络事件，捕获响应体并解析推文数据
3. **滚动触发分页** — X 使用滚动懒加载，滚动到底部会自动触发新的 `UserTweets` 请求

## 拦截的接口

```
https://x.com/i/api/graphql/{hash}/UserTweets?variables=...
```

hash 是动态的，用 URL 包含 `UserTweets` 字符串来匹配。

## 响应数据结构

```
data
└── user
    └── result
        └── timeline
            └── timeline
                └── instructions[]
                    └── [type: "TimelineAddEntries"]
                        └── entries[]
                            └── content
                                └── itemContent
                                    └── tweet_results
                                        └── result
                                            ├── rest_id        ← 推文 ID
                                            ├── tweet?         ← 有时有这层，有时没有
                                            └── legacy
                                                ├── created_at ← 发布时间
                                                └── full_text  ← 推文内容
```

注意点：
- 路径是 `timeline.timeline`，不是 `timeline_v2.timeline`（X 已改版）
- `result` 下有时直接挂 `legacy`，有时套了一层 `tweet`，需要兼容两种情况

## 时间格式

`created_at` 的原始格式为 Twitter 标准时间格式，时区固定为 UTC：

```
Mon Mar 30 16:03:39 +0000 2026
```

输出时可转换为更易读的 ISO 格式：

```js
new Date(created_at).toISOString().slice(0, 19).replace('T', ' ')
// => "2026-03-30 16:03:39"
```

## 实现代码

```js
import WebSocket from 'ws';

const CDP_HOST = 'localhost';
const CDP_PORT = 9222;
const TARGET_URL = 'https://x.com/thedankoe';
const MAX_TWEETS = 50;
const SCROLL_STEP = 2000;
const SCROLL_PAUSE = 500;
const IDLE_AFTER_SCROLLS = 3;

// 开新 Tab
async function openTab(url) {
  const res = await fetch(`http://${CDP_HOST}:${CDP_PORT}/json/new?${encodeURIComponent(url)}`, { method: 'PUT' });
  if (!res.ok) throw new Error(`Failed to open tab: HTTP ${res.status}`);
  return await res.json();
}

async function closeTab(tabId) {
  await fetch(`http://${CDP_HOST}:${CDP_PORT}/json/close/${tabId}`).catch(() => {});
}

// 连接 WebSocket
function connectWs(wsUrl) {
  return new Promise((resolve, reject) => {
    const ws = new WebSocket(wsUrl);
    ws.once('open', () => resolve(ws));
    ws.once('error', reject);
  });
}

// 发送 CDP 命令
let _seq = 1;
function send(ws, method, params = {}) {
  return new Promise((resolve, reject) => {
    const id = _seq++;
    const handler = (raw) => {
      const msg = JSON.parse(raw.toString());
      if (msg.id !== id) return;
      ws.off('message', handler);
      if (msg.error) reject(new Error(msg.error.message));
      else resolve(msg.result);
    };
    ws.on('message', handler);
    ws.send(JSON.stringify({ id, method, params }));
  });
}

// 从 UserTweets 响应中提取推文
function extractTweetsFromResponse(data) {
  const tweets = [];
  const instructions =
    data?.data?.user?.result?.timeline?.timeline?.instructions ?? [];

  for (const instruction of instructions) {
    if (instruction.type !== 'TimelineAddEntries') continue;
    for (const entry of instruction.entries ?? []) {
      const tweetResult = entry.content?.itemContent?.tweet_results?.result;
      if (tweetResult) {
        const tweet = parseTweet(tweetResult);
        if (tweet) tweets.push(tweet);
      }
    }
  }
  return tweets;
}

// 兼容 result 直接挂 legacy 或套了一层 tweet 的两种结构
function parseTweet(result) {
  const tweet = result?.tweet ?? result;
  const legacy = tweet?.legacy;
  const id = tweet?.rest_id;
  if (!legacy || !id) return null;
  return {
    id,
    time: legacy.created_at,
    text: legacy.full_text,
  };
}

async function main() {
  const tab = await openTab('about:blank');
  const ws = await connectWs(tab.webSocketDebuggerUrl);

  const tweets = new Map();
  const pending = new Set();    // 已发出、等待响应的 requestId
  const processing = new Set(); // 正在处理响应体的 requestId

  const onMessage = async (raw) => {
    const msg = JSON.parse(raw.toString());
    if (!msg.method) return;

    if (msg.method === 'Network.requestWillBeSent') {
      const { requestId, request } = msg.params;
      if (request.url.includes('UserTweets')) pending.add(requestId);
    } else if (msg.method === 'Network.loadingFinished') {
      const { requestId } = msg.params;
      if (!pending.has(requestId)) return;
      pending.delete(requestId);
      processing.add(requestId);
      try {
        const result = await send(ws, 'Network.getResponseBody', { requestId });
        const text = result.base64Encoded
          ? Buffer.from(result.body, 'base64').toString()
          : result.body;
        const found = extractTweetsFromResponse(JSON.parse(text));
        for (const t of found) tweets.set(t.id, t);
      } catch {}
      processing.delete(requestId);
    }
  };

  try {
    await send(ws, 'Network.enable');
    ws.on('message', onMessage);
    await send(ws, 'Page.navigate', { url: TARGET_URL });

    // 等待初始加载
    await new Promise(r => setTimeout(r, 5000));

    // 滚动触发分页
    let idleScrolls = 0;
    while (tweets.size < MAX_TWEETS) {
      const before = tweets.size;
      await send(ws, 'Runtime.evaluate', {
        expression: `window.scrollBy(0, ${SCROLL_STEP})`,
      });

      // 等待 pending/processing 全部完成，再判断是否有新推文
      await new Promise(r => setTimeout(r, SCROLL_PAUSE));
      const deadline = Date.now() + 5000;
      while ((pending.size > 0 || processing.size > 0) && Date.now() < deadline) {
        await new Promise(r => setTimeout(r, 200));
      }

      if (tweets.size === before) {
        if (++idleScrolls >= IDLE_AFTER_SCROLLS) break;
      } else {
        idleScrolls = 0;
      }
    }
  } finally {
    ws.off('message', onMessage);
    ws.close();
    await closeTab(tab.id);
  }
}

main().catch(console.error);
```

## 关键坑点

**时序问题**：`getResponseBody` 是异步操作，如果滚动后立刻检查 `tweets.size`，响应体还没解析完，会误判为"没有新推文"，导致提前停止。

解决方式：用 `pending` 和 `processing` 两个集合追踪请求状态，滚动后等两个集合都清空再做判断。
