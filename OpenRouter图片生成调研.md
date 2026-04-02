# OpenRouter 图片生成 API 调研记录

## 目标

使用 OpenRouter SDK 调用 `google/gemini-3.1-flash-image-preview` 模型生成图片，并将图片保存到本地文件。

---

## 核心结论

### 1. 返回的图片是 base64 data URL，不是 HTTP 链接

响应中的图片以 data URL 形式内嵌，格式为：

```
data:image/jpeg;base64,/9j/4AAQ...
```

格式规律：`data:<mimeType>;base64,<data>`，从中可以直接解析出图片格式（jpeg/png/webp 等）。

---

### 2. SDK 的正确 import 方式

官方示例使用默认导出，实际是具名导出：

```js
// 错误（官方示例有误）
import OpenRouter from "@openrouter/sdk";

// 正确
import { OpenRouter, HTTPClient } from "@openrouter/sdk";
```

---

### 3. SDK 的参数结构

官方示例中参数直接平铺，实际需要包在 `chatGenerationParams` 字段里：

```js
// 错误（官方示例有误）
await openrouter.chat.send({
  model: "...",
  messages: [...],
  modalities: [...]
});

// 正确
await openrouter.chat.send({
  chatGenerationParams: {
    model: "...",
    messages: [...],
    modalities: [...]
  }
});
```

---

### 4. 响应的图片字段是驼峰命名

SDK 返回的图片对象字段是 `imageUrl`，不是 `image_url`：

```js
// 错误
image.image_url.url

// 正确
image.imageUrl.url
```

响应路径：`result.choices[0].message.images[0].imageUrl.url`

---

### 5. Node.js 内置 fetch 不支持 HTTP_PROXY 环境变量

Node.js 18+ 内置的 `fetch` 基于 undici，但**不会自动读取 `HTTP_PROXY`/`HTTPS_PROXY` 环境变量**。

解决方案：安装 `undici`，用其 `fetch` + `ProxyAgent` 替换 SDK 的默认 fetcher，并通过 `HTTPClient` 注入。

注意事项：
- `setGlobalDispatcher` 无效，因为安装的 `undici` 包与 Node 内置 fetch 使用的是不同的 undici 实例
- undici fetch 不接受 `Request` 对象，需手动解构为 `url`、`method`、`headers`、`body`
- 发送 body 时必须加 `duplex: "half"`，否则报错

---

## 最终可运行代码

**环境准备（项目目录内执行一次）：**

```bash
npm install @openrouter/sdk undici
```

**`gen_image.mjs`：**

```js
import { ProxyAgent, fetch as undiciFetch } from "undici";
import { OpenRouter, HTTPClient } from "@openrouter/sdk";
import fs from "fs";

const proxyAgent = new ProxyAgent(process.env.HTTPS_PROXY);
const httpClient = new HTTPClient({
  fetcher: (req) => undiciFetch(req.url, {
    method: req.method,
    headers: Object.fromEntries(req.headers.entries()),
    body: req.body,
    duplex: "half",
    dispatcher: proxyAgent,
  }),
});

const openrouter = new OpenRouter({
  apiKey: process.env.OPENROUTER_API_KEY,
  httpClient,
});

const result = await openrouter.chat.send({
  chatGenerationParams: {
    model: "google/gemini-3.1-flash-image-preview",
    messages: [
      {
        role: "user",
        content: "Generate a beautiful sunset over mountains",
      },
    ],
    modalities: ["image", "text"],
  },
});

const image = result.choices[0].message.images[0];
const dataUrl = image.imageUrl.url;
const [meta, base64Data] = dataUrl.split(",");
const ext = meta.match(/data:image\/([^;]+)/)[1];
const outputPath = `/tmp/${Date.now()}.${ext}`;
fs.writeFileSync(outputPath, Buffer.from(base64Data, "base64"));
console.log(`Saved to ${outputPath}`);
```

**运行方式：**

```bash
OPENROUTER_API_KEY=你的key HTTPS_PROXY=http://127.0.0.1:1234 node gen_image.mjs
```

---

## 带结构化提示词的完整代码

将 content 替换为结构化 JSON 提示词，用于生成特定场景的图片：

