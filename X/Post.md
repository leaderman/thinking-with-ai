# X 推文类型研究

## 研究目标

通过 CDP 拦截接口的方式，打开单条推文的页面，获取推文数据并区分推文类型。

## 拦截的接口

打开单条推文页面后，X 会同时触发两个接口：

```
https://x.com/i/api/graphql/{hash}/TweetDetail?variables=...
https://x.com/i/api/graphql/{hash}/TweetResultByRestId?variables=...
```

两个接口返回的推文核心数据结构相同。`TweetDetail` 返回完整会话（含回复），`TweetResultByRestId` 只返回该推文本身。

## TweetDetail 数据路径

```
data
└── threaded_conversation_with_injections_v2
    └── instructions[]
        └── [type: "TimelineAddEntries"]
            └── entries[]
                └── content
                    └── itemContent
                        └── tweet_results
                            └── result
```

## 推文类型区分

三种类型均通过同一接口返回，区分依据在于 `result`（或 `result.tweet`）中的字段：

### 纯文本推文

- `extended_entities` 不存在或 `extended_entities.media` 为空
- 不含 `article` 字段

```
result
├── legacy
│   ├── full_text        ← 推文正文
│   └── entities.media   ← 不存在或为空数组
└── (无 article 字段)
```

示例：`https://x.com/thedankoe/status/2038648359817482251`

---

### 图文推文（文本 + 图片）

- `legacy.extended_entities.media` 中包含媒体项
- `media[].type` = `"photo"`
- `media_url_https` 即图片地址

```
result
└── legacy
    ├── full_text
    └── extended_entities
        └── media[]
            ├── type              ← "photo"
            └── media_url_https   ← 图片 URL
```

示例：`https://x.com/thedankoe/status/1903846467141521502`

---

### 视频推文（文本 + 视频）

- `legacy.extended_entities.media` 中包含媒体项
- `media[].type` = `"video"` 或 `"animated_gif"`
- `media_url_https` 是视频封面图，视频本身在 `video_info.variants[]` 中，按码率列出多个地址

```
result
└── legacy
    ├── full_text
    └── extended_entities
        └── media[]
            ├── type                       ← "video" | "animated_gif"
            ├── media_url_https            ← 封面图 URL
            └── video_info
                ├── duration_millis        ← 视频时长（毫秒）
                └── variants[]
                    ├── content_type       ← "video/mp4" | "application/x-mpegURL"
                    ├── bitrate            ← 码率（bps）
                    └── url                ← 视频地址
```

示例：`https://x.com/thedankoe/status/2031765325654716480`

---

### 文章推文（Article）

- `result` 中存在 `article` 字段
- `legacy.full_text` 只有一个 `t.co` 短链，正文内容在 `article` 里
- 文章正文以 Draft.js 的 `blocks` 格式存储，每个 block 是一个段落

```
result
├── legacy
│   └── full_text        ← 仅含 t.co 短链，非正文
└── article
    └── article_results
        └── result
            └── content_state
                └── blocks[]
                    ├── text    ← 段落文本
                    └── type    ← "unstyled" | "header-one" | "header-two" | ...
```

示例：`https://x.com/thedankoe/status/2036824811712942576`

## 类型判断逻辑

```js
function getTweetType(tweet) {
  if (tweet.article) return 'article';
  const media = tweet.legacy?.extended_entities?.media;
  if (media && media.length > 0) {
    const type = media[0].type;
    if (type === 'video' || type === 'animated_gif') return 'video';
    return 'photo';
  }
  return 'text';
}
```

## 可行性结论

**完全可行。** 四种类型的推文均通过同一个 `TweetDetail` 接口返回，数据在页面加载时自动触发，无需额外操作。区分逻辑：先看 `article` 字段，再看 `extended_entities.media[].type`，无媒体则为纯文本。
