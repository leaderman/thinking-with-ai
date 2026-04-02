# YouMind Nano Banana Pro Prompts 调研

来源：https://youmind.com/zh-CN/nano-banana-pro-prompts

## 数据获取方式

页面首屏数据通过 SSR 渲染在 HTML 中，滚动加载时触发分页 API。

**端点：** `POST https://youmind.com/youhome-api/prompts`

**请求体：**
```json
{
  "model": "nano-banana-pro",
  "page": 1,
  "limit": 18,
  "locale": "zh-CN",
  "campaign": "nano-banana-pro-prompts",
  "filterMode": "imageCategories"
}
```

**响应结构（每个 prompt）：**

| 字段 | 说明 |
|---|---|
| `id` | 唯一 ID |
| `title` | 卡片标题 |
| `content` | 完整 prompt，含 `{argument name="..." default="..."}` 参数定义 |
| `description` | 描述文字 |
| `author` | `{name, link}` - prompt 作者（X 账号） |
| `media` | 示例图片 URL 数组 |
| `mediaThumbnails` | 缩略图 URL 数组 |
| `featured` | 是否精选 |
| `sourceLink` | 原始 X 推文链接 |
| `likes` | 点赞数 |

总计约 12,222 条 prompt，通过 `page` 参数分页获取。

## 案例：名人金句卡（ID: 151）

**标题：** 带肖像和中英文定制的宽引言卡

**content：**
```
一张宽幅的名人金句卡，棕色背景，衬线体浅金色"{argument name="金句" default="保持饥饿，保持愚蠢"}"，小字"——{argument name="作者" default="Steve Jobs"}"，文字前面带一个大的淡淡的引号。人物头像在左边，文字在右边，文字占画面比例 2/3，人物占 1/3，人物有一点渐变过渡的感觉。
```

**参数（从 content 中解析）：**
- `金句`，默认值：保持饥饿，保持愚蠢
- `作者`，默认值：Steve Jobs

## 注意点

- **需要 cookie**：接口依赖浏览器 cookie，直接裸调返回空数据，须在已登录的页面上下文中发起请求
- **参数嵌在 content 里**：没有独立的 arguments 字段，需用正则解析 `{argument name="..." default="..."}`
- **人物头像不在接口里**：`author` 只有名字和 X 链接，头像需从 X 账号另行获取
