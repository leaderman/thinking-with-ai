# 微信公众号 Markdown 渲染器实现思路

## 目标

输入 Markdown 字符串，输出微信公众号兼容的 HTML，可直接提交给微信草稿 API。

---

## 依赖

- `marked` — Markdown 转 HTML
- `highlight.js` — 代码块语法高亮

---

## 核心步骤

### 第一步：定义样式

参考 [doocs/md](https://github.com/doocs/md) 项目 `default.css`，把每种元素的样式整理成 JS 对象：

```js
const styles = {
  h1: 'font-size:24px;font-weight:bold;text-align:center;...',
  h2: 'display:table;color:#fff;background:#3f9dff;...',
  p:  'margin:1.5em 8px;letter-spacing:0.1em;...',
  // ...
}
```

### 第二步：自定义 renderer

重写 marked 每个元素的输出，把样式直接写进 `style` 属性：

```js
const renderer = {
  heading(text, level) {
    return `<h${level} style="${styles[`h${level}`]}">${text}</h${level}>`
  },
  paragraph(text) {
    return `<p style="${styles.p}">${text}</p>`
  },
  blockquote(text) {
    return `<blockquote style="${styles.blockquote}">${text}</blockquote>`
  },
  code(code, lang) {
    // 用 highlight.js 做语法高亮，输出内联 style 而非 class
  },
  // table、list、listitem、link、image、strong、em ...
}

marked.use({ renderer })
```

### 第三步：代码高亮内联化

`highlight.js` 默认输出带 class 的 HTML，微信会剥掉 class。需要把高亮结果的 class 转成内联 `style="color:#xxx"`，确保代码颜色在微信里正常显示。

### 第四步：渲染

```js
const html = marked.parse(markdown)
```

### 第五步：提交微信草稿 API

```js
fetch('https://api.weixin.qq.com/cgi-bin/draft/add?access_token=...', {
  method: 'POST',
  body: JSON.stringify({
    articles: [{ title, content: html, thumb_media_id }]
  })
})
```

---

## 项目结构

```
my-md-renderer/
├── src/
│   ├── styles.js     # 各元素的内联样式定义
│   ├── renderer.js   # 自定义 marked renderer
│   └── index.js      # 对外暴露 render(markdown) 函数
├── package.json
└── test.js           # 测试脚本
```
