# OpenRouter 图片生成 - 参考图用法

## 模型

`google/gemini-3.1-flash-image-preview` 支持图片输入（input modalities 包含 image + text），可将输入图片作为参考进行图像生成或编辑。

## 使用方式

将 `content` 从字符串改为数组，同时传入文字 prompt 和参考图：

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
            "text": "Generate a beautiful sunset over mountains in the same style as this image"
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

## 图片格式

支持公开 URL 或 base64 编码：

```json
{
  "type": "image_url",
  "image_url": {
    "url": "data:image/jpeg;base64,{encoded_data}"
  }
}
```

支持的格式：`image/png`、`image/jpeg`、`image/webp`、`image/gif`

## 注意事项

- text 放在 image 前面（OpenRouter 官方建议）
- 多张参考图可以放多个 `image_url` 条目，上限因模型而异
- `super_resolution_references`（参考图增强）仅支持 Sourceful 模型，不适用于 Gemini

## 参考文档

- [OpenRouter Image Inputs](https://openrouter.ai/docs/guides/overview/multimodal/images)
- [OpenRouter Image Generation](https://openrouter.ai/docs/guides/overview/multimodal/image-generation)
