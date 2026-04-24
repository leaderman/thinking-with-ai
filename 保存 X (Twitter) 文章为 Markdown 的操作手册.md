# 保存 X (Twitter) 文章为 Markdown 的操作手册

## 分工原则

- **代码**：只处理规则固定的机械操作（连接 CDP、导航、滚动、下载图片、写文件）
- **AI**：处理所有需要语义理解的工作（识别标题/正文、清理噪声、处理特殊字符、生成 Markdown）

---

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

> **注意**：滚动阶段目标标签页需保持激活状态（前台可见），否则 Chrome 会对后台标签页做渲染节流，懒加载图片可能无法触发。

---

## 第四步：脚本输出原始数据

脚本完成后，在文章目录下输出以下文件，供 AI 在下一步处理：

```
/tmp/{status_id}/
├── .code/
│   └── save-x-article.js   ← 执行脚本（隐藏目录）
├── raw.txt                  ← 页面原始文本（article.innerText，未经任何处理）
├── images.json              ← 图片 URL 列表及本地文件名映射
├── article.md               ← 由 AI 生成（下一步）
└── images/
    ├── cover.jpg            ← 封面图（第一张图，只保存不引用）
    ├── img-01.jpg
    ├── img-02.jpg
    └── ...
```

- `raw.txt`：直接写入 `article.innerText`，不做任何过滤或格式化
- `images.json`：格式为 `[{ "url": "...", "file": "cover.jpg" }, ...]`
- 图片 URL 中 `name=small` / `name=medium` 替换为 `name=large` 获取高清版本
- 下载时带上 `User-Agent` 和 `Referer: https://x.com/` 请求头

### 图片过滤规则（代码执行，规则固定）
- `src` 不为空，且不是 `data:` base64 图
- 不含 `profile_images`（头像）
- 不含 `emoji`（表情）
- 宽度大于 80px（排除像素追踪图）
- 对 src 去重

---

## 第五步：AI 处理原始数据，生成 Markdown

AI 读取 `raw.txt` 和 `images.json`，生成 `article.md`。

### AI 负责的工作

**识别元数据**
- 标题：从正文语义中判断（不依赖固定选择器）
- 作者：识别 `@handle` 格式
- 发布时间：识别时间戳格式

**清理噪声**
由 AI 语义判断，而非固定规则。典型噪声包括：
- 点赞数、转发数、浏览量等统计数字行
- X UI 按钮文字（Reply、Repost、Like、Follow 等）
- 页面底部的推广内容（Want to publish、Upgrade to Premium 等）
- 时间戳行、分隔符行

**处理特殊字符**
- 识别正文中可能与 Markdown 语法冲突的字符（`$`、`*`、`_` 等），按语义决定是否转义
- 代码块内容保持原样，不转义

**图片插入**
- 封面图（`cover.jpg`）：不写入 Markdown，仅保存文件
- 其余图片（`img-01.jpg` 起）：根据正文语义判断合适的插入位置

### 输出格式

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

---

## 第六步：保存结果校验

AI 生成 `article.md` 后，检查以下项目：

### 标题
- 是否为文章实际标题（非作者名、非 `(1) AuthorName` 等）
- 是否完整，未被截断

### 正文
- 是否有实质内容（非空）
- 图片引用格式是否正确（`![Image](images/img-xx.jpg)`）
- 图片文件是否实际存在于 `images/` 目录

### 末尾
- 确认没有 X UI 噪声残留

### 封面图
- `images/cover.jpg` 是否存在且文件大小正常（非 0 字节）

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

若脚本不存在或校验失败，则从手册末尾的代码块重新写入。

---

## 脚本代码

脚本只负责机械操作：导航、滚动、提取原始文本、下载图片、写文件。不做任何语义处理。

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

  // 导航到文章
  console.log('Navigating to article...');
  await send('Page.navigate', { url: ARTICLE_URL });
  await waitEvent('Page.loadEventFired');
  console.log('Page loaded, waiting for React render...');
  await sleep(4000);

  // 滚动触发懒加载
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

  // 提取原始文本（不做任何处理）
  const { result: { value: rawText } } = await send('Runtime.evaluate', {
    expression: `(() => {
      const el = document.querySelector('main article');
      return el ? el.innerText : document.body.innerText;
    })()`,
    returnByValue: true,
  });

  // 提取图片 URL
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

  // 下载图片
  fs.mkdirSync(IMAGES_DIR, { recursive: true });
  console.log('Downloading images...');
  const imageMap = [];
  for (let i = 0; i < imageUrls.length; i++) {
    const filename = i === 0 ? 'cover.jpg' : `img-${String(i).padStart(2, '0')}.jpg`;
    const dest = path.join(IMAGES_DIR, filename);
    try {
      await downloadFile(imageUrls[i], dest);
      imageMap.push({ url: imageUrls[i], file: filename });
      console.log(`  Downloaded: ${filename}`);
    } catch (err) {
      console.error(`  Failed: ${filename} — ${err.message}`);
    }
  }

  // 写入原始数据文件
  fs.writeFileSync(path.join(OUT_DIR, 'raw.txt'), rawText || '', 'utf8');
  fs.writeFileSync(path.join(OUT_DIR, 'images.json'), JSON.stringify(imageMap, null, 2), 'utf8');

  console.log(`\nDone!`);
  console.log(`Raw text: ${OUT_DIR}/raw.txt`);
  console.log(`Images:   ${IMAGES_DIR}/ (${imageMap.length} files)`);
  console.log(`Next: AI reads raw.txt and images.json to generate article.md`);

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
| 图片数量比实际少 | 滚动阶段标签页不在前台，懒加载未触发 | 确保目标标签页保持激活，滚到页面最底部后再等待 |
| 选择器失效 | X 频繁更新 DOM 结构 | 降级到 `document.body.innerText` + 全量 img 标签 |
| 图片下载失败 | X 图片需要合法 Referer 或 UA | 已在脚本中加入 `User-Agent` 和 `Referer` 请求头 |
| 页面需要登录才能看 | X 部分内容需登录 | 确保 Chrome 已登录 X 账号 |
