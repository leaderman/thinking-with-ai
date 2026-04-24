# 保存 X (Twitter) 文章为 Markdown 的操作手册

## 前提条件

- Chrome 已开启 CDP，默认端口 9222
- Node.js >= 21（内置 `WebSocket`，无需任何第三方包）

---

## 第一步：确认页面已在 Chrome 中打开

通过 CDP HTTP 接口列出当前所有标签页，找到目标 URL 对应的页面和它的 WebSocket 调试地址：

```
GET http://localhost:9222/json/list
```

如果目标页面尚未打开，则通过浏览器级 WebSocket（`/json/version` 获取地址）使用 `Target.createTarget` 开一个新标签页，再用 `Page.navigate` 导航到目标 URL。

---

## 第二步：等待页面完全加载

X 是单页应用，加载分两个阶段：

1. **HTML 框架加载**：监听 `Page.loadEventFired` 事件
2. **JS 渲染完成**：`loadEventFired` 之后额外等待 **3～5 秒**，让 React 完成渲染

---

## 第三步：滚动页面，触发懒加载图片

X 的文章图片全部懒加载，必须滚动到图片位置才会加载进 DOM。

操作方式：用 `Runtime.evaluate` 注入 JS，从页面顶部开始，每次滚动约 80% 视口高度，每步间隔 300ms，一直滚到页面底部，再滚回顶部。

滚动完成后再等待 **1～2 秒**，让图片请求完成。

---

## 第四步：提取页面内容

### 文字内容
直接读取文章容器的 `innerText`，这是最稳定的方式，不依赖任何 class 或 data-testid。

X 文章页面的文章容器通常是 `main` 标签下的 `article` 元素，但**不要硬依赖这个选择器**，如果找不到就降级用 `document.body.innerText`。

### 图片列表
用 `querySelectorAll('img')` 获取页面上所有图片，过滤条件：
- `src` 不为空，且不是 `data:` base64 图
- 不是头像图（URL 中不含 `profile_images`）
- 不是表情图（URL 中不含 `emoji`）
- 宽度大于 80px（排除像素追踪图）
- 对 src 去重（同一张图可能出现多次）

X 的图片 URL 形如：
```
https://pbs.twimg.com/media/XXXX?format=jpg&name=small
```
下载时将 `name=small` / `name=medium` 替换为 `name=large` 获取高清版本。

### 元数据
- 标题：优先读文章内 `article h1` 元素的文字（X Article 长文有独立标题）；若不存在则取 `article.innerText` 第一条长度 > 10 的非空行作为标题
- 发布时间：读文章内 `<time>` 元素的 `datetime` 属性
- 作者：读 `[data-testid="User-Name"]` 区域的文字（这个 testid 相对稳定）

---

## 第五步：下载图片到本地

目录结构固定如下，以文章 URL 中的 **status ID** 作为目录名：

```
/tmp/{status_id}/
├── .code/
│   └── save-x-article.js   ← 执行脚本（隐藏目录）
├── article.md
└── images/
    ├── cover.jpg            ← 封面图（第一张图）
    ├── img-01.jpg
    ├── img-02.jpg
    └── ...
```

- 基础目录：`/tmp`
- 文章目录：`/tmp/{status_id}/`（status ID 取自 URL，如 `https://x.com/user/status/2040202068091142208` → `2040202068091142208`）
- 文章文件：`/tmp/{status_id}/article.md`
- 图片目录：`/tmp/{status_id}/images/`
- **封面图固定命名为 `cover.jpg`**，其余图片按顺序命名 `img-01.jpg`、`img-02.jpg`……
- 下载时带上 `User-Agent` 请求头，避免被拒绝

---

## 第六步：生成 Markdown

结构如下：

```markdown
# 文章标题

**Author:** 作者名 ([@handle](https://x.com/handle))
**Published:** 发布日期
**Source:** 原文 URL

（正文段落）

![Image](images/img-01.jpg)

（继续正文）
...
```

