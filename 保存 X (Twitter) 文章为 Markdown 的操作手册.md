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

## 第三步：滚动页面，触发懒加载图片 + 截图

X 的文章图片全部懒加载，必须滚动到图片位置才会加载进 DOM。

操作方式：用 `Runtime.evaluate` 注入 JS，从页面顶部开始，每次滚动约 80% 视口高度，每步间隔 300ms，一直滚到页面底部，再滚回顶部。**每滚动一步，用 `Page.captureScreenshot` 截取当前视口**，按顺序保存到 `screenshots/` 目录（`shot-00.jpg`、`shot-01.jpg`……）。

滚动完成后再等待 **1～2 秒**，让图片请求完成。

> **注意**：滚动阶段目标标签页需保持激活状态（前台可见），否则 Chrome 会对后台标签页做渲染节流，懒加载图片可能无法触发。

---

## 第四步：脚本输出原始数据

脚本完成后，在文章目录下输出以下文件，供 AI 在下一步处理：

```
/tmp/{status_id}/
├── .code/
│   └── save-x-article.js   ← 执行脚本（隐藏目录）
├── raw.json                 ← 文字与图片的交替有序序列
├── article.md               ← 由 AI 生成（下一步）
├── images/
│   ├── cover.jpg            ← 封面图（第一张图，只保存不引用）
│   ├── img-01.jpg
│   └── ...
└── screenshots/
    ├── shot-00.jpg          ← 按滚动顺序的页面截图
    ├── shot-01.jpg
    └── ...
```

### raw.json 格式

按 DOM 顺序记录文字块和图片的交替序列，保留图片在正文中的原始位置：

```json
{
  "url": "https://x.com/...",
  "sequence": [
    { "type": "text", "content": "段落文字..." },
    { "type": "image", "file": "cover.jpg" },
    { "type": "text", "content": "继续正文..." },
    { "type": "image", "file": "img-01.jpg" },
    { "type": "text", "content": "更多正文..." }
  ]
}
```

### 图片提取方式（代码执行）

用 placeholder 替换法保留图片位置信息：
1. 找到所有符合条件的 `<img>` 元素，在原位置替换为 `___IMG_N___` 文本占位符
2. 对 article 调用 `innerText`（占位符出现在正确位置）
3. 还原 DOM
4. 按占位符分割文本，得到文字块与图片的交替序列

### 图片过滤规则（规则固定，代码执行）
- `src` 不为空，且不是 `data:` base64 图
- 不含 `profile_images`（头像）
- 不含 `emoji`（表情）
- 宽度大于 80px（排除像素追踪图）
- 对 src 去重

图片 URL 中 `name=small` / `name=medium` 替换为 `name=large` 获取高清版本，下载时带上 `User-Agent` 和 `Referer: https://x.com/` 请求头。

---

## 第五步：AI 处理原始数据，生成 Markdown

