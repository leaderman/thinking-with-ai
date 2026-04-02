# OpenRouter 图片生成 - 参考图用法调研

## 结论

`google/gemini-3.1-flash-image-preview` 支持通过多模态 content 数组传入参考图，SDK 原生支持此用法，经实测可正常生成图片。

---

## 一、参考图的传入方式

### HTTP API（curl）

将 `content` 从字符串改为数组，text 放在 image 前面：

```bash
curl https://openrouter.ai/api/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENROUTER_API_KEY" \
  -d '{
    "model": "google/gemini-3.1-flash-image-preview",
    "messages": [
      {
        "role": "user",
        "content": [
          {
            "type": "text",
            "text": "你的提示词"
          },
          {
            "type": "image_url",
            "image_url": {
              "url": "https://your-reference-image.jpg"
            }
          }
        ]
      }
    ],
    "modalities": ["image", "text"]
  }'
```

### @openrouter/sdk

SDK 内部使用驼峰命名，`imageUrl` 而非 `image_url`（序列化时自动转换）：

```js
content: [
  {
    type: "text",
    text: "你的提示词",
  },
  {
    type: "image_url",
    imageUrl: {
      url: process.env.REFERENCE_IMAGE_URL,
    },
  },
]
```

图片支持公开 URL 或 base64：`"url": "data:image/jpeg;base64,xxx"`

支持格式：`image/png`、`image/jpeg`、`image/webp`、`image/gif`

---

## 二、SDK 坑点（来自 gen_image.mjs 调研）

| 问题 | 错误写法 | 正确写法 |
|---|---|---|
| import | `import OpenRouter from "@openrouter/sdk"` | `import { OpenRouter, HTTPClient } from "@openrouter/sdk"` |
| 参数结构 | 直接平铺 `model`、`messages` | 包在 `chatGenerationParams` 字段里 |
| 响应图片字段 | `image.image_url.url` | `image.imageUrl.url` |
| 响应路径 | — | `result.choices[0].message.images[0].imageUrl.url` |

### Node.js 代理问题

Node.js 18+ 内置 fetch 不读取 `HTTPS_PROXY` 环境变量，需用 `undici` 手动注入：

```js
import { ProxyAgent, fetch as undiciFetch } from "undici";
import { OpenRouter, HTTPClient } from "@openrouter/sdk";

const proxyAgent = new ProxyAgent(process.env.HTTPS_PROXY);
const httpClient = new HTTPClient({
  fetcher: (req) => undiciFetch(req.url, {
    method: req.method,
    headers: Object.fromEntries(req.headers.entries()),
    body: req.body,
    duplex: "half",          // 发送 body 必须加，否则报错
    dispatcher: proxyAgent,
  }),
});
```

注意：`setGlobalDispatcher` 无效，因为安装的 `undici` 与 Node 内置 fetch 使用的是不同实例。

---

## 三、费用查询

生成图片的费用无法从 `chat.send` 的响应 `usage` 字段直接拿到（只有 token 数，无 cost），需通过 generation 接口单独查询：

```js
// 需等待几秒让数据入库，3 秒不够，建议 5 秒以上
await new Promise(resolve => setTimeout(resolve, 5000));
const gen = await openrouter.generations.getGeneration({ id: result.id });
console.log("总费用(USD):", gen.data.totalCost);
console.log("图片tokens:", gen.data.nativeTokensCompletionImages);
```

### 定价（google/gemini-3.1-flash-image-preview）

| 费用项 | 单价 |
|---|---|
| Input tokens | $0.50 / M |
| Output tokens（文本） | $3 / M |
| Output tokens（图片） | $60 / M |

图片生成 token 单价是文本输出的 20 倍，是费用大头。

---

## 四、最终可运行代码

**环境准备：**

```bash
npm install @openrouter/sdk undici
```

**`gen_image_with_ref.mjs`：**

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
        content: [
          {
            type: "text",
            text: "你的提示词",
          },
          {
            type: "image_url",
            imageUrl: {
              url: process.env.REFERENCE_IMAGE_URL,
            },
          },
        ],
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

// 查询费用（需等待数据入库）
await new Promise(resolve => setTimeout(resolve, 5000));
const gen = await openrouter.generations.getGeneration({ id: result.id });
console.log("总费用(USD):", gen.data.totalCost);
```

**运行：**

```bash
OPENROUTER_API_KEY=你的key HTTPS_PROXY=http://127.0.0.1:1234 REFERENCE_IMAGE_URL=https://your-image.jpg node gen_image_with_ref.mjs
```

---

## 五、注意事项

- text 放在 image 前面（OpenRouter 官方建议）
- 多张参考图可放多个 `image_url` 条目，上限因模型而异
- `super_resolution_references` 仅支持 Sourceful 模型，不适用于 Gemini

## 参考文档

- [OpenRouter Image Inputs](https://openrouter.ai/docs/guides/overview/multimodal/images)
- [OpenRouter Image Generation](https://openrouter.ai/docs/guides/overview/multimodal/image-generation)
- [google/gemini-3.1-flash-image-preview](https://openrouter.ai/google/gemini-3.1-flash-image-preview)
