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
├── article.md
└── images/
    ├── cover.jpg       ← 封面图（第一张图）
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

![Cover](images/cover.jpg)

（正文段落）

![Image](images/img-01.jpg)

（继续正文）
...
```

**图片插入位置**：由于 X 的 DOM 结构复杂，图片在正文中的精确位置难以稳定获取。
实用策略：第一张图作为封面（`cover.jpg`）放在标题下方，其余图片（`img-01.jpg` 起）**按照它们在 DOM 中出现的顺序**，穿插在对应段落附近插入。

**噪声清理**：`innerText` 会包含点赞数、转发数、浏览量等 UI 文字（如 "28 79 795 372K"），以及重复的标题行，需要识别并去掉。判断依据：
- 纯数字行或带 K/M 后缀的数字行
- 与标题完全重复的行
- 作者名和 @handle 独占一行（已在 header 里出现过）

---

## 注意事项

| 问题 | 原因 | 应对 |
|------|------|------|
| 图片数量比实际少 | 滚动不够，懒加载未触发 | 确保滚到页面最底部，并等待足够时间 |
| 文字连在一起没有换行 | X 用大量 `<span>` 嵌套，`innerText` 层级处理不对 | 用 article 级别的 `innerText` 而非逐节点遍历 |
| 选择器失效 | X 频繁更新 DOM 结构 | 降级到 `document.body.innerText` + 全量 img 标签 |
| 图片下载失败 | X 图片需要合法 Referer 或 UA | 加 `User-Agent: Mozilla/5.0` 请求头 |
| 页面需要登录才能看 | X 部分内容需登录 | 确保 Chrome 已登录 X 账号 |