```js
import { ProxyAgent, fetch as undiciFetch } from "undici";
import { OpenRouter, HTTPClient } from "@openrouter/sdk";
import fs from "fs";

const proxyAgent = new ProxyAgent(process.env.HTTPS_PROXY);
const httpClient = new HTTPClient({
  fetcher: (req) => undiciFetch(req.url, {
    method: req.method,
    headers: Object.fromEntries(req.headers.entries()),
    body: req.body,
    duplex: "half",
    dispatcher: proxyAgent,
  }),
});

const openrouter = new OpenRouter({
  apiKey: process.env.OPENROUTER_API_KEY,
  httpClient,
});

const result = await openrouter.chat.send({
  chatGenerationParams: {
    model: "google/gemini-3.1-flash-image-preview",
    messages: [
      {
        role: "user",
        content: JSON.stringify({
          type: "image_prompt",
          style: {
            genre: "纪实生活摄影",
            aesthetic: "大胆，略带复古感",
            contrast: "高对比度",
            resolution: "8K",
            detail: "高细节织物纹理",
            aspect_ratio: "9:16",
          },
          subject: {
            identity: "{argument name=\"celebrity identity\" default=\"Millie Bobby Brown\"}",
            appearance: {
              hair: "长发，波浪卷，略微偏分",
              expression: "自信，强势，直视镜头",
              makeup: {
                lipstick: "浓郁红色",
                eyebrows: "修饰过的眉形",
                eyeliner: "利落眼线",
                mascara: "根根分明",
              },
              accessories: ["小巧金色圆环耳饰"],
            },
            pose: {
              position: "坐在地板上",
              posture: "不对称姿势，向后倾斜",
              legs: {
                right_leg: "弯曲，膝盖抬起",
                left_leg: "靠近地面平放",
              },
              support: "左臂支撑身体，手掌撑在身后地板上",
              head: "头部略微后仰",
              gaze_direction: "向下看向镜头",
            },
            clothing: {
              outfit: "短款亮白色挂脖围裙",
              details: {
                waist: "系有蝴蝶结形状的布艺腰带",
                chest_patch: "右胸上方有写着 'Giulia' 字样的手写体小徽章",
              },
              footwear: "深色尖头高跟靴",
              additional_clothing: "无可见其他衣物",
            },
          },
          environment: {
            location: "夜晚的现代厨房",
            flooring: "浅色木质复合地板",
            cabinets: "浅色木质复合橱柜",
            background: {
              window: "黑暗，未开灯",
              plant: "大型盆栽龟背竹，叶片宽大翠绿",
            },
          },
          lighting: {
            type: "强烈的正面直射闪光灯",
            effect: "强高对比度光效，在地板和后墙上投下锐利的深色阴影",
          },
          camera: {
            angle: "低角度，贴近地面",
            tilt: "略微仰拍",
          },
        }),
      },
    ],
    modalities: ["image", "text"],
  },
});

const image = result.choices[0].message.images[0];
const dataUrl = image.imageUrl.url;
const [meta, base64Data] = dataUrl.split(",");
const ext = meta.match(/data:image\/([^;]+)/)[1];
const outputPath = `/tmp/${Date.now()}.${ext}`;
fs.writeFileSync(outputPath, Buffer.from(base64Data, "base64"));
console.log(`Saved to ${outputPath}`);
```

---

## 替换 identity 为中国女性描述的版本

将 `identity` 从名人参数改为通用的人物描述，避免名人相关限制：

> 仅 `identity` 字段有变化，其余内容与上一版本相同。

```js
import { ProxyAgent, fetch as undiciFetch } from "undici";
import { OpenRouter, HTTPClient } from "@openrouter/sdk";
import fs from "fs";

const proxyAgent = new ProxyAgent(process.env.HTTPS_PROXY);
const httpClient = new HTTPClient({
  fetcher: (req) => undiciFetch(req.url, {
    method: req.method,
    headers: Object.fromEntries(req.headers.entries()),
    body: req.body,
    duplex: "half",
    dispatcher: proxyAgent,
  }),
});

const openrouter = new OpenRouter({
  apiKey: process.env.OPENROUTER_API_KEY,
  httpClient,
});

const result = await openrouter.chat.send({
  chatGenerationParams: {
    model: "google/gemini-3.1-flash-image-preview",
    messages: [
      {
        role: "user",
        content: JSON.stringify({
          type: "image_prompt",
          style: {
            genre: "纪实生活摄影",
            aesthetic: "大胆，略带复古感",
            contrast: "高对比度",
            resolution: "8K",
            detail: "高细节织物纹理",
            aspect_ratio: "9:16",
          },
          subject: {
            identity: "a young Chinese woman, East Asian facial features",
            appearance: {
              hair: "长发，波浪卷，略微偏分",
              expression: "自信，强势，直视镜头",
              makeup: {
                lipstick: "浓郁红色",
                eyebrows: "修饰过的眉形",
                eyeliner: "利落眼线",
                mascara: "根根分明",
              },
              accessories: ["小巧金色圆环耳饰"],
            },
            pose: {
              position: "坐在地板上",
              posture: "不对称姿势，向后倾斜",
              legs: {
                right_leg: "弯曲，膝盖抬起",
                left_leg: "靠近地面平放",
              },
              support: "左臂支撑身体，手掌撑在身后地板上",
              head: "头部略微后仰",
              gaze_direction: "向下看向镜头",
            },
            clothing: {
              outfit: "短款亮白色挂脖围裙",
              details: {
                waist: "系有蝴蝶结形状的布艺腰带",
                chest_patch: "右胸上方有写着 'Giulia' 字样的手写体小徽章",
              },
              footwear: "深色尖头高跟靴",
              additional_clothing: "无可见其他衣物",
            },
          },
          environment: {
            location: "夜晚的现代厨房",
            flooring: "浅色木质复合地板",
            cabinets: "浅色木质复合橱柜",
            background: {
              window: "黑暗，未开灯",
              plant: "大型盆栽龟背竹，叶片宽大翠绿",
            },
          },
          lighting: {
            type: "强烈的正面直射闪光灯",
            effect: "强高对比度光效，在地板和后墙上投下锐利的深色阴影",
          },
          camera: {
            angle: "低角度，贴近地面",
            tilt: "略微仰拍",
          },
        }),
      },
    ],
    modalities: ["image", "text"],
  },
});

const image = result.choices[0].message.images[0];
const dataUrl = image.imageUrl.url;
const [meta, base64Data] = dataUrl.split(",");
const ext = meta.match(/data:image\/([^;]+)/)[1];
const outputPath = `/tmp/${Date.now()}.${ext}`;
fs.writeFileSync(outputPath, Buffer.from(base64Data, "base64"));
console.log(`Saved to ${outputPath}`);
```
