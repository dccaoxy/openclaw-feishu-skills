# 飞书音频消息 API 参考

## 获取 Access Token

```
POST https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal
```

```json
{
  "app_id": "cli_xxx",
  "app_secret": "xxx"
}
```

响应：
```json
{
  "code": 0,
  "msg": "ok",
  "tenant_access_token": "t-xxx"
}
```

## 上传音频文件

```
POST https://open.feishu.cn/open-apis/im/v1/files
```

**关键：必须包含 `duration` 参数（毫秒），否则语音显示时长为0**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| file_type | string | ✅ |固定为 `opus` |
| file_name | string | ✅ | 文件名 |
| duration | int | ✅ | **音频时长（毫秒）** |
| file | binary | ✅ | 文件内容 |

## 发送音频消息

```
POST https://open.feishu.cn/open-apis/im/v1/messages?receive_id_type=chat_id
```

Body:
```json
{
  "receive_id": "<chat_id>",
  "msg_type": "audio",
  "content": "{\"file_key\": \"<file_key>\"}"
}
```

响应：
```json
{
  "code": 0,
  "data": {
    "body": {
      "content": "{\"file_key\":\"file_v3_xxx\",\"duration\":2607}"
    },
    "chat_id": "oc_xxx",
    "message_id": "om_xxx",
    "msg_type": "audio"
  },
  "msg": "success"
}
```

## 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 语音显示0秒 | 上传时缺少 duration | 必须加 `duration=<毫秒>` |
| 发送为文件而非语音 | msg_type 错误 | 用 `audio` 而非 `file` |
| Token 过期 | Token 有效期2小时 | 重新获取 |

## 文件格式要求

- 格式：OPUS（需用 ffmpeg 转换）
- MP3 → OPUS 转换命令：
  ```bash
  ffmpeg -i input.mp3 -c:a libopus -b:a 128k output.opus -y
  ```

## 获取时长

```bash
# 方式1：ffprobe（输出秒）
DURATION=$(ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 input.opus)

# 方式2：转换为毫秒
DURATION_MS=$(echo "$DURATION * 1000" | bc)
```
