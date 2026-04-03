# 微信稳定版 Access Token 获取

## 背景

微信提供了一个独立的稳定版 access_token 接口，与标准 `getAccessToken` 完全隔离，适合对 token 稳定性要求较高的场景。

文档地址：https://developers.weixin.qq.com/doc/service/api/base/api_getstableaccesstoken.html

## 接口信息

- **请求方式**：POST
- **URL**：`https://api.weixin.qq.com/cgi-bin/stable_token`
- **频率限制**：每分钟最多 10,000 次

## 请求参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| grant_type | string | 是 | 固定值 `client_credential` |
| appid | string | 是 | 公众号/小程序的 AppID |
| secret | string | 是 | AppSecret |
| force_refresh | boolean | 否 | 是否强制刷新，默认 false |

## 调用示例

### 普通获取

```bash
curl -X POST https://api.weixin.qq.com/cgi-bin/stable_token \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "client_credential",
    "appid": "YOUR_APPID",
    "secret": "YOUR_APPSECRET"
  }'
```

### 强制刷新

```bash
curl -X POST https://api.weixin.qq.com/cgi-bin/stable_token \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "client_credential",
    "appid": "YOUR_APPID",
    "secret": "YOUR_APPSECRET",
    "force_refresh": true
  }'
```

## 响应示例

```json
{
  "access_token": "ACCESS_TOKEN",
  "expires_in": 7200
}
```

## 注意事项

- token 有效期 **7200 秒**（2小时）
- 普通模式下，token 未过期时重复调用会返回同一个 token（稳定性保证）
- `force_refresh` 每天限制 **20 次**，且两次调用间隔不少于 **30 秒**
- 此接口与标准 `getAccessToken` **完全隔离**，互不影响