**封面图**：`cover.jpg` 只需下载保存，**不在 Markdown 中插入引用**，供外部调用方使用。

**其余图片插入位置**：由于 X 的 DOM 结构复杂，图片在正文中的精确位置难以稳定获取。
实用策略：`img-01.jpg` 起，**按照它们在 DOM 中出现的顺序**，穿插在对应段落附近插入。

**噪声清理**：`innerText` 会包含点赞数、转发数、浏览量等 UI 文字（如 "28 79 795 372K"），以及重复的标题行，需要识别并去掉。判断依据：
- 纯数字行或带 K/M 后缀的数字行
- 与标题完全重复的行
- 作者名和 @handle 独占一行（已在 header 里出现过）

---

## 执行脚本

### 脚本位置

脚本以代码块形式保存在本手册末尾。执行前先将其写入文章目录下的隐藏目录：

```bash
mkdir -p /tmp/{status_id}/.code
# 将下方代码块内容写入：
# /tmp/{status_id}/.code/save-x-article.js
```

### 运行方式

```bash
# 用法：node /tmp/{status_id}/.code/save-x-article.js <文章URL> <CDP WebSocket地址>
# CDP WebSocket地址 从 http://localhost:9222/json/list 的 webSocketDebuggerUrl 字段获取

node /tmp/2040202068091142208/.code/save-x-article.js \
  "https://x.com/Jahjiren/status/2040202068091142208" \
  "ws://localhost:9222/devtools/page/XXXX"
```

### 校验

执行前用以下命令验证脚本语法完好：

```bash
node --check /tmp/{status_id}/.code/save-x-article.js
```

若脚本不存在或校验失败，则从手册末尾的代码块重新写入。若运行时出现提取异常（图片为 0、正文为空等），说明 X 的 DOM 结构已变更，需根据实际情况动态修复并更新手册中的代码块。

---

## 脚本代码

