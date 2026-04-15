# CDP 注入 DOM 元素

## 思路

1. 通过 `http://{HOST}/json` 获取所有标签页，找到目标页面
2. 用 WebSocket 连接该标签页的 CDP 调试地址
3. 通过 `Runtime.evaluate` 在页面上下文中执行 JS，操作 DOM

## 定位插入点

```js
// 找到包含目标文字的元素
const all = Array.from(document.querySelectorAll('*'));
const tingEl = all.find(el =>
  Array.from(el.childNodes).some(n => n.nodeType === 3 && n.textContent.includes('听抖音'))
);

// 向上找到带 data-popupid 的外层容器
let container = tingEl;
for (let i = 0; i < 5; i++) {
  container = container.parentElement;
  if (!container) break;
  if (container.hasAttribute('data-popupid')) break;
}
```

## 构造新元素

复用页面已有的 CSS 类，保持视觉风格一致：

```js
const wrapper = document.createElement('div');
wrapper.style.cssText = 'position: relative; color: rgb(255, 255, 255); cursor: pointer;';

const inner = document.createElement('div');
inner.className = 'fR9ZbClg JBKVqbn_'; // 复用"听抖音"的容器类

const iconSpan = document.createElement('span');
iconSpan.setAttribute('role', 'img');
iconSpan.className = 'semi-icon semi-icon-default';
iconSpan.innerHTML = `<svg>...</svg>`;

const label = document.createElement('div');
label.className = 'rWZP7wQY'; // 复用"听抖音"的文字类
label.textContent = '下载';

inner.appendChild(iconSpan);
inner.appendChild(label);
wrapper.appendChild(inner);

// 插入到目标元素之后
container.insertAdjacentElement('afterend', wrapper);
```

## 手写下载图标 SVG

基于 `viewBox="0 0 36 36"` 的 36×36 画布，中心点 `(18, 18)`：

```svg
<svg viewBox="0 0 36 36" fill="none" xmlns="http://www.w3.org/2000/svg"
     width="1em" height="1em" style="font-size:36px;">
  <path d="M18 4a1.5 1.5 0 0 1 1.5 1.5v14.379
           l4.94-4.94a1.5 1.5 0 1 1 2.12 2.122
           l-7.5 7.5a1.5 1.5 0 0 1-2.12 0
           l-7.5-7.5a1.5 1.5 0 1 1 2.12-2.121
           l4.94 4.939V5.5A1.5 1.5 0 0 1 18 4z
           M7 26.5a1.5 1.5 0 0 0 0 3h22a1.5 1.5 0 0 0 0-3H7z"
        fill="currentColor"/>
</svg>
```

**图形构成：**
- 第一段 `path`：从 `x=18` 顶部出发画竖线，末端接 V 形箭头（左右对称）
- `M7 26.5 ... h22`：底部横线，模拟托盘/底座

**关键技巧：**
- 使用 `fill="currentColor"` 继承父元素颜色，无需硬编码颜色值
- `font-size` 控制实际渲染尺寸，`width/height` 用 `1em` 自动跟随

## 防重复插入

```js
if (document.getElementById('__download_btn__')) return 'already inserted';
```

给注入的元素加一个唯一 `id`，避免多次执行脚本时重复插入。
