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

总计约 12,222 条 prompt，通过 `page` 参数分页获取。`limit` 最大值为 **100**，超过会被截断至 100。

**排序说明：** 不需要传排序参数，API 默认已按时间倒序返回。第 1 页会掺入精选 prompt（featured=true，日期不连续），从第 2 页起为纯粹的时间倒序数据。页面上的"按时间排序"选项仅影响 SSR 首屏渲染，对分页 API 无效。

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
- **translatedContent**：content 的英文翻译版本，可用于英文场景

## 接口调用示例

**请求**

```
POST https://youmind.com/youhome-api/prompts
Content-Type: application/json
```

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

**响应**

```json
{
  "prompts": [
    {
      "id": 151,
      "title": "带肖像和中英文定制的宽引言卡",
      "description": "一个用于生成宽幅引言卡的提示，卡片上有一位名人的肖像，背景为棕色，引言文字为浅金色衬线字体。布局为文字占据三分之二，人物占据三分之一。引言文字和作者可参数化以便重复使用。",
      "sourceLink": "https://x.com/stark_nico99/status/1991718646570426763",
      "sourcePublishedAt": "2025-11-21T04:01:44.000Z",
      "author": {
        "name": "Nicolechan",
        "link": "https://x.com/stark_nico99"
      },
      "content": "一张宽幅的名人金句卡，棕色背景，衬线体浅金色“{argument name=\"金句\" default=\"保持饥饿，保持愚蠢\"}”，小字“——{argument name=\"作者\" default=\"Steve Jobs\"}”，文字前面带一个大的淡淡的引号。人物头像在左边，文字在右边，文字占画面比例 2/3，人物占 1/3，人物有一点渐变过渡的感觉。",
      "media": [
        "https://cms-assets.youmind.com/media/1763886933714_5zqn1e_G6QBjQHbgAE3Yt_.jpg",
        "https://cms-assets.youmind.com/media/1763886938314_wbcfc7_G6QBiiracAInQ8z.jpg",
        "https://cms-assets.youmind.com/media/1763886941069_1d9ace_G6QBii_acAIRxKd.jpg",
        "https://cms-assets.youmind.com/media/1763886946388_nwahev_G6QBikOaEAAmYkO.jpg"
      ],
      "mediaThumbnails": [
        "https://cms-assets.youmind.com/media/1763886933714_5zqn1e_G6QBjQHbgAE3Yt_-300x168.jpg",
        "https://cms-assets.youmind.com/media/1763886938314_wbcfc7_G6QBiiracAInQ8z-300x168.jpg",
        "https://cms-assets.youmind.com/media/1763886941069_1d9ace_G6QBii_acAIRxKd-300x127.jpg",
        "https://cms-assets.youmind.com/media/1763886946388_nwahev_G6QBikOaEAAmYkO-300x166.jpg"
      ],
      "language": "zh",
      "translatedContent": "一张宽幅引言卡片，上面印有一位名人，背景为棕色，引言文字为浅金色衬线字体：“{argument name=\"famous_quote\" default=\"Stay Hungry, Stay Foolish\"}”，下方是较小的文字：“—{argument name=\"author\" default=\"Steve Jobs\"}”。文字前面有一个大而柔和的引号。人物肖像在左侧，文字在右侧。文字占据图片的三分之二，肖像占据三分之一，肖像部分带有轻微的渐变过渡效果。",
      "sourcePlatform": "twitter",
      "featured": true,
      "sort": 1,
      "searchIndex": "带肖像和中英文定制的宽引言卡 一个用于生成宽幅引言卡的提示，卡片上有一位名人的肖像，背景为棕色，引言文字为浅金色衬线字体。布局为文字占据三分之二，人物占据三分之一。引言文字和作者可参数化以便重复使用。 一张宽幅的名人金句卡，棕色背景，衬线体浅金色“{argument name=\"金句\" default=\"保持饥饿，保持愚蠢\"}”，小字“——{argument name=\"作者\" default=\"steve jobs\"}”，文字前面带一个大的淡淡的引号。人物头像在左边，文字在右边，文字占画面比例 2/3，人物占 1/3，人物有一点渐变过渡的感觉。 一张宽幅引言卡片，上面印有一位名人，背景为棕色，引言文字为浅金色衬线字体：“{argument name=\"famous_quote\" default=\"stay hungry, stay foolish\"}”，下方是较小的文字：“—{argument name=\"author\" default=\"steve jobs\"}”。文字前面有一个大而柔和的引号。人物肖像在左侧，文字在右侧。文字占据图片的三分之二，肖像占据三分之一，肖像部分带有轻微的渐变过渡效果。 nicolechan 151",
      "likes": 1,
      "resultsCount": 4,
      "needReferenceImages": true,
      "promptCategories": []
    },
    {
      "id": 6847,
      "title": "高级液态玻璃 Bento 网格产品信息图，含 8 个模块",
      "description": "使用 bento grid 8 模块布局创建信息图，用户可以指定食品、药品、科技等类别中的任何产品名称，选择语言、背景样式和主网格样式。",
      "sourceLink": "https://x.com/MansiSanghani1/status/2013550795224961492",
      "sourcePublishedAt": "2026-01-20T11:04:00.000Z",
      "author": {
        "name": "Mansi Sanghani",
        "link": "https://x.com/MansiSanghani1"
      },
      "content": "Input Variable: [insert product name]\nLanguage: [insert language]\n\nSystem Instruction:\nCreate an image of premium liquid glass Bento grid product infographic with 8 modules (card 2 to 8 show text titles only).\n1) Product Analysis:\n→ Identify product's dominant natural color → \"hero color\"\n→ Identify category: FOOD / MEDICINE / TECH\n2) Color Palette (derived from hero):\n→ Product + accents: full saturation hero color\n→ Icons, borders: muted hero (30-40% saturation, never black)\n3) Visual Style:\n→ Hero product: real photography (authentic, premium), 3D Glass version [choose one]\n→ Cards: Apple liquid glass (85-90% transparent) with Whisper-thin borders and Subtle drop shadow for floating depth and reflecting the background color\n→ Background stays behind cards and high blur where cards are [choose one]:\n  - Ethereal: product essence, light caustics, abstract glow\n  - Macro: product texture close-up, heavily blurred\n  - Pattern: product repeated softly at 10-15% opacity\n  - Context: relevant environment, blurred + desaturated\n→ Add subtle motion effect\n→ Asymmetric Bento grid, 16:9 landscape\n→ Hero card: 28-30% | Info modules: 70-72%\n4) Module Content (8 Cards):\nM1 — Hero: Product displayed as real photo / 3D glass / stylized interpretation (choose one)in beautiful form + product name label\nM2 — Core Benefits: 4 unique benefits + hero-color icons\nM3 — How to Use: 4 usage methods + icons\nM4 — Key Metrics: 5 EXACT data points\nFormat: [icon] [Label] [Bold Value] [Unit]\nFOOD: Calories: [X] kcal/100g, Carbs: [X]g (fiber [X]g, sugar [X]g), Protein: [X]g, [Key Vitamin]: [X]mg ([X]% DV), [Key Mineral]: [X]mg ([X]% DV)\nMEDICINE:Active: [name], Strength: [X] mg, Onset: [X] min, Duration: [X] hrs, Half-life: [X] hrs \nTECH:Chip: [model], Battery: [X] hrs, Weight: [X]g,[Key spec]: [value], Connectivity: [protocols]\nM5 — Who It's For: 4 recommended groups with green checkmark icons | 3 caution groups with amber warning icons\nM6 — Important Notes: 4 precautions + warning icons\nM7 — Quick Reference:\n→ FOOD: Glycemic Index + dietary tags with icons\n→ MEDICINE: Side effects + severity with icons\n→ TECH: Compatibility + certifications with icons\nM8 — Did You Know: 3 facts (origin, science, global stat) + icons\nOutput: 1 image, 16:9 landscape, ultra-premium liquid glass infographic.",
      "media": [
        "https://cms-assets.youmind.com/media/1768962051381_l9uih4_537980579-6f29d32a-c786-40c4-bd5a-79c640737496.png",
        "https://cms-assets.youmind.com/media/1768962076321_nu4c5q_537981099-d18d0e38-f7ac-4781-a5da-6d68e2380885.png"
      ],
      "mediaThumbnails": [
        "https://cms-assets.youmind.com/media/1768962051381_l9uih4_537980579-6f29d32a-c786-40c4-bd5a-79c640737496-300x167.png",
        "https://cms-assets.youmind.com/media/1768962076321_nu4c5q_537981099-d18d0e38-f7ac-4781-a5da-6d68e2380885-300x167.png"
      ],
      "language": "en",
      "translatedContent": "输入变量：[插入产品名称]\n语言：[插入语言]\n\n系统指令：\n创建一个包含 8 个模块（卡片 2 到 8 只显示文本标题）的优质液态玻璃 Bento 网格产品信息图。\n1) 产品分析：\n→ 识别产品的主导自然色 → “主色调”\n→ 识别类别：食品 / 药品 / 科技\n2) 调色板（源自主色调）：\n→ 产品 + 强调色：完全饱和的主色调\n→ 图标、边框：柔和的主色调（30-40% 饱和度，绝不使用黑色）\n3) 视觉风格：\n→ 主打产品：真实摄影（真实、优质）、3D 玻璃版本 [二选一]\n→ 卡片：Apple 液态玻璃（85-90% 透明），带有极细边框和微妙的阴影，营造浮动深度并反射背景颜色\n→ 背景保持在卡片后面，卡片所在区域高度模糊 [二选一]：\n  - 飘渺：产品精髓、柔和的焦散、抽象的光晕\n  - 微距：产品纹理特写、高度模糊\n  - 图案：产品柔和重复，不透明度为 10-15%\n  - 情境：相关环境，模糊 + 去饱和\n→ 添加微妙的动态效果\n→ 不对称 Bento 网格，16:9 横向\n→ 主卡片：28-30% | 信息模块：70-72%\n4) 模块内容（8 张卡片）：\nM1 — 主打：以真实照片 / 3D 玻璃 / 风格化诠释（三选一）的精美形式展示产品 + 产品名称标签\nM2 — 核心优势：4 个独特优势 + 主色调图标\nM3 — 使用方法：4 种使用方法 + 图标\nM4 — 关键指标：5 个精确数据点\n格式：[图标] [标签] [粗体值] [单位]\n食品：卡路里：[X] 千卡/100 克，碳水化合物：[X] 克（膳食纤维 [X] 克，糖 [X] 克），蛋白质：[X] 克，[关键维生素]：[X] 毫克（每日摄入量 [X]%），[关键矿物质]：[X] 毫克（每日摄入量 [X]%）\n药品：活性成分：[名称]，强度：[X] 毫克，起效时间：[X] 分钟，持续时间：[X] 小时，半衰期：[X] 小时\n科技：芯片：[型号]，电池续航：[X] 小时，重量：[X] 克，[关键规格]：[值]，连接性：[协议]\nM5 — 适用人群：4 组推荐人群，带有绿色对勾图标 | 3 组注意事项人群，带有琥珀色警告图标\nM6 — 重要提示：4 项注意事项 + 警告图标\nM7 — 快速参考：\n→ 食品：血糖指数 + 带有图标的膳食标签\n→ 药品：副作用 + 严重程度，带有图标\n→ 科技：兼容性 + 认证，带有图标\nM8 — 你知道吗：3 个事实（起源、科学、全球统计数据）+ 图标\n输出：1 张图片，16:9 横向，超优质液态玻璃信息图。",
      "sourcePlatform": "twitter",
      "featured": true,
      "sort": 2,
      "searchIndex": "高级液态玻璃 bento 网格产品信息图，含 8 个模块 使用 bento grid 8 模块布局创建信息图，用户可以指定食品、药品、科技等类别中的任何产品名称，选择语言、背景样式和主网格样式。 input variable: [insert product name]\nlanguage: [insert language]\n\nsystem instruction:\ncreate an image of premium liquid glass bento grid product infographic with 8 modules (card 2 to 8 show text titles only).\n1) product analysis:\n→ identify product's dominant natural color → \"hero color\"\n→ identify category: food / medicine / tech\n2) color palette (derived from hero):\n→ product + accents: full saturation hero color\n→ icons, borders: muted hero (30-40% saturation, never black)\n3) visual style:\n→ hero product: real photography (authentic, premium), 3d glass version [choose one]\n→ cards: apple liquid glass (85-90% transparent) with whisper-thin borders and subtle drop shadow for floating depth and reflecting the background color\n→ background stays behind cards and high blur where cards are [choose one]:\n  - ethereal: product essence, light caustics, abstract glow\n  - macro: product texture close-up, heavily blurred\n  - pattern: product repeated softly at 10-15% opacity\n  - context: relevant environment, blurred + desaturated\n→ add subtle motion effect\n→ asymmetric bento grid, 16:9 landscape\n→ hero card: 28-30% | info modules: 70-72%\n4) module content (8 cards):\nm1 — hero: product displayed as real photo / 3d glass / stylized interpretation (choose one)in beautiful form + product name label\nm2 — core benefits: 4 unique benefits + hero-color icons\nm3 — how to use: 4 usage methods + icons\nm4 — key metrics: 5 exact data points\nformat: [icon] [label] [bold value] [unit]\nfood: calories: [x] kcal/100g, carbs: [x]g (fiber [x]g, sugar [x]g), protein: [x]g, [key vitamin]: [x]mg ([x]% dv), [key mineral]: [x]mg ([x]% dv)\nmedicine:active: [name], strength: [x] mg, onset: [x] min, duration: [x] hrs, half-life: [x] hrs \ntech:chip: [model], battery: [x] hrs, weight: [x]g,[key spec]: [value], connectivity: [protocols]\nm5 — who it's for: 4 recommended groups with green checkmark icons | 3 caution groups with amber warning icons\nm6 — important notes: 4 precautions + warning icons\nm7 — quick reference:\n→ food: glycemic index + dietary tags with icons\n→ medicine: side effects + severity with icons\n→ tech: compatibility + certifications with icons\nm8 — did you know: 3 facts (origin, science, global stat) + icons\noutput: 1 image, 16:9 landscape, ultra-premium liquid glass infographic. 输入变量：[插入产品名称]\n语言：[插入语言]\n\n系统指令：\n创建一个包含 8 个模块（卡片 2 到 8 只显示文本标题）的优质液态玻璃 bento 网格产品信息图。\n1) 产品分析：\n→ 识别产品的主导自然色 → “主色调”\n→ 识别类别：食品 / 药品 / 科技\n2) 调色板（源自主色调）：\n→ 产品 + 强调色：完全饱和的主色调\n→ 图标、边框：柔和的主色调（30-40% 饱和度，绝不使用黑色）\n3) 视觉风格：\n→ 主打产品：真实摄影（真实、优质）、3d 玻璃版本 [二选一]\n→ 卡片：apple 液态玻璃（85-90% 透明），带有极细边框和微妙的阴影，营造浮动深度并反射背景颜色\n→ 背景保持在卡片后面，卡片所在区域高度模糊 [二选一]：\n  - 飘渺：产品精髓、柔和的焦散、抽象的光晕\n  - 微距：产品纹理特写、高度模糊\n  - 图案：产品柔和重复，不透明度为 10-15%\n  - 情境：相关环境，模糊 + 去饱和\n→ 添加微妙的动态效果\n→ 不对称 bento 网格，16:9 横向\n→ 主卡片：28-30% | 信息模块：70-72%\n4) 模块内容（8 张卡片）：\nm1 — 主打：以真实照片 / 3d 玻璃 / 风格化诠释（三选一）的精美形式展示产品 + 产品名称标签\nm2 — 核心优势：4 个独特优势 + 主色调图标\nm3 — 使用方法：4 种使用方法 + 图标\nm4 — 关键指标：5 个精确数据点\n格式：[图标] [标签] [粗体值] [单位]\n食品：卡路里：[x] 千卡/100 克，碳水化合物：[x] 克（膳食纤维 [x] 克，糖 [x] 克），蛋白质：[x] 克，[关键维生素]：[x] 毫克（每日摄入量 [x]%），[关键矿物质]：[x] 毫克（每日摄入量 [x]%）\n药品：活性成分：[名称]，强度：[x] 毫克，起效时间：[x] 分钟，持续时间：[x] 小时，半衰期：[x] 小时\n科技：芯片：[型号]，电池续航：[x] 小时，重量：[x] 克，[关键规格]：[值]，连接性：[协议]\nm5 — 适用人群：4 组推荐人群，带有绿色对勾图标 | 3 组注意事项人群，带有琥珀色警告图标\nm6 — 重要提示：4 项注意事项 + 警告图标\nm7 — 快速参考：\n→ 食品：血糖指数 + 带有图标的膳食标签\n→ 药品：副作用 + 严重程度，带有图标\n→ 科技：兼容性 + 认证，带有图标\nm8 — 你知道吗：3 个事实（起源、科学、全球统计数据）+ 图标\n输出：1 张图片，16:9 横向，超优质液态玻璃信息图。 mansi sanghani 6847",
      "likes": 0,
      "resultsCount": 1,
      "needReferenceImages": false,
      "promptCategories": []
    },
    {
      "id": 498,
      "title": "手绘风格标题图片提示（来自照片）",
      "description": "手绘风格的头部图片，描绘一个人正在介绍 Nano Banana Pro。",
      "sourceLink": "https://x.com/akirakudo_ai/status/1992096860765561190",
      "sourcePublishedAt": "2025-11-22T05:04:37.000Z",
      "author": {
        "name": "セミナー講師専門AIコンシェルジュ｜工藤 晶",
        "link": "https://x.com/akirakudo_ai"
      },
      "content": "アップロードした人物を完全再現\nその人物が「Nano Banana Pro」を紹介する note記事の見出し画像\n16：9の横長サイズ\nスタイル、カラー：シンプル、手描き風、斜体、青と緑のグラデーション\nタイトル：Google の新AI「Nano\nBanana Pro」徹底解説",
      "media": [
        "https://cms-assets.youmind.com/media/1763885651870_4szbai_G6VZiROagAAqsIh.jpg",
        "https://cms-assets.youmind.com/media/1763885654537_qf6h9o_G6VZiRWaIAA_9x5.jpg"
      ],
      "mediaThumbnails": [
        "https://cms-assets.youmind.com/media/1763885651870_4szbai_G6VZiROagAAqsIh-300x168.jpg",
        "https://cms-assets.youmind.com/media/1763885654537_qf6h9o_G6VZiRWaIAA_9x5-300x380.jpg"
      ],
      "language": "ja",
      "translatedContent": "完全重塑上传的人物。\n将其作为一篇笔记文章的标题图片，该人物在文章中介绍“Nano Banana Pro”。\n长宽比：横向 16:9。\n风格和颜色：简洁、手绘风格、斜体，带有蓝绿色渐变。\n标题文字：“深入解读 Google 全新 AI ‘Nano Banana Pro’”。",
      "sourcePlatform": "twitter",
      "featured": true,
      "sort": 2,
      "searchIndex": "手绘风格标题图片提示（来自照片） 手绘风格的头部图片，描绘一个人正在介绍 nano banana pro。 アップロードした人物を完全再現\nその人物が「nano banana pro」を紹介する note記事の見出し画像\n16：9の横長サイズ\nスタイル、カラー：シンプル、手描き風、斜体、青と緑のグラデーション\nタイトル：google の新ai「nano\nbanana pro」徹底解説 完全重塑上传的人物。\n将其作为一篇笔记文章的标题图片，该人物在文章中介绍“nano banana pro”。\n长宽比：横向 16:9。\n风格和颜色：简洁、手绘风格、斜体，带有蓝绿色渐变。\n标题文字：“深入解读 google 全新 ai ‘nano banana pro’”。 セミナー講師専門aiコンシェルジュ｜工藤 晶 498",
      "likes": 0,
      "resultsCount": 1,
      "needReferenceImages": false,
      "promptCategories": []
    },
    {
      "id": 380,
      "title": "德国水彩地图，附带各州名称",
      "description": "一个德语提示，用于生成一张水彩风格的德国地图，其中所有联邦州都用圆珠笔标注，适用于教育或信息图表风格的地图。",
      "sourceLink": "https://x.com/FlorianGallwitz/status/1991796624646091091",
      "sourcePublishedAt": "2025-11-21T09:11:35.000Z",
      "author": {
        "name": "Florian Gallwitz",
        "link": "https://x.com/FlorianGallwitz"
      },
      "content": "Erzeuge eine Karte von Deutschland im Wasserfarben-Stil, auf der alle Bundesländer mit Kugelschreiber benannt sind.",
      "media": [
        "https://cms-assets.youmind.com/media/1763886061720_fzgqaq_G6RIeSZXgAA7cOf.jpg"
      ],
      "mediaThumbnails": [
        "https://cms-assets.youmind.com/media/1763886061720_fzgqaq_G6RIeSZXgAA7cOf-300x164.jpg"
      ],
      "language": "de",
      "translatedContent": "生成一张水彩风格的德国地图，并在地图上用圆珠笔标出所有联邦州。",
      "sourcePlatform": "twitter",
      "featured": true,
      "sort": 2,
      "searchIndex": "德国水彩地图，附带各州名称 一个德语提示，用于生成一张水彩风格的德国地图，其中所有联邦州都用圆珠笔标注，适用于教育或信息图表风格的地图。 erzeuge eine karte von deutschland im wasserfarben-stil, auf der alle bundesländer mit kugelschreiber benannt sind. 生成一张水彩风格的德国地图，并在地图上用圆珠笔标出所有联邦州。 florian gallwitz 380",
      "likes": 1,
      "resultsCount": 1,
      "needReferenceImages": false,
      "promptCategories": []
    },
    {
      "id": 4031,
      "title": "元旦特辑：2026 祝福四格拼图",
      "description": "这是一个为 Nano Banana Pro 设计的详细多面板提示，用于创建一个 2x2 网格照片拼贴画。画面中，一位女性角色身着四套不同的服装，在四个不同的场景中拼凑一个拼图，拼图的中心显示“2026 新年快乐”。该提示详细说明了面部特征的精确保留、服装细节、背景元素以及时尚杂志风格的照片参数。",
      "sourceLink": "https://x.com/songguoxiansen/status/2005822648027091031",
      "sourcePublishedAt": "2025-12-30T02:06:00.000Z",
      "author": {
        "name": "松果先森",
        "link": "https://x.com/songguoxiansen"
      },
      "content": "[关键:保持精确的面部特征,保留原始脸部结构,图中角色与上传参考图完全一致] 高级摄影棚2x2宫格写真。左上格(海军蓝背景):人物穿海军蓝色制服风连衣裙,金色纽扣装饰,复古卷发配蓝色贝雷帽和珍珠耳环,她双手托举一块巨大的拼图块(左上角块,上面有\"20\"数字),向画面中心方向移动,眼神专注地看向中心拼图区域,表情认真,嘴角微笑,背景有海军条纹、船锚、\"启航新年\"字样。右上格(樱花粉背景):同一女性穿粉色蕾丝连衣裙,珍珠项链,公主头配粉色玫瑰花发夹和水晶耳环,她双手托举右上角拼图块(上面有\"26\"数字)向中心移动与左上格拼接,眼神看向拼图接缝处,表情专注期待,身体前倾,背景有粉色樱花、\"美好相遇\"字样、蝴蝶、花瓣。左下格(薄荷绿背景):同一女性穿薄荷绿色棉麻连衣裙,文艺风格,自然长发配绿色发带和木质耳环,她双手托举左下角拼图块(上面有\"元旦\"字样)向上移动与左上格对接,眼神看向拼图,表情认真,嘴巴微微抿起,背景有绿色植物、\"希望生长\"字样、新芽、叶子。右下格(柠檬黄背景):同一女性穿黄色连衣裙,向日葵图案,双马尾配黄色蝴蝶结,她双手托举最后一块右下角拼图(上面有\"快乐\"字样)完成拼图,四块拼图在画面中心完美拼成\"2026元旦快乐\"完整图案,她头后仰,眼睛看向完成的拼图,脸上洋溢着成功喜悦的笑容,画面中心爆出金色光芒和彩纸,背景有黄色太阳、\"圆满成功\"字样、笑脸、向日葵。四格之间拼图从四角汇聚到中心形成完整画面,清透妆容,明亮环形光,85mm镜头,f/1.8光圈,拼图互动四联构图,时尚杂志风格。",
      "media": [
        "https://cms-assets.youmind.com/media/1767455034932_ivuvu0_G9V-MszakAEAIBw.jpg"
      ],
      "mediaThumbnails": [
        "https://cms-assets.youmind.com/media/1767455034932_ivuvu0_G9V-MszakAEAIBw-300x300.jpg"
      ],
      "language": "zh",
      "translatedContent": "[关键：保持精确的面部特征，保留原始面部结构，图像中的人物必须与上传的参考图像完全一致] 高端照相馆 2x2 网格照片。左上角面板（海军蓝背景）：人物身穿海军蓝制服式连衣裙，饰有金色纽扣，复古卷发，戴蓝色贝雷帽和珍珠耳环。她双手举起一块巨大的拼图（左上角拼图，上面有数字“20”），将其移向画面中心。她的目光聚焦在中央拼图区域，表情严肃，略带微笑。背景是海军条纹、一个锚和文字“扬帆新年”。右上角面板（樱花粉背景）：同一位女士身穿粉色蕾丝连衣裙，戴珍珠项链，公主发型，戴粉色玫瑰发夹和水晶耳环。她双手举起右上角拼图（上面有数字“26”），将其移向中心，与左上角拼图连接。她的目光看向拼图接缝处，表情专注而期待，身体前倾。背景是粉色樱花、文字“美丽邂逅”、蝴蝶和花瓣。左下角面板（薄荷绿背景）：同一位女士身穿薄荷绿棉麻连衣裙，艺术风格，自然长发，戴绿色发带和木质耳环。她双手举起左下角拼图（上面有文字“元旦”），将其向上移动，与左上角拼图连接。她的目光看向拼图，表情严肃，嘴唇微抿。背景是绿色植物、文字“希望生长”、新芽和树叶。右下角面板（柠檬黄背景）：同一位女士身穿带有向日葵图案的黄色连衣裙，扎着黄色蝴蝶结的辫子。她将最后一块右下角拼图（上面有文字“快乐”）推入，完成拼图。四块拼图完美地在画面中心形成了完整的图案“2026 元旦快乐”。她向后仰头，看着完成的拼图，脸上洋溢着成功喜悦的笑容。画面中心爆发出金色光芒和五彩纸屑。背景是黄色太阳、文字“圆满成功”、笑脸和向日葵。拼图从四个角落汇聚到中心，形成一幅完整的画面。清晰的妆容，明亮的环形灯，85mm 镜头，f/1.8 光圈，四格构图，拼图互动，时尚杂志风格。",
      "sourcePlatform": "twitter",
      "featured": true,
      "sort": 3,
      "searchIndex": "元旦特辑：2026 祝福四格拼图 这是一个为 nano banana pro 设计的详细多面板提示，用于创建一个 2x2 网格照片拼贴画。画面中，一位女性角色身着四套不同的服装，在四个不同的场景中拼凑一个拼图，拼图的中心显示“2026 新年快乐”。该提示详细说明了面部特征的精确保留、服装细节、背景元素以及时尚杂志风格的照片参数。 [关键:保持精确的面部特征,保留原始脸部结构,图中角色与上传参考图完全一致] 高级摄影棚2x2宫格写真。左上格(海军蓝背景):人物穿海军蓝色制服风连衣裙,金色纽扣装饰,复古卷发配蓝色贝雷帽和珍珠耳环,她双手托举一块巨大的拼图块(左上角块,上面有\"20\"数字),向画面中心方向移动,眼神专注地看向中心拼图区域,表情认真,嘴角微笑,背景有海军条纹、船锚、\"启航新年\"字样。右上格(樱花粉背景):同一女性穿粉色蕾丝连衣裙,珍珠项链,公主头配粉色玫瑰花发夹和水晶耳环,她双手托举右上角拼图块(上面有\"26\"数字)向中心移动与左上格拼接,眼神看向拼图接缝处,表情专注期待,身体前倾,背景有粉色樱花、\"美好相遇\"字样、蝴蝶、花瓣。左下格(薄荷绿背景):同一女性穿薄荷绿色棉麻连衣裙,文艺风格,自然长发配绿色发带和木质耳环,她双手托举左下角拼图块(上面有\"元旦\"字样)向上移动与左上格对接,眼神看向拼图,表情认真,嘴巴微微抿起,背景有绿色植物、\"希望生长\"字样、新芽、叶子。右下格(柠檬黄背景):同一女性穿黄色连衣裙,向日葵图案,双马尾配黄色蝴蝶结,她双手托举最后一块右下角拼图(上面有\"快乐\"字样)完成拼图,四块拼图在画面中心完美拼成\"2026元旦快乐\"完整图案,她头后仰,眼睛看向完成的拼图,脸上洋溢着成功喜悦的笑容,画面中心爆出金色光芒和彩纸,背景有黄色太阳、\"圆满成功\"字样、笑脸、向日葵。四格之间拼图从四角汇聚到中心形成完整画面,清透妆容,明亮环形光,85mm镜头,f/1.8光圈,拼图互动四联构图,时尚杂志风格。 [关键：保持精确的面部特征，保留原始面部结构，图像中的人物必须与上传的参考图像完全一致] 高端照相馆 2x2 网格照片。左上角面板（海军蓝背景）：人物身穿海军蓝制服式连衣裙，饰有金色纽扣，复古卷发，戴蓝色贝雷帽和珍珠耳环。她双手举起一块巨大的拼图（左上角拼图，上面有数字“20”），将其移向画面中心。她的目光聚焦在中央拼图区域，表情严肃，略带微笑。背景是海军条纹、一个锚和文字“扬帆新年”。右上角面板（樱花粉背景）：同一位女士身穿粉色蕾丝连衣裙，戴珍珠项链，公主发型，戴粉色玫瑰发夹和水晶耳环。她双手举起右上角拼图（上面有数字“26”），将其移向中心，与左上角拼图连接。她的目光看向拼图接缝处，表情专注而期待，身体前倾。背景是粉色樱花、文字“美丽邂逅”、蝴蝶和花瓣。左下角面板（薄荷绿背景）：同一位女士身穿薄荷绿棉麻连衣裙，艺术风格，自然长发，戴绿色发带和木质耳环。她双手举起左下角拼图（上面有文字“元旦”），将其向上移动，与左上角拼图连接。她的目光看向拼图，表情严肃，嘴唇微抿。背景是绿色植物、文字“希望生长”、新芽和树叶。右下角面板（柠檬黄背景）：同一位女士身穿带有向日葵图案的黄色连衣裙，扎着黄色蝴蝶结的辫子。她将最后一块右下角拼图（上面有文字“快乐”）推入，完成拼图。四块拼图完美地在画面中心形成了完整的图案“2026 元旦快乐”。她向后仰头，看着完成的拼图，脸上洋溢着成功喜悦的笑容。画面中心爆发出金色光芒和五彩纸屑。背景是黄色太阳、文字“圆满成功”、笑脸和向日葵。拼图从四个角落汇聚到中心，形成一幅完整的画面。清晰的妆容，明亮的环形灯，85mm 镜头，f/1.8 光圈，四格构图，拼图互动，时尚杂志风格。 松果先森 4031",
      "likes": 0,
      "resultsCount": 1,
      "needReferenceImages": true,
      "promptCategories": []
    },
    {
      "id": 3438,
      "title": "一项发明的复古专利文件",
      "description": "一个旨在生成 19 世纪末美国复古专利文件图像的提示。它指定了精确的技术图纸、手写注释、陈旧的纸张纹理、锈斑、浮雕印章和蜡封等细节，非常适合历史或档案美学。",
      "sourceLink": "https://x.com/AllaAisling/status/2004212035333365763",
      "sourcePublishedAt": "2025-12-25T15:26:00.000Z",
      "author": {
        "name": "Alexandra Aisling",
        "link": "https://x.com/AllaAisling"
      },
      "content": "A vintage patent document for {argument name=\"invention\" default=\"INVENTION\"}, styled after late 1800s United States Patent Office filings. The page features precise technical drawings with numbered callouts (Fig. 1, Fig. 2, Fig. 3) showing front, side, and exploded views. Handwritten annotations in fountain-pen ink describe mechanisms. The paper is aged ivory with foxing stains and soft fold creases. An official embossed seal and red wax stamp appear in the corner. A hand-signed inventor's name and date appear at the bottom. The entire image feels like a recovered archival document—authoritative, historic, and slightly mysterious.",
      "media": [
        "https://cms-assets.youmind.com/media/1766940094520_1mg5pd_G8_m2ZVWEAAMG7y.jpg",
        "https://cms-assets.youmind.com/media/1766940095035_8t8iil_G8_mW4FWwAEwERE.jpg",
        "https://cms-assets.youmind.com/media/1766940095188_kt8ksq_G8_m_7hWoAAw19u.jpg",
        "https://cms-assets.youmind.com/media/1766940096864_fhv4oo_G8_nePrXUAAHvgn.jpg"
      ],
      "mediaThumbnails": [
        "https://cms-assets.youmind.com/media/1766940094520_1mg5pd_G8_m2ZVWEAAMG7y-300x224.jpg",
        "https://cms-assets.youmind.com/media/1766940095035_8t8iil_G8_mW4FWwAEwERE-300x224.jpg",
        "https://cms-assets.youmind.com/media/1766940095188_kt8ksq_G8_m_7hWoAAw19u-300x224.jpg",
        "https://cms-assets.youmind.com/media/1766940096864_fhv4oo_G8_nePrXUAAHvgn-300x224.jpg"
      ],
      "language": "en",
      "translatedContent": "一份关于 {argument name=\"invention\" default=\"INVENTION\"} 的复古专利文件，其风格仿照 19 世纪末美国专利局的备案文件。页面上印有精确的技术图纸，并带有编号标注（图 1、图 2、图 3），展示了正面、侧面和分解视图。钢笔墨水手写批注描述了机械装置。纸张呈陈旧的象牙色，带有锈斑和柔和的折痕。角落处有一个官方浮雕印章和红色蜡封。底部有发明者的手写签名和日期。整个图像给人一种从档案中找回的权威、历史且略带神秘感的文件之感。",
      "sourcePlatform": "twitter",
      "featured": true,
      "sort": 3,
      "searchIndex": "一项发明的复古专利文件 一个旨在生成 19 世纪末美国复古专利文件图像的提示。它指定了精确的技术图纸、手写注释、陈旧的纸张纹理、锈斑、浮雕印章和蜡封等细节，非常适合历史或档案美学。 a vintage patent document for {argument name=\"invention\" default=\"invention\"}, styled after late 1800s united states patent office filings. the page features precise technical drawings with numbered callouts (fig. 1, fig. 2, fig. 3) showing front, side, and exploded views. handwritten annotations in fountain-pen ink describe mechanisms. the paper is aged ivory with foxing stains and soft fold creases. an official embossed seal and red wax stamp appear in the corner. a hand-signed inventor's name and date appear at the bottom. the entire image feels like a recovered archival document—authoritative, historic, and slightly mysterious. 一份关于 {argument name=\"invention\" default=\"invention\"} 的复古专利文件，其风格仿照 19 世纪末美国专利局的备案文件。页面上印有精确的技术图纸，并带有编号标注（图 1、图 2、图 3），展示了正面、侧面和分解视图。钢笔墨水手写批注描述了机械装置。纸张呈陈旧的象牙色，带有锈斑和柔和的折痕。角落处有一个官方浮雕印章和红色蜡封。底部有发明者的手写签名和日期。整个图像给人一种从档案中找回的权威、历史且略带神秘感的文件之感。 alexandra aisling 3438",
      "likes": 0,
      "resultsCount": 4,
      "needReferenceImages": false,
      "promptCategories": []
    },
    {
      "id": 509,
      "title": "黑板风格的 AI 新闻摘要",
      "description": "一个日文提示，用于将 AI 新闻内容转化为手绘的、教师风格的黑板图，并附带解释。",
      "sourceLink": "https://x.com/okknews/status/1992173611520868372",
      "sourcePublishedAt": "2025-11-22T10:09:36.000Z",
      "author": {
        "name": "ひでもん | AI開発@ニュース発信",
        "link": "https://x.com/okknews"
      },
      "content": "以下を内容で黒板風の手書きみたいな感じでニュースをまとめて、先生が書いたみたいに図解や分かりやすい表現に噛み砕いてよろしく。\n—-\nGrokの検索結果",
      "media": [
        "https://cms-assets.youmind.com/media/1763885620059_vzaj75_G6WfVvIbAAEgvYg.jpg",
        "https://cms-assets.youmind.com/media/1763885622901_pk1vka_G6P2CkracAINIfP.jpg"
      ],
      "mediaThumbnails": [
        "https://cms-assets.youmind.com/media/1763885620059_vzaj75_G6WfVvIbAAEgvYg-300x300.jpg",
        "https://cms-assets.youmind.com/media/1763885622901_pk1vka_G6P2CkracAINIfP-300x300.jpg"
      ],
      "language": "ja",
      "translatedContent": "使用以下内容，以黑板风格、手写体的外观总结新闻，并像老师写的一样，用图表和易于理解的表达方式进行分解。\n—-\nGrok 的搜索结果",
      "sourcePlatform": "twitter",
      "featured": true,
      "sort": 3,
      "searchIndex": "黑板风格的 ai 新闻摘要 一个日文提示，用于将 ai 新闻内容转化为手绘的、教师风格的黑板图，并附带解释。 以下を内容で黒板風の手書きみたいな感じでニュースをまとめて、先生が書いたみたいに図解や分かりやすい表現に噛み砕いてよろしく。\n—-\ngrokの検索結果 使用以下内容，以黑板风格、手写体的外观总结新闻，并像老师写的一样，用图表和易于理解的表达方式进行分解。\n—-\ngrok 的搜索结果 ひでもん | ai開発@ニュース発信 509",
      "likes": 0,
      "resultsCount": 1,
      "needReferenceImages": false,
      "promptCategories": []
    },
    {
      "id": 553,
      "title": "详细的镜子自拍御宅族房间场景",
      "description": "一个非常详细的 Nano Banana 提示，描述了一位女性在蓝色调御宅族电脑角拍摄的镜面自拍，包含角色、环境、灯光、相机和负面提示的完整规格。",
      "sourceLink": "https://x.com/dotey/status/1976485558319722711",
      "sourcePublishedAt": "2025-10-10T03:10:53.000Z",
      "author": {
        "name": "宝玉",
        "link": "https://x.com/dotey"
      },
      "content": "### **场景**\n镜子自拍，御宅族电脑角落，蓝色调\n\n---\n\n### **主体**\n* **性别表现**: 女性\n* **年龄段**: 25岁左右\n* **种族**: 东亚\n* **身材**: 苗条，腰线分明；身材比例自然\n* **肤色**: 浅中性色调\n* **发型**:\n    * **长度**: 及腰长发\n    * **样式**: 直发，发尾微卷\n    * **颜色**: 中等棕色\n* **姿势**:\n    * **站姿**: 站立，轻微的对立式平衡站姿（contrapposto）\n    * **右手**: 手持手机挡住脸（身份被遮挡）\n    * **左臂**: 在躯干旁自然下垂\n    * **躯干**: 身体轻微后仰；露出腰腹\n* **着装**:\n    * **上衣**: 浅蓝色短款针织开衫，扣上前两颗纽扣；隐约可见蓝色法式内衣\n    * **下装**: 牛仔超短裤，两侧臀部各有一个蓝色缎带蝴蝶结\n    * **袜子**: 蓝白横条纹过膝长袜\n    * **配饰**: 蓝色可爱吉祥物手机壳\n\n---\n\n### **环境**\n* **描述**: 从挂墙镜中看到的卧室电脑角落\n* **陈设**:\n    * 白色书桌\n    * 单显示器，显示着柔和的蓝色壁纸（没有可读的文字）\n    * 机械键盘，白色键帽，放在蓝色桌垫上\n    * 鼠标，放在小号蓝色鼠标垫上\n    * PC主机在右侧，带有蓝色机箱灯效\n    * PC主机上或附近有三个动漫手办\n    * 墙上贴着一张佛塔海报\n    * 猫形台灯，带有蓝色点缀\n    * 一杯透明的玻璃水杯\n    * 窗边（镜头左侧）有一株高大的绿叶植物\n* **颜色替换**: 将所有原先的粉色元素（衣物和房间）替换为蓝色（婴儿蓝 -> 天空蓝/长春花蓝）。\n\n---\n\n### **灯光**\n* **光源**: 来自镜头左侧大窗户的日光，透过薄纱窗帘\n* **光线质感**: 柔和的漫射光\n* **白平衡 (K)**: 5200\n\n---\n\n### **相机**\n* **模式**: 智能手机后置摄像头通过镜子拍摄（无肖像/虚化模式）\n* **等效焦距 (mm)**: 26\n* **距离 (米)**:\n    * 主体到镜子: 0.6\n    * 相机到镜子: 0.5\n* **曝光**:\n    * 光圈 (f): 1.8\n    * 感光度 (ISO): 100\n    * 快门速度 (秒): 0.01\n    * 曝光补偿 (EV): -0.3\n* **对焦**: 对焦于镜中影像的躯干和短裤\n* **景深**: 自然的智能手机景深（深景深）；背景清晰可辨，无人为模糊\n* **构图**:\n    * **宽高比**: 1:1\n    * **裁剪**: 从头顶到大腿中部；画面包含书桌、显示器、PC主机和植物\n    * **角度**: 从镜子的视角轻微俯拍\n    * **构图备注**: 保持主体居中；为避免广角边缘拉伸，可以站远一些再进行方形裁剪\n\n---\n\n### **负面提示词**\n* 任何地方出现粉色/品红色\n* 美颜滤镜/磨皮皮肤；没有毛孔的外观\n* 夸张或扭曲的人体结构\n* NSFW，透视面料，走光\n* 商标，品牌名，可读的用户界面文本\n* 虚假的人像模式模糊，CGI/插画感",
      "media": [
        "https://cms-assets.youmind.com/media/1763889946850_689z0h_G23i3sJW0AASGUw.jpg"
      ],
      "mediaThumbnails": [
        "https://cms-assets.youmind.com/media/1763889946850_689z0h_G23i3sJW0AASGUw-300x300.jpg"
      ],
      "language": "zh",
      "translatedContent": "### 场景\n宅男风格电脑角里的镜面自拍，蓝色调。\n\n### 主体\n* 性别表达：女性\n* 年龄：25 岁左右\n* 种族：东亚裔\n* 身材：苗条，腰部线条分明；身体比例自然\n* 肤色：浅中性色调\n* 发型：\n    * 长度：及腰长发\n    * 样式：直发，发梢微卷\n    * 颜色：中棕色\n* 姿势：\n    * 站姿：略呈对立式平衡站姿\n    * 右手：手持智能手机置于脸前（身份隐藏）\n    * 左臂：自然下垂于身体两侧\n    * 躯干：身体略微后倾；腰部和腹部露出\n* 服装：\n    * 上衣：浅蓝色短款针织开衫，上面两颗纽扣扣上；隐约可见一件蓝色法式文胸\n    * 下装：牛仔超短裤，臀部两侧各有一个蓝色缎带蝴蝶结\n    * 袜子：蓝白条纹过膝袜\n    * 配饰：一个蓝色可爱吉祥物手机壳\n\n### 环境\n* 描述：通过壁挂镜看到的卧室电脑角\n* 家具：\n    * 白色书桌\n    * 单显示器显示柔和的蓝色壁纸（无可读文本）\n    * 机械键盘，白色键帽，置于蓝色桌垫上\n    * 鼠标置于一个小型蓝色鼠标垫上\n    * 右侧的电脑主机，带有蓝色机箱灯\n    * 电脑主机上或附近有三个动漫手办\n    * 墙上有一张宝塔海报\n    * 猫形台灯，带蓝色点缀\n    * 一杯透明玻璃水\n    * 窗边有一株高大的绿色阔叶植物（画面左侧）\n* 颜色替换：将所有原为粉色的元素（衣服和房间装饰）替换为蓝色调（从婴儿蓝到天蓝色/长春花蓝）。\n\n### 灯光\n* 光源：来自相机左侧大窗户的日光，透过薄纱窗帘\n* 光线质量：柔和、漫射光\n* 白平衡 (K)：5200\n\n### 相机\n* 模式：智能手机后置摄像头通过镜子拍摄（无人像/散景模式）\n* 等效焦距 (mm)：26\n* 距离 (m)：\n    * 主体到镜子：0.6\n    * 相机到镜子：0.5\n* 曝光：\n    * 光圈 (f)：1.8\n    * ISO：100\n    * 快门速度 (s)：0.01\n    * 曝光补偿 (EV)：-0.3\n* 对焦：对焦于镜面图像中的躯干和短裤\n* 景深：智能手机自然的深景深；背景清晰可见，无人工模糊\n* 构图：\n    * 宽高比：1:1\n    * 裁剪：从头顶到大腿中部；画面中包含书桌、显示器、电脑主机和植物\n    * 角度：从镜子的视角看，略微高角度\n    * 构图说明：主体居中；为避免广角边缘畸变，让她站得稍远一些，之后再裁剪成正方形。\n\n### 负面提示\n* 任何地方出现粉色/洋红色\n* 美颜滤镜/过度平滑的皮肤；无毛孔皮肤外观\n* 夸张或扭曲的解剖结构\n* 不雅内容、透视织物、走光\n* 标志、品牌名称或可读的用户界面文本\n* 虚假的人像模式模糊、CGI/插画感",
      "sourcePlatform": "twitter",
      "featured": true,
      "sort": 4,
      "searchIndex": "详细的镜子自拍御宅族房间场景 一个非常详细的 nano banana 提示，描述了一位女性在蓝色调御宅族电脑角拍摄的镜面自拍，包含角色、环境、灯光、相机和负面提示的完整规格。 ### **场景**\n镜子自拍，御宅族电脑角落，蓝色调\n\n---\n\n### **主体**\n* **性别表现**: 女性\n* **年龄段**: 25岁左右\n* **种族**: 东亚\n* **身材**: 苗条，腰线分明；身材比例自然\n* **肤色**: 浅中性色调\n* **发型**:\n    * **长度**: 及腰长发\n    * **样式**: 直发，发尾微卷\n    * **颜色**: 中等棕色\n* **姿势**:\n    * **站姿**: 站立，轻微的对立式平衡站姿（contrapposto）\n    * **右手**: 手持手机挡住脸（身份被遮挡）\n    * **左臂**: 在躯干旁自然下垂\n    * **躯干**: 身体轻微后仰；露出腰腹\n* **着装**:\n    * **上衣**: 浅蓝色短款针织开衫，扣上前两颗纽扣；隐约可见蓝色法式内衣\n    * **下装**: 牛仔超短裤，两侧臀部各有一个蓝色缎带蝴蝶结\n    * **袜子**: 蓝白横条纹过膝长袜\n    * **配饰**: 蓝色可爱吉祥物手机壳\n\n---\n\n### **环境**\n* **描述**: 从挂墙镜中看到的卧室电脑角落\n* **陈设**:\n    * 白色书桌\n    * 单显示器，显示着柔和的蓝色壁纸（没有可读的文字）\n    * 机械键盘，白色键帽，放在蓝色桌垫上\n    * 鼠标，放在小号蓝色鼠标垫上\n    * pc主机在右侧，带有蓝色机箱灯效\n    * pc主机上或附近有三个动漫手办\n    * 墙上贴着一张佛塔海报\n    * 猫形台灯，带有蓝色点缀\n    * 一杯透明的玻璃水杯\n    * 窗边（镜头左侧）有一株高大的绿叶植物\n* **颜色替换**: 将所有原先的粉色元素（衣物和房间）替换为蓝色（婴儿蓝 -> 天空蓝/长春花蓝）。\n\n---\n\n### **灯光**\n* **光源**: 来自镜头左侧大窗户的日光，透过薄纱窗帘\n* **光线质感**: 柔和的漫射光\n* **白平衡 (k)**: 5200\n\n---\n\n### **相机**\n* **模式**: 智能手机后置摄像头通过镜子拍摄（无肖像/虚化模式）\n* **等效焦距 (mm)**: 26\n* **距离 (米)**:\n    * 主体到镜子: 0.6\n    * 相机到镜子: 0.5\n* **曝光**:\n    * 光圈 (f): 1.8\n    * 感光度 (iso): 100\n    * 快门速度 (秒): 0.01\n    * 曝光补偿 (ev): -0.3\n* **对焦**: 对焦于镜中影像的躯干和短裤\n* **景深**: 自然的智能手机景深（深景深）；背景清晰可辨，无人为模糊\n* **构图**:\n    * **宽高比**: 1:1\n    * **裁剪**: 从头顶到大腿中部；画面包含书桌、显示器、pc主机和植物\n    * **角度**: 从镜子的视角轻微俯拍\n    * **构图备注**: 保持主体居中；为避免广角边缘拉伸，可以站远一些再进行方形裁剪\n\n---\n\n### **负面提示词**\n* 任何地方出现粉色/品红色\n* 美颜滤镜/磨皮皮肤；没有毛孔的外观\n* 夸张或扭曲的人体结构\n* nsfw，透视面料，走光\n* 商标，品牌名，可读的用户界面文本\n* 虚假的人像模式模糊，cgi/插画感 ### 场景\n宅男风格电脑角里的镜面自拍，蓝色调。\n\n### 主体\n* 性别表达：女性\n* 年龄：25 岁左右\n* 种族：东亚裔\n* 身材：苗条，腰部线条分明；身体比例自然\n* 肤色：浅中性色调\n* 发型：\n    * 长度：及腰长发\n    * 样式：直发，发梢微卷\n    * 颜色：中棕色\n* 姿势：\n    * 站姿：略呈对立式平衡站姿\n    * 右手：手持智能手机置于脸前（身份隐藏）\n    * 左臂：自然下垂于身体两侧\n    * 躯干：身体略微后倾；腰部和腹部露出\n* 服装：\n    * 上衣：浅蓝色短款针织开衫，上面两颗纽扣扣上；隐约可见一件蓝色法式文胸\n    * 下装：牛仔超短裤，臀部两侧各有一个蓝色缎带蝴蝶结\n    * 袜子：蓝白条纹过膝袜\n    * 配饰：一个蓝色可爱吉祥物手机壳\n\n### 环境\n* 描述：通过壁挂镜看到的卧室电脑角\n* 家具：\n    * 白色书桌\n    * 单显示器显示柔和的蓝色壁纸（无可读文本）\n    * 机械键盘，白色键帽，置于蓝色桌垫上\n    * 鼠标置于一个小型蓝色鼠标垫上\n    * 右侧的电脑主机，带有蓝色机箱灯\n    * 电脑主机上或附近有三个动漫手办\n    * 墙上有一张宝塔海报\n    * 猫形台灯，带蓝色点缀\n    * 一杯透明玻璃水\n    * 窗边有一株高大的绿色阔叶植物（画面左侧）\n* 颜色替换：将所有原为粉色的元素（衣服和房间装饰）替换为蓝色调（从婴儿蓝到天蓝色/长春花蓝）。\n\n### 灯光\n* 光源：来自相机左侧大窗户的日光，透过薄纱窗帘\n* 光线质量：柔和、漫射光\n* 白平衡 (k)：5200\n\n### 相机\n* 模式：智能手机后置摄像头通过镜子拍摄（无人像/散景模式）\n* 等效焦距 (mm)：26\n* 距离 (m)：\n    * 主体到镜子：0.6\n    * 相机到镜子：0.5\n* 曝光：\n    * 光圈 (f)：1.8\n    * iso：100\n    * 快门速度 (s)：0.01\n    * 曝光补偿 (ev)：-0.3\n* 对焦：对焦于镜面图像中的躯干和短裤\n* 景深：智能手机自然的深景深；背景清晰可见，无人工模糊\n* 构图：\n    * 宽高比：1:1\n    * 裁剪：从头顶到大腿中部；画面中包含书桌、显示器、电脑主机和植物\n    * 角度：从镜子的视角看，略微高角度\n    * 构图说明：主体居中；为避免广角边缘畸变，让她站得稍远一些，之后再裁剪成正方形。\n\n### 负面提示\n* 任何地方出现粉色/洋红色\n* 美颜滤镜/过度平滑的皮肤；无毛孔皮肤外观\n* 夸张或扭曲的解剖结构\n* 不雅内容、透视织物、走光\n* 标志、品牌名称或可读的用户界面文本\n* 虚假的人像模式模糊、cgi/插画感 宝玉 553",
      "likes": 0,
      "resultsCount": 4,
      "needReferenceImages": false,
      "promptCategories": []
    },
    {
      "id": 811,
      "title": "江户时代浮世绘风格的现代场景再现",
      "description": "一个高度结构化的图像提示，用于描绘一个被重新构想为江户时代日本浮世绘木版画的现代场景，并详细指导如何处理时代错位的科技、构图、纹理和色彩。",
      "sourceLink": "https://x.com/VoxcatAI/status/1995497350543110411",
      "sourcePublishedAt": "2025-12-01T14:16:57.000Z",
      "author": {
        "name": "VoxcatAI",
        "link": "https://x.com/VoxcatAI"
      },
      "content": "A Japanese Edo-period Ukiyo-e woodblock print (浮世絵 木版画). The overall feeling is a surreal collaboration between masters like Hokusai and Hiroshige, reimagining modern technology through an ancient lens.\n\n**The Scene:** {argument name=\"现代场景\" default=\"繁忙的涩谷十字路口\"}\n\n**Edo Transformation Logic:**\nCharacters wear Edo-era kimono but perform modern actions. All technology is transformed into surreal Edo equivalents:\n* **Smartphones** are glowing, illustrated paper scrolls being read intently.\n* **Metro stations/Trains** are giant articulated wooden centipede carriages shuffling through crowds.\n* **Skyscrapers** are reimagined as endless, towering wooden pagodas reaching into dramatic clouds.\n* **Robots/Mecha** appear as giant, armored woodblock golems.\n\nThe composition uses a flattened perspective with large, bold, hand-carved ink outlines (太い墨線). The background features heavily stylized Ukiyo-e wave patterns and dramatic, swirling clouds, with a distant Mt. Fuji visible on the horizon.\n\nThe image must look like a physical print, not a digital painting.\n* **Texture:** Strong visible wood grain texture (木目) and rough paper fibers throughout the piece.\n* **Printing Imperfections:** Pigment bleeding is evident. Crucially, simulate hand-pressed plates with slight **color misalignment (版ズレ)** for authenticity.\n* **Color Palette:** Strictly limited to traditional mineral pigments. Dominant use of Prussian blue (浮世絵ブルー), vermilion red, and muted yellow ochre.\n* **Lighting:** Soft, flat, shadow-free lighting with no digital gradients.\n\n* **Aspect Ratio:** 3:4 vertical poster.\n* **Extras:** Include vertical Japanese calligraphy describing the scene and a traditional red artist seal stamp (落款) in a corner.",
      "media": [
        "https://cms-assets.youmind.com/media/1764915832381_renotr_G7FuPlzbYAAsuo2.jpg"
      ],
      "mediaThumbnails": [
        "https://cms-assets.youmind.com/media/1764915832381_renotr_G7FuPlzbYAAsuo2-300x401.jpg"
      ],
      "language": "en",
      "translatedContent": "一幅日本江户时代的浮世绘木版画。整体感觉是葛饰北斋和歌川广重等大师的超现实合作，通过古老的视角重新构想现代科技。\n\n**场景：** {argument name=\"modern scene\" default=\"繁忙的涩谷十字路口\"}\n\n**江户时代转换逻辑：**\n人物身着江户时代的和服，但进行着现代活动。所有科技都被转化为超现实的江户时代等效物：\n* 智能手机是发光的、精心绘制的纸质卷轴，人们正聚精会神地阅读。\n* 地铁站和火车是巨大的铰接式木制蜈蚣车厢，在人群中穿梭。\n* 摩天大楼被重新构想成无尽的、高耸入云的木制宝塔，直插戏剧性的云层。\n* 机器人和机甲则以巨大的、身披盔甲的木版傀儡形象出现。\n\n构图采用扁平透视，带有粗大、醒目的手刻墨线轮廓。背景是高度风格化的浮世绘波浪图案和戏剧性的漩涡状云朵，远处地平线上可见富士山。\n\n图像必须看起来像一幅实物版画，而非数字绘画。\n* 纹理：整个作品中都有清晰可见的木纹纹理和粗糙的纸纤维。\n* 印刷缺陷：颜料渗色明显。模拟手工压制版画，带有轻微的颜色错位，以增加真实感。\n* 色彩调色板：严格限于传统的矿物颜料，主要使用普鲁士蓝、朱红色和柔和的赭黄色。\n* 光照：柔和、平坦、无阴影的光照，没有数字渐变。\n\n宽高比为 3:4 的竖向海报。包含描述场景的竖排日文书法，并在一个角落有一个传统的红色艺术家印章。",
      "sourcePlatform": "twitter",
      "featured": true,
      "sort": 5,
      "searchIndex": "江户时代浮世绘风格的现代场景再现 一个高度结构化的图像提示，用于描绘一个被重新构想为江户时代日本浮世绘木版画的现代场景，并详细指导如何处理时代错位的科技、构图、纹理和色彩。 a japanese edo-period ukiyo-e woodblock print (浮世絵 木版画). the overall feeling is a surreal collaboration between masters like hokusai and hiroshige, reimagining modern technology through an ancient lens.\n\n**the scene:** {argument name=\"现代场景\" default=\"繁忙的涩谷十字路口\"}\n\n**edo transformation logic:**\ncharacters wear edo-era kimono but perform modern actions. all technology is transformed into surreal edo equivalents:\n* **smartphones** are glowing, illustrated paper scrolls being read intently.\n* **metro stations/trains** are giant articulated wooden centipede carriages shuffling through crowds.\n* **skyscrapers** are reimagined as endless, towering wooden pagodas reaching into dramatic clouds.\n* **robots/mecha** appear as giant, armored woodblock golems.\n\nthe composition uses a flattened perspective with large, bold, hand-carved ink outlines (太い墨線). the background features heavily stylized ukiyo-e wave patterns and dramatic, swirling clouds, with a distant mt. fuji visible on the horizon.\n\nthe image must look like a physical print, not a digital painting.\n* **texture:** strong visible wood grain texture (木目) and rough paper fibers throughout the piece.\n* **printing imperfections:** pigment bleeding is evident. crucially, simulate hand-pressed plates with slight **color misalignment (版ズレ)** for authenticity.\n* **color palette:** strictly limited to traditional mineral pigments. dominant use of prussian blue (浮世絵ブルー), vermilion red, and muted yellow ochre.\n* **lighting:** soft, flat, shadow-free lighting with no digital gradients.\n\n* **aspect ratio:** 3:4 vertical poster.\n* **extras:** include vertical japanese calligraphy describing the scene and a traditional red artist seal stamp (落款) in a corner. 一幅日本江户时代的浮世绘木版画。整体感觉是葛饰北斋和歌川广重等大师的超现实合作，通过古老的视角重新构想现代科技。\n\n**场景：** {argument name=\"modern scene\" default=\"繁忙的涩谷十字路口\"}\n\n**江户时代转换逻辑：**\n人物身着江户时代的和服，但进行着现代活动。所有科技都被转化为超现实的江户时代等效物：\n* 智能手机是发光的、精心绘制的纸质卷轴，人们正聚精会神地阅读。\n* 地铁站和火车是巨大的铰接式木制蜈蚣车厢，在人群中穿梭。\n* 摩天大楼被重新构想成无尽的、高耸入云的木制宝塔，直插戏剧性的云层。\n* 机器人和机甲则以巨大的、身披盔甲的木版傀儡形象出现。\n\n构图采用扁平透视，带有粗大、醒目的手刻墨线轮廓。背景是高度风格化的浮世绘波浪图案和戏剧性的漩涡状云朵，远处地平线上可见富士山。\n\n图像必须看起来像一幅实物版画，而非数字绘画。\n* 纹理：整个作品中都有清晰可见的木纹纹理和粗糙的纸纤维。\n* 印刷缺陷：颜料渗色明显。模拟手工压制版画，带有轻微的颜色错位，以增加真实感。\n* 色彩调色板：严格限于传统的矿物颜料，主要使用普鲁士蓝、朱红色和柔和的赭黄色。\n* 光照：柔和、平坦、无阴影的光照，没有数字渐变。\n\n宽高比为 3:4 的竖向海报。包含描述场景的竖排日文书法，并在一个角落有一个传统的红色艺术家印章。 voxcatai 811",
      "likes": 0,
      "resultsCount": 1,
      "needReferenceImages": false,
      "promptCategories": []
    },
    {
      "id": 532,
      "title": "火车广告风格的书籍广告图片",
      "description": "一个详细的日文提示，用于生成一个 16:9 的商业书籍风格广告，其中包含特定书籍图片和日文文案要点。",
      "sourceLink": "https://x.com/kawai_design/status/1992142466255114727",
      "sourcePublishedAt": "2025-11-22T08:05:50.000Z",
      "author": {
        "name": "KAWAI",
        "link": "https://x.com/kawai_design"
      },
      "content": "広告画像を生成してください\n\n==== 広告の仕様 ===\n- アスペクト比: 16:9（横長）\n- 広告したい商品: 添付画像1枚目の本\n- アイキャッチ: 添付画像1枚目の本を立体的に配置\n- 使用言語: 日本語\n- テイスト: ビジネス書の広告\n\n# 掲載文字:\n- プリデッドコピー: | 【 発売1週間ほどで重版決定 】\n\n書籍「{argument name=\"書籍名\" default=\"AIでゼロからデザイン\"}」好評発売中\n\nAmazon 売れ筋ランキング\n商業デザイン売上 1位 を記録（10/15 調べ）\nhttps://t.co/QxbYpfFVj6",
      "media": [
        "https://cms-assets.youmind.com/media/1763885539326_yao7in_G6WBYReawAAcp2x.jpg"
      ],
      "mediaThumbnails": [
        "https://cms-assets.youmind.com/media/1763885539326_yao7in_G6WBYReawAAcp2x-300x168.jpg"
      ],
      "language": "ja",
      "translatedContent": "请生成一张广告图片。\n\n==== 广告规格 ===\n- 宽高比：16:9（横向）\n- 广告产品：第一张附件图片中的书籍\n- 主要吸睛点：将第一张附件图片中的书籍以三维方式呈现\n- 语言：日语\n- 风格：商业书籍广告\n\n# 需包含的文字：\n- 前导文案：【发售约一周后决定加印】\n\n书籍《{argument name=\"book_title_en\" default=\"Designing from Zero with AI\"}》现正热销中。\n\n亚马逊畅销书排行榜\n商业设计类销量排名第一（截至 10/15）\nhttps://t.co/QxbYpfFVj6",
      "sourcePlatform": "twitter",
      "sort": 3,
      "searchIndex": "火车广告风格的书籍广告图片 一个详细的日文提示，用于生成一个 16:9 的商业书籍风格广告，其中包含特定书籍图片和日文文案要点。 広告画像を生成してください\n\n==== 広告の仕様 ===\n- アスペクト比: 16:9（横長）\n- 広告したい商品: 添付画像1枚目の本\n- アイキャッチ: 添付画像1枚目の本を立体的に配置\n- 使用言語: 日本語\n- テイスト: ビジネス書の広告\n\n# 掲載文字:\n- プリデッドコピー: | 【 発売1週間ほどで重版決定 】\n\n書籍「{argument name=\"書籍名\" default=\"aiでゼロからデザイン\"}」好評発売中\n\namazon 売れ筋ランキング\n商業デザイン売上 1位 を記録（10/15 調べ）\nhttps://t.co/qxbypffvj6 请生成一张广告图片。\n\n==== 广告规格 ===\n- 宽高比：16:9（横向）\n- 广告产品：第一张附件图片中的书籍\n- 主要吸睛点：将第一张附件图片中的书籍以三维方式呈现\n- 语言：日语\n- 风格：商业书籍广告\n\n# 需包含的文字：\n- 前导文案：【发售约一周后决定加印】\n\n书籍《{argument name=\"book_title_en\" default=\"designing from zero with ai\"}》现正热销中。\n\n亚马逊畅销书排行榜\n商业设计类销量排名第一（截至 10/15）\nhttps://t.co/qxbypffvj6 kawai 532",
      "likes": 1,
      "resultsCount": 0,
      "needReferenceImages": true,
      "promptCategories": []
    },
    {
      "id": 13135,
      "title": "微型草莓兔",
      "description": "用户描述生成的图像带有甜美的草莓香气，并称其为“微型草莓兔”（Micro Strawberry Rabbit），同时提到这是一次生成成功的。这段描述性文字作为图像生成模型 nano-banana-pro 的提示词。",
      "sourceLink": "https://x.com/nonoyam_0/status/2039592939698143463",
      "sourcePublishedAt": "2026-04-02T06:37:05.000Z",
      "author": {
        "name": "𑣲노노얌💛",
        "link": "https://x.com/nonoyam_0"
      },
      "content": "진짜 달달한 딸기향이야￼￼￼￼￼￼🍓\n1트만에 뽑았음\n이렇게 귀여운 마이크로딸기리빗이라니🐰🍓",
      "media": [
        "https://cms-assets.youmind.com/media/1775112289126_tcj6dt_HE4W99abAAAkOOE.jpg",
        "https://cms-assets.youmind.com/media/1775112289364_s9mef8_HE4W99ba0AApZ9Y.jpg",
        "https://cms-assets.youmind.com/media/1775112289300_oy7tbd_HE4W99baQAALuxD.jpg"
      ],
      "mediaThumbnails": [
        "https://cms-assets.youmind.com/media/1775112289126_tcj6dt_HE4W99abAAAkOOE-300x301.jpg",
        "https://cms-assets.youmind.com/media/1775112289364_s9mef8_HE4W99ba0AApZ9Y-300x376.jpg",
        "https://cms-assets.youmind.com/media/1775112289300_oy7tbd_HE4W99baQAALuxD-300x400.jpg"
      ],
      "language": "ko",
      "translatedContent": "闻起来真的有甜甜的草莓味￼￼￼￼￼￼🍓\n一次就生成成功了\n超可爱的微型草莓兔🐰🍓",
      "sourcePlatform": "twitter",
      "searchIndex": "微型草莓兔 用户描述生成的图像带有甜美的草莓香气，并称其为“微型草莓兔”（micro strawberry rabbit），同时提到这是一次生成成功的。这段描述性文字作为图像生成模型 nano-banana-pro 的提示词。 진짜 달달한 딸기향이야￼￼￼￼￼￼🍓\n1트만에 뽑았음\n이렇게 귀여운 마이크로딸기리빗이라니🐰🍓 闻起来真的有甜甜的草莓味￼￼￼￼￼￼🍓\n一次就生成成功了\n超可爱的微型草莓兔🐰🍓 𑣲노노얌💛 13135",
      "likes": 0,
      "resultsCount": 0,
      "needReferenceImages": false,
      "promptCategories": []
    },
    {
      "id": 13123,
      "title": "80 年代复古健美风的 Margot Robbie 替身",
      "description": "一份为 Nano Banana Pro 准备的详细 JSON 提示词，用于生成一张超逼真的图像：身着 1980 年代复古健美服（紧身连衣裤、氨纶短裤、金色腰带）的 Margot Robbie 替身，在明亮的阳光下摆拍，呈现黄金时刻的质感，强调鲜艳的色彩和高技术画质。",
      "sourceLink": "https://x.com/Artist04048661/status/2039536333329240331",
      "sourcePublishedAt": "2026-04-02T02:52:09.000Z",
      "author": {
        "name": "LexiPrompt",
        "link": "https://x.com/Artist04048661"
      },
      "content": "{\n  \"image_generation_prompt\": {\n    \"subject\": {\n      \"description\": \"Woman with a strong resemblance to {argument name=\"subject name\" default=\"Margot Robbie\"}.\",\n      \"hair\": \"Voluminous messy updo bun, loose strands framing face, held back.\",\n      \"face\": \"Dramatic retro makeup, sharp eyeliner, matte skin finish.\",\n      \"body\": \"Athletic build, tanned smooth skin.\"\n    },\n    \"attire\": {\n      \"clothing\": \"High-cut geometric patterned leotard over pale yellow spandex shorts, wide gold belt.\",\n      \"style\": \"1980s retro aerobics fashion.\"\n    },\n    \"styling_and_accessories\": {\n      \"jewelry\": [\n        \"Gold hoop earrings\",\n        \"Pink terrycloth headband\",\n        \"Pink wristbands\",\n        \"Clear blue tumbler with pink straw\"\n      ]\n    },\n    \"environment\": {\n      \"setting\": \"Sunny outdoor paved pathway.\",\n      \"background\": \"Heavily blurred palm trees and hazy bright sky.\",\n      \"water\": \"None.\"\n    },\n    \"pose\": {\n      \"posture\": \"Standing facing away, twisting upper body to look back over shoulder.\",\n      \"arms\": \"One hand holding cup to lips, other hand resting on lower back.\",\n      \"angle\": \"Medium shot, eye-level.\"\n    },\n    \"lighting_and_mood\": {\n      \"lighting\": \"Bright natural sunlight, warm golden hour glow, soft shadows.\",\n      \"mood\": \"Vibrant, energetic, nostalgic.\",\n      \"colors\": \"Neon pink, pale yellow, bright cyan, gold.\"\n    },\n    \"camera_and_technical\": {\n      \"style\": \"Ultra Photorealistic, RAW photo.\",\n      \"lens\": \"85mm\",\n      \"aperture\": \"f/1.8\",\n      \"quality_tags\": [\n        \"8k resolution\",\n        \"highly detailed\",\n        \"volumetric lighting\",\n        \"ray tracing reflections\",\n        \"hyper-realistic texture\",\n        \"Hasselblad photography\"\n      ]\n    }\n  }\n}",
      "media": [
        "https://cms-assets.youmind.com/media/1775112283959_d24770_HE3jWSDWgAAbk6y.jpg"
      ],
      "mediaThumbnails": [
        "https://cms-assets.youmind.com/media/1775112283959_d24770_HE3jWSDWgAAbk6y-300x402.jpg"
      ],
      "language": "en",
      "translatedContent": "{\n  \"image_generation_prompt\": {\n    \"subject\": {\n      \"description\": \"长相酷似 {argument name=\"subject name\" default=\"Margot Robbie\"} 的女性。\",\n      \"hair\": \"蓬松凌乱的盘发发髻，几缕碎发散落在脸颊旁，向后梳理。\",\n      \"face\": \"复古浓妆，精致的眼线，哑光肤质。\",\n      \"body\": \"健美体格，古铜色光滑肌肤。\"\n    },\n    \"attire\": {\n      \"clothing\": \"几何图案高开叉紧身连衣裤，外搭淡黄色氨纶短裤，配有宽大的金色腰带。\",\n      \"style\": \"1980 年代复古健美时尚。\"\n    },\n    \"styling_and_accessories\": {\n      \"jewelry\": [\n        \"金色大圆耳环\",\n        \"粉色毛巾布发带\",\n        \"粉色护腕\",\n        \"透明蓝色水杯，配粉色吸管\"\n      ]\n    },\n    \"environment\": {\n      \"setting\": \"阳光明媚的户外铺装小径。\",\n      \"background\": \"高度模糊的棕榈树和朦胧明亮的天空。\",\n      \"water\": \"无。\"\n    },\n    \"pose\": {\n      \"posture\": \"背对镜头站立，上半身扭转，回头看向后方。\",\n      \"arms\": \"一只手拿着杯子放在唇边，另一只手放在后腰处。\",\n      \"angle\": \"中景，平视角度。\"\n    },\n    \"lighting_and_mood\": {\n      \"lighting\": \"明亮的自然阳光，温暖的黄金时刻光泽，柔和的阴影。\",\n      \"mood\": \"充满活力、动感、怀旧。\",\n      \"colors\": \"霓虹粉、淡黄、亮青色、金色。\"\n    },\n    \"camera_and_technical\": {\n      \"style\": \"超逼真写实，RAW 原片。\",\n      \"lens\": \"85mm\",\n      \"aperture\": \"f/1.8\",\n      \"quality_tags\": [\n        \"8k 分辨率\",\n        \"高细节\",\n        \"体积光\",\n        \"光线追踪反射\",\n        \"超写实纹理\",\n        \"哈苏摄影\"\n      ]\n    }\n  }\n}",
      "sourcePlatform": "twitter",
      "searchIndex": "80 年代复古健美风的 margot robbie 替身 一份为 nano banana pro 准备的详细 json 提示词，用于生成一张超逼真的图像：身着 1980 年代复古健美服（紧身连衣裤、氨纶短裤、金色腰带）的 margot robbie 替身，在明亮的阳光下摆拍，呈现黄金时刻的质感，强调鲜艳的色彩和高技术画质。 {\n  \"image_generation_prompt\": {\n    \"subject\": {\n      \"description\": \"woman with a strong resemblance to {argument name=\"subject name\" default=\"margot robbie\"}.\",\n      \"hair\": \"voluminous messy updo bun, loose strands framing face, held back.\",\n      \"face\": \"dramatic retro makeup, sharp eyeliner, matte skin finish.\",\n      \"body\": \"athletic build, tanned smooth skin.\"\n    },\n    \"attire\": {\n      \"clothing\": \"high-cut geometric patterned leotard over pale yellow spandex shorts, wide gold belt.\",\n      \"style\": \"1980s retro aerobics fashion.\"\n    },\n    \"styling_and_accessories\": {\n      \"jewelry\": [\n        \"gold hoop earrings\",\n        \"pink terrycloth headband\",\n        \"pink wristbands\",\n        \"clear blue tumbler with pink straw\"\n      ]\n    },\n    \"environment\": {\n      \"setting\": \"sunny outdoor paved pathway.\",\n      \"background\": \"heavily blurred palm trees and hazy bright sky.\",\n      \"water\": \"none.\"\n    },\n    \"pose\": {\n      \"posture\": \"standing facing away, twisting upper body to look back over shoulder.\",\n      \"arms\": \"one hand holding cup to lips, other hand resting on lower back.\",\n      \"angle\": \"medium shot, eye-level.\"\n    },\n    \"lighting_and_mood\": {\n      \"lighting\": \"bright natural sunlight, warm golden hour glow, soft shadows.\",\n      \"mood\": \"vibrant, energetic, nostalgic.\",\n      \"colors\": \"neon pink, pale yellow, bright cyan, gold.\"\n    },\n    \"camera_and_technical\": {\n      \"style\": \"ultra photorealistic, raw photo.\",\n      \"lens\": \"85mm\",\n      \"aperture\": \"f/1.8\",\n      \"quality_tags\": [\n        \"8k resolution\",\n        \"highly detailed\",\n        \"volumetric lighting\",\n        \"ray tracing reflections\",\n        \"hyper-realistic texture\",\n        \"hasselblad photography\"\n      ]\n    }\n  }\n} {\n  \"image_generation_prompt\": {\n    \"subject\": {\n      \"description\": \"长相酷似 {argument name=\"subject name\" default=\"margot robbie\"} 的女性。\",\n      \"hair\": \"蓬松凌乱的盘发发髻，几缕碎发散落在脸颊旁，向后梳理。\",\n      \"face\": \"复古浓妆，精致的眼线，哑光肤质。\",\n      \"body\": \"健美体格，古铜色光滑肌肤。\"\n    },\n    \"attire\": {\n      \"clothing\": \"几何图案高开叉紧身连衣裤，外搭淡黄色氨纶短裤，配有宽大的金色腰带。\",\n      \"style\": \"1980 年代复古健美时尚。\"\n    },\n    \"styling_and_accessories\": {\n      \"jewelry\": [\n        \"金色大圆耳环\",\n        \"粉色毛巾布发带\",\n        \"粉色护腕\",\n        \"透明蓝色水杯，配粉色吸管\"\n      ]\n    },\n    \"environment\": {\n      \"setting\": \"阳光明媚的户外铺装小径。\",\n      \"background\": \"高度模糊的棕榈树和朦胧明亮的天空。\",\n      \"water\": \"无。\"\n    },\n    \"pose\": {\n      \"posture\": \"背对镜头站立，上半身扭转，回头看向后方。\",\n      \"arms\": \"一只手拿着杯子放在唇边，另一只手放在后腰处。\",\n      \"angle\": \"中景，平视角度。\"\n    },\n    \"lighting_and_mood\": {\n      \"lighting\": \"明亮的自然阳光，温暖的黄金时刻光泽，柔和的阴影。\",\n      \"mood\": \"充满活力、动感、怀旧。\",\n      \"colors\": \"霓虹粉、淡黄、亮青色、金色。\"\n    },\n    \"camera_and_technical\": {\n      \"style\": \"超逼真写实，raw 原片。\",\n      \"lens\": \"85mm\",\n      \"aperture\": \"f/1.8\",\n      \"quality_tags\": [\n        \"8k 分辨率\",\n        \"高细节\",\n        \"体积光\",\n        \"光线追踪反射\",\n        \"超写实纹理\",\n        \"哈苏摄影\"\n      ]\n    }\n  }\n} lexiprompt 13123",
      "likes": 0,
      "resultsCount": 0,
      "needReferenceImages": false,
      "promptCategories": []
    },
    {
      "id": 13128,
      "title": "Millie Bobby Brown 的电影感节日摄影作品",
      "description": "这是一个为 Nano Banana 2 设计的高细节、电影感、照片级真实感的提示词，用于生成 Millie Bobby Brown 在摩天轮附近的户外节日场景照片。提示词详细说明了人物外观、服装、动态姿势、光照条件（黄昏光线结合霓虹轮廓光）、相机设置（竖屏构图、长焦镜头、浅景深）以及整体氛围。",
      "sourceLink": "https://x.com/Giulia_4i/status/2039474538212401240",
      "sourcePublishedAt": "2026-04-01T22:46:36.000Z",
      "author": {
        "name": "Giulia",
        "link": "https://x.com/Giulia_4i"
      },
      "content": "\"Create a highly detailed, cinematic, photorealistic lifestyle photograph of {argument name=\"subject\" default=\"Millie Bobby Brown\"} at an outdoor festival during early evening. She has long, slightly wavy hair and a playful expression, smiling with her teeth visible while looking back over her right shoulder directly at the camera. Her makeup is light and natural. She is wearing sunglasses resting on top of her head, large teal-colored drop-shaped earrings, and a bright yellow festival wristband on her right wrist.\n\nShe has a slim, fit body and is dressed in a light greenish-gray cropped short-sleeve t-shirt with a faded print on the front, paired with light blue ripped denim cut-off shorts. On her feet, she wears classic black high-top canvas sneakers with white laces, a white toe cap, and white soles.\n\nHer pose is dynamic and confident: she is standing with her back facing the camera, her torso twisted as she looks over her right shoulder. Her right arm is slightly raised with a relaxed hand near her waist, while her left arm hangs naturally. Her weight is mostly on her extended left leg, with her right leg slightly bent, creating a casual and energetic posture.\n\nThe setting is a lively outdoor festival, resembling an amusement park. In the background, there is a large Ferris wheel illuminated with neon pink and white lights along its spokes and rim. Blurred figures of people walking can be seen in the background, and the ground appears dark, like a temporary paved surface.\n\nThe lighting combines soft dusk light with strong, cool artificial ambient light coming from the Ferris wheel. The neon lights create a glowing rim light around her hair, shoulders, and silhouette, while a soft front fill light clearly illuminates her face and body.\n\nThe overall mood is vibrant, youthful, energetic, and cinematic, capturing the lively atmosphere of a summer festival. The color palette emphasizes neon pink, bright white, light denim blue, greenish-gray, black, and soft pink tones.\n\nThe image should be captured in a portrait orientation (9:16), using a slightly low camera angle and a tight framing (medium close-up or American shot), with the subject occupying almost the entire frame. Use a telephoto lens between 85mm and 135mm to compress perspective and emphasize the subject, with a shallow depth of field to keep her extremely sharp while the background remains softly blurred with bokeh. The final result should be highly detailed, with sharp focus on the subject, vibrant and contrasting colors, professional lighting, and 8K resolution quality.\"",
      "media": [
        "https://cms-assets.youmind.com/media/1775112286065_i3xthk_HE2rSFhXkAAGgfN.jpg",
        "https://cms-assets.youmind.com/media/1775112286121_ci0x0k_HE2rSMaboAAoGaX.jpg",
        "https://cms-assets.youmind.com/media/1775112286153_qoy1f1_HE2rSJoXEAAym-q.jpg"
      ],
      "mediaThumbnails": [
        "https://cms-assets.youmind.com/media/1775112286065_i3xthk_HE2rSFhXkAAGgfN-300x537.jpg",
        "https://cms-assets.youmind.com/media/1775112286121_ci0x0k_HE2rSMaboAAoGaX-300x537.jpg",
        "https://cms-assets.youmind.com/media/1775112286153_qoy1f1_HE2rSJoXEAAym-q-300x537.jpg"
      ],
      "language": "en",
      "translatedContent": "\"创作一张高细节、电影感、照片级真实感的生活化照片，主角是 {argument name=\"subject\" default=\"Millie Bobby Brown\"}，场景设定在傍晚的户外节日现场。她留着微卷的长发，表情俏皮，露齿微笑，正回头看向右肩，直视镜头。妆容清透自然。她头顶架着一副太阳镜，佩戴着硕大的青色水滴状耳环，右手腕戴着亮黄色的节日手环。\n\n她身材苗条匀称，身穿一件浅灰绿色的短袖露脐 T 恤，正面有褪色印花，搭配浅蓝色破洞牛仔短裤。脚穿经典的黑色高帮帆布鞋，配有白色鞋带、白色鞋头和白色鞋底。\n\n她的姿势动态十足且自信：背对镜头站立，上半身扭转，回头看向右肩。右手微微抬起，手部自然放松地垂在腰间，左臂自然下垂。重心主要落在伸直的左腿上，右腿微屈，呈现出一种随性而充满活力的姿态。\n\n背景是一个热闹的户外节日现场，类似于游乐园。背景中有一座巨大的摩天轮，轮辐和边缘闪烁着霓虹粉色和白色的灯光。背景中可以看到模糊的行人身影，地面呈现出深色，像是临时铺设的路面。\n\n光线结合了柔和的黄昏光和来自摩天轮的强烈冷色调人工环境光。霓虹灯在她的头发、肩膀和轮廓周围营造出明亮的轮廓光，而柔和的前补光则清晰地照亮了她的面部和身体。\n\n整体氛围充满活力、青春、动感且具有电影质感，捕捉到了夏季节日的热闹气息。色彩基调强调霓虹粉、亮白、浅牛仔蓝、灰绿、黑色和柔粉色调。\n\n图像应采用竖屏构图（9:16），使用略微仰拍的视角和紧凑的取景（中景或美式景别），使主体几乎占据整个画面。使用 85mm 到 135mm 之间的长焦镜头来压缩透视并突出主体，配合浅景深效果，确保主体极其清晰，同时背景保持柔和的虚化效果。最终成品应具备高细节、主体对焦锐利、色彩鲜艳且对比强烈、专业布光以及 8K 分辨率的品质。\"",
      "sourcePlatform": "twitter",
      "searchIndex": "millie bobby brown 的电影感节日摄影作品 这是一个为 nano banana 2 设计的高细节、电影感、照片级真实感的提示词，用于生成 millie bobby brown 在摩天轮附近的户外节日场景照片。提示词详细说明了人物外观、服装、动态姿势、光照条件（黄昏光线结合霓虹轮廓光）、相机设置（竖屏构图、长焦镜头、浅景深）以及整体氛围。 \"create a highly detailed, cinematic, photorealistic lifestyle photograph of {argument name=\"subject\" default=\"millie bobby brown\"} at an outdoor festival during early evening. she has long, slightly wavy hair and a playful expression, smiling with her teeth visible while looking back over her right shoulder directly at the camera. her makeup is light and natural. she is wearing sunglasses resting on top of her head, large teal-colored drop-shaped earrings, and a bright yellow festival wristband on her right wrist.\n\nshe has a slim, fit body and is dressed in a light greenish-gray cropped short-sleeve t-shirt with a faded print on the front, paired with light blue ripped denim cut-off shorts. on her feet, she wears classic black high-top canvas sneakers with white laces, a white toe cap, and white soles.\n\nher pose is dynamic and confident: she is standing with her back facing the camera, her torso twisted as she looks over her right shoulder. her right arm is slightly raised with a relaxed hand near her waist, while her left arm hangs naturally. her weight is mostly on her extended left leg, with her right leg slightly bent, creating a casual and energetic posture.\n\nthe setting is a lively outdoor festival, resembling an amusement park. in the background, there is a large ferris wheel illuminated with neon pink and white lights along its spokes and rim. blurred figures of people walking can be seen in the background, and the ground appears dark, like a temporary paved surface.\n\nthe lighting combines soft dusk light with strong, cool artificial ambient light coming from the ferris wheel. the neon lights create a glowing rim light around her hair, shoulders, and silhouette, while a soft front fill light clearly illuminates her face and body.\n\nthe overall mood is vibrant, youthful, energetic, and cinematic, capturing the lively atmosphere of a summer festival. the color palette emphasizes neon pink, bright white, light denim blue, greenish-gray, black, and soft pink tones.\n\nthe image should be captured in a portrait orientation (9:16), using a slightly low camera angle and a tight framing (medium close-up or american shot), with the subject occupying almost the entire frame. use a telephoto lens between 85mm and 135mm to compress perspective and emphasize the subject, with a shallow depth of field to keep her extremely sharp while the background remains softly blurred with bokeh. the final result should be highly detailed, with sharp focus on the subject, vibrant and contrasting colors, professional lighting, and 8k resolution quality.\" \"创作一张高细节、电影感、照片级真实感的生活化照片，主角是 {argument name=\"subject\" default=\"millie bobby brown\"}，场景设定在傍晚的户外节日现场。她留着微卷的长发，表情俏皮，露齿微笑，正回头看向右肩，直视镜头。妆容清透自然。她头顶架着一副太阳镜，佩戴着硕大的青色水滴状耳环，右手腕戴着亮黄色的节日手环。\n\n她身材苗条匀称，身穿一件浅灰绿色的短袖露脐 t 恤，正面有褪色印花，搭配浅蓝色破洞牛仔短裤。脚穿经典的黑色高帮帆布鞋，配有白色鞋带、白色鞋头和白色鞋底。\n\n她的姿势动态十足且自信：背对镜头站立，上半身扭转，回头看向右肩。右手微微抬起，手部自然放松地垂在腰间，左臂自然下垂。重心主要落在伸直的左腿上，右腿微屈，呈现出一种随性而充满活力的姿态。\n\n背景是一个热闹的户外节日现场，类似于游乐园。背景中有一座巨大的摩天轮，轮辐和边缘闪烁着霓虹粉色和白色的灯光。背景中可以看到模糊的行人身影，地面呈现出深色，像是临时铺设的路面。\n\n光线结合了柔和的黄昏光和来自摩天轮的强烈冷色调人工环境光。霓虹灯在她的头发、肩膀和轮廓周围营造出明亮的轮廓光，而柔和的前补光则清晰地照亮了她的面部和身体。\n\n整体氛围充满活力、青春、动感且具有电影质感，捕捉到了夏季节日的热闹气息。色彩基调强调霓虹粉、亮白、浅牛仔蓝、灰绿、黑色和柔粉色调。\n\n图像应采用竖屏构图（9:16），使用略微仰拍的视角和紧凑的取景（中景或美式景别），使主体几乎占据整个画面。使用 85mm 到 135mm 之间的长焦镜头来压缩透视并突出主体，配合浅景深效果，确保主体极其清晰，同时背景保持柔和的虚化效果。最终成品应具备高细节、主体对焦锐利、色彩鲜艳且对比强烈、专业布光以及 8k 分辨率的品质。\" giulia 13128",
      "likes": 0,
      "resultsCount": 0,
      "needReferenceImages": false,
      "promptCategories": []
    },
    {
      "id": 13107,
      "title": "2000 年代初风格：身着粉色西装外套的人物数码照片",
      "description": "这是一个高度具体的提示词，用于生成具有 2000 年代初数码相机风格的超写实照片。画面主体身着色彩鲜艳的超大款粉色西装外套，坐在欧洲鹅卵石街道旁的咖啡桌前。包含详细的负面提示词以确保图像质量。",
      "sourceLink": "https://x.com/Chryzleenprompt/status/2039470296743444885",
      "sourcePublishedAt": "2026-04-01T22:29:44.000Z",
      "author": {
        "name": "Chryz leen",
        "link": "https://x.com/Chryzleenprompt"
      },
      "content": "Create image: Use the attached reference image as a visual guide for the character’s appearance. Preserve key visual traits such as facial structure and overall proportions, while allowing small natural variations typical of real photography. The generated subject should clearly resemble the reference while remaining a natural photographic interpretation.\nA hyper-realistic photograph in the style of an authentic early 2000s digital photo captures the subject seated at a round marble-topped café table on a narrow European cobblestone street. The subject is positioned in a medium-wide shot, angled three-quarters toward the camera with the right elbow resting on the tabletop and the chin cradled gently in the palm of the hand. The person’s legs are crossed at the knee, with the top leg extended forward to reveal white low-top sneakers with flat cotton laces. The subject wears a vibrant, saturated {argument name=\"blazer color\" default=\"bubblegum-pink\"} oversized blazer with structured shoulders and wide notched lapels, paired with straight-leg light-wash denim jeans featuring exaggerated four-inch cuffs at the hem. A rectangular fuchsia pink clutch with a hard-shell finish rests on the table beside a small glass. The makeup aesthetic features a frosted pink lip with a high-shine gloss and a shimmering pale eyeshadow. The voluminous, long wavy hair is styled in heavy, face-framing layers. The background is a detailed urban scene with a massive stone medieval tower rising at the end of a corridor of multi-story buildings in shades of ochre and terracotta, decorated with climbing green vines and retractable canvas awnings. The image exhibits the distinct qualities of a vintage point-and-shoot camera, including a slight overexposure from a direct flash, noticeable digital noise in the shadows, and a flattened dynamic range that emphasizes the electric saturation of the pink attire against the weathered textures of the cobblestone street.\nAspect Ratio: 3:4\n\nNegative:\nextra limbs, extra hands, extra fingers, missing fingers, fused fingers, dislocated hands, broken wrists, unnatural hand pose, twisted arms, duplicated body parts, bad anatomy, incorrect anatomy, broken anatomy, impossible pose, warped body, distorted proportions, ai artifacts, poorly drawn hands, malformed anatomy",
      "media": [
        "https://cms-assets.youmind.com/media/1775112275466_xthbhz_HE2nbIfaUAAYIRK.jpg"
      ],
      "mediaThumbnails": [
        "https://cms-assets.youmind.com/media/1775112275466_xthbhz_HE2nbIfaUAAYIRK-300x402.jpg"
      ],
      "language": "en",
      "translatedContent": "创建图像：使用随附的参考图作为人物外观的视觉指南。保留面部结构和整体比例等关键视觉特征，同时允许真实摄影中常见的细微自然变化。生成的人物应清晰地呈现参考图特征，同时保持自然的摄影质感。\n\n一张具有 2000 年代初真实数码相机风格的超写实照片，捕捉了主体坐在狭窄欧洲鹅卵石街道旁大理石圆桌前的场景。主体采用中景拍摄，身体向镜头方向侧转四分之三，右肘靠在桌面上，下巴轻轻托在手掌中。双腿在膝盖处交叉，上方的腿向前伸展，露出带有扁平棉质鞋带的白色低帮运动鞋。主体身着一件色彩鲜艳、饱和度高的 {argument name=\"blazer color\" default=\"泡泡糖粉色\"} 超大款西装外套，肩部线条结构分明，配有宽大的缺角翻领，下身搭配直筒浅色牛仔裤，裤脚处有夸张的 4 英寸翻边。桌上放着一个长方形紫红色硬壳手拿包，旁边是一个小玻璃杯。妆容采用磨砂粉色唇妆，带有高光泽感，并配有闪亮的浅色眼影。丰盈的长波浪卷发采用厚重的修颜层次剪裁。背景是详细的城市景观，街道尽头矗立着一座巨大的中世纪石塔，两侧是赭色和赤陶色的多层建筑，装饰着攀爬的绿植和可伸缩的帆布遮阳棚。图像呈现出复古卡片相机的独特质感，包括直闪光灯造成的轻微过曝、阴影中明显的数码噪点，以及扁平化的动态范围，突显了粉色服装在风化鹅卵石街道纹理衬托下的电光饱和度。\n\n宽高比：3:4\n\n负面提示词：\n多余的肢体、多余的手、多余的手指、手指缺失、手指粘连、手部脱臼、手腕骨折、不自然的手部姿势、手臂扭曲、身体部位重复、糟糕的解剖结构、错误的解剖结构、破碎的解剖结构、不可能的姿势、身体变形、比例失调、AI 伪影、手部绘制拙劣、解剖结构畸形",
      "sourcePlatform": "twitter",
      "searchIndex": "2000 年代初风格：身着粉色西装外套的人物数码照片 这是一个高度具体的提示词，用于生成具有 2000 年代初数码相机风格的超写实照片。画面主体身着色彩鲜艳的超大款粉色西装外套，坐在欧洲鹅卵石街道旁的咖啡桌前。包含详细的负面提示词以确保图像质量。 create image: use the attached reference image as a visual guide for the character’s appearance. preserve key visual traits such as facial structure and overall proportions, while allowing small natural variations typical of real photography. the generated subject should clearly resemble the reference while remaining a natural photographic interpretation.\na hyper-realistic photograph in the style of an authentic early 2000s digital photo captures the subject seated at a round marble-topped café table on a narrow european cobblestone street. the subject is positioned in a medium-wide shot, angled three-quarters toward the camera with the right elbow resting on the tabletop and the chin cradled gently in the palm of the hand. the person’s legs are crossed at the knee, with the top leg extended forward to reveal white low-top sneakers with flat cotton laces. the subject wears a vibrant, saturated {argument name=\"blazer color\" default=\"bubblegum-pink\"} oversized blazer with structured shoulders and wide notched lapels, paired with straight-leg light-wash denim jeans featuring exaggerated four-inch cuffs at the hem. a rectangular fuchsia pink clutch with a hard-shell finish rests on the table beside a small glass. the makeup aesthetic features a frosted pink lip with a high-shine gloss and a shimmering pale eyeshadow. the voluminous, long wavy hair is styled in heavy, face-framing layers. the background is a detailed urban scene with a massive stone medieval tower rising at the end of a corridor of multi-story buildings in shades of ochre and terracotta, decorated with climbing green vines and retractable canvas awnings. the image exhibits the distinct qualities of a vintage point-and-shoot camera, including a slight overexposure from a direct flash, noticeable digital noise in the shadows, and a flattened dynamic range that emphasizes the electric saturation of the pink attire against the weathered textures of the cobblestone street.\naspect ratio: 3:4\n\nnegative:\nextra limbs, extra hands, extra fingers, missing fingers, fused fingers, dislocated hands, broken wrists, unnatural hand pose, twisted arms, duplicated body parts, bad anatomy, incorrect anatomy, broken anatomy, impossible pose, warped body, distorted proportions, ai artifacts, poorly drawn hands, malformed anatomy 创建图像：使用随附的参考图作为人物外观的视觉指南。保留面部结构和整体比例等关键视觉特征，同时允许真实摄影中常见的细微自然变化。生成的人物应清晰地呈现参考图特征，同时保持自然的摄影质感。\n\n一张具有 2000 年代初真实数码相机风格的超写实照片，捕捉了主体坐在狭窄欧洲鹅卵石街道旁大理石圆桌前的场景。主体采用中景拍摄，身体向镜头方向侧转四分之三，右肘靠在桌面上，下巴轻轻托在手掌中。双腿在膝盖处交叉，上方的腿向前伸展，露出带有扁平棉质鞋带的白色低帮运动鞋。主体身着一件色彩鲜艳、饱和度高的 {argument name=\"blazer color\" default=\"泡泡糖粉色\"} 超大款西装外套，肩部线条结构分明，配有宽大的缺角翻领，下身搭配直筒浅色牛仔裤，裤脚处有夸张的 4 英寸翻边。桌上放着一个长方形紫红色硬壳手拿包，旁边是一个小玻璃杯。妆容采用磨砂粉色唇妆，带有高光泽感，并配有闪亮的浅色眼影。丰盈的长波浪卷发采用厚重的修颜层次剪裁。背景是详细的城市景观，街道尽头矗立着一座巨大的中世纪石塔，两侧是赭色和赤陶色的多层建筑，装饰着攀爬的绿植和可伸缩的帆布遮阳棚。图像呈现出复古卡片相机的独特质感，包括直闪光灯造成的轻微过曝、阴影中明显的数码噪点，以及扁平化的动态范围，突显了粉色服装在风化鹅卵石街道纹理衬托下的电光饱和度。\n\n宽高比：3:4\n\n负面提示词：\n多余的肢体、多余的手、多余的手指、手指缺失、手指粘连、手部脱臼、手腕骨折、不自然的手部姿势、手臂扭曲、身体部位重复、糟糕的解剖结构、错误的解剖结构、破碎的解剖结构、不可能的姿势、身体变形、比例失调、ai 伪影、手部绘制拙劣、解剖结构畸形 chryz leen 13107",
      "likes": 0,
      "resultsCount": 0,
      "needReferenceImages": true,
      "promptCategories": []
    },
    {
      "id": 13106,
      "title": "暴雨中老人的黑白肖像",
      "description": "这是一条详细的提示词，用于生成一张高对比度、质感粗犷的黑白特写肖像，描绘一位被暴雨淋透的老人。该提示词采用了特定的相机和胶片模拟设置（Leica M11, Ilford HP5），并呈现 Sebastiao Salgado 的摄影风格。",
      "sourceLink": "https://x.com/kaanakz/status/2039467371111284817",
      "sourcePublishedAt": "2026-04-01T22:18:07.000Z",
      "author": {
        "name": "Kaan",
        "link": "https://x.com/kaanakz"
      },
      "content": "Weathered elderly male, wire-rimmed spectacles, drenched skin and hair, moisture-slicked jacket, macro texture, intense gaze, tight close-up, clinging water droplets, heavy downpour, blurred rain streaks, pitch-black atmosphere, high-contrast monochrome, chiaroscuro lighting, specular highlights, Leica M11 Monochrom, 50mm f/1.4, Ilford HP5 grain, pushed +2 stops, high micro-contrast, Sebastiao Salgado style, gritty realism, 1:1.",
      "media": [
        "https://cms-assets.youmind.com/media/1775112275139_e2kqtr_HE2i-k6aoAAIqoy.jpg",
        "https://cms-assets.youmind.com/media/1775112275227_rj08vu_HE2jDWgWwAAS2f9.jpg"
      ],
      "mediaThumbnails": [
        "https://cms-assets.youmind.com/media/1775112275139_e2kqtr_HE2i-k6aoAAIqoy-300x300.jpg",
        "https://cms-assets.youmind.com/media/1775112275227_rj08vu_HE2jDWgWwAAS2f9-300x300.jpg"
      ],
      "language": "en",
      "translatedContent": "饱经风霜的老年男性，佩戴金属框眼镜，皮肤和头发被雨水淋透，外套上满是湿润的痕迹，宏观纹理，目光深邃，紧凑特写，附着的水滴，倾盆大雨，模糊的雨丝，漆黑的氛围，高对比度黑白，明暗对照法照明，镜面高光，Leica M11 Monochrom，50mm f/1.4，Ilford HP5 颗粒感，增感 +2 档，高微对比度，Sebastiao Salgado 风格，粗犷写实，1:1。",
      "sourcePlatform": "twitter",
      "searchIndex": "暴雨中老人的黑白肖像 这是一条详细的提示词，用于生成一张高对比度、质感粗犷的黑白特写肖像，描绘一位被暴雨淋透的老人。该提示词采用了特定的相机和胶片模拟设置（leica m11, ilford hp5），并呈现 sebastiao salgado 的摄影风格。 weathered elderly male, wire-rimmed spectacles, drenched skin and hair, moisture-slicked jacket, macro texture, intense gaze, tight close-up, clinging water droplets, heavy downpour, blurred rain streaks, pitch-black atmosphere, high-contrast monochrome, chiaroscuro lighting, specular highlights, leica m11 monochrom, 50mm f/1.4, ilford hp5 grain, pushed +2 stops, high micro-contrast, sebastiao salgado style, gritty realism, 1:1. 饱经风霜的老年男性，佩戴金属框眼镜，皮肤和头发被雨水淋透，外套上满是湿润的痕迹，宏观纹理，目光深邃，紧凑特写，附着的水滴，倾盆大雨，模糊的雨丝，漆黑的氛围，高对比度黑白，明暗对照法照明，镜面高光，leica m11 monochrom，50mm f/1.4，ilford hp5 颗粒感，增感 +2 档，高微对比度，sebastiao salgado 风格，粗犷写实，1:1。 kaan 13106",
      "likes": 0,
      "resultsCount": 0,
      "needReferenceImages": true,
      "promptCategories": []
    },
    {
      "id": 13133,
      "title": "Nano Banana 2 的图表风格提示词",
      "description": "一套包含五种风格模板的提示词，用于通过 Nano Banana 2 生成稳定且结构化的图表。用户可以替换括号内的内容，以创建不同类型的视觉辅助工具，例如菜单式列表、处方单、登机牌、仪表盘 UI 以及诊断流程图。",
      "sourceLink": "https://x.com/ChatgptAIskill/status/2039448975426879733",
      "sourcePublishedAt": "2026-04-01T21:05:01.000Z",
      "author": {
        "name": "ケンイチ | AIスキルアカデミー『誰でもわかるAI活用術』",
        "link": "https://x.com/ChatgptAIskill"
      },
      "content": "① お品書き風\n「{argument name=\"副業スキル\" default=\"副業で稼げるスキル\"}厳選5品」を和食料亭のお品書き風の図解にして。縦書き・毛筆・和紙風の背景で。\n\n② 処方箋風\n「{argument name=\"AIの使い方\" default=\"AIの正しい使い方\"}」を医療の処方箋風の図解にして。Rxマーク・白背景・医療ブルーで。\n\n③ 搭乗券風\n「{argument name=\"出発地\" default=\"AIなし生活\"}→{argument name=\"目的地\" default=\"AI活用後の生活\"}」を航空搭乗券風の図解にして。出発地・到着地・バーコード付きで。\n\n④ ダッシュボードUI風\n「{argument name=\"比較対象\" default=\"ChatGPT vs Claude\"}性能比較」をダークモードのダッシュボードUI風の図解にして。KPIカード・グラフ付きで。\n\n⑤ 診断チャート風\n「あなたに合う{argument name=\"AIツール\" default=\"AIツール\"}診断」をYes/No分岐のフローチャート風の図解にして。矢印・色分けノード付きで。",
      "media": [
        "https://cms-assets.youmind.com/media/1775112288342_6nv3zt_HE2QnibbMAAf3cI.jpg"
      ],
      "mediaThumbnails": [
        "https://cms-assets.youmind.com/media/1775112288342_6nv3zt_HE2QnibbMAAf3cI-300x400.jpg"
      ],
      "language": "ja",
      "translatedContent": "① 菜单风格\n以日式餐厅菜单的风格，创建一个关于“5 个精选 {argument name=\"side job skills\" default=\"副业技能\"}”的图表。使用竖排文字、笔触效果以及类似和纸的背景。\n\n② 处方单风格\n以医疗处方单的风格，创建一个关于“{argument name=\"AI usage\" default=\"AI 的正确使用方法\"}”的图表。包含 Rx 标志、白色背景以及医疗蓝配色。\n\n③ 登机牌风格\n以航空公司登机牌的风格，创建一个关于“{argument name=\"departure point\" default=\"没有 AI 的生活\"} → {argument name=\"destination\" default=\"使用 AI 后的生活\"}”的图表。包含出发地、目的地以及条形码。\n\n④ 仪表盘 UI 风格\n以深色模式仪表盘 UI 的风格，创建一个对比“{argument name=\"comparison subjects\" default=\"ChatGPT 与 Claude\"}”性能的图表。包含 KPI 卡片和图表。\n\n⑤ 诊断流程图风格\n以“是/否”分支流程图的风格，为“{argument name=\"AI tool\" default=\"AI 工具\"}”创建一个适合你的诊断图表。包含箭头和颜色编码的节点。",
      "sourcePlatform": "twitter",
      "searchIndex": "nano banana 2 的图表风格提示词 一套包含五种风格模板的提示词，用于通过 nano banana 2 生成稳定且结构化的图表。用户可以替换括号内的内容，以创建不同类型的视觉辅助工具，例如菜单式列表、处方单、登机牌、仪表盘 ui 以及诊断流程图。 ① お品書き風\n「{argument name=\"副業スキル\" default=\"副業で稼げるスキル\"}厳選5品」を和食料亭のお品書き風の図解にして。縦書き・毛筆・和紙風の背景で。\n\n② 処方箋風\n「{argument name=\"aiの使い方\" default=\"aiの正しい使い方\"}」を医療の処方箋風の図解にして。rxマーク・白背景・医療ブルーで。\n\n③ 搭乗券風\n「{argument name=\"出発地\" default=\"aiなし生活\"}→{argument name=\"目的地\" default=\"ai活用後の生活\"}」を航空搭乗券風の図解にして。出発地・到着地・バーコード付きで。\n\n④ ダッシュボードui風\n「{argument name=\"比較対象\" default=\"chatgpt vs claude\"}性能比較」をダークモードのダッシュボードui風の図解にして。kpiカード・グラフ付きで。\n\n⑤ 診断チャート風\n「あなたに合う{argument name=\"aiツール\" default=\"aiツール\"}診断」をyes/no分岐のフローチャート風の図解にして。矢印・色分けノード付きで。 ① 菜单风格\n以日式餐厅菜单的风格，创建一个关于“5 个精选 {argument name=\"side job skills\" default=\"副业技能\"}”的图表。使用竖排文字、笔触效果以及类似和纸的背景。\n\n② 处方单风格\n以医疗处方单的风格，创建一个关于“{argument name=\"ai usage\" default=\"ai 的正确使用方法\"}”的图表。包含 rx 标志、白色背景以及医疗蓝配色。\n\n③ 登机牌风格\n以航空公司登机牌的风格，创建一个关于“{argument name=\"departure point\" default=\"没有 ai 的生活\"} → {argument name=\"destination\" default=\"使用 ai 后的生活\"}”的图表。包含出发地、目的地以及条形码。\n\n④ 仪表盘 ui 风格\n以深色模式仪表盘 ui 的风格，创建一个对比“{argument name=\"comparison subjects\" default=\"chatgpt 与 claude\"}”性能的图表。包含 kpi 卡片和图表。\n\n⑤ 诊断流程图风格\n以“是/否”分支流程图的风格，为“{argument name=\"ai tool\" default=\"ai 工具\"}”创建一个适合你的诊断图表。包含箭头和颜色编码的节点。 ケンイチ | aiスキルアカデミー『誰でもわかるai活用術』 13133",
      "likes": 0,
      "resultsCount": 0,
      "needReferenceImages": false,
      "promptCategories": []
    },
    {
      "id": 13095,
      "title": "Anya Taylor Joy 风格 iPhone 杂志大片特写",
      "description": "这是一个为 Google Nano Banana Pro 设计的结构化提示词，旨在生成一张由 Anya Taylor Joy 启发、超写实的高级时装杂志风格特写肖像。图像模拟 iPhone 17 Pro 的拍摄效果，利用强烈的侧面日光，营造出锐利的阴影并突显雕塑般的面部轮廓，传达出一种冷峻、克制且神秘的氛围。",
      "sourceLink": "https://x.com/bananababydoll/status/2039437142800417169",
      "sourcePublishedAt": "2026-04-01T20:18:00.000Z",
      "author": {
        "name": "babydoll",
        "link": "https://x.com/bananababydoll"
      },
      "content": "{\n  \"meta\": {\n    \"camera\": \"iPhone 17 Pro\",\n    \"lens\": \"35mm\",\n    \"aspect_ratio\": \"9:16\",\n    \"quality\": \"ultra photorealistic\",\n    \"style\": \"iphone editorial close-up, natural grain, soft cinematic shadows\"\n  },\n\n  \"subject\": {\n    \"identity\": \"{argument name=\"subject identity\" default=\"anya taylor joy inspired\"}\",\n    \"face\": {\n      \"structure\": \"elongated oval face, high cheekbones, wide-set eyes, delicate jawline\",\n      \"skin\": \"porcelain pale, smooth but realistic texture\",\n      \"expression\": \"stoic, distant, slightly mysterious\"\n    },\n    \"hair\": {\n      \"color\": \"platinum blonde\",\n      \"style\": \"long straight hair, slightly tucked behind ears, soft strands framing face\"\n    },\n    \"body\": {\n      \"type\": \"slender high-fashion physique\",\n      \"presence\": \"elegant, elongated lines\"\n    }\n  },\n\n  \"scene\": {\n    \"location\": \"minimalist interior wall\",\n    \"time\": \"late afternoon\",\n    \"atmosphere\": \"quiet, still, almost surreal\"\n  },\n\n  \"lighting\": {\n    \"type\": \"strong side daylight\",\n    \"effect\": \"sharp light slicing across face, deep shadow on one side, sculpted cheekbones\"\n  },\n\n  \"camera\": {\n    \"perspective\": \"close-up portrait\",\n    \"angle\": \"slightly low angle\",\n    \"distance\": \"face and upper torso\",\n    \"imperfection\": \"slight exposure imbalance, realistic iPhone HDR\"\n  },\n\n  \"outfit\": {\n    \"top\": \"minimal thin white top\",\n    \"details\": \"clean fabric, subtle tension, no distraction\"\n  },\n\n  \"vibe\": {\n    \"energy\": \"cold beauty, controlled presence\",\n    \"mood\": \"editorial stillness\",\n    \"aesthetic\": \"high fashion but captured casually\"\n  }\n}",
      "media": [
        "https://cms-assets.youmind.com/media/1775112268498_8848lv_HEad8MFX0AA57iH.jpg",
        "https://cms-assets.youmind.com/media/1775112268502_9snnl6_HEad8MEXYAAzLrZ.jpg"
      ],
      "mediaThumbnails": [
        "https://cms-assets.youmind.com/media/1775112268498_8848lv_HEad8MFX0AA57iH-300x537.jpg",
        "https://cms-assets.youmind.com/media/1775112268502_9snnl6_HEad8MEXYAAzLrZ-300x537.jpg"
      ],
      "language": "en",
      "translatedContent": "{\n  \"meta\": {\n    \"camera\": \"iPhone 17 Pro\",\n    \"lens\": \"35mm\",\n    \"aspect_ratio\": \"9:16\",\n    \"quality\": \"超写实\",\n    \"style\": \"iPhone 杂志大片特写，自然颗粒感，柔和的电影感阴影\"\n  },\n\n  \"subject\": {\n    \"identity\": \"{argument name=\\\"subject identity\\\" default=\\\"Anya Taylor Joy 风格\\\"}\",\n    \"face\": {\n      \"structure\": \"修长的椭圆形脸，高颧骨，宽眼距，精致的下颌线\",\n      \"skin\": \"瓷白肤色，平滑但具有真实的纹理\",\n      \"expression\": \"冷漠，疏离，带有一丝神秘感\"\n    },\n    \"hair\": {\n      \"color\": \"铂金色\",\n      \"style\": \"长直发，微微别在耳后，柔和的发丝修饰脸型\"\n    },\n    \"body\": {\n      \"type\": \"纤细的高级时装模特体型\",\n      \"presence\": \"优雅，线条修长\"\n    }\n  },\n\n  \"scene\": {\n    \"location\": \"极简主义室内墙面\",\n    \"time\": \"午后\",\n    \"atmosphere\": \"安静，静止，近乎超现实\"\n  },\n\n  \"lighting\": {\n    \"type\": \"强烈的侧面日光\",\n    \"effect\": \"锐利的光线横切过面部，一侧呈现深邃阴影，突显雕塑般的颧骨\"\n  },\n\n  \"camera\": {\n    \"perspective\": \"特写肖像\",\n    \"angle\": \"微仰角\",\n    \"distance\": \"面部及上半身\",\n    \"imperfection\": \"轻微的曝光不平衡，真实的 iPhone HDR 效果\"\n  },\n\n  \"outfit\": {\n    \"top\": \"极简白色薄上衣\",\n    \"details\": \"干净的面料，微妙的褶皱，无干扰元素\"\n  },\n\n  \"vibe\": {\n    \"energy\": \"冷艳，克制的表现力\",\n    \"mood\": \"杂志般的静谧感\",\n    \"aesthetic\": \"高级时装感，但以随手拍的方式呈现\"\n  }\n}",
      "sourcePlatform": "twitter",
      "searchIndex": "anya taylor joy 风格 iphone 杂志大片特写 这是一个为 google nano banana pro 设计的结构化提示词，旨在生成一张由 anya taylor joy 启发、超写实的高级时装杂志风格特写肖像。图像模拟 iphone 17 pro 的拍摄效果，利用强烈的侧面日光，营造出锐利的阴影并突显雕塑般的面部轮廓，传达出一种冷峻、克制且神秘的氛围。 {\n  \"meta\": {\n    \"camera\": \"iphone 17 pro\",\n    \"lens\": \"35mm\",\n    \"aspect_ratio\": \"9:16\",\n    \"quality\": \"ultra photorealistic\",\n    \"style\": \"iphone editorial close-up, natural grain, soft cinematic shadows\"\n  },\n\n  \"subject\": {\n    \"identity\": \"{argument name=\"subject identity\" default=\"anya taylor joy inspired\"}\",\n    \"face\": {\n      \"structure\": \"elongated oval face, high cheekbones, wide-set eyes, delicate jawline\",\n      \"skin\": \"porcelain pale, smooth but realistic texture\",\n      \"expression\": \"stoic, distant, slightly mysterious\"\n    },\n    \"hair\": {\n      \"color\": \"platinum blonde\",\n      \"style\": \"long straight hair, slightly tucked behind ears, soft strands framing face\"\n    },\n    \"body\": {\n      \"type\": \"slender high-fashion physique\",\n      \"presence\": \"elegant, elongated lines\"\n    }\n  },\n\n  \"scene\": {\n    \"location\": \"minimalist interior wall\",\n    \"time\": \"late afternoon\",\n    \"atmosphere\": \"quiet, still, almost surreal\"\n  },\n\n  \"lighting\": {\n    \"type\": \"strong side daylight\",\n    \"effect\": \"sharp light slicing across face, deep shadow on one side, sculpted cheekbones\"\n  },\n\n  \"camera\": {\n    \"perspective\": \"close-up portrait\",\n    \"angle\": \"slightly low angle\",\n    \"distance\": \"face and upper torso\",\n    \"imperfection\": \"slight exposure imbalance, realistic iphone hdr\"\n  },\n\n  \"outfit\": {\n    \"top\": \"minimal thin white top\",\n    \"details\": \"clean fabric, subtle tension, no distraction\"\n  },\n\n  \"vibe\": {\n    \"energy\": \"cold beauty, controlled presence\",\n    \"mood\": \"editorial stillness\",\n    \"aesthetic\": \"high fashion but captured casually\"\n  }\n} {\n  \"meta\": {\n    \"camera\": \"iphone 17 pro\",\n    \"lens\": \"35mm\",\n    \"aspect_ratio\": \"9:16\",\n    \"quality\": \"超写实\",\n    \"style\": \"iphone 杂志大片特写，自然颗粒感，柔和的电影感阴影\"\n  },\n\n  \"subject\": {\n    \"identity\": \"{argument name=\\\"subject identity\\\" default=\\\"anya taylor joy 风格\\\"}\",\n    \"face\": {\n      \"structure\": \"修长的椭圆形脸，高颧骨，宽眼距，精致的下颌线\",\n      \"skin\": \"瓷白肤色，平滑但具有真实的纹理\",\n      \"expression\": \"冷漠，疏离，带有一丝神秘感\"\n    },\n    \"hair\": {\n      \"color\": \"铂金色\",\n      \"style\": \"长直发，微微别在耳后，柔和的发丝修饰脸型\"\n    },\n    \"body\": {\n      \"type\": \"纤细的高级时装模特体型\",\n      \"presence\": \"优雅，线条修长\"\n    }\n  },\n\n  \"scene\": {\n    \"location\": \"极简主义室内墙面\",\n    \"time\": \"午后\",\n    \"atmosphere\": \"安静，静止，近乎超现实\"\n  },\n\n  \"lighting\": {\n    \"type\": \"强烈的侧面日光\",\n    \"effect\": \"锐利的光线横切过面部，一侧呈现深邃阴影，突显雕塑般的颧骨\"\n  },\n\n  \"camera\": {\n    \"perspective\": \"特写肖像\",\n    \"angle\": \"微仰角\",\n    \"distance\": \"面部及上半身\",\n    \"imperfection\": \"轻微的曝光不平衡，真实的 iphone hdr 效果\"\n  },\n\n  \"outfit\": {\n    \"top\": \"极简白色薄上衣\",\n    \"details\": \"干净的面料，微妙的褶皱，无干扰元素\"\n  },\n\n  \"vibe\": {\n    \"energy\": \"冷艳，克制的表现力\",\n    \"mood\": \"杂志般的静谧感\",\n    \"aesthetic\": \"高级时装感，但以随手拍的方式呈现\"\n  }\n} babydoll 13095",
      "likes": 0,
      "resultsCount": 0,
      "needReferenceImages": false,
      "promptCategories": []
    },
    {
      "id": 13115,
      "title": "由切片果蔬重构的角色",
      "description": "这是一个为 Nano Banana Pro 设计的详细提示词，旨在将上传的角色参考图转换为完全由层叠、切片果蔬构成的逼真 3D 渲染图。在确保所有材质（包括鞋履）均由果蔬切片构成的前提下，精准保留原角色的比例、色彩和特征。",
      "sourceLink": "https://x.com/edizkan_/status/2039431574916104474",
      "sourcePublishedAt": "2026-04-01T19:55:52.000Z",
      "author": {
        "name": "Edizkan ⭕🦇",
        "link": "https://x.com/edizkan_"
      },
      "content": "create a version of this character made entirely of sliced vegetables and fruits vegetables and fruits: should match the uploaded character’s colors as closely as possible, without changing their natural look slices: slightly misaligned, with visible separation between layers structure: a small bite taken out of the top of the head, forming a clean cut surface ambience: kitchen setting, on a cutting board with scattered vegetables and a knife nearby camera: slightly top-down angle effect: tilt-shift  Likeness Preservation (CRITICAL): Maintain exact proportions, eye shape, spacing, stylization, silhouette, and material identity from the reference Do not humanize Do not reinterpret anatomy Do not modify facial structure Identity must remain perfectly intact and instantly recognizable Character Compatibility Rules: If the uploaded character is 2D, convert it into a believable realistic 3D render while preserving the original design, proportions, colors, and visual identity Do not modify facial structure  Do not add a mouth or nose if the uploaded character does not have them  👟 FOOTWEAR & ACCESSORY MATERIAL OVERRIDE (CRITICAL FIX) All parts of the character, including footwear and clothing, must follow the same fruit and vegetable slice construction. • shoes must be made entirely of stacked fruit and vegetable slices  • preserve the exact shoe shape, proportions, and design  • recreate panels, sole, and details using layered slices Material examples for shoes: • sole → dense stacked potato or root vegetable slices  • upper panels → fruit/vegetable slices arranged to match original color blocking  • edges and seams → visible slice layering and separation 📷⚠️ 0px; letter-spacing: normal; orphans: 2; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px;\"> IMPORTANT:  No original materials should remain (no fabric, rubber, or leather).  Everything must be converted into fruit/vegetable slice construction while keeping the original design recognizable. ar 1:1",
      "media": [
        "https://cms-assets.youmind.com/media/1775112279816_ougqwk_HE2DyO3WwAAhQtr.jpg"
      ],
      "mediaThumbnails": [
        "https://cms-assets.youmind.com/media/1775112279816_ougqwk_HE2DyO3WwAAhQtr-300x300.jpg"
      ],
      "language": "en",
      "translatedContent": "创建该角色的一个版本，完全由切片蔬菜和水果构成。蔬菜和水果：应尽可能与上传角色的颜色匹配，且不改变其自然外观。切片：略微错位，层与层之间有明显的间隙。结构：头顶被咬掉一小块，形成一个干净的切面。环境：厨房场景，位于项目上，周围散落着蔬菜和一把刀。相机：略微俯视的角度。效果：移轴摄影。 相似度保留（关键）：严格保持参考图中的精确比例、眼睛形状、间距、风格化处理、轮廓和材质特征。不要将其拟人化。不要重新诠释解剖结构。不要修改面部结构。角色身份必须保持完整且一眼可辨。 角色兼容性规则：如果上传的角色是 2D 的，请将其转换为可信的逼真 3D 渲染，同时保留原始设计、比例、颜色和视觉特征。不要修改面部结构。如果上传的角色没有嘴巴或鼻子，请勿添加。 👟 鞋履与配饰材质覆盖（关键修正） 角色的所有部分，包括鞋履和服装，必须遵循相同的果蔬切片构造。 • 鞋子必须完全由堆叠的果蔬切片制成。 • 保留精确的鞋型、比例和设计。 • 使用层叠切片重现鞋面、鞋底和细节。 鞋子材质示例： • 鞋底 → 紧密堆叠的土豆或根茎类蔬菜切片。 • 鞋面拼接 → 按照原始色块排列的果蔬切片。 • 边缘和接缝 → 可见的切片层叠和缝隙。 📷⚠️ 0px; letter-spacing: normal; orphans: 2; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px;\"> 重要提示：不得保留任何原始材质（无织物、橡胶或皮革）。所有内容必须转换为果蔬切片构造，同时确保原始设计可被识别。 ar 1:1",
      "sourcePlatform": "twitter",
      "searchIndex": "由切片果蔬重构的角色 这是一个为 nano banana pro 设计的详细提示词，旨在将上传的角色参考图转换为完全由层叠、切片果蔬构成的逼真 3d 渲染图。在确保所有材质（包括鞋履）均由果蔬切片构成的前提下，精准保留原角色的比例、色彩和特征。 create a version of this character made entirely of sliced vegetables and fruits vegetables and fruits: should match the uploaded character’s colors as closely as possible, without changing their natural look slices: slightly misaligned, with visible separation between layers structure: a small bite taken out of the top of the head, forming a clean cut surface ambience: kitchen setting, on a cutting board with scattered vegetables and a knife nearby camera: slightly top-down angle effect: tilt-shift  likeness preservation (critical): maintain exact proportions, eye shape, spacing, stylization, silhouette, and material identity from the reference do not humanize do not reinterpret anatomy do not modify facial structure identity must remain perfectly intact and instantly recognizable character compatibility rules: if the uploaded character is 2d, convert it into a believable realistic 3d render while preserving the original design, proportions, colors, and visual identity do not modify facial structure  do not add a mouth or nose if the uploaded character does not have them  👟 footwear & accessory material override (critical fix) all parts of the character, including footwear and clothing, must follow the same fruit and vegetable slice construction. • shoes must be made entirely of stacked fruit and vegetable slices  • preserve the exact shoe shape, proportions, and design  • recreate panels, sole, and details using layered slices material examples for shoes: • sole → dense stacked potato or root vegetable slices  • upper panels → fruit/vegetable slices arranged to match original color blocking  • edges and seams → visible slice layering and separation 📷⚠️ 0px; letter-spacing: normal; orphans: 2; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px;\"> important:  no original materials should remain (no fabric, rubber, or leather).  everything must be converted into fruit/vegetable slice construction while keeping the original design recognizable. ar 1:1 创建该角色的一个版本，完全由切片蔬菜和水果构成。蔬菜和水果：应尽可能与上传角色的颜色匹配，且不改变其自然外观。切片：略微错位，层与层之间有明显的间隙。结构：头顶被咬掉一小块，形成一个干净的切面。环境：厨房场景，位于项目上，周围散落着蔬菜和一把刀。相机：略微俯视的角度。效果：移轴摄影。 相似度保留（关键）：严格保持参考图中的精确比例、眼睛形状、间距、风格化处理、轮廓和材质特征。不要将其拟人化。不要重新诠释解剖结构。不要修改面部结构。角色身份必须保持完整且一眼可辨。 角色兼容性规则：如果上传的角色是 2d 的，请将其转换为可信的逼真 3d 渲染，同时保留原始设计、比例、颜色和视觉特征。不要修改面部结构。如果上传的角色没有嘴巴或鼻子，请勿添加。 👟 鞋履与配饰材质覆盖（关键修正） 角色的所有部分，包括鞋履和服装，必须遵循相同的果蔬切片构造。 • 鞋子必须完全由堆叠的果蔬切片制成。 • 保留精确的鞋型、比例和设计。 • 使用层叠切片重现鞋面、鞋底和细节。 鞋子材质示例： • 鞋底 → 紧密堆叠的土豆或根茎类蔬菜切片。 • 鞋面拼接 → 按照原始色块排列的果蔬切片。 • 边缘和接缝 → 可见的切片层叠和缝隙。 📷⚠️ 0px; letter-spacing: normal; orphans: 2; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px;\"> 重要提示：不得保留任何原始材质（无织物、橡胶或皮革）。所有内容必须转换为果蔬切片构造，同时确保原始设计可被识别。 ar 1:1 edizkan ⭕🦇 13115",
      "likes": 0,
      "resultsCount": 0,
      "needReferenceImages": true,
      "promptCategories": []
    }
  ],
  "total": 12222,
  "page": 1,
  "limit": 18,
  "totalPages": 679,
  "hasMore": true
}
```