```javascript
// save-x-article.js
// 用法: node save-x-article.js <article_url> <ws_url>
// 依赖: Node.js >= 21 (内置 WebSocket，无需第三方包)

import https from 'https';
import http from 'http';
import fs from 'fs';
import path from 'path';
import { URL } from 'url';

const [,, ARTICLE_URL, WS_URL] = process.argv;
if (!ARTICLE_URL || !WS_URL) {
  console.error('用法: node save-x-article.js <article_url> <ws_url>');
  process.exit(1);
}

const STATUS_ID = ARTICLE_URL.match(/\/status\/(\d+)/)?.[1];
if (!STATUS_ID) {
  console.error('无法从 URL 中解析 status ID:', ARTICLE_URL);
  process.exit(1);
}

const OUT_DIR = `/tmp/${STATUS_ID}`;
const IMAGES_DIR = path.join(OUT_DIR, 'images');

let msgId = 1;
const pending = new Map();
const eventWaiters = new Map();

function sleep(ms) {
  return new Promise(r => setTimeout(r, ms));
}

function downloadFile(url, dest) {
  return new Promise((resolve, reject) => {
    url = url.replace(/name=(small|medium|orig)/, 'name=large');
    const parsed = new URL(url);
    const client = parsed.protocol === 'https:' ? https : http;
    const file = fs.createWriteStream(dest);
    const req = client.get(url, {
      headers: {
        'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
        'Referer': 'https://x.com/',
      }
    }, (res) => {
      if (res.statusCode === 301 || res.statusCode === 302) {
        file.close();
        fs.unlinkSync(dest);
        return downloadFile(res.headers.location, dest).then(resolve).catch(reject);
      }
      res.pipe(file);
      file.on('finish', () => { file.close(); resolve(); });
    });
    req.on('error', (err) => { fs.unlink(dest, () => {}); reject(err); });
  });
}

async function main() {
  console.log('Connecting to Chrome CDP...');
  const ws = new WebSocket(WS_URL);

  await new Promise((resolve, reject) => {
    ws.addEventListener('open', resolve);
    ws.addEventListener('error', (e) => reject(new Error(e.message || 'WebSocket error')));
  });

  ws.addEventListener('message', (event) => {
    const msg = JSON.parse(event.data);
    if (msg.id && pending.has(msg.id)) {
      const { resolve, reject } = pending.get(msg.id);
      pending.delete(msg.id);
      if (msg.error) reject(new Error(msg.error.message));
      else resolve(msg.result);
    }
    if (msg.method && eventWaiters.has(msg.method)) {
      eventWaiters.get(msg.method)(msg.params);
      eventWaiters.delete(msg.method);
    }
  });

  const send = (method, params = {}) => new Promise((resolve, reject) => {
    const id = msgId++;
    pending.set(id, { resolve, reject });
    ws.send(JSON.stringify({ id, method, params }));
  });

  const waitEvent = (eventName, timeout = 20000) => new Promise((resolve, reject) => {
    const timer = setTimeout(() => reject(new Error(`Timeout: ${eventName}`)), timeout);
    eventWaiters.set(eventName, (params) => { clearTimeout(timer); resolve(params); });
  });

  await send('Page.enable');
  await send('Runtime.enable');

  // 第一步：导航到文章
  console.log('Navigating to article...');
  await send('Page.navigate', { url: ARTICLE_URL });
  await waitEvent('Page.loadEventFired');
  console.log('Page loaded, waiting for React render...');
  await sleep(4000);

  // 第二步：滚动触发懒加载
  console.log('Scrolling to trigger lazy loading...');
  await send('Runtime.evaluate', {
    expression: `(async () => {
      const delay = ms => new Promise(r => setTimeout(r, ms));
      const step = Math.floor(window.innerHeight * 0.8);
      let pos = 0;
      while (pos < document.body.scrollHeight) {
        window.scrollTo(0, pos);
        await delay(300);
        pos += step;
      }
      window.scrollTo(0, 0);
      await delay(1500);
    })()`,
    awaitPromise: true,
    timeout: 30000,
  });
  await sleep(2000);

  // 第三步：提取元数据
  console.log('Extracting metadata...');
  const { result: { value: meta } } = await send('Runtime.evaluate', {
    expression: `(() => {
      const h1El = document.querySelector('article h1');
      const articleEl = document.querySelector('main article');
      const firstLine = articleEl
        ? (articleEl.innerText || '').split('\\n').map(l => l.trim()).find(l => l.length > 10)
        : '';
      return {
        title: h1El ? h1El.innerText.trim() : (firstLine || ''),
        published: (document.querySelector('article time') || {}).getAttribute?.('datetime') || '',
        author: (document.querySelector('[data-testid="User-Name"]') || {}).innerText?.trim() || '',
      };
    })()`,
    returnByValue: true,
  });
  console.log('Meta:', JSON.stringify(meta));

  // 第四步：提取正文
  const { result: { value: rawText } } = await send('Runtime.evaluate', {
    expression: `(() => {
      const el = document.querySelector('main article');
      return el ? el.innerText : document.body.innerText;
    })()`,
    returnByValue: true,
  });

  // 第五步：提取图片
  const { result: { value: imageUrls } } = await send('Runtime.evaluate', {
    expression: `(() => {
      const seen = new Set();
      return Array.from(document.querySelectorAll('img')).filter(img => {
        const src = img.src || '';
        if (!src || src.startsWith('data:')) return false;
        if (src.includes('profile_images') || src.includes('emoji')) return false;
        if (img.width <= 80) return false;
        if (seen.has(src)) return false;
        seen.add(src);
        return true;
      }).map(img => img.src);
    })()`,
    returnByValue: true,
  });
  console.log(`Found ${imageUrls.length} images`);

  // 第六步：清理正文噪声
  const titleLine = meta.title || '';
  const authorLines = meta.author ? meta.author.split('\n').map(l => l.trim()) : [];
  const cleanLines = (rawText || '').split('\n').filter(line => {
    const t = line.trim();
    if (!t) return false;
    if (/^[\d\s.,KMB%]+$/.test(t)) return false;
    if (t === titleLine) return false;
    if (authorLines.includes(t)) return false;
    if (/^(Reply|Repost|Like|Bookmark|Share|Views|Follow|Following|Followers|Likes|Reposts|Quotes|Want to publish|Upgrade to Premium|Paid partnership)$/i.test(t)) return false;
    return true;
  });

  // 第七步：下载图片
  fs.mkdirSync(IMAGES_DIR, { recursive: true });
  console.log('Downloading images...');
  const localImages = [];
  for (let i = 0; i < imageUrls.length; i++) {
    const filename = i === 0 ? 'cover.jpg' : `img-${String(i).padStart(2, '0')}.jpg`;
    const dest = path.join(IMAGES_DIR, filename);
    try {
      await downloadFile(imageUrls[i], dest);
      localImages.push(filename);
      console.log(`  Downloaded: ${filename}`);
    } catch (err) {
      console.error(`  Failed: ${filename} — ${err.message}`);
    }
  }

  // 第八步：生成 Markdown
  const authorParts = meta.author.split('\n');
  const displayName = authorParts[0] || 'Unknown';
  const handle = authorParts.find(p => p.startsWith('@')) || '';
  const handleClean = handle.replace('@', '');
  const publishedDate = meta.published ? meta.published.split('T')[0] : '';

  let md = `# ${titleLine || 'X Article'}\n\n`;
  md += `**Author:** ${displayName}${handle ? ` ([${handle}](https://x.com/${handleClean}))` : ''}\n`;
  md += `**Published:** ${publishedDate}\n`;
  md += `**Source:** ${ARTICLE_URL}\n\n`;

  // 封面图只保存文件，不写入 markdown
  const restImages = localImages.slice(1);
  const chunkSize = restImages.length > 0
    ? Math.floor(cleanLines.length / (restImages.length + 1))
    : cleanLines.length;

  let imgIdx = 0;
  for (let i = 0; i < cleanLines.length; i++) {
    md += cleanLines[i] + '\n';
    if (restImages.length > 0 && (i + 1) % chunkSize === 0 && imgIdx < restImages.length) {
      md += `\n![Image](images/${restImages[imgIdx]})\n\n`;
      imgIdx++;
    }
  }
  while (imgIdx < restImages.length) {
    md += `\n![Image](images/${restImages[imgIdx]})\n`;
    imgIdx++;
  }

  fs.writeFileSync(path.join(OUT_DIR, 'article.md'), md, 'utf8');
  console.log(`\nDone!`);
  console.log(`Article: ${OUT_DIR}/article.md`);
  console.log(`Images:  ${IMAGES_DIR}/ (${localImages.length} files, cover: cover.jpg)`);

  ws.close();
}

main().catch(err => {
  console.error('Error:', err.message);
  process.exit(1);
});
```

---

## 注意事项

| 问题 | 原因 | 应对 |
|------|------|------|
| 图片数量比实际少 | 滚动不够，懒加载未触发 | 确保滚到页面最底部，并等待足够时间 |
| 文字连在一起没有换行 | X 用大量 `<span>` 嵌套，`innerText` 层级处理不对 | 用 article 级别的 `innerText` 而非逐节点遍历 |
| 选择器失效 | X 频繁更新 DOM 结构 | 降级到 `document.body.innerText` + 全量 img 标签 |
| 图片下载失败 | X 图片需要合法 Referer 或 UA | 加 `User-Agent: Mozilla/5.0` 请求头 |
| 页面需要登录才能看 | X 部分内容需登录 | 确保 Chrome 已登录 X 账号 |