AI 读取 `raw.json` 生成 `article.md`。`screenshots/` 目录作为视觉参考资料，AI 可在整个处理过程中按需取用：处理文本时遇到不确定的地方可查截图确认，最后校验格式时也可拿截图与原文对照。不规定具体在哪步参考，由 AI 自行判断。

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
- `raw.json` 的 `sequence` 已按 DOM 顺序记录文字块和图片的交替位置，AI 直接按顺序组装即可
- 封面图（`cover.jpg`）：跳过，不写入 Markdown
- 其余图片（`img-01.jpg` 起）：按 sequence 中的位置插入对应文字块之间

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
const SCREENSHOTS_DIR = path.join(OUT_DIR, 'screenshots');

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

  // 滚动触发懒加载，同时截图
  console.log('Scrolling and taking screenshots...');
  fs.mkdirSync(SCREENSHOTS_DIR, { recursive: true });

  const { result: { value: pageInfo } } = await send('Runtime.evaluate', {
    expression: `({ scrollHeight: document.body.scrollHeight, viewHeight: window.innerHeight })`,
    returnByValue: true,
  });

  const step = Math.floor(pageInfo.viewHeight * 0.8);
  let pos = 0;
  let shotIdx = 0;

  while (pos < pageInfo.scrollHeight) {
    await send('Runtime.evaluate', {
      expression: `window.scrollTo(0, ${pos})`,
      returnByValue: true,
    });
    await sleep(300);

    const shot = await send('Page.captureScreenshot', { format: 'jpeg', quality: 80 });
    const shotFile = path.join(SCREENSHOTS_DIR, `shot-${String(shotIdx).padStart(2, '0')}.jpg`);
    fs.writeFileSync(shotFile, Buffer.from(shot.data, 'base64'));
    shotIdx++;
    pos += step;
  }

  await send('Runtime.evaluate', { expression: `window.scrollTo(0, 0)`, returnByValue: true });
  console.log(`Screenshots: ${shotIdx} captured`);
  await sleep(2000);

  // 提取文字与图片的交替序列（保留图片在正文中的原始位置）
  const { result: { value: extracted } } = await send('Runtime.evaluate', {
    expression: `(() => {
      const article = document.querySelector('main article') || document.body;
      const seen = new Set();
      const saved = [];

      // 找到所有符合条件的图片，替换为占位符
      Array.from(article.querySelectorAll('img')).forEach(img => {
        const src = img.src || '';
        if (!src || src.startsWith('data:')) return;
        if (src.includes('profile_images') || src.includes('emoji')) return;
        if (img.width <= 80) return;
        if (seen.has(src)) return;
        seen.add(src);
        const i = saved.length;
        const ph = document.createElement('span');
        ph.textContent = ' ___IMG_' + i + '___ ';
        img.replaceWith(ph);
        saved.push({ img, ph, url: src });
      });

      // 获取包含占位符的文本
      const text = article.innerText;

      // 还原 DOM
      saved.forEach(({ img, ph }) => ph.replaceWith(img));

      return JSON.stringify({ text, count: saved.length, urls: saved.map(s => s.url) });
    })()`,
    returnByValue: true,
  });

  const { text: rawText, urls: imageUrls } = JSON.parse(extracted);
  console.log(`Found ${imageUrls.length} images`);

  // 下载图片
  fs.mkdirSync(IMAGES_DIR, { recursive: true });
  console.log('Downloading images...');
  const fileMap = {};
  for (let i = 0; i < imageUrls.length; i++) {
    const filename = i === 0 ? 'cover.jpg' : `img-${String(i).padStart(2, '0')}.jpg`;
    const dest = path.join(IMAGES_DIR, filename);
    try {
      await downloadFile(imageUrls[i], dest);
      fileMap[i] = filename;
      console.log(`  Downloaded: ${filename}`);
    } catch (err) {
      console.error(`  Failed: img ${i} — ${err.message}`);
    }
  }

  // 按占位符分割，构建文字与图片的交替序列
  const parts = rawText.split(/\s*___IMG_(\d+)___\s*/);
  const sequence = [];
  for (let i = 0; i < parts.length; i++) {
    if (i % 2 === 0) {
      if (parts[i].trim()) sequence.push({ type: 'text', content: parts[i] });
    } else {
      const idx = parseInt(parts[i]);
      if (fileMap[idx]) sequence.push({ type: 'image', file: fileMap[idx] });
    }
  }

  // 写入 raw.json
  const raw = { url: ARTICLE_URL, sequence };
  fs.writeFileSync(path.join(OUT_DIR, 'raw.json'), JSON.stringify(raw, null, 2), 'utf8');

  console.log(`\nDone!`);
  console.log(`Raw data:    ${OUT_DIR}/raw.json`);
  console.log(`Images:      ${IMAGES_DIR}/ (${Object.keys(fileMap).length} files)`);
  console.log(`Screenshots: ${SCREENSHOTS_DIR}/ (${shotIdx} files)`);
  console.log(`Next: AI reads raw.json (+ screenshots/ as needed) to generate article.md`);

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
